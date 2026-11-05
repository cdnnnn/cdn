//background.js
// Service worker: owns recording state, saved workflows and the replay "run" state.
// Replay state lives in chrome.storage.session so it survives page navigations and worker restarts.
importScripts('shared.js');

const SS = chrome.storage.session;
const SL = chrome.storage.local;

// ---------- storage helpers ----------
const getRec = async () => (await SS.get('rec')).rec || null;
const getRun = async () => (await SS.get('run')).run || null;
const getWorkflows = async () => (await SL.get('workflows')).workflows || {};
async function getSettings() {
  const s = (await SL.get('settings')).settings || {};
  return { stepDelay: 300, waitTimeout: 15, ...s };
}
async function setRec(r) {
  if (r) await SS.set({ rec: r }); else await SS.remove('rec');
  await refreshBadge();
}
async function setRun(r) {
  if (r) await SS.set({ run: r }); else await SS.remove('run');
  await refreshBadge();
}

// Handlers read-modify-write shared state, so run them one at a time.
let chain = Promise.resolve();
function locked(fn) {
  const p = chain.then(fn, fn);
  chain = p.catch(() => {});
  return p;
}

async function refreshBadge() {
  const [rec, run] = await Promise.all([getRec(), getRun()]);
  let text = '', color = '#555';
  if (rec && rec.active) { text = 'REC'; color = '#d93025'; }
  else if (run) {
    if (run.status === 'running') { text = '▶'; color = '#1a73e8'; }
    else if (run.status === 'failed') { text = '!'; color = '#d93025'; }
    else if (run.status === 'done') { text = '✓'; color = '#188038'; }
  }
  try {
    await chrome.action.setBadgeText({ text });
    await chrome.action.setBadgeBackgroundColor({ color });
  } catch (e) { /* ignore */ }
}

function notifyTab(tabId) {
  if (tabId == null) return;
  chrome.tabs.sendMessage(tabId, { type: 'mode-changed' }).catch(() => {});
}

async function gotoUrl(tabId, url) {
  const strip = (u) => (u || '').split('#')[0];
  const tab = await chrome.tabs.get(tabId);
  if (strip(tab.url) === strip(url)) await chrome.tabs.reload(tabId);
  else await chrome.tabs.update(tabId, { url });
}

const startIndexOf = (wf) => (wf.steps[0] && wf.steps[0].type === 'navigate' ? 1 : 0);
const startUrlOf = (wf) => (wf.steps[0] && wf.steps[0].type === 'navigate' ? wf.steps[0].url : wf.startUrl);

function sameLoc(a, b) {
  return !!a && !!b && a.css === b.css && a.id === b.id && a.name === b.name && a.value === b.value;
}

function finalizeSteps(steps) {
  const used = new Set();
  const strip = (u) => (u || '').split('#')[0];
  steps.forEach((s, i) => {
    // A click / Enter whose next recorded step happened on a different page leads to a navigation.
    const next = steps[i + 1];
    if ((s.type === 'click' || s.type === 'key') && next && next.url && s.url && strip(next.url) !== strip(s.url)) {
      s.navigates = true;
    }
    if (s.type === 'input' || s.type === 'select') {
      s.variable = true;
      s.param = RA.uniqueName(RA.slug(s.label || (s.loc && (s.loc.name || s.loc.placeholder || s.loc.id)) || s.type), used);
    } else if (s.type === 'check') {
      s.variable = false;
      s.param = RA.uniqueName(RA.slug(s.label || (s.loc && (s.loc.name || s.loc.id)) || 'checkbox'), used);
    }
    s.desc = RA.describe(s);
  });
  return steps;
}

// ---------- run progression ----------
async function endRowOrRun(run, ok, error) {
  run.results.push({ row: run.rowIndex, ok, error: error || null });
  run.claimedBy = null;
  const more = run.rowIndex + 1 < run.rows.length;
  if (ok || (run.continueOnError && more)) {
    if (more) {
      run.rowIndex += 1;
      run.stepIndex = run.startIndex;
      await setRun(run);
      await gotoUrl(run.tabId, run.startUrl);
      return { ok: true, next: true };
    }
    run.status = 'done';
    run.finishedAt = Date.now();
    await setRun(run);
    return { ok: true, done: true };
  }
  run.status = 'failed';
  run.error = error || 'Failed';
  run.errorStep = run.stepIndex;
  run.finishedAt = Date.now();
  await setRun(run);
  return { ok: true, failed: true };
}

async function modeFor(tabId, resetClaim) {
  const [rec, run] = await Promise.all([getRec(), getRun()]);
  const out = { recording: false, stepCount: 0, run: false, runInfo: null };
  if (rec && rec.active && rec.tabId === tabId) {
    out.recording = true;
    out.stepCount = Math.max(0, rec.steps.length - 1);
  }
  if (run && run.tabId === tabId) {
    if (run.status === 'running') {
      if (resetClaim && run.claimedBy) { run.claimedBy = null; await setRun(run); }
      out.run = true;
    }
    out.runInfo = {
      status: run.status, error: run.error, rowIndex: run.rowIndex,
      rowCount: run.rows.length, results: run.results, name: run.workflowName,
    };
  }
  return out;
}

// ---------- message handling ----------
async function handle(msg, sender) {
  const tabId = sender && sender.tab ? sender.tab.id : null;

  switch (msg.type) {
    case 'hello': return modeFor(tabId, true);
    case 'mode': return modeFor(tabId, false);

    // ----- recording -----
    case 'recStart': {
      const existing = await getRec();
      if (existing && existing.active) return { ok: false, error: 'Already recording' };
      const run = await getRun();
      if (run && run.status === 'running') return { ok: false, error: 'A replay is running' };
      await setRec({
        active: true, tabId: msg.tabId, name: msg.name || 'Untitled workflow',
        startUrl: msg.url, startedAt: Date.now(),
        steps: [{ type: 'navigate', url: msg.url }],
      });
      notifyTab(msg.tabId);
      return { ok: true };
    }
    case 'recStep': {
      const rec = await getRec();
      if (!rec || !rec.active || rec.tabId !== tabId) return { ok: false };
      const step = msg.step;
      const last = rec.steps[rec.steps.length - 1];
      if (msg.replace && last && last.type === step.type && sameLoc(last.loc, step.loc)) {
        rec.steps[rec.steps.length - 1] = step;
      } else {
        rec.steps.push(step);
      }
      await setRec(rec);
      return { ok: true, count: rec.steps.length - 1 };
    }
    case 'recStop': {
      const rec = await getRec();
      if (!rec) return { ok: false, error: 'Not recording' };
      await setRec(null);
      notifyTab(rec.tabId);
      if (msg.cancel) return { ok: true, cancelled: true };
      if (rec.steps.length < 2) return { ok: false, error: 'No steps were recorded' };
      const wf = {
        id: 'wf_' + Date.now().toString(36),
        name: rec.name, createdAt: Date.now(),
        startUrl: rec.startUrl, steps: finalizeSteps(rec.steps),
      };
      const all = await getWorkflows();
      all[wf.id] = wf;
      await SL.set({ workflows: all });
      return { ok: true, workflowId: wf.id };
    }

    // ----- saved workflows -----
    case 'wfSave': {
      const all = await getWorkflows();
      all[msg.workflow.id] = msg.workflow;
      await SL.set({ workflows: all });
      return { ok: true };
    }
    case 'wfDelete': {
      const all = await getWorkflows();
      delete all[msg.id];
      await SL.set({ workflows: all });
      return { ok: true };
    }

    // ----- replay -----
    case 'runStart': {
      const wf = (await getWorkflows())[msg.workflowId];
      if (!wf) return { ok: false, error: 'Workflow not found' };
      const existing = await getRun();
      if (existing && existing.status === 'running') return { ok: false, error: 'A replay is already running' };
      const rec = await getRec();
      if (rec && rec.active) return { ok: false, error: 'Stop recording first' };
      const url = startUrlOf(wf);
      let runTab = msg.tabId;
      if (msg.newTab) {
        const t = await chrome.tabs.create({ url, active: true });
        runTab = t.id;
      }
      const run = {
        id: 'run_' + Date.now().toString(36),
        workflowId: wf.id, workflowName: wf.name, workflow: wf,
        tabId: runTab, rows: msg.rows && msg.rows.length ? msg.rows : [{}],
        rowIndex: 0, startIndex: startIndexOf(wf), startUrl: url,
        stepIndex: startIndexOf(wf), status: 'running', claimedBy: null,
        continueOnError: !!msg.continueOnError, error: null, results: [],
        startedAt: Date.now(),
      };
      await setRun(run);
      if (!msg.newTab) await gotoUrl(runTab, url);
      return { ok: true };
    }
    case 'claim': {
      const run = await getRun();
      if (!run || run.status !== 'running' || run.tabId !== tabId) return { ok: false, reason: 'idle' };
      if (run.claimedBy && run.claimedBy !== msg.instance) return { ok: false, reason: 'claimed' };
      run.claimedBy = msg.instance;
      await setRun(run);
      const settings = await getSettings();
      const base = {
        ok: true, rowIndex: run.rowIndex, rowCount: run.rows.length,
        total: run.workflow.steps.length, stepDelay: settings.stepDelay,
        waitTimeout: settings.waitTimeout * 1000,
      };
      if (run.stepIndex >= run.workflow.steps.length) return { ...base, finalize: true };
      return {
        ...base, index: run.stepIndex, step: run.workflow.steps[run.stepIndex],
        params: run.rows[run.rowIndex] || {},
      };
    }
    case 'advance': {
      const run = await getRun();
      if (!run || run.status !== 'running' || run.tabId !== tabId) return { ok: false };
      if (run.stepIndex !== msg.index || run.claimedBy !== msg.instance) return { ok: false };
      run.stepIndex += 1;
      run.claimedBy = null;
      await setRun(run);
      return { ok: true };
    }
    case 'finishRow': {
      const run = await getRun();
      if (!run || run.status !== 'running' || run.tabId !== tabId || run.claimedBy !== msg.instance) return { ok: false };
      return endRowOrRun(run, true);
    }
    case 'fail': {
      const run = await getRun();
      if (!run || run.status !== 'running' || run.tabId !== tabId) return { ok: false };
      run.stepIndex = Math.min(msg.index != null ? msg.index : run.stepIndex, run.workflow.steps.length - 1);
      return endRowOrRun(run, false, msg.message);
    }
    case 'navigate': {
      const run = await getRun();
      if (!run || run.status !== 'running' || run.tabId !== tabId) return { ok: false };
      await chrome.tabs.update(tabId, { url: msg.url });
      return { ok: true };
    }
    case 'stopRun': {
      const run = await getRun();
      if (!run) return { ok: true };
      if (run.status === 'running') {
        run.status = 'stopped'; run.claimedBy = null; run.finishedAt = Date.now();
        await setRun(run);
      }
      notifyTab(run.tabId);
      return { ok: true };
    }
    case 'clearRun': {
      const run = await getRun();
      await setRun(null);
      if (run) notifyTab(run.tabId);
      return { ok: true };
    }
    default:
      return { ok: false, error: 'Unknown message ' + msg.type };
  }
}

chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  locked(() => handle(msg, sender))
    .then(sendResponse)
    .catch((e) => sendResponse({ ok: false, error: String((e && e.message) || e) }));
  return true;
});

chrome.tabs.onRemoved.addListener((tabId) => {
  locked(async () => {
    const run = await getRun();
    if (run && run.status === 'running' && run.tabId === tabId) {
      run.status = 'failed';
      run.error = 'The tab was closed during the run';
      run.finishedAt = Date.now();
      await setRun(run);
    }
  });
});

chrome.runtime.onInstalled.addListener(() => refreshBadge());
chrome.runtime.onStartup.addListener(() => refreshBadge());

























//content.js
// Content script: records what you do on the page, and replays recorded workflows.
(() => {
  if (window.__repeatAgentLoaded) return;
  window.__repeatAgentLoaded = true;

  const instance = Math.random().toString(36).slice(2) + Date.now().toString(36);
  const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
  const norm = (s) => (s || '').replace(/\s+/g, ' ').trim();
  const esc = (s) =>
    String(s == null ? '' : s).replace(/[&<>"']/g, (c) => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
  const send = (msg) =>
    new Promise((resolve) => {
      try {
        chrome.runtime.sendMessage(msg, (r) => { void chrome.runtime.lastError; resolve(r); });
      } catch (e) { resolve(undefined); }
    });
  class Abort extends Error {}

  let unloading = false;
  window.addEventListener('pagehide', () => { unloading = true; });
  window.addEventListener('beforeunload', () => { unloading = true; });
  window.addEventListener('pageshow', (e) => { if (e.persisted) { unloading = false; refreshMode(); } });

  // =====================================================================
  // Overlay (small floating panel in a shadow root)
  // =====================================================================
  let host = null, box = null;
  function ensureOverlay() {
    if (host && host.isConnected) return;
    host = document.createElement('div');
    host.id = '__repeat_agent_overlay';
    host.style.cssText = 'all:initial;position:fixed;z-index:2147483647;right:16px;bottom:16px;';
    const root = host.attachShadow({ mode: 'open' });
    root.innerHTML = `<style>
      .box{font:13px/1.4 system-ui,sans-serif;color:#202124;background:#fff;border:1px solid #dadce0;border-left:4px solid var(--c,#1a73e8);
        border-radius:8px;padding:10px 12px;min-width:240px;max-width:340px;box-shadow:0 4px 16px rgba(0,0,0,.2)}
      .t{font-weight:600;margin-bottom:2px}.m{color:#5f6368;word-break:break-word}
      .r{margin-top:8px;display:flex;gap:6px;justify-content:flex-end}
      button{font:inherit;cursor:pointer;border:0;border-radius:6px;padding:5px 10px;background:var(--c,#1a73e8);color:#fff}
      button.g{background:#f1f3f4;color:#202124}
    </style><div class="box"></div>`;
    box = root.querySelector('.box');
    document.documentElement.appendChild(host);
  }
  function setOverlay(html, handlers, color) {
    ensureOverlay();
    box.style.setProperty('--c', color || '#1a73e8');
    box.innerHTML = html;
    box.querySelectorAll('[data-act]').forEach((b) => {
      const fn = handlers && handlers[b.getAttribute('data-act')];
      if (fn) b.addEventListener('click', fn);
    });
  }
  function hideOverlay() {
    if (host) host.remove();
    host = null; box = null;
  }
  const inOverlay = (e) => !!host && e.composedPath().includes(host);

  function showRecording(count) {
    setOverlay(
      `<div class="t">● Recording</div><div class="m">${count} step${count === 1 ? '' : 's'} captured. Do the task once as usual.</div>
       <div class="r"><button class="g" data-act="cancel">Discard</button><button data-act="stop">Stop &amp; save</button></div>`,
      {
        stop: async () => {
          const r = await send({ type: 'recStop' });
          if (r && r.ok) {
            setOverlay('<div class="t">Saved ✓</div><div class="m">Open the Repeat Agent popup to review the fields and run it.</div>', {}, '#188038');
            setTimeout(refreshMode, 3500);
          } else {
            setOverlay(`<div class="t">Could not save</div><div class="m">${esc((r && r.error) || 'Unknown error')}</div>
              <div class="r"><button data-act="ok">OK</button></div>`, { ok: refreshMode }, '#d93025');
          }
        },
        cancel: async () => { await send({ type: 'recStop', cancel: true }); },
      },
      '#d93025'
    );
  }

  // =====================================================================
  // Locators: describe an element in several ways so it can be found again
  // =====================================================================
  const stableId = (id) => !!id && !/(\d{3,}|[:«»]|^(react|radix|headlessui|ember|mui|rc)[-_])/i.test(id);
  const attrSel = (n, v) => `[${n}="${String(v).replace(/\\/g, '\\\\').replace(/"/g, '\\"')}"]`;

  function labelOf(el) {
    const lb = el.getAttribute && el.getAttribute('aria-labelledby');
    if (lb) {
      const t = lb.split(/\s+/).map((id) => document.getElementById(id)).filter(Boolean).map((n) => norm(n.textContent)).join(' ');
      if (t) return clean(t);
    }
    if (el.labels && el.labels.length) {
      const t = norm(el.labels[0].textContent);
      if (t) return clean(t);
    }
    const wrap = el.closest && el.closest('label');
    if (wrap) {
      const t = norm(wrap.textContent);
      if (t) return clean(t);
    }
    const aria = el.getAttribute && el.getAttribute('aria-label');
    return aria ? clean(aria) : '';
  }
  const clean = (t) => t.replace(/\s*[*:]+\s*$/, '').slice(0, 80);

  function cssPath(el) {
    const parts = [];
    let node = el;
    while (node && node.nodeType === 1 && node !== document.documentElement) {
      if (node.id && stableId(node.id)) { parts.unshift('#' + CSS.escape(node.id)); break; }
      let part = node.tagName.toLowerCase();
      const parent = node.parentElement;
      if (parent) {
        const same = Array.from(parent.children).filter((c) => c.tagName === node.tagName);
        if (same.length > 1) part += `:nth-of-type(${same.indexOf(node) + 1})`;
      }
      parts.unshift(part);
      node = parent;
    }
    return parts.join(' > ');
  }

  const isFormControl = (el) => ['INPUT', 'SELECT', 'TEXTAREA'].includes(el.tagName);

  function visibleText(el) {
    if (el.tagName === 'INPUT') return norm(el.value);
    return norm(el.textContent);
  }

  function buildLocator(el) {
    const tag = el.tagName.toLowerCase();
    const loc = { tag };
    if (el.id && stableId(el.id)) loc.id = el.id;
    for (const a of ['data-testid', 'data-test', 'data-cy', 'data-qa', 'data-test-id']) {
      const v = el.getAttribute(a);
      if (v) { loc.testAttr = a; loc.testVal = v; break; }
    }
    const name = el.getAttribute('name');
    if (name) loc.name = name;
    if (tag === 'input') {
      loc.type = (el.type || 'text').toLowerCase();
      if (loc.type === 'radio' || loc.type === 'checkbox') loc.value = el.value;
    }
    const aria = el.getAttribute('aria-label');
    if (aria) loc.aria = aria;
    const ph = el.getAttribute('placeholder');
    if (ph) loc.placeholder = ph;
    const lbl = labelOf(el);
    if (lbl) loc.label = lbl;
    const role = el.getAttribute('role');
    if (role) loc.role = role;
    if (!el.isContentEditable && !['SELECT', 'TEXTAREA'].includes(el.tagName) && !(el.tagName === 'INPUT' && !['submit', 'button', 'reset'].includes(loc.type))) {
      const t = visibleText(el);
      if (t && t.length <= 80) loc.text = t;
    }
    if (tag === 'a') {
      const h = el.getAttribute('href');
      if (h && !/^javascript:/i.test(h)) loc.href = h.slice(0, 200);
    }
    loc.css = cssPath(el);
    return loc;
  }

  function isVisible(el) {
    const r = el.getBoundingClientRect();
    if (!(r.width > 0 && r.height > 0)) return false;
    const cs = getComputedStyle(el);
    return cs.visibility !== 'hidden' && cs.display !== 'none';
  }

  function resolveLocator(loc, allowHidden) {
    const qa = (sel) => { try { return Array.from(document.querySelectorAll(sel)); } catch (e) { return []; } };
    const typeOk = (e) => !loc.type || (e.type || '').toLowerCase() === loc.type;
    const valueOk = (e) => loc.value === undefined || e.value === loc.value;
    const textEq = (e) => visibleText(e) === loc.text;
    const deepest = (list) => list.filter((c) => !Array.from(c.children).some((ch) => list.includes(ch)));

    const strategies = [
      () => (loc.id ? [document.getElementById(loc.id)].filter(Boolean) : []),
      () => (loc.testAttr ? qa(attrSel(loc.testAttr, loc.testVal)) : []),
      () => (loc.name ? qa(loc.tag + attrSel('name', loc.name)).filter(typeOk).filter(valueOk) : []),
      () => (loc.aria ? qa(loc.tag + attrSel('aria-label', loc.aria)) : []),
      () => (loc.label && (isFormLike(loc)) ? qa(loc.tag).filter((e) => labelOf(e) === loc.label).filter(typeOk).filter(valueOk) : []),
      () => (loc.placeholder ? qa(loc.tag + attrSel('placeholder', loc.placeholder)) : []),
      () => (loc.text && loc.href ? deepest(qa(loc.tag).filter(textEq)).filter((e) => e.getAttribute('href') === loc.href) : []),
      () => (loc.text ? deepest(qa(loc.tag).filter(textEq)) : []),
      () => (loc.css ? qa(loc.css) : []),
    ];

    let fallback = null;
    for (const fn of strategies) {
      const list = fn();
      if (!list.length) continue;
      const pool = allowHidden ? list : list.filter(isVisible);
      if (pool.length === 1) return pool[0];
      if (pool.length > 1 && !fallback) fallback = pool[0];
    }
    return fallback;
  }
  const isFormLike = (loc) => ['input', 'select', 'textarea'].includes(loc.tag) || !!loc.role || loc.tag === 'div';

  async function waitFor(step, timeoutMs, allowHidden) {
    const end = Date.now() + timeoutMs;
    for (;;) {
      const el = resolveLocator(step.loc, allowHidden);
      if (el) return el;
      if (Date.now() > end) {
        const loc = step.loc || {};
        const what = step.label || loc.text || loc.aria || loc.placeholder || loc.name || loc.css || 'the element';
        throw new Error('Could not find "' + what + '" on the page (' + step.type + ' step)');
      }
      await sleep(200);
    }
  }

  // =====================================================================
  // Recorder
  // =====================================================================
  let recOn = false;
  const lastVal = new WeakMap();
  const initialVal = new WeakMap();

  function startRecorder() {
    if (recOn) return;
    recOn = true;
    document.addEventListener('click', onClick, true);
    document.addEventListener('change', onChange, true);
    document.addEventListener('keydown', onKey, true);
    document.addEventListener('focusin', onFocusIn, true);
    document.addEventListener('focusout', onFocusOut, true);
  }
  function stopRecorder() {
    if (!recOn) return;
    recOn = false;
    document.removeEventListener('click', onClick, true);
    document.removeEventListener('change', onChange, true);
    document.removeEventListener('keydown', onKey, true);
    document.removeEventListener('focusin', onFocusIn, true);
    document.removeEventListener('focusout', onFocusOut, true);
  }

  async function record(step, replace) {
    step.url = location.href;
    const r = await send({ type: 'recStep', step, replace: !!replace });
    if (r && r.ok) showRecording(r.count);
  }

  const CLICKABLE =
    'button, a, [role="button"], [role="link"], [role="tab"], [role="menuitem"], [role="option"], [role="checkbox"], [role="radio"], [role="switch"], input[type="submit"], input[type="button"], input[type="reset"], input[type="image"], summary, label, [onclick]';

  function onClick(e) {
    if (!e.isTrusted || inOverlay(e) || !(e.target instanceof Element)) return;
    let el = e.target.closest(CLICKABLE) || e.target;
    if (el === document.body || el === document.documentElement) return;
    if (el.tagName === 'LABEL' && el.control) return; // the control's own change event is recorded
    if (['SELECT', 'TEXTAREA'].includes(el.tagName) || el.isContentEditable) return;
    if (el.tagName === 'INPUT' && !['submit', 'button', 'reset', 'image'].includes((el.type || '').toLowerCase())) return;
    record({ type: 'click', loc: buildLocator(el), label: clickLabel(el) });
  }
  function clickLabel(el) {
    return clean(visibleText(el) || el.getAttribute('aria-label') || el.getAttribute('title') || el.getAttribute('value') || '');
  }

  function onChange(e) {
    if (!e.isTrusted || inOverlay(e) || !(e.target instanceof Element)) return;
    recordField(e.target);
  }
  function onFocusIn(e) {
    const el = e.target;
    if (el instanceof Element && el.isContentEditable && !isFormControl(el)) initialVal.set(el, el.innerText);
  }
  function onFocusOut(e) {
    const el = e.target;
    if (!(el instanceof Element) || !el.isContentEditable || isFormControl(el) || inOverlay(e)) return;
    if (el.innerText !== initialVal.get(el)) recordField(el);
  }
  function onKey(e) {
    if (!e.isTrusted || e.key !== 'Enter' || inOverlay(e) || !(e.target instanceof Element)) return;
    const el = e.target;
    if (el.tagName !== 'INPUT' || !isTextLike(el)) return;
    recordField(el); // flush the typed value first so the order is right
    record({ type: 'key', key: 'Enter', loc: buildLocator(el), label: labelOf(el) || el.name || '' });
  }

  function isTextLike(el) {
    const t = (el.type || 'text').toLowerCase();
    return !['checkbox', 'radio', 'file', 'submit', 'button', 'reset', 'image', 'hidden'].includes(t);
  }
  const fieldLabel = (el) => labelOf(el) || clean(el.getAttribute('placeholder') || el.getAttribute('name') || el.id || el.type || '');

  function recordField(el) {
    const tag = el.tagName;
    if (tag === 'SELECT') {
      const opts = Array.from(el.options).slice(0, 80).map((o) => norm(o.textContent));
      if (el.multiple) {
        const sel = Array.from(el.selectedOptions).map((o) => norm(o.textContent));
        record({ type: 'select', multi: true, loc: buildLocator(el), label: fieldLabel(el), value: sel.join(','), text: sel.join(','), options: opts }, true);
      } else {
        const o = el.selectedOptions[0];
        record({ type: 'select', loc: buildLocator(el), label: fieldLabel(el), value: el.value, text: o ? norm(o.textContent) : '', options: opts }, true);
      }
      return;
    }
    if (tag === 'INPUT') {
      const type = (el.type || 'text').toLowerCase();
      if (type === 'checkbox' || type === 'radio') {
        record({ type: 'check', kind: type, loc: buildLocator(el), label: fieldLabel(el), checked: el.checked }, true);
        return;
      }
      if (type === 'file') {
        record({ type: 'manual', message: 'Choose the file for "' + (fieldLabel(el) || 'file upload') + '", then press Continue' });
        return;
      }
      if (!isTextLike(el)) return;
    }
    if (tag === 'INPUT' || tag === 'TEXTAREA' || el.isContentEditable) {
      const editable = !isFormControl(el);
      const host = editable ? el.closest('[contenteditable]:not([contenteditable="false"])') || el : el;
      const value = editable ? host.innerText : el.value;
      if (lastVal.get(host) === value) return;
      lastVal.set(host, value);
      const ac = (el.getAttribute('autocomplete') || '').toLowerCase();
      const secret = (el.type || '').toLowerCase() === 'password' || /cc-|one-time-code/.test(ac);
      record({
        type: 'input', editable, loc: buildLocator(host), label: fieldLabel(host),
        value: secret ? '' : value, secret,
      }, true);
    }
  }

  // =====================================================================
  // Replayer
  // =====================================================================
  function setNativeValue(el, value) {
    const desc = Object.getOwnPropertyDescriptor(Object.getPrototypeOf(el), 'value');
    if (desc && desc.set) desc.set.call(el, value); else el.value = value;
  }
  const fire = (el, type) => el.dispatchEvent(new Event(type, { bubbles: true }));

  function highlight(el) {
    try {
      const old = el.style.outline;
      el.style.outline = '3px solid #f9ab00';
      setTimeout(() => { el.style.outline = old; }, 700);
    } catch (e) { /* ignore */ }
  }

  function fillInput(el, value) {
    el.scrollIntoView({ block: 'center', inline: 'center' });
    el.focus();
    if (el.isContentEditable && !isFormControl(el)) {
      document.execCommand('selectAll', false);
      if (!document.execCommand('insertText', false, value)) {
        el.textContent = value;
        el.dispatchEvent(new InputEvent('input', { bubbles: true, inputType: 'insertText', data: value }));
      }
    } else {
      setNativeValue(el, value);
      fire(el, 'input');
      fire(el, 'change');
    }
    if (el.blur) el.blur();
  }

  function chooseOption(el, step, value) {
    const wanted = String(value).split(step.multi ? ',' : '\u0000').map((v) => v.trim()).filter((v, i, a) => v !== '' || a.length === 1);
    const opts = Array.from(el.options);
    if (step.multi) opts.forEach((o) => { o.selected = false; });
    wanted.forEach((w) => {
      const lw = w.toLowerCase();
      const o =
        opts.find((x) => x.value === w) ||
        opts.find((x) => norm(x.textContent).toLowerCase() === lw) ||
        opts.find((x) => x.value.toLowerCase() === lw) ||
        opts.find((x) => norm(x.textContent).toLowerCase().includes(lw) && lw !== '');
      if (!o) throw new Error('Option "' + w + '" not found in "' + (step.label || 'dropdown') + '"');
      if (step.multi) o.selected = true; else el.value = o.value;
    });
    fire(el, 'input');
    fire(el, 'change');
  }

  const parseBool = (v) => /^(true|yes|y|1|on|checked)$/i.test(String(v).trim());

  function clickElement(el) {
    el.scrollIntoView({ block: 'center', inline: 'center' });
    highlight(el);
    const o = { bubbles: true, cancelable: true, view: window };
    try { el.dispatchEvent(new PointerEvent('pointerdown', o)); } catch (e) { /* older engines */ }
    el.dispatchEvent(new MouseEvent('mousedown', o));
    if (el.focus) el.focus();
    try { el.dispatchEvent(new PointerEvent('pointerup', o)); } catch (e) { /* ignore */ }
    el.dispatchEvent(new MouseEvent('mouseup', o));
    el.click();
  }

  function pressEnter(el) {
    const o = { key: 'Enter', code: 'Enter', keyCode: 13, which: 13, bubbles: true, cancelable: true };
    const proceed = el.dispatchEvent(new KeyboardEvent('keydown', o));
    el.dispatchEvent(new KeyboardEvent('keypress', o));
    if (proceed) {
      const form = el.form || el.closest('form');
      if (form) { if (form.requestSubmit) form.requestSubmit(); else form.submit(); }
    }
    el.dispatchEvent(new KeyboardEvent('keyup', o));
  }

  async function settle(ms) {
    const end = Date.now() + ms;
    while (Date.now() < end) {
      if (unloading) return 'unloading';
      await sleep(50);
    }
    return unloading ? 'unloading' : 'ok';
  }

  // After a click / Enter. If the recording showed this step leads to another page, wait for that
  // navigation (or an in-page URL change) so the next step never runs against the old page.
  async function afterAction(r, step) {
    const href = location.href;
    const expectNav = !!step.navigates;
    const end = Date.now() + (expectNav ? 15000 : Math.max(r.stepDelay, 500));
    while (Date.now() < end) {
      if (unloading) return 'unloading';
      if (expectNav && location.href !== href) {
        await sleep(300);
        return unloading ? 'unloading' : 'ok';
      }
      await sleep(50);
    }
    return unloading ? 'unloading' : 'ok';
  }

  function valueFor(step, params) {
    if (step.variable && step.param && Object.prototype.hasOwnProperty.call(params || {}, step.param)) {
      return params[step.param];
    }
    if (step.secret) throw new Error('No value was provided for the secret field "' + (step.label || step.param) + '"');
    if (step.type === 'select') return step.text || step.value;
    return step.value;
  }

  async function waitManual(message) {
    const res = await new Promise((resolve) => {
      setOverlay(
        `<div class="t">⏸ Your turn</div><div class="m">${esc(message)}</div>
         <div class="r"><button class="g" data-act="stop">Stop</button><button data-act="go">Continue</button></div>`,
        {
          go: () => resolve('go'),
          stop: async () => { await send({ type: 'stopRun' }); resolve('stop'); },
        },
        '#f9ab00'
      );
    });
    if (res === 'stop') throw new Abort();
  }

  async function executeStep(r) {
    const { step, params, index } = r;
    const timeout = r.waitTimeout;
    const commit = async () => {
      const a = await send({ type: 'advance', instance, index });
      if (!a || !a.ok) throw new Abort();
    };
    try {
      if (step.pauseBefore) await waitManual('Paused before: ' + RA.describe(step));
      showStatus(r);
      switch (step.type) {
        case 'navigate':
          await commit();
          await send({ type: 'navigate', url: step.url });
          return (await settle(8000)) === 'unloading' ? 'unloading' : 'ok';
        case 'click': {
          const el = await waitFor(step, timeout, false);
          await commit();
          clickElement(el);
          return afterAction(r, step);
        }
        case 'input': {
          const el = await waitFor(step, timeout, false);
          highlight(el);
          fillInput(el, String(valueFor(step, params)));
          await commit();
          return 'ok';
        }
        case 'select': {
          const el = await waitFor(step, timeout, true);
          highlight(el);
          chooseOption(el, step, valueFor(step, params));
          await commit();
          return 'ok';
        }
        case 'check': {
          const el = await waitFor(step, timeout, true);
          const want = step.kind === 'radio' ? true : step.variable ? parseBool(valueFor(step, params)) : !!step.checked;
          if (el.checked !== want) el.click();
          await commit();
          return 'ok';
        }
        case 'key': {
          const el = await waitFor(step, timeout, false);
          await commit();
          pressEnter(el);
          return afterAction(r, step);
        }
        case 'manual':
          await waitManual(step.message || 'Please do this step manually');
          await commit();
          return 'ok';
        default:
          throw new Error('Unknown step type: ' + step.type);
      }
    } catch (e) {
      if (e instanceof Abort) return 'abort';
      await send({ type: 'fail', instance, index, message: (e && e.message) || String(e) });
      return 'failed';
    }
  }

  function showStatus(r) {
    const row = r.rowCount > 1 ? `Row ${r.rowIndex + 1}/${r.rowCount} · ` : '';
    setOverlay(
      `<div class="t">▶ Replaying</div><div class="m">${row}Step ${r.index + 1}/${r.total}: ${esc(RA.describe(r.step))}</div>
       <div class="r"><button class="g" data-act="stop">Stop</button></div>`,
      { stop: async () => { await send({ type: 'stopRun' }); } }
    );
  }

  let looping = false;
  let navigatingAway = false; // the background is sending this tab to the next batch row
  async function runLoop() {
    if (looping || navigatingAway) return;
    looping = true;
    try {
      if (document.readyState !== 'complete') await new Promise((res) => window.addEventListener('load', res, { once: true }));
      await sleep(300);
      for (;;) {
        if (unloading) return;
        const r = await send({ type: 'claim', instance });
        if (!r || !r.ok) break;
        if (r.finalize) {
          setOverlay('<div class="t">▶ Finishing…</div>', {});
          if ((await settle(1200)) === 'unloading') return;
          const f = await send({ type: 'finishRow', instance });
          if (f && f.ok && f.next) { navigatingAway = true; return; } // tab is reloading for the next row
          break;
        }
        const res = await executeStep(r);
        if (res === 'unloading' || res === 'abort' || res === 'failed') break;
        await sleep(r.stepDelay);
      }
    } finally {
      looping = false;
      if (!unloading && !navigatingAway) refreshMode();
    }
  }

  // =====================================================================
  // Mode handling
  // =====================================================================
  function applyMode(m) {
    if (!m) return;
    if (m.recording) {
      startRecorder();
      showRecording(m.stepCount);
      return;
    }
    stopRecorder();
    if (m.run) {
      runLoop();
      return;
    }
    const ri = m.runInfo;
    if (ri && ri.status === 'done') {
      const bad = ri.results.filter((x) => !x.ok).length;
      setOverlay(
        `<div class="t">✓ Finished</div><div class="m">${esc(ri.name)} — ${ri.results.length - bad} succeeded${bad ? ', ' + bad + ' failed' : ''}.</div>
         <div class="r"><button data-act="ok">Dismiss</button></div>`,
        { ok: async () => { await send({ type: 'clearRun' }); } }, '#188038'
      );
    } else if (ri && ri.status === 'failed') {
      setOverlay(
        `<div class="t">✗ Stopped</div><div class="m">${esc(ri.error || 'Something went wrong')}</div>
         <div class="r"><button data-act="ok">Dismiss</button></div>`,
        { ok: async () => { await send({ type: 'clearRun' }); } }, '#d93025'
      );
    } else {
      hideOverlay();
    }
  }

  async function refreshMode() { applyMode(await send({ type: 'mode' })); }

  chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
    if (msg && msg.type === 'ping') sendResponse({ pong: true });
    else if (msg && msg.type === 'mode-changed') refreshMode();
  });

  (async () => { applyMode(await send({ type: 'hello' })); })();
})();























//manifest.json
{
  "manifest_version": 3,
  "name": "Repeat Agent",
  "version": "0.1.0",
  "description": "Record a repetitive web task once (e.g. filling a form), then replay it with new values.",
  "permissions": ["storage", "tabs", "scripting"],
  "host_permissions": ["<all_urls>"],
  "background": { "service_worker": "background.js" },
  "action": {
    "default_popup": "popup.html",
    "default_title": "Repeat Agent",
    "default_icon": { "16": "icons/icon16.png", "32": "icons/icon32.png" }
  },
  "content_scripts": [
    {
      "matches": ["http://*/*", "https://*/*"],
      "js": ["shared.js", "content.js"],
      "run_at": "document_idle"
    }
  ],
  "icons": {
    "16": "icons/icon16.png",
    "32": "icons/icon32.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  }
}

















//popup.css
* { box-sizing: border-box; }
body { width: 400px; max-height: 580px; overflow-y: auto; margin: 0; font: 13px/1.4 system-ui, sans-serif; color: #202124; background: #fff; }
header { padding: 10px 14px; background: #1a73e8; color: #fff; display: flex; flex-direction: column; }
header .sub { font-size: 11px; opacity: .85; }
main { padding: 10px 14px 14px; }
h4 { margin: 12px 0 6px; font-size: 13px; }
h4:first-child { margin-top: 0; }
section { margin-bottom: 8px; }
input[type=text], input:not([type]), input[type=password], input[type=number], select, textarea {
  width: 100%; padding: 6px 8px; border: 1px solid #dadce0; border-radius: 6px; font: inherit; margin: 2px 0 6px;
}
textarea { min-height: 70px; font-family: ui-monospace, monospace; font-size: 12px; }
button { font: inherit; cursor: pointer; border: 1px solid #dadce0; background: #f8f9fa; border-radius: 6px; padding: 5px 10px; }
button:hover { background: #f1f3f4; }
button.primary { background: #1a73e8; color: #fff; border-color: #1a73e8; width: 100%; padding: 8px; }
button.primary:hover { background: #1765cc; }
button.link { border: 0; background: none; color: #1a73e8; padding: 2px 4px; }
button.danger { color: #d93025; }
.hint { color: #5f6368; font-size: 12px; margin: 4px 0; }
.card { border: 1px solid #dadce0; border-radius: 8px; padding: 8px 10px; margin-bottom: 8px; }
.card .nm { font-weight: 600; }
.card .meta { color: #5f6368; font-size: 12px; margin-bottom: 6px; }
.card .acts { display: flex; gap: 6px; flex-wrap: wrap; }
.bar { display: flex; justify-content: space-between; align-items: center; gap: 8px; padding: 8px 14px; font-size: 12px; }
.bar.rec { background: #fce8e6; color: #a50e0e; }
.bar.run { background: #e8f0fe; color: #174ea6; }
.bar.ok { background: #e6f4ea; color: #0d652d; }
.bar.err { background: #fce8e6; color: #a50e0e; }
.bar span.b { display: flex; gap: 6px; flex-shrink: 0; }
.notice { padding: 6px 8px; border-radius: 6px; background: #e8f0fe; color: #174ea6; margin-bottom: 8px; }
.notice.err { background: #fce8e6; color: #a50e0e; }
label.f { display: block; font-size: 12px; color: #3c4043; }
label.f small { color: #80868b; }
label.c { display: flex; align-items: center; gap: 6px; margin: 4px 0; }
label.c input { width: auto; margin: 0; }
details { margin: 6px 0; }
summary { cursor: pointer; color: #1a73e8; }
.step { border-top: 1px solid #eee; padding: 6px 0; }
.step .sd { font-size: 12px; margin-bottom: 4px; word-break: break-word; }
.step .n { display: inline-block; min-width: 18px; color: #80868b; }
.step .row { display: flex; gap: 6px; align-items: center; }
.step .row input[type=text] { margin: 0; }
.step .sa { display: flex; justify-content: space-between; align-items: center; font-size: 12px; margin-top: 2px; }
.step label.c { margin: 0; font-size: 12px; }




















//popup.html
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <link rel="stylesheet" href="popup.css">
</head>
<body>
  <header><b>Repeat Agent</b><span class="sub">record once · replay with new values</span></header>
  <div id="banner"></div>
  <main id="main"></main>
  <script src="shared.js"></script>
  <script src="popup.js"></script>
</body>
</html>




















//popup.js
(() => {
  const $ = (s, r = document) => r.querySelector(s);
  const esc = (s) =>
    String(s == null ? '' : s).replace(/[&<>"']/g, (c) => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
  const bg = (msg) =>
    new Promise((res) =>
      chrome.runtime.sendMessage(msg, (r) => { void chrome.runtime.lastError; res(r || { ok: false, error: 'No response from the background worker' }); })
    );

  const S = {
    workflows: {}, rec: null, run: null,
    settings: { stepDelay: 300, waitTimeout: 15 },
    view: { name: 'home' }, draft: null, notice: '',
  };

  async function load() {
    const [l, s] = await Promise.all([
      chrome.storage.local.get(['workflows', 'settings']),
      chrome.storage.session.get(['rec', 'run']),
    ]);
    S.workflows = l.workflows || {};
    S.settings = { stepDelay: 300, waitTimeout: 15, ...(l.settings || {}) };
    S.rec = s.rec || null;
    S.run = s.run || null;
  }
  const flash = (text, err) => { S.notice = text ? `<div class="notice${err ? ' err' : ''}">${esc(text)}</div>` : ''; };

  // ---------------------------------------------------------------- banner
  function renderBanner() {
    const { rec, run } = S;
    let h = '';
    if (rec && rec.active) {
      h += `<div class="bar rec"><span>● Recording “${esc(rec.name)}” · ${rec.steps.length - 1} steps</span>
        <span class="b"><button data-act="recstop">Stop &amp; save</button><button data-act="reccancel">Discard</button></span></div>`;
    }
    if (run) {
      const total = run.workflow ? run.workflow.steps.length : 0;
      const bad = run.results.filter((r) => !r.ok).length;
      const good = run.results.length - bad;
      if (run.status === 'running') {
        h += `<div class="bar run"><span>▶ Running “${esc(run.workflowName)}” · row ${run.rowIndex + 1}/${run.rows.length} · step ${Math.min(run.stepIndex + 1, total)}/${total}</span>
          <span class="b"><button data-act="runstop">Stop</button></span></div>`;
      } else if (run.status === 'done') {
        h += `<div class="bar ok"><span>✓ Finished “${esc(run.workflowName)}” — ${good} ok${bad ? ', ' + bad + ' failed' : ''}</span>
          <span class="b"><button data-act="rundismiss">Dismiss</button></span></div>`;
      } else {
        const label = run.status === 'stopped' ? '■ Stopped' : '✗ Failed: ' + esc(run.error || 'error');
        h += `<div class="bar err"><span>${label}</span><span class="b"><button data-act="rundismiss">Dismiss</button></span></div>`;
      }
    }
    $('#banner').innerHTML = h;
  }

  // ---------------------------------------------------------------- home
  function hostOf(wf) { try { return new URL(wf.startUrl).host; } catch (e) { return ''; } }
  function renderHome() {
    const wfs = Object.values(S.workflows).sort((a, b) => b.createdAt - a.createdAt);
    const cards = wfs.map((w) => `<div class="card">
        <div class="nm">${esc(w.name)}</div>
        <div class="meta">${w.steps.length - 1} steps · ${esc(hostOf(w))}</div>
        <div class="acts">
          <button data-act="run" data-id="${w.id}">▶ Run</button>
          <button data-act="edit" data-id="${w.id}">Edit</button>
          <button data-act="copy" data-id="${w.id}">Copy JSON</button>
          <button class="danger" data-act="del" data-id="${w.id}">Delete</button>
        </div></div>`).join('');
    $('#main').innerHTML = `${S.notice}
      <section><h4>Record a new workflow</h4>
        <input type="text" id="wfName" placeholder="Name, e.g. Register new customer">
        <button class="primary" data-act="rec">● Start recording on this tab</button>
        <p class="hint">Do the task once on the page. When finished, press <b>Stop &amp; save</b> in the panel on the page.
        Typed values become fields you can fill next time.</p></section>
      <section><h4>Saved workflows</h4>${cards || '<p class="hint">Nothing saved yet.</p>'}</section>
      <details><summary>Settings &amp; import</summary>
        <label class="f">Speed
          <select id="speed">
            <option value="120">Fast</option><option value="300">Normal</option><option value="800">Slow (safer on heavy sites)</option>
          </select></label>
        <label class="f">Wait up to (seconds) for a field to appear
          <input type="number" id="timeout" min="3" max="120" value="${S.settings.waitTimeout}"></label>
        <label class="f">Import a workflow (paste JSON)</label>
        <textarea id="impText"></textarea>
        <button data-act="import">Import</button>
      </details>`;
    $('#speed').value = String(S.settings.stepDelay);
  }

  // ---------------------------------------------------------------- run view
  function paramsOf(wf) {
    const out = [], seen = new Set();
    for (const s of wf.steps) {
      if (!s.variable || !s.param || seen.has(s.param)) continue;
      seen.add(s.param);
      out.push({
        param: s.param, label: s.label || s.param, type: s.type, secret: !!s.secret, options: s.options || [],
        def: s.type === 'check' ? (s.checked ? 'yes' : 'no') : s.type === 'select' ? s.text || s.value || '' : s.secret ? '' : s.value || '',
      });
    }
    return out;
  }
  function renderRun() {
    const wf = S.workflows[S.view.id];
    if (!wf) { S.view = { name: 'home' }; return render(); }
    const fields = paramsOf(wf).map((p) => {
      const id = 'dl_' + p.param;
      let input;
      if (p.type === 'check') {
        input = `<select data-param="${esc(p.param)}"><option value="yes"${p.def === 'yes' ? ' selected' : ''}>yes</option><option value="no"${p.def === 'no' ? ' selected' : ''}>no</option></select>`;
      } else {
        input = `<input type="${p.secret ? 'password' : 'text'}" data-param="${esc(p.param)}" value="${esc(p.def)}" ${p.options.length ? `list="${id}"` : ''} ${p.secret ? 'autocomplete="off" placeholder="not stored — enter it for this run"' : ''}>
          ${p.options.length ? `<datalist id="${id}">${p.options.map((o) => `<option value="${esc(o)}">`).join('')}</datalist>` : ''}`;
      }
      return `<label class="f">${esc(p.label)} <small>{{${esc(p.param)}}}</small>${input}</label>`;
    }).join('');
    $('#main').innerHTML = `${S.notice}
      <button class="link" data-act="back">← Back</button>
      <h4>${esc(wf.name)}</h4>
      ${fields || '<p class="hint">This workflow has no variable fields. It will just replay.</p>'}
      <details><summary>Paste values (one “field: value” per line, or JSON)</summary>
        <textarea id="qf" placeholder="full_name: Asha Rao&#10;email: asha@example.com"></textarea></details>
      <details><summary>Batch — run once per row (CSV with header row, or JSON list)</summary>
        <textarea id="batch" placeholder="full_name,email&#10;Asha Rao,asha@example.com&#10;Ravi K,ravi@example.com"></textarea>
        <p class="hint">Column names = the field names shown above. Blank cells use the value in the form.</p></details>
      <label class="c"><input type="checkbox" id="newTab" checked> Open in a new tab</label>
      <label class="c"><input type="checkbox" id="skip"> Keep going if a row fails</label>
      <button class="primary" data-act="go" data-id="${wf.id}">▶ Run</button>`;
  }

  // ---------------------------------------------------------------- edit view
  function renderEdit() {
    const d = S.draft;
    if (!d) { S.view = { name: 'home' }; return render(); }
    const steps = d.steps.map((s, i) => {
      let editor = '';
      if (s.type === 'navigate') {
        editor = `<input type="text" data-f="url" value="${esc(s.url)}">`;
      } else if (['input', 'select', 'check'].includes(s.type)) {
        const secret = !!s.secret;
        editor = `<div class="row">
          <label class="c"><input type="checkbox" data-f="variable" ${s.variable || secret ? 'checked' : ''} ${secret ? 'disabled' : ''}> field</label>
          <input type="text" data-f="param" value="${esc(s.param)}" title="Field name">
          ${s.type === 'check'
            ? `<label class="c"><input type="checkbox" data-f="checked" ${s.checked ? 'checked' : ''}> default on</label>`
            : secret ? '' : `<input type="text" data-f="def" value="${esc(s.type === 'select' ? s.text || s.value || '' : s.value || '')}" title="Default value" placeholder="default">`}
        </div>`;
      }
      return `<div class="step" data-i="${i}">
        <div class="sd"><span class="n">${i + 1}</span>${esc(RA.describe(s))}</div>${editor}
        <div class="sa">${i > 0 ? `<label class="c"><input type="checkbox" data-f="pauseBefore" ${s.pauseBefore ? 'checked' : ''}> pause before (e.g. CAPTCHA)</label>
          <button class="link danger" data-act="delstep" data-i="${i}">remove step</button>` : '<span></span>'}</div>
      </div>`;
    }).join('');
    $('#main').innerHTML = `${S.notice}
      <button class="link" data-act="back">← Back</button>
      <h4>Edit workflow</h4>
      <input type="text" id="wfRename" value="${esc(d.name)}">
      <p class="hint">Tick <b>field</b> on a step to make its value something you supply each run. Untick to keep the recorded value fixed.</p>
      ${steps}
      <button class="primary" data-act="savedit" style="margin-top:10px">Save changes</button>`;
  }

  // Copy what the user typed in the editor back into the draft.
  function syncDraft() {
    const d = S.draft;
    if (!d || S.view.name !== 'edit') return;
    const nm = $('#wfRename');
    if (nm) d.name = nm.value.trim() || d.name;
    document.querySelectorAll('.step').forEach((el) => {
      const s = d.steps[+el.dataset.i];
      if (!s) return;
      const get = (f) => el.querySelector(`[data-f="${f}"]`);
      if (get('url')) s.url = get('url').value.trim();
      if (get('pauseBefore')) s.pauseBefore = get('pauseBefore').checked;
      if (get('param')) s.param = RA.slug(get('param').value);
      if (get('variable')) s.variable = s.secret ? true : get('variable').checked;
      if (get('checked')) s.checked = get('checked').checked;
      if (get('def')) {
        const v = get('def').value;
        if (s.type === 'select') { s.text = v; s.value = v; } else s.value = v;
      }
    });
  }

  function render() {
    renderBanner();
    const v = S.view.name;
    if (v === 'home') renderHome(); else if (v === 'run') renderRun(); else if (v === 'edit') renderEdit();
  }

  // ---------------------------------------------------------------- actions
  async function ensureContent(tabId) {
    try {
      const r = await chrome.tabs.sendMessage(tabId, { type: 'ping' });
      if (r && r.pong) return;
    } catch (e) { /* not injected yet */ }
    await chrome.scripting.executeScript({ target: { tabId }, files: ['shared.js', 'content.js'] });
  }

  async function startRecording() {
    const name = ($('#wfName').value || '').trim();
    if (!name) { flash('Give the workflow a name first.', true); return render(); }
    const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
    if (!tab || !/^https?:/i.test(tab.url || '')) {
      flash('Open the web page where the task starts (http/https), then try again.', true);
      return render();
    }
    try { await ensureContent(tab.id); } catch (e) {
      flash('Cannot record on this page (browser-protected pages are not allowed).', true);
      return render();
    }
    const r = await bg({ type: 'recStart', name, tabId: tab.id, url: tab.url });
    if (!r.ok) { flash(r.error || 'Could not start recording', true); return render(); }
    window.close();
  }

  async function goRun(id) {
    const wf = S.workflows[id];
    const params = paramsOf(wf);
    const form = {};
    document.querySelectorAll('[data-param]').forEach((el) => { form[el.dataset.param] = el.value; });
    let rows;
    try { rows = RA.parseTable($('#batch').value); } catch (e) {
      flash('Batch data problem: ' + e.message, true);
      return renderRunKeepingInputs();
    }
    if (!rows.length) rows = [{}];
    rows = rows.map((r) => {
      const o = {};
      params.forEach((p) => {
        const v = r[p.param];
        o[p.param] = v === undefined || v === '' ? form[p.param] : v;
      });
      return o;
    });
    for (let i = 0; i < rows.length; i++) {
      for (const p of params) {
        if (p.secret && !rows[i][p.param]) {
          flash(`“${p.label}” is a secret field and is not stored — type it in the form (row ${i + 1}).`, true);
          return renderRunKeepingInputs();
        }
      }
    }
    const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
    const r = await bg({
      type: 'runStart', workflowId: id, rows, newTab: $('#newTab').checked,
      tabId: tab && tab.id, continueOnError: $('#skip').checked,
    });
    if (!r.ok) { flash(r.error || 'Could not start', true); return renderRunKeepingInputs(); }
    window.close();
  }
  // Re-render the run view without losing what the user typed.
  function renderRunKeepingInputs() {
    const saved = {};
    document.querySelectorAll('[data-param]').forEach((el) => { saved[el.dataset.param] = el.value; });
    const qf = $('#qf') && $('#qf').value, batch = $('#batch') && $('#batch').value;
    renderRun();
    document.querySelectorAll('[data-param]').forEach((el) => { if (saved[el.dataset.param] !== undefined) el.value = saved[el.dataset.param]; });
    if (qf) $('#qf').value = qf;
    if (batch) { $('#batch').value = batch; document.querySelector('#batch').closest('details').open = true; }
  }

  async function saveEdit() {
    syncDraft();
    const d = S.draft;
    for (const s of d.steps) {
      if (s.variable && !s.param) { flash('Every field needs a name.', true); return renderEdit(); }
      s.desc = RA.describe(s);
    }
    if (!/^https?:/i.test(d.steps[0].url || '')) { flash('The start URL must begin with http:// or https://', true); return renderEdit(); }
    d.startUrl = d.steps[0].url;
    const r = await bg({ type: 'wfSave', workflow: d });
    await load();
    S.draft = null; S.view = { name: 'home' };
    flash(r.ok ? 'Saved.' : r.error || 'Save failed', !r.ok);
    render();
  }

  const handlers = {
    rec: startRecording,
    async recstop() {
      const r = await bg({ type: 'recStop' });
      await load();
      if (r.ok) {
        S.view = { name: 'edit', id: r.workflowId };
        S.draft = JSON.parse(JSON.stringify(S.workflows[r.workflowId]));
        flash('Saved. Check the field names below — these are what you will fill next time.');
      } else flash(r.error || 'Could not save', true);
      render();
    },
    async reccancel() { await bg({ type: 'recStop', cancel: true }); await load(); render(); },
    async runstop() { await bg({ type: 'stopRun' }); await load(); render(); },
    async rundismiss() { await bg({ type: 'clearRun' }); await load(); render(); },
    run(el) { S.view = { name: 'run', id: el.dataset.id }; flash(''); render(); },
    edit(el) { S.view = { name: 'edit', id: el.dataset.id }; S.draft = JSON.parse(JSON.stringify(S.workflows[el.dataset.id])); flash(''); render(); },
    back() { S.view = { name: 'home' }; S.draft = null; flash(''); render(); },
    go(el) { return goRun(el.dataset.id); },
    savedit: saveEdit,
    delstep(el) { syncDraft(); S.draft.steps.splice(+el.dataset.i, 1); render(); },
    async del(el) {
      const wf = S.workflows[el.dataset.id];
      if (!confirm(`Delete “${wf.name}”?`)) return;
      await bg({ type: 'wfDelete', id: wf.id });
      await load(); flash('Deleted.'); render();
    },
    async copy(el) {
      await navigator.clipboard.writeText(JSON.stringify(S.workflows[el.dataset.id], null, 2));
      flash('Workflow JSON copied to the clipboard.'); render();
    },
    async import() {
      try {
        const wf = JSON.parse($('#impText').value);
        if (!wf || !Array.isArray(wf.steps) || !wf.steps.length || wf.steps[0].type !== 'navigate') throw new Error('Not a Repeat Agent workflow');
        wf.id = 'wf_' + Date.now().toString(36);
        wf.createdAt = Date.now();
        const r = await bg({ type: 'wfSave', workflow: wf });
        if (!r.ok) throw new Error(r.error);
        await load(); flash('Imported “' + wf.name + '”.');
      } catch (e) { flash('Import failed: ' + e.message, true); }
      render();
    },
  };

  document.addEventListener('click', (e) => {
    const el = e.target.closest('[data-act]');
    if (!el) return;
    const fn = handlers[el.dataset.act];
    if (fn) fn(el);
  });

  document.addEventListener('input', (e) => {
    // "Paste values" box fills the form fields.
    if (e.target.id === 'qf') {
      let vals;
      try { vals = RA.parseQuickFill(e.target.value); } catch (err) { return; }
      document.querySelectorAll('[data-param]').forEach((el) => {
        if (vals[el.dataset.param] !== undefined) el.value = vals[el.dataset.param];
      });
    }
  });
  document.addEventListener('change', async (e) => {
    if (e.target.id === 'speed' || e.target.id === 'timeout') {
      const settings = { stepDelay: +$('#speed').value, waitTimeout: Math.max(3, +$('#timeout').value || 15) };
      await chrome.storage.local.set({ settings });
      S.settings = settings;
    }
  });

  // Live status updates without clobbering whatever the user is typing.
  chrome.storage.onChanged.addListener(async () => {
    await load();
    if (S.view.name === 'home') render(); else renderBanner();
  });

  load().then(render);
})();
























//shared.js
// Helpers shared by the popup, the background worker and the content script.
var RA = (() => {
  const slug = (s) =>
    String(s == null ? '' : s)
      .toLowerCase()
      .replace(/[^a-z0-9]+/g, '_')
      .replace(/^_+|_+$/g, '')
      .slice(0, 40) || 'field';

  const uniqueName = (base, used) => {
    let n = base, i = 2;
    while (used.has(n)) n = base + '_' + i++;
    used.add(n);
    return n;
  };

  const q = (s) => '"' + String(s == null ? '' : s).replace(/\s+/g, ' ').trim().slice(0, 50) + '"';

  function describe(s) {
    const loc = s.loc || {};
    const target = s.label || loc.text || loc.aria || loc.placeholder || loc.name || loc.css || 'element';
    switch (s.type) {
      case 'navigate':
        return 'Open ' + s.url;
      case 'click':
        return 'Click ' + q(target);
      case 'input':
        if (s.variable) return 'Type {{' + s.param + '}} into ' + q(target);
        return 'Type ' + (s.secret ? '(secret)' : q(s.value)) + ' into ' + q(target);
      case 'select':
        if (s.variable) return 'Choose {{' + s.param + '}} in ' + q(target);
        return 'Choose ' + q(s.text || s.value) + ' in ' + q(target);
      case 'check':
        if (s.variable) return 'Set {{' + s.param + '}} (yes/no) on ' + q(target);
        return (s.checked ? 'Check ' : 'Uncheck ') + q(target);
      case 'key':
        return 'Press ' + (s.key || 'Enter') + ' in ' + q(target);
      case 'manual':
        return s.message || 'Wait for you to do something';
      default:
        return s.type;
    }
  }

  // CSV / TSV parser (quotes supported). Tabs are accepted so rows pasted from a spreadsheet work.
  function parseCsv(text) {
    const rows = [];
    let row = [], cell = '', inQ = false;
    for (let i = 0; i < text.length; i++) {
      const c = text[i];
      if (inQ) {
        if (c === '"') {
          if (text[i + 1] === '"') { cell += '"'; i++; } else inQ = false;
        } else cell += c;
      } else if (c === '"') inQ = true;
      else if (c === ',' || c === '\t') { row.push(cell); cell = ''; }
      else if (c === '\n' || c === '\r') {
        if (c === '\r' && text[i + 1] === '\n') i++;
        row.push(cell); rows.push(row); row = []; cell = '';
      } else cell += c;
    }
    if (cell !== '' || row.length) { row.push(cell); rows.push(row); }
    return rows.filter((r) => r.some((c) => c.trim() !== ''));
  }

  const slugKeys = (obj) => {
    const out = {};
    for (const k of Object.keys(obj)) out[slug(k)] = obj[k] == null ? '' : String(obj[k]);
    return out;
  };

  // "[{...},{...}]", "{...}" or CSV with a header row -> array of {param: value}
  function parseTable(text) {
    const t = (text || '').trim();
    if (!t) return [];
    if (t[0] === '[') {
      const arr = JSON.parse(t);
      if (!Array.isArray(arr)) throw new Error('JSON must be a list of objects');
      return arr.map(slugKeys);
    }
    if (t[0] === '{') return [slugKeys(JSON.parse(t))];
    const rows = parseCsv(t);
    if (rows.length < 2) throw new Error('Need a header row plus at least one data row');
    const head = rows[0].map((h) => slug(h));
    return rows.slice(1).map((r) => {
      const o = {};
      head.forEach((h, i) => { o[h] = r[i] == null ? '' : r[i]; });
      return o;
    });
  }

  // "name: John\nemail: j@x.com" or JSON object -> {name:'John', email:'j@x.com'}
  function parseQuickFill(text) {
    const t = (text || '').trim();
    if (!t) return {};
    if (t[0] === '{') return slugKeys(JSON.parse(t));
    const out = {};
    for (const line of t.split(/\r?\n/)) {
      const m = line.match(/^\s*([^:=]+?)\s*[:=]\s*(.*)$/);
      if (m) out[slug(m[1])] = m[2].trim();
    }
    return out;
  }

  return { slug, uniqueName, describe, parseCsv, parseTable, parseQuickFill };
})();
