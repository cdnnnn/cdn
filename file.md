<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Content Analytics — One Stop Video Hub</title>
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
  .mark{ display:flex; align-items:center; gap:12px; margin-bottom:28px; }
  .mark .glyph{ width:34px; height:34px; border-radius:10px; background:var(--accent); position:relative; }
  .mark .glyph::before{ content:""; position:absolute; inset:0; margin:auto; width:0; height:0; border-left:10px solid var(--brand-900); border-top:6px solid transparent; border-bottom:6px solid transparent; transform:translateX(1px); }
  .mark span{ font-weight:700; font-size:20px; letter-spacing:-.03em; }

  .kicker{ font-family:'SFMono-Regular', Consolas, monospace; font-size:12.5px; letter-spacing:.14em; text-transform:uppercase; color:var(--brand); margin:0 0 12px; }
  h1{ font-size:clamp(24px,3.4vw,40px); font-weight:700; letter-spacing:-.032em; color:var(--brand-900); margin:0 0 14px; text-align:center; max-width:20ch; line-height:1.14; }
  .sub{ font-size:15.5px; color:var(--ink-2); max-width:56ch; text-align:center; margin:0 0 30px; line-height:1.6; }

  .visual{ display:flex; align-items:center; justify-content:center; gap:22px; min-height:190px; flex-wrap:wrap; }

  .bar{ position:fixed; bottom:0; left:0; right:0; height:4px; background:var(--line); z-index:20; }
  .bar span{ display:block; height:100%; background:var(--brand); width:0; transition:width .4s ease; }

  .navzone{ position:fixed; top:0; bottom:0; width:14%; z-index:15; cursor:pointer; }
  .navzone.left{ left:0; } .navzone.right{ right:0; }

  .arrow{
    position:fixed; top:50%; transform:translateY(-50%); z-index:16;
    width:42px; height:42px; border-radius:50%; border:1px solid var(--line);
    background:rgba(255,255,255,.9); display:flex; align-items:center; justify-content:center;
    cursor:pointer; color:var(--brand-900); opacity:0; transition:opacity .2s, border-color .2s, background .2s;
    pointer-events:none;
  }
  .deck:hover .arrow{ opacity:1; pointer-events:auto; }
  .arrow:hover{ border-color:var(--brand); background:#fff; }
  .arrow.left{ left:20px; } .arrow.right{ right:20px; }
  .arrow svg{ width:15px; height:15px; }
  .arrow[disabled]{ opacity:0 !important; pointer-events:none; }

  .counter{ position:fixed; top:22px; right:26px; z-index:16; font-family:'SFMono-Regular', Consolas, monospace; font-size:12.5px; color:var(--ink-3); background:rgba(255,255,255,.85); border:1px solid var(--line); border-radius:999px; padding:6px 13px; }
  .dots{ position:fixed; bottom:15px; left:50%; transform:translateX(-50%); z-index:16; display:flex; gap:6px; flex-wrap:wrap; max-width:60vw; justify-content:center; }
  .dot{ width:6px; height:6px; border-radius:50%; background:var(--line); border:0; padding:0; cursor:pointer; }
  .dot.is-on{ background:var(--brand); }
  .hint{ position:fixed; bottom:18px; right:26px; z-index:16; font-size:11.5px; color:var(--ink-3); }

  .icon-circle{ width:80px; height:80px; border-radius:50%; display:flex; align-items:center; justify-content:center; flex:none; }
  .icon-circle svg{ width:34px; height:34px; }
  .icon-circle.lg{ width:96px; height:96px; }
  .icon-circle.lg svg{ width:42px; height:42px; }
  .icon-circle.sm{ width:56px; height:56px; }
  .icon-circle.sm svg{ width:24px; height:24px; }
  .ic-blue{ background:var(--tint); color:var(--brand); }
  .ic-red{ background:var(--red-tint); color:var(--red); }
  .ic-green{ background:var(--green-tint); color:var(--green); }
  .ic-navy{ background:var(--brand-900); color:#fff; }
  .ic-purple{ background:var(--purple-tint); color:var(--purple); }
  .ic-amber{ background:#FAEEDA; color:#854F0B; }

  .badge{ font-family:'SFMono-Regular', Consolas, monospace; font-size:11.5px; font-weight:600; padding:3px 10px; border-radius:999px; }
  .badge.problem{ background:var(--red-tint); color:var(--red); }
  .badge.solution{ background:var(--green-tint); color:var(--green); }

  .card{ display:flex; flex-direction:column; align-items:center; gap:9px; width:150px; }
  .card .label{ font-size:13px; font-weight:600; color:var(--brand-900); text-align:center; }
  .card .sublabel{ font-size:11.5px; color:var(--ink-3); text-align:center; }

  .flow-arrow{ width:26px; height:13px; flex:none; color:var(--ink-3); }
  .flow-arrow svg{ width:100%; height:100%; }

  .silos{ display:flex; gap:18px; flex-wrap:wrap; justify-content:center; max-width:520px; }
  .silo{ display:flex; flex-direction:column; align-items:center; gap:7px; }
  .silo .box{
    width:60px; height:60px; border-radius:12px; background:var(--surface); border:1.5px dashed var(--line);
    display:flex; align-items:center; justify-content:center; position:relative;
  }
  .silo .box svg{ width:24px; height:24px; color:var(--ink-3); }
  .silo .dupe{
    position:absolute; top:-7px; right:-7px; background:var(--red); color:#fff;
    font-family:'SFMono-Regular', Consolas, monospace; font-size:9.5px; font-weight:700;
    border-radius:999px; padding:2px 6px;
  }
  .silo .name{ font-size:11.5px; color:var(--ink-3); }

  .stat-row{ display:flex; gap:30px; flex-wrap:wrap; justify-content:center; }
  .stat{ display:flex; flex-direction:column; align-items:center; gap:6px; width:150px; }
  .stat .lbl{ font-size:12px; color:var(--ink-3); text-align:center; }
  .stat .lbltitle{ font-size:13.5px; font-weight:700; }
  .stat .lbltitle.red{ color:var(--red); }

  .feature-row{ display:flex; gap:20px; flex-wrap:wrap; justify-content:center; }
  .feature{
    width:172px; background:var(--surface); border:1px solid var(--line); border-radius:14px;
    padding:18px 16px; display:flex; flex-direction:column; align-items:center; gap:10px;
    text-align:center; box-shadow:0 10px 22px -18px rgba(16,26,74,.3);
  }
  .feature .ftitle{ font-size:13px; font-weight:700; color:var(--brand-900); }
  .feature .fbody{ font-size:11.5px; color:var(--ink-2); line-height:1.5; }

  .compare{ display:flex; align-items:center; gap:30px; flex-wrap:wrap; justify-content:center; }

  .videocard{
    width:160px; background:var(--surface); border:1px solid var(--line); border-radius:14px;
    padding:16px; display:flex; flex-direction:column; align-items:center; gap:8px; position:relative;
    box-shadow:0 10px 22px -18px rgba(16,26,74,.3);
  }
  .videocard .banner{ font-size:10.5px; font-weight:700; padding:3px 8px; border-radius:6px; text-align:center; }
  .videocard .banner.info{ background:var(--tint); color:var(--brand); }

  .recap-row{ display:flex; gap:26px; flex-wrap:wrap; justify-content:center; }
  .recap{ display:flex; flex-direction:column; align-items:center; gap:8px; width:110px; }
  .recap .rl{ font-size:12px; font-weight:600; color:var(--brand-900); text-align:center; }

  @media (max-width:760px){
    .visual, .compare, .stat-row{ flex-direction:column; }
    .flow-arrow{ width:13px; height:26px; transform:rotate(90deg); }
    .arrow{ display:none; }
  }
  @media (prefers-reduced-motion: reduce){ *{ animation:none !important; } }
</style>
</head>
<body>

<div class="deck" id="deck">

  <div class="slide title is-active" data-slide="0">
    <div class="mark"><span class="glyph"></span><span>Content Analytics</span></div>
    <p class="kicker">A simple walkthrough</p>
    <h1>Content Analytics — One Stop Video Hub</h1>
    <p class="sub">A few problems with how we handle training videos today — and a simple solution for each.</p>
  </div>

  <div class="slide" data-slide="1">
    <span class="badge problem">Problem 1</span>
    <h1 style="margin-top:14px;">Different departments keep making the same training video</h1>
    <p class="sub">Finance, HR, Dev, QC, and Ops all need similar training, like onboarding or basic company policies. But there's no shared place to check what already exists — so each department ends up recording its own version of a video that's already been made elsewhere in the company.</p>
    <div class="visual">
      <div class="silos">
        <div class="silo"><div class="box"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="6" width="13" height="12" rx="2"/><path d="M16 10l5-3v10l-5-3z"/></svg><span class="dupe">AGAIN</span></div><span class="name">Finance team</span></div>
        <div class="silo"><div class="box"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="6" width="13" height="12" rx="2"/><path d="M16 10l5-3v10l-5-3z"/></svg><span class="dupe">AGAIN</span></div><span class="name">HR team</span></div>
        <div class="silo"><div class="box"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="6" width="13" height="12" rx="2"/><path d="M16 10l5-3v10l-5-3z"/></svg><span class="dupe">AGAIN</span></div><span class="name">Dev team</span></div>
        <div class="silo"><div class="box"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="6" width="13" height="12" rx="2"/><path d="M16 10l5-3v10l-5-3z"/></svg><span class="dupe">AGAIN</span></div><span class="name">QC team</span></div>
        <div class="silo"><div class="box"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="6" width="13" height="12" rx="2"/><path d="M16 10l5-3v10l-5-3z"/></svg><span class="dupe">AGAIN</span></div><span class="name">Ops team</span></div>
      </div>
    </div>
  </div>

  <div class="slide" data-slide="2">
    <span class="badge solution">Solution 1</span>
    <h1 style="margin-top:14px;">Check first. Reuse if it's already there.</h1>
    <p class="sub">Before anyone creates a new training video, the platform looks for a match. If one already exists, the team reuses it instead of making a new one.</p>
    <div class="visual">
      <div class="card">
        <div class="icon-circle ic-blue"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="6" width="13" height="12" rx="2"/><path d="M16 10l5-3v10l-5-3z"/></svg></div>
        <div class="label">A team wants to create a video</div>
      </div>
      <div class="flow-arrow"><svg viewBox="0 0 34 16" fill="none"><path d="M0 8h28M22 2l8 6-8 6" stroke="currentColor" stroke-width="2"/></svg></div>
      <div class="card">
        <div class="icon-circle ic-blue"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="11" cy="11" r="7"/><path d="m20 20-3.6-3.6"/></svg></div>
        <div class="label">We check the library</div>
      </div>
      <div class="flow-arrow"><svg viewBox="0 0 34 16" fill="none"><path d="M0 8h28M22 2l8 6-8 6" stroke="currentColor" stroke-width="2"/></svg></div>
      <div class="card">
        <div class="icon-circle ic-green"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M20 6 9 17l-5-5"/></svg></div>
        <div class="label">Found it — reuse it</div>
        <div class="sublabel">no need to create a new one</div>
      </div>
    </div>
  </div>

  <div class="slide" data-slide="3">
    <span class="badge problem">Problem 2</span>
    <h1 style="margin-top:14px;">Newer videos don't get shown to people watching the old one</h1>
    <p class="sub">A video covers a topic well. Later, a new video is made that adds more detail on the same topic. Someone watching the old video today has no way of knowing the newer video exists.</p>
    <div class="visual">
      <div class="compare">
        <div class="videocard">
          <div class="icon-circle sm ic-blue"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="6" width="13" height="12" rx="2"/><path d="M16 10l5-3v10l-5-3z"/></svg></div>
          <div class="label">Video — MLCC basics</div>
          <div class="sublabel">made in January</div>
        </div>
        <div class="flow-arrow"><svg viewBox="0 0 34 16" fill="none"><path d="M0 8h28M22 2l8 6-8 6" stroke="currentColor" stroke-width="2"/></svg></div>
        <div class="videocard">
          <div class="icon-circle sm ic-amber"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="6" width="13" height="12" rx="2"/><path d="M16 10l5-3v10l-5-3z"/></svg></div>
          <div class="label">Video — MLCC, new details</div>
          <div class="sublabel">made in June, covers more</div>
        </div>
      </div>
      <p class="sub" style="margin:22px 0 0; font-size:13.5px;">Both videos are correct — the second one just covers more. But today, nothing tells the viewer it exists.</p>
    </div>
  </div>

  <div class="slide" data-slide="4">
    <span class="badge solution">Solution 2</span>
    <h1 style="margin-top:14px;">Suggest the newer video while the old one plays</h1>
    <p class="sub">While someone watches the older video, we show the newer one as a suggestion, clearly tagged "Latest video" — so they know more information is available.</p>
    <div class="visual">
      <div class="card">
        <div class="icon-circle ic-blue"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="6" width="13" height="12" rx="2"/><path d="M16 10l5-3v10l-5-3z"/></svg></div>
        <div class="label">Now playing</div>
        <div class="sublabel">MLCC basics</div>
      </div>
      <div class="flow-arrow"><svg viewBox="0 0 34 16" fill="none"><path d="M0 8h28M22 2l8 6-8 6" stroke="currentColor" stroke-width="2"/></svg></div>
      <div class="videocard">
        <div class="banner info">🆕 Latest video</div>
        <div class="icon-circle sm ic-green"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="6" width="13" height="12" rx="2"/><path d="M16 10l5-3v10l-5-3z"/></svg></div>
        <div class="label">Suggested</div>
        <div class="sublabel">MLCC, new details</div>
      </div>
    </div>
  </div>

  <div class="slide" data-slide="5">
    <span class="badge problem">Problem 3</span>
    <h1 style="margin-top:14px;">Finding one fact means watching the whole video</h1>
    <p class="sub">Need just one fact from a long video? Today you have to fast-forward and rewind by hand, or watch the whole video again just to find it.</p>
    <div class="visual">
      <div class="icon-circle lg ic-red"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="12" r="9"/><path d="M12 7v5l4 2"/></svg></div>
    </div>
  </div>

  <div class="slide" data-slide="6">
    <span class="badge solution">Solution 3</span>
    <h1 style="margin-top:14px;">Just ask the video what you want to know</h1>
    <p class="sub">Type a question and get a direct answer right away. The AI has already gone through the whole video for you, so you don't have to.</p>
    <div class="visual">
      <div class="feature-row">
        <div class="feature">
          <div class="icon-circle sm ic-navy"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M8 6h13M8 12h13M8 18h13"/><circle cx="3" cy="6" r="1" fill="currentColor"/><circle cx="3" cy="12" r="1" fill="currentColor"/><circle cx="3" cy="18" r="1" fill="currentColor"/></svg></div>
          <div class="ftitle">Get a summary</div>
          <div class="fbody">The whole video in a few lines</div>
        </div>
        <div class="feature">
          <div class="icon-circle sm ic-navy"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M7 7h.01M3 11l8-8h8v8l-8 8-8-8z"/></svg></div>
          <div class="ftitle">See the key topics</div>
          <div class="fbody">What the video is really about</div>
        </div>
        <div class="feature">
          <div class="icon-circle sm ic-navy"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="8" r="4"/><path d="M4 20c0-4.4 3.6-7 8-7s8 2.6 8 7"/></svg></div>
          <div class="ftitle">Ask anything</div>
          <div class="fbody">Type your question, get an answer</div>
        </div>
      </div>
    </div>
  </div>

  <div class="slide" data-slide="7">
    <span class="badge problem">Problem 4</span>
    <h1 style="margin-top:14px;">We don't know if training is actually working</h1>
    <p class="sub">Once a video is uploaded, we have no idea who finished it, who stopped halfway, or which part confused people.</p>
    <div class="visual">
      <div class="stat-row">
        <div class="stat">
          <div class="icon-circle sm ic-red"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M4 20V12M10 20V6M16 20v-9M22 20H2"/></svg></div>
          <div class="lbltitle red">We don't know</div>
          <div class="lbl">how many people finished it</div>
        </div>
        <div class="stat">
          <div class="icon-circle sm ic-red"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="12" r="9"/><path d="M12 7v5l3 3"/></svg></div>
          <div class="lbltitle red">We don't know</div>
          <div class="lbl">where people stop watching</div>
        </div>
        <div class="stat">
          <div class="icon-circle sm ic-red"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M12 2v4M12 18v4M4.9 4.9l2.8 2.8M16.3 16.3l2.8 2.8M2 12h4M18 12h4M4.9 19.1l2.8-2.8M16.3 7.7l2.8-2.8"/></svg></div>
          <div class="lbltitle red">We don't know</div>
          <div class="lbl">what confuses people most</div>
        </div>
      </div>
    </div>
  </div>

  <div class="slide" data-slide="8">
    <span class="badge solution">Solution 4</span>
    <h1 style="margin-top:14px;">Show how well each video is working</h1>
    <p class="sub">We track how many people finished, where they stopped, and what they asked the AI most. That tells us what's working and what to improve.</p>
    <div class="visual">
      <div class="feature-row">
        <div class="feature">
          <div class="icon-circle sm ic-green"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M20 6 9 17l-5-5"/></svg></div>
          <div class="ftitle">Who finished it</div>
          <div class="fbody">See who watched to the end</div>
        </div>
        <div class="feature">
          <div class="icon-circle sm ic-green"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M4 20V12M10 20V6M16 20v-9M22 20H2"/></svg></div>
          <div class="ftitle">Where they stopped</div>
          <div class="fbody">The exact moment people leave</div>
        </div>
        <div class="feature">
          <div class="icon-circle sm ic-green"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="8" r="4"/><path d="M4 20c0-4.4 3.6-7 8-7s8 2.6 8 7"/></svg></div>
          <div class="ftitle">What they asked</div>
          <div class="fbody">The questions people needed answered</div>
        </div>
      </div>
    </div>
  </div>

  <div class="slide" data-slide="9">
    <p class="kicker">To sum up</p>
    <h1>A few problems. One simple solution for each.</h1>
    <p class="sub">One video library that doesn't repeat itself, points people to newer information, answers questions with AI, and shows what's actually working.</p>
    <div class="visual">
      <div class="recap-row">
        <div class="recap"><div class="icon-circle sm ic-blue"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="11" cy="11" r="7"/><path d="m20 20-3.6-3.6"/></svg></div><div class="rl">No repeats</div></div>
        <div class="recap"><div class="icon-circle sm ic-amber"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="6" width="13" height="12" rx="2"/><path d="M16 10l5-3v10l-5-3z"/></svg></div><div class="rl">Newer video suggested</div></div>
        <div class="recap"><div class="icon-circle sm ic-navy"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="8" r="4"/><path d="M4 20c0-4.4 3.6-7 8-7s8 2.6 8 7"/></svg></div><div class="rl">Ask the AI</div></div>
        <div class="recap"><div class="icon-circle sm ic-purple"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M4 20V12M10 20V6M16 20v-9M22 20H2"/></svg></div><div class="rl">Clear numbers</div></div>
      </div>
    </div>
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
  <div class="counter" id="counter">1 / 10</div>
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
