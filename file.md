/* AIDLC Workspace — demo */
:root{
  --bg:#F4F5FB; --surface:#FFFFFF; --border:#E1E4EF; --border-strong:#A7AEC4;
  --ink:#161A2B; --muted:#565E78;
  --primary:#4F46E5; --primary-hover:#4338CA; --primary-tint:#EEF0FF; --primary-ink:#3730A3;
  --grad:linear-gradient(120deg,#4F46E5 0%,#7C3AED 55%,#C026D3 100%);
  --wait:#B45309; --wait-text:#8A4104; --wait-bg:#FFF1DE;
  --ok:#15803D; --ok-text:#166534; --ok-bg:#DCFCE7;
  --fail-text:#9C1F17; --fail-bg:#FDECEA;
  --side:#FFFFFF; --side-text:#2A3050; --side-muted:#565E78;
  --shadow:0 1px 2px rgba(22,26,43,.05),0 6px 20px rgba(22,26,43,.05);
  --font:"Segoe UI",Roboto,Arial,sans-serif; --font-mono:ui-monospace,Menlo,Consolas,monospace;
}
/* step colours: --c is the strong shade, --c-bg the soft tint */
.c-blue{--c:#1D4ED8;--c-bg:#DBEAFE}
.c-violet{--c:#6D28D9;--c-bg:#EDE9FE}
.c-teal{--c:#0F766E;--c-bg:#CCFBF1}
.c-orange{--c:#C2410C;--c-bg:#FFEDD5}
.c-pink{--c:#BE185D;--c-bg:#FCE7F3}
.c-green{--c:#15803D;--c-bg:#DCFCE7}

*{box-sizing:border-box}
html,body{height:100%}
body{margin:0;background:var(--bg);color:var(--ink);font-family:var(--font);font-size:14px;line-height:1.5}
h1,h2,h3,p{margin:0}
button,input,textarea{font:inherit;color:inherit}
a{color:var(--primary)}a:hover{color:var(--primary-hover)}
:focus-visible{outline:2px solid var(--primary);outline-offset:2px}
[hidden]{display:none!important}
.sr-only{position:absolute;width:1px;height:1px;overflow:hidden;clip:rect(0 0 0 0);white-space:nowrap}
.muted{color:var(--muted)}.small{font-size:13px}.strong{font-weight:600}
.mono{font-family:var(--font-mono);font-size:12px}
.row{display:flex;flex-wrap:wrap;gap:12px;align-items:center}
.between{justify-content:space-between}
.stack{display:flex;flex-direction:column;gap:16px}

/* ---------- Frame: sidebar | main | chat (400px) ---------- */
.app{height:100vh;height:100dvh;display:grid;grid-template-columns:248px minmax(0,1fr) 400px;grid-template-rows:auto minmax(0,1fr)}
.topbar{grid-column:1 / -1;background:var(--surface);border-bottom:1px solid var(--border);padding:0 20px;min-height:60px;display:flex;align-items:center;gap:20px;position:relative;z-index:2}
.brand{display:flex;align-items:center;gap:10px;font-weight:700;font-size:18px;color:var(--ink);text-decoration:none;letter-spacing:.01em}
.brand:hover{color:var(--ink)}
.brand__mark{width:34px;height:34px;border-radius:10px;background:var(--grad);color:#fff;display:flex;align-items:center;justify-content:center;box-shadow:0 4px 10px rgba(79,70,229,.3)}
.brand__sub{font-weight:500;color:var(--muted)}
.crumbs{flex:1;min-width:0;display:flex;flex-wrap:wrap;align-items:center;gap:6px;font-size:13px;color:var(--muted)}
.crumbs a{color:var(--muted);text-decoration:none;font-weight:500;padding:8px 2px}
.crumbs a:hover{color:var(--primary)}
.crumbs .here{color:var(--ink);font-weight:600}

.btn{min-height:44px;padding:0 18px;border-radius:12px;border:1px solid var(--border-strong);background:var(--surface);color:var(--ink);font-weight:600;cursor:pointer;text-decoration:none;display:inline-flex;align-items:center;justify-content:center;gap:8px;transition:background .15s,box-shadow .15s,transform .15s}
.btn:hover{background:var(--primary-tint);border-color:var(--primary);color:var(--primary-ink)}
.btn--primary{background:var(--primary);border-color:var(--primary);color:#fff;box-shadow:0 4px 12px rgba(79,70,229,.28)}
.btn--primary:hover{background:var(--primary-hover);border-color:var(--primary-hover);color:#fff;transform:translateY(-1px)}
.btn--light{background:#fff;border-color:#fff;color:var(--primary-ink)}
.btn--light:hover{background:var(--primary-tint);border-color:#fff}
.btn--quiet{min-height:38px;padding:0 14px;border-color:var(--border);font-weight:500}
.btn--icon{width:46px;padding:0;flex:none}
.btn:disabled{background:#E7E9F2;border-color:#E7E9F2;color:var(--muted);cursor:not-allowed;box-shadow:none;transform:none}

.icon-tile{flex:none;width:40px;height:40px;border-radius:12px;background:var(--c-bg,var(--primary-tint));color:var(--c,var(--primary));display:flex;align-items:center;justify-content:center;font-weight:700;font-size:18px}
.avatar{flex:none;width:46px;height:46px;border-radius:14px;background:var(--c);color:#fff;display:flex;align-items:center;justify-content:center;font-weight:700;font-size:15px;letter-spacing:.02em}
.avatar--sm{width:26px;height:26px;border-radius:8px;font-size:11px}

/* ---------- Sidebar (light) ---------- */
.sidebar{background:linear-gradient(180deg,#F7F6FF,#FFFFFF 240px);border-right:1px solid var(--border);color:var(--side-text);padding:18px 12px;overflow-y:auto;display:flex;flex-direction:column;gap:4px}
.side-label{font-size:11px;font-weight:700;letter-spacing:.08em;color:var(--side-muted);padding:8px 8px 6px}
.side-link{display:flex;align-items:center;justify-content:space-between;gap:8px;min-height:40px;padding:6px 8px;border-radius:10px;color:var(--side-text);text-decoration:none;font-weight:500;transition:background .15s}
.side-link:hover{background:var(--primary-tint);color:var(--primary-ink)}
.side-link[aria-current="page"]{background:#E0E4FF;color:var(--primary-ink);font-weight:700}
.side-link__main{display:flex;align-items:center;gap:10px;min-width:0}
.side-link__name{min-width:0;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
.side-link--project{font-weight:600;color:var(--ink)}
.side-sub{margin:2px 0 8px 20px;padding-left:10px;border-left:2px solid #DCDFF5;display:flex;flex-direction:column;gap:2px}
.side-sub .side-link{font-size:13px;min-height:36px}
.side-sub .side-label{padding:10px 8px 2px}
.side-new{margin-top:10px;background:var(--primary-tint);border:1px dashed #A5ABEF;color:var(--primary-ink)}
.side-new:hover{background:#E0E4FF;border-color:var(--primary)}
.tag{flex:none;font-size:11px;font-weight:700;padding:2px 8px;border-radius:999px}
.tag--wait{background:var(--wait-bg);color:var(--wait-text)}
.tag--run{background:#E0E7FF;color:var(--primary-ink)}
.tag--done,.tag--ok{background:var(--ok-bg);color:var(--ok-text)}

/* ---------- Main ---------- */
.main{overflow-y:auto;padding:28px 28px 56px}
.main__inner{max-width:920px;margin:0 auto;display:flex;flex-direction:column;gap:22px}
.main__inner.enter{animation:enter .22s ease-out}
@keyframes enter{from{opacity:0;transform:translateY(6px)}to{opacity:1;transform:none}}
.page-title{font-weight:700;font-size:28px;line-height:1.2;overflow-wrap:anywhere;letter-spacing:-.01em}
.section-title{font-weight:700;font-size:17px}

.hero{background:var(--grad);color:#fff;border-radius:20px;padding:26px 28px;display:flex;flex-wrap:wrap;align-items:center;justify-content:space-between;gap:16px;box-shadow:0 10px 28px rgba(79,70,229,.25);position:relative;overflow:hidden}
.hero::after{content:"";position:absolute;right:-60px;top:-80px;width:260px;height:260px;border-radius:50%;background:rgba(255,255,255,.1);pointer-events:none}
.hero::before{content:"";position:absolute;right:120px;bottom:-110px;width:200px;height:200px;border-radius:50%;background:rgba(255,255,255,.08);pointer-events:none}
.hero>*{position:relative;z-index:1}
.hero__text{display:flex;flex-direction:column;gap:6px;min-width:0;flex:1 1 320px}
.hero__kicker{font-weight:600;opacity:.92}
.hero__side{display:flex;flex-direction:column;align-items:flex-end;gap:8px;font-weight:600}
.hero .pill{background:#fff}

.stat-row{display:grid;grid-template-columns:repeat(4,minmax(0,1fr));gap:14px}
.stat-card{background:var(--surface);border:1px solid var(--border);border-radius:16px;padding:14px;display:flex;align-items:center;gap:12px;box-shadow:var(--shadow)}
.stat-card__text{display:flex;flex-direction:column;min-width:0}
.stat-card .small{color:var(--muted)}
.stat{font-weight:700;font-size:24px;line-height:1.2}

.card{background:var(--surface);border:1px solid var(--border);border-radius:16px;padding:22px;display:flex;flex-direction:column;gap:14px;box-shadow:var(--shadow)}
.card--wait{border:2px solid #F0B35A;padding:21px;background:linear-gradient(180deg,#FFF8EA,#FFFFFF 70%)}
.card--wait .icon-tile{background:var(--wait-bg);color:var(--wait)}
.card--run{border-color:#C7CCFB}
.icon-tile--spin{animation:pulse 1.4s ease-in-out infinite}
@keyframes pulse{0%,100%{box-shadow:0 0 0 0 rgba(79,70,229,.35)}50%{box-shadow:0 0 0 8px rgba(79,70,229,0)}}

.pill{display:inline-flex;align-items:center;gap:6px;padding:4px 11px;border-radius:999px;font-size:12px;font-weight:700;white-space:nowrap}
.pill::before{content:"";width:7px;height:7px;border-radius:50%;background:currentColor}
.pill--done{background:var(--ok-bg);color:var(--ok-text)}
.pill--wait{background:var(--wait-bg);color:var(--wait-text)}
.pill--run{background:#E0E7FF;color:var(--primary-ink)}
.pill--idle{background:#ECEEF5;color:var(--muted)}
.pill--fail{background:var(--fail-bg);color:var(--fail-text)}
.skill{padding:3px 9px;border-radius:8px;background:var(--primary-tint);color:var(--primary-ink);font-family:var(--font-mono);font-size:12px}
.skills{display:flex;flex-wrap:wrap;gap:6px}
.note{padding:12px 14px;border-radius:12px;background:#F1F3FA;font-size:13px}
.note--done,.note--ok{background:var(--ok-bg);color:var(--ok-text);display:flex;align-items:center;gap:10px}
.empty{padding:36px;border:2px dashed #C9CEDF;border-radius:16px;text-align:center;color:var(--muted);display:flex;flex-direction:column;align-items:center;gap:14px;background:rgba(255,255,255,.6)}

/* project cards and feature rows */
.grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(270px,1fr));gap:16px}
.pcard{background:var(--surface);border:1px solid var(--border);border-radius:16px;padding:18px;display:flex;flex-direction:column;gap:12px;text-decoration:none;color:var(--ink);min-height:168px;box-shadow:var(--shadow);transition:transform .15s,box-shadow .15s,border-color .15s;text-align:left}
.pcard:hover{transform:translateY(-3px);border-color:var(--primary);color:var(--ink);box-shadow:0 12px 28px rgba(79,70,229,.16)}
.pcard__head{display:flex;flex-direction:column;min-width:0}
.pcard__name{font-weight:700;font-size:18px;overflow-wrap:anywhere;line-height:1.25}
.pcard__repo{display:flex;align-items:center;gap:6px;min-width:0}
.pcard__repo span{min-width:0;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
.pcard__foot{margin-top:auto;display:flex;flex-wrap:wrap;gap:6px;align-items:center}
.pcard--new{border:2px dashed #C9CEDF;background:transparent;box-shadow:none;align-items:center;justify-content:center;gap:8px;cursor:pointer;text-align:center}
.frow{background:var(--surface);border:1px solid var(--border);border-radius:14px;padding:14px 16px;display:flex;flex-wrap:wrap;align-items:center;justify-content:space-between;gap:10px 16px;text-decoration:none;color:var(--ink);box-shadow:var(--shadow);transition:transform .15s,border-color .15s}
.frow:hover{border-color:var(--primary);color:var(--ink);transform:translateX(3px)}
.frow__text{display:flex;flex-direction:column;min-width:0;flex:1 1 240px}
.frow__title{font-weight:600;font-size:15px;overflow-wrap:anywhere}
.segs{display:inline-flex;gap:3px}
.segs i{width:16px;height:7px;border-radius:4px;background:#DDE1EC}
.segs i.on{background:var(--c)}

/* tabs */
.tabs{display:flex;flex-wrap:wrap;gap:4px;border-bottom:1px solid var(--border)}
.tab{min-height:44px;padding:0 14px;border:0;border-bottom:3px solid transparent;background:none;font-weight:600;color:var(--muted);cursor:pointer;text-decoration:none;display:inline-flex;align-items:center}
.tab:hover{color:var(--primary)}
.tab[aria-selected="true"],.tab[aria-current="page"]{color:var(--primary-ink);border-bottom-color:var(--primary)}
.seg{display:inline-flex;flex-wrap:wrap;gap:4px;padding:4px;border-radius:12px;background:#ECEEF6;align-self:flex-start}
.seg__btn{min-height:38px;padding:0 14px;border:0;border-radius:9px;background:none;font-weight:600;color:var(--muted);cursor:pointer}
.seg__btn:hover{color:var(--ink)}
.seg__btn[aria-selected="true"]{background:#fff;color:var(--primary-ink);box-shadow:0 1px 3px rgba(22,26,43,.14)}

/* stepper */
.stepper{display:grid;grid-template-columns:repeat(6,minmax(0,1fr));gap:10px}
.stp{min-width:0;display:flex;flex-direction:column;gap:8px;padding:12px;border:1px solid var(--border);border-radius:16px;background:var(--surface);text-decoration:none;color:var(--ink);box-shadow:var(--shadow);transition:transform .15s,border-color .15s}
.stp:hover{transform:translateY(-2px);border-color:var(--c);color:var(--ink)}
.stp__icon{position:relative;width:40px;height:40px;border-radius:12px;background:var(--c-bg);color:var(--c);display:flex;align-items:center;justify-content:center}
.stp__badge{position:absolute;right:-6px;top:-6px;width:18px;height:18px;border-radius:50%;background:var(--ok);color:#fff;border:2px solid #fff;display:flex;align-items:center;justify-content:center}
.stp__badge svg{width:9px;height:9px}
.stp__name{display:flex;flex-direction:column;font-weight:700;font-size:13px;line-height:1.3}
.stp__n{font-size:11px;font-weight:600;color:var(--muted);letter-spacing:.03em}
.stp__status{font-size:12px;color:var(--muted);font-weight:600}
.stp.is-approved .stp__icon,.stp.is-done .stp__icon{background:var(--c);color:#fff}
.stp.is-approved .stp__status,.stp.is-done .stp__status{color:var(--ok-text)}
.stp.is-locked{background:#F7F8FC;box-shadow:none}
.stp.is-locked .stp__icon{background:#ECEEF5;color:#7A8399}
.stp.is-locked .stp__name{color:var(--muted)}
.stp.is-running .stp__icon{animation:pulse 1.4s ease-in-out infinite}
.stp.is-running .stp__status,.stp.is-ready .stp__status{color:var(--primary-ink)}
.stp.is-awaiting{background:#FFF9EE;border-color:#F0B35A}
.stp.is-awaiting .stp__status{color:var(--wait-text)}
.stp[aria-current="true"]{border:2px solid var(--c);padding:11px;box-shadow:0 8px 22px rgba(22,26,43,.1)}
.stp.is-awaiting[aria-current="true"]{border-color:var(--wait)}
@keyframes spin{to{transform:rotate(360deg)}}
@media (prefers-reduced-motion:reduce){*{animation:none!important;transition:none!important}}

/* tables, lists, bars */
.table-wrap{overflow-x:auto;border:1px solid var(--border);border-radius:12px}
table{width:100%;min-width:460px;border-collapse:collapse;font-size:13px}
th{text-align:left;color:var(--muted);font-weight:700;font-size:12px;letter-spacing:.03em;padding:10px 14px;background:#F6F7FC;border-bottom:1px solid var(--border)}
td{padding:11px 14px;border-bottom:1px solid var(--border);vertical-align:top;overflow-wrap:anywhere}
tbody tr:last-child td{border-bottom:0}
tbody tr:hover td{background:#FAFAFF}
.list{margin:0;padding:0;list-style:none;border:1px solid var(--border);border-radius:12px}
.list>li{padding:11px 14px;display:flex;flex-wrap:wrap;justify-content:space-between;gap:8px;border-bottom:1px solid var(--border)}
.list>li:last-child{border-bottom:0}
.kv{display:flex;justify-content:space-between;gap:8px;font-size:13px;margin-bottom:4px}
.bar{position:relative;height:10px;border-radius:5px;background:#E4E7F2;margin-bottom:10px}
.bar__fill{height:100%;border-radius:5px;background:var(--grad)}
.bar__target{position:absolute;top:-4px;width:3px;height:18px;border-radius:2px;background:var(--wait)}
.tiles{display:flex;flex-wrap:wrap;gap:12px}
.tile{flex:1 1 140px;min-width:0;padding:14px 16px;border-radius:14px;background:#F1F3FA;display:flex;flex-direction:column;gap:2px}
.tile--ok{background:var(--ok-bg);color:var(--ok-text)}
.tile--bad{background:var(--fail-bg);color:var(--fail-text)}
.runbar{height:8px;border-radius:4px;background:#E4E7F2;overflow:hidden}
.runbar>div{height:100%;border-radius:4px;background:var(--grad);transition:width .6s ease}
.tasks{margin:0;padding:0;list-style:none;display:flex;flex-direction:column;gap:10px}
.tasks li{display:flex;align-items:center;gap:10px}
.dot{width:20px;height:20px;border-radius:50%;border:2px solid #C3C8DA;flex:none;display:flex;align-items:center;justify-content:center;color:#fff}
.dot svg{width:11px;height:11px}
.dot--done{background:var(--ok);border-color:var(--ok)}
.dot--doing{border-color:var(--primary);border-top-color:transparent;animation:spin .8s linear infinite}

/* forms */
.field{display:flex;flex-direction:column;gap:6px}
.field label,.field legend{font-size:13px;font-weight:600;padding:0}
.field input[type=text],.field textarea{width:100%;min-height:44px;padding:10px 12px;border:1px solid var(--border-strong);border-radius:12px;background:var(--surface)}
.field input[type=text]:focus,.field textarea:focus,.composer__form textarea:focus{outline:none;border-color:var(--primary);box-shadow:0 0 0 3px rgba(79,70,229,.2)}
.field textarea{resize:vertical}
.error{color:var(--fail-text);font-size:13px;font-weight:600}
.choice{display:flex;align-items:flex-start;gap:10px;padding:12px;margin-top:8px;line-height:1.4;border:1px solid var(--border);border-radius:12px;cursor:pointer}
.choice:has(input:checked){border-color:var(--primary);background:var(--primary-tint)}
.choice input{flex:none;margin-top:2px;width:18px;height:18px;accent-color:var(--primary)}
fieldset{margin:0;padding:0;border:0}
.dialog{width:min(520px,calc(100vw - 32px));border:0;border-radius:20px;padding:26px;color:var(--ink);box-shadow:0 24px 60px rgba(22,26,43,.3)}
.dialog::backdrop{background:rgba(23,26,46,.55)}
.dialog form{display:flex;flex-direction:column;gap:16px}
.dialog h2{font-weight:700;font-size:20px}

/* ---------- Chat (right, 400px) ---------- */
.chat{background:var(--surface);border-left:1px solid var(--border);display:flex;flex-direction:column;min-height:0}
.chat__head{padding:14px 16px;border-bottom:1px solid var(--border);display:flex;align-items:center;gap:12px;background:linear-gradient(180deg,#F3F1FF,#FFFFFF)}
.chat__headtext{flex:1;min-width:0;display:flex;flex-direction:column}
.chat__title{font-weight:700;font-size:15px}
.chat__ctx{font-size:12px;color:var(--muted);overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
.bot{flex:none;width:38px;height:38px;border-radius:12px;background:var(--grad);color:#fff;display:flex;align-items:center;justify-content:center;box-shadow:0 4px 10px rgba(79,70,229,.3)}
.bot--sm{width:28px;height:28px;border-radius:9px;box-shadow:none}
.bot--sm svg{width:15px;height:15px}
.chat__scroll{flex:1;min-height:0;overflow-y:auto;background:#FBFBFE}
.chat__msgs{padding:16px;display:flex;flex-direction:column;gap:14px}
.msg{display:flex;gap:8px;align-items:flex-start}
.msg--user{justify-content:flex-end}
.msg__body{display:flex;flex-direction:column;gap:4px;align-items:flex-start;min-width:0}
.bubble{max-width:100%;padding:10px 14px;border-radius:16px;overflow-wrap:anywhere}
.msg--user .bubble{max-width:86%;background:var(--grad);color:#fff;border-bottom-right-radius:5px;white-space:pre-wrap;box-shadow:0 4px 12px rgba(79,70,229,.22)}
.msg--ai .bubble{background:#fff;border:1px solid var(--border);border-top-left-radius:5px}
.msg__link{font-weight:700;font-size:13px;padding:6px 2px;text-decoration:none}
.msg__link:hover{text-decoration:underline}
.composer{border-top:1px solid var(--border);padding:12px 16px 10px;display:flex;flex-direction:column;gap:10px;background:#fff}
.chips{display:flex;flex-wrap:wrap;gap:8px}
.chips:empty{display:none}
.chip{min-height:38px;padding:6px 13px;border-radius:999px;border:1px solid #D4D8F7;background:var(--primary-tint);color:var(--primary-ink);font-weight:600;font-size:13px;cursor:pointer;text-align:left;transition:background .15s}
.chip:hover{background:#E0E4FF}
.chip--primary{background:var(--primary);border-color:var(--primary);color:#fff}
.chip--primary:hover{background:var(--primary-hover)}
.composer__form{display:flex;gap:8px;align-items:flex-end}
.composer__form textarea{flex:1;min-height:46px;max-height:120px;padding:11px 14px;border:1px solid var(--border-strong);border-radius:14px;resize:none;background:var(--surface)}
.composer__form textarea:disabled{background:#F1F3FA}
.composer__form .btn{min-height:46px;border-radius:14px}
.composer__note{font-size:12px;color:var(--muted);text-align:center}

/* ---------- Smaller screens ---------- */
@media (max-width:1180px){
  .app{grid-template-columns:minmax(0,1fr) 400px}
  .sidebar{display:none}
  .stat-row{grid-template-columns:repeat(2,minmax(0,1fr))}
  .stepper{grid-template-columns:repeat(3,minmax(0,1fr))}
}
@media (max-width:820px){
  html,body{height:auto}
  .app{height:auto;display:flex;flex-direction:column}
  .main{overflow:visible;padding:20px 16px 28px}
  .chat{border-left:0;border-top:1px solid var(--border);height:75vh}
  .topbar{flex-wrap:wrap;padding:8px 16px}
  .hero{padding:20px}
  .hero__side{align-items:flex-start}
  .stepper{grid-template-columns:repeat(2,minmax(0,1fr))}
}
