//app.js
/* AIDLC Workspace — demo.
   Sections: 1 helpers · 2 mock content · 3 state · 4 lifecycle engine · 5 chat assistant · 6 views · 7 events */
(function(){
'use strict';

/* ================= 1. Helpers ================= */
var $ = function(id){ return document.getElementById(id); };
function esc(t){ return String(t).replace(/[&<>"']/g, function(c){ return {'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]; }); }
function when(iso){ try { return new Date(iso).toLocaleString(undefined,{day:'numeric',month:'short',hour:'2-digit',minute:'2-digit'}); } catch(e){ return ''; } }
function now(){ return new Date().toISOString(); }
function tbl(heads, rows){
  return '<div class="table-wrap"><table><thead><tr>' + heads.map(function(h){return '<th scope="col">'+h+'</th>';}).join('') +
    '</tr></thead><tbody>' + rows.map(function(r){return '<tr>'+r.map(function(c){return '<td>'+c+'</td>';}).join('')+'</tr>';}).join('') + '</tbody></table></div>';
}
function mono(t){ return '<span class="mono">'+t+'</span>'; }
function pill(t, kind){ return '<span class="pill pill--'+kind+'">'+t+'</span>'; }
function skillTags(list){ return '<div class="skills">'+list.map(function(k){ return '<span class="skill">'+k+'</span>'; }).join('')+'</div>'; }
function barRow(label, pct){
  return '<div><div class="kv"><span>'+label+'</span><span class="strong">'+pct+'%</span></div><div class="bar" role="img" aria-label="'+label+' '+pct+' percent"><div class="bar__fill" style="width:'+pct+'%"></div><div class="bar__target" style="left:80%"></div></div></div>';
}
var CHECK = '<svg aria-hidden="true" width="13" height="13" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M3 8.5l3.2 3.2L13 5"/></svg>';

/* ================= 2. Mock content ================= */
var STEPS = [
  { id:1, name:'Plan', outputs:'Work Plan', skills:['AX-ai-workflow'], tasks:['Reading the feature request','Scanning the codebase for impacted modules','Breaking the work into tasks','Writing the Work Plan'] },
  { id:2, name:'Design', outputs:'SRS + SAD + Traceability', skills:['AX-ai-workflow'], tasks:['Deriving requirements from the Work Plan','Writing the SRS','Writing the SAD','Building the traceability matrix'] },
  { id:3, name:'Implement', outputs:'Code + Unit Test + Coverage', skills:['AX-ai-workflow'], tasks:['Generating code that follows the existing patterns','Writing unit tests','Running unit tests','Measuring coverage'] },
  { id:4, name:'Test (Dev)', outputs:'E2E TCs + Mock E2E', skills:['AX-ai-workflow'], tasks:['Writing E2E test cases from the SRS','Setting up mocked services','Running E2E against mocks','Collecting results'] },
  { id:5, name:'Review and PR', outputs:'AI Code Review + Bitbucket PR', skills:['AX-ai-workflow','AX-pull-request'], tasks:['Reviewing the code changes','Fixing review findings','Opening the Bitbucket pull request','Linking the PR to the ticket'] }
];
var VERIF = [
  { id:'e2e', name:'System E2E testing', what:'Run E2E tests on the real running application (not mocks)', skills:['AX-e2e-execution'], cta:'Run system E2E tests',
    tasks:['Connecting to the staging application','Running 26 E2E tests on the real application','Collecting logs and screenshots'] },
  { id:'bugs', name:'Bug reporting', what:'Analyze failures, create bug report and JIRA issues, update the work plan', skills:['AX-bug-report','AX-DOC-to-item'], cta:'Analyze failures and file bugs',
    tasks:['Analyzing 2 failures','Writing the bug report','Creating JIRA issues','Updating the work plan'] },
  { id:'nightly', name:'Nightly testing', what:'Autonomous test generation, hallucination detection, coverage improvement', skills:['AX-test-expert'], cta:'Simulate tonight\'s run',
    tasks:['Generating new tests','Running generated tests','Checking tests for hallucinations','Measuring coverage'] }
];
var MODULES = [['auth/','Sign-in and sessions',14],['orders/','Order list and status',31],['notifications/','Email and in-app notices',18],['reports/','Exports',9],['shared/','Utilities and UI kit',22]];

/* Deliverables for one step of one feature. Content is sample data shaped around the feature title. */
function deliverables(f, n){
  var T = esc(f.title), p = proj(f.pid), existing = p.repo.mode === 'existing', branch = 'feature/'+f.key;
  if (n === 1) return [
    { title:'Work Plan', summary:'6 tasks · 7.5 working days', html:
      '<p class="note">'+(existing ? 'Based on a scan of the existing codebase. 3 existing modules are affected; everything else stays untouched.' : 'New codebase. The project scaffold is created as part of WP-03.')+'</p>' +
      tbl(['Task','Description','Estimate'],[
        [mono('WP-01'), existing ? 'Analyze existing code and impacted modules' : 'Choose stack and project structure','0.5 d'],
        [mono('WP-02'),'Design the changes for "'+T+'"','1 d'],
        [mono('WP-03'), existing ? 'Implement backend changes' : 'Create scaffold and backend','2 d'],
        [mono('WP-04'),'Implement screen changes','2 d'],
        [mono('WP-05'),'Unit tests and E2E tests','1.5 d'],
        [mono('WP-06'),'Code review and pull request','0.5 d']]) +
      (existing ? '<h3 class="strong">Impact on existing code</h3>'+tbl(['Module','Change'],[[mono('orders/'),'Modified: new service method and route'],[mono('notifications/'),'Modified: one new notice template'],[mono('shared/'),'Reused: form and table components'],[mono('auth/, reports/'),'No change']]) : '') }
  ];
  if (n === 2) return [
    { title:'SRS', summary:'5 requirements', html: tbl(['ID','Requirement','Priority'],[
        [mono('SRS-01'),'A user can use "'+T+'" from the main screen','Must'],
        [mono('SRS-02'),'Input is validated and errors are shown clearly','Must'],
        [mono('SRS-03'),'The result is saved and still visible after reload','Must'],
        [mono('SRS-04'),'Existing behaviour is unchanged','Must'],
        [mono('SRS-05'),'Each action is written to the audit log','Should']]) },
    { title:'SAD', summary:'4 components touched', html: tbl(['Component','Change'],[
        ['Web client','One new screen, one changed screen'],
        ['API gateway','One new route with authorization'],
        ['Service layer','New logic for "'+T+'"'],
        ['Database','One new table, one migration']]) },
    { title:'Traceability', summary:'5 of 5 requirements traced', html: tbl(['Requirement','Component','Work plan task'],[
        [mono('SRS-01'),'Web client, API gateway',mono('WP-04')],
        [mono('SRS-02'),'Web client, Service layer',mono('WP-03')],
        [mono('SRS-03'),'Service layer, Database',mono('WP-03')],
        [mono('SRS-04'),'All',mono('WP-05')],
        [mono('SRS-05'),'Service layer',mono('WP-03')]]) }
  ];
  if (n === 3) return [
    { title:'Code', summary:'14 files changed on '+branch, html: tbl(['Area','New files','Modified files','Lines'],[
        ['Backend','4', existing ? '3' : '0','+486 −22'],['Screens','3', existing ? '2' : '0','+352 −14'],['Database','1','0','+38'],['Tests','1','0','+410']]) },
    { title:'Unit tests', summary:'48 passing, 0 failing', html: tbl(['Area','Tests','Passed','Failed'],[['Backend','27','27','0'],['Screens','15','15','0'],['Database','6','6','0']]) },
    { title:'Coverage', summary:'87% lines · target 80%', html: barRow('Overall',87)+barRow('Backend',90)+barRow('Screens',84)+barRow('Database',88)+'<p class="muted small">The orange mark is the 80% target.</p>' }
  ];
  if (n === 4) return [
    { title:'E2E test cases', summary:'8 test cases for 5 requirements', html: tbl(['ID','Scenario','Requirement'],[
        [mono('TC-01'),'Happy path for "'+T+'"',mono('SRS-01')],
        [mono('TC-02'),'Entry point is visible only to signed-in users',mono('SRS-01')],
        [mono('TC-03'),'Empty input is rejected with a clear message',mono('SRS-02')],
        [mono('TC-04'),'Invalid input is rejected with a clear message',mono('SRS-02')],
        [mono('TC-05'),'Result is still there after reload',mono('SRS-03')],
        [mono('TC-06'),'Existing main flow still works',mono('SRS-04')],
        [mono('TC-07'),'Existing reports still export',mono('SRS-04')],
        [mono('TC-08'),'Audit log entry is written',mono('SRS-05')]]) },
    { title:'Mock E2E results', summary:'8 of 8 passed against mocks', html: '<p class="note">Run against mocked services. The real application is tested in the verification phase.</p>' +
        tbl(['Test case','Result','Duration'],[[mono('TC-01'),'Pass','1.4 s'],[mono('TC-02'),'Pass','0.9 s'],[mono('TC-03'),'Pass','1.1 s'],[mono('TC-04'),'Pass','1.0 s'],[mono('TC-05'),'Pass','2.1 s'],[mono('TC-06'),'Pass','2.4 s'],[mono('TC-07'),'Pass','1.8 s'],[mono('TC-08'),'Pass','1.2 s']]) }
  ];
  return [
    { title:'AI code review', summary:'0 critical · 1 major (fixed) · 2 minor', html: tbl(['Severity','Finding','State'],[
        ['Major','New query loads all rows before filtering','Fixed'],['Minor','Duplicate validation helper','Open'],['Minor','Missing doc comment on the new service method','Open']]) },
    { title:'Bitbucket PR', summary:'PR #'+(40+f.num)+' · '+branch+' → '+esc(p.repo.branch), html: '<ul class="list">' +
        '<li><span class="strong">Title</span><span>'+f.key+' '+T+'</span></li>' +
        '<li><span class="strong">Branch</span><span class="mono">'+branch+' → '+esc(p.repo.branch)+'</span></li>' +
        '<li><span class="strong">Changes</span><span>14 files · +1,286 −36</span></li>' +
        '<li><span class="strong">Build</span><span>Passing</span></li>' +
        '<li><span class="strong">Reviewers</span><span>2 requested</span></li></ul>' }
  ];
}
function verResult(f, id){
  var p = proj(f.pid), k1 = p.key+'-'+(f.num*2+300), k2 = p.key+'-'+(f.num*2+301);
  if (id === 'e2e') return '<div class="tiles"><div class="tile"><span class="muted small">Passed</span><span class="stat">24 / 26</span></div><div class="tile"><span class="muted small">Failed</span><span class="stat">2</span></div><div class="tile"><span class="muted small">Environment</span><span class="stat">Staging</span></div></div>' +
    tbl(['Suite','Passed','Failed','Result'],[['New feature flow',7,1,pill('Fail','fail')],['Validation',5,0,'Pass'],['Persistence',4,1,pill('Fail','fail')],['Regression: existing features',6,0,'Pass'],['Audit log',2,0,'Pass']]);
  if (id === 'bugs') return '<p class="note note--done">Bug report created with steps to reproduce, logs and a suspected cause for each failure.</p>' +
    tbl(['JIRA issue','Bug','Severity','Work plan'],[
      ['<span class="strong">'+k1+'</span>','Second save within one second is lost','High',mono('WP-07')],
      ['<span class="strong">'+k2+'</span>','Result missing after reload on a slow connection','Medium',mono('WP-08')]]);
  return '<div class="tiles"><div class="tile"><span class="muted small">Tests generated</span><span class="stat">12</span><span class="muted small">9 kept</span></div><div class="tile"><span class="muted small">Hallucinations</span><span class="stat">3</span><span class="muted small">discarded</span></div><div class="tile"><span class="muted small">Coverage</span><span class="stat">87% → 90%</span></div></div>' +
    '<h3 class="strong">Discarded as hallucinated</h3><ul class="list">' +
    [['test_feature_sends_sms','No SMS channel in the SAD'],['test_feature_bulk_import','No bulk import in the SRS'],['test_feature_undo_endpoint','Calls an endpoint that does not exist']].map(function(r){ return '<li>'+mono(r[0])+'<span class="muted small">'+r[1]+'</span></li>'; }).join('')+'</ul>';
}

/* ================= 3. State ================= */
var KEY = 'aidlc-workspace-demo-v1', TICK = 750;
var S, runs = {}, ui = { tab:{}, changeOpen:false, error:'' };

function seed(){
  var s = { seq:2, projects:[], features:[], threads:{ home:[] } };
  var day = 86400000, t0 = Date.now() - 6*day;
  s.projects.push({ id:'p1', name:'Sample Project', key:'SMP', repo:{ mode:'existing', url:'bitbucket.org/sample-org/sample-project', branch:'develop', scanned:new Date(t0).toISOString() }, created:new Date(t0).toISOString(), nextNum:102 });
  var f = newFeatureObj('f1','p1','SMP',101,'Order tracking page','Customers can see the status of their orders on one page.');
  STEPS.forEach(function(x, i){ f.steps[x.id] = 'approved'; f.gates[x.id] = { at:new Date(t0 + (i+1)*day/2).toISOString() }; });
  f.ver = { e2e:'done', bugs:'done', nightly:'done' }; f.updated = new Date(t0 + 4*day).toISOString();
  f.msgs = [{ from:'ai', html:'This feature is complete: all 5 gates were approved and verification has run. Open any step on the left to see what was produced.', t:f.updated }];
  s.features.push(f);
  s.threads.home.push({ from:'ai', html:'Welcome. Pick a project on the left, or create a new one. Inside a project, tell me the feature you want and I\'ll take it through the 5 steps, stopping for your approval after each.', t:now() });
  s.threads.p1 = [{ from:'ai', html:'I\'ve scanned the <span class="strong">Sample Project</span> codebase. Describe a new feature here and I\'ll start with Step 1 · Plan.', t:now() }];
  return s;
}
function newFeatureObj(id, pid, key, num, title, request){
  var f = { id:id, pid:pid, num:num, key:key+'-'+num, title:title, request:request, steps:{}, rev:{}, gates:{}, feedback:{}, ver:{ e2e:'idle', bugs:'idle', nightly:'idle' }, busy:null, msgs:[], created:now(), updated:now() };
  STEPS.forEach(function(x){ f.steps[x.id] = 'locked'; f.rev[x.id] = 0; f.feedback[x.id] = []; });
  return f;
}
function load(){ try { var o = JSON.parse(localStorage.getItem(KEY)); if (o && Array.isArray(o.projects) && Array.isArray(o.features) && o.threads) return o; } catch(e){} return null; }
function save(){ try { localStorage.setItem(KEY, JSON.stringify(S)); } catch(e){} }

function proj(id){ return S.projects.filter(function(p){ return p.id === id; })[0]; }
function feat(id){ return S.features.filter(function(f){ return f.id === id; })[0]; }
function featsOf(pid){ return S.features.filter(function(f){ return f.pid === pid; }); }
function step(n){ return STEPS[n-1]; }
function verif(id){ return VERIF.filter(function(v){ return v.id === id; })[0]; }
function approvedCount(f){ return STEPS.filter(function(s){ return f.steps[s.id] === 'approved'; }).length; }
function gateStep(f){ var g = STEPS.filter(function(s){ return f.steps[s.id] === 'awaiting'; })[0]; return g ? g.id : 0; }
function runningStep(f){ var g = STEPS.filter(function(s){ return f.steps[s.id] === 'running'; })[0]; return g ? g.id : 0; }
function nextVer(f){ if (approvedCount(f) < 5) return null; var v = VERIF.filter(function(x){ return f.ver[x.id] === 'idle'; })[0]; return v ? v.id : null; }
function complete(f){ return approvedCount(f) === 5 && !nextVer(f) && !f.busy; }
function featStatus(f){
  if (f.busy) return { label:'AI working', kind:'run' };
  if (gateStep(f)) return { label:'Needs your approval', kind:'wait' };
  if (complete(f)) return { label:'Complete', kind:'done' };
  if (approvedCount(f) === 5) return { label:'Verification', kind:'idle' };
  return { label:'Not started', kind:'idle' };
}
function makeKey(name){ var k = name.replace(/[^A-Za-z]/g,'').slice(0,3).toUpperCase(); return k.length ? k : 'PRJ'; }

/* ================= 4. Lifecycle engine (simulated AI work) ================= */
function say(f, html, go, goLabel){ f.msgs.push({ from:'ai', html:html, go:go, goLabel:goLabel, t:now() }); f.updated = now(); }
function simulate(f, tasks, title, skills, done){
  if (runs[f.id]) clearInterval(runs[f.id].timer);
  var r = runs[f.id] = { i:0, tasks:tasks, title:title, skills:skills, timer:setInterval(function(){
    r.i++;
    if (r.i >= r.tasks.length){ clearInterval(r.timer); delete runs[f.id]; f.busy = null; done(); f.updated = now(); save(); render(); }
    else if (route().fid === f.id) render();
  }, TICK) };
}
function startStep(f, n, rework){
  var st = step(n);
  f.steps[n] = 'running'; f.busy = { type:'step', n:n, rework:!!rework };
  simulate(f, rework ? ['Reading your feedback','Updating the deliverables','Re-running the checks'] : st.tasks, (rework ? 'Reworking' : 'Working on')+' Step '+n+' · '+st.name, st.skills, function(){
    f.steps[n] = 'awaiting';
    say(f, 'Step '+n+' · '+st.name+(f.rev[n] ? ' (revision '+(f.rev[n]+1)+')' : '')+' is ready. I\'ve paused at stage gate '+n+' and need your approval to continue.', f.id+'-step-'+n, 'Review Step '+n+' · '+st.name);
  });
}
function approve(f, n){
  if (f.steps[n] !== 'awaiting') return;
  f.steps[n] = 'approved'; f.gates[n] = { at:now() };
  if (n < 5){ say(f, 'Gate '+n+' approved. Starting Step '+(n+1)+' · '+step(n+1).name+'.'); startStep(f, n+1); }
  else say(f, 'All 5 gates are approved, so development is complete. The verification phase is open. Shall I run the E2E tests on the real running application?', f.id+'-verify', 'Open verification');
}
function requestChanges(f, n, text){
  if (f.steps[n] !== 'awaiting') return;
  f.feedback[n].unshift({ t:now(), text:text }); f.rev[n]++;
  say(f, 'Understood. I\'ll rework Step '+n+' · '+step(n).name+' with your feedback and bring it back to this gate.');
  startStep(f, n, true);
}
function runVer(f, id){
  var v = verif(id);
  f.ver[id] = 'running'; f.busy = { type:'ver', v:id };
  simulate(f, v.tasks, v.name, v.skills, function(){
    f.ver[id] = 'done';
    if (id === 'e2e') say(f, 'System E2E finished: 24 of 26 passed, 2 failed on the real application. Shall I analyze them and file bugs?', f.id+'-verify', 'See E2E results');
    if (id === 'bugs') say(f, 'Bug report written, 2 JIRA issues created and the work plan updated. Nightly testing runs on its own at 02:00. Want to see a simulated run?', f.id+'-verify', 'See bug report');
    if (id === 'nightly') say(f, 'Nightly run finished: 12 tests generated, 3 discarded as hallucinated, coverage 87% → 90%. This feature is complete.', f.id+'-verify', 'See nightly results');
  });
}
function createProject(name, mode, url, branch){
  var id = 'p'+(S.seq++);
  var p = { id:id, name:name, key:makeKey(name), repo:{ mode:mode, url: mode === 'existing' ? url : '', branch: branch || 'main', scanned: mode === 'existing' ? now() : null }, created:now(), nextNum:101 };
  S.projects.push(p);
  S.threads[id] = [{ from:'ai', t:now(), html: mode === 'existing' ?
    'Project created. I connected to the repository and scanned the codebase. Describe a new feature here and I\'ll start with Step 1 · Plan.' :
    'Project created with a new, empty codebase. Describe the first feature here and I\'ll start with Step 1 · Plan.' }];
  return p;
}
function createFeature(p, title, request){
  var f = newFeatureObj('f'+(S.seq++), p.id, p.key, p.nextNum++, title, request || title);
  S.features.push(f);
  f.msgs.push({ from:'user', text:request || title, t:now() });
  say(f, 'Got it. I created '+f.key+' in '+esc(p.name)+' and I\'m starting Step 1 · Plan'+(p.repo.mode === 'existing' ? ' using the existing codebase' : '')+'.');
  startStep(f, 1);
  return f;
}

/* ================= 5. Chat assistant (scripted) ================= */
var YES = /^(approve|approved|yes|yep|ok|okay|sure|looks good|lgtm|go ahead|go|proceed|continue|next|run it|do it)\b/i;
function statusText(f){
  var g = gateStep(f);
  if (f.busy) return 'I\'m working right now. I\'ll pause and ask you when I\'m done.';
  if (g) return approvedCount(f)+' of 5 gates approved. Step '+g+' · '+step(g).name+' is waiting for your approval.';
  if (nextVer(f)) return 'All 5 gates approved. Next in verification: '+verif(nextVer(f)).name+'.';
  return 'This feature is complete: 5 approved steps and the verification phase.';
}
function chatSend(text){
  var r = route(), t = text.trim(); if (!t) return;
  if (r.v === 'feature'){
    var f = feat(r.fid); if (f.busy) return;
    f.msgs.push({ from:'user', text:t, t:now() });
    if (/^(help|\?)$/i.test(t) || /how does/i.test(t)) say(f, 'I do the work for each step, then stop at a stage gate. Nothing continues until you approve. You can approve here or with the button on the left. '+statusText(f));
    else if (/\b(status|where are we|progress)\b/i.test(t)) say(f, statusText(f));
    else if (gateStep(f)){ if (YES.test(t)) approve(f, gateStep(f)); else requestChanges(f, gateStep(f), t); }
    else if (nextVer(f)){
      var nv = nextVer(f), want = /night/i.test(t) ? 'nightly' : /bug|jira|file/i.test(t) ? 'bugs' : /e2e|test/i.test(t) ? 'e2e' : YES.test(t) ? nv : null;
      if (want === nv) runVer(f, nv); else say(f, 'The next thing I can do is: '+verif(nv).cta.toLowerCase()+'. Say "yes" or use the button.');
    }
    else say(f, 'This feature is complete. To build something else, go back to the project and describe a new feature.', f.pid, 'Back to project');
  } else if (r.v === 'project'){
    var p = proj(r.pid), th = S.threads[p.id] = S.threads[p.id] || [];
    th.push({ from:'user', text:t, t:now() });
    if (/^(help|\?)$/i.test(t) || /how does/i.test(t)){ th.push({ from:'ai', t:now(), html:'Describe a feature in a sentence or two. I\'ll create it in this project and take it through Plan, Design, Implement, Test and Review, stopping for your approval after each step.' }); }
    else {
      var title = t.replace(/\s+/g,' '), cut = title.split(/[,.;:]| with | so that | that | for the /)[0].trim(); if (cut.length >= 12) title = cut; title = title.charAt(0).toUpperCase() + title.slice(1); if (title.length > 60) title = title.slice(0,57).trim()+'…';
      var nf = createFeature(p, title, t);
      th.push({ from:'ai', t:now(), html:'Created feature '+nf.key+'. I\'m working on it now.', go:nf.id, goLabel:'Open '+nf.key });
      location.hash = nf.id;
    }
  } else {
    S.threads.home.push({ from:'user', text:t, t:now() });
    S.threads.home.push({ from:'ai', t:now(), html:'Open a project first, then describe the feature you want. You can also create a new project with the button on the left.' });
  }
  save(); render();
}

/* ================= 6. Views ================= */
function route(){
  var h = (location.hash || '').replace('#',''), m;
  if ((m = /^(p\d+)(-repo)?$/.exec(h)) && proj(m[1])) return { v:'project', pid:m[1], tab:m[2] ? 'repo' : 'features' };
  if ((m = /^(f\d+)(?:-step-([1-5])|-(verify))?$/.exec(h)) && feat(m[1])){
    var f = feat(m[1]), sel = m[2] ? +m[2] : m[3] ? 'verify' : (gateStep(f) || runningStep(f) || (approvedCount(f) === 5 ? 'verify' : Math.min(approvedCount(f)+1, 5)));
    return { v:'feature', fid:f.id, pid:f.pid, sel:sel };
  }
  return { v:'home' };
}

function renderCrumbs(r){
  var h = '<a href="#projects"'+(r.v === 'home' ? ' class="here"' : '')+'>Projects</a>';
  if (r.pid){ var p = proj(r.pid); h += '<span aria-hidden="true">/</span><a href="#'+p.id+'"'+(r.v === 'project' ? ' class="here"' : '')+'>'+esc(p.name)+'</a>'; }
  if (r.fid){ h += '<span aria-hidden="true">/</span><span class="here">'+esc(feat(r.fid).title)+'</span>'; }
  $('crumbs').innerHTML = h;
}

function renderSidebar(r){
  var h = '<div class="side-label">PROJECTS</div>';
  S.projects.forEach(function(p){
    var fs = featsOf(p.id), need = fs.filter(function(f){ return gateStep(f) && !f.busy; }).length, open = r.pid === p.id;
    h += '<a class="side-link" href="#'+p.id+'"'+(r.v === 'project' && open && r.tab === 'features' ? ' aria-current="page"' : '')+'><span>'+esc(p.name)+'</span>'+(need ? '<span class="tag tag--wait">'+need+'</span>' : '')+'</a>';
    if (open){
      h += '<div class="side-sub"><a class="side-link" href="#'+p.id+'-repo"'+(r.v === 'project' && r.tab === 'repo' ? ' aria-current="page"' : '')+'><span>Repository</span></a>';
      if (fs.length) h += '<div class="side-label">FEATURES</div>';
      fs.forEach(function(f){ var st = featStatus(f), tag = st.kind === 'wait' ? '<span class="tag tag--wait">Needs you</span>' : st.kind === 'run' ? '<span class="tag tag--run">Working</span>' : st.kind === 'done' ? '<span class="tag tag--done">Done</span>' : '';
        h += '<a class="side-link" href="#'+f.id+'"'+(r.fid === f.id ? ' aria-current="page"' : '')+'><span>'+esc(f.title)+'</span>'+tag+'</a>'; });
      h += '</div>';
    }
  });
  h += '<button type="button" class="btn side-new" data-act="new-project">New project</button>';
  $('sidebar').innerHTML = h;
}

function viewHome(){
  var h = '<div class="row between"><div><h1 class="page-title">Projects</h1><p class="muted">Each project has its own repository and features.</p></div><button type="button" class="btn btn--primary" data-act="new-project">New project</button></div><div class="grid">';
  S.projects.forEach(function(p){
    var fs = featsOf(p.id), need = fs.filter(function(f){ return gateStep(f) && !f.busy; }).length, busy = fs.filter(function(f){ return f.busy; }).length;
    h += '<a class="pcard" href="#'+p.id+'"><span class="pcard__name">'+esc(p.name)+'</span>' +
      '<span class="muted small">'+(p.repo.mode === 'existing' ? 'Existing codebase · '+esc(p.repo.url) : 'New codebase')+'</span>' +
      '<span class="pcard__foot"><span class="pill pill--idle">'+fs.length+' feature'+(fs.length === 1 ? '' : 's')+'</span>'+(need ? pill(need+' need'+(need === 1 ? 's' : '')+' approval','wait') : '')+(busy ? pill(busy+' in progress','run') : '')+'</span></a>';
  });
  return h + '</div>';
}

function viewProject(r){
  var p = proj(r.pid), fs = featsOf(p.id);
  var h = '<div class="stack" style="gap:6px"><span class="muted small">Project · '+p.key+'</span><h1 class="page-title">'+esc(p.name)+'</h1></div>' +
    '<nav class="tabs" aria-label="Project sections"><a class="tab" href="#'+p.id+'"'+(r.tab === 'features' ? ' aria-current="page"' : '')+'>Features</a><a class="tab" href="#'+p.id+'-repo"'+(r.tab === 'repo' ? ' aria-current="page"' : '')+'>Repository</a></nav>';
  if (r.tab === 'repo'){
    if (p.repo.mode === 'existing'){
      h += '<section class="card"><div class="row between"><h2 class="section-title">Connected repository</h2>'+pill('Connected','done')+'</div>' +
        '<ul class="list"><li><span class="strong">Repository</span><span class="mono">'+esc(p.repo.url)+'</span></li><li><span class="strong">Base branch</span><span class="mono">'+esc(p.repo.branch)+'</span></li><li><span class="strong">Last scanned</span><span>'+when(p.repo.scanned)+'</span></li></ul>' +
        '<div><button type="button" class="btn" data-act="rescan" data-id="'+p.id+'">Scan again</button></div></section>' +
        '<section class="card"><h2 class="section-title">What the AI found in the codebase</h2><p class="muted small">New features reuse these modules and follow the existing patterns. Sample scan data.</p>' +
        '<div class="tiles"><div class="tile"><span class="muted small">Files</span><span class="stat">94</span></div><div class="tile"><span class="muted small">Modules</span><span class="stat">5</span></div><div class="tile"><span class="muted small">Existing tests</span><span class="stat">212</span></div><div class="tile"><span class="muted small">Coverage</span><span class="stat">81%</span></div></div>' +
        tbl(['Module','Purpose','Files'], MODULES.map(function(m){ return [mono(m[0]), m[1], m[2]]; }))+'</section>';
    } else {
      h += '<div class="empty"><p class="strong" style="color:var(--ink)">New codebase</p><p>This project has no existing code. The AI creates the project scaffold in Step 3 of the first feature.</p></div>';
    }
    return h;
  }
  h += '<div class="row between"><p class="muted">Each feature goes through 5 steps and a verification phase.</p><button type="button" class="btn btn--primary" data-act="new-feature" data-id="'+p.id+'">New feature</button></div>';
  if (!fs.length) return h + '<div class="empty"><p class="strong" style="color:var(--ink)">No features yet</p><p>Describe the feature you want in the chat on the right, or use the New feature button.</p></div>';
  h += '<div class="stack" style="gap:10px">';
  fs.slice().reverse().forEach(function(f){
    var st = featStatus(f), a = approvedCount(f);
    h += '<a class="frow" href="#'+f.id+'"><span class="frow__text"><span class="frow__title">'+esc(f.title)+'</span><span class="muted small">'+f.key+' · updated '+when(f.updated)+'</span></span>' +
      '<span class="row" style="gap:10px"><span class="muted small">'+a+' of 5 steps</span><span class="minibar" role="img" aria-label="'+a+' of 5 steps approved"><div style="width:'+(a*20)+'%"></div></span>'+pill(st.label, st.kind)+'</span></a>';
  });
  return h + '</div>';
}

function taskList(f){
  var r = runs[f.id]; if (!r) return '';
  return '<section class="card"><div class="row between"><h2 class="section-title">'+r.title+'</h2>'+skillTags(r.skills)+'</div><ul class="tasks">' +
    r.tasks.map(function(t, i){ var c = i < r.i ? 'dot dot--done' : i === r.i ? 'dot dot--doing' : 'dot'; return '<li><span class="'+c+'">'+(i < r.i ? CHECK : '')+'</span><span'+(i > r.i ? ' class="muted"' : '')+'>'+t+'</span></li>'; }).join('') +
    '</ul><p class="muted small">The AI will pause and ask for your approval when this is done.</p></section>';
}

function viewFeature(r){
  var f = feat(r.fid), p = proj(f.pid), st = featStatus(f), nv = nextVer(f);
  var h = '<div class="stack" style="gap:6px"><span class="muted small">'+f.key+' · '+esc(p.name)+'</span><div class="row"><h1 class="page-title">'+esc(f.title)+'</h1>'+pill(st.label, st.kind)+'</div>' +
    (f.request && f.request !== f.title ? '<p class="muted">'+esc(f.request)+'</p>' : '')+'</div>';

  h += '<nav class="stepper" aria-label="Steps">';
  STEPS.forEach(function(s){
    var x = f.steps[s.id], label = x === 'approved' ? 'Approved' : x === 'awaiting' ? 'Needs you' : x === 'running' ? 'AI working' : 'Locked';
    h += '<a class="stp is-'+x+'" href="#'+f.id+'-step-'+s.id+'"'+(r.sel === s.id ? ' aria-current="true"' : '')+'><span class="stp__top"><span class="num">'+(x === 'approved' ? CHECK : s.id)+'</span><span class="stp__name">'+s.name+'</span></span><span class="stp__status">'+label+'</span></a>';
  });
  var vAll = approvedCount(f) === 5, vDone = vAll && !nv && !(f.busy && f.busy.type === 'ver'), vRun = f.busy && f.busy.type === 'ver';
  var vc = !vAll ? 'locked' : vDone ? 'done' : vRun ? 'running' : 'ready';
  h += '<a class="stp is-'+vc+'" href="#'+f.id+'-verify"'+(r.sel === 'verify' ? ' aria-current="true"' : '')+'><span class="stp__top"><span class="num">'+(vDone ? CHECK : 'V')+'</span><span class="stp__name">Verification</span></span><span class="stp__status">'+(vc === 'locked' ? 'Locked' : vc === 'done' ? 'Done' : vc === 'running' ? 'AI working' : 'Ready')+'</span></a></nav>';

  if (r.sel === 'verify') return h + viewVerify(f);

  var n = r.sel, s = step(n), x = f.steps[n];
  h += '<div class="row between"><div><h2 class="section-title">Step '+n+' · '+s.name+(f.rev[n] ? ' · revision '+(f.rev[n]+1) : '')+'</h2><p class="muted small">Outputs: '+s.outputs+'</p></div>'+skillTags(s.skills)+'</div>';
  if (x === 'locked') return h + '<div class="empty"><p>Locked. This step opens after stage gate '+(n-1)+' is approved.</p></div>';
  if (x === 'running') return h + taskList(f);

  if (x === 'awaiting'){
    h += '<section class="card card--wait"><h2 class="section-title">Stage gate '+n+' · your approval is needed</h2><p>The AI has paused. Review the deliverables below, then approve to start '+(n < 5 ? 'Step '+(n+1)+' · '+step(n+1).name : 'the verification phase')+'.</p>';
    if (ui.changeOpen) h += '<div class="field"><label for="gate-comment">What should change?</label><textarea id="gate-comment" rows="3"></textarea>'+(ui.error ? '<span class="error" role="alert">'+ui.error+'</span>' : '')+'</div>' +
      '<div class="row"><button type="button" class="btn btn--primary" data-act="send-changes" data-id="'+f.id+'" data-n="'+n+'">Send back to the AI</button><button type="button" class="btn" data-act="cancel-changes">Cancel</button></div>';
    else h += '<div class="row"><button type="button" class="btn btn--primary" data-act="approve" data-id="'+f.id+'" data-n="'+n+'">Approve</button><button type="button" class="btn" data-act="open-changes">Request changes</button></div>';
    h += '</section>';
  } else {
    h += '<p class="note note--done"><span class="strong">Stage gate '+n+' approved by you</span> · '+when(f.gates[n].at)+'</p>';
  }

  var ds = deliverables(f, n), tk = f.id+'-'+n, ti = Math.min(ui.tab[tk] || 0, ds.length-1);
  h += '<section class="card"><h2 class="section-title">Deliverables</h2>';
  if (ds.length > 1) h += '<div class="tabs" role="tablist">'+ds.map(function(d, i){ return '<button type="button" class="tab" role="tab" aria-selected="'+(i === ti)+'" data-act="tab" data-k="'+tk+'" data-i="'+i+'">'+d.title+'</button>'; }).join('')+'</div>';
  h += '<div class="stack" role="tabpanel"><p><span class="strong">'+ds[ti].title+'</span> <span class="muted">· '+ds[ti].summary+'</span></p>'+ds[ti].html+'</div></section>';

  if (f.feedback[n].length) h += '<section class="card"><h2 class="section-title">Changes you requested</h2><ul class="list">'+f.feedback[n].map(function(fb){ return '<li><span>"'+esc(fb.text)+'"</span><span class="muted small">'+when(fb.t)+'</span></li>'; }).join('')+'</ul></section>';
  return h;
}

function viewVerify(f){
  var h = '<div><h2 class="section-title">Verification phase</h2><p class="muted small">Begins after stage gate 5. Each stage runs when you ask for it.</p></div>';
  if (approvedCount(f) < 5) return h + '<div class="empty"><p>Locked. Verification opens after all 5 gates are approved ('+approvedCount(f)+' of 5 so far).</p></div>';
  var nv = nextVer(f);
  VERIF.forEach(function(v, i){
    var x = f.ver[v.id];
    h += '<section class="card"><div class="row between"><div><h2 class="section-title">'+(i+1)+' · '+v.name+'</h2><p class="muted small">'+v.what+'</p></div><div class="row" style="gap:8px">'+(x === 'done' ? pill('Done','done') : x === 'running' ? pill('AI working','run') : pill('Not run','idle'))+skillTags(v.skills)+'</div></div>';
    if (x === 'done') h += verResult(f, v.id);
    else if (x === 'running'){ var r = runs[f.id]; if (r) h += '<ul class="tasks">'+r.tasks.map(function(t, k){ var c = k < r.i ? 'dot dot--done' : k === r.i ? 'dot dot--doing' : 'dot'; return '<li><span class="'+c+'">'+(k < r.i ? CHECK : '')+'</span><span'+(k > r.i ? ' class="muted"' : '')+'>'+t+'</span></li>'; }).join('')+'</ul>'; }
    else if (v.id === nv && !f.busy) h += '<div><button type="button" class="btn btn--primary" data-act="ver" data-id="'+f.id+'" data-v="'+v.id+'">'+v.cta+'</button></div>';
    else h += '<p class="muted small">Available after the previous stage.</p>';
    h += '</section>';
  });
  return h;
}

function renderChat(r){
  var f = r.fid ? feat(r.fid) : null, p = r.pid ? proj(r.pid) : null;
  var msgs = f ? f.msgs : p ? (S.threads[p.id] = S.threads[p.id] || []) : S.threads.home;
  $('chat-head').innerHTML = '<span class="chat__title">AI assistant</span><span class="chat__ctx">'+(f ? 'Feature · '+esc(f.title) : p ? 'Project · '+esc(p.name) : 'All projects')+'</span>';
  var box = $('chat-msgs'), sig = (f ? f.id : p ? p.id : 'home')+':'+msgs.length;
  if (box.getAttribute('data-sig') !== sig){
    box.innerHTML = msgs.map(function(m){
      if (m.from === 'user') return '<div class="msg msg--user"><div class="bubble">'+esc(m.text)+'</div></div>';
      return '<div class="msg msg--ai"><div class="bubble">'+m.html+'</div>'+(m.go ? '<a class="msg__link" href="#'+m.go+'">'+esc(m.goLabel)+' →</a>' : '')+'</div>';
    }).join('');
    box.setAttribute('data-sig', sig);
    var sc = $('chat-scroll'); sc.scrollTop = sc.scrollHeight;
  }
  var chips = '', ph = 'Ask a question', dis = false;
  if (f){
    var g = gateStep(f), nv = nextVer(f);
    if (f.busy){ ph = 'The AI is working…'; dis = true; }
    else if (g){ ph = 'Type "approve" or your changes'; chips = '<button type="button" class="chip chip--primary" data-act="approve" data-id="'+f.id+'" data-n="'+g+'">Approve step '+g+'</button><button type="button" class="chip" data-act="chat-focus">Request changes</button><button type="button" class="chip" data-send="What\'s the status?">Status</button>'; }
    else if (nv){ ph = 'Type "yes" to continue'; chips = '<button type="button" class="chip chip--primary" data-act="ver" data-id="'+f.id+'" data-v="'+nv+'">'+verif(nv).cta+'</button><button type="button" class="chip" data-send="What\'s the status?">Status</button>'; }
    else { ph = 'This feature is complete'; }
  } else if (p){
    ph = 'Describe a new feature for '+p.name; chips = '<button type="button" class="chip" data-send="Build a KPI dashboard page with revenue, orders and active users for the last 30 days, with a date filter and a trend chart">Example: Build a KPI dashboard page with revenue, orders and active users</button><button type="button" class="chip" data-send="How does this work?">How does this work?</button>';
  } else { ph = 'Open a project to start'; chips = '<button type="button" class="chip" data-act="new-project">New project</button>'; }
  var input = $('chat-input'); input.placeholder = ph; input.disabled = dis; $('chat-send').disabled = dis;
  if ($('chips').getAttribute('data-h') !== chips){ $('chips').innerHTML = chips; $('chips').setAttribute('data-h', chips); }
}

function render(){
  var r = route();
  renderCrumbs(r); renderSidebar(r);
  var ta = $('gate-comment'), keep = ta ? { v:ta.value, focus:document.activeElement === ta } : null;
  $('main').innerHTML = '<div class="main__inner">'+(r.v === 'feature' ? viewFeature(r) : r.v === 'project' ? viewProject(r) : viewHome())+'</div>';
  var ta2 = $('gate-comment'); if (ta2 && keep){ ta2.value = keep.v; if (keep.focus) ta2.focus(); }
  renderChat(r);
}

/* ================= 7. Events ================= */
function openDialog(html){ var d = $('dlg'); d.innerHTML = html; if (d.showModal) d.showModal(); else d.setAttribute('open',''); var first = d.querySelector('input[type=text],textarea'); if (first) first.focus(); }
function closeDialog(){ var d = $('dlg'); if (d.close) d.close(); else d.removeAttribute('open'); }

var actions = {
  'new-project':function(){
    openDialog('<form data-form="project" novalidate><h2>New project</h2>' +
      '<div class="field"><label for="np-name">Project name</label><input type="text" id="np-name" maxlength="60"><span class="error" id="np-err" role="alert" hidden>Enter a project name.</span></div>' +
      '<fieldset class="field"><legend>Codebase</legend><label class="choice"><input type="radio" name="mode" value="existing" checked><span><span class="strong">Use an existing codebase</span><br><span class="muted small">Connect a repository. The AI scans it and builds new features on top.</span></span></label>' +
      '<label class="choice"><input type="radio" name="mode" value="new"><span><span class="strong">Start a new codebase</span><br><span class="muted small">The AI creates the project scaffold with the first feature.</span></span></label></fieldset>' +
      '<div id="np-repo" class="stack" style="gap:12px"><div class="field"><label for="np-url">Repository URL</label><input type="text" id="np-url" placeholder="bitbucket.org/your-team/your-repo"><span class="error" id="np-url-err" role="alert" hidden>Enter the repository URL.</span></div>' +
      '<div class="field"><label for="np-branch">Base branch</label><input type="text" id="np-branch" value="main"></div></div>' +
      '<div class="row" style="justify-content:flex-end"><button type="button" class="btn" data-act="close-dialog">Cancel</button><button type="submit" class="btn btn--primary">Create project</button></div></form>');
  },
  'new-feature':function(el){
    openDialog('<form data-form="feature" data-id="'+el.getAttribute('data-id')+'" novalidate><h2>New feature</h2>' +
      '<div class="field"><label for="nf-title">Feature name</label><input type="text" id="nf-title" maxlength="60" placeholder="KPI dashboard page"><span class="error" id="nf-err" role="alert" hidden>Enter a feature name.</span></div>' +
      '<div class="field"><label for="nf-desc">What should it do? (optional)</label><textarea id="nf-desc" rows="3"></textarea></div>' +
      '<p class="muted small">The AI starts Step 1 · Plan right away and stops for your approval when the plan is ready.</p>' +
      '<div class="row" style="justify-content:flex-end"><button type="button" class="btn" data-act="close-dialog">Cancel</button><button type="submit" class="btn btn--primary">Create and start</button></div></form>');
  },
  'close-dialog':closeDialog,
  rescan:function(el){ proj(el.getAttribute('data-id')).repo.scanned = now(); },
  approve:function(el){ var f = feat(el.getAttribute('data-id')), n = +el.getAttribute('data-n'); if (f.busy) return; f.msgs.push({ from:'user', text:'Approve', t:now() }); ui.changeOpen = false; approve(f, n); },
  'open-changes':function(){ ui.changeOpen = true; ui.error = ''; setTimeout(function(){ var t = $('gate-comment'); if (t) t.focus(); }, 0); },
  'cancel-changes':function(){ ui.changeOpen = false; ui.error = ''; },
  'send-changes':function(el){
    var f = feat(el.getAttribute('data-id')), n = +el.getAttribute('data-n'), text = ($('gate-comment').value || '').trim();
    if (!text){ ui.error = 'Describe what should change.'; return; }
    f.msgs.push({ from:'user', text:text, t:now() }); ui.changeOpen = false; ui.error = ''; $('gate-comment').value = ''; requestChanges(f, n, text);
  },
  ver:function(el){ var f = feat(el.getAttribute('data-id')), v = el.getAttribute('data-v'); if (f.busy || nextVer(f) !== v) return; f.msgs.push({ from:'user', text:verif(v).cta, t:now() }); runVer(f, v); },
  tab:function(el){ ui.tab[el.getAttribute('data-k')] = +el.getAttribute('data-i'); },
  'chat-focus':function(){ $('chat-input').focus(); },
  reset:function(){ Object.keys(runs).forEach(function(k){ clearInterval(runs[k].timer); delete runs[k]; }); S = seed(); ui = { tab:{}, changeOpen:false, error:'' }; location.hash = 'projects'; }
};

document.addEventListener('click', function(e){
  var el = e.target.closest('[data-act],[data-send]'); if (!el) return;
  if (el.hasAttribute('data-send')) return chatSend(el.getAttribute('data-send'));
  var a = actions[el.getAttribute('data-act')]; if (!a) return;
  a(el); save(); render();
});
document.addEventListener('input', function(e){ var er = e.target.parentNode && e.target.parentNode.querySelector('.error'); if (er && e.target.closest('.dialog')) er.hidden = true; });
document.addEventListener('change', function(e){
  if (e.target.name === 'mode'){ var box = $('np-repo'); if (box) box.hidden = e.target.value !== 'existing'; }
});
document.addEventListener('submit', function(e){
  var form = e.target, kind = form.getAttribute('data-form');
  e.preventDefault();
  if (form.id === 'chat-form'){ var i = $('chat-input'); chatSend(i.value); i.value = ''; i.style.height = ''; return; }
  if (kind === 'project'){
    var name = $('np-name').value.trim(), mode = form.querySelector('input[name=mode]:checked').value, url = $('np-url').value.trim();
    $('np-err').hidden = !!name; $('np-url-err').hidden = !(mode === 'existing' && !url);
    if (!name){ $('np-name').focus(); return; }
    if (mode === 'existing' && !url){ $('np-url').focus(); return; }
    var p = createProject(name, mode, url, $('np-branch').value.trim());
    closeDialog(); save(); location.hash = p.id; render();
  }
  if (kind === 'feature'){
    var title = $('nf-title').value.trim();
    $('nf-err').hidden = !!title;
    if (!title){ $('nf-title').focus(); return; }
    var f = createFeature(proj(form.getAttribute('data-id')), title, $('nf-desc').value.trim());
    closeDialog(); save(); location.hash = f.id; render();
  }
});
$('chat-input').addEventListener('keydown', function(e){ if (e.key === 'Enter' && !e.shiftKey){ e.preventDefault(); $('chat-send').click(); } });
$('chat-input').addEventListener('input', function(){ this.style.height = ''; this.style.height = Math.min(this.scrollHeight, 120)+'px'; });
window.addEventListener('hashchange', function(){ ui.changeOpen = false; ui.error = ''; render(); $('main').scrollTop = 0; });

/* ================= Start ================= */
S = load() || seed();
S.features.forEach(function(f){ if (f.busy){ if (f.busy.type === 'step') startStep(f, f.busy.n, f.busy.rework); else runVer(f, f.busy.v); } });
save(); render();
})();































//index.html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>AIDLC Workspace</title>
<link rel="stylesheet" href="styles.css">
</head>
<body>
<div class="app">

  <header class="topbar">
    <a class="brand" href="#projects">
      <svg aria-hidden="true" width="26" height="26" viewBox="0 0 28 28" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="3" width="22" height="22" rx="6"/><path d="M9 14h10M14 9v10"/></svg>
      <span>AIDLC</span>
    </a>
    <nav class="crumbs" id="crumbs" aria-label="Breadcrumb"></nav>
    <button type="button" class="btn btn--quiet" data-act="reset">Reset demo</button>
  </header>

  <aside class="sidebar" id="sidebar" aria-label="Projects"></aside>

  <main class="main" id="main"></main>

  <section class="chat" aria-label="AI assistant">
    <div class="chat__head" id="chat-head"></div>
    <div class="chat__scroll" id="chat-scroll"><div class="chat__msgs" id="chat-msgs"></div></div>
    <div class="composer">
      <div class="chips" id="chips"></div>
      <form class="composer__form" id="chat-form" autocomplete="off">
        <label class="sr-only" for="chat-input">Message the AI assistant</label>
        <textarea id="chat-input" rows="1"></textarea>
        <button type="submit" class="btn btn--primary" id="chat-send">Send</button>
      </form>
      <p class="composer__note">Demo with mock data. Replies are scripted.</p>
    </div>
  </section>

</div>

<dialog id="dlg" class="dialog"></dialog>

<script src="app.js"></script>
</body>
</html>
























//styles.css
/* AIDLC Workspace — demo */
:root{
  --bg:#F4F5F7; --surface:#FFFFFF; --border:#DDE1E8; --border-strong:#A9B1BF;
  --ink:#141A23; --muted:#566173;
  --accent:#1D55C4; --accent-hover:#143D8F; --accent-bg:#E4ECFB;
  --wait:#B4570A; --wait-text:#8A4104; --wait-bg:#FFF1DE;
  --done-bg:#E6EAF0; --fail-text:#9C1F17; --fail-bg:#FDECEA;
  --font-display:"Segoe UI",Roboto,Arial,sans-serif; --font-mono:ui-monospace,Menlo,monospace;
}
*{box-sizing:border-box}
html,body{height:100%}
body{margin:0;background:var(--bg);color:var(--ink);font-family:"Segoe UI",Roboto,Arial,sans-serif;font-size:14px;line-height:1.5}
h1,h2,h3,p{margin:0}
button,input,textarea{font:inherit;color:inherit}
a{color:var(--accent)}a:hover{color:var(--accent-hover)}
:focus-visible{outline:2px solid var(--accent);outline-offset:2px}
[hidden]{display:none!important}
.sr-only{position:absolute;width:1px;height:1px;overflow:hidden;clip:rect(0 0 0 0);white-space:nowrap}
.muted{color:var(--muted)}.small{font-size:13px}.strong{font-weight:600}
.mono{font-family:var(--font-mono);font-size:12px}
.row{display:flex;flex-wrap:wrap;gap:12px;align-items:center}
.between{justify-content:space-between}
.stack{display:flex;flex-direction:column;gap:16px}

/* ---------- Frame: sidebar | main | chat (400px) ---------- */
.app{height:100vh;height:100dvh;display:grid;grid-template-columns:240px minmax(0,1fr) 400px;grid-template-rows:auto minmax(0,1fr)}
.topbar{grid-column:1 / -1;background:var(--surface);border-bottom:1px solid var(--border);padding:0 20px;min-height:56px;display:flex;align-items:center;gap:20px}
.brand{display:flex;align-items:center;gap:10px;font-family:var(--font-display);font-weight:600;font-size:18px;color:var(--ink);text-decoration:none}
.brand:hover{color:var(--ink)}
.crumbs{flex:1;min-width:0;display:flex;flex-wrap:wrap;align-items:center;gap:6px;font-size:13px;color:var(--muted)}
.crumbs a{color:var(--muted);text-decoration:none;font-weight:500;padding:8px 2px}
.crumbs a:hover{color:var(--ink)}
.crumbs .here{color:var(--ink);font-weight:600}

.btn{min-height:44px;padding:0 18px;border-radius:10px;border:1px solid var(--border-strong);background:var(--surface);color:var(--ink);font-weight:600;cursor:pointer;text-decoration:none;display:inline-flex;align-items:center;justify-content:center;gap:8px}
.btn:hover{background:var(--bg);color:var(--ink)}
.btn--primary{background:var(--accent);border-color:var(--accent);color:#fff}
.btn--primary:hover{background:var(--accent-hover);border-color:var(--accent-hover);color:#fff}
.btn--quiet{min-height:38px;padding:0 14px;border-color:var(--border);font-weight:500}
.btn:disabled{background:var(--done-bg);border-color:var(--done-bg);color:var(--muted);cursor:not-allowed}

/* ---------- Sidebar ---------- */
.sidebar{background:var(--surface);border-right:1px solid var(--border);padding:16px 12px;overflow-y:auto;display:flex;flex-direction:column;gap:4px}
.side-label{font-size:12px;font-weight:600;letter-spacing:.06em;color:var(--muted);padding:8px 8px 6px;display:flex;justify-content:space-between;align-items:center}
.side-link{display:flex;align-items:center;justify-content:space-between;gap:8px;min-height:40px;padding:6px 8px;border-radius:8px;color:var(--ink);text-decoration:none;font-weight:500}
.side-link:hover{background:var(--bg);color:var(--ink)}
.side-link[aria-current="page"]{background:var(--accent-bg);color:var(--accent-hover);font-weight:600}
.side-link span:first-child{min-width:0;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
.side-sub{margin:0 0 6px 10px;padding-left:8px;border-left:1px solid var(--border);display:flex;flex-direction:column;gap:2px}
.side-sub .side-link{font-size:13px;min-height:36px}
.side-sub .side-label{padding:8px 8px 2px;font-size:11px}
.tag{flex:none;font-size:11px;font-weight:600;padding:1px 7px;border-radius:999px}
.tag--wait{background:var(--wait-bg);color:var(--wait-text)}
.tag--run{background:var(--accent-bg);color:var(--accent-hover)}
.tag--done{background:var(--done-bg);color:var(--ink)}
.side-new{margin-top:8px}

/* ---------- Main ---------- */
.main{overflow-y:auto;padding:28px 28px 48px}
.main__inner{max-width:900px;margin:0 auto;display:flex;flex-direction:column;gap:22px}
.page-title{font-family:var(--font-display);font-weight:600;font-size:28px;line-height:1.2;overflow-wrap:anywhere}
.section-title{font-family:var(--font-display);font-weight:600;font-size:17px}
.card{background:var(--surface);border:1px solid var(--border);border-radius:14px;padding:20px;display:flex;flex-direction:column;gap:14px}
.card--wait{border:2px solid var(--wait);padding:19px}
.pill{padding:3px 10px;border-radius:999px;font-size:12px;font-weight:600;white-space:nowrap}
.pill--done{background:var(--done-bg)}
.pill--wait{background:var(--wait-bg);color:var(--wait-text)}
.pill--run{background:var(--accent-bg);color:var(--accent-hover)}
.pill--idle{border:1px solid var(--border-strong);color:var(--muted)}
.pill--fail{background:var(--fail-bg);color:var(--fail-text)}
.skill{padding:2px 8px;border-radius:6px;background:var(--bg);border:1px solid var(--border);font-family:var(--font-mono);font-size:12px}
.skills{display:flex;flex-wrap:wrap;gap:6px}
.note{padding:12px 14px;border-radius:10px;background:var(--bg);font-size:13px}
.note--done{background:var(--done-bg)}
.empty{padding:32px;border:1px dashed var(--border-strong);border-radius:14px;text-align:center;color:var(--muted);display:flex;flex-direction:column;align-items:center;gap:14px}

/* project cards and feature rows */
.grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(260px,1fr));gap:16px}
.pcard{background:var(--surface);border:1px solid var(--border);border-radius:14px;padding:18px;display:flex;flex-direction:column;gap:8px;text-decoration:none;color:var(--ink);min-height:150px}
.pcard:hover{border-color:var(--border-strong);color:var(--ink)}
.pcard__name{font-family:var(--font-display);font-weight:600;font-size:19px;overflow-wrap:anywhere}
.pcard__foot{margin-top:auto;display:flex;flex-wrap:wrap;gap:6px;align-items:center}
.frow{background:var(--surface);border:1px solid var(--border);border-radius:12px;padding:14px 16px;display:flex;flex-wrap:wrap;align-items:center;justify-content:space-between;gap:10px 16px;text-decoration:none;color:var(--ink)}
.frow:hover{border-color:var(--border-strong);color:var(--ink)}
.frow__text{display:flex;flex-direction:column;min-width:0;flex:1 1 240px}
.frow__title{font-weight:600;overflow-wrap:anywhere}
.minibar{width:90px;height:6px;border-radius:3px;background:var(--border)}
.minibar>div{height:100%;border-radius:3px;background:var(--ink)}

/* tabs */
.tabs{display:flex;flex-wrap:wrap;gap:4px;border-bottom:1px solid var(--border)}
.tab{min-height:44px;padding:0 14px;border:0;border-bottom:2px solid transparent;background:none;font-weight:500;color:var(--muted);cursor:pointer;text-decoration:none;display:inline-flex;align-items:center}
.tab:hover{color:var(--ink)}
.tab[aria-selected="true"],.tab[aria-current="page"]{color:var(--ink);font-weight:600;border-bottom-color:var(--accent)}

/* stepper */
.stepper{display:grid;grid-template-columns:repeat(6,minmax(0,1fr));gap:8px}
.stp{min-width:0;display:flex;flex-direction:column;gap:6px;padding:10px;border:1px solid var(--border);border-radius:12px;background:var(--surface);text-decoration:none;color:var(--ink)}
.stp:hover{border-color:var(--border-strong);color:var(--ink)}
.stp__top{display:flex;align-items:center;gap:6px}
.stp .num{width:24px;height:24px;font-size:12px}
.num{flex:none;width:28px;height:28px;border-radius:50%;display:flex;align-items:center;justify-content:center;font-weight:600;font-size:13px;border:2px solid var(--border-strong);color:var(--muted);background:var(--surface)}
.stp__name{font-weight:600;font-size:13px}
.stp__status{font-size:12px;color:var(--muted);font-weight:500}
.stp.is-locked{background:var(--bg);border-style:dashed}
.stp.is-locked .stp__name{color:var(--muted);font-weight:500}
.stp.is-approved .num,.stp.is-done .num{background:var(--ink);border-color:var(--ink);color:#fff}
.stp.is-running .num{border-color:var(--accent);border-top-color:transparent;color:transparent;animation:spin .8s linear infinite}
.stp.is-running .stp__status{color:var(--accent-hover);font-weight:600}
.stp.is-awaiting .num{background:var(--wait);border-color:var(--wait);color:#fff}
.stp.is-awaiting .stp__status{color:var(--wait-text);font-weight:600}
.stp.is-ready .num{border-color:var(--accent);color:var(--accent)}
.stp[aria-current="true"]{border:2px solid var(--accent);padding:9px}
.stp.is-awaiting[aria-current="true"]{border-color:var(--wait)}
@keyframes spin{to{transform:rotate(360deg)}}
@media (prefers-reduced-motion:reduce){.stp.is-running .num,.dot--doing{animation:none!important}}

/* tables, lists, bars */
.table-wrap{overflow-x:auto}
table{width:100%;min-width:460px;border-collapse:collapse;font-size:13px}
th{text-align:left;color:var(--muted);font-weight:600;padding:8px 10px;border-bottom:1px solid var(--border)}
td{padding:10px;border-bottom:1px solid var(--border);vertical-align:top;overflow-wrap:anywhere}
th:first-child,td:first-child{padding-left:0}th:last-child,td:last-child{padding-right:0}
tbody tr:last-child td{border-bottom:0}
.list{margin:0;padding:0;list-style:none;border:1px solid var(--border);border-radius:10px}
.list>li{padding:10px 14px;display:flex;flex-wrap:wrap;justify-content:space-between;gap:8px;border-bottom:1px solid var(--border)}
.list>li:last-child{border-bottom:0}
.kv{display:flex;justify-content:space-between;gap:8px;font-size:13px}
.bar{position:relative;height:10px;border-radius:5px;background:var(--border)}
.bar__fill{height:100%;border-radius:5px;background:var(--ink)}
.bar__target{position:absolute;top:-4px;width:2px;height:18px;background:var(--wait)}
.tiles{display:flex;flex-wrap:wrap;gap:12px}
.tile{flex:1 1 140px;min-width:0;padding:14px;border-radius:10px;background:var(--bg);display:flex;flex-direction:column;gap:2px}
.stat{font-family:var(--font-display);font-weight:600;font-size:22px}
.tasks{margin:0;padding:0;list-style:none;display:flex;flex-direction:column;gap:10px}
.tasks li{display:flex;align-items:center;gap:10px}
.dot{width:18px;height:18px;border-radius:50%;border:2px solid var(--border-strong);flex:none;display:flex;align-items:center;justify-content:center;color:#fff}
.dot--done{background:var(--ink);border-color:var(--ink)}
.dot--doing{border-color:var(--accent);border-top-color:transparent;animation:spin .8s linear infinite}

/* forms */
.field{display:flex;flex-direction:column;gap:6px}
.field label,.field legend{font-size:13px;font-weight:600;padding:0}
.field input[type=text],.field textarea{width:100%;min-height:44px;padding:10px 12px;border:1px solid var(--border-strong);border-radius:10px;background:var(--surface)}
.field textarea{resize:vertical}
.error{color:var(--fail-text);font-size:13px;font-weight:500}
.choice{display:flex;align-items:flex-start;gap:10px;padding:12px;margin-top:8px;line-height:1.4;border:1px solid var(--border);border-radius:10px;cursor:pointer}
.choice input{flex:none;margin-top:2px;width:18px;height:18px;accent-color:var(--accent)}
fieldset{margin:0;padding:0;border:0}

.dialog{width:min(520px,calc(100vw - 32px));border:1px solid var(--border);border-radius:16px;padding:24px;color:var(--ink)}
.dialog::backdrop{background:rgba(20,26,35,.45)}
.dialog form{display:flex;flex-direction:column;gap:16px}
.dialog h2{font-family:var(--font-display);font-weight:600;font-size:20px}

/* ---------- Chat (right, 400px) ---------- */
.chat{background:var(--surface);border-left:1px solid var(--border);display:flex;flex-direction:column;min-height:0}
.chat__head{padding:14px 16px;border-bottom:1px solid var(--border);display:flex;flex-direction:column;gap:2px}
.chat__title{font-family:var(--font-display);font-weight:600;font-size:16px}
.chat__ctx{font-size:12px;color:var(--muted);overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
.chat__scroll{flex:1;min-height:0;overflow-y:auto}
.chat__msgs{padding:16px;display:flex;flex-direction:column;gap:14px}
.msg{display:flex;flex-direction:column;gap:6px;align-items:flex-start}
.msg--user{align-items:flex-end}
.bubble{max-width:88%;padding:9px 13px;border-radius:14px;overflow-wrap:anywhere}
.msg--user .bubble{background:var(--accent);color:#fff;border-bottom-right-radius:4px;white-space:pre-wrap}
.msg--ai .bubble{background:var(--bg);border-bottom-left-radius:4px}
.msg__link{font-weight:600;font-size:13px;padding:6px 0}
.composer{border-top:1px solid var(--border);padding:12px 16px 10px;display:flex;flex-direction:column;gap:10px}
.chips{display:flex;flex-wrap:wrap;gap:8px}
.chips:empty{display:none}
.chip{min-height:38px;padding:0 13px;border-radius:999px;border:1px solid var(--border-strong);background:var(--surface);font-weight:500;font-size:13px;cursor:pointer}
.chip:hover{background:var(--bg)}
.chip--primary{background:var(--accent);border-color:var(--accent);color:#fff;font-weight:600}
.chip--primary:hover{background:var(--accent-hover)}
.composer__form{display:flex;gap:8px;align-items:flex-end}
.composer__form textarea{flex:1;min-height:44px;max-height:120px;padding:10px 12px;border:1px solid var(--border-strong);border-radius:12px;resize:none;background:var(--surface)}
.composer__form textarea:disabled{background:var(--bg)}
.composer__note{font-size:12px;color:var(--muted);text-align:center}

/* ---------- Smaller screens ---------- */
@media (max-width:1180px){
  .stepper{grid-template-columns:repeat(3,minmax(0,1fr))}
  .app{grid-template-columns:minmax(0,1fr) 400px}
  .sidebar{display:none}
}
@media (max-width:820px){
  html,body{height:auto}
  .app{height:auto;display:flex;flex-direction:column}
  .main{overflow:visible;padding:20px 16px 28px}
  .chat{border-left:0;border-top:1px solid var(--border);height:75vh}
  .topbar{flex-wrap:wrap;padding:8px 16px}
}
