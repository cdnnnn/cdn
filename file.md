//src/results/demoData.js
// Dummy data for the Results page, so it can be shown without the backend.
// It answers the same paths as the real /api/datagen API (see source.js), so switching back is one flag.

// ------------------------------------------------------------------ jobs
const JOBS = [// Routing: the landing page at #/ (the first screen), and the app shell with sidebar and top bar at #/generate and #/results.
// Features are added one piece at a time; see README.md for the build order.
import { useCallback, useEffect, useState } from "react";
import { Icon } from "./ui.jsx";
import Landing from "./Landing.jsx";
import Sidebar from "./Sidebar.jsx";
import { emit, lastOf, useBus } from "./bus.js";
import GeneratePage, { STEPS } from "./generate/GeneratePage.jsx";
import "./shell.css";
import "./rich.css";
import "./theme-dark.css";
import ThemeToggle from "./ThemeToggle.jsx";
import ResultsPage from "./results/ResultsPage.jsx";

const PAGES = {
  generate: { label: "Generate", nav: "Generate Dataset", icon: "spark", title: "Generate Dataset",
    sub: "Turn your documents into grounded question-and-answer pairs for fine-tuning." },
  results: { label: "Results", nav: "Results", icon: "results", title: "Results",
    sub: "Review the generated pairs, their scores, and download the dataset." },
};
// "#how" and "#features" are anchors on the landing page, so anything that is not an app page shows the landing page.
const routeFromHash = () => { const r = (location.hash || "").replace(/^#\/?/, ""); return PAGES[r] ? r : "home"; };

export default function App() {
  const [route, setRoute] = useState(routeFromHash);
  useEffect(() => {
    const h = () => setRoute(routeFromHash());
    window.addEventListener("hashchange", h);
    return () => window.removeEventListener("hashchange", h);
  }, []);
  if (route === "home") return <Landing />;
  const page = PAGES[route];

  return (
    <div className="app">
      <Sidebar route={route} pages={PAGES} />

      <main className="main">
        <PageHeader route={route} page={page} />
        <div className="content" key={route}>
          {route === "generate" ? <GeneratePage openResults={() => { location.hash = "#/results"; }} /> : <ResultsPage />}
        </div>
      </main>
    </div>
  );
}

// Page header: breadcrumb, title and one line of context on the left; the state of the current job and actions on the right.
function PageHeader({ route, page }) {
  const [job, setJob] = useState(() => lastOf("job") || null);
  useBus("job", useCallback((j) => setJob(j), []));
  const [rjob, setRjob] = useState(() => lastOf("results-job") || null);
  useBus("results-job", useCallback((j) => setRjob(j), []));
  const onGen = route === "generate";
  const open = onGen && job?.id;
  return (
    <header className="ph">
      <div className="ph-main">
        <span className="ph-ico"><Icon name={page.icon} size={20} /></span>
        <div className="ph-t">
          <div className="ph-crumbs">
            <a href="#/">SLM Studio</a><Icon name="chevron" size={12} />
            <a href={`#/${route}`}>{page.nav}</a>
            {onGen && <><Icon name="chevron" size={12} />{open ? <span className="num">{job.id}</span> : <span>New job</span>}</>}
            {!onGen && rjob?.id && <><Icon name="chevron" size={12} /><span className="num">{rjob.name || rjob.id}</span></>}
          </div>
          <h1>{page.title}</h1>
          <p>{open ? <>Following <b>{job.name || job.id}</b>. You can leave this page; the job keeps running on the server.</> : page.sub}</p>
        </div>
      </div>
      <div className="ph-side">
        {onGen && !open && job && (
          <span className="ph-state">
            <span className="ph-state-l">Draft</span>
            <span className="ph-state-v">Step <b className="num">{(job.step ?? 0) + 1}</b> of {STEPS.length}</span>
            <span className={`ph-state-dot ${job.ready ? "ok" : ""}`} title={job.ready ? "Ready to start" : "Something still needs your input"} />
          </span>
        )}
        {open && <button type="button" className="btn" onClick={() => emit("new-job")}><Icon name="plus" size={14} />New job</button>}
        {onGen && !open && <a className="btn ghost" href="#/results"><Icon name="results" size={14} />Results</a>}
        {!onGen && <a className="btn" href="#/generate"><Icon name="plus" size={14} />New dataset</a>}
        <span className="ph-sep" aria-hidden="true" />
        <ThemeToggle />
      </div>
    </header>
  );
}

  { id: "dg-9882c2ad", name: "hr_policies", status: "done", created: "2026-11-24T16:42:00Z" },
  { id: "dg-e7b77487", name: "benefits_faq", status: "done", created: "2026-11-22T09:15:00Z" },
  { id: "dg-31c0f9e2", name: "it_security_handbook", status: "running", created: "2026-11-25T06:05:00Z" },
  { id: "dg-5d8ad240", name: "onboarding_v1", status: "failed", created: "2026-11-20T11:30:00Z" },
];

const REQUEST = (name, extra = {}) => ({
  name, mode: "chat",
  endpoints: { teacher: { base_url: "http://gpu-01:8001/v1", model: "Qwen/Qwen2.5-72B-Instruct" }, vision: { base_url: "http://gpu-01:8002/v1", model: "Qwen/Qwen2.5-VL-7B-Instruct" } },
  parse: { read_images: true, vision_analysis: true }, judge: { min_grounded: 4 }, output: { split_by: "fact", split: [90, 5, 5] }, ...extra,
});

// ------------------------------------------------------------------ Q&A pairs of hr_policies
const SRC = {
  policy: ["leave_policy_2026.pdf", "Leave policy 2026"],
  elig: ["leave_policy_2026.pdf", "Leave policy 2026 > Eligibility"],
  carry: ["leave_policy_2026.pdf", "Leave policy 2026 > Carry-over"],
  sick: ["leave_policy_2026.pdf", "Leave policy 2026 > Sick leave"],
  parent: ["leave_policy_2026.pdf", "Leave policy 2026 > Parental leave"],
  request: ["leave_requests.docx", "Leave requests > How to request leave"],
  approve: ["leave_requests.docx", "Leave requests > Approval"],
  remote: ["remote_work_guidelines.md", "Remote work > Who can work remotely"],
  equip: ["remote_work_guidelines.md", "Remote work > Equipment"],
  travel: ["travel_expenses.xlsx", "Travel expenses > Per diem"],
};

// [id, status, type, source key, question, answer, scores [g,c,cl,sc,u], extra]
const P = [
  [1, "kept", "factual", "elig", "How many days of annual leave do new employees earn each month?", "New employees earn 2 days of annual leave for every full month worked, starting from their first day. Leave earned during probation can be used once probation ends.", [5, 5, 5, 5, 5], { review: { action: "approve" } }],
  [2, "kept", "factual", "carry", "How many unused leave days can I carry over into next year?", "You can carry over up to 5 unused days into the next calendar year. Carried-over days must be used by 31 March, after which they expire.", [5, 5, 5, 5, 5], { review: { action: "approve" } }],
  [3, "kept", "explanation", "carry", "Why do carried-over leave days expire at the end of March?", "The policy sets the 31 March deadline so that leave is taken for rest during the year rather than saved up. It also keeps the company's holiday liability predictable at the start of each financial year.", [4, 5, 5, 5, 4], {}],
  [4, "kept", "how-to", "request", "How do I request annual leave in the HR portal?", "Open the HR portal, go to Time off and choose New request. Pick the leave type and the start and end dates, add an optional note for your manager, then select Submit. Your manager gets an email and the request shows as Pending until they decide.", [5, 5, 5, 5, 5], { review: { action: "approve", edited: true } }],
  [5, "kept", "from_images", "request", "What fields does the leave request form in the screenshot ask for?", "The form asks for the leave type, a start date and an end date (both with date pickers), whether each day is a full or half day, and an optional note for your manager.", [5, 5, 4, 5, 5], { source: "image", chunk_id: "c0014" }],
  [6, "kept", "factual", "sick", "Do I need a doctor's note for sick leave?", "A doctor's note is only needed when you are off sick for more than 3 consecutive working days. For shorter absences, tell your manager before 10:00 on the first day.", [5, 5, 5, 5, 5], {}],
  [7, "kept", "comparison", "parent", "How does maternity leave differ from paternity leave?", "Maternity leave is 26 weeks, of which the first 16 weeks are on full pay. Paternity leave is 4 weeks on full pay and must be taken within 6 months of the birth or adoption.", [5, 5, 5, 5, 5], {}],
  [8, "kept", "how-to", "approve", "What happens after my manager approves a leave request?", "Once approved, the days are taken off your balance straight away and the leave appears in the team calendar. You get a confirmation email, and you can still cancel the request in the portal up to the day before it starts.", [4, 5, 5, 5, 4], {}],
  [9, "kept", "factual", "remote", "Who is allowed to work remotely?", "Employees who have passed probation and whose role is marked remote-eligible can work remotely up to 3 days a week, with their manager's agreement.", [5, 5, 5, 4, 5], { review: { action: "reject" } }],
  [10, "kept", "from_images", "equip", "What equipment is shown in the home office setup diagram?", "The diagram shows a laptop on a stand, an external monitor at eye level, a separate keyboard and mouse, a headset, and an adjustable chair. Labels note that the company provides all of these except the chair.", [4, 5, 4, 5, 4], { source: "image", chunk_id: "c0031" }],
  [11, "kept", "factual", "equip", "Can I claim money back for a home office chair?", "Yes. You can claim up to €250 once every three years for an ergonomic chair, using the Equipment claim form with the receipt attached.", [5, 4, 5, 5, 5], {}],
  [12, "kept", "factual", "travel", "What is the daily meal allowance on a business trip?", "The per diem for meals is €45 per full travel day and €22 for the day you leave and the day you come back. Meals paid by a client or at a conference are deducted.", [5, 5, 5, 5, 5], {}],
  [13, "kept", "factual", "elig", "How many leave days do new employees get each month?", "New employees earn 2 days of annual leave for every full month worked, starting from their first day.", [5, 5, 5, 5, 4], { variant_of: "q00001" }],
  [14, "kept", "factual", "elig", "new hire leave per month??", "New employees earn 2 days of annual leave for every full month worked, starting from their first day.", [5, 5, 4, 5, 4], { variant_of: "q00001" }],
  [15, "kept", "factual", "carry", "Can leftover vacation days roll over to next year?", "Yes, up to 5 unused days roll over into the next calendar year, and they must be used by 31 March.", [5, 5, 5, 5, 5], { variant_of: "q00002" }],
  [16, "kept", "factual", "sick", "When is a medical certificate required for being off sick?", "You only need a doctor's note when you are off sick for more than 3 consecutive working days.", [5, 5, 5, 5, 4], { variant_of: "q00006" }],
  [17, "kept", "decline", null, "What's a good recipe for banana bread?", "I can only help with questions about the company's HR policies, such as leave, remote work and expenses. For anything else, please use another resource.", [5, 5, 5, 5, 4], { source: "decline", decline_kind: "Off topic" }],
  [18, "kept", "decline", null, "What is my colleague Sarah's salary?", "I can't share personal information about other employees. For questions about your own pay, please contact HR directly.", [5, 5, 5, 5, 5], { source: "decline", decline_kind: "Personal data" }],
  [19, "kept", "general", null, "What is the difference between gross and net salary?", "Gross salary is the full amount before any deductions. Net salary is what you actually receive after income tax, social security and other deductions are taken off.", [5, 5, 5, 5, 4], { source: "general", general_topic: "workplace" }],
  [20, "kept", "general", null, "How should I write a polite out-of-office message?", "Say when you are away and when you will be back, who to contact for urgent matters, and thank the sender. Keep it short, for example: I'm out of the office until 14 March with limited access to email. For urgent questions please contact Maria Lopez.", [4, 5, 5, 5, 5], { source: "general", general_topic: "workplace" }],
  [21, "kept", "explanation", "remote", "Why does remote work require passing probation first?", "Probation is when new employees learn the team's ways of working and build relationships, which is easier in person. Passing it also shows the manager can trust the employee to work independently.", [4, 4, 5, 5, 4], {}],
  [22, "kept", "how-to", "travel", "How do I claim travel expenses after a trip?", "Within 30 days of returning, open Expenses in the HR portal, choose New claim, add each cost with its receipt photo, and submit it to your manager. Approved claims are paid with the next salary.", [5, 5, 5, 5, 5], {}],
  [30, "dropped", "factual", "parent", "Is paternity leave paid at 80%?", "Paternity leave is paid at 80% of your salary for the first two weeks.", [2, 1, 5, 5, 4], { drop_reason: "low grounded score", why_grounded: "The context says paternity leave is on full pay; the 80% figure is not in the source.", why_correct: "Contradicts the policy, which states 4 weeks on full pay." }],
  [31, "dropped", "explanation", "policy", "What does the policy say about this?", "The policy covers all permanent and fixed-term employees.", [4, 4, 2, 2, 2], { drop_reason: "low clarity", why_clear: "The question is vague: 'this' does not refer to anything.", why_self_contained: "Needs the document open to make sense." }],
  [32, "dropped", "how-to", "approve", "How do I escalate a rejected leave request?", "Email the CEO directly and they will overrule your manager.", [1, 1, 4, 5, 2], { drop_reason: "low grounded score", why_grounded: "Nothing in the context mentions escalation to the CEO." }],
  [33, "dropped", "factual", "travel", "Can I fly business class?", "", null, { drop_reason: "no answer" }],
  [40, "duplicate", "factual", "elig", "How much annual leave do new starters earn per month?", "New starters earn 2 days of annual leave for each full month worked.", [5, 5, 5, 5, 5], { duplicate_of: "q00001" }],
  [41, "duplicate", "factual", "carry", "How many vacation days carry over?", "Up to 5 days carry over into the next year.", [5, 5, 4, 5, 4], { duplicate_of: "q00002" }],
  [42, "duplicate", "how-to", "request", "Where do I submit a leave request?", "Leave requests are submitted in the HR portal under Time off.", [5, 5, 5, 5, 4], { duplicate_of: "q00004" }],
  [43, "duplicate", "factual", "sick", "Is a sick note needed for a 2-day absence?", "No, a doctor's note is only needed after more than 3 consecutive working days.", [5, 5, 5, 5, 5], { duplicate_of: "q00006" }],
  [44, "duplicate", "factual", "equip", "Will the company pay for my office chair?", "You can claim up to €250 every three years for an ergonomic chair.", [5, 5, 5, 5, 5], { duplicate_of: "q00011" }],
];
const SPLITS = { 7: "val", 22: "test" };

const ROWS = P.map(([n, status, type, src, question, answer, sc, extra]) => {
  const [doc, path] = src ? SRC[src] : [null, null];
  const s = sc ? { score_grounded: sc[0], score_correct: sc[1], score_clear: sc[2], score_self_contained: sc[3], score_useful: sc[4] } : {};
  const chunk = `c${String(n * 3 + 1).padStart(4, "0")}`;
  return {
    q_id: `q${String(n).padStart(5, "0")}`, status, type, question, answer, doc_name: doc, heading_path: path,
    split: status === "kept" ? SPLITS[n] || "train" : null, source: "document",
    chunk_id: src ? chunk : null, related_ids: src && n % 3 === 0 ? JSON.stringify([`c${String(n * 3 + 2).padStart(4, "0")}`]) : "[]", ...s, ...extra,
  };
});

// ------------------------------------------------------------------ reports
const HR_REPORT = {
  kept: 22, avg_grounded: 4.8, from_images: 2, split_by: "fact",
  funnel: { questions_generated: 31, after_question_dedupe: 26, answered: 25, passed_judge: 22, kept: 22 },
  splits: { train: 20, val: 1, test: 1 },
  mix: { documents: 12, paraphrases: 4, decline: 2, general: 2 },
  image_funnel: { found: 6, read: 4, image_chunks: 3, questions: 4, answered: 3, kept: 2, vision_analysis: true, read_images: true, skip_reasons: { "too small": 1, "decorative": 1 } },
  drop_reasons: { "duplicate question": 5, "low grounded score": 2, "low clarity": 1, "no answer": 1 },
  scores_of_kept: {
    grounded: { 4: 5, 5: 17 }, correct: { 4: 2, 5: 20 }, clear: { 4: 3, 5: 19 }, self_contained: { 4: 1, 5: 21 }, useful: { 4: 8, 5: 14 },
  },
  types: { factual: 10, "how-to": 3, explanation: 2, from_images: 2, comparison: 1, decline: 2, general: 2 },
  split_coverage: {
    "Leave policy 2026": { train: 9, val: 1 }, "Leave requests": { train: 3, test: 1 }, "Remote work": { train: 4 }, "Travel expenses": { train: 2 }, "Parental leave": { val: 1 },
  },
  warnings: [
    "General knowledge is on, and the decline rule 'Off topic' declines questions unrelated to the documents: the model is taught to both answer and refuse general questions.",
    "The section 'Parental leave' only appears in validation, so the model never sees it during training.",
  ],
};

const BENEFITS_REPORT = {
  kept: 148, avg_grounded: 4.6, from_images: 11, split_by: "section",
  funnel: { questions_generated: 212, after_question_dedupe: 181, answered: 176, passed_judge: 152, kept: 148 },
  splits: { train: 132, val: 8, test: 8 },
  mix: { documents: 96, paraphrases: 32, decline: 8, general: 12 },
  image_funnel: { found: 24, read: 19, image_chunks: 14, questions: 18, answered: 16, kept: 11, vision_analysis: true, read_images: true },
  drop_reasons: { "duplicate question": 31, "low grounded score": 18, "low clarity": 6, "no answer": 5, "too long": 4 },
  scores_of_kept: { grounded: { 3: 4, 4: 48, 5: 96 }, correct: { 4: 22, 5: 126 }, clear: { 3: 6, 4: 40, 5: 102 }, self_contained: { 4: 18, 5: 130 }, useful: { 3: 9, 4: 55, 5: 84 } },
  types: { factual: 71, "how-to": 24, explanation: 19, from_images: 11, comparison: 12, decline: 8, general: 3 },
  split_coverage: { "Health insurance": { train: 41, val: 3, test: 2 }, "Pension plan": { train: 33, val: 2, test: 3 }, "Gym membership": { train: 18, val: 1, test: 1 },
    "Bike to work": { train: 22, val: 1, test: 1 }, "Meal vouchers": { train: 18, val: 1, test: 1 } },
  warnings: [],
};

// ------------------------------------------------------------------ source data
const img = (label) => `data:image/svg+xml;utf8,${encodeURIComponent(`<svg xmlns="http://www.w3.org/2000/svg" width="320" height="220" viewBox="0 0 320 220"><rect width="320" height="220" rx="12" fill="#f8fafc"/><rect x="20" y="20" width="280" height="26" rx="6" fill="#fde2e4"/><text x="32" y="38" font-family="Segoe UI,Arial" font-size="13" fill="#9f1239">${label}</text><rect x="20" y="62" width="130" height="14" rx="4" fill="#e2e8f0"/><rect x="20" y="86" width="280" height="22" rx="6" fill="#fff" stroke="#cbd5e1"/><rect x="20" y="118" width="280" height="22" rx="6" fill="#fff" stroke="#cbd5e1"/><rect x="20" y="150" width="180" height="22" rx="6" fill="#fff" stroke="#cbd5e1"/><rect x="220" y="178" width="80" height="26" rx="7" fill="#f2545b"/></svg>`)}`;

const DOCS = [
  { doc_id: "d1", name: "leave_policy_2026.pdf", ext: ".pdf", status: "ok", pages: 12, chunks: 14, images: [
    { pic_id: "p1", figure: 1, page: 3, status: "read", source: "vision", type: "table", file: "leave-table", caption: "Leave days per year of service", visible_text: "Years of service | Annual leave\n0–2 | 24 days\n3–5 | 26 days\n6+ | 28 days", description: "A table that maps years of service to the number of annual leave days.", heading_path: "Leave policy 2026 > Eligibility", width: 920, height: 410 },
    { pic_id: "p2", figure: 2, page: 1, status: "skipped", reason: "decorative", source: "vision", file: "", heading_path: "Leave policy 2026", width: 160, height: 60 }],
    sections: [{ heading_path: "Leave policy 2026", chunks: 2 }, { heading_path: "Leave policy 2026 > Eligibility", chunks: 3 }, { heading_path: "Leave policy 2026 > Carry-over", chunks: 2 },
      { heading_path: "Leave policy 2026 > Sick leave", chunks: 3 }, { heading_path: "Leave policy 2026 > Parental leave", chunks: 4 }] },
  { doc_id: "d2", name: "leave_requests.docx", ext: ".docx", status: "ok", pages: 4, chunks: 6, images: [
    { pic_id: "p3", figure: 1, page: 2, status: "read", source: "vision", type: "screenshot", file: "leave-form", caption: "The leave request form in the HR portal", visible_text: "New request · Leave type · Start date · End date · Full day / Half day · Note to manager · Submit", description: "A web form with a leave type dropdown, two date pickers, a full or half day toggle, a note field and a Submit button.", heading_path: "Leave requests > How to request leave", width: 1280, height: 860 },
    { pic_id: "p4", figure: 2, page: 3, status: "read", source: "ocr", type: "screenshot", file: "approval", caption: "Approval email", visible_text: "Your leave request for 12–16 May has been approved by Jan de Vries.", heading_path: "Leave requests > Approval", width: 900, height: 300 }],
    sections: [{ heading_path: "Leave requests", chunks: 1 }, { heading_path: "Leave requests > How to request leave", chunks: 3 }, { heading_path: "Leave requests > Approval", chunks: 2 }] },
  { doc_id: "d3", name: "remote_work_guidelines.md", ext: ".md", status: "ok", pages: 0, chunks: 5, images: [
    { pic_id: "p5", figure: 1, status: "read", source: "vision", type: "diagram", file: "home-office", caption: "Recommended home office setup", description: "A labelled drawing of a desk with a laptop on a stand, an external monitor, keyboard, mouse, headset and an adjustable chair.", heading_path: "Remote work > Equipment", width: 1024, height: 680 },
    { pic_id: "p6", figure: 2, status: "skipped", reason: "too small", source: "vision", file: "", heading_path: "Remote work", width: 48, height: 48 }],
    sections: [{ heading_path: "Remote work", chunks: 1 }, { heading_path: "Remote work > Who can work remotely", chunks: 2 }, { heading_path: "Remote work > Equipment", chunks: 2 }] },
  { doc_id: "d4", name: "travel_expenses.xlsx", ext: ".xlsx", status: "ok", pages: 0, chunks: 4, images: [],
    sections: [{ heading_path: "Travel expenses", chunks: 1 }, { heading_path: "Travel expenses > Per diem", chunks: 2 }, { heading_path: "Travel expenses > Claims", chunks: 1 }] },
  { doc_id: "d5", name: "scanned_contract_2019.pdf", ext: ".pdf", status: "skipped", reason: "no text layer (scanned) and OCR is off", chunks: 0, images: [], sections: [] },
];
const IMAGES = { "leave-table": img("Leave days per year of service"), "leave-form": img("New leave request"), approval: img("Approval email"), "home-office": img("Home office setup") };

const MD = {
  d1: "# Leave policy 2026\n\nThis policy applies to all permanent and fixed-term employees, including those on probation.\n\n## Eligibility\n\nEmployees earn 2 days of annual leave for every full month worked, starting from their first day.\n\n<!-- image: figure 1, page 3 -->\nTable: years of service and annual leave days.\n\n## Carry-over\n\nUp to 5 unused days can be carried over into the next calendar year and must be used by 31 March.\n\n## Sick leave\n\nTell your manager before 10:00 on the first day. A doctor's note is needed after more than 3 consecutive working days.\n\n## Parental leave\n\nMaternity leave is 26 weeks, the first 16 on full pay. Paternity leave is 4 weeks on full pay, taken within 6 months.\n",
  d2: "# Leave requests\n\n## How to request leave\n\nOpen the HR portal, go to Time off and choose New request.\n\n<!-- image: figure 1, page 2 -->\nScreenshot: the leave request form.\n\n## Approval\n\nYour manager gets an email and the request shows as Pending until they decide.\n",
  d3: "# Remote work\n\n## Who can work remotely\n\nEmployees who passed probation in a remote-eligible role can work remotely up to 3 days a week.\n\n## Equipment\n\nThe company provides a laptop, monitor, keyboard, mouse and headset. You can claim up to €250 every three years for a chair.\n",
  d4: "# Travel expenses\n\n## Per diem\n\n| Day | Meals |\n|---|---|\n| Full travel day | €45 |\n| Departure / return day | €22 |\n\n## Claims\n\nSubmit claims within 30 days in Expenses in the HR portal.\n",
};

const KB = {
  overview: "Four HR documents for all employees: the 2026 leave policy (accrual, carry-over, sick and parental leave), how to request and approve leave in the HR portal, remote work rules and equipment, and travel expense allowances.",
  documents: [
    { doc_id: "d1", name: "leave_policy_2026.pdf", summary: "Who is covered, how annual leave accrues and carries over, and the rules for sick and parental leave.", topics: ["annual leave", "carry-over", "sick leave", "parental leave"] },
    { doc_id: "d2", name: "leave_requests.docx", summary: "Step-by-step guide to requesting leave in the HR portal and what happens after approval.", topics: ["HR portal", "approval"] },
    { doc_id: "d3", name: "remote_work_guidelines.md", summary: "Who can work remotely, how often, and what equipment the company provides.", topics: ["remote work", "equipment"] },
    { doc_id: "d4", name: "travel_expenses.xlsx", summary: "Daily meal allowances and how to claim travel costs.", topics: ["per diem", "expenses"] },
  ],
  topics: [
    { document: "leave_policy_2026.pdf", top_heading: "Leave policy 2026", chunks: 14, kept_pairs: 10 },
    { document: "leave_requests.docx", top_heading: "Leave requests", chunks: 6, kept_pairs: 4 },
    { document: "remote_work_guidelines.md", top_heading: "Remote work", chunks: 5, kept_pairs: 4 },
    { document: "travel_expenses.xlsx", top_heading: "Travel expenses", chunks: 4, kept_pairs: 2 },
  ],
  glossary: [
    { term: "Probation", definition: "The first 6 months of employment, during which notice periods are shorter.", doc_ids: "d1,d3" },
    { term: "Carry-over", definition: "Unused leave days moved into the next calendar year, up to 5.", doc_ids: "d1" },
    { term: "HR portal", definition: "The self-service site for leave requests, expenses and personal details.", doc_ids: "d2,d4" },
    { term: "Per diem", definition: "A fixed daily allowance for meals while travelling for work.", doc_ids: "d4" },
    { term: "Remote-eligible", definition: "A role that the company has marked as possible to do from home.", doc_ids: "d3" },
  ],
};

// ------------------------------------------------------------------ the fake API
const reviews = {}; // q_id -> review, kept for this browser session
const wait = (v, ms = 220) => new Promise((r) => setTimeout(() => r(structuredClone(v)), ms));
const fail = (status, message) => new Promise((_, rej) => setTimeout(() => { const e = new Error(message); e.status = status; rej(e); }, 200));

function rowsFor(jobId, qs) {
  const base = jobId === "dg-9882c2ad" ? ROWS : ROWS.map((r) => ({ ...r, doc_name: r.doc_name && r.doc_name.replace(/leave_policy_2026|leave_requests|remote_work_guidelines|travel_expenses/, "benefits_guide") }));
  const all = base.map((r) => ({ ...r, review: reviews[`${jobId}/${r.q_id}`] || r.review || null }));
  const isImg = (r) => r.status === "kept" && r.source === "image";
  const by = { kept: (r) => r.status === "kept", dropped: (r) => r.status === "dropped", duplicate: (r) => r.status === "duplicate", image: isImg,
    review: (r) => r.status === "kept" && !r.review?.action, all: () => true };
  const counts = Object.fromEntries(Object.entries(by).map(([k, f]) => [k, all.filter(f).length]));
  const q = (qs.get("q") || "").toLowerCase(), type = qs.get("type") || "";
  const list = all.filter(by[qs.get("status") || "kept"] || by.kept).filter((r) => !type || r.type === type)
    .filter((r) => !q || `${r.question} ${r.answer}`.toLowerCase().includes(q));
  const offset = Number(qs.get("offset") || 0), limit = Number(qs.get("limit") || 15);
  return { rows: list.slice(offset, offset + limit), total: list.length, counts };
}

export function demoApi(path) {
  const [p, query = ""] = path.split("?");
  const qs = new URLSearchParams(query);
  if (p === "/jobs") return wait(JOBS);
  const m = p.match(/^\/jobs\/([^/]+)(\/.*)?$/);
  if (!m) return fail(404, "Not found");
  const [, id, rest = ""] = m;
  const job = JOBS.find((j) => j.id === id);
  if (!job) return fail(404, "Job not found");
  if (!rest) return wait({ job_id: id, name: job.name, status: job.status, request: REQUEST(job.name), error: job.status === "failed" ? "The teacher model at gpu-01:8001 stopped answering (connection refused) during the Answers step." : null });
  if (job.status === "running") return fail(409, "The job is still running");
  if (job.status === "failed") return fail(410, "This job failed before it produced a dataset.");
  const rep = id === "dg-9882c2ad" ? HR_REPORT : BENEFITS_REPORT;
  if (rest === "/report") return wait(rep);
  if (rest === "/rows") return wait(rowsFor(id, qs));
  if (rest === "/documents") return wait(DOCS);
  const md = rest.match(/^\/documents\/([^/]+)\/markdown$/);
  if (md) return wait(MD[decodeURIComponent(md[1])] || "", 150);
  if (rest === "/kb") return wait(KB);
  return fail(404, "Not found");
}

export function demoPost(path, body = {}) {
  const acc = path.match(/^\/jobs\/([^/]+)\/accept$/);
  if (acc) { const rep = acc[1] === "dg-9882c2ad" ? HR_REPORT : BENEFITS_REPORT; return wait({ rows: rep.kept, path: `/data/datasets/${JOBS.find((j) => j.id === acc[1])?.name}/all.jsonl` }, 600); }
  const rv = path.match(/^\/jobs\/([^/]+)\/rows\/([^/]+)$/);
  if (rv) {
    const key = `${rv[1]}/${decodeURIComponent(rv[2])}`;
    if (body.action === "reset") delete reviews[key];
    else if (body.action === "edit") reviews[key] = { ...(reviews[key] || {}), action: "approve", edited: true, question: body.question, answer: body.answer };
    else reviews[key] = { ...(reviews[key] || {}), action: body.action };
    return wait({ review: reviews[key] || {} }, 180);
  }
  return fail(404, "Not found");
}

// downloads: build the file in the browser from the dummy rows
export function demoFileUrl(jobId, name) {
  const toChat = (r) => JSON.stringify({ messages: [{ role: "user", content: r.question }, { role: "assistant", content: r.answer }] });
  const kept = ROWS.filter((r) => r.status === "kept" && reviews[`${jobId}/${r.q_id}`]?.action !== "reject");
  const pick = { "train.jsonl": (r) => r.split === "train", "val.jsonl": (r) => r.split === "val", "test.jsonl": (r) => r.split === "test", "all.jsonl": () => true }[name];
  const text = pick ? kept.filter(pick).map(toChat).join("\n") : JSON.stringify(jobId === "dg-9882c2ad" ? HR_REPORT : BENEFITS_REPORT, null, 2);
  return `data:${name.endsWith(".json") ? "application/json" : "application/x-ndjson"};charset=utf-8,${encodeURIComponent(text)}`;
}
export const demoImageUrl = (_jobId, file) => IMAGES[file] || "";













//src/results/source.js
// Where the Results page gets its data from.
// DEMO = true shows the dummy data in demoData.js; set it to false to use the real /api/datagen backend.
import { api as realApi, fileUrl as realFileUrl, imageUrl as realImageUrl, post as realPost } from "../api.js";
import { demoApi, demoFileUrl, demoImageUrl, demoPost } from "./demoData.js";

export const DEMO = true;

export const api = DEMO ? demoApi : realApi;
export const post = DEMO ? demoPost : realPost;
export const fileUrl = DEMO ? demoFileUrl : realFileUrl;
export const imageUrl = DEMO ? demoImageUrl : realImageUrl;













//ResultsPage.jsx
// Results page as a guided, step-by-step flow (like Generate): pick a dataset, see the overview,
// review the pairs, look at the insights if you like, then accept and download.
import { useCallback, useEffect, useLayoutEffect, useRef, useState } from "react";
import { api, DEMO, fileUrl, post } from "./source.js";
import { emit } from "../bus.js";
import { fmtN } from "../model.js";
import { Empty, Icon, useBusy, useToast } from "../ui.jsx";
import { Bars, Funnel, ImagePath, ScoreStacks, SplitTable, StackBar, TypeDonut, useCount } from "./Insights.jsx";
import Pair from "./Pair.jsx";
import SourceDrawer from "./SourceDrawer.jsx";
import "./results.css";

const STEPS = [
  { key: "pick", title: "Choose dataset", icon: "database", lead: "Pick the generated dataset you want to look at. Only finished jobs have results." },
  { key: "overview", title: "Overview", icon: "chart", lead: "A quick look at what was generated, and anything to check before training." },
  { key: "review", title: "Review pairs", icon: "list", lead: "Read the question-and-answer pairs. Approve the good ones, reject the bad ones, or edit them. You can skip this step." },
  { key: "insights", title: "Insights", icon: "layers", lead: "Optional. Charts that show where the questions came from, how they scored, and why some were removed." },
  { key: "finish", title: "Accept & download", icon: "download", lead: "Write the final dataset with your reviews applied, and download the files for training." },
];
const SHOW = [["kept", "Kept"], ["review", "To review"], ["dropped", "Dropped"], ["all", "All"]];
const MORE = [["duplicate", "Duplicates"], ["image", "From images"]];
const TYPES = [["", "All types"], ["factual", "Factual"], ["explanation", "Explanation"], ["how-to", "How-to"], ["from_images", "From images"],
  ["comparison", "Comparison"], ["decline", "Decline"], ["general", "General knowledge"]];
const LIMIT = 10;
const KEY = "slm-results-job";
const STATUS = { done: ["Finished", "ok"], running: ["Still running", "run"], queued: ["Queued", "run"], failed: ["Failed", "bad"], cancelled: ["Cancelled", "muted"] };
const firstJob = () => new URL(location.href).searchParams.get("job") || localStorage.getItem(KEY);
const when = (iso) => { try { return new Date(iso).toLocaleString(undefined, { day: "numeric", month: "short", hour: "2-digit", minute: "2-digit" }); } catch { return ""; } };

export default function ResultsPage() {
  const toast = useToast();
  const [jobs, setJobs] = useState(null);
  const [jobId, setJobId] = useState(firstJob);
  const [job, setJob] = useState(null);
  const [report, setReport] = useState(null);
  const [problem, setProblem] = useState(null); // { kind: "running" | "failed", text }
  const [step, setStep] = useState(0);
  const [counts, setCounts] = useState(null);
  const [accepted, setAccepted] = useState(null);
  const [drawer, setDrawer] = useState(null);

  // direction of travel, so each step slides in from the right side
  const prev = useRef(0);
  const dir = step >= prev.current ? "fwd" : "back";
  useEffect(() => { prev.current = step; }, [step]);

  useEffect(() => {
    api("/jobs").then((list) => {
      setJobs(list);
      setJobId((id) => (id && list.some((j) => j.id === id) ? id : (list.find((j) => j.status === "done") || list[0])?.id || null));
    }).catch((e) => { setJobs([]); toast(`API not reachable: ${e.message}`, "error"); });
  }, [toast]);

  useEffect(() => {
    setJob(null); setReport(null); setProblem(null); setCounts(null); setAccepted(null);
    if (!jobId) return;
    localStorage.setItem(KEY, jobId);
    api(`/jobs/${jobId}`).then(setJob).catch(() => {});
    api(`/jobs/${jobId}/report`).then(setReport)
      .catch((e) => setProblem(e.status === 409 ? { kind: "running", text: "This job is still running. Its results appear here as soon as it finishes." } : { kind: "failed", text: e.message }));
  }, [jobId]);
  // counts of the review tabs, loaded once so the steps can show progress
  useEffect(() => {
    if (!report) return;
    api(`/jobs/${jobId}/rows?status=kept&limit=1`).then((x) => setCounts((c) => c || x.counts)).catch(() => {});
  }, [report, jobId]);
  useEffect(() => { emit("results-job", jobId ? { id: jobId, name: job?.name || job?.request?.name } : null); }, [jobId, job]);

  // the soft card behind the active step, measured so it always sits under the right one
  const stepperRef = useRef(null);
  const [ind, setInd] = useState(null);
  useLayoutEffect(() => {
    const place = () => {
      const el = stepperRef.current?.querySelector(".stp.on");
      const n = el ? { left: el.offsetLeft, top: el.offsetTop, width: el.offsetWidth, height: el.offsetHeight } : null;
      setInd((c) => (c && n && c.left === n.left && c.top === n.top && c.width === n.width && c.height === n.height) || c === n ? c : n);
    };
    place();
    window.addEventListener("resize", place);
    return () => window.removeEventListener("resize", place);
  });

  const openGenerate = (id) => { const u = new URL(location.href); id ? u.searchParams.set("job", id) : u.searchParams.delete("job"); u.hash = "#/generate"; location.href = u.toString(); };
  const onCounts = useCallback((c) => setCounts(c), []);
  const ready = !!report;
  const cur = STEPS[step];
  const reviewed = counts ? (counts.kept ?? 0) - (counts.review ?? 0) : 0;
  const notes = {
    pick: jobId ? { ok: ready, note: job?.name || jobs?.find((j) => j.id === jobId)?.name || jobId } : { note: "Pick a dataset" },
    overview: ready ? { ok: step > 1, note: `${fmtN(report.kept)} rows` } : { note: "—" },
    review: ready ? { ok: step > 2, note: counts ? `${fmtN(reviewed)} of ${fmtN(counts.kept)} reviewed` : "Optional" } : { note: "—" },
    insights: ready ? { ok: step > 3, note: "Optional" } : { note: "—" },
    finish: ready ? { ok: !!accepted, note: accepted ? "Accepted" : "Not yet" } : { note: "—" },
  };

  if (jobs && jobs.length === 0) {
    return <section className="card rs-empty"><Empty icon="results" title="No datasets yet">Generate your first dataset and its results show up here.
      <a className="btn primary" href="#/generate"><Icon name="plus" size={14} />New dataset</a></Empty></section>;
  }

  return (
    <div className="gen rs">
      <nav className="stepper" aria-label="Results steps" ref={stepperRef} style={{ "--p": step / (STEPS.length - 1) }}>
        <span className="stp-progress" aria-hidden="true"><i /></span>
        {ind && <span className="stp-ind" aria-hidden="true" style={{ transform: `translate(${ind.left}px, ${ind.top}px)`, width: ind.width, height: ind.height }} />}
        {STEPS.map((s, i) => {
          const st = notes[s.key];
          return (
            <button type="button" key={s.key} className={`stp${i === step ? " on" : ""}${st.ok ? " ok" : ""}`} disabled={i > 0 && !ready} onClick={() => setStep(i)}>
              <span className="stp-n">{st.ok && i !== step ? <Icon name="check" size={14} stroke={2.6} className="pop" /> : <span className="pop">{i + 1}</span>}</span>
              <span className="stp-t"><b>{s.title}</b><small>{st.note}</small></span>
            </button>
          );
        })}
      </nav>

      <section className="card gen-main rs-main">
        <header className={`gen-hd anim-${dir}`} key={`hd${step}`}>
          <span className="gen-hd-ico"><Icon name={cur.icon} size={20} /></span>
          <div className="gen-hd-t">
            <div className="gen-hd-meta">
              <span className="step-n">Step {step + 1} of {STEPS.length}</span>
              <span className="step-dots" aria-hidden="true">{STEPS.map((s2, i) => <i key={s2.key} className={i < step ? "done" : i === step ? "on" : ""} />)}</span>
              {DEMO && <span className="rs-demo" title="Dummy data: set DEMO to false in src/results/source.js to use the real API">Demo data</span>}
            </div>
            <h2>{cur.title}</h2><p>{cur.lead}</p>
          </div>
        </header>

        <div className={`gen-bd anim-${dir}`} key={`bd${step}`}>
          {cur.key === "pick" && <PickStep jobs={jobs} jobId={jobId} onPick={setJobId} problem={problem} job={job} openGenerate={openGenerate} />}
          {cur.key === "overview" && ready && <OverviewStep r={report} onSource={() => setDrawer("documents")} />}
          {cur.key === "review" && ready && <Review jobId={jobId} job={job} onCounts={onCounts} />}
          {cur.key === "insights" && ready && <Insights r={report} />}
          {cur.key === "finish" && ready && <FinishStep jobId={jobId} job={job} r={report} counts={counts} accepted={accepted} setAccepted={setAccepted} goReview={() => setStep(2)} />}
        </div>

        <footer className="gen-ft">
          <button type="button" className="btn ghost" disabled={step === 0} onClick={() => setStep(step - 1)}><Icon name="back" size={15} />Back</button>
          <div className="grow" />
          {step === 2 && <button type="button" className="btn ghost" onClick={() => setStep(4)}>Skip to accept</button>}
          {step < STEPS.length - 1
            ? <button type="button" className="btn dark" disabled={!ready} onClick={() => setStep(step + 1)}>Next: {STEPS[step + 1].title}<Icon name="arrow" size={15} /></button>
            : <a className="btn ghost" href="#/generate"><Icon name="plus" size={14} />Generate another dataset</a>}
        </footer>
      </section>
      <SourceDrawer jobId={jobId} job={job} tab={drawer} setTab={setDrawer} onClose={() => setDrawer(null)} />
    </div>
  );
}

// ------------------------------------------------------------------ step 1: choose a dataset
function PickStep({ jobs, jobId, onPick, problem, job, openGenerate }) {
  if (!jobs) return <div className="rs-skel rows"><i /><i /></div>;
  return (
    <div className="rs-pickstep">
      <div className="rs-jobs" role="radiogroup" aria-label="Datasets">
        {jobs.map((j) => {
          const [l, tone] = STATUS[j.status] || [j.status, "muted"];
          return (
            <button type="button" role="radio" aria-checked={j.id === jobId} key={j.id} className={`rs-job${j.id === jobId ? " on" : ""}`} onClick={() => onPick(j.id)}>
              <span className="rs-radio" aria-hidden="true" />
              <span className="rs-job-t"><b>{j.name || j.id}</b><small className="mono">{j.id}{j.created ? ` · ${when(j.created)}` : ""}</small></span>
              <span className={`rs-status ${tone}`}><span className={`rs-dot ${tone}`} />{l}</span>
            </button>
          );
        })}
      </div>
      {problem && (
        <div className={`rs-callout ${problem.kind}`}>
          <Icon name={problem.kind === "running" ? "clock" : "alert"} size={16} />
          <div><b>{problem.kind === "running" ? "Not finished yet" : "This job has no results"}</b><p>{job?.error || problem.text}</p></div>
          <button type="button" className="btn small" onClick={() => openGenerate(jobId)}>{problem.kind === "running" ? "Follow the progress" : "Open it on Generate"}</button>
        </div>
      )}
    </div>
  );
}

// ------------------------------------------------------------------ step 2: the key numbers and what to check
function OverviewStep({ r, onSource }) {
  const f = r.funnel || {}, sp = r.splits || {};
  const removed = Object.values(r.drop_reasons || {}).reduce((a, b) => a + b, 0);
  const kept = useCount(r.kept);
  const warns = r.warnings || [];
  return (
    <>
      <section className="rs-sum">
        <div><span>Rows in the dataset</span><b className="num">{fmtN(Math.round(kept))}</b><small>from {fmtN(f.questions_generated)} questions</small></div>
        <div><span>Average grounded score</span><b className="num">{r.avg_grounded ?? "—"}<em>/ 5</em></b><small>how well answers match the source</small></div>
        <div><span>Train · validation · test</span><b className="num">{fmtN(sp.train || 0)}<em> · {fmtN(sp.val || 0)} · {fmtN(sp.test || 0)}</em></b><small>split by {r.split_by || "section"}</small></div>
        <div><span>Removed</span><b className="num">{fmtN(removed)}</b><small>duplicates and low scores</small></div>
      </section>
      {warns.length > 0 ? (
        <div className="rs-callout warn"><Icon name="alert" size={16} />
          <div><b>Check before training</b><ul>{warns.map((w) => <li key={w}>{w}</li>)}</ul></div></div>
      ) : (
        <div className="rs-callout ok"><Icon name="check" size={16} /><div><b>Nothing to worry about</b><p>No warnings for this dataset.</p></div></div>
      )}
      <div className="rs-hint">
        <Icon name="book" size={16} />
        <span>Want to see what the questions were made from? Open the parsed documents, images and knowledge base.</span>
        <button type="button" className="btn small" onClick={onSource}>Open source data</button>
      </div>
    </>
  );
}

// ------------------------------------------------------------------ step 5: accept and download
function FinishStep({ jobId, job, r, counts, accepted, setAccepted, goReview }) {
  const toast = useToast();
  const [busy, run] = useBusy();
  const [all, setAll] = useState(null);
  useEffect(() => { api(`/jobs/${jobId}/rows?status=kept&limit=10000`).then((x) => setAll(x.rows)).catch(() => {}); }, [jobId]);
  const n = (f) => (all ? all.filter(f).length : 0);
  const approved = n((x) => x.review?.action === "approve"), rejected = n((x) => x.review?.action === "reject"), edited = n((x) => x.review?.edited);
  const open = all ? all.length - approved - rejected : counts?.review ?? 0;
  const final = (r.kept ?? 0) - rejected;
  const accept = () => run(async () => {
    try { setAccepted(await post(`/jobs/${jobId}/accept`)); toast("Dataset written"); } catch (e) { toast(e.message, "error"); }
  });
  const files = [["train.jsonl", "Train set", "for training the model"], ["val.jsonl", "Validation set", "for checking during training"], ["test.jsonl", "Test set", "for the final evaluation"],
    ["quality_report.json", "Quality report", "all the numbers from this page"], ...(accepted ? [["all.jsonl", "Everything", "all rows in one file"]] : [])];
  return (
    <>
      <div className="rs-fin">
        <div className="rs-fin-n"><b className="num">{fmtN(final)}</b><span>rows will be written</span></div>
        <ul className="rs-fin-l">
          <li><i className="ok" /><b className="num">{fmtN(approved)}</b> approved{edited ? ` (${fmtN(edited)} edited)` : ""}</li>
          <li><i className="bad" /><b className="num">{fmtN(rejected)}</b> rejected, left out</li>
          <li><i /><b className="num">{fmtN(open)}</b> not reviewed, kept as they are</li>
        </ul>
        {open > 0 && <button type="button" className="btn small ghost" onClick={goReview}><Icon name="back" size={13} />Back to review</button>}
      </div>

      <div className={`rs-accept${accepted ? " done" : ""}`}>
        <span className="rs-num">1</span>
        <div className="grow"><b>{accepted ? "Dataset written" : "Accept the dataset"}</b>
          <p>{accepted ? <>Wrote {fmtN(accepted.rows)} rows to <span className="mono">{accepted.path}</span></> : "Saves the final files on the server, with your reviews applied."}</p></div>
        <button type="button" className={`btn ${accepted ? "" : "primary"}`} disabled={busy || job?.status !== "done"} onClick={accept}>
          <Icon name={accepted ? "refresh" : "check"} size={15} />{busy ? "Writing…" : accepted ? "Accept again" : "Accept dataset"}</button>
      </div>

      <div className="rs-dl">
        <div className="rs-dl-h"><span className="rs-num">2</span><div><b>Download the files</b><p>Use these to fine-tune your model.</p></div></div>
        <div className="rs-files">
          {files.map(([fn, l, d]) => (
            <a key={fn} className="rs-file" href={fileUrl(jobId, fn)} download={fn}>
              <span className="rs-file-ico"><Icon name="doc" size={16} /></span>
              <span className="rs-file-t"><b>{l}</b><small>{d}</small></span>
              <span className="mono rs-file-n">{fn}</span>
              <Icon name="download" size={15} className="rs-file-dl" />
            </a>
          ))}
        </div>
      </div>
    </>
  );
}

function Insights({ r }) {
  const f = r.funnel || {}, im = r.image_funnel || {};
  const mix = r.mix || { documents: r.kept - (r.paraphrases || 0) - (r.declines || 0), paraphrases: r.paraphrases || 0, decline: r.declines || 0, general: 0 };
  return (
    <div className="rs-ins">
      <Panel title="Where the questions went" d={0}>
        <Funnel steps={[["Generated", f.questions_generated], ["After dedupe", f.after_question_dedupe], ["Answered", f.answered], ["Passed the judge", f.passed_judge], ["In the dataset", f.kept, true]]} />
      </Panel>
      <Panel title="Judge scores of kept pairs" d={1}><ScoreStacks scores={r.scores_of_kept || {}} /></Panel>
      <Panel title="What the dataset is made of" d={2}>
        <StackBar parts={[["From documents", mix.documents, "var(--mix-1)"], ["Paraphrases", mix.paraphrases, "var(--mix-2)"], ["Declines", mix.decline, "var(--mix-3)"], ["General", mix.general, "var(--mix-4)"]]} />
      </Panel>
      <Panel title="Question types" d={3}><TypeDonut types={r.types || {}} /></Panel>
      <Panel title="Why pairs were removed" d={4}>
        <Bars empty="Nothing was removed." rows={Object.entries(r.drop_reasons || {}).sort((a, b) => b[1] - a[1]).map(([k, v]) => [k, v, /duplicate/.test(k) ? "var(--mix-3)" : "var(--accent-400)"])} />
      </Panel>
      <Panel title="Images to Q&A pairs" right={im.read_images === false ? "images ignored" : im.vision_analysis ? "vision model" : "OCR only"} d={5}><ImagePath im={im} /></Panel>
      <Panel title="Split per section" right={r.split_by ? `by ${r.split_by}` : ""} d={6} wide><SplitTable cov={r.split_coverage || {}} /></Panel>
    </div>
  );
}

function Panel({ title, right, d = 0, wide, children }) {
  return (
    <section className={`rs-panel${wide ? " wide" : ""}`} style={{ "--d": d }}>
      <header><h3>{title}</h3><div className="grow" />{right && <span className="rs-note-r">{right}</span>}</header>
      {children}
    </section>
  );
}

// ------------------------------------------------------------------ review list
function Review({ jobId, job, onCounts }) {
  const toast = useToast();
  const [filter, setFilter] = useState({ status: "kept", type: "", q: "", offset: 0 });
  const [rows, setRows] = useState(null);
  const [loading, setLoading] = useState(true);

  const load = useCallback(() => {
    const qs = new URLSearchParams({ status: filter.status, offset: filter.offset, limit: LIMIT });
    if (filter.type) qs.set("type", filter.type);
    if (filter.q) qs.set("q", filter.q);
    setLoading(true);
    api(`/jobs/${jobId}/rows?${qs}`).then(setRows).catch((e) => { if (e.status !== 409) toast(e.message, "error"); }).finally(() => setLoading(false));
  }, [jobId, filter, toast]);
  useEffect(() => { const t = setTimeout(load, filter.q ? 300 : 0); return () => clearTimeout(t); }, [load, filter.q]);

  useEffect(() => { if (rows?.counts) onCounts?.(rows.counts); }, [rows, onCounts]);
  const setF = (k, v) => setFilter((x) => ({ ...x, [k]: v, offset: k === "offset" ? v : 0 }));
  const c = rows?.counts || {};
  const more = MORE.find(([k]) => k === filter.status);
  return (
    <section className="rs-review">
      <div className="rs-filters">
        <div className="rs-seg" role="tablist">
          {SHOW.map(([k, l]) => (
            <button type="button" role="tab" aria-selected={filter.status === k} key={k} className={filter.status === k ? "on" : ""} onClick={() => setF("status", k)}>
              {l}<span className="num">{fmtN(c[k] ?? 0)}</span></button>
          ))}
          {more && <button type="button" className="on" role="tab" aria-selected="true">{more[1]}<span className="num">{fmtN(c[more[0]] ?? 0)}</span></button>}
        </div>
        <div className="grow" />
        <label className="search"><Icon name="search" size={14} /><input placeholder="Search" value={filter.q} onChange={(e) => setF("q", e.target.value)} /></label>
        <label className="rs-select" title="Filter">
          <select value={filter.type || (more ? `@${more[0]}` : "")} aria-label="Filter"
            onChange={(e) => { const v = e.target.value; if (v.startsWith("@")) setFilter((x) => ({ ...x, status: v.slice(1), type: "", offset: 0 })); else setF("type", v); }}>
            <optgroup label="Question type">{TYPES.map(([v, l]) => <option key={v} value={v}>{l}</option>)}</optgroup>
            <optgroup label="Show only">{MORE.map(([k, l]) => <option key={k} value={`@${k}`}>{l} ({fmtN(c[k] ?? 0)})</option>)}</optgroup>
          </select>
        </label>
      </div>

      <div className={`rs-list${loading ? " is-loading" : ""}`} key={`${filter.status}|${filter.type}|${filter.offset}`}>
        {!rows ? <div className="rs-skel rows"><i /><i /><i /></div>
          : rows.rows.length === 0 ? <Empty icon="search" title="Nothing here">Try another filter or search.</Empty>
          : rows.rows.map((row, i) => (
            <Pair key={row.q_id} i={i} row={row} jobId={jobId} job={job}
              onChange={(review) => setRows((x) => {
                const was = !!row.review?.action, now = !!review?.action;
                const counts = { ...x.counts, review: (x.counts?.review ?? 0) + (was && !now ? 1 : !was && now ? -1 : 0) };
                return { ...x, counts, rows: x.rows.map((y) => (y.q_id === row.q_id ? { ...y, review } : y)) };
              })} />
          ))}
      </div>

      {rows && rows.total > LIMIT && (
        <footer className="rs-pager">
          <span className="muted small"><b className="num">{filter.offset + 1}–{Math.min(filter.offset + LIMIT, rows.total)}</b> of <b className="num">{fmtN(rows.total)}</b></span>
          <div className="grow" />
          <button type="button" className="btn small ghost" disabled={filter.offset === 0} onClick={() => setF("offset", Math.max(0, filter.offset - LIMIT))}><Icon name="back" size={13} />Previous</button>
          <button type="button" className="btn small ghost" disabled={filter.offset + LIMIT >= rows.total} onClick={() => setF("offset", filter.offset + LIMIT)}>Next<Icon name="arrow" size={13} /></button>
        </footer>
      )}
    </section>
  );
}























//Pair.jsx
// One Q&A pair in the review list. The question and answer come first; scores, context and other details open on request.
import { useState } from "react";
import { post } from "./source.js";
import { topicLabel } from "../model.js";
import { Icon, useBusy, useToast } from "../ui.jsx";

export const SCORES = { grounded: "Grounded", correct: "Correct", clear: "Clear", self_contained: "Self-contained", useful: "Useful", trace: "Trace" };

export default function Pair({ row, i = 0, jobId, job, onChange }) {
  const toast = useToast();
  const rv = row.review || {};
  const [edit, setEdit] = useState(false);
  const [open, setOpen] = useState(false);
  const [q, setQ] = useState(rv.question || row.question);
  const [a, setA] = useState(rv.answer || row.answer);
  const [busy, run] = useBusy();
  const act = (action, extra = {}) => run(async () => {
    try {
      const res = await post(`/jobs/${jobId}/rows/${encodeURIComponent(row.q_id)}`, { action, ...extra });
      onChange(Object.keys(res.review || {}).length ? res.review : null);
      setEdit(false);
    } catch (e) { toast(e.message, "error"); }
  });

  let related = [];
  try { related = JSON.parse(row.related_ids || "[]"); } catch { /* ignore */ }
  const keys = Object.keys(SCORES).filter((k) => row[`score_${k}`] != null);
  const avg = keys.length ? keys.reduce((s, k) => s + row[`score_${k}`], 0) / keys.length : null;
  const kept = row.status === "kept";
  const origin = row.doc_name || (row.source === "general" ? "General knowledge" : row.source === "decline" ? "Decline rule" : "");
  const section = (row.heading_path || "").split(" > ").pop();
  const kind = row.decline_kind || (row.general_topic ? topicLabel(row.general_topic) : "");
  const minG = job?.request?.judge?.min_grounded ?? 4;
  const why = row.status === "dropped" ? keys.filter((k) => row[`score_${k}`] < 3 || (k === "grounded" && row.score_grounded < minG))
    .map((k) => row[`why_${k}`]).filter(Boolean)[0] : "";
  const shownQ = rv.question || row.question, shownA = rv.answer || row.answer;
  const state = !kept ? ["bad", row.status === "duplicate" && row.duplicate_of ? `Duplicate of ${row.duplicate_of}` : row.drop_reason || row.status]
    : rv.action === "reject" ? ["bad", "Rejected"] : rv.edited ? ["ok", "Edited"] : null;

  return (
    <article className={`qa${rv.action ? ` is-${rv.action}` : ""}${!kept ? " is-out" : ""}`} style={{ "--i": i }}>
      <div className="qa-main">
        {edit ? (
          <div className="qa-edit">
            <label><span>Question</span><textarea rows={2} value={q} onChange={(e) => setQ(e.target.value)} /></label>
            <label><span>Answer</span><textarea rows={4} value={a} onChange={(e) => setA(e.target.value)} /></label>
            <div className="qa-edit-act">
              <button type="button" className="btn small ghost" onClick={() => setEdit(false)}>Cancel</button>
              <button type="button" className="btn small primary" disabled={busy} onClick={() => act("edit", { question: q.trim(), answer: a.trim() })}><Icon name="check" size={13} />Save</button>
            </div>
          </div>
        ) : (
          <>
            <p className="qa-q">{shownQ}</p>
            <p className="qa-a">{shownA || <i>No answer</i>}</p>
          </>
        )}
        {why && <p className="qa-why">{why}</p>}
        <div className="qa-meta">
          {origin && <span className="ellipsis" title={row.heading_path || origin}>{origin}{section && section !== origin ? ` · ${section}` : ""}</span>}
          {row.type && <span className="qa-type">{row.type.replace(/_/g, " ")}</span>}
          {kind && <span className="qa-type">{kind}</span>}
          {avg != null && <span className="qa-avg" title="Average judge score">{avg.toFixed(1)}<small>/5</small></span>}
          <button type="button" className={`qa-more-btn${open ? " on" : ""}`} onClick={() => setOpen((o) => !o)} aria-expanded={open}>
            {open ? "Less" : "Details"}<Icon name="chevron" size={12} /></button>
        </div>
        {open && (
          <div className="qa-more">
            {keys.length > 0 && (
              <div className="qa-scores">
                {keys.map((k) => {
                  const v = row[`score_${k}`];
                  return (
                    <div key={k} className={`qa-score ${v >= 4 ? "hi" : v === 3 ? "mid" : "lo"}`}>
                      <span>{SCORES[k]}</span><span className="qa-pips">{[1, 2, 3, 4, 5].map((n) => <i key={n} className={n <= v ? "on" : ""} />)}</span><b className="num">{v}</b>
                      {row[`why_${k}`] && <small>{row[`why_${k}`]}</small>}
                    </div>
                  );
                })}
              </div>
            )}
            {row.thinking && <div className="qa-kv"><span>Reasoning</span><p>{row.thinking}</p></div>}
            <div className="qa-kv"><span>Details</span>
              <p className="qa-chips">
                <span className="cite">{row.q_id}</span>
                {row.split && <span className="cite">{row.split}</span>}
                {row.variant_of && <span className="cite">paraphrase of {row.variant_of}</span>}
                {row.chunk_id && <span className={`cite${row.source === "image" ? " img" : ""}`}>context {row.chunk_id}{row.source === "image" ? " · image" : ""}</span>}
                {related.map((c) => <span key={c} className="cite">{c}</span>)}
                {row.heading_path && <span className="muted small">{row.heading_path.replace(/ > /g, " › ")}</span>}
              </p>
            </div>
          </div>
        )}
      </div>

      <div className="qa-side">
        {state && <span className={`qa-state ${state[0]}`}>{state[1]}</span>}
        {kept && !edit && (
          <div className="qa-act">
            <button type="button" className={`qa-btn yes${rv.action === "approve" ? " on" : ""}`} disabled={busy} onClick={() => act(rv.action === "approve" ? "reset" : "approve")}
              title={rv.action === "approve" ? "Undo approve" : "Approve"} aria-label="Approve"><Icon name="check" size={15} /></button>
            <button type="button" className={`qa-btn no${rv.action === "reject" ? " on" : ""}`} disabled={busy} onClick={() => act(rv.action === "reject" ? "reset" : "reject")}
              title={rv.action === "reject" ? "Undo reject" : "Reject"} aria-label="Reject"><Icon name="x" size={15} /></button>
            <button type="button" className="qa-btn" onClick={() => { setQ(shownQ); setA(shownA); setEdit(true); }} title="Edit" aria-label="Edit"><Icon name="edit" size={14} /></button>
          </div>
        )}
      </div>
    </article>
  );
}




























//Insights.jsx
// Small charts for the Results page. Plain HTML/CSS, coloured from the theme tokens so they work in light and dark mode.
import { useEffect, useState } from "react";
import { fmtN } from "../model.js";
import { SCORES } from "./Pair.jsx";

const calm = () => typeof matchMedia === "function" && matchMedia("(prefers-reduced-motion: reduce)").matches;

// counts a number up from 0 with an ease-out, for the headline figures
export function useCount(target, ms = 900) {
  const [v, setV] = useState(calm() ? target : 0);
  useEffect(() => {
    if (typeof target !== "number" || calm()) { setV(target); return undefined; }
    let raf, t0;
    const step = (t) => { t0 ??= t; const k = Math.min(1, (t - t0) / ms); setV(target * (1 - (1 - k) ** 3)); if (k < 1) raf = requestAnimationFrame(step); };
    raf = requestAnimationFrame(step);
    return () => cancelAnimationFrame(raf);
  }, [target, ms]);
  return v;
}

export function Meter({ value }) {
  return <span className="rs-meter"><i style={{ "--w": `${Math.max(0, Math.min(1, value)) * 100}%` }} /></span>;
}

export function Donut({ value, label, sub }) {
  const r = 26, c = 2 * Math.PI * r;
  return (
    <span className="rs-donut">
      <svg viewBox="0 0 64 64" aria-hidden="true">
        <defs><linearGradient id="rs-g" x1="0" y1="0" x2="1" y2="1"><stop offset="0" stopColor="var(--grad-a)" /><stop offset="1" stopColor="var(--grad-b)" /></linearGradient></defs>
        <circle cx="32" cy="32" r={r} className="trk" />
        <circle cx="32" cy="32" r={r} className="val" style={{ "--c": c, "--o": c * (1 - Math.max(0, Math.min(1, value))) }} />
      </svg>
      <span className="rs-donut-t"><b className="num">{label}</b>{sub && <small>{sub}</small>}</span>
    </span>
  );
}

export function StackBar({ parts, note }) {
  const tot = parts.reduce((a, p) => a + (p[1] || 0), 0) || 1;
  return (
    <div className="rs-stack">
      <span className="rs-stack-bar">{parts.filter((p) => p[1]).map(([l, v, c], k) => <i key={l} title={`${l}: ${fmtN(v)}`} style={{ "--w": `${(100 * v) / tot}%`, "--c": c, "--k": k }} />)}</span>
      <div className="rs-legend">
        {parts.map(([l, v, c]) => <span key={l} style={{ "--c": c }}><i />{l}<b className="num">{fmtN(v)}</b></span>)}
      </div>
      {note && <small className="muted">{note}</small>}
    </div>
  );
}

export function Funnel({ steps }) {
  const max = Math.max(1, ...steps.map((s) => s[1] || 0));
  return (
    <div className="rs-funnel">
      {steps.map(([l, v, end], k) => {
        const prev = k ? steps[k - 1][1] : null;
        const delta = prev != null && v != null && prev > 0 && v < prev ? Math.round((100 * (prev - v)) / prev) : null;
        return (
          <div className={`rs-fstep${end ? " end" : ""}`} key={l} style={{ "--k": k }}>
            <span className="rs-fl">{l}</span>
            <span className="rs-ft"><i style={{ "--w": `${(100 * (v || 0)) / max}%` }} /></span>
            <b className="num">{fmtN(v)}</b>
            <small className="rs-fd num">{delta != null ? `−${delta}%` : ""}</small>
          </div>
        );
      })}
    </div>
  );
}

const SCALE = ["var(--sc-1)", "var(--sc-2)", "var(--sc-3)", "var(--sc-4)", "var(--sc-5)"];
export function ScoreStacks({ scores }) {
  const rows = Object.entries(scores);
  if (!rows.length) return <div className="muted small">No judged pairs kept.</div>;
  return (
    <div className="rs-scores">
      {rows.map(([k, m], j) => {
        const tot = Object.values(m).reduce((a, b) => a + b, 0) || 1;
        const avg = Object.entries(m).reduce((a, [n, c]) => a + Number(n) * c, 0) / tot;
        return (
          <div className="rs-srow" key={k} style={{ "--k": j }}>
            <span className="rs-sl">{SCORES[k] || k}</span>
            <span className="rs-sbar">{[1, 2, 3, 4, 5].map((n) => {
              const p = (100 * (m[n] || 0)) / tot;
              return p ? <i key={n} style={{ "--w": `${p}%`, "--c": SCALE[n - 1] }} title={`score ${n}: ${Math.round(p)}%`} /> : null;
            })}</span>
            <b className="num">{avg.toFixed(1)}</b>
          </div>
        );
      })}
      <div className="rs-sleg">{[1, 2, 3, 4, 5].map((n) => <span key={n} style={{ "--c": SCALE[n - 1] }}><i />{n}</span>)}<span className="muted">· average on the right</span></div>
    </div>
  );
}

const TYPE_C = { factual: "var(--mix-1)", explanation: "var(--accent-400)", "how-to": "var(--mix-2)", comparison: "var(--split-2)",
  from_images: "var(--violet)", decline: "var(--mix-3)", general: "var(--green)" };
export function TypeDonut({ types }) {
  const rows = Object.entries(types).sort((a, b) => b[1] - a[1]);
  const tot = rows.reduce((a, r) => a + r[1], 0);
  if (!tot) return <div className="muted small">No questions yet.</div>;
  let at = 0;
  const stops = rows.map(([k, v]) => { const c = TYPE_C[k] || "var(--faint)"; const s = `${c} ${at}% ${(at += (100 * v) / tot)}%`; return s; }).join(", ");
  return (
    <div className="rs-types">
      <span className="rs-pie" style={{ "--g": `conic-gradient(${stops})` }}><span><b className="num">{fmtN(tot)}</b><small>pairs</small></span></span>
      <div className="rs-tlist">
        {rows.map(([k, v]) => (
          <span key={k} style={{ "--c": TYPE_C[k] || "var(--faint)" }}><i />{k.replace(/_/g, " ")}<b className="num">{fmtN(v)}</b><small className="num">{Math.round((100 * v) / tot)}%</small></span>
        ))}
      </div>
    </div>
  );
}

export function Bars({ rows, empty }) {
  if (!rows.length) return <div className="muted small">{empty}</div>;
  const max = Math.max(1, ...rows.map((r) => r[1] || 0));
  return (
    <div className="rs-bars">
      {rows.map(([l, v, c], k) => (
        <div className="rs-brow" key={l} style={{ "--k": k }}>
          <span className="rs-bl" title={l}>{l}</span>
          <span className="rs-bt"><i style={{ "--w": `${(100 * v) / max}%`, "--c": c || "var(--accent)" }} /></span>
          <b className="num">{fmtN(v)}</b>
        </div>
      ))}
    </div>
  );
}

export function ImagePath({ im }) {
  const steps = [["Found", im.found], ["Read", im.read], ["Chunks", im.image_chunks], ["Questions", im.questions], ["Answered", im.answered], ["Kept", im.kept]];
  return (
    <>
      <div className="rs-ipath">
        {steps.map(([l, v], k) => <div key={l} className={k === steps.length - 1 ? "end" : ""} style={{ "--k": k }}><b className="num">{fmtN(v ?? 0)}</b><small>{l}</small></div>)}
      </div>
      {(im.hint || im.without_image_data > 0) && <div className="rs-note">{[im.hint, im.without_image_data ? `${im.without_image_data} picture(s) had no image data.` : ""].filter(Boolean).join(" ")}</div>}
      {!im.hint && Object.keys(im.skip_reasons || {}).length > 0 && <div className="muted small">Skipped: {Object.entries(im.skip_reasons).map(([k, v]) => `${v} × ${k}`).join(" · ")}</div>}
    </>
  );
}

export function SplitTable({ cov }) {
  const rows = Object.entries(cov);
  if (!rows.length) return <div className="muted small">No sections.</div>;
  return (
    <table className="rs-tbl">
      <thead><tr><th>Section</th><th>Train</th><th>Val</th><th>Test</th></tr></thead>
      <tbody>{rows.map(([sec, c]) => (
        <tr key={sec} className={!c.train && (c.val || c.test) ? "bad" : ""}><td title={sec}>{sec}</td>
          <td className="num">{fmtN(c.train || 0)}</td><td className="num">{fmtN(c.val || 0)}</td><td className="num">{fmtN(c.test || 0)}</td></tr>
      ))}</tbody>
    </table>
  );
}


























//SourceDrawer.jsx
// Source data drawer for the Results page: the parsed documents and images, and the knowledge base. Ported from ui-react.
import { useEffect, useRef, useState } from "react";
import { createPortal } from "react-dom";
import { api, imageUrl } from "./source.js";
import { fmtN } from "../model.js";
import { Drawer, Empty, Icon } from "../ui.jsx";

const ext = (e) => { e = (e || "").replace(".", "").toLowerCase(); return e === "htm" ? "html" : e === "markdown" ? "md" : e; };

export default function SourceDrawer({ jobId, job, tab, setTab, onClose }) {
  const open = !!tab && !!jobId;
  if (!open) return null;
  // rendered on <body> so the page's entrance animation can't trap the fixed overlay
  return createPortal(
    <Drawer open={open} title="Source data" onClose={onClose}
      tabs={<div className="pilltabs inline-tabs">
        <button type="button" className={tab === "documents" ? "on" : ""} onClick={() => setTab("documents")}><Icon name="doc" size={14} />Documents &amp; images</button>
        <button type="button" className={tab === "kb" ? "on" : ""} onClick={() => setTab("kb")}><Icon name="book" size={14} />Knowledge base</button>
      </div>}>
      {open && (tab === "documents" ? <Documents jobId={jobId} job={job} /> : <KnowledgeBase jobId={jobId} job={job} />)}
    </Drawer>,
    document.body,
  );
}

function Documents({ jobId, job }) {
  const [docs, setDocs] = useState(null);
  const [wait, setWait] = useState(false);
  const [sel, setSel] = useState(null);
  const [md, setMd] = useState("");
  const [imgTab, setImgTab] = useState("read");
  const mdRef = useRef(null);
  useEffect(() => {
    api(`/jobs/${jobId}/documents`).then((d) => { setDocs(d); setWait(false); setSel((s) => s || d.find((x) => x.status === "ok")?.doc_id || d[0]?.doc_id); })
      .catch((e) => { if (e.status === 409) setWait(true); });
  }, [jobId, job?.status]);
  const d = docs?.find((x) => x.doc_id === sel);
  useEffect(() => {
    setMd("");
    if (d?.status === "ok") api(`/jobs/${jobId}/documents/${encodeURIComponent(d.doc_id)}/markdown`).then(setMd).catch(() => setMd("Markdown is not ready yet."));
  }, [jobId, d?.doc_id, d?.status]);

  if (wait) return <Empty icon="doc" title="The documents are not parsed yet" />;
  if (!docs) return <div className="muted">Loading…</div>;
  const read = d ? d.images.filter((i) => i.status === "read") : [], skipped = d ? d.images.filter((i) => i.status !== "read") : [];
  const vision = job?.request?.parse?.vision_analysis && job?.request?.endpoints?.vision;
  const imgs = imgTab === "read" ? read : skipped;
  const jump = (name) => {
    const el = [...(mdRef.current?.querySelectorAll(".md-h") || [])].find((x) => x.dataset.h === name);
    if (el) mdRef.current.scrollTop = el.offsetTop - mdRef.current.offsetTop - 8;
  };
  return (
    <div className="src-grid">
      <div className="src-col">
        <section className="card"><div className="card-hd"><div className="card-t"><h2>Files</h2></div></div>
          <div className="card-bd">
            {docs.map((x) => {
              const r = x.images.filter((i) => i.status === "read").length;
              return (
                <button type="button" key={x.doc_id} className={`file pick${x.doc_id === sel ? " on" : ""}`} onClick={() => setSel(x.doc_id)}>
                  <span className={`ext ${ext(x.ext)}`}>{ext(x.ext).toUpperCase()}</span>
                  <span className="file-n">{x.name}<small>{x.status === "ok" ? [x.pages ? `${x.pages} pages` : "", `${x.chunks} chunks`, x.images.length ? `${r} of ${x.images.length} images read` : ""].filter(Boolean).join(" · ") : x.reason}</small></span>
                  <span className={`tag ${x.status === "ok" ? "ok" : "bad"}`}>{x.status === "ok" ? "Read" : "Skipped"}</span>
                </button>
              );
            })}
          </div></section>
        {d && <section className="card"><div className="card-hd"><div className="card-t"><h2>Sections</h2><small>{d.name}</small></div></div>
          <div className="card-bd">
            {d.sections.map((s) => {
              const depth = (s.heading_path || "").split(" > ").length - 1, name = (s.heading_path || "(no heading)").split(" > ").pop();
              const n = d.images.filter((i) => i.heading_path === s.heading_path).length;
              return <button type="button" key={s.heading_path} className="sec-row" style={{ paddingLeft: 8 + 14 * Math.min(depth, 3) }} onClick={() => jump(name)}>
                <span className="ellipsis">{name}</span><span className="muted small">{s.chunks} chunk{s.chunks > 1 ? "s" : ""}{n ? ` · ${n} image${n > 1 ? "s" : ""}` : ""}</span></button>;
            })}
          </div></section>}
      </div>
      {d && <div className="src-col">
        <section className="card"><div className="card-hd"><div className="card-t"><h2>Parsed Markdown</h2><small>{d.name} · {d.chunks} chunks{d.pages ? ` · ${d.pages} pages` : ""}</small></div></div>
          <div className="card-bd"><div className="md" ref={mdRef}>{d.status === "ok" ? <Markdown text={md} /> : `Not read: ${d.reason}`}</div></div></section>
        <section className="card"><div className="card-hd"><div className="card-t"><h2>Images</h2><small>{d.name}</small></div><div className="grow" />
          <div className="pilltabs"><button type="button" className={imgTab === "read" ? "on" : ""} onClick={() => setImgTab("read")}>Read<span>{read.length}</span></button>
            <button type="button" className={imgTab === "skipped" ? "on" : ""} onClick={() => setImgTab("skipped")}>Skipped<span>{skipped.length}</span></button></div></div>
          <div className="card-bd">
            {!vision && d.images.length > 0 && <div className="info small">Vision analysis was off for this job: images keep only their OCR text.</div>}
            {imgs.length === 0 ? <div className="muted small">No {imgTab === "read" ? "images read" : "images skipped"} in this document.</div> : imgs.map((i) => (
              <div className="imgcard" key={i.pic_id || i.file}>
                <div className="thumb">{i.file && <img loading="lazy" alt={`Figure ${i.figure}`} src={imageUrl(jobId, i.file)} />}</div>
                <div className="img-b">
                  <div className="img-h"><b>{i.figure ? `Figure ${i.figure}` : "Image"}{i.page ? ` · page ${Math.round(i.page)}` : ""}</b>
                    {(i.type || i.source) && <span className="tag">{i.type || i.source}</span>}<div className="grow" />
                    <span className={`tag ${i.status === "read" ? "ok" : "bad"}`}>{i.status === "read" ? `Read · ${i.source === "ocr" ? "OCR" : "vision model"}` : `Skipped${i.reason ? ` · ${i.reason}` : ""}`}</span></div>
                  {i.caption && <p><small>Caption</small>{i.caption}</p>}
                  {i.visible_text && <p><small>Visible text</small>{i.visible_text.length > 600 ? `${i.visible_text.slice(0, 600)}…` : i.visible_text}</p>}
                  {i.description && <p><small>Description</small>{i.description}</p>}
                  <span className="muted small">{i.heading_path}{i.width ? ` · ${i.width}×${i.height}px` : ""}</span>
                </div>
              </div>
            ))}
          </div></section>
      </div>}
    </div>
  );
}

function Markdown({ text }) {
  const out = [];
  let img = null;
  text.split("\n").forEach((line, k) => {
    if (line.startsWith("<!-- image:")) { img = [line]; return; }
    if (img) {
      if (line.trim() === "") { out.push(<span key={`i${k}`} className="md-img">{img.join("\n")}</span>); img = null; return; }
      img.push(line); return;
    }
    const h = line.match(/^(#{1,6})\s+(.*)$/);
    out.push(h ? <span key={k} className="md-h" data-h={h[2].trim()}>{line}{"\n"}</span> : <span key={k}>{line}{"\n"}</span>);
  });
  if (img) out.push(<span key="last" className="md-img">{img.join("\n")}</span>);
  return <>{out}</>;
}

function KnowledgeBase({ jobId, job }) {
  const [kb, setKb] = useState(null);
  const [wait, setWait] = useState(false);
  useEffect(() => {
    api(`/jobs/${jobId}/kb`).then((x) => { setKb(x); setWait(false); }).catch((e) => { if (e.status === 409) setWait(true); });
  }, [jobId, job?.status]);
  if (wait) return <Empty icon="book" title="The knowledge base is not built yet" />;
  if (!kb) return <div className="muted">Loading…</div>;
  const byDoc = {};
  kb.topics.forEach((t) => { (byDoc[t.document] ||= { chunks: 0, kept: 0, topics: [] }); byDoc[t.document].chunks += t.chunks; byDoc[t.document].kept += t.kept_pairs; byDoc[t.document].topics.push(t); });
  const maxT = Math.max(1, ...kb.topics.map((t) => t.kept_pairs));
  const names = Object.fromEntries(kb.documents.map((d) => [d.doc_id, d.name]));
  return (
    <div className="kb">
      <section className="card"><div className="card-hd"><span className="card-ico sm"><Icon name="book" size={15} /></span><div className="card-t"><h2>Collection overview</h2><small>Written by the teacher from the document cards</small></div></div>
        <div className="card-bd"><p className="overview">{kb.overview || "No overview (turned off or not written)."}</p></div></section>
      <div className="doccards">
        {kb.documents.map((d) => (
          <div className="doccard" key={d.doc_id}><b>{d.name}</b><p>{d.summary}</p>
            {(d.topics || []).length > 0 && <div className="chips">{d.topics.map((t) => <span className="chip static" key={t}>{t}</span>)}</div>}
            <span className="muted small">{byDoc[d.name]?.topics.length || 0} sections · {byDoc[d.name]?.chunks || 0} chunks</span></div>
        ))}
      </div>
      <div className="kb-grid">
        <section className="card"><div className="card-hd"><div className="card-t"><h2>Coverage</h2><small>Kept Q&amp;A pairs per section</small></div></div>
          <div className="card-bd">
            {Object.entries(byDoc).map(([doc, x]) => (
              <div key={doc} className="cov-doc"><b>{doc}</b><span className="muted small">{fmtN(x.kept)} pairs</span>
                {x.topics.map((t) => <div className="bar-row" key={t.top_heading}><span className="bar-l" title={t.top_heading}>{t.top_heading || "(no heading)"}</span>
                  <span className="bar-t"><i style={{ width: `${(100 * t.kept_pairs) / maxT}%` }} /></span><span className="bar-v">{fmtN(t.kept_pairs)}</span></div>)}
              </div>
            ))}
          </div></section>
        <section className="card"><div className="card-hd"><div className="card-t"><h2>Glossary</h2><small>{kb.glossary.length} terms</small></div></div>
          <div className="card-bd">
            {kb.glossary.length === 0 ? <div className="muted small">No glossary (turned off or no terms found).</div> :
              <table className="tbl"><thead><tr><th>Term</th><th>Definition</th><th>Found in</th></tr></thead>
                <tbody>{kb.glossary.map((g) => <tr key={g.term}><td><b>{g.term}</b></td><td>{g.definition}</td><td className="muted">{String(g.doc_ids || "").split(",").filter(Boolean).map((x) => names[x] || x).join(", ")}</td></tr>)}</tbody></table>}
          </div></section>
      </div>
    </div>
  );
}




















//results.css
/* ================================================================== Results page (simple version)
   Calm surfaces, coral only where it means something (the dataset, high scores, the main action).
   Every colour comes from the theme tokens, so light and dark mode both work. */
.rs { display: grid; gap: 18px; --sc-1: color-mix(in srgb, var(--accent) 10%, var(--line)); --sc-2: color-mix(in srgb, var(--accent) 28%, var(--surface));
  --sc-3: color-mix(in srgb, var(--accent) 50%, var(--surface)); --sc-4: color-mix(in srgb, var(--accent) 75%, var(--surface)); --sc-5: var(--accent); }
.rs .num { font-variant-numeric: tabular-nums; }

/* ---- entrance: cards rise in one after another */
.rs-in { animation: rs-rise .55s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(var(--d, 0) * 55ms); }
@keyframes rs-rise { from { opacity: 0; transform: translateY(12px); } to { opacity: 1; transform: none; } }

.rs-stack { display: grid; gap: 12px; }
.rs-stack-bar { display: flex; height: 10px; border-radius: 99px; overflow: hidden; gap: 2px; background: var(--line-2); }
.rs-stack-bar i { width: var(--w); background: var(--c); transform-origin: left; animation: rs-grow .9s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.25s + var(--k) * 90ms); }
.rs-legend { display: grid; grid-template-columns: 1fr 1fr; gap: 6px 14px; }
.rs-legend span { display: flex; align-items: center; gap: 7px; font-size: 13px; color: var(--ink-2); min-width: 0; }
.rs-legend i { width: 8px; height: 8px; border-radius: 3px; background: var(--c); flex: none; }
.rs-legend b { margin-left: auto; font-weight: 650; color: var(--ink); }
.rs-stack > small { font-size: 12.5px; }
.rs-funnel { display: grid; gap: 8px; }
.rs-fstep { display: grid; grid-template-columns: 112px minmax(0, 1fr) 30px 40px; gap: 10px; align-items: center; font-size: 13px; position: relative; }
.rs-fl { color: var(--ink-2); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.rs-ft { height: 22px; border-radius: 7px; background: var(--surface-3); overflow: hidden; }
.rs-ft i { display: block; height: 100%; width: var(--w); border-radius: 7px; background: color-mix(in srgb, var(--accent) 22%, var(--surface));
  transform-origin: left; animation: rs-grow .9s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.3s + var(--k) * 90ms); }
.rs-fstep.end .rs-ft i { background: var(--grad); }
.rs-fstep b { text-align: right; font-weight: 650; }
.rs-fd { font-size: 11.5px; font-weight: 600; color: var(--faint); text-align: right; }

.rs-scores { display: grid; gap: 9px; }
.rs-srow { display: grid; grid-template-columns: 104px minmax(0, 1fr) 30px; gap: 10px; align-items: center; font-size: 13px; }
.rs-sl { color: var(--ink-2); }
.rs-sbar { display: flex; height: 12px; border-radius: 99px; overflow: hidden; gap: 2px; background: var(--line-2); }
.rs-sbar i { width: var(--w); background: var(--c); transform-origin: left; animation: rs-grow .8s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.3s + var(--k) * 70ms); }
.rs-srow b { text-align: right; font-weight: 650; }
.rs-sleg { display: flex; gap: 10px; flex-wrap: wrap; font-size: 12px; color: var(--muted); padding-top: 4px; }
.rs-sleg span { display: inline-flex; align-items: center; gap: 4px; }
.rs-sleg i { width: 9px; height: 9px; border-radius: 3px; background: var(--c); }

.rs-types { display: flex; gap: 18px; align-items: center; }
.rs-pie { width: 104px; height: 104px; border-radius: 50%; background: var(--g); flex: none; display: grid; place-items: center; position: relative;
  -webkit-mask: radial-gradient(circle, transparent 0 0, #000 0); animation: rs-spin-in 1s cubic-bezier(.22, 1, .36, 1) .2s both; }
.rs-pie > span { width: 70px; height: 70px; border-radius: 50%; background: var(--surface); display: grid; place-content: center; text-align: center; line-height: 1.1;
  box-shadow: 0 0 0 1px var(--line-2); }
.rs-pie b { font-size: 18px; font-weight: 750; } .rs-pie small { font-size: 11px; color: var(--muted); }
@keyframes rs-spin-in { from { opacity: 0; transform: rotate(-90deg) scale(.85); } }
.rs-tlist { display: grid; gap: 7px; flex: 1; min-width: 0; }
.rs-tlist span { display: grid; grid-template-columns: 10px minmax(0, 1fr) auto 36px; gap: 8px; align-items: center; font-size: 13px; color: var(--ink-2); text-transform: capitalize; }
.rs-tlist i { width: 9px; height: 9px; border-radius: 3px; background: var(--c); }
.rs-tlist b { font-weight: 650; color: var(--ink); } .rs-tlist small { color: var(--muted); text-align: right; font-size: 12px; }

.rs-bars { display: grid; gap: 9px; }
.rs-brow { display: grid; grid-template-columns: minmax(0, 140px) minmax(0, 1fr) 32px; gap: 10px; align-items: center; font-size: 13px; }
.rs-bl { color: var(--ink-2); overflow: hidden; text-overflow: ellipsis; white-space: nowrap; text-transform: capitalize; }
.rs-bt { height: 8px; border-radius: 99px; background: var(--line-2); overflow: hidden; }
.rs-bt i { display: block; height: 100%; width: var(--w); border-radius: 99px; background: var(--c); transform-origin: left;
  animation: rs-grow .8s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.3s + var(--k) * 80ms); }
.rs-brow b { text-align: right; font-weight: 650; }

.rs-ipath { display: grid; grid-template-columns: repeat(6, 1fr); gap: 0; position: relative; }
.rs-ipath::before { content: ""; position: absolute; left: 8%; right: 8%; top: 17px; height: 2px; background: linear-gradient(90deg, var(--line), color-mix(in srgb, var(--accent) 45%, var(--line))); }
.rs-ipath > div { display: grid; justify-items: center; gap: 5px; position: relative; animation: rs-rise .5s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(.3s + var(--k) * 70ms); }
.rs-ipath b { width: 36px; height: 36px; border-radius: 50%; display: grid; place-items: center; font-size: 14px; font-weight: 700; background: var(--surface);
  border: 1px solid var(--line-strong); color: var(--ink); box-shadow: 0 0 0 4px var(--surface); }
.rs-ipath .end b { background: var(--grad); color: #fff; border-color: transparent; box-shadow: 0 0 0 4px var(--surface), 0 4px 10px -3px rgba(var(--accent-rgb), .6); }
.rs-ipath small { font-size: 11.5px; color: var(--muted); }
.rs-note { font-size: 12.5px; color: var(--amber-ink); background: var(--amber-50); border: 1px solid var(--amber-200); border-radius: 9px; padding: 7px 10px; }

.rs-tbl { width: 100%; border-collapse: separate; border-spacing: 0; font-size: 13px; }
.rs-tbl th { text-align: left; font-size: 11px; font-weight: 700; letter-spacing: .5px; text-transform: uppercase; color: var(--faint); padding: 0 8px 8px; border-bottom: 1px solid var(--line); }
.rs-tbl th:not(:first-child), .rs-tbl td:not(:first-child) { text-align: right; width: 52px; }
.rs-tbl td { padding: 8px; border-bottom: 1px solid var(--line-2); color: var(--ink-2); }
.rs-tbl td:first-child { max-width: 170px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; color: var(--ink); font-weight: 500; }
.rs-tbl tr:last-child td { border-bottom: 0; }
.rs-tbl tbody tr { transition: background .12s; } .rs-tbl tbody tr:hover { background: var(--surface-2); }
.rs-tbl tr.bad td { color: var(--red); }

@keyframes rs-grow { from { transform: scaleX(0); } }
.rs-skel { display: grid; grid-template-columns: 1fr 1fr; gap: 14px; padding: 12px; }
.rs-skel i { height: 110px; border-radius: 14px; background: linear-gradient(90deg, var(--surface-3) 0%, var(--surface-2) 50%, var(--surface-3) 100%); background-size: 200% 100%;
  animation: rs-shimmer 1.4s ease-in-out infinite; }
.rs-skel.rows { grid-template-columns: 1fr; padding: 0; } .rs-skel.rows i { height: 120px; }
@keyframes rs-shimmer { from { background-position: 100% 0; } to { background-position: -100% 0; } }
.rs-empty { padding: 8px; }
.rs-empty .empty > span { display: grid; justify-items: center; gap: 14px; }
.rs-chev { color: var(--faint); transform: rotate(90deg); transition: transform .25s cubic-bezier(.22, 1, .36, 1); flex: none; }
.rs-chev.sm { margin-left: -2px; }

/* ---- the four key numbers, in one calm strip */
.rs-sum { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); background: var(--surface); border: 1px solid var(--line); border-radius: 14px; animation: rs-rise .5s cubic-bezier(.22, 1, .36, 1) both; }
.rs-sum > div { display: grid; gap: 4px; padding: 18px 22px; min-width: 0; }
.rs-sum > div + div { border-left: 1px solid var(--line-2); }
.rs-sum span { font-size: 13px; color: var(--muted); }
.rs-sum b { font-family: var(--font); font-size: 28px; font-weight: 700; letter-spacing: -.6px; line-height: 1.15; color: var(--ink); }
.rs-sum b em { font-style: normal; font-size: 16px; font-weight: 500; color: var(--muted); letter-spacing: 0; margin-left: 3px; }
.rs-sum small { font-size: 12.5px; color: var(--faint); }
.rs-sum > div:first-child b { background: var(--grad); -webkit-background-clip: text; background-clip: text; color: transparent; }

/* ---- review */
.rs-review { display: grid; gap: 14px; animation: rs-rise .45s cubic-bezier(.22, 1, .36, 1) both; }
.rs-filters { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }
.rs-seg { display: inline-flex; gap: 2px; padding: 3px; border-radius: 10px; background: var(--surface-3); border: 1px solid var(--line-2); }
.rs-seg button { display: inline-flex; align-items: center; gap: 6px; height: 30px; padding: 0 12px; border: 0; border-radius: 7px; background: none; cursor: pointer;
  font: inherit; font-size: 13.5px; font-weight: 600; color: var(--muted); transition: color .15s, background .15s, box-shadow .15s; }
.rs-seg button:hover { color: var(--ink); }
.rs-seg button.on { background: var(--surface); color: var(--ink); box-shadow: 0 1px 2px rgba(16, 24, 40, .1); }
.rs-seg button span { font-size: 12px; color: var(--faint); font-weight: 600; }
.rs-seg button.on span { color: var(--accent-600); }
.rs-filters .search { height: 36px; }
.rs-filters .search input { width: 170px !important; background: transparent !important; color: var(--ink); }
.rs-select { display: inline-flex; align-items: center; height: 36px; border: 1px solid var(--line-strong); border-radius: 9px; background: var(--surface); box-shadow: var(--shadow-xs); }
.rs-select:focus-within { border-color: var(--accent-400); box-shadow: var(--ring); }
.rs-select select { border: 0 !important; box-shadow: none !important; background-color: transparent; height: 34px; padding: 0 28px 0 12px !important; font-size: 13.5px; color: var(--ink); }

.rs-list { display: grid; background: var(--surface); border: 1px solid var(--line); border-radius: 14px; overflow: hidden; transition: opacity .2s; }
.rs-list.is-loading { opacity: .6; }
.rs-list > .empty { padding: 48px 20px; }
.rs-list > .rs-skel { padding: 12px; }
.rs-pager { display: flex; align-items: center; gap: 8px; padding: 0 4px; }

/* ---- one pair: a row in the list */
.qa { display: grid; grid-template-columns: minmax(0, 1fr) auto; gap: 20px; padding: 18px 20px; position: relative; transition: background .2s;
  animation: rs-rise .4s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(var(--i, 0) * 35ms); }
.qa + .qa { border-top: 1px solid var(--line-2); }
.qa:hover { background: var(--surface-2); }
.qa::before { content: ""; position: absolute; left: 0; top: 18px; bottom: 18px; width: 3px; border-radius: 0 3px 3px 0; background: transparent; transition: background .2s; }
.qa.is-approve::before { background: var(--green); }
.qa.is-reject::before { background: var(--red); }
.qa.is-reject .qa-q, .qa.is-reject .qa-a { opacity: .5; }
.qa.is-reject .qa-q { text-decoration: line-through; text-decoration-color: color-mix(in srgb, var(--red) 50%, transparent); }
.qa-main { display: grid; gap: 6px; min-width: 0; }
.qa-q { margin: 0; font-size: 15.5px; font-weight: 600; color: var(--ink); line-height: 1.45; }
.qa-a { margin: 0; font-size: 14.5px; color: var(--ink-2); line-height: 1.6; white-space: pre-wrap; }
.qa.is-out .qa-a { color: var(--muted); }
.qa-why { margin: 2px 0 0; font-size: 13px; color: var(--red-ink); padding-left: 10px; border-left: 2px solid var(--red-200); }
.qa-meta { display: flex; align-items: center; gap: 8px; margin-top: 4px; font-size: 12.5px; color: var(--muted); min-width: 0; flex-wrap: wrap; }
.qa-meta > span.ellipsis { max-width: 360px; }
.qa-type { padding: 1px 8px; border-radius: 99px; background: var(--surface-3); color: var(--ink-2); font-weight: 500; text-transform: capitalize; }
.qa-avg { font-weight: 650; color: var(--ink-2); }
.qa-avg small { font-weight: 400; color: var(--faint); margin-left: 1px; }
.qa-more-btn { display: inline-flex; align-items: center; gap: 3px; border: 0; background: none; padding: 0; cursor: pointer; font: inherit; font-size: 12.5px; font-weight: 600; color: var(--accent-600); }
.qa-more-btn svg { transform: rotate(90deg); transition: transform .2s; }
.qa-more-btn.on svg { transform: rotate(-90deg); }
.qa-more { display: grid; gap: 12px; margin-top: 8px; padding: 14px; border-radius: 12px; background: var(--surface-2); border: 1px solid var(--line-2); animation: rs-rise .3s ease both; }
.qa:hover .qa-more { background: var(--surface); }
.qa-scores { display: grid; grid-template-columns: repeat(auto-fill, minmax(190px, 1fr)); gap: 8px 18px; }
.qa-score { display: grid; grid-template-columns: 1fr auto 16px; gap: 8px; align-items: center; font-size: 13px; color: var(--ink-2); }
.qa-score b { text-align: right; font-weight: 650; }
.qa-score small { grid-column: 1 / -1; font-size: 12px; color: var(--muted); margin-top: -3px; }
.qa-pips { display: inline-flex; gap: 2px; }
.qa-pips i { width: 12px; height: 5px; border-radius: 3px; background: var(--line); }
.qa-score.hi .qa-pips i.on { background: var(--accent); }
.qa-score.mid .qa-pips i.on { background: var(--amber); }
.qa-score.lo .qa-pips i.on { background: var(--red); }
.qa-kv { display: grid; grid-template-columns: 80px minmax(0, 1fr); gap: 10px; font-size: 13px; }
.qa-kv > span { color: var(--faint); font-weight: 600; font-size: 12px; padding-top: 2px; }
.qa-kv p { margin: 0; color: var(--ink-2); }
.qa-chips { display: flex; flex-wrap: wrap; gap: 5px; align-items: center; }

.qa-edit { display: grid; gap: 10px; animation: rs-rise .3s ease both; }
.qa-edit label { display: grid; gap: 4px; }
.qa-edit span { font-size: 12px; font-weight: 600; color: var(--faint); }
.qa-edit textarea { width: 100%; resize: vertical; }
.qa-edit-act { display: flex; gap: 6px; justify-content: flex-end; }

.qa-side { display: flex; flex-direction: column; align-items: flex-end; gap: 10px; }
.qa-state { font-size: 12px; font-weight: 600; padding: 2px 9px; border-radius: 99px; white-space: nowrap; background: var(--surface-3); color: var(--muted); }
.qa-state.ok { background: var(--green-50); color: var(--green-600); } .qa-state.bad { background: var(--red-50); color: var(--red); }
.qa-act { display: flex; gap: 4px; }
.qa-btn { width: 32px; height: 32px; display: grid; place-items: center; border-radius: 9px; border: 1px solid var(--line); background: var(--surface); color: var(--muted);
  cursor: pointer; transition: color .15s, background .15s, border-color .15s, transform .15s; }
.qa-btn:hover:not(:disabled) { color: var(--ink); border-color: var(--line-strong); }
.qa-btn:active:not(:disabled) { transform: scale(.94); }
.qa-btn:disabled { opacity: .5; cursor: default; }
.qa-btn.yes:hover:not(:disabled) { color: var(--green-600); border-color: var(--green-200); background: var(--green-50); }
.qa-btn.no:hover:not(:disabled) { color: var(--red); border-color: var(--red-200); background: var(--red-50); }
.qa-btn.yes.on, .qa-btn.yes.on:hover:not(:disabled) { color: #fff; background: var(--green-600); border-color: var(--green-600); }
.qa-btn.no.on, .qa-btn.no.on:hover:not(:disabled) { color: #fff; background: var(--red); border-color: var(--red); }

/* ---- insights */
.rs-ins { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 16px; align-items: start; }
.rs-panel { display: grid; gap: 14px; align-content: start; padding: 18px 20px; background: var(--surface); border: 1px solid var(--line); border-radius: 14px; animation: rs-rise .5s cubic-bezier(.22, 1, .36, 1) both; animation-delay: calc(var(--d, 0) * 50ms); }
.rs-panel.wide { grid-column: 1 / -1; }
.rs-panel > header { display: flex; align-items: center; gap: 8px; }
.rs-panel h3 { margin: 0; font-size: 14.5px; font-weight: 650; }
.rs-note-r { font-size: 12.5px; color: var(--muted); }

/* ---- guided flow pieces */
.rs-main { min-height: 520px; }
.rs .stp:disabled { cursor: default; opacity: .55; }
.rs-demo { font-size: 11.5px; font-weight: 600; color: var(--muted); padding: 1px 8px; border-radius: 99px; border: 1px dashed var(--line-strong); }
.rs-dot { width: 7px; height: 7px; border-radius: 50%; background: var(--faint); flex: none; display: inline-block; }
.rs-dot.ok { background: var(--green); } .rs-dot.run { background: var(--accent); } .rs-dot.bad { background: var(--red); }

.rs-pickstep { display: grid; gap: 16px; max-width: 760px; }
.rs-jobs { display: grid; gap: 10px; }
.rs-job { display: flex; align-items: center; gap: 14px; padding: 14px 16px; border-radius: 12px; border: 1px solid var(--line); background: var(--surface);
  cursor: pointer; text-align: left; font: inherit; color: var(--ink); transition: border-color .15s, box-shadow .15s, background .15s; }
.rs-job:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); }
.rs-job.on { border-color: var(--accent-400); box-shadow: var(--ring); background: color-mix(in srgb, var(--accent) 3%, var(--surface)); }
.rs-radio { width: 18px; height: 18px; border-radius: 50%; border: 2px solid var(--line-strong); flex: none; display: grid; place-items: center; transition: border-color .15s; }
.rs-radio::after { content: ""; width: 8px; height: 8px; border-radius: 50%; background: var(--grad); transform: scale(0); transition: transform .25s cubic-bezier(.34, 1.56, .64, 1); }
.rs-job.on .rs-radio { border-color: var(--accent); } .rs-job.on .rs-radio::after { transform: scale(1); }
.rs-job-t { display: grid; gap: 2px; flex: 1; min-width: 0; }
.rs-job-t b { font-size: 15px; font-weight: 600; }
.rs-job-t small { font-size: 12px; color: var(--muted); background: none; padding: 0; }
.rs-status { display: inline-flex; align-items: center; gap: 6px; font-size: 12.5px; font-weight: 600; color: var(--muted); white-space: nowrap; }
.rs-status.ok { color: var(--green-600); } .rs-status.bad { color: var(--red); } .rs-status.run { color: var(--accent-600); }

.rs-callout { display: flex; align-items: flex-start; gap: 12px; padding: 14px 16px; border-radius: 12px; border: 1px solid var(--line); background: var(--surface-2); font-size: 14px; }
.rs-callout > svg { margin-top: 2px; flex: none; }
.rs-callout > div { flex: 1; min-width: 0; }
.rs-callout b { display: block; font-weight: 600; margin-bottom: 2px; }
.rs-callout p { margin: 0; color: var(--ink-2); }
.rs-callout ul { margin: 4px 0 0; padding-left: 18px; line-height: 1.6; }
.rs-callout.warn { background: var(--amber-50); border-color: var(--amber-200); color: var(--amber-ink-2); } .rs-callout.warn b, .rs-callout.warn > svg { color: var(--amber-ink); }
.rs-callout.ok { background: var(--green-50); border-color: var(--green-200); } .rs-callout.ok > svg, .rs-callout.ok b { color: var(--green-600); }
.rs-callout.failed { background: var(--red-50); border-color: var(--red-200); } .rs-callout.failed > svg, .rs-callout.failed b { color: var(--red); }
.rs-callout.running > svg, .rs-callout.running b { color: var(--accent-600); }
.rs-hint { display: flex; align-items: center; gap: 12px; padding: 12px 16px; border-radius: 12px; border: 1px dashed var(--line-strong); color: var(--ink-2); font-size: 14px; }
.rs-hint > svg { color: var(--accent); flex: none; } .rs-hint > span { flex: 1; }

.rs-fin { display: flex; align-items: center; gap: 28px; flex-wrap: wrap; padding: 18px 22px; border-radius: 14px; border: 1px solid var(--line); background: var(--surface-2); }
.rs-fin-n { display: grid; }
.rs-fin-n b { font-size: 40px; font-weight: 750; letter-spacing: -1px; line-height: 1; background: var(--grad); -webkit-background-clip: text; background-clip: text; color: transparent; font-family: var(--font); }
.rs-fin-n span { font-size: 13px; color: var(--muted); margin-top: 4px; }
.rs-fin-l { list-style: none; margin: 0; padding: 0 0 0 28px; border-left: 1px solid var(--line); display: grid; gap: 6px; font-size: 14px; color: var(--ink-2); flex: 1; }
.rs-fin-l li { display: flex; align-items: center; gap: 8px; }
.rs-fin-l i { width: 8px; height: 8px; border-radius: 50%; background: var(--line-strong); } .rs-fin-l i.ok { background: var(--green); } .rs-fin-l i.bad { background: var(--red); }
.rs-fin-l b { font-weight: 650; color: var(--ink); min-width: 18px; }
.rs-num { width: 28px; height: 28px; border-radius: 50%; display: grid; place-items: center; font-size: 13px; font-weight: 700; background: var(--surface-3); color: var(--ink-2); flex: none; }
.rs-accept { display: flex; align-items: center; gap: 14px; padding: 16px 18px; border-radius: 14px; border: 1px solid color-mix(in srgb, var(--accent) 22%, var(--line));
  background: color-mix(in srgb, var(--accent) 4%, var(--surface)); }
.rs-accept .rs-num { background: var(--grad); color: #fff; }
.rs-accept.done { border-color: var(--green-200); background: var(--green-50); } .rs-accept.done .rs-num { background: var(--green-600); }
.rs-accept b, .rs-dl-h b { font-size: 15px; font-weight: 650; }
.rs-accept p, .rs-dl-h p { margin: 2px 0 0; font-size: 13.5px; color: var(--muted); }
.rs-accept .mono { color: var(--green-600); }
.rs-dl { display: grid; gap: 12px; padding: 16px 18px; border-radius: 14px; border: 1px solid var(--line); }
.rs-dl-h { display: flex; align-items: center; gap: 14px; }
.rs-files { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 10px; padding-left: 42px; }
.rs-file { display: flex; align-items: center; gap: 12px; padding: 11px 14px; border-radius: 11px; border: 1px solid var(--line); background: var(--surface); color: var(--ink);
  text-decoration: none; transition: border-color .15s, box-shadow .15s, transform .15s; }
.rs-file:hover { border-color: var(--accent-200); box-shadow: var(--shadow-md); transform: translateY(-1px); }
.rs-file-ico { width: 34px; height: 34px; border-radius: 9px; display: grid; place-items: center; background: var(--accent-50); color: var(--accent-600); flex: none; }
.rs-file-t { display: grid; flex: 1; min-width: 0; } .rs-file-t b { font-size: 14px; font-weight: 600; } .rs-file-t small { font-size: 12.5px; color: var(--muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.rs-file-n { font-size: 11.5px; color: var(--faint); }
.rs-file-dl { color: var(--faint); transition: color .15s, transform .2s; flex: none; }
.rs-file:hover .rs-file-dl { color: var(--accent); transform: translateY(2px); }

@media (max-width: 1100px) { .rs-files { grid-template-columns: 1fr; padding-left: 0; } .rs-sum { grid-template-columns: 1fr 1fr; } .rs-sum > div:nth-child(3) { border-left: 0; } .rs-sum > div:nth-child(n+3) { border-top: 1px solid var(--line-2); } .rs-ins { grid-template-columns: 1fr; } }
@media (max-width: 760px) { .qa { grid-template-columns: 1fr; } .qa-side { flex-direction: row; justify-content: space-between; } }
@media (prefers-reduced-motion: reduce) { .rs *, .rs { animation: none !important; transition: none !important; } }
