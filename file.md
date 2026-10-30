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
      push({ t: "select", d: describe(el), value: o ? clean(o.text) : el.value, choices: [...el.options].map((x) => clean(x.text)).filter(Boolean).slice(0, 60) });
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
    // A choice from a dropdown / list / menu (custom widgets, not <select>): remember it so it can become an input.
    const box = el.closest('[role=listbox],[role=menu],[role=tree],[role=combobox],[class*=dropdown i],[class*=select i],[class*=autocomplete i],[class*=suggest i],[class*=popup i],[class*=popover i],[class*=overlay i],[class*=options i],[class*=menu i],[class*=chosen i],[class*=choices i],[class*=listbox i]');
    if ((el.matches('[role=option],[role=menuitem],[role=treeitem],li,option') || (box && box !== el)) && box && !el.closest("table,[role=grid],[class*=calendar i],[class*=datepicker i]") && stp.d.label && stp.d.label.length < 70) {
      const prevC = R.steps[R.steps.length - 1];
      if (prevC && prevC.t === "click" && Date.now() - prevC.at < 20000) {
        stp.pick = true;
        const sibs = el.parentElement ? [...el.parentElement.children].filter((x) => x.tagName === el.tagName) : [];
        stp.choices = (sibs.length > 1 ? sibs : [...box.querySelectorAll(el.tagName)]).map((x) => clean(x.innerText !== undefined ? x.innerText : x.textContent)).filter((x) => x && x.length < 70).slice(0, 60);
      }
    }
    { const fl = fieldLabel(el); if (fl) stp.fl = fl; }
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

// A cheap fingerprint of the page, to check that a step really changed something.
function pageSigFn() {
  const t = (document.body ? document.body.innerText || document.body.textContent || "" : "").slice(0, 30000);
  let h = 5381;
  for (let i = 0; i < t.length; i++) h = ((h << 5) + h + t.charCodeAt(i)) | 0;
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














//skills.js
// Skills: record a task once (by doing it, or save a successful agent run), then replay it fast and deterministically.
// Loaded after sidepanel.js (uses its globals: $, T, i18n, addMessage, requestApproval, settle, takeSnapshot, ...).
// Page-side helpers (recorderFn, replayStepFn, pageSigFn, fetchFn) live in inpage.js.

let SKILLS = [];
let recording = null; // { tabId, host, startUrl, timer, onUpdated } while the user is recording
let actionLog = []; // successful actions of the current agent run (so a good run can be saved as a skill)
let lastRequestText = "";
let skillRunUsed = false;
let agentHost = "";

const hostOf = (u) => { try { return new URL(u).host; } catch (e) { return ""; } };
const stripHash = (u) => String(u || "").split("#")[0];
const slug = (s) => (String(s || "").toLowerCase().replace(/[^a-z0-9À-￿]+/g, "_").replace(/^_+|_+$/g, "") || "value").slice(0, 30);

async function skillsLoad() {
  try { const { skills } = await chrome.storage.local.get("skills"); SKILLS = Array.isArray(skills) ? skills : []; } catch (e) { SKILLS = []; }
  return SKILLS;
}
async function skillsSave() {
  try { await chrome.storage.local.set({ skills: SKILLS }); } catch (e) {}
  renderSkills();
}
const skillsForHost = (h) => SKILLS.filter((s) => s.host === h);

// ---------- what the model is told ----------
function skillsPrompt() {
  const list = [...skillsForHost(agentHost), ...SKILLS.filter((s) => s.host !== agentHost)].slice(0, 25);
  if (!list.length) return "";
  return (
    "\n\nSaved skills. When one matches the request, call run_skill with its name and the parameter values you know from the user's request (omit unknown ones: the user is shown a form to confirm or fill in the inputs before it runs). It replays a recorded, proven path and is faster and more reliable than doing the steps yourself. A skill for another site runs automatically in that site's tab (it is found or opened for you), so you may use it from any tab. If a skill fails, continue manually from where it stopped.\n" +
    list.map((s) => `- ${s.name}${s.host !== agentHost ? ` [site ${s.host}]` : ""}${s.description ? ": " + s.description : ""}. Parameters: ${s.params.length ? s.params.map((p) => `${p.name}${p.label ? ` (${p.label})` : ""}${p.default ? ` default "${String(p.default).slice(0, 40)}"` : ""}${p.choices && p.choices.length ? ` options: ${p.choices.slice(0, 12).join(" | ")}` : ""}`).join(", ") : "none"}`).join("\n")
  );
}
const RUN_SKILL_TOOL = {
  name: "run_skill",
  description: "Run a saved skill (a recorded sequence of steps for this site). Give the skill name and a params object with a value for each parameter.",
  input_schema: { type: "object", properties: { name: { type: "string" }, params: { type: "object" } }, required: ["name"] },
};
const skillTools = () => (SKILLS.length ? [RUN_SKILL_TOOL] : []);

// ---------- describing steps ----------
const lbl = (d) => (d && (d.dynValue || d.label || d.placeholder || d.name || d.id || d.testid)) || (d && d.tag) || "";
function stepText(st, val) {
  const label = lbl(st.d);
  const v = val != null ? val : st.value;
  if (st.t === "click") return i18n("sk.click", { label });
  if (st.t === "type") return st.sensitive ? i18n("sk.typeSecret", { label }) : i18n("sk.type", { value: v, label });
  if (st.t === "select") return i18n("sk.select", { value: v, label });
  if (st.t === "key") return i18n("sk.key", { key: st.key, label });
  return st.t;
}
const stepIsFinal = (st) => st.t === "click" ? (isRiskyLabel(lbl(st.d)) || (st.d && st.d.type === "submit")) : st.t === "key" && st.key === "Enter";

// ---------- recording ----------
function recBarShow(n) {
  const bar = $("recBar");
  bar.hidden = false;
  $("recText").textContent = i18n("sk.recording", { n: n || 0 });
}
async function startRecording() {
  if (running || recording) return;
  const tab = await getActiveTab();
  if (!tab || !/^https?:/.test(tab.url || "")) { addMessage("err", i18n("err.unreadable")); return; }
  const hint = document.querySelector(".hint");
  if (hint) hint.remove();
  agentTabId = tab.id;
  recording = { tabId: tab.id, host: hostOf(tab.url), startUrl: tab.url };
  try {
    await chrome.scripting.executeScript({ target: { tabId: tab.id }, func: recorderFn, args: [{ cmd: "start" }], world: "MAIN" });
  } catch (e) { recording = null; addMessage("err", e.message); return; }
  recording.onUpdated = (id, info) => {
    if (recording && id === recording.tabId && info.status === "complete") {
      chrome.scripting.executeScript({ target: { tabId: id }, func: recorderFn, args: [{ cmd: "start", resume: true }], world: "MAIN" }).catch(() => {});
    }
  };
  chrome.tabs.onUpdated.addListener(recording.onUpdated);
  recording.timer = setInterval(async () => {
    try {
      const [r] = await chrome.scripting.executeScript({ target: { tabId: recording.tabId }, func: recorderFn, args: [{ cmd: "dump" }], world: "MAIN" });
      recBarShow(r.result && r.result.steps ? r.result.steps.length : 0);
    } catch (e) {}
  }, 1500);
  recBarShow(0);
  addMessage("note", i18n("sk.recStarted"));
}

async function stopRecording() {
  if (!recording) return;
  const rec = recording;
  recording = null;
  clearInterval(rec.timer);
  try { chrome.tabs.onUpdated.removeListener(rec.onUpdated); } catch (e) {}
  $("recBar").hidden = true;
  let data = null;
  try {
    const tid = rec.tabId;
    const [r] = await chrome.scripting.executeScript({ target: { tabId: tid }, func: recorderFn, args: [{ cmd: "stop" }], world: "MAIN" });
    data = r.result;
  } catch (e) { addMessage("err", e.message); return; }
  if (!data || !data.steps || !data.steps.length) { addMessage("note", i18n("sk.recEmpty")); return; }
  const draft = draftFromRecording(data, rec);
  T("skill-recorded", { steps: draft.steps.length, params: draft.params.length, api: !!draft.api });
  openSkillEditor(draft);
}

// Turn the recorded events into a skill draft: parameters from typed values, plus an optional direct-request fast path.
function draftFromRecording(data, rec) {
  const steps = data.steps.map((s) => ({ t: s.t, d: s.d, value: s.value, key: s.key, sensitive: !!s.sensitive, effect: s.effect, at: s.at, url: s.url, icon: s.icon, near: s.near, ro: s.ro, pick: s.pick, choices: s.choices, fl: s.fl }));
  const params = [];
  const used = new Set();
  for (const st of steps) {
    if ((st.t === "type" || st.t === "select") && !st.sensitive && st.value) {
      const pl = (st.d && st.d.fl) || lbl(st.d);
      let n = slug(pl), k = 2;
      while (used.has(n)) n = slug(pl) + "_" + k++;
      used.add(n);
      st.param = n;
      params.push({ name: n, label: pl, default: st.value, choices: st.choices });
    }
  }
  // Date pickers: clicks that lead up to a value appearing in a read-only field (open the picker, pick a day) are best-effort at replay;
  // the field itself is then set from the parameter, so a picker that cannot be driven does not break the skill.
  steps.forEach((st, j) => {
    if (!(st.t === "type" && st.ro && st.param)) return;
    const nums = String(st.value).match(/\d+/g) || [];
    for (let k = j - 1, prevAt = st.at; k >= 0 && steps[k].t === "click" && prevAt - steps[k].at < 20000; prevAt = steps[k].at, k--) {
      steps[k].opt = true;
      const lab = String(steps[k].d && steps[k].d.label || "").trim();
      if (/^\d{1,2}$/.test(lab)) {
        const idx = nums.findIndex((n) => Number(n) === Number(lab));
        if (idx >= 0) { steps[k].dayOf = st.param; steps[k].dayIdx = idx; steps[k].d.daySel = true; }
      }
    }
  });
  const draft = { id: "sk_" + Date.now().toString(36), name: "", description: "", host: rec.host, startUrl: rec.startUrl, params, steps, api: null, useApi: true, created: Date.now(), uses: 0, ok: 0, fail: 0 };
  // Choices made in custom dropdowns / lists (not typed, not <select>): each one is an input too.
  steps.forEach((st, i) => {
    if (!(st.t === "click" && st.pick && !st.opt && !st.dayOf && !st.param)) return;
    if (dynLink(draft, i).label || dynLink(draft, i).ctx) return; // a typeahead suggestion: already follows what was typed
    const opener = i > 0 && steps[i - 1].t === "click" ? steps[i - 1] : null;
    const label = (opener && (opener.fl || lbl(opener.d))) || st.fl || "Option";
    let n = slug(label) || "option", k = 2;
    while (used.has(n)) n = (slug(label) || "option") + "_" + k++;
    used.add(n);
    st.param = n;
    params.push({ name: n, label: String(label).slice(0, 40), default: st.d.label, choices: st.choices });
    if (opener) opener.loose = true; // the opener often shows the current value, which will differ next time
  });
  draft.api = detectApi(data, steps, params, rec.startUrl);
  return draft;
}

const BEACON = /collect|analytics|telemetry|track|beacon|metric|sentry|segment|logging|\/log\b|gtm|pixel/i;
function detectApi(data, steps, params, startUrl) {
  let origin = "";
  try { origin = new URL(startUrl).origin; } catch (e) {}
  const reqs = (data.reqs || []).filter((r) => {
    try { return new URL(r.url).origin === origin; } catch (e) { return false; }
  }).filter((r) => !r.opaque && !BEACON.test(r.url) && r.status >= 200 && r.status < 300);
  if (!reqs.length) return null;
  const lastStepAt = steps.length ? steps[steps.length - 1].at : 0;
  // the data-changing request closest to the end of the recording is the "do it" request
  const req = reqs[reqs.length - 1];
  if (lastStepAt && req.at < steps[0].at) return null;
  const ct = (req.headers["content-type"] || "").toLowerCase();
  let body = req.body || "";
  let url = req.url;
  const found = new Set();
  const variants = (v) => {
    const out = [];
    if (ct.includes("json")) out.push(JSON.stringify(v).slice(1, -1));
    else if (ct.includes("x-www-form-urlencoded")) out.push(encodeURIComponent(v).replace(/%20/g, "+"), encodeURIComponent(v));
    out.push(v);
    return out;
  };
  for (const p of params) {
    if (String(p.default).length < 3) return null; // too short to locate reliably
    let hit = false;
    for (const vv of variants(p.default)) {
      if (vv && body.includes(vv)) { body = body.split(vv).join("{{" + p.name + "}}"); hit = true; break; }
    }
    if (!hit) {
      const enc = encodeURIComponent(p.default);
      if (url.includes(enc)) { url = url.split(enc).join("{{" + p.name + "}}"); hit = true; }
    }
    if (!hit) return null; // a parameter the request does not use: cannot be replayed directly
    found.add(p.name);
  }
  return { method: req.method, url, headers: req.headers, body: body || null, contentType: ct };
}

function fillTokens(str, values, enc) {
  return String(str).replace(/\{\{([^}]+)\}\}/g, (m, k) => (k in values ? enc(String(values[k])) : m));
}
function buildRequest(api, values) {
  const ct = api.contentType || "";
  const bodyEnc = ct.includes("json") ? (v) => JSON.stringify(v).slice(1, -1) : ct.includes("x-www-form-urlencoded") ? (v) => encodeURIComponent(v).replace(/%20/g, "+") : (v) => v;
  return { method: api.method, url: fillTokens(api.url, values, encodeURIComponent), headers: api.headers, body: api.body == null ? null : fillTokens(api.body, values, bodyEnc) };
}

// ---------- saving a successful agent run as a skill ----------
function draftFromRun(host, startUrl) {
  const steps = [];
  const params = [];
  const used = new Set();
  const req = (lastRequestText || "").toLowerCase();
  const addParam = (st, val) => {
    if (!val || String(val).length < 2 || !req.includes(String(val).toLowerCase())) return;
    let n = slug(lbl(st.d)), k = 2;
    while (used.has(n)) n = slug(lbl(st.d)) + "_" + k++;
    used.add(n);
    st.param = n;
    params.push({ name: n, label: lbl(st.d), default: String(val) });
  };
  for (const a of actionLog) {
    const d = a.el && a.el.d ? a.el.d : null;
    if (a.name === "click" && d) {
      const prev = steps[steps.length - 1];
      if (prev && prev.t === "click" && lbl(prev.d) === lbl(d) && prev.d.css === d.css) continue; // an immediate repeat is noise
      steps.push({ t: "click", d });
    } else if (a.name === "type_text" && d) {
      const st = { t: "type", d, value: a.input.text, sensitive: false };
      addParam(st, a.input.text);
      steps.push(st);
      if (a.input.submit) steps.push({ t: "key", key: "Enter", d });
    } else if (a.name === "select_option" && d) {
      const st = { t: "select", d, value: a.input.option };
      addParam(st, a.input.option);
      steps.push(st);
    } else if (a.name === "press_key" && d !== undefined) {
      steps.push({ t: "key", key: a.input.key, d: d || {} });
    }
  }
  return { id: "sk_" + Date.now().toString(36), name: "", description: (lastRequestText || "").slice(0, 140), host, startUrl, params, steps, api: null, useApi: false, created: Date.now(), uses: 0, ok: 0, fail: 0 };
}
function offerSaveSkill(host, startUrl) {
  if (RUNNER || skillRunUsed || actionLog.length < 2) return;
  const div = addMessage("approval", "");
  const t = document.createElement("div");
  t.className = "title";
  t.textContent = i18n("sk.offer");
  const btns = document.createElement("div");
  btns.className = "btns";
  const yes = document.createElement("button");
  yes.className = "yes";
  yes.textContent = i18n("sk.offerYes");
  const no = document.createElement("button");
  no.className = "no";
  no.textContent = i18n("sk.offerNo");
  yes.onclick = () => { div.remove(); const d = draftFromRun(host, startUrl); if (d.steps.length) openSkillEditor(d); };
  no.onclick = () => div.remove();
  btns.append(yes, no);
  div.append(t, btns);
}

// ---------- values that steer later steps (typeahead suggestions, rows named after the value) ----------
// A click on "Project Alpha" right after typing "Alpha" must follow the parameter next time ("Project Beta"), not repeat the recorded label.
const normT = (s) => String(s || "").replace(/\s+/g, " ").trim().toLowerCase();
function valueMatches(text, value) {
  const v = normT(value);
  if (v.length < 2) return false;
  const t = normT(text);
  return v.split(" ").filter(Boolean).every((tok) => t.includes(tok));
}
// For click step i: which earlier parameter does its label / row context follow?
function dynLink(draft, i) {
  const st = draft.steps[i];
  const out = {};
  if (st.t !== "click" || !st.d) return out;
  for (let j = i - 1; j >= 0; j--) {
    const pj = draft.steps[j];
    if (!pj.param) continue;
    const p = draft.params.find((x) => x.name === pj.param);
    if (!p) continue;
    if (!out.label && st.d.label && valueMatches(st.d.label, p.default)) out.label = p.name;
    else if (!out.ctx && st.d.ctx && valueMatches(st.d.ctx, p.default)) out.ctx = p.name;
    if (out.label && out.ctx) break;
  }
  return out;
}
function linkDynamic(draft) {
  draft.steps.forEach((st, i) => {
    if (!st.d) return;
    delete st.d.dyn; delete st.d.ctxDyn;
    if (st.t === "click" && st.param) { st.d.dyn = st.param; return; }
    const l = dynLink(draft, i);
    if (l.label) st.d.dyn = l.label;
    if (l.ctx) st.d.ctxDyn = l.ctx;
  });
}
// The step as it should be searched for on THIS run (current parameter values instead of the recorded ones).
function resolveStep(st, values) {
  const r = JSON.parse(JSON.stringify(st));
  const d = r.d || (r.d = {});
  if (d.dyn && values[d.dyn] != null) {
    d.dynValue = String(values[d.dyn]);
    d.id = d.testid = d.css = d.name = d.href = ""; d.nth = 0;
  }
  if (d.ctxDyn && values[d.ctxDyn] != null) {
    d.ctx = String(values[d.ctxDyn]);
    d.id = d.testid = d.css = ""; d.nth = 0;
  }
  if (r.dayOf && values[r.dayOf] != null) {
    const nums = String(values[r.dayOf]).match(/\d+/g) || [];
    if (nums[r.dayIdx] != null) { d.label = String(Number(nums[r.dayIdx])); d.id = d.testid = d.css = d.name = ""; d.nth = 0; }
  }
  return r;
}

// ---------- editor card ----------
function mkEl(tag, cls, text) {
  const e = document.createElement(tag);
  if (cls) e.className = cls;
  if (text != null) e.textContent = text;
  return e;
}
function openSkillEditor(draft) {
  const card = addMessage("skillcard", "");
  card.append(mkEl("div", "title", i18n("sk.edTitle")));
  const name = mkEl("input");
  name.placeholder = i18n("sk.name");
  name.value = draft.name || "";
  const desc = mkEl("input");
  desc.placeholder = i18n("sk.desc");
  desc.value = draft.description || "";
  card.append(mkEl("label", "", i18n("sk.name")), name, mkEl("label", "", i18n("sk.desc")), desc);
  const list = mkEl("div", "steps");
  const render = () => {
    list.textContent = "";
    draft.steps.forEach((st, i) => {
      const row = mkEl("div", "srow");
      row.append(mkEl("span", "sn", String(i + 1)));
      const lk = dynLink(draft, i);
      row.append(mkEl("span", "stext", stepText(st) + (lk.label || lk.ctx ? " " + i18n("sk.follows", { param: lk.label || lk.ctx }) : "")));
      if ((st.t === "type" || st.t === "select" || (st.t === "click" && (st.pick || st.param))) && !st.sensitive) {
        const cb = mkEl("input");
        cb.type = "checkbox";
        cb.checked = !!st.param;
        cb.title = i18n("sk.param");
        const pn = mkEl("input", "pname");
        pn.value = st.param || slug(lbl(st.d));
        pn.hidden = !st.param;
        cb.onchange = () => {
          if (cb.checked) {
            st.param = pn.value || slug(lbl(st.d));
            if (!draft.params.some((p) => p.name === st.param)) draft.params.push({ name: st.param, label: lbl(st.d), default: st.t === "click" ? st.d.label : st.value, choices: st.choices });
          } else {
            draft.params = draft.params.filter((p) => p.name !== st.param);
            st.param = null;
          }
          draft.api = null; // the direct request was derived with the old parameter set
          render();
        };
        pn.onchange = () => {
          const p = draft.params.find((x) => x.name === st.param);
          const nn = slug(pn.value);
          if (p) p.name = nn;
          st.param = nn;
          pn.value = nn;
        };
        row.append(cb, mkEl("span", "plabel", i18n("sk.param")), pn);
      } else if (st.sensitive) {
        row.append(mkEl("span", "plabel", i18n("sk.secret")));
      }
      const del = mkEl("a", "sdel", "✕");
      del.href = "#";
      del.title = i18n("sk.delete");
      del.onclick = (e) => { e.preventDefault(); const s = draft.steps[i]; draft.steps.splice(i, 1); if (s.param) draft.params = draft.params.filter((p) => p.name !== s.param); draft.api = null; render(); };
      row.append(del);
      list.append(row);
    });
  };
  render();
  card.append(list);
  let apiCb = null;
  if (draft.api) {
    const wrap = mkEl("label", "apiopt");
    apiCb = mkEl("input");
    apiCb.type = "checkbox";
    apiCb.checked = draft.useApi !== false;
    wrap.append(apiCb, document.createTextNode(" " + i18n("sk.useApi")));
    card.append(wrap, mkEl("div", "small", i18n("sk.apiInfo", { req: `${draft.api.method} ${draft.api.url.replace(/^https?:\/\/[^/]+/, "").slice(0, 80)}` })));
  }
  const btns = mkEl("div", "btns");
  const save = mkEl("button", "yes", i18n("sk.save"));
  const cancel = mkEl("button", "no", i18n("sk.cancel"));
  save.onclick = async () => {
    const nm = name.value.trim();
    if (!nm) { name.focus(); name.classList.add("bad"); return; }
    draft.name = nm.slice(0, 60);
    draft.description = desc.value.trim().slice(0, 200);
    draft.useApi = apiCb ? apiCb.checked : false;
    if (!draft.steps.length) { card.remove(); return; }
    // keep each parameter's default in sync with its (possibly edited) step
    draft.params = draft.params.filter((p) => draft.steps.some((s) => s.param === p.name));
    linkDynamic(draft);
    await skillsLoad();
    SKILLS = SKILLS.filter((s) => !(s.host === draft.host && s.name.toLowerCase() === draft.name.toLowerCase()));
    SKILLS.push(draft);
    await skillsSave();
    card.remove();
    addMessage("note", i18n("sk.saved", { name: draft.name }));
  };
  cancel.onclick = () => card.remove();
  btns.append(save, cancel);
  card.append(btns);
}

// ---------- the skills list ----------
function renderSkills() {
  const box = $("skillsList");
  if (!box) return;
  box.textContent = "";
  $("skillsTitle").textContent = i18n("sk.listTitle", { n: SKILLS.length });
  $("skillsBox").hidden = SKILLS.length === 0 && !recording;
  if (!SKILLS.length) return;
  for (const sk of SKILLS) {
    const row = mkEl("div", "skill");
    const top = mkEl("div", "sktop");
    top.append(mkEl("b", "", sk.name), mkEl("span", "skhost", sk.host));
    row.append(top);
    if (sk.description) row.append(mkEl("div", "skdesc", sk.description));
    row.append(mkEl("div", "skmeta", `${i18n("sk.stepsN", { n: sk.steps.length })}${sk.api && sk.useApi ? " · " + i18n("sk.fast") : ""} · ${i18n("sk.uses", { n: sk.uses || 0 })}${sk.fail ? " · " + i18n("sk.failN", { n: sk.fail }) : ""}`));
    const form = mkEl("div", "skform");
    const inputs = {};
    for (const p of sk.params) {
      const lab = mkEl("label", "", p.label || p.name);
      const inp = mkEl("input");
      inp.value = p.default || "";
      if (p.choices && p.choices.length) {
        const dl = mkEl("datalist");
        dl.id = "dll_" + sk.id + "_" + p.name;
        p.choices.forEach((c) => { const o = mkEl("option"); o.value = c; dl.append(o); });
        inp.setAttribute("list", dl.id);
        form.append(dl);
      }
      inputs[p.name] = inp;
      form.append(lab, inp);
    }
    const btns = mkEl("div", "tbtns");
    const run = mkEl("button", "yes", i18n("sk.runNow"));
    run.onclick = () => {
      const values = {};
      for (const p of sk.params) values[p.name] = inputs[p.name].value;
      runSkillFromList(sk, values);
    };
    const del = mkEl("button", "no", i18n("sk.delete"));
    del.onclick = async () => {
      if (!confirm(i18n("sk.deleteConfirm", { name: sk.name }))) return;
      SKILLS = SKILLS.filter((s) => s.id !== sk.id);
      await skillsSave();
    };
    btns.append(run, del);
    row.append(form, btns);
    box.append(row);
  }
}

async function runSkillFromList(sk, values) {
  if (running) return;
  const tab = await getActiveTab();
  if (!tab || !/^https?:/.test(tab.url || "")) { addMessage("err", i18n("err.unreadable")); return; }
  const hint = document.querySelector(".hint");
  if (hint) hint.remove();
  addMessage("user", i18n("sk.running", { name: sk.name }));
  setRunning(true);
  try {
    const target = await resolveSkillTab(sk, tab);
    agentTabId = target.id;
    typedLog = [];
    const res = await executeSkill(sk, values, target);
    addMessage(res.ok ? "note" : "err", res.ok ? i18n("sk.done", { name: sk.name }) + " " + (res.text || "") : res.text);
  } catch (e) {
    addMessage("err", e.message);
  } finally {
    setRunning(false);
  }
}

// A skill belongs to a site, not to the tab you happen to be looking at. Find (or open) that site's tab and work there.
async function resolveSkillTab(sk, tab) {
  if (hostOf(tab.url) === sk.host) return tab;
  let all = [];
  try { all = await chrome.tabs.query({}); } catch (e) {}
  const same = all.filter((t) => hostOf(t.url) === sk.host);
  const target = same.find((t) => stripHash(t.url) === stripHash(sk.startUrl)) || same[0];
  if (target) {
    try { await chrome.tabs.update(target.id, { active: true }); } catch (e) {}
    logLine(i18n("sk.switched", { host: sk.host }));
    return target;
  }
  logLine(i18n("sk.opening", { host: sk.host }));
  const t = await chrome.tabs.create({ url: sk.startUrl, active: true });
  await settle(t.id);
  return await chrome.tabs.get(t.id);
}

// ---------- running a skill ----------
async function doRunSkill(input, tab) {
  await skillsLoad();
  const host = hostOf(tab.url);
  const want = String(input.name || "").trim().toLowerCase();
  const byName = SKILLS.filter((s) => s.id === input.name || s.name.toLowerCase() === want);
  const sk = byName.find((s) => s.host === host) || byName[0];
  if (!sk) return { error: true, text: `No saved skill "${input.name}". Available: ${SKILLS.map((s) => s.name).join(", ") || "(none)"}. Continue manually.` };
  skillRunUsed = true;
  const given = input.params && typeof input.params === "object" ? input.params : {};
  const values = {};
  const missing = [];
  for (const p of sk.params) {
    let v = given[p.name];
    if (v == null || v === "") v = p.default;
    if (v == null || v === "") missing.push(p.name);
    values[p.name] = v == null ? "" : String(v);
  }
  // Ask the user to confirm / fill the inputs before anything runs (skipped for unattended runs, and with auto-approve when nothing is missing).
  if (sk.params.length && !RUNNER && (missing.length || !$("autoApprove").checked)) {
    const got = await askSkillInputs(sk, values, missing);
    if (!got) return { ok: false, denied: true, error: true, text: "The user cancelled running this skill. Do not retry it. Ask how to proceed." };
    Object.assign(values, got);
  } else if (missing.length) return { error: true, text: `Skill "${sk.name}" needs a value for: ${missing.join(", ")}. Ask the user, then call run_skill again.` };
  tab = await resolveSkillTab(sk, tab);
  agentTabId = tab.id;
  const res = await executeSkill(sk, values, tab);
  let snap = "";
  try { snap = await takeSnapshot(); } catch (e) {}
  return { error: !res.ok, text: res.text + (snap ? "\n\n" + snap : "") };
}

// Shows the skill's inputs (pre-filled from the request / recording) and waits for the user to confirm or change them.
function askSkillInputs(sk, values, missing) {
  return new Promise((resolve) => {
    const div = addMessage("approval", "");
    div.append(mkEl("div", "title", i18n("sk.askTitle", { name: sk.name })));
    div.append(mkEl("div", "skhint", i18n(missing.length ? "sk.askMissing" : "sk.askCheck")));
    const form = mkEl("div", "skform");
    const inputs = {};
    for (const p of sk.params) {
      const lab = mkEl("label", "", (p.label || p.name) + (missing.includes(p.name) ? " *" : ""));
      const inp = mkEl("input");
      inp.value = values[p.name] || "";
      if (p.choices && p.choices.length) {
        const dl = mkEl("datalist");
        dl.id = "dl_" + sk.id + "_" + p.name;
        p.choices.forEach((c) => { const o = mkEl("option"); o.value = c; dl.append(o); });
        inp.setAttribute("list", dl.id);
        form.append(dl);
      }
      if (missing.includes(p.name)) inp.classList.add("need");
      inputs[p.name] = inp;
      form.append(lab, inp);
    }
    const btns = mkEl("div", "btns");
    const yes = mkEl("button", "yes", i18n("sk.runNow"));
    const no = mkEl("button", "no", i18n("btn.deny"));
    const done = (res) => { yes.disabled = no.disabled = true; for (const k in inputs) inputs[k].disabled = true; approvalResolver = null; resolve(res); };
    yes.onclick = () => {
      const out = {};
      for (const p of sk.params) out[p.name] = inputs[p.name].value.trim();
      const empty = sk.params.filter((p) => !out[p.name]);
      if (empty.length) { empty.forEach((p) => inputs[p.name].classList.add("need")); inputs[empty[0].name].focus(); return; }
      done(out);
    };
    no.onclick = () => done(null);
    approvalResolver = (ok) => done(ok ? null : null);
    btns.append(yes, no);
    div.append(form, btns);
    const first = sk.params.find((p) => missing.includes(p.name)) || sk.params[0];
    try { inputs[first.name].focus(); } catch (e) {}
  });
}

async function stepApproval(sk, st, values, tab, i) {
  if ($("autoApprove").checked) { logLine(i18n("log.auto", { action: stepText(st, st.param ? values[st.param] : null) })); return true; }
  const filled = sk.params.length ? sk.params.map((p) => `  ${p.label || p.name}: ${values[p.name]}`).join("\n") : "  " + i18n("d.none");
  return await requestApproval(stepText(st, st.param ? values[st.param] : null), i18n("why.final"), `${i18n("d.page")}: ${tab.title}\n${tab.url}\n\n${i18n("act.skill", { name: sk.name })}\n${i18n("d.fields")}:\n${filled}`);
}

async function executeSkill(sk, values, tab) {
  const t0 = Date.now();
  T("skill-run", { name: sk.name, steps: sk.steps.length, api: !!(sk.api && sk.useApi), values });
  let result;
  try { result = await executeSkillInner(sk, values, tab); } catch (e) { result = { ok: false, text: `Skill "${sk.name}" failed: ${e.message}` }; }
  await skillsLoad();
  const cur = SKILLS.find((s) => s.id === sk.id);
  if (cur) { cur.uses = (cur.uses || 0) + 1; if (result.ok) cur.ok = (cur.ok || 0) + 1; else if (!result.denied) cur.fail = (cur.fail || 0) + 1; await skillsSave(); }
  T("skill-done", { ok: result.ok, ms: Date.now() - t0, text: String(result.text).slice(0, 200) });
  return result;
}

async function executeSkillInner(sk, values, tab) {
  // 1) direct request, when the recording found one that fits every parameter
  if (sk.api && sk.useApi !== false) {
    const req = buildRequest(sk.api, values);
    let ok = true;
    if (!$("autoApprove").checked) {
      const body = req.body ? "\n\n" + String(req.body).slice(0, 600) : "";
      ok = await requestApproval(i18n("act.skill", { name: sk.name }), i18n("why.skillApi"), `${i18n("d.page")}: ${tab.title}\n${tab.url}\n\n${req.method} ${req.url}${body}`);
    } else logLine(i18n("log.auto", { action: i18n("act.skill", { name: sk.name }) }));
    if (!ok) return { ok: false, denied: true, text: "The user denied running this skill. Do not retry it. Ask how to proceed." };
    if (stopFlag) return { ok: false, text: "Stopped by user." };
    logLine("▶ " + i18n("sk.direct"));
    const [r] = await chrome.scripting.executeScript({ target: { tabId: agentTabId }, func: fetchFn, args: [req] });
    const res = r.result || { ok: false, status: 0, text: "no response" };
    T("skill-api", { status: res.status });
    if (res.ok) return { ok: true, mode: "api", text: `Direct request succeeded (HTTP ${res.status}). Response: ${res.text.slice(0, 400)}` };
    logLine("✖ " + i18n("sk.directFailed", { status: res.status }));
    if (res.status >= 400 && res.status < 500 && res.status !== 401 && res.status !== 403 && res.status !== 404) {
      return { ok: false, text: `Skill "${sk.name}": the app rejected the direct request (HTTP ${res.status}): ${res.text.slice(0, 300)}. Check the values; do not retry blindly.` };
    }
  } else if ($("stepMode").checked) {
    const ok = await requestApproval(i18n("act.skill", { name: sk.name }), i18n("why.step"), sk.steps.map((s, i) => `${i + 1}. ${stepText(s, s.param ? values[s.param] : null)}`).join("\n"));
    if (!ok) return { ok: false, denied: true, text: "The user denied running this skill. Do not retry it. Ask how to proceed." };
  }

  // 2) replay the recorded UI steps, checking each one
  const done = [];
  let noEffect = null;
  // Already past the first steps? (e.g. the recording began on page 1 and clicked through to page 2, and we are on page 2 now.)
  let i0 = 0;
  try {
    const here = stripHash((await chrome.tabs.get(agentTabId)).url);
    while (i0 < sk.steps.length - 1 && sk.steps[i0].t === "click" && stripHash(sk.steps[i0].url) !== here && stripHash(sk.steps[i0 + 1].url) === here) i0++;
    if (i0) logLine(i18n("sk.skipped", { n: i0 }));
  } catch (e) {}
  for (let i = i0; i < sk.steps.length; i++) {
    if (stopFlag) return { ok: false, text: "Stopped by user." };
    const st = sk.steps[i];
    const val = st.param ? values[st.param] : st.value;
    const rs = resolveStep(st, values);
    const txt = stepText(rs, val);
    if (st.t === "type" && st.sensitive) return { ok: false, text: `Skill "${sk.name}" stopped at step ${i + 1}: "${lbl(st.d)}" is a sensitive field. Ask the user to fill it in, then continue manually. Done so far: ${done.join("; ") || "(nothing)"}.` };
    if (stepIsFinal(st) && !(await stepApproval(sk, st, values, tab, i))) return { ok: false, denied: true, text: "The user denied this action. Do not retry it. Ask how to proceed." };
    logLine(`▶ ${i + 1}/${sk.steps.length} ${txt}`);
    if (st.opt) {
      // best-effort step (opening / driving a date picker): a failure here is fine, the date field is set from the parameter afterwards
      let r0 = { ok: false };
      try { [{ result: r0 }] = await chrome.scripting.executeScript({ target: { tabId: agentTabId }, func: replayStepFn, args: [rs, null, 1800] }); } catch (e) {}
      if (!r0 || !r0.ok) logLine("↷ " + i18n("sk.optSkipped", { n: i + 1 }));
      await settle(agentTabId, true);
      done.push(txt);
      continue;
    }
    let sigBefore = "";
    if (st.t === "click" && st.effect) { try { [{ result: sigBefore }] = await chrome.scripting.executeScript({ target: { tabId: agentTabId }, func: pageSigFn }); } catch (e) {} }
    const tryStep = async (timeout) => {
      try {
        const [r] = await chrome.scripting.executeScript({ target: { tabId: agentTabId }, func: replayStepFn, args: [resolveStep(st, values), val == null ? null : String(val), timeout] });
        return r.result || { ok: false, msg: "no result" };
      } catch (e) { return { ok: false, msg: e.message }; }
    };
    let res = await tryStep(i === i0 ? 2500 : 6000);
    if (!res.ok && i === i0) {
      // not on the starting page: go there (same site) and try again
      const cur = (await chrome.tabs.get(agentTabId)).url;
      if (sk.startUrl && stripHash(cur) !== stripHash(sk.startUrl) && hostOf(sk.startUrl) === sk.host) {
        await chrome.tabs.update(agentTabId, { url: sk.startUrl });
        await settle(agentTabId);
        res = await tryStep(6000);
      }
    }
    if (!res.ok) {
      T("skill-step-failed", { i, text: txt, msg: res.msg });
      const suspect = noEffect ? ` Note: step ${noEffect.i + 1} (${noEffect.txt}) showed no visible change and may be the cause.` : "";
      return { ok: false, text: `Skill "${sk.name}" stopped at step ${i + 1}/${sk.steps.length} (${txt}): ${res.msg}${suspect} Done so far: ${done.join("; ") || "(nothing)"}. Continue this task manually from the current page.` };
    }
    if (st.t === "click" && st.effect) await sleep(350); // let a form postback / navigation actually begin before checking whether the tab is ready
    await settle(agentTabId, st.t !== "click" && st.t !== "key");
    if (newTabId) { agentTabId = newTabId; newTabId = null; await settle(agentTabId); }
    if (st.t === "click" && st.effect && sigBefore) {
      // The recording showed this click changing the page. Give a slow result (server postback, partial update) time to arrive.
      // If nothing visibly changes we do NOT click again (that could repeat an action) and we do not fail yet: the next step
      // proves whether the click mattered; if that step cannot be found, the message names this click as the suspect.
      let changed = false;
      for (let k = 0; k < 10 && !changed; k++) {
        let sig = "";
        try { [{ result: sig }] = await chrome.scripting.executeScript({ target: { tabId: agentTabId }, func: pageSigFn }); } catch (e) { changed = true; break; }
        changed = sig !== sigBefore;
        if (!changed) await sleep(300);
      }
      if (!changed) {
        noEffect = { i, txt };
        T("skill-no-effect", { i, text: txt });
        logLine("⚠ " + i18n("sk.noEffect", { n: i + 1 }));
      } else if (noEffect && noEffect.i < i) noEffect = null;
    }
    done.push(txt);
    if (st.t === "type" && sk.params.length) typedLog.push({ label: lbl(st.d), value: String(val) });
  }
  return { ok: true, mode: "ui", text: `Skill "${sk.name}" completed all ${sk.steps.length} steps.${noEffect ? ` Note: step ${noEffect.i + 1} (${noEffect.txt}) showed no visible change; check the result.` : ""}` };
}

// ---------- wiring ----------
function skillsInit() {
  skillsLoad().then(renderSkills);
  if ($("recBtn")) {
    $("recBtn").onclick = (e) => { e.preventDefault(); if (recording) stopRecording(); else startRecording(); };
    $("recStop").onclick = () => stopRecording();
    $("skillsBtn").onclick = (e) => { e.preventDefault(); const b = $("skillsBox"); b.hidden = false; b.open = !b.open; };
  }
  try {
    chrome.storage.onChanged.addListener((ch, area) => { if (area === "local" && ch.skills) { SKILLS = Array.isArray(ch.skills.newValue) ? ch.skills.newValue : []; renderSkills(); } });
  } catch (e) {}
  i18nOnChange(renderSkills);
}
skillsInit();

















//manifest.json
{
  "manifest_version": 3,
  "name": "Page Agent",
  "version": "1.0.7",
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
