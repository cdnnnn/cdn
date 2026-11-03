//inpage.js
// Functions in this file are injected into the page via chrome.scripting.executeScript({func}).
// They must be fully self-contained (no references to outside variables).

function snapshotFn(maxText, maxEls, opts) {
  maxEls = maxEls || 200;
  // Shadow hosts are discovered ONCE per snapshot (a full "*" scan per query is far too slow on pages with thousands of nodes).
  let hostCache = null;
  function shadowHosts() {
    if (hostCache) return hostCache;
    hostCache = [];
    const stack = [document];
    while (stack.length) {
      const r = stack.pop();
      r.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) { hostCache.push(e); stack.push(e.shadowRoot); } });
    }
    return hostCache;
  }
  function deepAll(selector, root, out) {
    root = root || document;
    out = out || [];
    root.querySelectorAll(selector).forEach((e) => out.push(e));
    if (root === document) for (const h of shadowHosts()) h.shadowRoot.querySelectorAll(selector).forEach((e) => out.push(e));
    return out;
  }

  deepAll("[data-aq-id]").forEach((e) => e.removeAttribute("data-aq-id"));
  const SEL =
    'a[href],button,input,select,textarea,summary,[role=button],[role=link],[role=checkbox],[role=radio],' +
    '[role=tab],[role=menuitem],[role=option],[role=combobox],[role=switch],[role=textbox],' +
    '[contenteditable=""],[contenteditable="true"],[onclick],[tabindex]:not([tabindex="-1"])';
  const SENS =
    /\bpass(word|code|phrase)\b|\bcard.?(number|no)\b|\bcc-(number|csc|exp)|\bcvv\b|\bcvc\b|\botp\b|one.?time.?(code|password)|\bssn\b|aadhaar|social.?security|contrase[ñn]a|clave de acceso|n[úu]mero de (tarjeta|cuenta)|mot de passe|code secret|num[ée]ro de carte|passwort|kennwort|kartennummer|prüfziffer|senha|n[úu]mero do cart[ãa]o|пароль|номер карты|одноразов|密码|口令|卡号|银行卡|验证码|身份证|パスワード|暗証|カード番号|認証コード|ワンタイム|كلمة (المرور|السر)|رقم البطاقة|رمز التحقق|पासवर्ड|कार्ड नंबर|ओटीपी|आधार|পাসওয়ার্ড|কার্ড নম্বর|ওটিপি|கடவுச்சொல்|அட்டை எண்|ఓటిపి|పాస్‌వర్డ్|పాస్వర్డ్|కార్డ్ నంబర్|पासवर्ड|કાર્ડ નંબર|પાસવર્ડ|ಪಾಸ್‌ವರ್ಡ್|ಪಾಸ್ವರ್ಡ್|ಕಾರ್ಡ್ ಸಂಖ್ಯೆ|പാസ്‌വേഡ്|പാസ്വേഡ്|കാർഡ് നമ്പർ|ਪਾਸਵਰਡ|ਕਾਰਡ ਨੰਬਰ|ପାସୱାର୍ଡ|କାର୍ଡ ନମ୍ବର|ଓଟିପି|비밀번호|패스워드|암호|카드 ?번호|인증 ?(번호|코드)|보안 ?코드|주민 ?등록|일회용/i;
  const clean = (s) => (s || "").replace(/\s+/g, " ").trim();
  const txt = (el) => (el.innerText !== undefined ? el.innerText : el.textContent) || "";
  const vh = window.innerHeight || 800;

  const isVisible = (el) => {
    const r = el.getBoundingClientRect();
    if (r.width < 2 || r.height < 2) return false;
    const cs = getComputedStyle(el);
    return cs.visibility !== "hidden" && cs.display !== "none" && cs.opacity !== "0";
  };

  const labelOf = (el) => {
    let l = el.getAttribute("aria-label");
    const lb = el.getAttribute("aria-labelledby");
    if (!l && lb) l = lb.split(" ").map((i) => { const n = document.getElementById(i); return n ? txt(n) : ""; }).join(" ");
    if (!l && el.labels && el.labels[0]) l = txt(el.labels[0]);
    if (!l && ["INPUT", "TEXTAREA", "SELECT"].includes(el.tagName)) l = el.placeholder || el.name || el.id;
    if (!l && el.tagName === "INPUT" && ["submit", "button"].includes(el.type)) l = el.value;
    if (!l) { const img = el.querySelector && el.querySelector("img[alt]"); l = txt(el) || el.title || el.alt || (img && img.alt) || ""; }
    l = clean(l);
    if (!l) {
      const svgTitle = el.querySelector && el.querySelector("svg title");
      l = clean(svgTitle ? svgTitle.textContent : "");
    }
    if (!l) {
      const hint = clean(el.getAttribute("data-testid") || el.getAttribute("data-test") || el.id || (typeof el.className === "string" ? el.className : ""));
      if (hint) l = "(icon: " + hint.slice(0, 40) + ")";
    }
    return l.slice(0, 80);
  };

  const base = new Set(deepAll(SEL));
  // Hidden checkbox/radio (custom switches): the visible label is what a user clicks.
  base.forEach((el) => {
    if (el.tagName === "INPUT" && (el.type === "checkbox" || el.type === "radio") && !isVisible(el) && el.labels && el.labels[0]) base.add(el.labels[0]);
  });
  // Clickable things with no semantic markup (div/span with a click handler show a pointer cursor).
  if (base.size <= 400) {
    const vw = window.innerWidth || 1200;
    let scanned = 0;
    for (const el of deepAll("div,span,li,label,svg,img,td,p,i,section,article")) {
      if (++scanned > 4000) break;
      if (base.has(el)) continue;
      if (getComputedStyle(el).cursor !== "pointer") continue;
      const par = el.parentElement;
      if (par && getComputedStyle(par).cursor === "pointer") continue;
      if (el.querySelector(SEL)) continue;
      const r = el.getBoundingClientRect();
      if (r.width * r.height > 0.5 * vw * vh) continue;
      base.add(el);
    }
  }
  const viewCache = new Map();
  const inView = (el) => { let v = viewCache.get(el); if (v === undefined) { const r = el.getBoundingClientRect(); v = r.bottom > 0 && r.top < vh; viewCache.set(el, v); } return v; };
  let cand = [...base].filter((el) => !(el.tagName === "INPUT" && el.type === "hidden"));
  cand.sort((a, b) => (a.compareDocumentPosition(b) & 4 ? -1 : 1));
  let nodes;
  if (cand.length > maxEls * 3) {
    // Huge page (thousands of cards/rows): only the first maxEls VISIBLE controls matter (on-screen first), so stop checking
    // computed styles once we have enough instead of testing every node.
    const a = [], b = [];
    for (const el of cand) (inView(el) ? a : b).push(el);
    nodes = [];
    for (const el of a) { if (nodes.length >= maxEls) break; if (isVisible(el)) nodes.push(el); }
    for (const el of b) { if (nodes.length >= maxEls) break; if (isVisible(el)) nodes.push(el); }
    nodes.sort((x, y) => (x.compareDocumentPosition(y) & 4 ? -1 : 1));
  } else {
    nodes = cand.filter(isVisible);
    if (nodes.length > maxEls) {
      const a = nodes.filter(inView);
      const b = nodes.filter((n) => !inView(n));
      const keep = new Set([...a, ...b].slice(0, maxEls));
      nodes = nodes.filter((n) => keep.has(n));
    }
  }

  // Compact "how to find this element again" descriptor, used when a successful run is saved as a skill.
  const okId = (id) => !!id && id.length < 60 && !/\d{3,}|^:|[:\s]/.test(id);
  const cssPath = (el) => {
    const parts = [];
    let n = el;
    for (let i = 0; n && n.nodeType === 1 && i < 6; i++, n = n.parentElement) {
      if (okId(n.id)) { parts.unshift("#" + String(n.id).replace(/[^\w-]/g, "\\$&")); break; }
      let s = n.tagName.toLowerCase();
      // position among same-tag siblings, counted by walking backwards (capped, so huge lists stay cheap)
      let idx = 1, steps = 0, sib = n.previousElementSibling, capped = false;
      while (sib && steps++ < 40) { if (sib.tagName === n.tagName) idx++; sib = sib.previousElementSibling; }
      if (sib) capped = true;
      if (!capped && (idx > 1 || (n.nextElementSibling && n.nextElementSibling.tagName === n.tagName))) s += ":nth-of-type(" + idx + ")";
      parts.unshift(s);
    }
    return parts.join(" > ");
  };
  const describeForSkill = (el, label) => {
    const row = el.closest && el.closest("tr,li,[role=row],[role=listitem],[role=dialog],dialog,form");
    let ctx = row && row !== el ? clean(txt(row)).slice(0, 50) : "";
    if (ctx === label) ctx = "";
    const d = {
      tag: el.tagName.toLowerCase(), role: el.getAttribute("role") || "", type: el.tagName === "INPUT" ? el.type : "",
      id: okId(el.id) ? el.id : "", name: el.getAttribute("name") || "",
      testid: el.getAttribute("data-testid") || el.getAttribute("data-test") || el.getAttribute("data-qa") || el.getAttribute("data-cy") || "",
      label: /^\(icon:/.test(label) ? "" : label, placeholder: el.getAttribute("placeholder") || "", ctx, nth: 0, css: "",
    };
    try { d.css = cssPath(el); } catch (e) {}
    if (el.tagName === "A" && el.getAttribute("href")) { try { d.href = new URL(el.href).pathname.slice(0, 100); } catch (e) {} }
    return d;
  };

  const elements = nodes.map((el, i) => {
    const id = i + 1;
    el.setAttribute("data-aq-id", String(id));
    const tag = el.tagName.toLowerCase();
    const type = tag === "input" ? (el.type || "text") : "";
    const label = labelOf(el);
    const sensitive =
      type === "password" || SENS.test([el.getAttribute("autocomplete"), el.name, el.id, label].join(" "));
    const o = { id, tag, role: el.getAttribute("role") || "", type, label, sensitive, inView: inView(el) };
    o.d = describeForSkill(el, label);
    if (!sensitive && (tag === "input" || tag === "textarea") && !["checkbox", "radio", "submit", "button"].includes(type)) {
      o.value = String(el.value || "").slice(0, 60);
    }
    if (tag === "select") {
      const sel = el.options[el.selectedIndex];
      o.value = sel ? clean(sel.text) : "";
      o.options = [...el.options].slice(0, 25).map((x) => clean(x.text));
    }
    if (tag === "label" && el.control && (el.control.type === "checkbox" || el.control.type === "radio")) { o.checked = !!el.control.checked; o.role = "toggle"; }
    else if (type === "checkbox" || type === "radio") o.checked = !!el.checked;
    else if (el.getAttribute("aria-checked")) o.checked = el.getAttribute("aria-checked") === "true";
    if (el.disabled || el.getAttribute("aria-disabled") === "true") o.disabled = true;
    if (el.required) o.required = true;
    if (tag === "a" && el.href) {
      try { const u = new URL(el.href); o.href = (u.host + u.pathname).slice(0, 80); } catch (e) {}
    }
    return o;
  });

  // Tour mode only: also expose page AREAS (nav, sidebar, header, tables, forms, headings...) so a guide can explain
  // the whole app, and tell each control which area/row it sits in so it is described accurately.
  let regions = [];
  if (opts && opts.tour) {
    const RSEL =
      "header,nav,aside,main,footer,section,form,table,dialog,h1,h2,h3," +
      "[role=banner],[role=navigation],[role=complementary],[role=main],[role=contentinfo],[role=region],[role=tablist],[role=toolbar]," +
      "[role=search],[role=grid],[role=table],[role=dialog],[role=menu],[role=tree]," +
      "[class*=sidebar],[class*=side-bar],[class*=navbar],[class*=toolbar],[class*=breadcrumb],[class*=drawer],[class*=modal],[class*=tabs]" +
      (opts.full ? ",h4,canvas,[class*=chart],[class*=card],[class*=widget],[class*=kpi],[class*=tile],[class*=panel],[class*=banner],[class*=alert]" : "");
    const KIND_OK = /^(header|nav|aside|main|footer|table|form|dialog|banner|navigation|complementary|contentinfo|tablist|toolbar|search|grid|menu|tree|canvas)$/;
    const regionEls = [];
    const regionOf = new Map();
    let n = elements.length;
    const vwid = window.innerWidth || 1200;
    for (const el of deepAll(RSEL)) {
      if (regions.length >= (opts.full ? 80 : 40)) break;
      if (el.hasAttribute("data-aq-id")) continue;
      const r = el.getBoundingClientRect();
      if (r.width < 60 || r.height < 24 || !isVisible(el)) continue;
      if (r.right <= 0 || r.left >= vwid) continue;
      const tag = el.tagName.toLowerCase();
      const kind = el.getAttribute("role") || (/^h[1-3]$/.test(tag) ? "heading" : tag);
      let label = clean(el.getAttribute("aria-label") || "");
      const lb = el.getAttribute("aria-labelledby");
      if (!label && lb) label = clean(lb.split(" ").map((i) => { const x = document.getElementById(i); return x ? txt(x) : ""; }).join(" "));
      if (!label && tag === "table") { const c = el.querySelector("caption"); label = clean(c ? txt(c) : ""); }
      if (!label && kind === "heading") label = clean(txt(el));
      if (!label) { const h = el.querySelector("h1,h2,h3,h4,legend"); label = clean(h ? txt(h) : ""); }
      if (!label && opts.full && !KIND_OK.test(kind)) label = clean(txt(el)).slice(0, 50); // cards/widgets: name them by their first text
      label = label.slice(0, 60);
      if (!label && !KIND_OK.test(kind)) continue; // an unlabeled generic <section>/div adds noise
      n++;
      el.setAttribute("data-aq-id", String(n));
      const reg = { id: n, kind, label };
      regions.push(reg);
      regionOf.set(el, reg);
      regionEls.push([el, reg]);
    }
    nodes.forEach((el, i) => {
      let p = el.parentElement || (el.getRootNode && el.getRootNode().host), g = 0;
      while (p && g++ < 14) {
        const reg = regionOf.get(p);
        if (reg && reg.kind !== "heading") { elements[i].area = reg.kind + (reg.label ? ' "' + reg.label + '"' : ""); break; }
        p = p.parentElement || (p.getRootNode && p.getRootNode().host);
      }
      const row = el.closest && el.closest("tr,li,[role=row],[role=listitem]");
      if (row) {
        const rt = clean(txt(row)).slice(0, 60);
        if (rt && rt !== elements[i].label) elements[i].ctx = rt;
      }
    });
    // reading order (document order) across controls AND areas, so a tour can walk the page top to bottom
    const seqItems = nodes.map((el, i) => [el, elements[i]]).concat(regionEls);
    seqItems.sort((a, b) => {
      if (a[0] === b[0]) return 0;
      const c = a[0].compareDocumentPosition(b[0]);
      return c & 4 ? -1 : c & 2 ? 1 : 0;
    });
    seqItems.forEach((x, k) => { x[1].seq = k; });
  }

  // Repeated structures (cards, rows, list items...). The model only ever sees a slice of a big page, so it needs to know
  // what repeats and how often, and can then use query_page to count/list them instantly instead of scrolling.
  let repeats = [], totalElements = 0;
  try {
    const all = document.body ? document.body.getElementsByTagName("*") : [];
    totalElements = all.length;
    const lim = Math.min(all.length, 60000);
    const cssEsc = (c) => (typeof CSS !== "undefined" && CSS.escape ? CSS.escape(c) : c.replace(/[^\w-]/g, "\\$&"));
    const keyCache = new Map();
    const counts = new Map();
    for (let i = 0; i < lim; i++) {
      const e = all[i];
      const cn = e.className;
      if (typeof cn !== "string" || !cn) continue;
      const ck = e.tagName + "|" + cn;
      let key = keyCache.get(ck);
      if (key === undefined) {
        const cls = cn.split(/\s+/).filter((c) => c && c.length <= 40 && !/\d/.test(c)).slice(0, 3);
        key = cls.length ? e.tagName.toLowerCase() + "." + cls.map((c) => cssEsc(c)).join(".") : "";
        keyCache.set(ck, key);
      }
      if (!key) continue;
      const c = counts.get(key);
      if (c) c.n++; else counts.set(key, { n: 1, first: e });
    }
    const cands = [...counts.entries()].filter(([, v]) => v.n >= 6).map(([sel, v]) => ({ sel, n: v.n, first: v.first, size: v.first.getElementsByTagName("*").length }));
    cands.sort((a, b) => b.n - a.n || b.size - a.size);
    repeats = cands.slice(0, 8).map((c) => {
      let n = c.n;
      try { n = document.querySelectorAll(c.sel).length; } catch (e) {}
      return { selector: c.sel, count: n, sample: clean(c.first.textContent || "").slice(0, 70) };
    });
  } catch (e) {}

  const body = (document.body ? txt(document.body) : "").replace(/\n{3,}/g, "\n\n");
  return {
    repeats,
    totalElements,
    url: location.href,
    title: document.title,
    scrollY: Math.round(window.scrollY),
    scrollMax: Math.max(0, Math.round(document.documentElement.scrollHeight - vh)),
    iframes: document.querySelectorAll("iframe").length,
    elements,
    regions,
    text: body.slice(0, maxText),
    totalChars: body.length,
  };
}

// Read-only aggregate query over the WHOLE page (not just the visible slice). Used for "how many ...", "list all ...", "which ... are ..." on big pages.
function queryFn(q) {
  const clean = (x) => (x || "").replace(/\s+/g, " ").trim();
  const out = { ok: true };
  try {
    const sel = String(q.selector || "").trim();
    if (!sel) return { ok: false, msg: "selector is required (a CSS selector such as .card or tr.row)." };
    const nodes = [];
    const walk = (root) => {
      root.querySelectorAll(sel).forEach((e) => nodes.push(e));
      root.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) walk(e.shadowRoot); });
    };
    try { walk(document); } catch (e) { return { ok: false, msg: "Invalid CSS selector: " + e.message }; }
    let m = nodes;
    if (q.text_contains) {
      const needle = String(q.text_contains).toLowerCase();
      m = m.filter((e) => (e.textContent || "").toLowerCase().includes(needle));
    }
    if (q.visible_only) {
      m = m.filter((e) => {
        const r = e.getBoundingClientRect();
        if (r.width < 2 || r.height < 2) return false;
        const cs = getComputedStyle(e);
        return cs.visibility !== "hidden" && cs.display !== "none";
      });
    }
    out.count = m.length;
    out.matchedBeforeFilters = nodes.length;
    const nlist = Math.max(0, Math.min(Number(q.list) || 0, 50));
    if (nlist) out.items = m.slice(0, nlist).map((e) => clean(e.textContent).slice(0, 160));
    if (q.count_by) {
      const g = new Map();
      for (const e of m) {
        let t = "";
        try { const x = e.querySelector(String(q.count_by)); t = x ? clean(x.textContent).slice(0, 60) : "(none)"; } catch (err) { return { ok: false, msg: "Invalid count_by selector: " + err.message }; }
        g.set(t, (g.get(t) || 0) + 1);
      }
      out.groups = [...g.entries()].sort((a, b) => b[1] - a[1]).slice(0, 30).map(([value, n]) => ({ value, n }));
      out.distinct = g.size;
    }
    if (q.attribute) {
      const vals = m.slice(0, 50).map((e) => e.getAttribute(String(q.attribute))).filter((v) => v != null);
      out.attributes = vals.map((v) => String(v).slice(0, 120));
    }
    return out;
  } catch (e) {
    return { ok: false, msg: String(e && e.message || e) };
  }
}

// Resolves as soon as the page has stopped changing (no DOM mutations for quietMs), or after maxMs at the latest.
// Much faster than sleeping a fixed time after every action.
function quietFn(quietMs, maxMs) {
  return new Promise((resolve) => {
    const t0 = Date.now();
    let last = t0;
    const mo = new MutationObserver(() => { last = Date.now(); });
    try {
      mo.observe(document.documentElement, { subtree: true, childList: true, characterData: true, attributes: true, attributeFilter: ["class", "hidden", "open", "disabled", "aria-expanded", "aria-hidden", "aria-busy"] });
    } catch (e) {}
    const iv = setInterval(() => {
      const now = Date.now();
      if (now - last >= quietMs || now - t0 >= maxMs) { clearInterval(iv); mo.disconnect(); resolve(now - t0); }
    }, 25);
  });
}

function readFn(max) {
  const t = (document.body ? (document.body.innerText !== undefined ? document.body.innerText : document.body.textContent) : "") || "";
  const body = t.replace(/\n{3,}/g, "\n\n");
  return { text: body.slice(0, max), totalChars: body.length };
}

function actFn(name, a) {
  function deepAll(selector, root, out) {
    root = root || document;
    out = out || [];
    root.querySelectorAll(selector).forEach((e) => out.push(e));
    root.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) deepAll(selector, e.shadowRoot, out); });
    return out;
  }
  const find = (id) => deepAll('[data-aq-id="' + id + '"]')[0] || null;
  const notFound = { ok: false, msg: "Element not found. The page changed; use ids from the latest snapshot." };
  const key = (el, k) => {
    for (const t of ["keydown", "keypress", "keyup"]) {
      el.dispatchEvent(new KeyboardEvent(t, { key: k, code: k, bubbles: true, cancelable: true }));
    }
  };
  const fire = (el) => {
    el.dispatchEvent(new Event("input", { bubbles: true }));
    el.dispatchEvent(new Event("change", { bubbles: true }));
  };
  try {
    if (name === "click") {
      const el = find(a.id);
      if (!el) return notFound;
      if (el.disabled) return { ok: false, msg: "Element is disabled." };
      if (el.scrollIntoView) el.scrollIntoView({ block: "center", inline: "center" });
      if (el.focus) el.focus({ preventScroll: true });
      try {
        const r = el.getBoundingClientRect();
        const o = { bubbles: true, cancelable: true, view: window, clientX: r.left + r.width / 2, clientY: r.top + r.height / 2, button: 0 };
        if (typeof PointerEvent === "function") el.dispatchEvent(new PointerEvent("pointerdown", { ...o, pointerType: "mouse", isPrimary: true }));
        el.dispatchEvent(new MouseEvent("mousedown", o));
        if (typeof PointerEvent === "function") el.dispatchEvent(new PointerEvent("pointerup", { ...o, pointerType: "mouse", isPrimary: true }));
        el.dispatchEvent(new MouseEvent("mouseup", o));
      } catch (e) {}
      el.click();
      return { ok: true, msg: "Clicked." };
    }
    if (name === "type_text") {
      const el = find(a.id);
      if (!el) return notFound;
      if (el.scrollIntoView) el.scrollIntoView({ block: "center" });
      if (el.focus) el.focus();
      const isTA = el instanceof HTMLTextAreaElement;
      if (isTA || el instanceof HTMLInputElement) {
        const proto = isTA ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
        const set = Object.getOwnPropertyDescriptor(proto, "value").set;
        set.call(el, (a.append ? el.value : "") + a.text);
        fire(el);
      } else if (el.isContentEditable) {
        if (!a.append) document.execCommand("selectAll", false);
        document.execCommand("insertText", false, a.text);
      } else {
        return { ok: false, msg: "Element is not a text field." };
      }
      if (a.submit) {
        if (el.form && el.form.requestSubmit) el.form.requestSubmit();
        else key(el, "Enter");
      }
      return { ok: true, msg: "Typed." };
    }
    if (name === "select_option") {
      const el = find(a.id);
      if (!el) return notFound;
      if (el.tagName !== "SELECT") return { ok: false, msg: "Not a <select>. Click it and choose the option instead." };
      const want = String(a.option).trim().toLowerCase();
      const opts = [...el.options];
      const opt =
        opts.find((o) => o.text.trim().toLowerCase() === want || o.value.toLowerCase() === want) ||
        opts.find((o) => o.text.toLowerCase().includes(want));
      if (!opt) return { ok: false, msg: "No such option. Available: " + opts.slice(0, 25).map((o) => o.text.trim()).join(" | ") };
      const set = Object.getOwnPropertyDescriptor(HTMLSelectElement.prototype, "value").set;
      set.call(el, opt.value);
      fire(el);
      return { ok: true, msg: "Selected " + opt.text.trim() + "." };
    }
    if (name === "scroll") {
      const h = window.innerHeight || 800;
      if (a.direction === "top") window.scrollTo(0, 0);
      else if (a.direction === "bottom") window.scrollTo(0, document.documentElement.scrollHeight);
      else window.scrollBy(0, (a.direction === "up" ? -1 : 1) * Math.round(h * 0.8));
      return { ok: true, msg: "Scrolled." };
    }
    if (name === "press_key") {
      const el = document.activeElement || document.body;
      if (a.key === "Enter" && (el.tagName === "BUTTON" || el.tagName === "A")) el.click();
      else key(el, a.key);
      return { ok: true, msg: "Pressed " + a.key + "." };
    }
    return { ok: false, msg: "Unknown action " + name };
  } catch (e) {
    return { ok: false, msg: String(e) };
  }
}

// ---------- guided tour overlay (runs in the page; drawn in a shadow root, never touches the page's own DOM/styles) ----------
// steps: [{id?, title, text, click?}]  L: translated labels  o: {finished, rtl}
// A step that points at an element is only shown if that element is REALLY visible on screen (rendered, not clipped by a
// scroll area / collapsed panel, not covered). Otherwise the step is dropped, so the guide never highlights empty space.
// Reports back with chrome.runtime.sendMessage({__pa_tour:true, type:"advance"|"exit", reason}).
function tourFn(steps, L, o) {
  if (window.__paTour) window.__paTour.destroy();
  steps = steps.slice();
  const send = (m) => { try { chrome.runtime.sendMessage({ __pa_tour: true, ...m }); } catch (e) {} };
  const deepAll = (selector, root = document, out = []) => {
    root.querySelectorAll(selector).forEach((e) => out.push(e));
    root.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) deepAll(selector, e.shadowRoot, out); });
    return out;
  };
  const findEl = (s) => (s && s.id != null ? deepAll('[data-aq-id="' + s.id + '"]')[0] || null : null);

  // ----- is this element really on screen? -----
  const okStyle = (c) => {
    if (!c || !c.isConnected || !c.getBoundingClientRect) return false;
    const cs = getComputedStyle(c);
    if (cs.display === "none" || cs.visibility === "hidden" || parseFloat(cs.opacity) < 0.05) return false;
    if (c.closest && c.closest("[hidden],[inert]")) return false;
    return true;
  };
  // visible part of the element after clipping by the viewport and by every overflow:hidden/auto/scroll ancestor
  const clipRect = (el, r) => {
    let Lf = Math.max(0, r.left), Tp = Math.max(0, r.top), Rt = Math.min(window.innerWidth, r.right), Bt = Math.min(window.innerHeight, r.bottom);
    let cur = el, p = el.parentNode, guard = 0;
    while (p && guard++ < 60) {
      if (p.nodeType === 11) { cur = p.host; p = p.host && p.host.parentNode; continue; }
      if (p.nodeType !== 1) break;
      if (getComputedStyle(cur).position === "fixed") break; // fixed elements ignore ancestors' overflow
      if (p !== document.documentElement && p !== document.body) {
        const cs = getComputedStyle(p);
        if (cs.overflowX !== "visible" || cs.overflowY !== "visible") {
          const pr = p.getBoundingClientRect();
          Lf = Math.max(Lf, pr.left); Tp = Math.max(Tp, pr.top); Rt = Math.min(Rt, pr.right); Bt = Math.min(Bt, pr.bottom);
        }
      }
      cur = p; p = p.parentNode;
    }
    return { left: Lf, top: Tp, right: Rt, bottom: Bt, width: Math.max(0, Rt - Lf), height: Math.max(0, Bt - Tp) };
  };
  const covered = (c, cr) => {
    if (typeof document.elementFromPoint !== "function") return false;
    const pts = [[0.5, 0.5], [0.2, 0.3], [0.8, 0.7]];
    for (const [fx, fy] of pts) {
      const t = document.elementFromPoint(cr.left + cr.width * fx, cr.top + cr.height * fy);
      if (!t) return false;
      const host = c.getRootNode && c.getRootNode().host;
      if (t === c || c.contains(t) || t.contains(c) || (host && (host === t || host.contains(t)))) return false;
    }
    return true;
  };
  // returns {el, rect} or null. With scroll=true, scrolls the element into view first (also inside scroll containers).
  const locate = (s, scroll) => {
    const el0 = findEl(s);
    if (!el0) return null;
    const cands = [el0];
    if (el0.tagName === "INPUT" && ["checkbox", "radio", "file"].includes(el0.type)) {
      if (el0.labels && el0.labels[0]) cands.push(el0.labels[0]);
      const lab = el0.closest && el0.closest("label");
      if (lab) cands.push(lab);
    }
    for (const c of cands) {
      if (!okStyle(c)) continue;
      let r = c.getBoundingClientRect();
      if (r.width < 4 || r.height < 4) continue;
      const need = (rr) => 0.5 * Math.min(rr.width * rr.height, 0.4 * window.innerWidth * window.innerHeight);
      let cr = clipRect(c, r);
      if (scroll && cr.width * cr.height < need(r)) {
        try { c.scrollIntoView({ block: "center", inline: "center", behavior: "instant" }); } catch (e) {}
        r = c.getBoundingClientRect();
        cr = clipRect(c, r);
      }
      if (cr.width < 4 || cr.height < 4 || cr.width * cr.height < need(r)) continue;
      if (covered(c, cr)) continue;
      return { el: c, orig: el0, rect: cr };
    }
    return null;
  };

  // ----- UI -----
  const host = document.createElement("div");
  host.setAttribute("data-pa-tour", "1");
  host.style.cssText = "all:initial;position:fixed;inset:0;z-index:2147483647;pointer-events:none;";
  host.dir = o && o.rtl ? "rtl" : "ltr";
  const root = host.attachShadow({ mode: "open" });
  const style = document.createElement("style");
  style.textContent = `
    :host{all:initial}
    *{box-sizing:border-box;font-family:system-ui,-apple-system,"Segoe UI",Roboto,sans-serif}
    .dim{position:fixed;inset:0;background:rgba(15,23,42,.55);pointer-events:none}
    .ring{position:fixed;border:3px solid #3b82f6;border-radius:8px;box-shadow:0 0 0 9999px rgba(15,23,42,.55);pointer-events:none;transition:all .18s ease}
    .tip{position:fixed;width:min(320px,calc(100vw - 24px));background:#fff;color:#1f2328;border-radius:12px;padding:12px 14px;box-shadow:0 10px 30px rgba(0,0,0,.35);pointer-events:auto;font-size:14px;line-height:1.45}
    .top{display:flex;justify-content:space-between;align-items:center;gap:8px;margin-bottom:4px}
    .count{font-size:12px;color:#57606a}
    .x{all:unset;cursor:pointer;font-size:18px;line-height:1;padding:2px 6px;border-radius:6px;color:#57606a}
    .x:hover{background:#eef2f7}
    h3{margin:0 0 4px;font-size:15px}
    p{margin:0 0 8px;white-space:pre-wrap;overflow-wrap:anywhere}
    .hintc{font-size:12px;color:#0b57d0;margin-bottom:8px}
    .btns{display:flex;justify-content:flex-end;gap:8px}
    button.b{all:unset;cursor:pointer;padding:6px 14px;border-radius:8px;font-size:13px;font-weight:600;background:#eef2f7;color:#1f2328}
    button.b.next{background:#0b57d0;color:#fff}
    button.b:focus-visible{outline:2px solid #0b57d0;outline-offset:2px}
    [hidden]{display:none!important}
    @media (prefers-color-scheme:dark){.tip{background:#1e2530;color:#e6edf3}.count,.x{color:#9aa7b4}.x:hover{background:#2b3544}.hintc{color:#7db4ff}button.b{background:#2b3544;color:#e6edf3}button.b.next{background:#2f6fe0;color:#fff}}
  `;
  const mk = (tag, cls, text) => { const e = document.createElement(tag); if (cls) e.className = cls; if (text != null) e.textContent = text; return e; };
  const dim = mk("div", "dim"), ring = mk("div", "ring"), tip = mk("div", "tip");
  tip.setAttribute("role", "dialog");
  tip.setAttribute("aria-live", "polite");
  const count = mk("span", "count"), xBtn = mk("button", "x", "×");
  xBtn.setAttribute("aria-label", L.exit);
  xBtn.title = L.exit;
  const top = mk("div", "top");
  top.append(count, xBtn);
  const h = mk("h3"), p = mk("p"), hint = mk("div", "hintc");
  const back = mk("button", "b back", L.back), next = mk("button", "b next");
  const btns = mk("div", "btns");
  btns.append(back, next);
  tip.append(top, h, p, hint, btns);
  root.append(style, dim, ring, tip);
  document.documentElement.appendChild(host);

  let i = 0, waiting = false, clickEls = [], clickFn = null, timer = null, shown = 0;
  const detach = () => { if (clickFn) clickEls.forEach((e) => e.removeEventListener("click", clickFn, true)); clickEls = []; clickFn = null; };

  function position() {
    if (waiting || !steps[i]) return;
    const s = steps[i];
    const loc = s.id != null ? locate(s, false) : null;
    const vw = window.innerWidth, vh = window.innerHeight, pad = 6, gap = 12;
    const tw = tip.offsetWidth || 320, th = tip.offsetHeight || 140;
    if (!loc) {
      ring.hidden = true; dim.hidden = false;
      tip.style.left = Math.max(12, (vw - tw) / 2) + "px";
      tip.style.top = Math.max(12, (vh - th) / 2) + "px";
      return;
    }
    const r = loc.rect;
    dim.hidden = true; ring.hidden = false;
    ring.style.left = r.left - pad + "px"; ring.style.top = r.top - pad + "px";
    ring.style.width = r.width + pad * 2 + "px"; ring.style.height = r.height + pad * 2 + "px";
    let ty = r.bottom + pad + gap;
    if (ty + th > vh - 8) ty = r.top - pad - gap - th;
    if (ty < 8) ty = Math.min(vh - th - 8, Math.max(8, r.bottom + pad + gap)); // no room above or below: overlap rather than leave the screen
    let tx = r.left + r.width / 2 - tw / 2;
    tx = Math.max(12, Math.min(vw - tw - 12, tx));
    tip.style.left = tx + "px"; tip.style.top = ty + "px";
  }

  function paint(loc) {
    detach();
    waiting = false;
    const s = steps[i], last = i === steps.length - 1;
    count.textContent = L.step.replace("{n}", ((o && o.offset) || 0) + i + 1).replace("{total}", ((o && o.offset) || 0) + steps.length + ((o && o.rest) || 0));
    h.textContent = s.title || ""; h.hidden = !s.title;
    p.textContent = s.text || "";
    const wantsClick = !!(s.click && loc);
    hint.textContent = wantsClick ? L.click : ""; hint.hidden = !wantsClick;
    back.hidden = i === 0;
    next.textContent = last && o && o.finished ? L.done : L.next;
    btns.hidden = false; xBtn.hidden = false;
    if (wantsClick) {
      clickEls = loc.orig === loc.el ? [loc.el] : [loc.el, loc.orig];
      clickFn = () => { setTimeout(() => advance("click"), 0); }; // let the page handle the click first
      clickEls.forEach((e) => e.addEventListener("click", clickFn, true));
    }
    position();
    try { next.focus({ preventScroll: true }); } catch (e) {}
  }

  // Show the first step at/after (dir>0) or at/before (dir<0) index k whose target can really be seen; drop the others.
  function show(k, dir) {
    while (k >= 0 && k < steps.length) {
      const s = steps[k];
      const loc = s.id == null ? null : locate(s, true);
      if (s.id == null || loc) { i = k; paint(loc); return true; }
      steps.splice(k, 1);
      if (k < i) i--; // keep i pointing at the step currently on screen
      if (dir < 0) k--;
    }
    return false;
  }

  function advance(reason) {
    detach();
    if (o && o.finished && reason === "next") { send({ type: "advance", reason, last: true, shown }); return destroy(); }
    // more to show: wait for the extension to prepare the next part
    waiting = true;
    ring.hidden = true; dim.hidden = true;
    count.textContent = ""; h.hidden = true; hint.hidden = true; btns.hidden = true; xBtn.hidden = true;
    p.textContent = L.wait;
    tip.style.left = Math.max(12, window.innerWidth - tip.offsetWidth - 16) + "px";
    tip.style.top = "16px";
    send({ type: "advance", reason, shown });
  }

  const onKey = (e) => { if (e.key === "Escape") { send({ type: "exit" }); destroy(); } };
  function destroy() {
    detach();
    clearInterval(timer);
    window.removeEventListener("keydown", onKey, true);
    window.removeEventListener("resize", position);
    window.removeEventListener("scroll", position, true);
    host.remove();
    if (window.__paTour && window.__paTour.host === host) window.__paTour = null;
  }
  next.onclick = () => { if (!show(i + 1, 1)) advance("next"); };
  back.onclick = () => { show(i - 1, -1); };
  xBtn.onclick = () => { send({ type: "exit" }); destroy(); };
  window.addEventListener("keydown", onKey, true);
  window.addEventListener("resize", position);
  window.addEventListener("scroll", position, true);
  timer = setInterval(position, 250);
  window.__paTour = { destroy, host };
  // Drop steps whose target cannot really be shown right now, up front, so "Step n of N" is honest and nothing points at empty space.
  steps = steps.filter((s) => s.id == null || locate(s, true));
  shown = steps.length;
  if (!show(0, 1)) advance("empty");
  return true;
}

function tourClearFn() {
  if (window.__paTour) window.__paTour.destroy();
  document.querySelectorAll("[data-pa-tour]").forEach((e) => e.remove());
  return true;
}

// ======================= Skills: recording, replay, direct requests =======================
// Everything below is injected into the page with chrome.scripting.executeScript, so each function is self-contained.

// Records what the user does on the page (clicks, typed values, selects, keys) plus mutating network requests.
// cmd: { cmd: "start", resume?: boolean } | { cmd: "dump" } | { cmd: "stop" }
function recorderFn(c) {
  const KEY = "__paRecState";
  const W = window;
  const clean = (s) => (s || "").replace(/\s+/g, " ").trim();
  const txt = (el) => clean(el.innerText !== undefined ? el.innerText : el.textContent);
  const okId = (id) => !!id && id.length < 60 && !/\d{3,}|^:|[:\s]/.test(id);
  const labelOf = (el) => {
    let l = el.getAttribute("aria-label");
    const lb = el.getAttribute("aria-labelledby");
    if (!l && lb) l = lb.split(" ").map((i) => { const n = document.getElementById(i); return n ? txt(n) : ""; }).join(" ");
    if (!l && el.labels && el.labels[0]) l = txt(el.labels[0]);
    if (!l && ["INPUT", "TEXTAREA", "SELECT"].includes(el.tagName)) l = el.placeholder || el.name || (okId(el.id) ? el.id : "");
    if (!l && el.tagName === "INPUT" && ["submit", "button"].includes(el.type)) l = el.value;
    if (!l) { const img = el.querySelector && el.querySelector("img[alt]"); l = txt(el) || el.title || el.alt || (img && img.alt) || ""; }
    return clean(l).slice(0, 80);
  };
  const cssPath = (el) => {
    const parts = [];
    let n = el;
    for (let i = 0; n && n.nodeType === 1 && i < 6; i++, n = n.parentElement) {
      if (okId(n.id)) { parts.unshift("#" + String(n.id).replace(/[^\w-]/g, "\\$&")); break; }
      let s = n.tagName.toLowerCase();
      const p = n.parentElement;
      if (p) { const same = [...p.children].filter((x) => x.tagName === n.tagName); if (same.length > 1) s += ":nth-of-type(" + (same.indexOf(n) + 1) + ")"; }
      parts.unshift(s);
    }
    return parts.join(" > ");
  };
  // The text next to / above a control (its visible caption), for naming an input when it has no real label.
  const fieldLabel = (el) => {
    let n = el, fl = "";
    for (let k = 0; k < 4 && n && !fl; k++, n = n.parentElement) {
      const lb = n.querySelector && n.querySelector("label");
      if (lb && !lb.contains(el)) { fl = clean(lb.innerText || lb.textContent); break; }
      const ps = n.previousElementSibling;
      if (ps && !/^(INPUT|SELECT|TEXTAREA|BUTTON)$/.test(ps.tagName)) { const t = clean(ps.innerText || ps.textContent); if (t && t.length < 40) fl = t; }
    }
    return fl.slice(0, 40);
  };
  const describe = (el) => {
    const label = labelOf(el);
    const row = el.closest("tr,li,[role=row],[role=listitem],[role=dialog],dialog,form");
    let ctx = row && row !== el ? txt(row).slice(0, 50) : "";
    if (ctx === label) ctx = "";
    let nth = 0, same = 1;
    try {
      const all = [...document.querySelectorAll(el.tagName.toLowerCase())].filter((x) => labelOf(x) === label);
      same = all.length; nth = Math.max(0, all.indexOf(el));
    } catch (e) {}
    const d = {
      tag: el.tagName.toLowerCase(), role: el.getAttribute("role") || "", type: el.tagName === "INPUT" ? el.type : "",
      id: okId(el.id) ? el.id : "", name: el.getAttribute("name") || "",
      testid: el.getAttribute("data-testid") || el.getAttribute("data-test") || el.getAttribute("data-qa") || el.getAttribute("data-cy") || "",
      label, placeholder: el.getAttribute("placeholder") || "", ctx, nth, same, css: cssPath(el),
    };
    if (/^(INPUT|SELECT|TEXTAREA)$/.test(el.tagName) && (!label || label === d.id || label === d.name)) { const fl = fieldLabel(el); if (fl) d.fl = fl; }
    if (el.tagName === "A" && el.getAttribute("href")) { try { d.href = new URL(el.href).pathname.slice(0, 100); } catch (e) {} }
    return d;
  };
  const save = () => {
    try {
      const R = W.__paRec;
      sessionStorage.setItem(KEY, JSON.stringify({ steps: R.steps, reqs: R.reqs.slice(-40) }));
    } catch (e) {}
  };

  if (c.cmd === "dump" || c.cmd === "stop") {
    const R = W.__paRec;
    let out;
    if (R) out = { steps: R.steps, reqs: R.reqs, url: location.href, title: document.title };
    else { try { out = Object.assign({ url: location.href, title: document.title }, JSON.parse(sessionStorage.getItem(KEY) || "{}")); } catch (e) { out = { steps: [], reqs: [], url: location.href, title: document.title }; } }
    if (c.cmd === "stop") {
      if (R) { R.on = false; (R.off || []).forEach((f) => { try { f(); } catch (e) {} }); }
      try { sessionStorage.removeItem(KEY); } catch (e) {}
    }
    return out;
  }

  // ---- start / resume
  if (W.__paRec && W.__paRec.on) return { ok: true, already: true };
  let prev = null;
  if (c.resume) { try { prev = JSON.parse(sessionStorage.getItem(KEY) || "null"); } catch (e) {} }
  const R = (W.__paRec = { on: true, steps: (prev && prev.steps) || [], reqs: (prev && prev.reqs) || [], muts: 0, off: [] });
  const mo = new MutationObserver((list) => { R.muts += list.length; });
  mo.observe(document.documentElement, { subtree: true, childList: true, characterData: true });
  R.off.push(() => mo.disconnect());

  const push = (st) => {
    st.at = Date.now(); st.url = location.href;
    const m0 = R.muts, u0 = location.href;
    R.steps.push(st);
    if (R.steps.length > 1500) R.steps.shift();
    setTimeout(() => { st.effect = R.muts - m0 > 2 || location.href !== u0; save(); }, 1200);
    save();
  };
  const isTextual = (el) => el.tagName === "TEXTAREA" || (el.tagName === "INPUT" && !["checkbox", "radio", "button", "submit", "reset", "file", "image", "range", "color"].includes(el.type));
  const flush = (el) => {
    if (!el || !(isTextual(el))) return;
    const sensitive = el.type === "password";
    const val = sensitive ? "(secret)" : el.value;
    if (el.__paLast === val || (el.__paLast === undefined && !el.value)) { el.__paLast = val; return; }
    el.__paLast = val;
    const d = describe(el);
    const last = R.steps[R.steps.length - 1];
    if (last && last.t === "type" && last.d.css === d.css && last.d.label === d.label) { last.value = sensitive ? "" : el.value; last.at = Date.now(); save(); return; }
    const stp = { t: "type", d, value: sensitive ? "" : el.value, sensitive };
    if (el.readOnly) stp.ro = true;
    push(stp);
  };
  // Read-only fields (date pickers): the user cannot type, a widget fills them after a click. Watch for that.
  const DATEISH = /\b\d{1,4}[\/\-.\s][A-Za-z0-9]{1,9}[\/\-.\s]\d{1,4}\b/;
  const seen = new WeakMap();
  const scanRO = (init) => {
    document.querySelectorAll("input[readonly]").forEach((inp) => {
      if (!isTextual(inp) || inp.type === "password") return;
      const v = inp.value, p = seen.get(inp);
      seen.set(inp, v);
      if (init || p === undefined || p === v || !v || inp.__paLast === v) return;
      if (!(DATEISH.test(v) || inp.type === "date")) return;
      inp.__paLast = v;
      push({ t: "type", d: describe(inp), value: v, sensitive: false, ro: true });
    });
  };
  scanRO(true);
  const scanSoon = () => { setTimeout(scanRO, 150); setTimeout(scanRO, 800); };
  const onChange = (e) => {
    const el = e.target;
    if (!el || !el.tagName) return;
    if (el.tagName === "SELECT") {
      const o = el.options[el.selectedIndex];
      const choices = [...el.options].map((x) => clean(x.text)).filter(Boolean).slice(0, 60);
      if (el.multiple) {
        const picked = [...el.selectedOptions].map((x) => clean(x.text)).filter(Boolean);
        const d0 = describe(el), last = R.steps[R.steps.length - 1];
        if (last && last.t === "select" && last.multi && last.d.css === d0.css) { last.value = picked.join("; "); last.at = Date.now(); save(); }
        else push({ t: "select", multi: true, d: d0, value: picked.join("; "), choices });
      } else push({ t: "select", d: describe(el), value: o ? clean(o.text) : el.value, choices });
    } else flush(el);
  };
  const CLICKABLE = 'a,button,input,select,textarea,summary,label,[role=button],[role=link],[role=tab],[role=menuitem],[role=option],[role=checkbox],[role=radio],[role=switch],[role=treeitem],[role=gridcell],[onclick],li,[tabindex]';
  const onClick = (e) => {
    if (!e.isTrusted && !W.__paAllowSynthetic) return;
    flush(document.activeElement);
    let el = e.target && e.target.closest ? e.target.closest(CLICKABLE) : null;
    if (!el) el = e.target;
    if (!el || !el.tagName) return;
    if (isTextual(el) || el.tagName === "SELECT" || el.isContentEditable) return; // focusing a field is not a step; its value is
    const last = R.steps[R.steps.length - 1];
    if (el.tagName === "INPUT" && last && last.t === "click" && Date.now() - last.at < 150 && last.d.tag === "label") return;
    const stp = { t: "click", d: describe(el) };
    // checkbox / radio (the control itself, or its label)
    {
      const ctl = el.tagName === "INPUT" ? el : el.tagName === "LABEL" ? el.control : null;
      if (ctl && (ctl.type === "checkbox" || ctl.type === "radio")) {
        stp.kind = ctl.type;
        if (ctl.type === "checkbox") stp.checked = el.tagName === "INPUT" ? ctl.checked : !ctl.checked;
        else {
          stp.checked = true;
          const grp = ctl.name ? [...document.querySelectorAll('input[type=radio]')].filter((x) => x.name === ctl.name) : [ctl];
          stp.choices = grp.map((x) => clean((x.labels && x.labels[0] ? (x.labels[0].innerText !== undefined ? x.labels[0].innerText : x.labels[0].textContent) : "") || x.getAttribute("aria-label") || x.value)).filter(Boolean).slice(0, 40);
          const fs = ctl.closest("fieldset"); const lg = fs && fs.querySelector("legend");
          if (lg) stp.fl = clean(lg.textContent).slice(0, 40);
        }
      }
    }
    // A choice from a dropdown / list / menu (custom widgets, not <select>): remember it so it can become an input.
    const box = el.closest('[role=listbox],[role=menu],[role=tree],[role=combobox],[class*=dropdown i],[class*=select i],[class*=autocomplete i],[class*=suggest i],[class*=popup i],[class*=popover i],[class*=overlay i],[class*=options i],[class*=menu i],[class*=chosen i],[class*=choices i],[class*=listbox i]');
    if ((el.matches('[role=option],[role=menuitem],[role=treeitem],li,option') || (box && box !== el)) && box && !el.closest("table,[role=grid],[class*=calendar i],[class*=datepicker i]") && stp.d.label && stp.d.label.length < 70) {
      const prevC = R.steps[R.steps.length - 1];
      if (prevC && prevC.t === "click" && Date.now() - prevC.at < 20000) {
        stp.pick = true;
        try { stp.box = cssPath(box); } catch (e) {}
        const sibs = el.parentElement ? [...el.parentElement.children].filter((x) => x.tagName === el.tagName) : [];
        stp.choices = (sibs.length > 1 ? sibs : [...box.querySelectorAll(el.tagName)]).map((x) => clean(x.innerText !== undefined ? x.innerText : x.textContent)).filter((x) => x && x.length < 70).slice(0, 60);
      }
    }
    if (!stp.fl) { const fl = fieldLabel(el); if (fl) stp.fl = fl; }
    const plain = !txt(el) && !/^(INPUT|TEXTAREA|SELECT)$/.test(el.tagName) || /^(IMG|SVG|I)$/i.test(el.tagName) || (el.tagName === "INPUT" && el.type === "image");
    if (plain) {
      stp.icon = true;
      let p = el.parentElement;
      for (let k = 0; k < 3 && p; k++, p = p.parentElement) {
        const n = p.querySelector("input:not([type=hidden]):not([type=checkbox]):not([type=radio]):not([type=submit]):not([type=button]):not([type=image])");
        if (n && n !== el) { stp.near = describe(n); break; }
      }
    }
    push(stp);
    scanSoon();
  };
  const KEYS = ["Enter", "Escape", "ArrowDown", "ArrowUp"];
  const onKey = (e) => {
    if ((!e.isTrusted && !W.__paAllowSynthetic) || !KEYS.includes(e.key)) return;
    const el = e.target;
    if (!el || !el.tagName) return;
    if (e.key === "Enter" && !(el.tagName === "INPUT" || el.getAttribute("role") === "combobox" || el.getAttribute("role") === "textbox")) return;
    flush(el);
    push({ t: "key", key: e.key, d: describe(el) });
  };
  const onBlurCE = (e) => {
    const el = e.target;
    if (el && el.isContentEditable && el.tagName) {
      const v = clean(el.textContent);
      if (el.__paLast !== v && v) { el.__paLast = v; push({ t: "type", d: describe(el), value: v, sensitive: false }); }
    }
  };
  document.addEventListener("change", onChange, true);
  document.addEventListener("click", onClick, true);
  document.addEventListener("keydown", onKey, true);
  document.addEventListener("focusout", onBlurCE, true);
  R.off.push(() => {
    document.removeEventListener("change", onChange, true);
    document.removeEventListener("click", onClick, true);
    document.removeEventListener("keydown", onKey, true);
    document.removeEventListener("focusout", onBlurCE, true);
  });

  // ---- network: remember requests that change data (POST/PUT/PATCH/DELETE)
  const keepHeaders = (h) => {
    const out = {};
    try {
      const put = (k, v) => { k = String(k).toLowerCase(); if (k === "content-type" || k === "accept" || k.startsWith("x-")) out[k] = String(v); };
      if (!h) return out;
      if (typeof h.forEach === "function" && !Array.isArray(h)) h.forEach((v, k) => put(k, v));
      else if (Array.isArray(h)) h.forEach(([k, v]) => put(k, v));
      else Object.keys(h).forEach((k) => put(k, h[k]));
    } catch (e) {}
    return out;
  };
  const note = (method, url, headers, body) => {
    method = String(method || "GET").toUpperCase();
    if (["GET", "HEAD", "OPTIONS"].includes(method) || !R.on) return null;
    let abs = url;
    try { abs = new URL(url, location.href).href; } catch (e) {}
    let b = null;
    if (typeof body === "string") b = body.slice(0, 20000);
    else if (body && typeof body.toString === "function" && body instanceof URLSearchParams) b = body.toString();
    const r = { method, url: abs, headers: keepHeaders(headers), body: b, opaque: body != null && b == null, at: Date.now(), status: 0 };
    R.reqs.push(r);
    if (R.reqs.length > 80) R.reqs.shift();
    save();
    return r;
  };
  if (!W.__paNetPatched) {
    W.__paNetPatched = true;
    const of = W.fetch;
    W.fetch = function (input, init) {
      let rec = null;
      try {
        const f = W.__paNoteFetch;
        if (f) {
          const isReq = typeof Request !== "undefined" && input instanceof Request;
          rec = f(isReq ? input.method : init && init.method, isReq ? input.url : String(input), (init && init.headers) || (isReq ? input.headers : null), init && init.body);
        }
      } catch (e) {}
      const p = of.apply(this, arguments);
      if (rec) p.then((resp) => { rec.status = resp.status; }, () => {});
      return p;
    };
    const XO = XMLHttpRequest.prototype.open, XS = XMLHttpRequest.prototype.send, XH = XMLHttpRequest.prototype.setRequestHeader;
    XMLHttpRequest.prototype.open = function (m, u) { this.__pa = { m, u, h: {} }; return XO.apply(this, arguments); };
    XMLHttpRequest.prototype.setRequestHeader = function (k, v) { if (this.__pa) this.__pa.h[k] = v; return XH.apply(this, arguments); };
    XMLHttpRequest.prototype.send = function (body) {
      try {
        const f = W.__paNoteFetch;
        const rec = this.__pa && f ? f(this.__pa.m, this.__pa.u, this.__pa.h, body) : null;
        if (rec) this.addEventListener("loadend", () => { rec.status = this.status; });
      } catch (e) {}
      return XS.apply(this, arguments);
    };
  }
  W.__paNoteFetch = note;
  return { ok: true };
}

// Finds one recorded step's element on the current page and performs it. step = { t, d, value?, key? }.
async function replayStepFn(step, value, timeoutMs) {
  const clean = (s) => (s || "").replace(/\s+/g, " ").trim();
  const norm = (s) => clean(s).toLowerCase();
  const txt = (el) => clean(el.innerText !== undefined ? el.innerText : el.textContent);
  const okId = (id) => !!id && id.length < 60 && !/\d{3,}|^:|[:\s]/.test(id);
  const labelOf = (el) => {
    let l = el.getAttribute("aria-label");
    const lb = el.getAttribute("aria-labelledby");
    if (!l && lb) l = lb.split(" ").map((i) => { const n = document.getElementById(i); return n ? txt(n) : ""; }).join(" ");
    if (!l && el.labels && el.labels[0]) l = txt(el.labels[0]);
    if (!l && ["INPUT", "TEXTAREA", "SELECT"].includes(el.tagName)) l = el.placeholder || el.name || (okId(el.id) ? el.id : "");
    if (!l && el.tagName === "INPUT" && ["submit", "button"].includes(el.type)) l = el.value;
    if (!l) { const img = el.querySelector && el.querySelector("img[alt]"); l = txt(el) || el.title || el.alt || (img && img.alt) || ""; }
    return clean(l).slice(0, 80);
  };
  const visible = (el) => {
    const r = el.getBoundingClientRect();
    if (r.width < 2 || r.height < 2) return false;
    const cs = getComputedStyle(el);
    return cs.visibility !== "hidden" && cs.display !== "none";
  };
  const deepQ = (sel) => {
    const out = [...document.querySelectorAll(sel)];
    const stack = [document];
    while (stack.length) {
      const r = stack.pop();
      r.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) { e.shadowRoot.querySelectorAll(sel).forEach((x) => out.push(x)); stack.push(e.shadowRoot); } });
    }
    return out;
  };
  const d = step.d || {};
  const find = () => {
    const attr = (n, v) => '[' + n + '="' + String(v).replace(/"/g, '\\"') + '"]';
    // exact identifiers first
    if (d.testid) {
      const q = deepQ(["data-testid", "data-test", "data-qa", "data-cy"].map((n) => attr(n, d.testid)).join(",")).filter(visible);
      if (q.length === 1) return q[0];
    }
    if (d.id) { const e = document.getElementById(d.id); if (e && visible(e)) return e; }
    // otherwise score the candidates of the same tag/role
    const sel = [d.tag || "*"].concat(d.role ? ['[role="' + d.role + '"]'] : []).join(",");
    const nl = norm(d.label), nc = norm(d.ctx).slice(0, 24);
    let best = [], bs = 0;
    for (const el of deepQ(sel).slice(0, 4000)) {
      let s = 0;
      if (d.testid && (el.getAttribute("data-testid") === d.testid || el.getAttribute("data-test") === d.testid)) s += 10;
      if (d.name && el.getAttribute("name") === d.name) s += 5;
      if (d.id && el.id === d.id) s += 8;
      const l = nl ? norm(labelOf(el)) : "";
      if (nl && l === nl) s += 6; else if (nl && l && (l.startsWith(nl) || nl.startsWith(l))) s += 3;
      if (d.placeholder && el.getAttribute("placeholder") === d.placeholder) s += 2;
      if (d.href && el.tagName === "A") { try { if (new URL(el.href).pathname === d.href) s += 4; } catch (e) {} }
      if (s < 4) continue;
      if (d.type && el.type === d.type) s += 1;
      if (nc) { const row = el.closest("tr,li,[role=row],[role=listitem],[role=dialog],dialog,form"); if (row && norm(txt(row)).includes(nc)) s += 3; }
      if (!visible(el)) continue;
      if (s > bs) { bs = s; best = [el]; } else if (s === bs) best.push(el);
    }
    if (best.length) return best[Math.min(d.nth || 0, best.length - 1)] || best[0];
    if (d.css) { try { const e = document.querySelector(d.css); if (e && visible(e) && (step.loose || !nl || norm(labelOf(e)) === nl || !labelOf(e))) return e; } catch (e) {} }
    return null;
  };
  // Dynamic target: the element is named after a parameter value (a typeahead suggestion, a row) rather than the recorded label.
  const findDyn = () => {
    const toks = norm(d.dynValue).split(" ").filter(Boolean);
    if (!toks.length) return null;
    const want = norm(d.dynValue);
    let best = null, bs = -1e9;
    for (const el of deepQ('[role=option],[role=menuitem],[role=treeitem],[role=row],[role=link],[role=button],li,a,button,div,span,td,tr,p,label,h1,h2,h3,h4').slice(0, 8000)) {
      const text = norm(txt(el));
      if (!text || text.length > 220 || !toks.every((t) => text.includes(t))) continue;
      if (!visible(el)) continue;
      let s = 10 - Math.max(0, text.length - want.length) / 15;
      if (text === want) s += 4;
      if (el.closest('[role=listbox],[role=menu],[role=tree],[class*="suggest" i],[class*="option" i],[class*="dropdown" i],[class*="autocomplete" i],[class*="popup" i],[class*="result" i]')) s += 3;
      if (el.tagName.toLowerCase() === d.tag) s += 2;
      if (d.role && el.getAttribute("role") === d.role) s += 2;
      if (d.ctx) { const row = el.closest("tr,li,[role=row],[role=listitem],[role=dialog],dialog,form"); if (row && norm(txt(row)).includes(norm(d.ctx).slice(0, 24))) s += 1; }
      if (s > bs) { bs = s; best = el; }
    }
    return best;
  };
  const t0 = Date.now(), limit = timeoutMs || 6000;
  const bare = step.t === "key" && !d.tag && !d.css;
  // A day in an open calendar popup: the cell whose whole text is the day number, not a greyed-out day of another month.
  const findDay = () => {
    const want = String(d.label);
    const cells = deepQ("td,th,button,a,span,div,li,[role=gridcell]").filter((e) => clean(e.textContent) === want && ![...e.children].some((c) => clean(c.textContent) === want)).filter(visible);
    const good = cells.filter((e) => {
      let n = e, cls = "";
      for (let k = 0; k < 2 && n; k++, n = n.parentElement) cls += " " + (n.className && n.className.baseVal !== undefined ? n.className.baseVal : n.className) + " " + (n.getAttribute("aria-disabled") === "true" ? "disabled" : "");
      return !/(^|[\s_-])(old|new|disabled|outside|other|muted|off|prev|next|adjacent)([\s_-]|$)|other-?month|out-of-?month/i.test(cls) && !e.disabled;
    });
    const inCal = (arr) => arr.filter((e) => e.closest("table,[role=grid],[class*=calendar i],[class*=datepicker i],[class*=picker i],[class*=ui-datepicker i]"));
    return inCal(good)[0] || inCal(cells)[0] || good[0] || null;
  };
  const look = () => (bare ? document.activeElement || document.body : d.daySel ? findDay() || find() : d.dynValue ? findDyn() : find());
  let el = look();
  while (!el && Date.now() - t0 < limit) { await new Promise((r) => setTimeout(r, 200)); el = look(); }
  if (!el) return { ok: false, msg: 'Could not find "' + (d.label || d.css || d.tag) + '" (' + step.t + ") on this page." };
  if (el.scrollIntoView) { try { el.scrollIntoView({ block: "center", inline: "center" }); } catch (e) {} }
  const press = (t) => {
    try {
      const r = t.getBoundingClientRect();
      const o = { bubbles: true, cancelable: true, view: window, clientX: r.left + r.width / 2, clientY: r.top + r.height / 2, button: 0 };
      if (typeof PointerEvent === "function") t.dispatchEvent(new PointerEvent("pointerdown", { ...o, pointerType: "mouse", isPrimary: true }));
      t.dispatchEvent(new MouseEvent("mousedown", o));
      if (typeof PointerEvent === "function") t.dispatchEvent(new PointerEvent("pointerup", { ...o, pointerType: "mouse", isPrimary: true }));
      t.dispatchEvent(new MouseEvent("mouseup", o));
    } catch (e) {}
    t.click();
  };
  const fire = (e) => { e.dispatchEvent(new Event("input", { bubbles: true })); e.dispatchEvent(new Event("change", { bubbles: true })); };
  try {
    if (step.t === "click") {
      if (el.disabled) return { ok: false, msg: '"' + (d.label || "element") + '" is disabled.' };
      if (step.kind === "checkbox" && step.want != null) {
        const ctl = el.tagName === "INPUT" ? el : el.control || el.querySelector("input[type=checkbox]");
        if (ctl && ctl.checked === !!step.want) return { ok: true, msg: "Already " + (step.want ? "checked" : "unchecked") + "." };
      }
      if (el.focus) el.focus({ preventScroll: true });
      if (!(step.icon && step.near)) { press(el); return { ok: true, msg: "Clicked." }; }
      // A calendar icon: the handler may sit on a neighbour. Click it; if the page did not react, try the next candidate (never re-click the same one).
      let muts = 0;
      const mo = new MutationObserver((l) => { for (const m of l) { if (m.target === el || /^(INPUT|TEXTAREA)$/.test(m.target.tagName)) continue; if (m.type === "childList" || /^(style|class|hidden|aria-expanded)$/.test(m.attributeName)) muts++; } });
      mo.observe(document.documentElement, { subtree: true, childList: true, attributes: true });
      const wait = (ms) => new Promise((r) => setTimeout(r, ms));
      const tried = new Set([el]);
      const attempt = async (t) => {
        if (!t || tried.has(t) || !t.tagName) return false;
        tried.add(t);
        if (t.focus) { try { t.focus({ preventScroll: true }); } catch (e) {} }
        press(t);
        await wait(300);
        return muts > 0;
      };
      press(el);
      await wait(300);
      let opened = muts > 0, how = "icon";
      if (!opened) {
        const r = el.getBoundingClientRect();
        let hit = null;
        try { hit = document.elementFromPoint(r.left + r.width / 2, r.top + r.height / 2); } catch (e) {}
        if (await attempt(hit)) { opened = true; how = "element at that point"; }
      }
      if (!opened) {
        let p = el.parentElement;
        for (let k = 0; k < 2 && p && !opened; k++, p = p.parentElement) if (await attempt(p)) { opened = true; how = "container"; }
      }
      if (!opened) {
        const nd = step.near;
        let near = null;
        if (nd.id) near = document.getElementById(nd.id);
        if (!near && nd.name) near = document.querySelector('input[name="' + String(nd.name).replace(/"/g, '\\"') + '"]');
        if (!near && nd.css) { try { near = document.querySelector(nd.css); } catch (e) {} }
        if (!near) { let p = el.parentElement; for (let k = 0; k < 3 && p && !near; k++, p = p.parentElement) near = p.querySelector("input:not([type=hidden]):not([type=checkbox]):not([type=radio]):not([type=submit]):not([type=button]):not([type=image])"); }
        if (near && await attempt(near)) { opened = true; how = "the date field"; }
      }
      mo.disconnect();
      return { ok: true, msg: opened ? "Clicked (" + how + ")." : "Clicked, no visible reaction." };
    }
    if (step.t === "type") {
      if (el.focus) el.focus();
      const v = value == null ? step.value || "" : String(value);
      const isTA = el instanceof HTMLTextAreaElement;
      if (isTA || el instanceof HTMLInputElement) {
        const proto = isTA ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
        Object.getOwnPropertyDescriptor(proto, "value").set.call(el, v);
        fire(el);
        if (el.value !== v) return { ok: false, msg: "The field did not accept the value." };
        const lastCh = v.slice(-1) || " ";
        for (const ty of ["keydown", "keypress", "keyup"]) el.dispatchEvent(new KeyboardEvent(ty, { key: lastCh, bubbles: true, cancelable: true }));
      } else if (el.isContentEditable) {
        document.execCommand("selectAll", false);
        document.execCommand("insertText", false, v);
      } else return { ok: false, msg: "Not a text field." };
      if (step.ro && el.blur) { try { el.blur(); } catch (e) {} }
      return { ok: true, msg: "Typed." };
    }
    if (step.t === "select") {
      if (el.tagName !== "SELECT") return { ok: false, msg: "Not a <select>." };
      if (step.multi || el.multiple) {
        const wants = String(value == null ? step.value : value).split(/\s*(?:;|\n|\|)\s*/).map(norm).filter(Boolean);
        let o2 = [...el.options];
        const findAll = () => wants.map((w) => o2.find((o) => norm(o.text) === w || norm(o.value) === w) || o2.find((o) => norm(o.text).includes(w)));
        let hits = findAll();
        while (hits.some((h) => !h) && Date.now() - t0 < Math.max(limit, 3500)) { await new Promise((r) => setTimeout(r, 200)); o2 = [...el.options]; hits = findAll(); }
        if (hits.some((h) => !h)) return { ok: false, msg: 'No option "' + wants[hits.findIndex((h) => !h)] + '". Available: ' + o2.slice(0, 20).map((o) => clean(o.text)).join(" | ") };
        for (const o of o2) o.selected = hits.includes(o);
        fire(el);
        return { ok: true, msg: "Selected " + hits.length + "." };
      }
      const want = norm(value == null ? step.value : value);
      let opts = [...el.options];
      const pickOpt = () => opts.find((o) => norm(o.text) === want || norm(o.value) === want) || opts.find((o) => norm(o.text).includes(want));
      let opt = pickOpt();
      // dependent dropdowns fill their options after the previous choice: wait for it
      while (!opt && Date.now() - t0 < Math.max(limit, 3500)) { await new Promise((r) => setTimeout(r, 200)); opts = [...el.options]; opt = pickOpt(); }
      if (!opt) return { ok: false, msg: 'No option "' + want + '". Available: ' + opts.slice(0, 20).map((o) => clean(o.text)).join(" | ") };
      Object.getOwnPropertyDescriptor(HTMLSelectElement.prototype, "value").set.call(el, opt.value);
      fire(el);
      return { ok: true, msg: "Selected." };
    }
    if (step.t === "key") {
      if (el.focus) el.focus();
      for (const ty of ["keydown", "keypress", "keyup"]) el.dispatchEvent(new KeyboardEvent(ty, { key: step.key, code: step.key, bubbles: true, cancelable: true }));
      if (step.key === "Enter" && el.form && el.tagName === "INPUT" && el.form.requestSubmit) { try { el.form.requestSubmit(); } catch (e) {} }
      return { ok: true, msg: "Pressed " + step.key + "." };
    }
    return { ok: false, msg: "Unknown step type " + step.t };
  } catch (e) {
    return { ok: false, msg: String(e && e.message || e) };
  }
}

// A cheap fingerprint of the page, to check that a step really changed something. Besides the text and element count it includes
// state that does not show in the text: field values, checked boxes, open menus and dialogs.
function pageSigFn() {
  const t = (document.body ? document.body.innerText || document.body.textContent || "" : "").slice(0, 30000);
  let h = 5381;
  const mix = (str) => { for (let i = 0; i < str.length; i++) h = ((h << 5) + h + str.charCodeAt(i)) | 0; };
  mix(t);
  let st = "";
  try {
    document.querySelectorAll("input,select,textarea").forEach((e, i) => {
      if (i > 300 || e.type === "password") return;
      st += (e.type === "checkbox" || e.type === "radio" ? (e.checked ? "1" : "0") : e.tagName === "SELECT" ? e.selectedIndex : String(e.value).slice(0, 40)) + "|";
    });
    st += document.querySelectorAll('[aria-expanded="true"],[aria-selected="true"],[aria-checked="true"],dialog[open],[role=dialog],[role=listbox],[role=menu]').length;
  } catch (e) {}
  mix(st);
  return location.href + "|" + (document.body ? document.body.getElementsByTagName("*").length : 0) + "|" + h;
}

// Sends one recorded request again, from the page (same cookies / origin as the app).
async function fetchFn(req) {
  try {
    const r = await fetch(req.url, { method: req.method, headers: req.headers || {}, body: req.body == null ? undefined : req.body, credentials: "include" });
    const text = await r.text();
    return { ok: r.ok, status: r.status, text: text.slice(0, 800) };
  } catch (e) {
    return { ok: false, status: 0, text: String(e && e.message || e) };
  }
}


// For the debug report: what form controls and open popups the page has right now (no passwords / secrets).
function fieldsFn() {
  const clean = (x) => (x || "").replace(/\s+/g, " ").trim();
  const vis = (e) => { const r = e.getBoundingClientRect(); if (r.width < 2 || r.height < 2) return false; const cs = getComputedStyle(e); return cs.visibility !== "hidden" && cs.display !== "none"; };
  const lab = (e) => clean(e.getAttribute("aria-label") || (e.labels && e.labels[0] && e.labels[0].textContent) || e.placeholder || e.name || e.id || "").slice(0, 60);
  const out = { url: location.href, title: document.title, controls: [], popups: [] };
  for (const e of [...document.querySelectorAll("input,select,textarea")].slice(0, 400)) {
    if (e.type === "hidden" || !vis(e)) continue;
    const c = { tag: e.tagName.toLowerCase(), type: e.type || "", label: lab(e), name: e.name || "", id: e.id || "" };
    if (e.readOnly) c.readonly = true;
    if (e.disabled) c.disabled = true;
    if (e.tagName === "SELECT") { c.multiple = e.multiple; c.options = [...e.options].slice(0, 40).map((o) => (o.selected ? "[x] " : "") + clean(o.text)); }
    else if (e.type === "checkbox" || e.type === "radio") c.checked = e.checked;
    else if (e.type === "password" || /otp|cc-|card|cvv|ssn/i.test((e.autocomplete || "") + e.name + e.id)) c.value = "(hidden)";
    else c.value = clean(e.value).slice(0, 80);
    out.controls.push(c);
    if (out.controls.length >= 80) break;
  }
  for (const e of [...document.querySelectorAll('[role=listbox],[role=menu],[role=dialog],[class*=dropdown-menu i],[class*=datepicker i],[class*=calendar i],[class*=popup i],[class*=popover i]')].slice(0, 60)) {
    if (!vis(e)) continue;
    out.popups.push({ tag: e.tagName.toLowerCase(), role: e.getAttribute("role") || "", cls: String(e.className && e.className.baseVal !== undefined ? e.className.baseVal : e.className).slice(0, 60), text: clean(e.innerText || e.textContent).slice(0, 160) });
    if (out.popups.length >= 10) break;
  }
  return out;
}


















//skills.js
// Functions in this file are injected into the page via chrome.scripting.executeScript({func}).
// They must be fully self-contained (no references to outside variables).

function snapshotFn(maxText, maxEls, opts) {
  maxEls = maxEls || 200;
  // Shadow hosts are discovered ONCE per snapshot (a full "*" scan per query is far too slow on pages with thousands of nodes).
  let hostCache = null;
  function shadowHosts() {
    if (hostCache) return hostCache;
    hostCache = [];
    const stack = [document];
    while (stack.length) {
      const r = stack.pop();
      r.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) { hostCache.push(e); stack.push(e.shadowRoot); } });
    }
    return hostCache;
  }
  function deepAll(selector, root, out) {
    root = root || document;
    out = out || [];
    root.querySelectorAll(selector).forEach((e) => out.push(e));
    if (root === document) for (const h of shadowHosts()) h.shadowRoot.querySelectorAll(selector).forEach((e) => out.push(e));
    return out;
  }

  deepAll("[data-aq-id]").forEach((e) => e.removeAttribute("data-aq-id"));
  const SEL =
    'a[href],button,input,select,textarea,summary,[role=button],[role=link],[role=checkbox],[role=radio],' +
    '[role=tab],[role=menuitem],[role=option],[role=combobox],[role=switch],[role=textbox],' +
    '[contenteditable=""],[contenteditable="true"],[onclick],[tabindex]:not([tabindex="-1"])';
  const SENS =
    /\bpass(word|code|phrase)\b|\bcard.?(number|no)\b|\bcc-(number|csc|exp)|\bcvv\b|\bcvc\b|\botp\b|one.?time.?(code|password)|\bssn\b|aadhaar|social.?security|contrase[ñn]a|clave de acceso|n[úu]mero de (tarjeta|cuenta)|mot de passe|code secret|num[ée]ro de carte|passwort|kennwort|kartennummer|prüfziffer|senha|n[úu]mero do cart[ãa]o|пароль|номер карты|одноразов|密码|口令|卡号|银行卡|验证码|身份证|パスワード|暗証|カード番号|認証コード|ワンタイム|كلمة (المرور|السر)|رقم البطاقة|رمز التحقق|पासवर्ड|कार्ड नंबर|ओटीपी|आधार|পাসওয়ার্ড|কার্ড নম্বর|ওটিপি|கடவுச்சொல்|அட்டை எண்|ఓటిపి|పాస్‌వర్డ్|పాస్వర్డ్|కార్డ్ నంబర్|पासवर्ड|કાર્ડ નંબર|પાસવર્ડ|ಪಾಸ್‌ವರ್ಡ್|ಪಾಸ್ವರ್ಡ್|ಕಾರ್ಡ್ ಸಂಖ್ಯೆ|പാസ്‌വേഡ്|പാസ്വേഡ്|കാർഡ് നമ്പർ|ਪਾਸਵਰਡ|ਕਾਰਡ ਨੰਬਰ|ପାସୱାର୍ଡ|କାର୍ଡ ନମ୍ବର|ଓଟିପି|비밀번호|패스워드|암호|카드 ?번호|인증 ?(번호|코드)|보안 ?코드|주민 ?등록|일회용/i;
  const clean = (s) => (s || "").replace(/\s+/g, " ").trim();
  const txt = (el) => (el.innerText !== undefined ? el.innerText : el.textContent) || "";
  const vh = window.innerHeight || 800;

  const isVisible = (el) => {
    const r = el.getBoundingClientRect();
    if (r.width < 2 || r.height < 2) return false;
    const cs = getComputedStyle(el);
    return cs.visibility !== "hidden" && cs.display !== "none" && cs.opacity !== "0";
  };

  const labelOf = (el) => {
    let l = el.getAttribute("aria-label");
    const lb = el.getAttribute("aria-labelledby");
    if (!l && lb) l = lb.split(" ").map((i) => { const n = document.getElementById(i); return n ? txt(n) : ""; }).join(" ");
    if (!l && el.labels && el.labels[0]) l = txt(el.labels[0]);
    if (!l && ["INPUT", "TEXTAREA", "SELECT"].includes(el.tagName)) l = el.placeholder || el.name || el.id;
    if (!l && el.tagName === "INPUT" && ["submit", "button"].includes(el.type)) l = el.value;
    if (!l) { const img = el.querySelector && el.querySelector("img[alt]"); l = txt(el) || el.title || el.alt || (img && img.alt) || ""; }
    l = clean(l);
    if (!l) {
      const svgTitle = el.querySelector && el.querySelector("svg title");
      l = clean(svgTitle ? svgTitle.textContent : "");
    }
    if (!l) {
      const hint = clean(el.getAttribute("data-testid") || el.getAttribute("data-test") || el.id || (typeof el.className === "string" ? el.className : ""));
      if (hint) l = "(icon: " + hint.slice(0, 40) + ")";
    }
    return l.slice(0, 80);
  };

  const base = new Set(deepAll(SEL));
  // Hidden checkbox/radio (custom switches): the visible label is what a user clicks.
  base.forEach((el) => {
    if (el.tagName === "INPUT" && (el.type === "checkbox" || el.type === "radio") && !isVisible(el) && el.labels && el.labels[0]) base.add(el.labels[0]);
  });
  // Clickable things with no semantic markup (div/span with a click handler show a pointer cursor).
  if (base.size <= 400) {
    const vw = window.innerWidth || 1200;
    let scanned = 0;
    for (const el of deepAll("div,span,li,label,svg,img,td,p,i,section,article")) {
      if (++scanned > 4000) break;
      if (base.has(el)) continue;
      if (getComputedStyle(el).cursor !== "pointer") continue;
      const par = el.parentElement;
      if (par && getComputedStyle(par).cursor === "pointer") continue;
      if (el.querySelector(SEL)) continue;
      const r = el.getBoundingClientRect();
      if (r.width * r.height > 0.5 * vw * vh) continue;
      base.add(el);
    }
  }
  const viewCache = new Map();
  const inView = (el) => { let v = viewCache.get(el); if (v === undefined) { const r = el.getBoundingClientRect(); v = r.bottom > 0 && r.top < vh; viewCache.set(el, v); } return v; };
  let cand = [...base].filter((el) => !(el.tagName === "INPUT" && el.type === "hidden"));
  cand.sort((a, b) => (a.compareDocumentPosition(b) & 4 ? -1 : 1));
  let nodes;
  if (cand.length > maxEls * 3) {
    // Huge page (thousands of cards/rows): only the first maxEls VISIBLE controls matter (on-screen first), so stop checking
    // computed styles once we have enough instead of testing every node.
    const a = [], b = [];
    for (const el of cand) (inView(el) ? a : b).push(el);
    nodes = [];
    for (const el of a) { if (nodes.length >= maxEls) break; if (isVisible(el)) nodes.push(el); }
    for (const el of b) { if (nodes.length >= maxEls) break; if (isVisible(el)) nodes.push(el); }
    nodes.sort((x, y) => (x.compareDocumentPosition(y) & 4 ? -1 : 1));
  } else {
    nodes = cand.filter(isVisible);
    if (nodes.length > maxEls) {
      const a = nodes.filter(inView);
      const b = nodes.filter((n) => !inView(n));
      const keep = new Set([...a, ...b].slice(0, maxEls));
      nodes = nodes.filter((n) => keep.has(n));
    }
  }

  // Compact "how to find this element again" descriptor, used when a successful run is saved as a skill.
  const okId = (id) => !!id && id.length < 60 && !/\d{3,}|^:|[:\s]/.test(id);
  const cssPath = (el) => {
    const parts = [];
    let n = el;
    for (let i = 0; n && n.nodeType === 1 && i < 6; i++, n = n.parentElement) {
      if (okId(n.id)) { parts.unshift("#" + String(n.id).replace(/[^\w-]/g, "\\$&")); break; }
      let s = n.tagName.toLowerCase();
      // position among same-tag siblings, counted by walking backwards (capped, so huge lists stay cheap)
      let idx = 1, steps = 0, sib = n.previousElementSibling, capped = false;
      while (sib && steps++ < 40) { if (sib.tagName === n.tagName) idx++; sib = sib.previousElementSibling; }
      if (sib) capped = true;
      if (!capped && (idx > 1 || (n.nextElementSibling && n.nextElementSibling.tagName === n.tagName))) s += ":nth-of-type(" + idx + ")";
      parts.unshift(s);
    }
    return parts.join(" > ");
  };
  const describeForSkill = (el, label) => {
    const row = el.closest && el.closest("tr,li,[role=row],[role=listitem],[role=dialog],dialog,form");
    let ctx = row && row !== el ? clean(txt(row)).slice(0, 50) : "";
    if (ctx === label) ctx = "";
    const d = {
      tag: el.tagName.toLowerCase(), role: el.getAttribute("role") || "", type: el.tagName === "INPUT" ? el.type : "",
      id: okId(el.id) ? el.id : "", name: el.getAttribute("name") || "",
      testid: el.getAttribute("data-testid") || el.getAttribute("data-test") || el.getAttribute("data-qa") || el.getAttribute("data-cy") || "",
      label: /^\(icon:/.test(label) ? "" : label, placeholder: el.getAttribute("placeholder") || "", ctx, nth: 0, css: "",
    };
    try { d.css = cssPath(el); } catch (e) {}
    if (el.tagName === "A" && el.getAttribute("href")) { try { d.href = new URL(el.href).pathname.slice(0, 100); } catch (e) {} }
    return d;
  };

  const elements = nodes.map((el, i) => {
    const id = i + 1;
    el.setAttribute("data-aq-id", String(id));
    const tag = el.tagName.toLowerCase();
    const type = tag === "input" ? (el.type || "text") : "";
    const label = labelOf(el);
    const sensitive =
      type === "password" || SENS.test([el.getAttribute("autocomplete"), el.name, el.id, label].join(" "));
    const o = { id, tag, role: el.getAttribute("role") || "", type, label, sensitive, inView: inView(el) };
    o.d = describeForSkill(el, label);
    if (!sensitive && (tag === "input" || tag === "textarea") && !["checkbox", "radio", "submit", "button"].includes(type)) {
      o.value = String(el.value || "").slice(0, 60);
    }
    if (tag === "select") {
      const sel = el.options[el.selectedIndex];
      o.value = sel ? clean(sel.text) : "";
      o.options = [...el.options].slice(0, 25).map((x) => clean(x.text));
    }
    if (tag === "label" && el.control && (el.control.type === "checkbox" || el.control.type === "radio")) { o.checked = !!el.control.checked; o.role = "toggle"; }
    else if (type === "checkbox" || type === "radio") o.checked = !!el.checked;
    else if (el.getAttribute("aria-checked")) o.checked = el.getAttribute("aria-checked") === "true";
    if (el.disabled || el.getAttribute("aria-disabled") === "true") o.disabled = true;
    if (el.required) o.required = true;
    if (tag === "a" && el.href) {
      try { const u = new URL(el.href); o.href = (u.host + u.pathname).slice(0, 80); } catch (e) {}
    }
    return o;
  });

  // Tour mode only: also expose page AREAS (nav, sidebar, header, tables, forms, headings...) so a guide can explain
  // the whole app, and tell each control which area/row it sits in so it is described accurately.
  let regions = [];
  if (opts && opts.tour) {
    const RSEL =
      "header,nav,aside,main,footer,section,form,table,dialog,h1,h2,h3," +
      "[role=banner],[role=navigation],[role=complementary],[role=main],[role=contentinfo],[role=region],[role=tablist],[role=toolbar]," +
      "[role=search],[role=grid],[role=table],[role=dialog],[role=menu],[role=tree]," +
      "[class*=sidebar],[class*=side-bar],[class*=navbar],[class*=toolbar],[class*=breadcrumb],[class*=drawer],[class*=modal],[class*=tabs]" +
      (opts.full ? ",h4,canvas,[class*=chart],[class*=card],[class*=widget],[class*=kpi],[class*=tile],[class*=panel],[class*=banner],[class*=alert]" : "");
    const KIND_OK = /^(header|nav|aside|main|footer|table|form|dialog|banner|navigation|complementary|contentinfo|tablist|toolbar|search|grid|menu|tree|canvas)$/;
    const regionEls = [];
    const regionOf = new Map();
    let n = elements.length;
    const vwid = window.innerWidth || 1200;
    for (const el of deepAll(RSEL)) {
      if (regions.length >= (opts.full ? 80 : 40)) break;
      if (el.hasAttribute("data-aq-id")) continue;
      const r = el.getBoundingClientRect();
      if (r.width < 60 || r.height < 24 || !isVisible(el)) continue;
      if (r.right <= 0 || r.left >= vwid) continue;
      const tag = el.tagName.toLowerCase();
      const kind = el.getAttribute("role") || (/^h[1-3]$/.test(tag) ? "heading" : tag);
      let label = clean(el.getAttribute("aria-label") || "");
      const lb = el.getAttribute("aria-labelledby");
      if (!label && lb) label = clean(lb.split(" ").map((i) => { const x = document.getElementById(i); return x ? txt(x) : ""; }).join(" "));
      if (!label && tag === "table") { const c = el.querySelector("caption"); label = clean(c ? txt(c) : ""); }
      if (!label && kind === "heading") label = clean(txt(el));
      if (!label) { const h = el.querySelector("h1,h2,h3,h4,legend"); label = clean(h ? txt(h) : ""); }
      if (!label && opts.full && !KIND_OK.test(kind)) label = clean(txt(el)).slice(0, 50); // cards/widgets: name them by their first text
      label = label.slice(0, 60);
      if (!label && !KIND_OK.test(kind)) continue; // an unlabeled generic <section>/div adds noise
      n++;
      el.setAttribute("data-aq-id", String(n));
      const reg = { id: n, kind, label };
      regions.push(reg);
      regionOf.set(el, reg);
      regionEls.push([el, reg]);
    }
    nodes.forEach((el, i) => {
      let p = el.parentElement || (el.getRootNode && el.getRootNode().host), g = 0;
      while (p && g++ < 14) {
        const reg = regionOf.get(p);
        if (reg && reg.kind !== "heading") { elements[i].area = reg.kind + (reg.label ? ' "' + reg.label + '"' : ""); break; }
        p = p.parentElement || (p.getRootNode && p.getRootNode().host);
      }
      const row = el.closest && el.closest("tr,li,[role=row],[role=listitem]");
      if (row) {
        const rt = clean(txt(row)).slice(0, 60);
        if (rt && rt !== elements[i].label) elements[i].ctx = rt;
      }
    });
    // reading order (document order) across controls AND areas, so a tour can walk the page top to bottom
    const seqItems = nodes.map((el, i) => [el, elements[i]]).concat(regionEls);
    seqItems.sort((a, b) => {
      if (a[0] === b[0]) return 0;
      const c = a[0].compareDocumentPosition(b[0]);
      return c & 4 ? -1 : c & 2 ? 1 : 0;
    });
    seqItems.forEach((x, k) => { x[1].seq = k; });
  }

  // Repeated structures (cards, rows, list items...). The model only ever sees a slice of a big page, so it needs to know
  // what repeats and how often, and can then use query_page to count/list them instantly instead of scrolling.
  let repeats = [], totalElements = 0;
  try {
    const all = document.body ? document.body.getElementsByTagName("*") : [];
    totalElements = all.length;
    const lim = Math.min(all.length, 60000);
    const cssEsc = (c) => (typeof CSS !== "undefined" && CSS.escape ? CSS.escape(c) : c.replace(/[^\w-]/g, "\\$&"));
    const keyCache = new Map();
    const counts = new Map();
    for (let i = 0; i < lim; i++) {
      const e = all[i];
      const cn = e.className;
      if (typeof cn !== "string" || !cn) continue;
      const ck = e.tagName + "|" + cn;
      let key = keyCache.get(ck);
      if (key === undefined) {
        const cls = cn.split(/\s+/).filter((c) => c && c.length <= 40 && !/\d/.test(c)).slice(0, 3);
        key = cls.length ? e.tagName.toLowerCase() + "." + cls.map((c) => cssEsc(c)).join(".") : "";
        keyCache.set(ck, key);
      }
      if (!key) continue;
      const c = counts.get(key);
      if (c) c.n++; else counts.set(key, { n: 1, first: e });
    }
    const cands = [...counts.entries()].filter(([, v]) => v.n >= 6).map(([sel, v]) => ({ sel, n: v.n, first: v.first, size: v.first.getElementsByTagName("*").length }));
    cands.sort((a, b) => b.n - a.n || b.size - a.size);
    repeats = cands.slice(0, 8).map((c) => {
      let n = c.n;
      try { n = document.querySelectorAll(c.sel).length; } catch (e) {}
      return { selector: c.sel, count: n, sample: clean(c.first.textContent || "").slice(0, 70) };
    });
  } catch (e) {}

  const body = (document.body ? txt(document.body) : "").replace(/\n{3,}/g, "\n\n");
  return {
    repeats,
    totalElements,
    url: location.href,
    title: document.title,
    scrollY: Math.round(window.scrollY),
    scrollMax: Math.max(0, Math.round(document.documentElement.scrollHeight - vh)),
    iframes: document.querySelectorAll("iframe").length,
    elements,
    regions,
    text: body.slice(0, maxText),
    totalChars: body.length,
  };
}

// Read-only aggregate query over the WHOLE page (not just the visible slice). Used for "how many ...", "list all ...", "which ... are ..." on big pages.
function queryFn(q) {
  const clean = (x) => (x || "").replace(/\s+/g, " ").trim();
  const out = { ok: true };
  try {
    const sel = String(q.selector || "").trim();
    if (!sel) return { ok: false, msg: "selector is required (a CSS selector such as .card or tr.row)." };
    const nodes = [];
    const walk = (root) => {
      root.querySelectorAll(sel).forEach((e) => nodes.push(e));
      root.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) walk(e.shadowRoot); });
    };
    try { walk(document); } catch (e) { return { ok: false, msg: "Invalid CSS selector: " + e.message }; }
    let m = nodes;
    if (q.text_contains) {
      const needle = String(q.text_contains).toLowerCase();
      m = m.filter((e) => (e.textContent || "").toLowerCase().includes(needle));
    }
    if (q.visible_only) {
      m = m.filter((e) => {
        const r = e.getBoundingClientRect();
        if (r.width < 2 || r.height < 2) return false;
        const cs = getComputedStyle(e);
        return cs.visibility !== "hidden" && cs.display !== "none";
      });
    }
    out.count = m.length;
    out.matchedBeforeFilters = nodes.length;
    const nlist = Math.max(0, Math.min(Number(q.list) || 0, 50));
    if (nlist) out.items = m.slice(0, nlist).map((e) => clean(e.textContent).slice(0, 160));
    if (q.count_by) {
      const g = new Map();
      for (const e of m) {
        let t = "";
        try { const x = e.querySelector(String(q.count_by)); t = x ? clean(x.textContent).slice(0, 60) : "(none)"; } catch (err) { return { ok: false, msg: "Invalid count_by selector: " + err.message }; }
        g.set(t, (g.get(t) || 0) + 1);
      }
      out.groups = [...g.entries()].sort((a, b) => b[1] - a[1]).slice(0, 30).map(([value, n]) => ({ value, n }));
      out.distinct = g.size;
    }
    if (q.attribute) {
      const vals = m.slice(0, 50).map((e) => e.getAttribute(String(q.attribute))).filter((v) => v != null);
      out.attributes = vals.map((v) => String(v).slice(0, 120));
    }
    return out;
  } catch (e) {
    return { ok: false, msg: String(e && e.message || e) };
  }
}

// Resolves as soon as the page has stopped changing (no DOM mutations for quietMs), or after maxMs at the latest.
// Much faster than sleeping a fixed time after every action.
function quietFn(quietMs, maxMs) {
  return new Promise((resolve) => {
    const t0 = Date.now();
    let last = t0;
    const mo = new MutationObserver(() => { last = Date.now(); });
    try {
      mo.observe(document.documentElement, { subtree: true, childList: true, characterData: true, attributes: true, attributeFilter: ["class", "hidden", "open", "disabled", "aria-expanded", "aria-hidden", "aria-busy"] });
    } catch (e) {}
    const iv = setInterval(() => {
      const now = Date.now();
      if (now - last >= quietMs || now - t0 >= maxMs) { clearInterval(iv); mo.disconnect(); resolve(now - t0); }
    }, 25);
  });
}

function readFn(max) {
  const t = (document.body ? (document.body.innerText !== undefined ? document.body.innerText : document.body.textContent) : "") || "";
  const body = t.replace(/\n{3,}/g, "\n\n");
  return { text: body.slice(0, max), totalChars: body.length };
}

function actFn(name, a) {
  function deepAll(selector, root, out) {
    root = root || document;
    out = out || [];
    root.querySelectorAll(selector).forEach((e) => out.push(e));
    root.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) deepAll(selector, e.shadowRoot, out); });
    return out;
  }
  const find = (id) => deepAll('[data-aq-id="' + id + '"]')[0] || null;
  const notFound = { ok: false, msg: "Element not found. The page changed; use ids from the latest snapshot." };
  const key = (el, k) => {
    for (const t of ["keydown", "keypress", "keyup"]) {
      el.dispatchEvent(new KeyboardEvent(t, { key: k, code: k, bubbles: true, cancelable: true }));
    }
  };
  const fire = (el) => {
    el.dispatchEvent(new Event("input", { bubbles: true }));
    el.dispatchEvent(new Event("change", { bubbles: true }));
  };
  try {
    if (name === "click") {
      const el = find(a.id);
      if (!el) return notFound;
      if (el.disabled) return { ok: false, msg: "Element is disabled." };
      if (el.scrollIntoView) el.scrollIntoView({ block: "center", inline: "center" });
      if (el.focus) el.focus({ preventScroll: true });
      try {
        const r = el.getBoundingClientRect();
        const o = { bubbles: true, cancelable: true, view: window, clientX: r.left + r.width / 2, clientY: r.top + r.height / 2, button: 0 };
        if (typeof PointerEvent === "function") el.dispatchEvent(new PointerEvent("pointerdown", { ...o, pointerType: "mouse", isPrimary: true }));
        el.dispatchEvent(new MouseEvent("mousedown", o));
        if (typeof PointerEvent === "function") el.dispatchEvent(new PointerEvent("pointerup", { ...o, pointerType: "mouse", isPrimary: true }));
        el.dispatchEvent(new MouseEvent("mouseup", o));
      } catch (e) {}
      el.click();
      return { ok: true, msg: "Clicked." };
    }
    if (name === "type_text") {
      const el = find(a.id);
      if (!el) return notFound;
      if (el.scrollIntoView) el.scrollIntoView({ block: "center" });
      if (el.focus) el.focus();
      const isTA = el instanceof HTMLTextAreaElement;
      if (isTA || el instanceof HTMLInputElement) {
        const proto = isTA ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
        const set = Object.getOwnPropertyDescriptor(proto, "value").set;
        set.call(el, (a.append ? el.value : "") + a.text);
        fire(el);
      } else if (el.isContentEditable) {
        if (!a.append) document.execCommand("selectAll", false);
        document.execCommand("insertText", false, a.text);
      } else {
        return { ok: false, msg: "Element is not a text field." };
      }
      if (a.submit) {
        if (el.form && el.form.requestSubmit) el.form.requestSubmit();
        else key(el, "Enter");
      }
      return { ok: true, msg: "Typed." };
    }
    if (name === "select_option") {
      const el = find(a.id);
      if (!el) return notFound;
      if (el.tagName !== "SELECT") return { ok: false, msg: "Not a <select>. Click it and choose the option instead." };
      const want = String(a.option).trim().toLowerCase();
      const opts = [...el.options];
      const opt =
        opts.find((o) => o.text.trim().toLowerCase() === want || o.value.toLowerCase() === want) ||
        opts.find((o) => o.text.toLowerCase().includes(want));
      if (!opt) return { ok: false, msg: "No such option. Available: " + opts.slice(0, 25).map((o) => o.text.trim()).join(" | ") };
      const set = Object.getOwnPropertyDescriptor(HTMLSelectElement.prototype, "value").set;
      set.call(el, opt.value);
      fire(el);
      return { ok: true, msg: "Selected " + opt.text.trim() + "." };
    }
    if (name === "scroll") {
      const h = window.innerHeight || 800;
      if (a.direction === "top") window.scrollTo(0, 0);
      else if (a.direction === "bottom") window.scrollTo(0, document.documentElement.scrollHeight);
      else window.scrollBy(0, (a.direction === "up" ? -1 : 1) * Math.round(h * 0.8));
      return { ok: true, msg: "Scrolled." };
    }
    if (name === "press_key") {
      const el = document.activeElement || document.body;
      if (a.key === "Enter" && (el.tagName === "BUTTON" || el.tagName === "A")) el.click();
      else key(el, a.key);
      return { ok: true, msg: "Pressed " + a.key + "." };
    }
    return { ok: false, msg: "Unknown action " + name };
  } catch (e) {
    return { ok: false, msg: String(e) };
  }
}

// ---------- guided tour overlay (runs in the page; drawn in a shadow root, never touches the page's own DOM/styles) ----------
// steps: [{id?, title, text, click?}]  L: translated labels  o: {finished, rtl}
// A step that points at an element is only shown if that element is REALLY visible on screen (rendered, not clipped by a
// scroll area / collapsed panel, not covered). Otherwise the step is dropped, so the guide never highlights empty space.
// Reports back with chrome.runtime.sendMessage({__pa_tour:true, type:"advance"|"exit", reason}).
function tourFn(steps, L, o) {
  if (window.__paTour) window.__paTour.destroy();
  steps = steps.slice();
  const send = (m) => { try { chrome.runtime.sendMessage({ __pa_tour: true, ...m }); } catch (e) {} };
  const deepAll = (selector, root = document, out = []) => {
    root.querySelectorAll(selector).forEach((e) => out.push(e));
    root.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) deepAll(selector, e.shadowRoot, out); });
    return out;
  };
  const findEl = (s) => (s && s.id != null ? deepAll('[data-aq-id="' + s.id + '"]')[0] || null : null);

  // ----- is this element really on screen? -----
  const okStyle = (c) => {
    if (!c || !c.isConnected || !c.getBoundingClientRect) return false;
    const cs = getComputedStyle(c);
    if (cs.display === "none" || cs.visibility === "hidden" || parseFloat(cs.opacity) < 0.05) return false;
    if (c.closest && c.closest("[hidden],[inert]")) return false;
    return true;
  };
  // visible part of the element after clipping by the viewport and by every overflow:hidden/auto/scroll ancestor
  const clipRect = (el, r) => {
    let Lf = Math.max(0, r.left), Tp = Math.max(0, r.top), Rt = Math.min(window.innerWidth, r.right), Bt = Math.min(window.innerHeight, r.bottom);
    let cur = el, p = el.parentNode, guard = 0;
    while (p && guard++ < 60) {
      if (p.nodeType === 11) { cur = p.host; p = p.host && p.host.parentNode; continue; }
      if (p.nodeType !== 1) break;
      if (getComputedStyle(cur).position === "fixed") break; // fixed elements ignore ancestors' overflow
      if (p !== document.documentElement && p !== document.body) {
        const cs = getComputedStyle(p);
        if (cs.overflowX !== "visible" || cs.overflowY !== "visible") {
          const pr = p.getBoundingClientRect();
          Lf = Math.max(Lf, pr.left); Tp = Math.max(Tp, pr.top); Rt = Math.min(Rt, pr.right); Bt = Math.min(Bt, pr.bottom);
        }
      }
      cur = p; p = p.parentNode;
    }
    return { left: Lf, top: Tp, right: Rt, bottom: Bt, width: Math.max(0, Rt - Lf), height: Math.max(0, Bt - Tp) };
  };
  const covered = (c, cr) => {
    if (typeof document.elementFromPoint !== "function") return false;
    const pts = [[0.5, 0.5], [0.2, 0.3], [0.8, 0.7]];
    for (const [fx, fy] of pts) {
      const t = document.elementFromPoint(cr.left + cr.width * fx, cr.top + cr.height * fy);
      if (!t) return false;
      const host = c.getRootNode && c.getRootNode().host;
      if (t === c || c.contains(t) || t.contains(c) || (host && (host === t || host.contains(t)))) return false;
    }
    return true;
  };
  // returns {el, rect} or null. With scroll=true, scrolls the element into view first (also inside scroll containers).
  const locate = (s, scroll) => {
    const el0 = findEl(s);
    if (!el0) return null;
    const cands = [el0];
    if (el0.tagName === "INPUT" && ["checkbox", "radio", "file"].includes(el0.type)) {
      if (el0.labels && el0.labels[0]) cands.push(el0.labels[0]);
      const lab = el0.closest && el0.closest("label");
      if (lab) cands.push(lab);
    }
    for (const c of cands) {
      if (!okStyle(c)) continue;
      let r = c.getBoundingClientRect();
      if (r.width < 4 || r.height < 4) continue;
      const need = (rr) => 0.5 * Math.min(rr.width * rr.height, 0.4 * window.innerWidth * window.innerHeight);
      let cr = clipRect(c, r);
      if (scroll && cr.width * cr.height < need(r)) {
        try { c.scrollIntoView({ block: "center", inline: "center", behavior: "instant" }); } catch (e) {}
        r = c.getBoundingClientRect();
        cr = clipRect(c, r);
      }
      if (cr.width < 4 || cr.height < 4 || cr.width * cr.height < need(r)) continue;
      if (covered(c, cr)) continue;
      return { el: c, orig: el0, rect: cr };
    }
    return null;
  };

  // ----- UI -----
  const host = document.createElement("div");
  host.setAttribute("data-pa-tour", "1");
  host.style.cssText = "all:initial;position:fixed;inset:0;z-index:2147483647;pointer-events:none;";
  host.dir = o && o.rtl ? "rtl" : "ltr";
  const root = host.attachShadow({ mode: "open" });
  const style = document.createElement("style");
  style.textContent = `
    :host{all:initial}
    *{box-sizing:border-box;font-family:system-ui,-apple-system,"Segoe UI",Roboto,sans-serif}
    .dim{position:fixed;inset:0;background:rgba(15,23,42,.55);pointer-events:none}
    .ring{position:fixed;border:3px solid #3b82f6;border-radius:8px;box-shadow:0 0 0 9999px rgba(15,23,42,.55);pointer-events:none;transition:all .18s ease}
    .tip{position:fixed;width:min(320px,calc(100vw - 24px));background:#fff;color:#1f2328;border-radius:12px;padding:12px 14px;box-shadow:0 10px 30px rgba(0,0,0,.35);pointer-events:auto;font-size:14px;line-height:1.45}
    .top{display:flex;justify-content:space-between;align-items:center;gap:8px;margin-bottom:4px}
    .count{font-size:12px;color:#57606a}
    .x{all:unset;cursor:pointer;font-size:18px;line-height:1;padding:2px 6px;border-radius:6px;color:#57606a}
    .x:hover{background:#eef2f7}
    h3{margin:0 0 4px;font-size:15px}
    p{margin:0 0 8px;white-space:pre-wrap;overflow-wrap:anywhere}
    .hintc{font-size:12px;color:#0b57d0;margin-bottom:8px}
    .btns{display:flex;justify-content:flex-end;gap:8px}
    button.b{all:unset;cursor:pointer;padding:6px 14px;border-radius:8px;font-size:13px;font-weight:600;background:#eef2f7;color:#1f2328}
    button.b.next{background:#0b57d0;color:#fff}
    button.b:focus-visible{outline:2px solid #0b57d0;outline-offset:2px}
    [hidden]{display:none!important}
    @media (prefers-color-scheme:dark){.tip{background:#1e2530;color:#e6edf3}.count,.x{color:#9aa7b4}.x:hover{background:#2b3544}.hintc{color:#7db4ff}button.b{background:#2b3544;color:#e6edf3}button.b.next{background:#2f6fe0;color:#fff}}
  `;
  const mk = (tag, cls, text) => { const e = document.createElement(tag); if (cls) e.className = cls; if (text != null) e.textContent = text; return e; };
  const dim = mk("div", "dim"), ring = mk("div", "ring"), tip = mk("div", "tip");
  tip.setAttribute("role", "dialog");
  tip.setAttribute("aria-live", "polite");
  const count = mk("span", "count"), xBtn = mk("button", "x", "×");
  xBtn.setAttribute("aria-label", L.exit);
  xBtn.title = L.exit;
  const top = mk("div", "top");
  top.append(count, xBtn);
  const h = mk("h3"), p = mk("p"), hint = mk("div", "hintc");
  const back = mk("button", "b back", L.back), next = mk("button", "b next");
  const btns = mk("div", "btns");
  btns.append(back, next);
  tip.append(top, h, p, hint, btns);
  root.append(style, dim, ring, tip);
  document.documentElement.appendChild(host);

  let i = 0, waiting = false, clickEls = [], clickFn = null, timer = null, shown = 0;
  const detach = () => { if (clickFn) clickEls.forEach((e) => e.removeEventListener("click", clickFn, true)); clickEls = []; clickFn = null; };

  function position() {
    if (waiting || !steps[i]) return;
    const s = steps[i];
    const loc = s.id != null ? locate(s, false) : null;
    const vw = window.innerWidth, vh = window.innerHeight, pad = 6, gap = 12;
    const tw = tip.offsetWidth || 320, th = tip.offsetHeight || 140;
    if (!loc) {
      ring.hidden = true; dim.hidden = false;
      tip.style.left = Math.max(12, (vw - tw) / 2) + "px";
      tip.style.top = Math.max(12, (vh - th) / 2) + "px";
      return;
    }
    const r = loc.rect;
    dim.hidden = true; ring.hidden = false;
    ring.style.left = r.left - pad + "px"; ring.style.top = r.top - pad + "px";
    ring.style.width = r.width + pad * 2 + "px"; ring.style.height = r.height + pad * 2 + "px";
    let ty = r.bottom + pad + gap;
    if (ty + th > vh - 8) ty = r.top - pad - gap - th;
    if (ty < 8) ty = Math.min(vh - th - 8, Math.max(8, r.bottom + pad + gap)); // no room above or below: overlap rather than leave the screen
    let tx = r.left + r.width / 2 - tw / 2;
    tx = Math.max(12, Math.min(vw - tw - 12, tx));
    tip.style.left = tx + "px"; tip.style.top = ty + "px";
  }

  function paint(loc) {
    detach();
    waiting = false;
    const s = steps[i], last = i === steps.length - 1;
    count.textContent = L.step.replace("{n}", ((o && o.offset) || 0) + i + 1).replace("{total}", ((o && o.offset) || 0) + steps.length + ((o && o.rest) || 0));
    h.textContent = s.title || ""; h.hidden = !s.title;
    p.textContent = s.text || "";
    const wantsClick = !!(s.click && loc);
    hint.textContent = wantsClick ? L.click : ""; hint.hidden = !wantsClick;
    back.hidden = i === 0;
    next.textContent = last && o && o.finished ? L.done : L.next;
    btns.hidden = false; xBtn.hidden = false;
    if (wantsClick) {
      clickEls = loc.orig === loc.el ? [loc.el] : [loc.el, loc.orig];
      clickFn = () => { setTimeout(() => advance("click"), 0); }; // let the page handle the click first
      clickEls.forEach((e) => e.addEventListener("click", clickFn, true));
    }
    position();
    try { next.focus({ preventScroll: true }); } catch (e) {}
  }

  // Show the first step at/after (dir>0) or at/before (dir<0) index k whose target can really be seen; drop the others.
  function show(k, dir) {
    while (k >= 0 && k < steps.length) {
      const s = steps[k];
      const loc = s.id == null ? null : locate(s, true);
      if (s.id == null || loc) { i = k; paint(loc); return true; }
      steps.splice(k, 1);
      if (k < i) i--; // keep i pointing at the step currently on screen
      if (dir < 0) k--;
    }
    return false;
  }

  function advance(reason) {
    detach();
    if (o && o.finished && reason === "next") { send({ type: "advance", reason, last: true, shown }); return destroy(); }
    // more to show: wait for the extension to prepare the next part
    waiting = true;
    ring.hidden = true; dim.hidden = true;
    count.textContent = ""; h.hidden = true; hint.hidden = true; btns.hidden = true; xBtn.hidden = true;
    p.textContent = L.wait;
    tip.style.left = Math.max(12, window.innerWidth - tip.offsetWidth - 16) + "px";
    tip.style.top = "16px";
    send({ type: "advance", reason, shown });
  }

  const onKey = (e) => { if (e.key === "Escape") { send({ type: "exit" }); destroy(); } };
  function destroy() {
    detach();
    clearInterval(timer);
    window.removeEventListener("keydown", onKey, true);
    window.removeEventListener("resize", position);
    window.removeEventListener("scroll", position, true);
    host.remove();
    if (window.__paTour && window.__paTour.host === host) window.__paTour = null;
  }
  next.onclick = () => { if (!show(i + 1, 1)) advance("next"); };
  back.onclick = () => { show(i - 1, -1); };
  xBtn.onclick = () => { send({ type: "exit" }); destroy(); };
  window.addEventListener("keydown", onKey, true);
  window.addEventListener("resize", position);
  window.addEventListener("scroll", position, true);
  timer = setInterval(position, 250);
  window.__paTour = { destroy, host };
  // Drop steps whose target cannot really be shown right now, up front, so "Step n of N" is honest and nothing points at empty space.
  steps = steps.filter((s) => s.id == null || locate(s, true));
  shown = steps.length;
  if (!show(0, 1)) advance("empty");
  return true;
}

function tourClearFn() {
  if (window.__paTour) window.__paTour.destroy();
  document.querySelectorAll("[data-pa-tour]").forEach((e) => e.remove());
  return true;
}

// ======================= Skills: recording, replay, direct requests =======================
// Everything below is injected into the page with chrome.scripting.executeScript, so each function is self-contained.

// Records what the user does on the page (clicks, typed values, selects, keys) plus mutating network requests.
// cmd: { cmd: "start", resume?: boolean } | { cmd: "dump" } | { cmd: "stop" }
function recorderFn(c) {
  const KEY = "__paRecState";
  const W = window;
  const clean = (s) => (s || "").replace(/\s+/g, " ").trim();
  const txt = (el) => clean(el.innerText !== undefined ? el.innerText : el.textContent);
  const okId = (id) => !!id && id.length < 60 && !/\d{3,}|^:|[:\s]/.test(id);
  const labelOf = (el) => {
    let l = el.getAttribute("aria-label");
    const lb = el.getAttribute("aria-labelledby");
    if (!l && lb) l = lb.split(" ").map((i) => { const n = document.getElementById(i); return n ? txt(n) : ""; }).join(" ");
    if (!l && el.labels && el.labels[0]) l = txt(el.labels[0]);
    if (!l && ["INPUT", "TEXTAREA", "SELECT"].includes(el.tagName)) l = el.placeholder || el.name || (okId(el.id) ? el.id : "");
    if (!l && el.tagName === "INPUT" && ["submit", "button"].includes(el.type)) l = el.value;
    if (!l) { const img = el.querySelector && el.querySelector("img[alt]"); l = txt(el) || el.title || el.alt || (img && img.alt) || ""; }
    return clean(l).slice(0, 80);
  };
  const cssPath = (el) => {
    const parts = [];
    let n = el;
    for (let i = 0; n && n.nodeType === 1 && i < 6; i++, n = n.parentElement) {
      if (okId(n.id)) { parts.unshift("#" + String(n.id).replace(/[^\w-]/g, "\\$&")); break; }
      let s = n.tagName.toLowerCase();
      const p = n.parentElement;
      if (p) { const same = [...p.children].filter((x) => x.tagName === n.tagName); if (same.length > 1) s += ":nth-of-type(" + (same.indexOf(n) + 1) + ")"; }
      parts.unshift(s);
    }
    return parts.join(" > ");
  };
  // The text next to / above a control (its visible caption), for naming an input when it has no real label.
  const fieldLabel = (el) => {
    let n = el, fl = "";
    for (let k = 0; k < 4 && n && !fl; k++, n = n.parentElement) {
      const lb = n.querySelector && n.querySelector("label");
      if (lb && !lb.contains(el)) { fl = clean(lb.innerText || lb.textContent); break; }
      const ps = n.previousElementSibling;
      if (ps && !/^(INPUT|SELECT|TEXTAREA|BUTTON)$/.test(ps.tagName)) { const t = clean(ps.innerText || ps.textContent); if (t && t.length < 40) fl = t; }
    }
    return fl.slice(0, 40);
  };
  const describe = (el) => {
    const label = labelOf(el);
    const row = el.closest("tr,li,[role=row],[role=listitem],[role=dialog],dialog,form");
    let ctx = row && row !== el ? txt(row).slice(0, 50) : "";
    if (ctx === label) ctx = "";
    let nth = 0, same = 1;
    try {
      const all = [...document.querySelectorAll(el.tagName.toLowerCase())].filter((x) => labelOf(x) === label);
      same = all.length; nth = Math.max(0, all.indexOf(el));
    } catch (e) {}
    const d = {
      tag: el.tagName.toLowerCase(), role: el.getAttribute("role") || "", type: el.tagName === "INPUT" ? el.type : "",
      id: okId(el.id) ? el.id : "", name: el.getAttribute("name") || "",
      testid: el.getAttribute("data-testid") || el.getAttribute("data-test") || el.getAttribute("data-qa") || el.getAttribute("data-cy") || "",
      label, placeholder: el.getAttribute("placeholder") || "", ctx, nth, same, css: cssPath(el),
    };
    if (/^(INPUT|SELECT|TEXTAREA)$/.test(el.tagName) && (!label || label === d.id || label === d.name)) { const fl = fieldLabel(el); if (fl) d.fl = fl; }
    if (el.tagName === "A" && el.getAttribute("href")) { try { d.href = new URL(el.href).pathname.slice(0, 100); } catch (e) {} }
    return d;
  };
  const save = () => {
    try {
      const R = W.__paRec;
      sessionStorage.setItem(KEY, JSON.stringify({ steps: R.steps, reqs: R.reqs.slice(-40) }));
    } catch (e) {}
  };

  if (c.cmd === "dump" || c.cmd === "stop") {
    const R = W.__paRec;
    let out;
    if (R) out = { steps: R.steps, reqs: R.reqs, url: location.href, title: document.title };
    else { try { out = Object.assign({ url: location.href, title: document.title }, JSON.parse(sessionStorage.getItem(KEY) || "{}")); } catch (e) { out = { steps: [], reqs: [], url: location.href, title: document.title }; } }
    if (c.cmd === "stop") {
      if (R) { R.on = false; (R.off || []).forEach((f) => { try { f(); } catch (e) {} }); }
      try { sessionStorage.removeItem(KEY); } catch (e) {}
    }
    return out;
  }

  // ---- start / resume
  if (W.__paRec && W.__paRec.on) return { ok: true, already: true };
  let prev = null;
  if (c.resume) { try { prev = JSON.parse(sessionStorage.getItem(KEY) || "null"); } catch (e) {} }
  const R = (W.__paRec = { on: true, steps: (prev && prev.steps) || [], reqs: (prev && prev.reqs) || [], muts: 0, off: [] });
  const mo = new MutationObserver((list) => { R.muts += list.length; });
  mo.observe(document.documentElement, { subtree: true, childList: true, characterData: true });
  R.off.push(() => mo.disconnect());

  const push = (st) => {
    st.at = Date.now(); st.url = location.href;
    const m0 = R.muts, u0 = location.href;
    R.steps.push(st);
    if (R.steps.length > 1500) R.steps.shift();
    setTimeout(() => { st.effect = R.muts - m0 > 2 || location.href !== u0; save(); }, 1200);
    save();
  };
  const isTextual = (el) => el.tagName === "TEXTAREA" || (el.tagName === "INPUT" && !["checkbox", "radio", "button", "submit", "reset", "file", "image", "range", "color"].includes(el.type));
  const flush = (el) => {
    if (!el || !(isTextual(el))) return;
    const sensitive = el.type === "password";
    const val = sensitive ? "(secret)" : el.value;
    if (el.__paLast === val || (el.__paLast === undefined && !el.value)) { el.__paLast = val; return; }
    el.__paLast = val;
    const d = describe(el);
    const last = R.steps[R.steps.length - 1];
    if (last && last.t === "type" && last.d.css === d.css && last.d.label === d.label) { last.value = sensitive ? "" : el.value; last.at = Date.now(); save(); return; }
    const stp = { t: "type", d, value: sensitive ? "" : el.value, sensitive };
    if (el.readOnly) stp.ro = true;
    push(stp);
  };
  // Read-only fields (date pickers): the user cannot type, a widget fills them after a click. Watch for that.
  const DATEISH = /\b\d{1,4}[\/\-.\s][A-Za-z0-9]{1,9}[\/\-.\s]\d{1,4}\b/;
  const seen = new WeakMap();
  const scanRO = (init) => {
    document.querySelectorAll("input[readonly]").forEach((inp) => {
      if (!isTextual(inp) || inp.type === "password") return;
      const v = inp.value, p = seen.get(inp);
      seen.set(inp, v);
      if (init || p === undefined || p === v || !v || inp.__paLast === v) return;
      if (!(DATEISH.test(v) || inp.type === "date")) return;
      inp.__paLast = v;
      push({ t: "type", d: describe(inp), value: v, sensitive: false, ro: true });
    });
  };
  scanRO(true);
  const scanSoon = () => { setTimeout(scanRO, 150); setTimeout(scanRO, 800); };
  const onChange = (e) => {
    const el = e.target;
    if (!el || !el.tagName) return;
    if (el.tagName === "SELECT") {
      const o = el.options[el.selectedIndex];
      const choices = [...el.options].map((x) => clean(x.text)).filter(Boolean).slice(0, 60);
      if (el.multiple) {
        const picked = [...el.selectedOptions].map((x) => clean(x.text)).filter(Boolean);
        const d0 = describe(el), last = R.steps[R.steps.length - 1];
        if (last && last.t === "select" && last.multi && last.d.css === d0.css) { last.value = picked.join("; "); last.at = Date.now(); save(); }
        else push({ t: "select", multi: true, d: d0, value: picked.join("; "), choices });
      } else push({ t: "select", d: describe(el), value: o ? clean(o.text) : el.value, choices });
    } else flush(el);
  };
  const CLICKABLE = 'a,button,input,select,textarea,summary,label,[role=button],[role=link],[role=tab],[role=menuitem],[role=option],[role=checkbox],[role=radio],[role=switch],[role=treeitem],[role=gridcell],[onclick],li,[tabindex]';
  const onClick = (e) => {
    if (!e.isTrusted && !W.__paAllowSynthetic) return;
    flush(document.activeElement);
    let el = e.target && e.target.closest ? e.target.closest(CLICKABLE) : null;
    if (!el) el = e.target;
    if (!el || !el.tagName) return;
    if (isTextual(el) || el.tagName === "SELECT" || el.isContentEditable) return; // focusing a field is not a step; its value is
    const last = R.steps[R.steps.length - 1];
    if (el.tagName === "INPUT" && last && last.t === "click" && Date.now() - last.at < 150 && last.d.tag === "label") return;
    const stp = { t: "click", d: describe(el) };
    // checkbox / radio (the control itself, or its label)
    {
      const ctl = el.tagName === "INPUT" ? el : el.tagName === "LABEL" ? el.control : null;
      if (ctl && (ctl.type === "checkbox" || ctl.type === "radio")) {
        stp.kind = ctl.type;
        if (ctl.type === "checkbox") stp.checked = el.tagName === "INPUT" ? ctl.checked : !ctl.checked;
        else {
          stp.checked = true;
          const grp = ctl.name ? [...document.querySelectorAll('input[type=radio]')].filter((x) => x.name === ctl.name) : [ctl];
          stp.choices = grp.map((x) => clean((x.labels && x.labels[0] ? (x.labels[0].innerText !== undefined ? x.labels[0].innerText : x.labels[0].textContent) : "") || x.getAttribute("aria-label") || x.value)).filter(Boolean).slice(0, 40);
          const fs = ctl.closest("fieldset"); const lg = fs && fs.querySelector("legend");
          if (lg) stp.fl = clean(lg.textContent).slice(0, 40);
        }
      }
    }
    // A choice from a dropdown / list / menu (custom widgets, not <select>): remember it so it can become an input.
    const box = el.closest('[role=listbox],[role=menu],[role=tree],[role=combobox],[class*=dropdown i],[class*=select i],[class*=autocomplete i],[class*=suggest i],[class*=popup i],[class*=popover i],[class*=overlay i],[class*=options i],[class*=menu i],[class*=chosen i],[class*=choices i],[class*=listbox i]');
    if ((el.matches('[role=option],[role=menuitem],[role=treeitem],li,option') || (box && box !== el)) && box && !el.closest("table,[role=grid],[class*=calendar i],[class*=datepicker i]") && stp.d.label && stp.d.label.length < 70) {
      const prevC = R.steps[R.steps.length - 1];
      if (prevC && prevC.t === "click" && Date.now() - prevC.at < 20000) {
        stp.pick = true;
        try { stp.box = cssPath(box); } catch (e) {}
        const sibs = el.parentElement ? [...el.parentElement.children].filter((x) => x.tagName === el.tagName) : [];
        stp.choices = (sibs.length > 1 ? sibs : [...box.querySelectorAll(el.tagName)]).map((x) => clean(x.innerText !== undefined ? x.innerText : x.textContent)).filter((x) => x && x.length < 70).slice(0, 60);
      }
    }
    if (!stp.fl) { const fl = fieldLabel(el); if (fl) stp.fl = fl; }
    const plain = !txt(el) && !/^(INPUT|TEXTAREA|SELECT)$/.test(el.tagName) || /^(IMG|SVG|I)$/i.test(el.tagName) || (el.tagName === "INPUT" && el.type === "image");
    if (plain) {
      stp.icon = true;
      let p = el.parentElement;
      for (let k = 0; k < 3 && p; k++, p = p.parentElement) {
        const n = p.querySelector("input:not([type=hidden]):not([type=checkbox]):not([type=radio]):not([type=submit]):not([type=button]):not([type=image])");
        if (n && n !== el) { stp.near = describe(n); break; }
      }
    }
    push(stp);
    scanSoon();
  };
  const KEYS = ["Enter", "Escape", "ArrowDown", "ArrowUp"];
  const onKey = (e) => {
    if ((!e.isTrusted && !W.__paAllowSynthetic) || !KEYS.includes(e.key)) return;
    const el = e.target;
    if (!el || !el.tagName) return;
    if (e.key === "Enter" && !(el.tagName === "INPUT" || el.getAttribute("role") === "combobox" || el.getAttribute("role") === "textbox")) return;
    flush(el);
    push({ t: "key", key: e.key, d: describe(el) });
  };
  const onBlurCE = (e) => {
    const el = e.target;
    if (el && el.isContentEditable && el.tagName) {
      const v = clean(el.textContent);
      if (el.__paLast !== v && v) { el.__paLast = v; push({ t: "type", d: describe(el), value: v, sensitive: false }); }
    }
  };
  document.addEventListener("change", onChange, true);
  document.addEventListener("click", onClick, true);
  document.addEventListener("keydown", onKey, true);
  document.addEventListener("focusout", onBlurCE, true);
  R.off.push(() => {
    document.removeEventListener("change", onChange, true);
    document.removeEventListener("click", onClick, true);
    document.removeEventListener("keydown", onKey, true);
    document.removeEventListener("focusout", onBlurCE, true);
  });

  // ---- network: remember requests that change data (POST/PUT/PATCH/DELETE)
  const keepHeaders = (h) => {
    const out = {};
    try {
      const put = (k, v) => { k = String(k).toLowerCase(); if (k === "content-type" || k === "accept" || k.startsWith("x-")) out[k] = String(v); };
      if (!h) return out;
      if (typeof h.forEach === "function" && !Array.isArray(h)) h.forEach((v, k) => put(k, v));
      else if (Array.isArray(h)) h.forEach(([k, v]) => put(k, v));
      else Object.keys(h).forEach((k) => put(k, h[k]));
    } catch (e) {}
    return out;
  };
  const note = (method, url, headers, body) => {
    method = String(method || "GET").toUpperCase();
    if (["GET", "HEAD", "OPTIONS"].includes(method) || !R.on) return null;
    let abs = url;
    try { abs = new URL(url, location.href).href; } catch (e) {}
    let b = null;
    if (typeof body === "string") b = body.slice(0, 20000);
    else if (body && typeof body.toString === "function" && body instanceof URLSearchParams) b = body.toString();
    const r = { method, url: abs, headers: keepHeaders(headers), body: b, opaque: body != null && b == null, at: Date.now(), status: 0 };
    R.reqs.push(r);
    if (R.reqs.length > 80) R.reqs.shift();
    save();
    return r;
  };
  if (!W.__paNetPatched) {
    W.__paNetPatched = true;
    const of = W.fetch;
    W.fetch = function (input, init) {
      let rec = null;
      try {
        const f = W.__paNoteFetch;
        if (f) {
          const isReq = typeof Request !== "undefined" && input instanceof Request;
          rec = f(isReq ? input.method : init && init.method, isReq ? input.url : String(input), (init && init.headers) || (isReq ? input.headers : null), init && init.body);
        }
      } catch (e) {}
      const p = of.apply(this, arguments);
      if (rec) p.then((resp) => { rec.status = resp.status; }, () => {});
      return p;
    };
    const XO = XMLHttpRequest.prototype.open, XS = XMLHttpRequest.prototype.send, XH = XMLHttpRequest.prototype.setRequestHeader;
    XMLHttpRequest.prototype.open = function (m, u) { this.__pa = { m, u, h: {} }; return XO.apply(this, arguments); };
    XMLHttpRequest.prototype.setRequestHeader = function (k, v) { if (this.__pa) this.__pa.h[k] = v; return XH.apply(this, arguments); };
    XMLHttpRequest.prototype.send = function (body) {
      try {
        const f = W.__paNoteFetch;
        const rec = this.__pa && f ? f(this.__pa.m, this.__pa.u, this.__pa.h, body) : null;
        if (rec) this.addEventListener("loadend", () => { rec.status = this.status; });
      } catch (e) {}
      return XS.apply(this, arguments);
    };
  }
  W.__paNoteFetch = note;
  return { ok: true };
}

// Finds one recorded step's element on the current page and performs it. step = { t, d, value?, key? }.
async function replayStepFn(step, value, timeoutMs) {
  const clean = (s) => (s || "").replace(/\s+/g, " ").trim();
  const norm = (s) => clean(s).toLowerCase();
  const txt = (el) => clean(el.innerText !== undefined ? el.innerText : el.textContent);
  const okId = (id) => !!id && id.length < 60 && !/\d{3,}|^:|[:\s]/.test(id);
  const labelOf = (el) => {
    let l = el.getAttribute("aria-label");
    const lb = el.getAttribute("aria-labelledby");
    if (!l && lb) l = lb.split(" ").map((i) => { const n = document.getElementById(i); return n ? txt(n) : ""; }).join(" ");
    if (!l && el.labels && el.labels[0]) l = txt(el.labels[0]);
    if (!l && ["INPUT", "TEXTAREA", "SELECT"].includes(el.tagName)) l = el.placeholder || el.name || (okId(el.id) ? el.id : "");
    if (!l && el.tagName === "INPUT" && ["submit", "button"].includes(el.type)) l = el.value;
    if (!l) { const img = el.querySelector && el.querySelector("img[alt]"); l = txt(el) || el.title || el.alt || (img && img.alt) || ""; }
    return clean(l).slice(0, 80);
  };
  const visible = (el) => {
    const r = el.getBoundingClientRect();
    if (r.width < 2 || r.height < 2) return false;
    const cs = getComputedStyle(el);
    return cs.visibility !== "hidden" && cs.display !== "none";
  };
  const deepQ = (sel) => {
    const out = [...document.querySelectorAll(sel)];
    const stack = [document];
    while (stack.length) {
      const r = stack.pop();
      r.querySelectorAll("*").forEach((e) => { if (e.shadowRoot) { e.shadowRoot.querySelectorAll(sel).forEach((x) => out.push(x)); stack.push(e.shadowRoot); } });
    }
    return out;
  };
  const d = step.d || {};
  const find = () => {
    const attr = (n, v) => '[' + n + '="' + String(v).replace(/"/g, '\\"') + '"]';
    // exact identifiers first
    if (d.testid) {
      const q = deepQ(["data-testid", "data-test", "data-qa", "data-cy"].map((n) => attr(n, d.testid)).join(",")).filter(visible);
      if (q.length === 1) return q[0];
    }
    if (d.id) { const e = document.getElementById(d.id); if (e && visible(e)) return e; }
    // otherwise score the candidates of the same tag/role
    const sel = [d.tag || "*"].concat(d.role ? ['[role="' + d.role + '"]'] : []).join(",");
    const nl = norm(d.label), nc = norm(d.ctx).slice(0, 24);
    let best = [], bs = 0;
    for (const el of deepQ(sel).slice(0, 4000)) {
      let s = 0;
      if (d.testid && (el.getAttribute("data-testid") === d.testid || el.getAttribute("data-test") === d.testid)) s += 10;
      if (d.name && el.getAttribute("name") === d.name) s += 5;
      if (d.id && el.id === d.id) s += 8;
      const l = nl ? norm(labelOf(el)) : "";
      if (nl && l === nl) s += 6; else if (nl && l && (l.startsWith(nl) || nl.startsWith(l))) s += 3;
      if (d.placeholder && el.getAttribute("placeholder") === d.placeholder) s += 2;
      if (d.href && el.tagName === "A") { try { if (new URL(el.href).pathname === d.href) s += 4; } catch (e) {} }
      if (s < 4) continue;
      if (d.type && el.type === d.type) s += 1;
      if (nc) { const row = el.closest("tr,li,[role=row],[role=listitem],[role=dialog],dialog,form"); if (row && norm(txt(row)).includes(nc)) s += 3; }
      if (!visible(el)) continue;
      if (s > bs) { bs = s; best = [el]; } else if (s === bs) best.push(el);
    }
    if (best.length) return best[Math.min(d.nth || 0, best.length - 1)] || best[0];
    if (d.css) { try { const e = document.querySelector(d.css); if (e && visible(e) && (step.loose || !nl || norm(labelOf(e)) === nl || !labelOf(e))) return e; } catch (e) {} }
    return null;
  };
  // Dynamic target: the element is named after a parameter value (a typeahead suggestion, a row) rather than the recorded label.
  const findDyn = () => {
    const toks = norm(d.dynValue).split(" ").filter(Boolean);
    if (!toks.length) return null;
    const want = norm(d.dynValue);
    let best = null, bs = -1e9;
    for (const el of deepQ('[role=option],[role=menuitem],[role=treeitem],[role=row],[role=link],[role=button],li,a,button,div,span,td,tr,p,label,h1,h2,h3,h4').slice(0, 8000)) {
      const text = norm(txt(el));
      if (!text || text.length > 220 || !toks.every((t) => text.includes(t))) continue;
      if (!visible(el)) continue;
      let s = 10 - Math.max(0, text.length - want.length) / 15;
      if (text === want) s += 4;
      if (el.closest('[role=listbox],[role=menu],[role=tree],[class*="suggest" i],[class*="option" i],[class*="dropdown" i],[class*="autocomplete" i],[class*="popup" i],[class*="result" i]')) s += 3;
      if (el.tagName.toLowerCase() === d.tag) s += 2;
      if (d.role && el.getAttribute("role") === d.role) s += 2;
      if (d.ctx) { const row = el.closest("tr,li,[role=row],[role=listitem],[role=dialog],dialog,form"); if (row && norm(txt(row)).includes(norm(d.ctx).slice(0, 24))) s += 1; }
      if (s > bs) { bs = s; best = el; }
    }
    return best;
  };
  const t0 = Date.now(), limit = timeoutMs || 6000;
  const bare = step.t === "key" && !d.tag && !d.css;
  // A day in an open calendar popup: the cell whose whole text is the day number, not a greyed-out day of another month.
  const findDay = () => {
    const want = String(d.label);
    const cells = deepQ("td,th,button,a,span,div,li,[role=gridcell]").filter((e) => clean(e.textContent) === want && ![...e.children].some((c) => clean(c.textContent) === want)).filter(visible);
    const good = cells.filter((e) => {
      let n = e, cls = "";
      for (let k = 0; k < 2 && n; k++, n = n.parentElement) cls += " " + (n.className && n.className.baseVal !== undefined ? n.className.baseVal : n.className) + " " + (n.getAttribute("aria-disabled") === "true" ? "disabled" : "");
      return !/(^|[\s_-])(old|new|disabled|outside|other|muted|off|prev|next|adjacent)([\s_-]|$)|other-?month|out-of-?month/i.test(cls) && !e.disabled;
    });
    const inCal = (arr) => arr.filter((e) => e.closest("table,[role=grid],[class*=calendar i],[class*=datepicker i],[class*=picker i],[class*=ui-datepicker i]"));
    return inCal(good)[0] || inCal(cells)[0] || good[0] || null;
  };
  const look = () => (bare ? document.activeElement || document.body : d.daySel ? findDay() || find() : d.dynValue ? findDyn() : find());
  let el = look();
  while (!el && Date.now() - t0 < limit) { await new Promise((r) => setTimeout(r, 200)); el = look(); }
  if (!el) return { ok: false, msg: 'Could not find "' + (d.label || d.css || d.tag) + '" (' + step.t + ") on this page." };
  if (el.scrollIntoView) { try { el.scrollIntoView({ block: "center", inline: "center" }); } catch (e) {} }
  const press = (t) => {
    try {
      const r = t.getBoundingClientRect();
      const o = { bubbles: true, cancelable: true, view: window, clientX: r.left + r.width / 2, clientY: r.top + r.height / 2, button: 0 };
      if (typeof PointerEvent === "function") t.dispatchEvent(new PointerEvent("pointerdown", { ...o, pointerType: "mouse", isPrimary: true }));
      t.dispatchEvent(new MouseEvent("mousedown", o));
      if (typeof PointerEvent === "function") t.dispatchEvent(new PointerEvent("pointerup", { ...o, pointerType: "mouse", isPrimary: true }));
      t.dispatchEvent(new MouseEvent("mouseup", o));
    } catch (e) {}
    t.click();
  };
  const fire = (e) => { e.dispatchEvent(new Event("input", { bubbles: true })); e.dispatchEvent(new Event("change", { bubbles: true })); };
  try {
    if (step.t === "click") {
      if (el.disabled) return { ok: false, msg: '"' + (d.label || "element") + '" is disabled.' };
      if (step.kind === "checkbox" && step.want != null) {
        const ctl = el.tagName === "INPUT" ? el : el.control || el.querySelector("input[type=checkbox]");
        if (ctl && ctl.checked === !!step.want) return { ok: true, msg: "Already " + (step.want ? "checked" : "unchecked") + "." };
      }
      if (el.focus) el.focus({ preventScroll: true });
      if (!(step.icon && step.near)) { press(el); return { ok: true, msg: "Clicked." }; }
      // A calendar icon: the handler may sit on a neighbour. Click it; if the page did not react, try the next candidate (never re-click the same one).
      let muts = 0;
      const mo = new MutationObserver((l) => { for (const m of l) { if (m.target === el || /^(INPUT|TEXTAREA)$/.test(m.target.tagName)) continue; if (m.type === "childList" || /^(style|class|hidden|aria-expanded)$/.test(m.attributeName)) muts++; } });
      mo.observe(document.documentElement, { subtree: true, childList: true, attributes: true });
      const wait = (ms) => new Promise((r) => setTimeout(r, ms));
      const tried = new Set([el]);
      const attempt = async (t) => {
        if (!t || tried.has(t) || !t.tagName) return false;
        tried.add(t);
        if (t.focus) { try { t.focus({ preventScroll: true }); } catch (e) {} }
        press(t);
        await wait(300);
        return muts > 0;
      };
      press(el);
      await wait(300);
      let opened = muts > 0, how = "icon";
      if (!opened) {
        const r = el.getBoundingClientRect();
        let hit = null;
        try { hit = document.elementFromPoint(r.left + r.width / 2, r.top + r.height / 2); } catch (e) {}
        if (await attempt(hit)) { opened = true; how = "element at that point"; }
      }
      if (!opened) {
        let p = el.parentElement;
        for (let k = 0; k < 2 && p && !opened; k++, p = p.parentElement) if (await attempt(p)) { opened = true; how = "container"; }
      }
      if (!opened) {
        const nd = step.near;
        let near = null;
        if (nd.id) near = document.getElementById(nd.id);
        if (!near && nd.name) near = document.querySelector('input[name="' + String(nd.name).replace(/"/g, '\\"') + '"]');
        if (!near && nd.css) { try { near = document.querySelector(nd.css); } catch (e) {} }
        if (!near) { let p = el.parentElement; for (let k = 0; k < 3 && p && !near; k++, p = p.parentElement) near = p.querySelector("input:not([type=hidden]):not([type=checkbox]):not([type=radio]):not([type=submit]):not([type=button]):not([type=image])"); }
        if (near && await attempt(near)) { opened = true; how = "the date field"; }
      }
      mo.disconnect();
      return { ok: true, msg: opened ? "Clicked (" + how + ")." : "Clicked, no visible reaction." };
    }
    if (step.t === "type") {
      if (el.focus) el.focus();
      const v = value == null ? step.value || "" : String(value);
      const isTA = el instanceof HTMLTextAreaElement;
      if (isTA || el instanceof HTMLInputElement) {
        const proto = isTA ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
        Object.getOwnPropertyDescriptor(proto, "value").set.call(el, v);
        fire(el);
        if (el.value !== v) return { ok: false, msg: "The field did not accept the value." };
        const lastCh = v.slice(-1) || " ";
        for (const ty of ["keydown", "keypress", "keyup"]) el.dispatchEvent(new KeyboardEvent(ty, { key: lastCh, bubbles: true, cancelable: true }));
      } else if (el.isContentEditable) {
        document.execCommand("selectAll", false);
        document.execCommand("insertText", false, v);
      } else return { ok: false, msg: "Not a text field." };
      if (step.ro && el.blur) { try { el.blur(); } catch (e) {} }
      return { ok: true, msg: "Typed." };
    }
    if (step.t === "select") {
      if (el.tagName !== "SELECT") return { ok: false, msg: "Not a <select>." };
      if (step.multi || el.multiple) {
        const wants = String(value == null ? step.value : value).split(/\s*(?:;|\n|\|)\s*/).map(norm).filter(Boolean);
        let o2 = [...el.options];
        const findAll = () => wants.map((w) => o2.find((o) => norm(o.text) === w || norm(o.value) === w) || o2.find((o) => norm(o.text).includes(w)));
        let hits = findAll();
        while (hits.some((h) => !h) && Date.now() - t0 < Math.max(limit, 3500)) { await new Promise((r) => setTimeout(r, 200)); o2 = [...el.options]; hits = findAll(); }
        if (hits.some((h) => !h)) return { ok: false, msg: 'No option "' + wants[hits.findIndex((h) => !h)] + '". Available: ' + o2.slice(0, 20).map((o) => clean(o.text)).join(" | ") };
        for (const o of o2) o.selected = hits.includes(o);
        fire(el);
        return { ok: true, msg: "Selected " + hits.length + "." };
      }
      const want = norm(value == null ? step.value : value);
      let opts = [...el.options];
      const pickOpt = () => opts.find((o) => norm(o.text) === want || norm(o.value) === want) || opts.find((o) => norm(o.text).includes(want));
      let opt = pickOpt();
      // dependent dropdowns fill their options after the previous choice: wait for it
      while (!opt && Date.now() - t0 < Math.max(limit, 3500)) { await new Promise((r) => setTimeout(r, 200)); opts = [...el.options]; opt = pickOpt(); }
      if (!opt) return { ok: false, msg: 'No option "' + want + '". Available: ' + opts.slice(0, 20).map((o) => clean(o.text)).join(" | ") };
      Object.getOwnPropertyDescriptor(HTMLSelectElement.prototype, "value").set.call(el, opt.value);
      fire(el);
      return { ok: true, msg: "Selected." };
    }
    if (step.t === "key") {
      if (el.focus) el.focus();
      for (const ty of ["keydown", "keypress", "keyup"]) el.dispatchEvent(new KeyboardEvent(ty, { key: step.key, code: step.key, bubbles: true, cancelable: true }));
      if (step.key === "Enter" && el.form && el.tagName === "INPUT" && el.form.requestSubmit) { try { el.form.requestSubmit(); } catch (e) {} }
      return { ok: true, msg: "Pressed " + step.key + "." };
    }
    return { ok: false, msg: "Unknown step type " + step.t };
  } catch (e) {
    return { ok: false, msg: String(e && e.message || e) };
  }
}

// A cheap fingerprint of the page, to check that a step really changed something. Besides the text and element count it includes
// state that does not show in the text: field values, checked boxes, open menus and dialogs.
function pageSigFn() {
  const t = (document.body ? document.body.innerText || document.body.textContent || "" : "").slice(0, 30000);
  let h = 5381;
  const mix = (str) => { for (let i = 0; i < str.length; i++) h = ((h << 5) + h + str.charCodeAt(i)) | 0; };
  mix(t);
  let st = "";
  try {
    document.querySelectorAll("input,select,textarea").forEach((e, i) => {
      if (i > 300 || e.type === "password") return;
      st += (e.type === "checkbox" || e.type === "radio" ? (e.checked ? "1" : "0") : e.tagName === "SELECT" ? e.selectedIndex : String(e.value).slice(0, 40)) + "|";
    });
    st += document.querySelectorAll('[aria-expanded="true"],[aria-selected="true"],[aria-checked="true"],dialog[open],[role=dialog],[role=listbox],[role=menu]').length;
  } catch (e) {}
  mix(st);
  return location.href + "|" + (document.body ? document.body.getElementsByTagName("*").length : 0) + "|" + h;
}

// Sends one recorded request again, from the page (same cookies / origin as the app).
async function fetchFn(req) {
  try {
    const r = await fetch(req.url, { method: req.method, headers: req.headers || {}, body: req.body == null ? undefined : req.body, credentials: "include" });
    const text = await r.text();
    return { ok: r.ok, status: r.status, text: text.slice(0, 800) };
  } catch (e) {
    return { ok: false, status: 0, text: String(e && e.message || e) };
  }
}


// For the debug report: what form controls and open popups the page has right now (no passwords / secrets).
function fieldsFn() {
  const clean = (x) => (x || "").replace(/\s+/g, " ").trim();
  const vis = (e) => { const r = e.getBoundingClientRect(); if (r.width < 2 || r.height < 2) return false; const cs = getComputedStyle(e); return cs.visibility !== "hidden" && cs.display !== "none"; };
  const lab = (e) => clean(e.getAttribute("aria-label") || (e.labels && e.labels[0] && e.labels[0].textContent) || e.placeholder || e.name || e.id || "").slice(0, 60);
  const out = { url: location.href, title: document.title, controls: [], popups: [] };
  for (const e of [...document.querySelectorAll("input,select,textarea")].slice(0, 400)) {
    if (e.type === "hidden" || !vis(e)) continue;
    const c = { tag: e.tagName.toLowerCase(), type: e.type || "", label: lab(e), name: e.name || "", id: e.id || "" };
    if (e.readOnly) c.readonly = true;
    if (e.disabled) c.disabled = true;
    if (e.tagName === "SELECT") { c.multiple = e.multiple; c.options = [...e.options].slice(0, 40).map((o) => (o.selected ? "[x] " : "") + clean(o.text)); }
    else if (e.type === "checkbox" || e.type === "radio") c.checked = e.checked;
    else if (e.type === "password" || /otp|cc-|card|cvv|ssn/i.test((e.autocomplete || "") + e.name + e.id)) c.value = "(hidden)";
    else c.value = clean(e.value).slice(0, 80);
    out.controls.push(c);
    if (out.controls.length >= 80) break;
  }
  for (const e of [...document.querySelectorAll('[role=listbox],[role=menu],[role=dialog],[class*=dropdown-menu i],[class*=datepicker i],[class*=calendar i],[class*=popup i],[class*=popover i]')].slice(0, 60)) {
    if (!vis(e)) continue;
    out.popups.push({ tag: e.tagName.toLowerCase(), role: e.getAttribute("role") || "", cls: String(e.className && e.className.baseVal !== undefined ? e.className.baseVal : e.className).slice(0, 60), text: clean(e.innerText || e.textContent).slice(0, 160) });
    if (out.popups.length >= 10) break;
  }
  return out;
}



















//manifest.json
{
  "manifest_version": 3,
  "name": "Page Agent",
  "version": "1.1.2",
  "description": "Ask questions about the current page, or tell it to do things (fill forms, create records, book tickets), now or later, with approval before final actions.",
  "permissions": [
    "tabs",
    "scripting",
    "storage",
    "sidePanel",
    "alarms",
    "notifications"
  ],
  "host_permissions": [
    "<all_urls>",
    "https://api.anthropic.com/*"
  ],
  "background": {
    "service_worker": "background.js"
  },
  "side_panel": {
    "default_path": "sidepanel.html"
  },
  "options_page": "options.html",
  "action": {
    "default_title": "Page Agent",
    "default_icon": {
      "16": "icons/icon16.png",
      "32": "icons/icon32.png",
      "48": "icons/icon48.png"
    }
  },
  "icons": {
    "16": "icons/icon16.png",
    "32": "icons/icon32.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  }
}
