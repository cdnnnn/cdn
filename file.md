<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>One platform for every video in the organization</title>
<style>
  :root{
    --paper:#F4F5F8; --surface:#FFFFFF;
    --ink:#141829; --ink-2:#4C5570; --ink-3:#848CA4; --line:#E1E4EE;
    --brand:#2743C4; --brand-900:#101A4A; --tint:#EAEDFB;
    --accent:#FFB454; --accent-ink:#3D2606;
    --red:#D85A30; --red-tint:#FAECE7;
    --green:#0F6E56; --green-tint:#E1F5EE;
    --purple:#7F77DD; --purple-tint:#EEEDFE;
    --font: 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  }
  *{ box-sizing:border-box; }
  html,body{ height:100%; margin:0; }
  body{ background:var(--brand-900); color:var(--ink); font-family:var(--font); overflow:hidden; }

  .deck{ position:relative; width:100vw; height:100vh; }

  .slide{
    position:absolute; inset:0;
    display:flex; flex-direction:column; align-items:center; justify-content:center;
    background:var(--paper);
    opacity:0; visibility:hidden;
    transform:translateX(60px);
    transition:opacity .5s ease, transform .5s ease, visibility 0s linear .5s;
    padding:6vh 8vw;
  }
  .slide.is-active{ opacity:1; visibility:visible; transform:translateX(0); transition:opacity .5s ease, transform .5s ease, visibility 0s; }
  .slide.is-prev{ transform:translateX(-60px); }

  .slide.title{ background:var(--brand-900); color:#fff; }
  .slide.title .kicker{ color:var(--accent); }
  .slide.title h1{ color:#fff; }
  .slide.title .sub{ color:#AEB6DA; }
  .mark{ display:flex; align-items:center; gap:12px; margin-bottom:30px; }
  .mark .glyph{ width:34px; height:34px; border-radius:10px; background:var(--accent); position:relative; }
  .mark .glyph::before{ content:""; position:absolute; inset:0; margin:auto; width:0; height:0; border-left:10px solid var(--brand-900); border-top:6px solid transparent; border-bottom:6px solid transparent; transform:translateX(1px); }
  .mark span{ font-weight:700; font-size:20px; letter-spacing:-.03em; }

  .kicker{ font-family:'SFMono-Regular', Consolas, monospace; font-size:13px; letter-spacing:.14em; text-transform:uppercase; color:var(--brand); margin:0 0 14px; }
  h1{ font-size:clamp(26px,3.8vw,44px); font-weight:700; letter-spacing:-.035em; color:var(--brand-900); margin:0 0 16px; text-align:center; max-width:19ch; line-height:1.12; }
  .sub{ font-size:16.5px; color:var(--ink-2); max-width:58ch; text-align:center; margin:0 0 36px; line-height:1.6; }

  .visual{ display:flex; align-items:center; justify-content:center; gap:26px; min-height:220px; flex-wrap:wrap; }

  .stepnum{ position:absolute; top:6vh; left:8vw; font-family:'SFMono-Regular', Consolas, monospace; font-size:14px; color:var(--ink-3); letter-spacing:.06em; }

  .bar{ position:fixed; bottom:0; left:0; right:0; height:4px; background:var(--line); z-index:20; }
  .bar span{ display:block; height:100%; background:var(--brand); width:0; transition:width .4s ease; }

  .navzone{ position:fixed; top:0; bottom:0; width:14%; z-index:15; cursor:pointer; }
  .navzone.left{ left:0; } .navzone.right{ right:0; }

  .arrow{
    position:fixed; top:50%; transform:translateY(-50%); z-index:16;
    width:44px; height:44px; border-radius:50%; border:1px solid var(--line);
    background:rgba(255,255,255,.9); display:flex; align-items:center; justify-content:center;
    cursor:pointer; color:var(--brand-900); opacity:0; transition:opacity .2s, border-color .2s, background .2s;
    pointer-events:none;
  }
  .deck:hover .arrow{ opacity:1; pointer-events:auto; }
  .arrow:hover{ border-color:var(--brand); background:#fff; }
  .arrow.left{ left:24px; } .arrow.right{ right:24px; }
  .arrow svg{ width:16px; height:16px; }
  .arrow[disabled]{ opacity:0 !important; pointer-events:none; }

  .counter{ position:fixed; top:24px; right:28px; z-index:16; font-family:'SFMono-Regular', Consolas, monospace; font-size:13px; color:var(--ink-3); background:rgba(255,255,255,.85); border:1px solid var(--line); border-radius:999px; padding:6px 14px; }
  .dots{ position:fixed; bottom:16px; left:50%; transform:translateX(-50%); z-index:16; display:flex; gap:7px; }
  .dot{ width:7px; height:7px; border-radius:50%; background:var(--line); border:0; padding:0; cursor:pointer; }
  .dot.is-on{ background:var(--brand); }
  .hint{ position:fixed; bottom:20px; right:28px; z-index:16; font-size:12px; color:var(--ink-3); }

  .icon-circle{ width:88px; height:88px; border-radius:50%; display:flex; align-items:center; justify-content:center; flex:none; }
  .icon-circle svg{ width:38px; height:38px; }
  .icon-circle.lg{ width:100px; height:100px; }
  .icon-circle.lg svg{ width:44px; height:44px; }
  .icon-circle.sm{ width:60px; height:60px; }
  .icon-circle.sm svg{ width:26px; height:26px; }
  .ic-blue{ background:var(--tint); color:var(--brand); }
  .ic-red{ background:var(--red-tint); color:var(--red); }
  .ic-green{ background:var(--green-tint); color:var(--green); }
  .ic-navy{ background:var(--brand-900); color:#fff; }
  .ic-purple{ background:var(--purple-tint); color:var(--purple); }
  .ic-amber{ background:#FAEEDA; color:#854F0B; }

  .badge{ font-family:'SFMono-Regular', Consolas, monospace; font-size:12px; font-weight:600; padding:3px 10px; border-radius:999px; }
  .badge.problem{ background:var(--red-tint); color:var(--red); }
  .badge.solution{ background:var(--green-tint); color:var(--green); }

  .card{ display:flex; flex-direction:column; align-items:center; gap:10px; width:160px; }
  .card .label{ font-size:13.5px; font-weight:600; color:var(--brand-900); text-align:center; }
  .card .sublabel{ font-size:12px; color:var(--ink-3); text-align:center; }

  .flow-arrow{ width:30px; height:14px; flex:none; color:var(--ink-3); }
  .flow-arrow svg{ width:100%; height:100%; }

  /* scattered silos */
  .silos{ display:flex; gap:20px; flex-wrap:wrap; justify-content:center; max-width:520px; }
  .silo{ display:flex; flex-direction:column; align-items:center; gap:8px; }
  .silo .box{
    width:64px; height:64px; border-radius:12px; background:var(--surface); border:1.5px dashed var(--line);
    display:flex; align-items:center; justify-content:center; position:relative;
  }
  .silo .box svg{ width:26px; height:26px; color:var(--ink-3); }
  .silo .dupe{
    position:absolute; top:-7px; right:-7px; background:var(--red); color:#fff;
    font-family:'SFMono-Regular', Consolas, monospace; font-size:10px; font-weight:700;
    border-radius:999px; padding:2px 6px;
  }
  .silo .name{ font-size:12px; color:var(--ink-3); }

  /* funnel into hub */
  .funnel-scene{ display:flex; align-items:center; gap:30px; }
  .hub{
    width:110px; height:110px; border-radius:50%; background:var(--brand-900); color:#fff;
    display:flex; align-items:center; justify-content:center; flex:none;
    box-shadow:0 16px 32px -16px rgba(16,26,74,.5);
  }
  .hub svg{ width:44px; height:44px; }

  .stat-row{ display:flex; gap:36px; }
  .stat{ display:flex; flex-direction:column; align-items:center; gap:6px; }
  .stat .num{ font-size:34px; font-weight:700; color:var(--brand-900); letter-spacing:-.02em; }
  .stat .num.red{ color:var(--red); }
  .stat .lbl{ font-size:12.5px; color:var(--ink-3); text-align:center; max-width:14ch; }

  .feature-row{ display:flex; gap:24px; flex-wrap:wrap; justify-content:center; }
  .feature{
    width:190px; background:var(--surface); border:1px solid var(--line); border-radius:16px;
    padding:20px 18px; display:flex; flex-direction:column; align-items:center; gap:12px;
    text-align:center; box-shadow:0 10px 24px -18px rgba(16,26,74,.3);
  }
  .feature .ftitle{ font-size:14px; font-weight:700; color:var(--brand-900); }
  .feature .fbody{ font-size:12.5px; color:var(--ink-2); line-height:1.5; }

  .speechcard{
    background:var(--surface); border:1px solid var(--line); border-radius:16px;
    padding:14px 18px; max-width:250px; font-size:14px; color:var(--ink-2); line-height:1.5;
    box-shadow:0 10px 24px -16px rgba(16,26,74,.3);
  }
  .speechcard.you{ border-color:var(--brand); }
  .speechcard .who{ font-size:11px; font-weight:700; letter-spacing:.04em; text-transform:uppercase; margin-bottom:5px; }
  .speechcard.you .who{ color:var(--brand); }
  .speechcard.ai .who{ color:var(--green); }

  .compare{ display:flex; align-items:center; gap:36px; }

  @media (max-width:760px){
    .visual, .compare, .funnel-scene, .stat-row{ flex-direction:column; }
    .flow-arrow{ width:14px; height:30px; transform:rotate(90deg); }
    .arrow{ display:none; }
  }
  @media (prefers-reduced-motion: reduce){ *{ animation:none !important; } }
</style>
</head>
<body>

<div class="deck" id="deck">

  <!-- 0: title -->
  <div class="slide title is-active" data-slide="0">
    <div class="mark"><span class="glyph"></span><span>One video hub</span></div>
    <p class="kicker">Business case walkthrough</p>
    <h1>One platform for every video in the organization</h1>
    <p class="sub">Why finding the right video today takes too long — and how a central, searchable, AI-assisted hub fixes that.</p>
  </div>

  <!-- 1: problem - scattered, duplicated -->
  <div class="slide" data-slide="1">
    <span class="badge problem">The problem</span>
    <h1 style="margin-top:16px;">Videos are scattered everywhere, with copies of the same one</h1>
    <p class="sub">Different teams upload to different places — shared drives, team folders, personal libraries. The same recording often gets saved in three places, slightly renamed each time.</p>
    <div class="visual">
      <div class="silos">
        <div class="silo">
          <div class="box"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M3 7h6l2 2h10v10H3z"/></svg><span class="dupe">×2</span></div>
          <span class="name">Sales drive</span>
        </div>
        <div class="silo">
          <div class="box"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M3 7h6l2 2h10v10H3z"/></svg></div>
          <span class="name">HR folder</span>
        </div>
        <div class="silo">
          <div class="box"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M3 7h6l2 2h10v10H3z"/></svg><span class="dupe">×3</span></div>
          <span class="name">Engineering wiki</span>
        </div>
        <div class="silo">
          <div class="box"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M3 7h6l2 2h10v10H3z"/></svg></div>
          <span class="name">Someone's laptop</span>
        </div>
      </div>
    </div>
  </div>

  <!-- 2: problem - time cost -->
  <div class="slide" data-slide="2">
    <span class="badge problem">The cost</span>
    <h1 style="margin-top:16px;">People spend real work time just hunting for a video</h1>
    <p class="sub">Ask three people where "the onboarding walkthrough" is and you'll get three different links — or none. That search happens over and over, across the whole company.</p>
    <div class="visual">
      <div class="stat-row">
        <div class="stat">
          <div class="icon-circle sm ic-red"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="11" cy="11" r="7"/><path d="m20 20-3.6-3.6"/></svg></div>
          <div class="num red">Manual</div>
          <div class="lbl">searching across drives, chats, and folders</div>
        </div>
        <div class="stat">
          <div class="icon-circle sm ic-red"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="12" r="9"/><path d="M12 7v5l3 3"/></svg></div>
          <div class="num red">Repeated</div>
          <div class="lbl">the same search, done by every employee</div>
        </div>
        <div class="stat">
          <div class="icon-circle sm ic-red"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M12 2v4M12 18v4M4.9 4.9l2.8 2.8M16.3 16.3l2.8 2.8M2 12h4M18 12h4M4.9 19.1l2.8-2.8M16.3 7.7l2.8-2.8"/></svg></div>
          <div class="num red">Unclear</div>
          <div class="lbl">which copy is the right, current one</div>
        </div>
      </div>
    </div>
  </div>

  <!-- 3: solution intro -->
  <div class="slide" data-slide="3">
    <span class="badge solution">The solution</span>
    <h1 style="margin-top:16px;">Bring every video into one, organization-wide platform</h1>
    <p class="sub">Every department's videos live in a single place — deduplicated and organized — so there is exactly one place to look, not five.</p>
    <div class="visual">
      <div class="funnel-scene">
        <div class="silos" style="gap:12px;">
          <div class="silo"><div class="box" style="width:44px;height:44px;"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" style="width:18px;height:18px;"><path d="M3 7h6l2 2h10v10H3z"/></svg></div></div>
          <div class="silo"><div class="box" style="width:44px;height:44px;"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" style="width:18px;height:18px;"><path d="M3 7h6l2 2h10v10H3z"/></svg></div></div>
          <div class="silo"><div class="box" style="width:44px;height:44px;"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" style="width:18px;height:18px;"><path d="M3 7h6l2 2h10v10H3z"/></svg></div></div>
        </div>
        <div class="flow-arrow"><svg viewBox="0 0 34 16" fill="none"><path d="M0 8h28M22 2l8 6-8 6" stroke="currentColor" stroke-width="2"/></svg></div>
        <div class="hub"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="4" width="18" height="16" rx="2"/><path d="M3 9h18"/></svg></div>
      </div>
    </div>
  </div>

  <!-- 4: feature - search -->
  <div class="slide" data-slide="4">
    <span class="stepnum">Feature 1</span>
    <p class="kicker">Find it instantly</p>
    <h1>A search engine built for this library</h1>
    <p class="sub">Search across every video in the organization at once, and get back the exact match — not ten near-duplicates to sort through by hand.</p>
    <div class="visual">
      <div class="icon-circle lg ic-blue"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="11" cy="11" r="7"/><path d="m20 20-3.6-3.6"/></svg></div>
      <div class="flow-arrow"><svg viewBox="0 0 34 16" fill="none"><path d="M0 8h28M22 2l8 6-8 6" stroke="currentColor" stroke-width="2"/></svg></div>
      <div class="card">
        <div class="icon-circle ic-blue"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M20 6 9 17l-5-5"/></svg></div>
        <div class="label">The one right video</div>
        <div class="sublabel">no duplicates, no guesswork</div>
      </div>
    </div>
  </div>

  <!-- 5: feature - recommendations -->
  <div class="slide" data-slide="5">
    <span class="stepnum">Feature 2</span>
    <p class="kicker">Suggested for you</p>
    <h1>Recommendations based on your department</h1>
    <p class="sub">Someone in Sales sees what's relevant to Sales. Someone in Engineering sees what's relevant to Engineering — instead of one generic list for everyone.</p>
    <div class="visual">
      <div class="feature-row">
        <div class="feature">
          <div class="icon-circle sm ic-purple"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="8" r="4"/><path d="M4 20c0-4.4 3.6-7 8-7s8 2.6 8 7"/></svg></div>
          <div class="ftitle">Sales</div>
          <div class="fbody">Pitch decks, product demos, deal reviews</div>
        </div>
        <div class="feature">
          <div class="icon-circle sm ic-purple"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="8" r="4"/><path d="M4 20c0-4.4 3.6-7 8-7s8 2.6 8 7"/></svg></div>
          <div class="ftitle">Engineering</div>
          <div class="fbody">Architecture reviews, tech talks, postmortems</div>
        </div>
        <div class="feature">
          <div class="icon-circle sm ic-purple"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="8" r="4"/><path d="M4 20c0-4.4 3.6-7 8-7s8 2.6 8 7"/></svg></div>
          <div class="ftitle">HR</div>
          <div class="fbody">Onboarding, policy updates, training</div>
        </div>
      </div>
    </div>
  </div>

  <!-- 6: feature - LLM interaction -->
  <div class="slide" data-slide="6">
    <span class="stepnum">Feature 3</span>
    <p class="kicker">Ask, don't rewatch</p>
    <h1>Talk to the video with AI, instead of scrubbing through it</h1>
    <p class="sub">Instead of rewatching a 40-minute recording to find one detail, just ask. The AI has already gone through the whole video and answers directly.</p>
    <div class="visual">
      <div class="icon-circle lg ic-navy"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="4" width="12" height="16" rx="2"/><path d="M19 8v3M19 15v3" stroke-linecap="round"/><path d="M17 10.5h4M17 16.5h4" stroke-linecap="round"/></svg></div>
      <div class="speechcard you">
        <div class="who">You ask</div>
        Which slide covered the Q3 budget numbers?
      </div>
      <div class="speechcard ai">
        <div class="who">AI answers</div>
        Around the 14-minute mark, right after the hiring plan.
      </div>
    </div>
  </div>

  <!-- 7: outcome -->
  <div class="slide" data-slide="7">
    <span class="badge solution">The outcome</span>
    <h1 style="margin-top:16px;">Less time searching, more time working</h1>
    <p class="sub">One place to look, a search that actually works, content matched to your role, and instant answers instead of a full rewatch. The time saved adds up across every employee, every week.</p>
    <div class="visual">
      <div class="compare">
        <div class="card">
          <div class="icon-circle ic-red"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="12" r="9"/><path d="M12 7v5l4 2"/></svg></div>
          <div class="label">Before</div>
          <div class="sublabel">manual search across scattered, duplicated files</div>
        </div>
        <span class="flow-arrow"><svg viewBox="0 0 34 16" fill="none"><path d="M0 8h28M22 2l8 6-8 6" stroke="currentColor" stroke-width="2"/></svg></span>
        <div class="card">
          <div class="icon-circle ic-green"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M20 6 9 17l-5-5"/></svg></div>
          <div class="label">After</div>
          <div class="sublabel">one search, the right video, an instant answer</div>
        </div>
      </div>
    </div>
  </div>

  <!-- 8: closing -->
  <div class="slide" data-slide="8">
    <div class="icon-circle ic-green" style="margin-bottom:20px;">
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.4"><path d="M5 12l5 5L19 7"/></svg>
    </div>
    <p class="kicker">In short</p>
    <h1>One home for every video. One search that works. One click to the answer.</h1>
    <p class="sub">A central, deduplicated video platform with department-aware recommendations and AI-powered Q&amp;A — built to save the whole organization time.</p>
  </div>

  <div class="bar"><span id="barFill"></span></div>
  <button class="arrow left" id="arrowLeft" aria-label="Previous slide">
    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M15 5l-7 7 7 7"/></svg>
  </button>
  <button class="arrow right" id="arrowRight" aria-label="Next slide">
    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M9 5l7 7-7 7"/></svg>
  </button>
  <div class="navzone left" id="zoneLeft"></div>
  <div class="navzone right" id="zoneRight"></div>
  <div class="counter" id="counter">1 / 9</div>
  <div class="dots" id="dots"></div>
  <div class="hint">Use ← → or click the edges</div>
</div>

<script>
  const slides = Array.from(document.querySelectorAll('.slide'));
  const total = slides.length;
  let current = 0;

  const barFill = document.getElementById('barFill');
  const counter = document.getElementById('counter');
  const dotsEl = document.getElementById('dots');
  const arrowLeft = document.getElementById('arrowLeft');
  const arrowRight = document.getElementById('arrowRight');

  slides.forEach((_, i) => {
    const dot = document.createElement('button');
    dot.className = 'dot';
    dot.setAttribute('aria-label', 'Go to slide ' + (i + 1));
    dot.addEventListener('click', () => goTo(i));
    dotsEl.appendChild(dot);
  });
  const dotEls = Array.from(dotsEl.children);

  function render() {
    slides.forEach((el, i) => {
      el.classList.remove('is-active', 'is-prev');
      if (i === current) el.classList.add('is-active');
      else if (i < current) el.classList.add('is-prev');
    });
    dotEls.forEach((d, i) => d.classList.toggle('is-on', i === current));
    counter.textContent = (current + 1) + ' / ' + total;
    barFill.style.width = ((current + 1) / total * 100) + '%';
    arrowLeft.disabled = current === 0;
    arrowRight.disabled = current === total - 1;
  }

  function goTo(index) {
    if (index < 0 || index >= total) return;
    current = index;
    render();
  }

  arrowLeft.addEventListener('click', () => goTo(current - 1));
  arrowRight.addEventListener('click', () => goTo(current + 1));
  document.getElementById('zoneLeft').addEventListener('click', () => goTo(current - 1));
  document.getElementById('zoneRight').addEventListener('click', () => goTo(current + 1));

  document.addEventListener('keydown', (e) => {
    if (e.key === 'ArrowRight' || e.key === ' ') goTo(current + 1);
    if (e.key === 'ArrowLeft') goTo(current - 1);
  });

  render();
</script>

</body>
</html>
