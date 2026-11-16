//app.js
/* AIDLC Usage Management — static mock.
   The idea: organise everything around FEATURES and where each one is in the AIDLC lifecycle.
   MOCK DATA stands in for what a backend would read from each repository's docs/ folder and pull request list. */
(function(){
'use strict';

var $ = function(id){ return document.getElementById(id); };
function esc(t){ return String(t).replace(/[&<>"']/g, function(c){ return {'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]; }); }

/* ================= Document types and lifecycle stages ================= */
var TYPES = {
  plan:  { name:'Work Plan',   plural:'Work Plans',  abbr:'PLAN', file:'work-plan.md',      outline:['Scope','Tasks and estimates','Dependencies','Impact on existing code'] },
  srs:   { name:'SRS',         plural:'SRS',         abbr:'SRS',  file:'srs.md',            outline:['Purpose','Functional requirements','Non-functional requirements','Acceptance criteria'] },
  sad:   { name:'SAD',         plural:'SAD',         abbr:'SAD',  file:'sad.md',            outline:['Context','Components','Data model','Interfaces','Decisions'] },
  tc:    { name:'Test Cases',  plural:'Test Cases',  abbr:'TEST', file:'e2e-test-cases.md', outline:['Test scope','Test cases','Traceability to SRS','Test data'] },
  review:{ name:'Code Review', plural:'Code Reviews',abbr:'REV',  file:'code-review.md',    outline:['Summary','Findings by severity','Fixes applied','Open items'] },
  bug:   { name:'Bug Report',  plural:'Bug Reports', abbr:'BUG',  file:'bug-report.md',     outline:['Failed tests','Steps to reproduce','Suspected cause','JIRA issues'] }
};
var TYPE_IDS = ['plan','srs','sad','tc','review','bug'];
// A feature sits in exactly one bucket: how far it has got.
var BUCKETS = [
  { name:'Planned',   hint:'Work Plan only' },
  { name:'Designed',  hint:'SRS and SAD done' },
  { name:'Tested',    hint:'Test Cases done' },
  { name:'In review', hint:'Pull request open' },
  { name:'Complete',  hint:'Pull request merged' }
];
var STEPS = [ { name:'Plan', types:['plan'] }, { name:'Design', types:['srs','sad'] }, { name:'Test', types:['tc'] }, { name:'Review', types:['review'] }, { name:'Pull request', pr:true } ];

/* ================= MOCK DATA ================= */
var TODAY = new Date(2026, 10, 16), DAY = 86400000;
var PEOPLE = ['Priya N.','Arjun K.','Meera S.','Rahul D.','Sana F.'];
var SEEDS = [
  { slug:'project1', name:'Project1', key:'P1', branch:'develop', pace:7,  features:['Order tracking page','Password reset by email','KPI dashboard page','Export invoices as CSV','Customer profile page','Order status notifications','Role based access','Audit log viewer'] },
  { slug:'project2', name:'Project2', key:'P2', branch:'main',    pace:9,  features:['Invoice generation','Tax calculation rules','Payment retry','Refund API','Monthly statement email','Currency rounding fix'] },
  { slug:'project3', name:'Project3', key:'P3', branch:'develop', pace:11, features:['Self service sign-up','Support ticket form','Knowledge base search','Saved addresses','Two step verification'] },
  { slug:'project4', name:'Project4', key:'P4', branch:'develop', pace:13, features:['Push notification settings','Offline order list','Biometric sign-in','Dark theme'] },
  { slug:'project5', name:'Project5', key:'P5', branch:'main',    pace:16, features:['Sales summary endpoint','Scheduled report export','Report access permissions'] }
];
function buildRepo(seed, ri){
  var r = { slug:seed.slug, name:seed.name, ws:'sample-org', key:seed.key, branch:seed.branch, docs:[], features:[] }, n = seed.features.length;
  seed.features.forEach(function(title, i){
    var ticket = seed.key+'-'+(101+i), fromEnd = n-1-i, depth = fromEnd === 0 ? 1 : fromEnd === 1 ? 2 : fromEnd === 2 ? 3 : 5;
    var start = +TODAY - (fromEnd*seed.pace + 3 + ri) * DAY, f = { ticket:ticket, title:title, repo:seed.slug, docs:{}, pr:null };
    function add(type, offset, rev){
      var d = { id:seed.slug+'-'+ticket+'-'+type, repo:seed.slug, type:type, ticket:ticket, feature:title, path:'docs/'+ticket+'/'+TYPES[type].file,
        author:PEOPLE[(i+ri+r.docs.length) % PEOPLE.length], date:new Date(Math.min(start + offset*DAY, +TODAY)),
        commit:(1048576 + (i+3)*7919*(r.docs.length+5) + ri*104729).toString(16).slice(0,7), revisions:rev, lines:80 + ((i*37 + r.docs.length*53) % 240) };
      r.docs.push(d); f.docs[type] = d;
    }
    add('plan', 0, 1 + (i % 2));
    if (depth >= 2){ add('srs', 1, 1 + (i % 3)); add('sad', 2, 1); }
    if (depth >= 3) add('tc', 4, 1 + ((i+1) % 2));
    if (depth >= 5){
      add('review', 5, 1); if (i % 3 === 0) add('bug', 7, 1);
      f.pr = { id:40+ri*20+i, status:(fromEnd > 3 || i % 2 === 0) ? 'Merged' : 'Open', author:PEOPLE[(i+ri+1) % PEOPLE.length], date:new Date(Math.min(start + 6*DAY, +TODAY)), branch:'feature/'+ticket, target:seed.branch };
    }
    f.bucket = f.pr ? (f.pr.status === 'Merged' ? 4 : 3) : f.docs.tc ? 2 : f.docs.srs ? 1 : 0;
    f.updated = Object.keys(f.docs).reduce(function(m, k){ return Math.max(m, +f.docs[k].date); }, 0);
    r.features.push(f);
  });
  r.features.reverse(); r.docs.sort(function(a, b){ return b.date - a.date; });
  return r;
}
var REPOS = SEEDS.map(buildRepo);

/* ================= State and helpers ================= */
var ui = { range:8 };
function resetView(){ ui.tab = 'features'; ui.sel = null; ui.bucket = null; ui.type = 'all'; ui.q = ''; ui.sort = { key:'date', dir:-1 }; ui.drawer = null; }
resetView();
function repo(slug){ return REPOS.filter(function(r){ return r.slug === slug; })[0]; }
function route(){ var m = /^repo-([a-z0-9-]+)$/.exec((location.hash || '').replace('#','')); return m && repo(m[1]) ? repo(m[1]) : null; }
function inRange(d){ return !ui.range || +d.date >= +TODAY - ui.range*7*DAY; }
function scope(docs){ return docs.filter(inRange); }
function count(docs, t){ return docs.filter(function(d){ return d.type === t; }).length; }
function prs(r){ return r.features.filter(function(f){ return f.pr && inRange(f.pr); }); }
function short(d){ return new Date(d).toLocaleDateString('en-GB', { day:'numeric', month:'short' }); }
function full(d){ return new Date(d).toLocaleDateString('en-GB', { day:'numeric', month:'short', year:'numeric' }); }
function ago(d){ var n = Math.round((TODAY - d) / DAY); return n <= 0 ? 'today' : n === 1 ? 'yesterday' : n < 14 ? n+' days ago' : Math.round(n/7)+' weeks ago'; }
function plural(n, one, many){ return n+' '+(n === 1 ? one : (many || one+'s')); }
function badge(t){ return '<span class="badge">'+TYPES[t].abbr+'</span>'; }
function periodText(){ return ui.range ? 'in the last '+ui.range+' weeks' : 'in total'; }
function toast(msg){ var t = $('toast'); t.textContent = msg; t.hidden = false; clearTimeout(toast.timer); toast.timer = setTimeout(function(){ t.hidden = true; }, 2600); }
function period(){ return '<div class="seg" role="group" aria-label="Period">'+[[4,'4 weeks'],[8,'8 weeks'],[12,'12 weeks'],[0,'All time']].map(function(o){ return '<button type="button" aria-pressed="'+(ui.range === o[0])+'" data-act="range" data-v="'+o[0]+'">'+o[1]+'</button>'; }).join('')+'</div>'; }
function track(f){ return '<span class="track" role="img" aria-label="Stage '+(f.bucket+1)+' of 5: '+BUCKETS[f.bucket].name+'">'+[0,1,2,3,4].map(function(k){ return '<i'+(k <= f.bucket ? ' class="on"' : '')+'></i>'; }).join('')+'</span>'; }
function nextStep(f){
  return f.bucket === 0 ? 'Next: the SRS and SAD have not been created yet.' : f.bucket === 1 ? 'Next: Test Cases have not been created yet.' : f.bucket === 2 ? 'Next: Code Review and the pull request are still to come.' :
    f.bucket === 3 ? 'Pull request #'+f.pr.id+' is open and waiting to be merged.' : 'Every document is in place and the pull request is merged.';
}
function figures(docs, prCount, clickable){
  var week = docs.filter(function(d){ return +d.date >= +TODAY - 7*DAY; });
  return '<div class="figures">'+['plan','srs','sad','tc'].map(function(t){ var w = count(week, t);
    return '<button type="button" class="fig" data-act="'+(clickable ? 'type' : 'none')+'" data-v="'+t+'"><span class="fig__n">'+count(docs, t)+'</span><span class="fig__l">'+TYPES[t].plural+'</span><span class="fig__d">'+(w ? '+'+w+' in the last 7 days' : 'None in the last 7 days')+'</span></button>'; }).join('') +
    '<button type="button" class="fig" data-act="'+(clickable ? 'tab' : 'none')+'" data-v="features"><span class="fig__n">'+prCount+'</span><span class="fig__l">Pull Requests</span><span class="fig__d">Raised by the AIDLC flow</span></button></div>';
}
function weeks(docs, n){ var out = [], i; for (i = n-1; i >= 0; i--){ var b = +TODAY - i*7*DAY - 6*DAY, e = b + 7*DAY; out.push({ begin:b, docs:docs.filter(function(d){ return +d.date >= b && +d.date < e; }) }); } return out; }

/* ================= Home: all repositories ================= */
function viewHome(){
  var all = scope([].concat.apply([], REPOS.map(function(r){ return r.docs; }))).sort(function(a, b){ return b.date - a.date; });
  var feats = [].concat.apply([], REPOS.map(function(r){ return r.features; })), done = feats.filter(function(f){ return f.bucket === 4; }).length;
  var h = '<div class="head"><div><h1>All repositories</h1><p class="lede"><b>'+plural(all.length,'document')+'</b> created '+periodText()+' across '+REPOS.length+' repositories. <b>'+done+' of '+feats.length+' features</b> are complete.</p></div>'+period()+'</div>' +
    figures(all, REPOS.reduce(function(n, r){ return n + prs(r).length; }, 0), false) + '<h2 class="h2">Choose a repository</h2><div class="repos">';
  REPOS.forEach(function(r){
    var d = scope(r.docs), by = BUCKETS.map(function(_, k){ return r.features.filter(function(f){ return f.bucket === k; }).length; }), w = weeks(r.docs, 8), max = Math.max.apply(null, w.map(function(x){ return x.docs.length; }).concat(1));
    h += '<a class="card repo" href="#repo-'+r.slug+'"><span class="repo__top"><span><span class="repo__name">'+esc(r.name)+'</span><br><span class="mono muted">'+r.ws+'/'+r.slug+'</span></span><span><span class="repo__n">'+d.length+'</span><br><span class="small muted">documents</span></span></span>' +
      '<span class="stack" role="img" aria-label="'+by.map(function(n, k){ return n+' '+BUCKETS[k].name; }).join(', ')+'">'+by.map(function(n, k){ return n ? '<i class="b'+(k+1)+'" style="flex:'+n+'"></i>' : ''; }).join('')+'</span>' +
      '<span class="key">'+by.map(function(n, k){ return n ? '<span class="b'+(k+1)+'"><i></i>'+n+' '+BUCKETS[k].name.toLowerCase()+'</span>' : ''; }).join('')+'</span>' +
      '<span class="repo__top" style="align-items:flex-end"><span class="small muted">'+plural(r.features.length,'feature')+' · updated '+ago(r.docs[0].date)+'</span><span class="spark" style="width:120px" role="img" aria-label="Documents per week, last 8 weeks">'+w.map(function(x){ return '<i style="height:'+Math.round(x.docs.length / max * 100)+'%"></i>'; }).join('')+'</span></span></a>';
  });
  return h + '</div><h2 class="h2">Latest documents</h2><div class="card">'+feed(all.slice(0, 6), true)+'</div>';
}

/* ================= Repository ================= */
function viewRepo(r){
  var docs = scope(r.docs), done = r.features.filter(function(f){ return f.bucket === 4; }).length, open = r.features.length - done;
  var h = '<nav class="crumbs" aria-label="Breadcrumb"><a href="#home">All repositories</a><span aria-hidden="true">/</span><label class="sr-only" for="repo-select">Repository</label><select id="repo-select">'+REPOS.map(function(x){ return '<option value="repo-'+x.slug+'"'+(x === r ? ' selected' : '')+'>'+esc(x.name)+'</option>'; }).join('')+'</select></nav>' +
    '<div class="head"><div><p class="eyebrow">bitbucket.org/'+r.ws+'/'+r.slug+' · '+r.branch+' · docs/</p><h1>'+esc(r.name)+'</h1><p class="lede"><b>'+plural(docs.length,'document')+'</b> '+periodText()+' for '+plural(r.features.length,'feature')+'. <b>'+done+' complete</b>, '+open+' in progress.</p></div>'+period()+'</div>' +
    figures(docs, prs(r).length, true) +
    '<div><h2 class="h2" style="margin-bottom:10px">Where features are now</h2><div class="pipe">'+BUCKETS.map(function(b, k){ var n = r.features.filter(function(f){ return f.bucket === k; }).length;
      return '<button type="button" class="stage b'+(k+1)+'" aria-pressed="'+(ui.bucket === k)+'" data-act="bucket" data-v="'+k+'"><span class="stage__bar"></span><span class="stage__n">'+n+'</span><span class="stage__l">'+b.name+'</span><span class="stage__h">'+b.hint+'</span></button>'; }).join('')+'</div></div>' +
    '<div class="tabs" role="tablist">'+[['features','Features'],['docs','Documents'],['activity','Activity']].map(function(t){ return '<button type="button" class="tab" role="tab" aria-selected="'+(ui.tab === t[0])+'" data-act="tab" data-v="'+t[0]+'">'+t[1]+'</button>'; }).join('')+'</div>';
  return h + (ui.tab === 'features' ? tabFeatures(r) : ui.tab === 'docs' ? tabDocs(r) : tabActivity(r));
}

function tabFeatures(r){
  var list = r.features.filter(function(f){ return ui.bucket === null || f.bucket === ui.bucket; });
  if (!list.length) return '<div class="card empty">No features are at this stage. <button type="button" class="btn" data-act="bucket" data-v="'+ui.bucket+'" style="margin-left:8px">Show all features</button></div>';
  var f = list.filter(function(x){ return x.ticket === ui.sel; })[0] || list[0];
  var h = '<div class="md"><div class="card flist" role="list">'+list.map(function(x){ var d = x.bucket === 4;
    return '<button type="button" role="listitem" class="fitem" aria-current="'+(x === f)+'" data-act="feature" data-v="'+x.ticket+'"><span class="fitem__top"><span class="fitem__title">'+esc(x.title)+'</span><span class="status status--'+(d ? 'done' : 'open')+'">'+BUCKETS[x.bucket].name+'</span></span>'+track(x)+'<span class="small muted">'+x.ticket+' · updated '+ago(x.updated)+'</span></button>'; }).join('')+'</div>';

  h += '<section class="card detail" aria-label="Feature details"><div><p class="eyebrow">docs/'+f.ticket+'/</p><h2>'+esc(f.title)+'</h2></div><p class="next'+(f.bucket === 4 ? ' next--done' : '')+'">'+nextStep(f)+'</p><ol class="steps">';
  STEPS.forEach(function(s, i){
    var body, done;
    if (s.pr){ done = !!f.pr; body = f.pr ? '<div class="doc" style="cursor:default"><span class="badge">PR</span><span class="doc__text"><span class="strong">Pull request #'+f.pr.id+' · '+f.pr.status+'</span><span class="small muted mono">'+f.pr.branch+' → '+f.pr.target+'</span><span class="small muted">'+esc(f.pr.author)+' · '+short(f.pr.date)+'</span></span></div>' : '<div class="todo">Not raised yet</div>'; }
    else { done = s.types.every(function(t){ return f.docs[t]; }); body = s.types.map(function(t){ return docCard(f.docs[t], t); }).join(''); }
    h += '<li class="step '+(done ? 'is-done' : 'is-todo')+'"><span class="step__dot">'+(done ? '✓' : i+1)+'</span><div class="step__body"><span class="step__name">'+s.name+'</span>'+body+'</div></li>';
  });
  if (f.docs.bug) h += '<li class="step is-done"><span class="step__dot">✓</span><div class="step__body"><span class="step__name">Verification</span>'+docCard(f.docs.bug, 'bug')+'</div></li>';
  return h + '</ol></section></div>';
}
function docCard(d, t, showFeature){
  if (!d) return '<div class="todo">'+TYPES[t].name+' not created yet</div>';
  return '<button type="button" class="doc" data-act="doc" data-v="'+d.id+'">'+badge(d.type)+'<span class="doc__text"><span class="strong">'+TYPES[d.type].name+(showFeature ? ' · '+esc(d.feature) : '')+'</span><span class="small muted">'+(showFeature === 'repo' ? esc(repo(d.repo).name)+' · ' : '')+esc(d.author)+' · '+short(d.date)+' · '+plural(d.revisions,'revision')+'</span></span><span class="doc__open">Open</span></button>';
}

function docList(r){
  var q = ui.q.trim().toLowerCase(), get = { path:function(d){ return d.path; }, type:function(d){ return TYPES[d.type].name; }, feature:function(d){ return d.feature.toLowerCase(); }, author:function(d){ return d.author; }, date:function(d){ return +d.date; }, rev:function(d){ return d.revisions; } }[ui.sort.key];
  return scope(r.docs).filter(function(d){ return (ui.type === 'all' || d.type === ui.type) && (!q || (d.path+' '+d.feature+' '+d.author+' '+d.ticket).toLowerCase().indexOf(q) >= 0); })
    .sort(function(a, b){ var x = get(a), y = get(b); return (x < y ? -1 : x > y ? 1 : 0) * ui.sort.dir; });
}
function docRows(r){
  var list = docList(r);
  return list.length ? list.map(function(d){ return '<tr><td><button type="button" class="row-btn" data-act="doc" data-v="'+d.id+'">'+badge(d.type)+'<span class="mono">'+esc(d.path)+'</span></button></td><td>'+esc(d.feature)+'</td><td>'+esc(d.author)+'</td><td class="muted">'+full(d.date)+'</td><td class="num">'+d.revisions+'</td></tr>'; }).join('')
    : '<tr><td colspan="5" class="empty">No documents match.</td></tr>';
}
function tabDocs(r){
  var docs = scope(r.docs);
  function th(key, label, num){ var on = ui.sort.key === key; return '<th'+(num ? ' class="num"' : '')+(on ? ' aria-sort="'+(ui.sort.dir > 0 ? 'ascending' : 'descending')+'"' : '')+'><button type="button" class="th-btn" data-act="sort" data-v="'+key+'">'+label+(on ? ' <span aria-hidden="true">'+(ui.sort.dir > 0 ? '↑' : '↓')+'</span>' : '')+'</button></th>'; }
  return '<div class="bar"><div class="chips" role="group" aria-label="Document type"><button type="button" class="chip" aria-pressed="'+(ui.type === 'all')+'" data-act="type" data-v="all">All '+docs.length+'</button>' +
    TYPE_IDS.map(function(t){ return '<button type="button" class="chip" aria-pressed="'+(ui.type === t)+'" data-act="type" data-v="'+t+'">'+TYPES[t].name+' '+count(docs, t)+'</button>'; }).join('')+'</div>' +
    '<label class="sr-only" for="doc-search">Search documents</label><input type="text" class="search" id="doc-search" placeholder="Filter by file, feature, ticket or person" value="'+esc(ui.q)+'"></div>' +
    '<div class="card table-wrap"><table><thead><tr>'+th('path','File')+th('feature','Feature')+th('author','Last changed by')+th('date','Last changed')+th('rev','Revisions',true)+'</tr></thead><tbody id="doc-rows">'+docRows(r)+'</tbody></table></div>';
}

function chart(docs){
  var n = ui.range || 12, w = weeks(docs, n), max = Math.max.apply(null, w.map(function(x){ return x.docs.length; }).concat(1));
  return '<div class="chart">'+w.map(function(x){ var t = x.docs.length;
    return '<button type="button" class="col" data-act="none" aria-label="Week of '+short(x.begin)+': '+plural(t,'document')+'"><span class="tip"><b>Week of '+short(x.begin)+'</b><br>'+(t ? TYPE_IDS.filter(function(k){ return count(x.docs, k); }).map(function(k){ return TYPES[k].name+': '+count(x.docs, k); }).join('<br>') : 'No documents')+'</span>' +
      '<span class="col__n">'+(t || '')+'</span>'+(t ? '<span class="col__bar" style="height:'+Math.round(t / max * 130)+'px"></span>' : '')+'</button>'; }).join('') +
    '</div><div class="xaxis">'+w.map(function(x, k){ return '<span>'+(n > 8 && k % 2 ? '' : short(x.begin))+'</span>'; }).join('')+'</div>';
}
function feed(docs, showRepo){
  if (!docs.length) return '<p class="empty">No documents in this period.</p>';
  var h = '<ul class="feed">', last = '';
  docs.forEach(function(d){
    var n = Math.round((TODAY - d.date) / DAY), group = n < 7 ? 'This week' : n < 14 ? 'Last week' : 'Earlier';
    if (group !== last){ h += '<li class="feed__day">'+group+'</li>'; last = group; }
    h += '<li>'+docCard(d, d.type, showRepo ? 'repo' : true)+'</li>';
  });
  return h + '</ul>';
}
function tabActivity(r){
  var docs = scope(r.docs);
  return '<section class="card" style="padding:22px"><h2 class="h2" style="margin-bottom:14px">Documents created per week</h2>'+chart(docs)+'<p class="small muted" style="margin-top:10px">Hover a week to see its breakdown by document type.</p></section>' +
    '<h2 class="h2">Timeline</h2><div class="card">'+feed(docs.slice(0, 14), false)+'</div>';
}

/* ================= Drawer ================= */
function findDoc(id){ var d; REPOS.forEach(function(r){ r.docs.forEach(function(x){ if (x.id === id) d = x; }); }); return d; }
function openDoc(id, keep){
  var d = findDoc(id); if (!d) return;
  var r = repo(d.repo), t = TYPES[d.type];
  if (!keep || !ui.drawer){ var cur = route(), list = (cur === r && ui.tab === 'docs') ? docList(r) : r.docs.filter(function(x){ return x.ticket === d.ticket; }).sort(function(a, b){ return a.date - b.date; }); if (list.indexOf(d) < 0) list = [d]; ui.drawer = { ids:list.map(function(x){ return x.id; }) }; }
  ui.drawer.id = id;
  var i = ui.drawer.ids.indexOf(id), n = ui.drawer.ids.length, el = $('drawer'), was = el.hidden;
  el.innerHTML = '<div class="drawer__head"><div><p class="eyebrow">'+d.ticket+' · '+esc(d.feature)+'</p><h2>'+t.name+'</h2></div><button type="button" class="btn btn--icon" data-act="close" aria-label="Close">✕</button></div>' +
    (n > 1 ? '<div class="nav"><button type="button" class="btn" data-act="step" data-v="-1"'+(i <= 0 ? ' disabled' : '')+'>← Previous</button><span class="small muted">'+(i+1)+' of '+n+'</span><button type="button" class="btn" data-act="step" data-v="1"'+(i >= n-1 ? ' disabled' : '')+'>Next →</button></div>' : '') +
    '<div class="pathbox">'+r.ws+'/'+r.slug+' @ '+r.branch+'<br>'+esc(d.path)+'</div>' +
    '<ul class="kv"><li><span class="strong">Last changed by</span><span>'+esc(d.author)+'</span></li><li><span class="strong">Last changed</span><span>'+full(d.date)+'</span></li><li><span class="strong">Last commit</span><span class="mono">'+d.commit+'</span></li><li><span class="strong">Revisions</span><span>'+d.revisions+'</span></li><li><span class="strong">Length</span><span>'+d.lines+' lines</span></li></ul>' +
    '<div><h3 class="h2" style="margin-bottom:8px">Sections</h3><ol class="outline">'+t.outline.map(function(s){ return '<li>'+s+'</li>'; }).join('')+'</ol></div>' +
    '<div class="bar"><button type="button" class="btn" data-act="copy" data-v="'+esc(d.path)+'">Copy file path</button><button type="button" class="btn" data-act="bb">Open in Bitbucket</button></div>' +
    '<p class="small muted">In the real application this panel also shows the rendered document.</p>';
  el.hidden = false; $('scrim').hidden = false;
  if (was) el.querySelector('[data-act="close"]').focus();
}
function closeDoc(){ $('drawer').hidden = true; $('scrim').hidden = true; ui.drawer = null; }
function stepDoc(dir){ if (!ui.drawer) return; var i = ui.drawer.ids.indexOf(ui.drawer.id) + dir; if (i >= 0 && i < ui.drawer.ids.length) openDoc(ui.drawer.ids[i], true); }

/* ================= Finder (global search) ================= */
function finder(q){
  var p = $('finder-panel'); q = q.trim().toLowerCase();
  if (q.length < 2){ p.hidden = true; return; }
  function has(s){ return s.toLowerCase().indexOf(q) >= 0; }
  var rs = REPOS.filter(function(r){ return has(r.name+' '+r.slug); }).slice(0, 3), fs = [], ds = [];
  REPOS.forEach(function(r){ r.features.forEach(function(f){ if (has(f.title+' '+f.ticket)) fs.push(f); }); r.docs.forEach(function(d){ if (has(d.path+' '+TYPES[d.type].name+' '+d.feature)) ds.push(d); }); });
  var h = '';
  if (rs.length) h += '<div class="finder__group">REPOSITORIES</div>'+rs.map(function(r){ return '<button type="button" class="finder__item" data-act="go-repo" data-v="'+r.slug+'"><span class="badge">REPO</span><span><span class="strong">'+esc(r.name)+'</span><span class="small muted">'+plural(r.features.length,'feature')+' · '+plural(r.docs.length,'document')+'</span></span></button>'; }).join('');
  if (fs.length) h += '<div class="finder__group">FEATURES</div>'+fs.slice(0, 4).map(function(f){ return '<button type="button" class="finder__item" data-act="go-feature" data-v="'+f.repo+'|'+f.ticket+'"><span class="badge">FEAT</span><span><span class="strong">'+esc(f.title)+'</span><span class="small muted">'+esc(repo(f.repo).name)+' · '+BUCKETS[f.bucket].name+'</span></span></button>'; }).join('');
  if (ds.length) h += '<div class="finder__group">DOCUMENTS</div>'+ds.slice(0, 4).map(function(d){ return '<button type="button" class="finder__item" data-act="go-doc" data-v="'+d.id+'">'+badge(d.type)+'<span><span class="strong">'+TYPES[d.type].name+' · '+esc(d.feature)+'</span><span class="small muted mono">'+esc(d.path)+'</span></span></button>'; }).join('');
  p.innerHTML = h || '<p class="empty">Nothing matches "'+esc(q)+'".</p>'; p.hidden = false;
}
function closeFinder(){ $('finder-panel').hidden = true; $('finder-input').value = ''; }
function go(slug, after){ if (location.hash === '#repo-'+slug){ after(); render(); } else { pending = after; location.hash = 'repo-'+slug; } }
var pending = null;

/* ================= Render and events ================= */
function render(){ var r = route(); $('main').innerHTML = r ? viewRepo(r) : viewHome(); }
var actions = {
  range:function(v){ ui.range = +v; },
  tab:function(v){ ui.tab = v; },
  type:function(v){ ui.tab = 'docs'; ui.type = v; },
  bucket:function(v){ ui.bucket = ui.bucket === +v ? null : +v; ui.tab = 'features'; ui.sel = null; },
  feature:function(v){ ui.sel = v; },
  sort:function(v){ if (ui.sort.key === v) ui.sort.dir = -ui.sort.dir; else ui.sort = { key:v, dir:(v === 'date' || v === 'rev') ? -1 : 1 }; },
  doc:function(v){ openDoc(v); return false; },
  close:function(){ closeDoc(); return false; },
  step:function(v){ stepDoc(+v); return false; },
  copy:function(v){ try { navigator.clipboard.writeText(v).then(function(){ toast('File path copied'); }, function(){ toast('Copy is not available here'); }); } catch(e){ toast('Copy is not available here'); } return false; },
  bb:function(){ toast('Mock: this would open the file in Bitbucket'); return false; },
  sync:function(){ var b = $('sync-btn'); b.disabled = true; b.innerHTML = '<span class="spin"></span>Syncing'; $('sync-label').textContent = 'Reading docs/ folders…';
    setTimeout(function(){ b.disabled = false; b.textContent = 'Sync'; $('sync-label').textContent = 'Synced just now'; toast('Synced '+REPOS.length+' repositories. No new documents (mock).'); }, 1400); return false; },
  'go-repo':function(v){ closeFinder(); location.hash = 'repo-'+v; return false; },
  'go-feature':function(v){ var p = v.split('|'); closeFinder(); go(p[0], function(){ ui.tab = 'features'; ui.bucket = null; ui.sel = p[1]; }); return false; },
  'go-doc':function(v){ var d = findDoc(v); closeFinder(); go(d.repo, function(){ ui.tab = 'features'; ui.bucket = null; ui.sel = d.ticket; setTimeout(function(){ openDoc(v); }, 0); }); return false; },
  none:function(){ return false; }
};
document.addEventListener('click', function(e){
  if (e.target.id === 'scrim') return closeDoc();
  if (!e.target.closest('.finder')) $('finder-panel').hidden = true;
  var el = e.target.closest('[data-act]'); if (!el) return;
  var name = el.getAttribute('data-act'), v = el.getAttribute('data-v'), a = actions[name]; if (!a) return;
  if (a(v) === false) return;
  render();
  var again = document.querySelector('#main [data-act="'+name+'"][data-v="'+v+'"]'); if (again) again.focus({ preventScroll:true });
});
document.addEventListener('input', function(e){
  if (e.target.id === 'doc-search'){ ui.q = e.target.value; var r = route(); if (r) $('doc-rows').innerHTML = docRows(r); }
  if (e.target.id === 'finder-input') finder(e.target.value);
});
document.addEventListener('change', function(e){ if (e.target.id === 'repo-select') location.hash = e.target.value; });
$('finder-input').addEventListener('keydown', function(e){
  if (e.key === 'Enter'){ var first = $('finder-panel').querySelector('.finder__item'); if (first && !$('finder-panel').hidden) first.click(); }
  if (e.key === 'Escape') closeFinder();
  if (e.key === 'ArrowDown'){ var f = $('finder-panel').querySelector('.finder__item'); if (f){ e.preventDefault(); f.focus(); } }
});
$('finder-panel').addEventListener('keydown', function(e){
  var items = [].slice.call(this.querySelectorAll('.finder__item')), i = items.indexOf(document.activeElement);
  if (e.key === 'ArrowDown' && items[i+1]){ e.preventDefault(); items[i+1].focus(); }
  if (e.key === 'ArrowUp'){ e.preventDefault(); (items[i-1] || $('finder-input')).focus(); }
  if (e.key === 'Escape'){ closeFinder(); $('finder-input').focus(); }
});
document.addEventListener('keydown', function(e){
  if ($('drawer').hidden) return;
  if (e.key === 'Escape') closeDoc();
  if (e.key === 'ArrowRight') stepDoc(1);
  if (e.key === 'ArrowLeft') stepDoc(-1);
});
window.addEventListener('hashchange', function(){ closeDoc(); resetView(); if (pending){ pending(); pending = null; } render(); window.scrollTo(0, 0); });
render();
})();


























//index.html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>AIDLC Usage Management</title>
<link rel="stylesheet" href="styles.css">
</head>
<body>

<header class="top">
  <div class="top__in">
    <a class="logo" href="#home"><span class="logo__mark" aria-hidden="true"></span>AIDLC Usage Management</a>
    <div class="finder">
      <label class="sr-only" for="finder-input">Search repositories, features and documents</label>
      <input type="text" id="finder-input" placeholder="Search repositories, features and documents" autocomplete="off">
      <div class="finder__panel" id="finder-panel" hidden></div>
    </div>
    <div class="top__right">
      <span class="muted small" id="sync-label">Synced 12 min ago</span>
      <button type="button" class="btn" id="sync-btn" data-act="sync">Sync</button>
      <span class="mock">Mock data</span>
    </div>
  </div>
</header>

<main class="page" id="main"></main>

<aside class="drawer" id="drawer" aria-label="Document details" hidden></aside>
<div class="scrim" id="scrim" hidden></div>
<div class="toast" id="toast" role="status" aria-live="polite" hidden></div>

<script src="app.js"></script>
</body>
</html>














//styles.css
/* AIDLC Usage Management — static mock. Calm, near-monochrome, one accent. */
:root{
  --bg:#F6F6F3; --surface:#FFFFFF; --line:#E2E2DC; --line-strong:#B4B7B2;
  --ink:#1B1F23; --muted:#5A636E;
  --accent:#0F6B5C; --accent-hover:#0B5448; --accent-tint:#E4F1EE;
  --s1:#D3E5E1; --s2:#A9CEC7; --s3:#74B0A6; --s4:#3F8D81; --s5:#0F6B5C;
  --warn:#8A4B08; --warn-bg:#FBF0DD;
  --font:"Segoe UI",Roboto,Arial,sans-serif; --mono:ui-monospace,Menlo,Consolas,monospace;
}
*{box-sizing:border-box}
body{margin:0;background:var(--bg);color:var(--ink);font-family:var(--font);font-size:14px;line-height:1.5}
h1,h2,h3,p{margin:0}
button,input,select{font:inherit;color:inherit}
a{color:var(--accent)}a:hover{color:var(--accent-hover)}
:focus-visible{outline:2px solid var(--accent);outline-offset:2px}
[hidden]{display:none!important}
.sr-only{position:absolute;width:1px;height:1px;overflow:hidden;clip:rect(0 0 0 0);white-space:nowrap}
.muted{color:var(--muted)}.small{font-size:13px}.strong{font-weight:600}
.mono{font-family:var(--mono);font-size:12px}

/* ---------- Top bar ---------- */
.top{position:sticky;top:0;z-index:5;background:rgba(255,255,255,.94);backdrop-filter:blur(6px);border-bottom:1px solid var(--line)}
.top__in{max-width:1180px;margin:0 auto;padding:10px 24px;display:flex;flex-wrap:wrap;align-items:center;gap:10px 20px}
.logo{display:flex;align-items:center;gap:10px;font-weight:700;font-size:16px;color:var(--ink);text-decoration:none;letter-spacing:-.01em}
.logo:hover{color:var(--ink)}
.logo__mark{width:22px;height:22px;border-radius:6px;background:var(--accent);box-shadow:inset -7px -7px 0 rgba(255,255,255,.28)}
.finder{position:relative;flex:1 1 320px;max-width:520px}
.finder input{width:100%;min-height:42px;padding:0 14px;border:1px solid var(--line-strong);border-radius:10px;background:var(--surface)}
.finder input:focus{outline:none;border-color:var(--accent);box-shadow:0 0 0 3px rgba(15,107,92,.16)}
.finder__panel{position:absolute;left:0;right:0;top:calc(100% + 6px);background:var(--surface);border:1px solid var(--line);border-radius:12px;box-shadow:0 18px 40px rgba(27,31,35,.14);padding:6px;max-height:420px;overflow-y:auto}
.finder__group{font-size:11px;font-weight:700;letter-spacing:.08em;color:var(--muted);padding:10px 10px 4px}
.finder__item{width:100%;display:flex;align-items:center;gap:10px;min-height:44px;padding:6px 10px;border:0;border-radius:8px;background:none;text-align:left;cursor:pointer}
.finder__item:hover,.finder__item:focus-visible{background:var(--accent-tint)}
.finder__item span:last-child{min-width:0;display:flex;flex-direction:column;line-height:1.3}
.top__right{margin-left:auto;display:flex;align-items:center;gap:12px}
.mock{font-size:11px;font-weight:700;padding:3px 9px;border-radius:999px;background:var(--warn-bg);color:var(--warn)}

.btn{min-height:40px;padding:0 14px;border-radius:10px;border:1px solid var(--line-strong);background:var(--surface);font-weight:600;cursor:pointer;display:inline-flex;align-items:center;gap:8px;text-decoration:none;color:var(--ink)}
.btn:hover{border-color:var(--accent);color:var(--accent-hover)}
.btn:disabled{opacity:.55;cursor:not-allowed}
.btn--icon{width:40px;padding:0;justify-content:center}
.spin{width:14px;height:14px;border-radius:50%;border:2px solid var(--accent);border-top-color:transparent;animation:spin .8s linear infinite}
@keyframes spin{to{transform:rotate(360deg)}}

/* ---------- Page ---------- */
.page{max-width:1180px;margin:0 auto;padding:32px 24px 72px;display:flex;flex-direction:column;gap:28px}
.crumbs{display:flex;flex-wrap:wrap;align-items:center;gap:8px;font-size:13px;color:var(--muted)}
.crumbs a{color:var(--muted);text-decoration:none;font-weight:600;padding:6px 0}
.crumbs a:hover{color:var(--accent)}
.crumbs select{min-height:36px;padding:0 10px;border:1px solid var(--line-strong);border-radius:8px;background:var(--surface);font-weight:600}
.head{display:flex;flex-wrap:wrap;align-items:flex-end;justify-content:space-between;gap:16px}
.head h1{font-size:32px;line-height:1.15;font-weight:700;letter-spacing:-.02em}
.eyebrow{font-family:var(--mono);font-size:12px;color:var(--muted);margin-bottom:6px}
.lede{font-size:17px;color:var(--muted);margin-top:8px;max-width:60ch}
.lede b{color:var(--ink);font-weight:600}
.seg{display:inline-flex;gap:2px;padding:3px;border-radius:10px;background:#E9E9E4}
.seg button{min-height:36px;padding:0 13px;border:0;border-radius:8px;background:none;font-weight:600;font-size:13px;color:var(--muted);cursor:pointer}
.seg button:hover{color:var(--ink)}
.seg button[aria-pressed="true"]{background:var(--surface);color:var(--ink);box-shadow:0 1px 2px rgba(27,31,35,.14)}
.h2{font-size:13px;font-weight:700;letter-spacing:.06em;text-transform:uppercase;color:var(--muted)}
.card{background:var(--surface);border:1px solid var(--line);border-radius:14px}

/* figures: one quiet row of numbers */
.figures{display:grid;grid-template-columns:repeat(5,minmax(0,1fr));background:var(--surface);border:1px solid var(--line);border-radius:14px;overflow:hidden}
.fig{display:flex;flex-direction:column;gap:2px;padding:18px 20px;border:0;border-right:1px solid var(--line);background:none;text-align:left;cursor:pointer}
.fig:last-child{border-right:0}
.fig:hover{background:#FAFAF8}
.fig__n{font-size:30px;font-weight:700;line-height:1.1;letter-spacing:-.02em;font-variant-numeric:tabular-nums}
.fig__l{font-weight:600}
.fig__d{font-size:12px;color:var(--muted)}

/* pipeline: where features are now */
.pipe{display:grid;grid-template-columns:repeat(5,minmax(0,1fr));gap:8px}
.stage{position:relative;display:flex;flex-direction:column;gap:8px;padding:14px 16px;border:1px solid var(--line);border-radius:12px;background:var(--surface);text-align:left;cursor:pointer}
.stage:hover{border-color:var(--accent)}
.stage[aria-pressed="true"]{border-color:var(--accent);box-shadow:inset 0 0 0 1px var(--accent);background:var(--accent-tint)}
.stage__bar{height:6px;border-radius:3px;background:var(--c)}
.stage__n{font-size:24px;font-weight:700;line-height:1;font-variant-numeric:tabular-nums}
.stage__l{font-size:13px;font-weight:600}
.stage__h{font-size:12px;color:var(--muted)}

/* tabs */
.tabs{display:flex;flex-wrap:wrap;gap:4px;border-bottom:1px solid var(--line)}
.tab{min-height:44px;padding:0 14px;border:0;border-bottom:2px solid transparent;background:none;font-weight:600;color:var(--muted);cursor:pointer}
.tab:hover{color:var(--ink)}
.tab[aria-selected="true"]{color:var(--ink);border-bottom-color:var(--accent)}

/* features: master and detail */
.md{display:grid;grid-template-columns:minmax(0,380px) minmax(0,1fr);gap:20px;align-items:start}
.flist{display:flex;flex-direction:column;overflow:hidden}
.fitem{display:flex;flex-direction:column;gap:8px;padding:14px 16px;border:0;border-bottom:1px solid var(--line);border-left:3px solid transparent;background:none;text-align:left;cursor:pointer}
.fitem:last-child{border-bottom:0}
.fitem:hover{background:#FAFAF8}
.fitem[aria-current="true"]{background:var(--accent-tint);border-left-color:var(--accent)}
.fitem__top{display:flex;justify-content:space-between;gap:10px;align-items:baseline}
.fitem__title{font-weight:600;font-size:15px}
.track{display:flex;gap:3px}
.track i{flex:1;height:6px;border-radius:3px;background:#E4E4DF}
.track i.on{background:var(--accent)}
.status{font-size:12px;font-weight:700;white-space:nowrap}
.status--done{color:var(--accent)}
.status--open{color:var(--warn)}
.detail{padding:24px;display:flex;flex-direction:column;gap:20px}
.detail h2{font-size:22px;font-weight:700;letter-spacing:-.01em}
.next{padding:12px 14px;border-radius:10px;background:var(--warn-bg);color:var(--warn);font-weight:600}
.next--done{background:var(--accent-tint);color:var(--accent-hover)}
.steps{margin:0;padding:0;list-style:none;display:flex;flex-direction:column}
.step{position:relative;display:grid;grid-template-columns:28px minmax(0,1fr);gap:14px;padding-bottom:20px}
.step:last-child{padding-bottom:0}
.step::before{content:"";position:absolute;left:13px;top:28px;bottom:0;width:2px;background:var(--line)}
.step:last-child::before{display:none}
.step.is-done::before{background:var(--accent)}
.step__dot{width:28px;height:28px;border-radius:50%;background:var(--accent);color:#fff;display:flex;align-items:center;justify-content:center;font-size:12px;font-weight:700;z-index:1}
.step.is-todo .step__dot{background:var(--surface);border:2px dashed var(--line-strong);color:var(--muted)}
.step__name{font-weight:700;padding-top:3px}
.step__body{display:flex;flex-direction:column;gap:8px}
.doc{display:flex;align-items:center;gap:12px;width:100%;padding:10px 12px;border:1px solid var(--line);border-radius:10px;background:var(--surface);text-align:left;cursor:pointer}
.doc:hover{border-color:var(--accent)}
.doc__text{display:flex;flex-direction:column;min-width:0;flex:1;line-height:1.35}
.doc__open{font-size:13px;font-weight:600;color:var(--accent)}
.todo{padding:10px 12px;border:1px dashed var(--line-strong);border-radius:10px;color:var(--muted);font-size:13px}
.badge{flex:none;min-width:38px;height:26px;padding:0 7px;border-radius:7px;background:#EEEEE9;color:var(--ink);font-family:var(--mono);font-size:11px;font-weight:700;display:inline-flex;align-items:center;justify-content:center}

/* documents table */
.bar{display:flex;flex-wrap:wrap;gap:10px;align-items:center}
.chips{display:flex;flex-wrap:wrap;gap:6px}
.chip{min-height:36px;padding:0 12px;border-radius:999px;border:1px solid var(--line-strong);background:var(--surface);font-weight:600;font-size:13px;cursor:pointer}
.chip:hover{border-color:var(--accent)}
.chip[aria-pressed="true"]{background:var(--ink);border-color:var(--ink);color:#fff}
.search{flex:1 1 220px;min-height:40px;padding:0 12px;border:1px solid var(--line-strong);border-radius:10px;background:var(--surface)}
.search:focus{outline:none;border-color:var(--accent);box-shadow:0 0 0 3px rgba(15,107,92,.16)}
.table-wrap{overflow-x:auto}
table{width:100%;min-width:640px;border-collapse:collapse;font-size:13px}
th{text-align:left;padding:10px 16px;border-bottom:1px solid var(--line);color:var(--muted);font-size:12px;font-weight:700;white-space:nowrap}
td{padding:6px 16px;border-bottom:1px solid var(--line);vertical-align:middle}
tbody tr:last-child td{border-bottom:0}
tbody tr:hover td{background:#FAFAF8}
.num{text-align:right;font-variant-numeric:tabular-nums}
.th-btn{border:0;background:none;padding:6px 0;min-height:32px;font:inherit;font-weight:700;color:inherit;cursor:pointer;display:inline-flex;gap:6px;align-items:center}
.th-btn:hover{color:var(--ink)}
th[aria-sort] .th-btn{color:var(--ink)}
.row-btn{display:flex;align-items:center;gap:10px;min-height:44px;border:0;background:none;padding:0;text-align:left;cursor:pointer}
.row-btn:hover .mono{color:var(--accent);text-decoration:underline}
.empty{padding:28px;text-align:center;color:var(--muted)}

/* activity */
.chart{display:grid;grid-auto-flow:column;grid-auto-columns:minmax(0,1fr);gap:10px;align-items:end;height:180px;border-bottom:1px solid var(--line)}
.col{position:relative;display:flex;flex-direction:column;justify-content:flex-end;align-items:center;gap:4px;height:100%;border:0;background:none;padding:0;cursor:default;border-radius:6px}
.col:hover,.col:focus-visible{background:#F1F1EC}
.col__bar{width:62%;max-width:46px;border-radius:5px 5px 0 0;background:var(--accent);transform-origin:bottom;animation:grow .4s ease-out both}
@keyframes grow{from{transform:scaleY(0)}}
.col__n{font-size:12px;font-weight:700;color:var(--muted)}
.tip{display:none;position:absolute;left:50%;bottom:calc(100% - 20px);transform:translateX(-50%);background:var(--ink);color:#fff;padding:9px 12px;border-radius:9px;font-size:12px;line-height:1.5;white-space:nowrap;z-index:3;text-align:left;pointer-events:none}
.col:hover .tip,.col:focus-visible .tip{display:block}
.xaxis{display:grid;grid-auto-flow:column;grid-auto-columns:minmax(0,1fr);gap:10px;font-size:12px;color:var(--muted);text-align:center;margin-top:6px}
.feed{margin:0;padding:0;list-style:none}
.feed li{border-bottom:1px solid var(--line)}
.feed li:last-child{border-bottom:0}
.feed .doc{border:0;border-radius:0;padding:10px 20px}
.feed .doc:hover{background:#FAFAF8}
.feed__day{padding:12px 20px 4px;font-size:12px;font-weight:700;color:var(--muted);border-bottom:0!important}

/* home: repository cards */
.repos{display:grid;grid-template-columns:repeat(2,minmax(0,1fr));gap:16px}
.repo{display:flex;flex-direction:column;gap:14px;padding:20px;text-decoration:none;color:var(--ink);transition:border-color .15s,transform .15s}
.repo:hover{border-color:var(--accent);color:var(--ink);transform:translateY(-2px)}
.repo__top{display:flex;justify-content:space-between;gap:12px;align-items:flex-start}
.repo__name{font-size:19px;font-weight:700;letter-spacing:-.01em}
.repo__n{font-size:26px;font-weight:700;line-height:1;text-align:right;font-variant-numeric:tabular-nums}
.stack{display:flex;height:10px;border-radius:5px;overflow:hidden;background:#E4E4DF;gap:2px}
.stack i{background:var(--c);min-width:0}
.key{display:flex;flex-wrap:wrap;gap:4px 14px;font-size:12px;color:var(--muted)}
.key span{display:inline-flex;align-items:center;gap:6px}
.key i{width:9px;height:9px;border-radius:2px;background:var(--c)}
.spark{display:flex;align-items:flex-end;gap:3px;height:28px}
.spark i{flex:1;background:var(--s3);border-radius:2px;min-height:2px}
.b1{--c:var(--s1)}.b2{--c:var(--s2)}.b3{--c:var(--s3)}.b4{--c:var(--s4)}.b5{--c:var(--s5)}

/* drawer and toast */
.scrim{position:fixed;inset:0;background:rgba(27,31,35,.4);z-index:9}
.drawer{position:fixed;top:0;right:0;bottom:0;width:min(440px,100vw);background:var(--surface);z-index:10;box-shadow:-12px 0 40px rgba(27,31,35,.18);padding:22px;overflow-y:auto;display:flex;flex-direction:column;gap:16px}
.drawer__head{display:flex;align-items:flex-start;justify-content:space-between;gap:12px}
.drawer h2{font-size:20px;font-weight:700}
.kv{margin:0;padding:0;list-style:none;border:1px solid var(--line);border-radius:10px}
.kv li{display:flex;justify-content:space-between;gap:12px;padding:10px 14px;border-bottom:1px solid var(--line)}
.kv li:last-child{border-bottom:0}
.pathbox{padding:10px 12px;border-radius:10px;background:#F1F1EC;font-family:var(--mono);font-size:12px;overflow-wrap:anywhere}
.outline{margin:0;padding-left:20px;display:flex;flex-direction:column;gap:6px}
.nav{display:flex;align-items:center;justify-content:space-between;gap:10px}
.toast{position:fixed;left:50%;bottom:24px;transform:translateX(-50%);background:var(--ink);color:#fff;padding:12px 18px;border-radius:10px;font-weight:600;z-index:20}

@media (max-width:900px){
  .md{grid-template-columns:minmax(0,1fr)}
  .repos{grid-template-columns:minmax(0,1fr)}
  .figures{grid-template-columns:repeat(2,minmax(0,1fr))}
  .fig{border-bottom:1px solid var(--line)}
  .pipe{grid-template-columns:repeat(2,minmax(0,1fr))}
  .page{padding:22px 16px 56px}
  .top__in{padding:10px 16px}
  .head h1{font-size:26px}
}
@media (prefers-reduced-motion:reduce){*{animation:none!important;transition:none!important}}
