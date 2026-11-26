//src/model.js
// Form state <-> JobRequest, option lists and small formatters.

export const STEP_TITLES = [
  ["upload", "Upload"], ["parse", "Parse"], ["images", "Images → text"], ["chunk", "Clean & chunk"],
  ["kb", "Knowledge base"], ["questions", "Questions"], ["dedupe_questions", "Question dedupe"], ["answers", "Answers"],
  ["judge", "Judge"], ["paraphrase", "Paraphrases"], ["decline", "Decline questions"], ["general", "General knowledge"],
  ["export", "Clean & export"],
];

export const QUESTION_TYPES = [
  ["factual", "Factual"], ["explanation", "Explanation"], ["how-to", "How-to"], ["from_images", "From images"],
  ["comparison", "Comparison across documents"],
];
export const P_STYLES = [["formal", "Formal"], ["casual", "Casual chat"], ["short", "Very short"], ["keywords", "Search keywords"], ["typos", "With typos"]];
export const G_TOPICS = [
  ["world_facts", "World facts & geography"], ["everyday_howto", "Everyday how-to"], ["science", "Science"],
  ["math", "Math & units"], ["language", "Language & writing"], ["history", "History & culture"], ["technology", "Technology"],
];
export const topicLabel = (t) => (G_TOPICS.find(([k]) => k === t) || [t, t])[1];

export const D_PRESETS = [
  { name: "Off topic", reply_mode: "fixed",
    scope: "Questions that have nothing to do with the subject of these documents, such as cooking, sport, weather, coding help or small talk.",
    reply: "I can only help with questions about the documents I was trained on, and this question is outside them." },
  { name: "Live data", reply_mode: "fixed",
    scope: "Questions about the current, live state of the systems or data these documents describe: how many items exist right now, today's numbers, the status of one specific record.",
    reply: "I don't have access to live or real-time data, so I can't answer that. Please check the system directly." },
  { name: "Not in the documents", reply_mode: "fixed",
    scope: "Questions on the same subject as these documents that a reader might ask, but that the documents do not answer (features, products or details never mentioned).",
    reply: "That isn't covered in the documentation I know, so I can't answer it reliably." },
];
const LEGACY_KINDS = ["off_topic", "live_data", "not_in_documents"];

export const MIX = {
  documents: ["From the documents", "var(--mix-1)"],
  paraphrases: ["Paraphrases", "var(--mix-2)"],
  decline: ["Decline", "var(--mix-3)"],
  general: ["General knowledge", "var(--mix-4)"],
};

// ------------------------------------------------------------------ dataset templates
export const QA_TEMPLATE = {
  id: "qa", name: "Question & answer", builtin: true,
  description: "Questions a reader would ask, answered from the documents (the original pipeline).",
  input: "question", output: "answer", instruction: "", input_prompt: "", output_prompt: "", labels: [], json_fields: [],
  context_in_row: false, format: "chat", variety: [], checks: { grounded: true, rubric: "", min_words: 0, max_words: 0 },
};
// fallbacks until GET /templates answers (same keys as app/schemas.py)
export const T_INPUTS = {
  question: "A question about the text", instruction: "An instruction or task about the text",
  text: "The text itself, with a fixed instruction", text_start: "The first part of the text", custom: "Your own prompt",
};
export const T_OUTPUTS = {
  answer: "The answer", completion: "A completion that follows the instruction", summary: "A summary", json: "JSON with your fields",
  label: "One label from your list", rewrite: "The text rewritten", continuation: "The rest of the text (no teacher)", custom: "Your own prompt",
};
export const T_FORMATS = {
  chat: "Chat messages (system / user / assistant)", instruction: "Instruction / input / output",
  completion: "Prompt / completion", text: "Raw text (continued pretraining)",
};
const IN_SHORT = { question: "Question", instruction: "Instruction", text: "Text + instruction", text_start: "Start of text", custom: "Input" };
const OUT_SHORT = { answer: "Answer", completion: "Completion", summary: "Summary", json: "JSON", label: "Label", rewrite: "Rewrite", continuation: "Rest of text", custom: "Output" };
const FMT_SHORT = { chat: "Chat", instruction: "Instruction / input / output", completion: "Prompt / completion", text: "Raw text" };
export const inShort = (t) => IN_SHORT[t?.input] || "Input";
export const outShort = (t) => OUT_SHORT[t?.output] || "Output";
export const fmtShort = (t) => FMT_SHORT[t?.format] || "Chat";
export const isQa = (t) => !t || (t.input === "question" && t.output === "answer" && !t.input_prompt && !t.output_prompt && !(t.variety || []).length);
export const teacherInputs = (t) => ["question", "instruction", "custom"].includes(t?.input || "question");
// [input, output] words for the Results page
export const ioWords = (t) => (isQa(t) ? ["Question", "Answer"] : t.input === "text" ? ["Instruction", outShort(t)] : [inShort(t), outShort(t)]);
export const sameTemplate = (a, b) => JSON.stringify(cleanTemplate(a)) === JSON.stringify(cleanTemplate(b));

export function cleanTemplate(t) {
  const c = { ...QA_TEMPLATE, ...t, checks: { ...QA_TEMPLATE.checks, ...(t?.checks || {}) } };
  return {
    id: c.id, name: c.name, description: c.description || "", input: c.input, output: c.output,
    instruction: c.instruction || "", input_prompt: c.input_prompt || "", output_prompt: c.output_prompt || "",
    labels: c.labels || [], json_fields: c.json_fields || [], context_in_row: !!c.context_in_row, format: c.format,
    variety: (c.variety || []).map((v) => ({ name: v.name, values: v.values })),
    checks: { grounded: !!c.checks.grounded, rubric: c.checks.rubric || "", min_words: Number(c.checks.min_words) || 0, max_words: Number(c.checks.max_words) || 0 },
  };
}

// the same rules as DatasetTemplate in app/schemas.py
export function templateError(t) {
  if (!/^[a-z0-9][a-z0-9_-]{0,48}$/.test(t.id || "")) return "Template id: lower-case letters, digits, - and _ only";
  if (!(t.name || "").trim()) return "Template: enter a name";
  if (t.input === "custom" && !t.input_prompt.trim()) return "Input 'Your own prompt': write the input prompt";
  if (t.output === "custom" && !t.output_prompt.trim()) return "Output 'Your own prompt': write the output prompt";
  if (t.output === "label" && t.labels.length < 2) return "Output 'One label': enter at least two labels";
  if (t.output === "json" && !t.json_fields.length) return "Output 'JSON': enter at least one field";
  if ((t.output === "continuation") !== (t.input === "text_start")) return "'The first part of the text' goes with output 'The rest of the text' only";
  if (t.checks.max_words && t.checks.min_words > t.checks.max_words) return "Checks: the minimum words must be at most the maximum";
  const names = t.variety.map((v) => v.name);
  const bad = names.find((n) => !/^[a-z][a-z0-9_]{0,30}$/.test(n));
  if (bad !== undefined) return `Variety field "${bad}": lower-case letters, digits and _ only, starting with a letter`;
  if (new Set(names).size !== names.length) return "Variety fields: every field needs its own name";
  const empty = t.variety.find((v) => !v.values.length);
  if (empty) return `Variety field "${empty.name}": enter at least one value`;
  return "";
}
export const slug = (s) => String(s || "").toLowerCase().replace(/[^a-z0-9]+/g, "-").replace(/^-+|-+$/g, "").slice(0, 48) || "template";

// ------------------------------------------------------------------ form
function withPresets(rules) {
  const list = rules.map((r) => ({ reply_mode: "fixed", scope: "", ...r, on: r.on ?? true }));
  D_PRESETS.forEach((p) => {
    if (!list.some((r) => r.name.toLowerCase() === p.name.toLowerCase())) list.push({ ...p, on: false });
  });
  return list;
}

export function defaultForm(defaults) {
  const ep = defaults?.endpoints || {};
  return {
    name: "dataset", mode: "chat", folders: "",
    teacherUrl: ep.teacher?.base_url || "", teacherModel: ep.teacher?.model || "",
    visionUrl: ep.vision?.base_url || "", visionModel: ep.vision?.model || "", apiKey: "", maxParallel: 16,
    vision: true, readImages: true, ocr: true, minOcrChars: 20, minImagePx: 100, skipImageTypes: "logo, icon, decoration", maxImages: 50,
    chunkMin: 500, chunkMax: 800, mergeBelow: 150, related: 4, summary: true, glossary: true, overview: true,
    tpq: 150, perMin: 2, perMax: 8, maxPairs: 2000, types: ["factual", "explanation", "how-to", "from_images"],
    answerLength: "short", tempQ: 0.7, tempA: 0.3, maxAnswerTokens: 512,
    minGrounded: 4, minOther: 3, jaccard: 0.8, teacherJaccard: 0.5, maxRowTokens: 2048,
    para: { enabled: false, count: 3, styles: ["formal", "casual", "short", "keywords"], temperature: 0.9 },
    decline: { enabled: false, share: 10, rules: withPresets(D_PRESETS.slice(0, 2)) },
    general: { enabled: false, share: 10, topics: ["world_facts", "everyday_howto", "science", "language"], answer_length: "short", temperature: 0.3 },
    systemPrompt: "", splitBy: "fact", split: [90, 5, 5], seed: 42, outPath: "",
    template: { ...QA_TEMPLATE },
  };
}

export function formFromRequest(r, defaults) {
  const f = defaultForm(defaults);
  const p = r.parse || {}, c = r.chunk || {}, q = r.questions || {}, o = r.output || {};
  const de = r.decline || {}, ge = r.general || {}, pa = r.paraphrase || {};
  let rules = de.rules;
  if (!rules && de.kinds) {
    rules = de.kinds.map((k) => ({ ...D_PRESETS[LEGACY_KINDS.indexOf(k)], reply: de.answers?.[k] || D_PRESETS[LEGACY_KINDS.indexOf(k)].reply }));
  }
  return {
    ...f,
    name: r.name ?? f.name, mode: r.mode ?? f.mode, folders: (r.files || []).join("\n"),
    teacherUrl: r.endpoints?.teacher?.base_url ?? f.teacherUrl, teacherModel: r.endpoints?.teacher?.model ?? f.teacherModel,
    visionUrl: r.endpoints?.vision?.base_url ?? f.visionUrl, visionModel: r.endpoints?.vision?.model ?? f.visionModel,
    apiKey: r.endpoints?.api_key && r.endpoints.api_key !== "***" ? r.endpoints.api_key : "",
    maxParallel: r.endpoints?.max_parallel ?? f.maxParallel,
    vision: p.vision_analysis ?? f.vision, readImages: p.read_images ?? f.readImages, ocr: p.ocr ?? f.ocr,
    minOcrChars: p.min_ocr_chars ?? f.minOcrChars, minImagePx: p.min_image_px ?? f.minImagePx,
    skipImageTypes: (p.skip_image_types || []).join(", ") || f.skipImageTypes, maxImages: p.max_images_per_doc ?? f.maxImages,
    chunkMin: c.min_tokens ?? f.chunkMin, chunkMax: c.max_tokens ?? f.chunkMax, mergeBelow: c.merge_below_tokens ?? f.mergeBelow,
    related: c.related_chunks ?? f.related, summary: c.summary ?? f.summary, glossary: c.glossary ?? f.glossary, overview: c.overview ?? f.overview,
    tpq: q.tokens_per_question ?? f.tpq, perMin: q.per_chunk?.[0] ?? f.perMin, perMax: q.per_chunk?.[1] ?? f.perMax,
    maxPairs: q.max_pairs ?? f.maxPairs, types: q.types ?? f.types, answerLength: q.answer_length ?? f.answerLength,
    tempQ: q.temperature_questions ?? f.tempQ, tempA: q.temperature_answers ?? f.tempA, maxAnswerTokens: q.max_answer_tokens ?? f.maxAnswerTokens,
    minGrounded: r.judge?.min_grounded ?? f.minGrounded, minOther: r.judge?.min_other ?? f.minOther,
    jaccard: r.dedupe?.jaccard ?? f.jaccard, teacherJaccard: r.dedupe?.teacher_check_jaccard ?? f.teacherJaccard,
    maxRowTokens: o.max_row_tokens ?? f.maxRowTokens,
    para: { ...f.para, ...pa },
    decline: { enabled: !!de.enabled, share: de.share ?? 10, rules: withPresets(rules || D_PRESETS.slice(0, 2)) },
    general: { ...f.general, ...ge },
    systemPrompt: o.system_prompt || "", splitBy: o.split_by || "fact", split: o.split || f.split, seed: o.seed ?? 42, outPath: o.path || "",
    template: r.template ? { ...cleanTemplate(r.template), builtin: !!r.template.builtin } : { ...QA_TEMPLATE },
  };
}

export const visionUsed = (f) => f.readImages && f.vision;

function num(v, label) {
  const n = typeof v === "number" ? v : Number(String(v).replace(/,/g, "").trim());
  if (String(v).trim() === "" || !Number.isFinite(n)) throw new Error(`${label}: enter a number`);
  return n;
}

export function endpointsForRequest(f) {
  if (!f.teacherUrl || !f.teacherModel) throw new Error("Teacher endpoint: enter the URL and model in Settings → Endpoints, then test the endpoints");
  const ep = { teacher: { base_url: f.teacherUrl.trim(), model: f.teacherModel }, max_parallel: num(f.maxParallel, "Parallel requests") };
  if (f.apiKey.trim()) ep.api_key = f.apiKey.trim();
  if (visionUsed(f)) {
    if (!f.visionUrl || !f.visionModel) throw new Error("Vision endpoint: enter the URL and model in Settings → Endpoints, or turn vision analysis off (OCR only)");
    ep.vision = { base_url: f.visionUrl.trim(), model: f.visionModel };
  }
  return ep;
}

export function endpointsForTest(f) {
  const ep = { teacher: { base_url: f.teacherUrl.trim(), model: f.teacherModel || "-" } };
  if (f.apiKey.trim()) ep.api_key = f.apiKey.trim();
  if (visionUsed(f) && f.visionUrl.trim()) ep.vision = { base_url: f.visionUrl.trim(), model: f.visionModel || "-" };
  return ep;
}

export function buildRequest(f, files) {
  const tErr = templateError(cleanTemplate(f.template));
  if (tErr) throw new Error(tErr);
  if (f.template.input === "question" && !f.types.length) throw new Error("Question types: pick at least one");
  const split = f.split.map((x, i) => num(x, ["Train", "Validation", "Test"][i] + " share"));
  if (split.reduce((a, b) => a + b, 0) !== 100) throw new Error("Split: train + validation + test must add up to 100");
  const rules = f.decline.rules.filter((r) => r.on).map(({ name, scope, reply, reply_mode }) => ({ name: name.trim(), scope: scope.trim(), reply: reply.trim(), reply_mode }));
  if (f.decline.enabled) {
    if (!rules.length) throw new Error("Decline answers: use at least one rule (Edit rules)");
    const bad = rules.find((r) => !r.name || !r.reply);
    if (bad) throw new Error(`Decline rule "${bad.name || "(no name)"}": enter a name and a reply`);
    if (new Set(rules.map((r) => r.name.toLowerCase())).size !== rules.length) throw new Error("Decline rules: every rule needs its own name");
  }
  if (f.para.enabled && !f.para.styles.length) throw new Error("Paraphrases: pick at least one style");
  if (f.general.enabled && !f.general.topics.length) throw new Error("General knowledge: pick at least one topic");
  return {
    name: f.name.trim() || "dataset", files, mode: f.mode, endpoints: endpointsForRequest(f),
    parse: {
      ocr: f.ocr, read_images: f.readImages, vision_analysis: f.vision, min_ocr_chars: num(f.minOcrChars, "Min OCR characters"),
      min_image_px: num(f.minImagePx, "Skip images smaller than"), max_images_per_doc: num(f.maxImages, "Max images per document"),
      skip_image_types: f.skipImageTypes.split(",").map((s) => s.trim()).filter(Boolean),
    },
    chunk: {
      min_tokens: num(f.chunkMin, "Chunk min"), max_tokens: num(f.chunkMax, "Chunk max"), merge_below_tokens: num(f.mergeBelow, "Merge below"),
      related_chunks: num(f.related, "Related chunks"), summary: f.summary, glossary: f.glossary, overview: f.overview,
    },
    questions: {
      tokens_per_question: num(f.tpq, "One question per"), per_chunk: [num(f.perMin, "Questions per chunk (min)"), num(f.perMax, "Questions per chunk (max)")],
      max_pairs: num(f.maxPairs, "Max Q&A pairs"), types: f.types, answer_length: f.answerLength,
      temperature_questions: num(f.tempQ, "Question temperature"), temperature_answers: num(f.tempA, "Answer temperature"),
      max_answer_tokens: num(f.maxAnswerTokens, "Max answer tokens"),
    },
    judge: { min_grounded: num(f.minGrounded, "Grounded"), min_other: num(f.minOther, "Other scores") },
    dedupe: { jaccard: num(f.jaccard, "Near-duplicate"), teacher_check_jaccard: num(f.teacherJaccard, "Ask the teacher") },
    paraphrase: { enabled: f.para.enabled, count: num(f.para.count, "Paraphrases per question"), styles: f.para.styles.length ? f.para.styles : ["formal"], temperature: num(f.para.temperature, "Paraphrase temperature") },
    decline: { enabled: f.decline.enabled, share: num(f.decline.share, "Decline share"), rules: f.decline.enabled ? rules : rules.filter((r) => r.name && r.reply) },
    general: { enabled: f.general.enabled, share: num(f.general.share, "General knowledge share"), topics: f.general.topics.length ? f.general.topics : ["world_facts"],
      answer_length: f.general.answer_length, temperature: num(f.general.temperature, "General answer temperature") },
    output: { path: f.outPath.trim() || null, system_prompt: f.systemPrompt.trim() || null, split_by: f.splitBy, split, seed: num(f.seed, "Seed"), max_row_tokens: num(f.maxRowTokens, "Max row length") },
    template: cleanTemplate(f.template),
  };
}

// expected rows per 100 document pairs
export function mixEstimate(f) {
  const n = (v, d) => (Number.isFinite(Number(v)) ? Number(v) : d);
  return {
    documents: 100,
    paraphrases: f.para.enabled ? 100 * n(f.para.count, 3) : 0,
    decline: f.decline.enabled ? n(f.decline.share, 10) : 0,
    general: f.general.enabled ? n(f.general.share, 10) : 0,
  };
}

export function mixWarnings(f) {
  const out = [];
  if (f.general.enabled && f.decline.enabled) {
    const r = f.decline.rules.find((x) => x.on && /off.?topic|nothing to do with/i.test(`${x.name} ${x.scope}`));
    if (r) out.push(`"${r.name}" declines unrelated questions while General knowledge answers them. Narrow that rule or switch it off.`);
  }
  if (f.general.enabled && /only/i.test(f.systemPrompt) && /document|answer only/i.test(f.systemPrompt)) {
    out.push("The system prompt says to answer only from the documents, but General knowledge adds answers from outside them.");
  }
  return out;
}

// ------------------------------------------------------------------ formatting
export const fmtN = (n) => (n ?? 0).toLocaleString();
export const pct = (a, b) => (b ? Math.round((100 * a) / b) : 0);
export const short = (m) => String(m || "").split("/").pop();
export const host = (u) => { try { return new URL(u).host; } catch { return u; } };
export const secs = (s) => (s == null ? "—" : s < 60 ? `${Math.round(s)} s` : s < 3600 ? `${(s / 60).toFixed(1)} min` : `${(s / 3600).toFixed(1)} h`);

const STEP_RESULT = {
  upload: (r) => `${r.files} files · ${r.repeated_or_unsupported} repeated or unsupported`,
  parse: (r) => `${r.documents} documents · ${r.pages} pages · ${r.images} images${r.images_without_data ? ` (+${r.images_without_data} without image data)` : ""}`,
  images: (r) => (r.reason ? r.reason : `${r.read} read · ${r.skipped} skipped · ${r.mode}`),
  chunk: (r) => `${r.chunks} chunks (${r.from_images} from images) · avg ${r.avg_tokens} tokens`,
  kb: (r) => `${r.document_cards} document cards · ${r.glossary_terms} glossary terms`,
  questions: (r) => `${fmtN(r.questions)} inputs from ${r.chunks} chunks`,
  dedupe_questions: (r) => `${r.removed} removed · ${fmtN(r.to_answer)} to answer`,
  answers: (r) => (r.teacher === false ? `${fmtN(r.answered)} taken from the text` : `${fmtN(r.answered)} answered · ${r.not_in_context} not in context · ${r.failed} failed${r.failed_checks ? ` · ${r.failed_checks} failed checks` : ""}`),
  judge: (r) => (r.reason ? r.reason : `${fmtN(r.judged)} judged · ${r.failed} failed`),
  paraphrase: (r) => (r.reason ? r.reason : `${fmtN(r.paraphrases)} paraphrases of ${fmtN(r.facts)} questions`),
  decline: (r) => (r.reason ? r.reason : `${r.declines} decline pairs · ${r.answered_by_documents} removed as answerable`),
  general: (r) => (r.reason ? r.reason : `${r.general} general pairs · ${r.answered_by_documents} answered by the documents · ${r.unsure} unsure`),
  export: (r) => `${fmtN(r.kept)} kept · ${fmtN(r.dropped)} dropped`,
};
// step names for templates other than Q&A
export function stepTitle(key, title, t) {
  if (isQa(t)) return title;
  return { questions: "Inputs", dedupe_questions: "Input dedupe", answers: "Outputs" }[key] || title;
}

export function stepResult(key, r) {
  if (!r || !Object.keys(r).length) return "";
  try {
    const s = STEP_RESULT[key](r);
    if (!s.includes("undefined")) return s;
  } catch { /* fall through */ }
  return Object.entries(r).filter(([, v]) => typeof v !== "object").map(([k, v]) => `${k} ${v}`).join(" · ");
}




















//src/ui.jsx
// Small shared components: icons, switch, segmented control, chips, field, modal, drawer, empty state, toasts.
import { createContext, useCallback, useContext, useEffect, useRef, useState } from "react";
import { createPortal } from "react-dom";

const P = {
  palette: <><path d="M12 3.5a8.5 8.5 0 1 0 0 17c1.2 0 1.8-.8 1.8-1.7 0-.5-.2-.9-.5-1.2-.3-.3-.5-.7-.5-1.2 0-.9.8-1.7 1.7-1.7h2a4 4 0 0 0 4-4c0-4-3.8-7.2-8.5-7.2z" /><circle cx="7.6" cy="11" r="1.1" /><circle cx="10" cy="7.4" r="1.1" /><circle cx="14.4" cy="7.4" r="1.1" /></>,
  lock: <><rect x="5" y="10.5" width="14" height="10" rx="2.5" /><path d="M8.5 10.5V7.5a3.5 3.5 0 0 1 7 0v3M12 14.5v2.5" /></>,
  doc: <><path d="M6 3h8l4 4v14H6z" /><path d="M14 3v4h4M9 12h6M9 16h6" /></>,
  upload: <><path d="M12 16V5M7.5 9.5 12 5l4.5 4.5" /><path d="M5 15v4h14v-4" /></>,
  folder: <path d="M3 7.5V18a1 1 0 0 0 1 1h16a1 1 0 0 0 1-1V9a1 1 0 0 0-1-1h-8L10 6H4a1 1 0 0 0-1 1.5z" />,
  server: <><rect x="4" y="4" width="16" height="7" rx="2" /><rect x="4" y="13" width="16" height="7" rx="2" /><path d="M8 7.5h.01M8 16.5h.01" /></>,
  chat: <><path d="M4 5h16v11H9l-5 4z" /><path d="M9.5 9a2.5 2.5 0 1 1 3.3 2.4c-.5.2-.8.5-.8 1" /></>,
  image: <><rect x="3.5" y="4.5" width="17" height="15" rx="2" /><circle cx="9" cy="10" r="1.6" /><path d="m20 16-4.5-4.5L7 19.5" /></>,
  mix: <><circle cx="12" cy="12" r="8.5" /><path d="M12 3.5V12l6 4.5" /></>,
  repeat: <><path d="M17 3l3 3-3 3" /><path d="M4 11V9a3 3 0 0 1 3-3h13M7 21l-3-3 3-3" /><path d="M20 13v2a3 3 0 0 1-3 3H4" /></>,
  shield: <><path d="M12 3 5 6v6c0 4.5 3 7.5 7 9 4-1.5 7-4.5 7-9V6z" /><path d="m9.5 9.5 5 5M14.5 9.5l-5 5" /></>,
  globe: <><circle cx="12" cy="12" r="8.5" /><path d="M3.5 12h17M12 3.5c2.5 2.6 3.5 5.5 3.5 8.5s-1 5.9-3.5 8.5c-2.5-2.6-3.5-5.5-3.5-8.5s1-5.9 3.5-8.5z" /></>,
  gear: <><circle cx="12" cy="12" r="3" /><path d="M19.4 15a1.7 1.7 0 0 0 .3 1.8l.1.1a2 2 0 1 1-2.8 2.8l-.1-.1a1.7 1.7 0 0 0-1.8-.3 1.7 1.7 0 0 0-1 1.5V21a2 2 0 1 1-4 0v-.1a1.7 1.7 0 0 0-1.1-1.5 1.7 1.7 0 0 0-1.8.3l-.1.1a2 2 0 1 1-2.8-2.8l.1-.1a1.7 1.7 0 0 0 .3-1.8 1.7 1.7 0 0 0-1.5-1H3a2 2 0 1 1 0-4h.1a1.7 1.7 0 0 0 1.5-1.1 1.7 1.7 0 0 0-.3-1.8l-.1-.1a2 2 0 1 1 2.8-2.8l.1.1a1.7 1.7 0 0 0 1.8.3H9a1.7 1.7 0 0 0 1-1.5V3a2 2 0 1 1 4 0v.1a1.7 1.7 0 0 0 1 1.5 1.7 1.7 0 0 0 1.8-.3l.1-.1a2 2 0 1 1 2.8 2.8l-.1.1a1.7 1.7 0 0 0-.3 1.8V9a1.7 1.7 0 0 0 1.5 1H21a2 2 0 1 1 0 4h-.1a1.7 1.7 0 0 0-1.5 1z" /></>,
  play: <path d="M7 4.5v15l12-7.5z" />,
  eye: <><path d="M2.5 12S6 5.5 12 5.5 21.5 12 21.5 12 18 18.5 12 18.5 2.5 12 2.5 12z" /><circle cx="12" cy="12" r="2.8" /></>,
  check: <path d="m5 12.5 4.5 4.5L19 7.5" />,
  x: <path d="M6 6l12 12M18 6 6 18" />,
  plus: <path d="M12 5v14M5 12h14" />,
  trash: <><path d="M4.5 7h15M9.5 7V4.5h5V7M6.5 7l1 13h9l1-13" /></>,
  edit: <><path d="M4 20h4L19 9l-4-4L4 16z" /><path d="m13.5 6.5 4 4" /></>,
  refresh: <><path d="M20 11a8 8 0 1 0-2.3 5.7" /><path d="M20 4v7h-7" /></>,
  download: <><path d="M12 4v11M7.5 10.5 12 15l4.5-4.5" /><path d="M5 19h14" /></>,
  database: <><ellipse cx="12" cy="6" rx="7.5" ry="3" /><path d="M4.5 6v12c0 1.7 3.4 3 7.5 3s7.5-1.3 7.5-3V6M4.5 12c0 1.7 3.4 3 7.5 3s7.5-1.3 7.5-3" /></>,
  chart: <><path d="M4 20V4M4 20h16" /><path d="M8 16v-4M12 16V8M16 16v-6" /></>,
  results: <><rect x="5" y="4.5" width="14" height="16.5" rx="2.5" /><path d="M9 4.5V3.5a1 1 0 0 1 1-1h4a1 1 0 0 1 1 1v1" /><path d="m8.8 12.3 2.2 2.2 4.2-4.4" /><path d="M9 17.8h6" /></>,
  alert: <><path d="M12 3.5 2.5 20h19z" /><path d="M12 10v4.5M12 17.5h.01" /></>,
  search: <><circle cx="11" cy="11" r="6.5" /><path d="m20 20-4.2-4.2" /></>,
  list: <path d="M8 6h12M8 12h12M8 18h12M4 6h.01M4 12h.01M4 18h.01" />,
  spark: <><path d="M12 3v4M12 17v4M3 12h4M17 12h4" /><path d="m6 6 2.5 2.5M15.5 15.5 18 18M18 6l-2.5 2.5M8.5 15.5 6 18" /></>,
  layers: <><path d="m12 3 9 5-9 5-9-5z" /><path d="m3 13 9 5 9-5" /></>,
  arrow: <path d="M5 12h14M13 6l6 6-6 6" />,
  stop: <rect x="6" y="6" width="12" height="12" rx="2" />,
  chevron: <path d="m9 6 6 6-6 6" />,
  back: <path d="M19 12H5M11 6l-6 6 6 6" />,
  clock: <><circle cx="12" cy="12" r="8.5" /><path d="M12 7.5V12l3 2" /></>,
  book: <><path d="M4 5.5A2.5 2.5 0 0 1 6.5 3H20v15H6.5A2.5 2.5 0 0 0 4 20.5z" /><path d="M4 20.5A2.5 2.5 0 0 0 6.5 23H20v-5" /></>,
};

export function Icon({ name, size = 16, stroke = 1.8, className = "" }) {
  return (
    <svg className={`ico ${className}`} width={size} height={size} viewBox="0 0 24 24" fill={name === "play" || name === "stop" ? "currentColor" : "none"}
      stroke="currentColor" strokeWidth={stroke} strokeLinecap="round" strokeLinejoin="round" aria-hidden="true">{P[name]}</svg>
  );
}

export function Switch({ on, onChange, label, disabled }) {
  return (
    <button type="button" className={`switch${on ? " on" : ""}`} role="switch" aria-checked={!!on} aria-label={label}
      disabled={disabled} onClick={() => onChange(!on)}><span /></button>
  );
}

export function Segmented({ value, options, onChange, small }) {
  return (
    <div className={`seg${small ? " small" : ""}`} role="radiogroup">
      {options.map(([v, l, hint]) => (
        <button type="button" key={String(v)} role="radio" aria-checked={value === v} className={value === v ? "on" : ""}
          title={hint || undefined} onClick={() => onChange(v)}>{l}</button>
      ))}
    </div>
  );
}

export function Chips({ options, value, onToggle, onRemove }) {
  return (
    <div className="chips">
      {options.map(([k, l, extra]) => {
        const on = value.includes(k);
        return (
          <span key={k} role="checkbox" aria-checked={on} tabIndex={0} className={`chip${on ? " on" : ""}`}
            onClick={() => onToggle(k)} onKeyDown={(e) => (e.key === " " || e.key === "Enter") && (e.preventDefault(), onToggle(k))}>
            {on && <Icon name="check" size={12} stroke={2.4} />}{l}{extra}
            {onRemove && onRemove(k) && (
              <button type="button" className="chip-x" aria-label={`Remove ${l}`} onClick={(e) => { e.stopPropagation(); onRemove(k)(); }}>
                <Icon name="x" size={11} stroke={2.4} />
              </button>
            )}
          </span>
        );
      })}
    </div>
  );
}

export function Field({ label, hint, children, className = "" }) {
  return (
    // a div, not a label: a label would forward clicks to the first button of a segmented control
    <div className={`field ${className}`} role="group" aria-label={label}>
      <span className="field-l">{label}</span>
      {children}
      {hint && <span className="field-h">{hint}</span>}
    </div>
  );
}

export function NumberInput({ value, onChange, step = 1, min, max, ...rest }) {
  return <input type="number" value={value} step={step} min={min} max={max} onChange={(e) => onChange(e.target.value === "" ? "" : Number(e.target.value))} {...rest} />;
}

function useEscape(onClose, active) {
  useEffect(() => {
    if (!active) return undefined;
    const h = (e) => e.key === "Escape" && onClose();
    window.addEventListener("keydown", h);
    return () => window.removeEventListener("keydown", h);
  }, [onClose, active]);
}

export function Modal({ open, title, subtitle, onClose, children, footer, wide }) {
  useEscape(onClose, open);
  if (!open) return null;
  // rendered into <body>, so a parent with a transform or filter (page and step animations) can never trap it
  return createPortal(
    <div className="overlay" onMouseDown={(e) => e.target === e.currentTarget && onClose()}>
      <div className={`modal${wide ? " wide" : ""}`} role="dialog" aria-modal="true" aria-label={title}>
        <div className="modal-hd">
          <div><h2>{title}</h2>{subtitle && <p>{subtitle}</p>}</div>
          <button type="button" className="icon-btn" onClick={onClose} aria-label="Close"><Icon name="x" /></button>
        </div>
        <div className="modal-bd">{children}</div>
        {footer && <div className="modal-ft">{footer}</div>}
      </div>
    </div>,
    document.body,
  );
}

export function Drawer({ open, title, onClose, children, tabs }) {
  useEscape(onClose, open);
  if (!open) return null;
  return (
    <div className="overlay drawer-wrap" onMouseDown={(e) => e.target === e.currentTarget && onClose()}>
      <aside className="drawer" role="dialog" aria-modal="true" aria-label={title}>
        <div className="drawer-hd"><h2>{title}</h2>{tabs}<div className="grow" />
          <button type="button" className="btn ghost" onClick={onClose}><Icon name="x" /> Close</button></div>
        <div className="drawer-bd">{children}</div>
      </aside>
    </div>
  );
}

export function Empty({ icon = "list", title, children }) {
  return <div className="empty"><span className="empty-ico"><Icon name={icon} size={22} /></span><b>{title}</b>{children && <span>{children}</span>}</div>;
}

// ------------------------------------------------------------------ toasts
const ToastCtx = createContext(() => {});
export const useToast = () => useContext(ToastCtx);
export function ToastProvider({ children }) {
  const [items, setItems] = useState([]);
  const id = useRef(0);
  const push = useCallback((text, kind = "info") => {
    const k = ++id.current;
    setItems((x) => [...x, { k, text, kind }]);
    setTimeout(() => setItems((x) => x.filter((t) => t.k !== k)), 4200);
  }, []);
  return (
    <ToastCtx.Provider value={push}>
      {children}
      <div className="toasts">{items.map((t) => <div key={t.k} className={`toast ${t.kind}`}><Icon name={t.kind === "error" ? "alert" : "check"} />{t.text}</div>)}</div>
    </ToastCtx.Provider>
  );
}

export function useBusy() {
  const [busy, setBusy] = useState(false);
  const run = useCallback(async (fn) => { setBusy(true); try { return await fn(); } finally { setBusy(false); } }, []);
  return [busy, run];
}















//GeneratePage.jsx
// Generate Dataset page: a guided, step-by-step setup with a live summary, and a progress view once a job runs.
// Same API as the original UI (/api/datagen); the form <-> JobRequest logic is in ../model.js.
import { useCallback, useEffect, useLayoutEffect, useRef, useState } from "react";
import { useStepper } from "../stepper.js";
import { API, api, post, upload } from "../api.js";
import { QA_TEMPLATE, T_FORMATS, T_INPUTS, T_OUTPUTS, buildRequest, defaultForm, endpointsForTest, formFromRequest, isQa, visionUsed } from "../model.js";
import { Icon, useToast } from "../ui.jsx";
import { emit, useBus } from "../bus.js";
import { AdvancedStep, DatasetStep, DocumentsStep, MixStep, ModelsStep } from "./steps.jsx";
import Summary from "./Summary.jsx";
import RunProgress from "./RunProgress.jsx";
import "./generate.css";

export const STEPS = [
  { key: "documents", title: "Documents", icon: "doc", lead: "Upload the files the model should learn from, or point to a folder on the server." },
  { key: "models", title: "Models", icon: "server", lead: "The teacher model writes and judges the Q&A pairs; the vision model reads images. Both must already be running." },
  { key: "questions", title: "Dataset & images", icon: "layers", lead: "Choose what is generated from each part of your documents, how images are read, and how the dataset is split." },
  { key: "mix", title: "Training mix", icon: "mix", lead: "Add extra rows next to the document Q&A, so the model stays helpful and well-behaved." },
  { key: "advanced", title: "Advanced", icon: "gear", lead: "Everything here has sensible defaults. Change only what you need." },
];

const jobFromUrl = () => new URL(location.href).searchParams.get("job");

export default function GeneratePage({ openResults }) {
  const toast = useToast();
  const [defaults, setDefaults] = useState(null);
  // dataset templates: Q&A until GET /templates answers (an older server has Q&A only)
  const [tpl, setTpl] = useState({ templates: [QA_TEMPLATE], inputs: T_INPUTS, outputs: T_OUTPUTS, formats: T_FORMATS });
  const loadTemplates = useCallback(async () => {
    try { const r = await api("/templates"); if (r?.templates?.length) setTpl(r); } catch { /* older server: Q&A only */ }
  }, []);
  useEffect(() => { loadTemplates(); }, [loadTemplates]);
  const [form, setForm] = useState(() => defaultForm(null));
  const [files, setFiles] = useState([]);
  const [step, setStep] = useState(0);
  const [ep, setEp] = useState({ state: "idle" });
  const [models, setModels] = useState({ teacher: [], vision: [] });
  const [jobId, setJobId] = useState(jobFromUrl);
  // remember the previous step, so the step content slides in from the right direction
  const prevStep = useRef(0);
  const dir = step >= prevStep.current ? "fwd" : "back";
  useEffect(() => { prevStep.current = step; }, [step]);
  // the soft card behind the active step and the progress line are measured from the real step buttons
  const [stepperRef, ind] = useStepper();
  const [job, setJob] = useState(null);
  const [progress, setProgress] = useState({});
  const [starting, setStarting] = useState(false);
  const [startErr, setStartErr] = useState("");
  const formRef = useRef(form);
  formRef.current = form;
  const set = (k, v) => setForm((f) => ({ ...f, [k]: v }));
  const setIn = (group, k, v) => setForm((f) => ({ ...f, [group]: { ...f[group], [k]: v } }));

  // ---- start-up: defaults, and follow a job that is already running
  useEffect(() => {
    (async () => {
      let d = null;
      try { d = await api("/defaults"); } catch (e) { toast(`API not reachable: ${e.message}`, "error"); }
      setDefaults(d || { input_roots: [] });
      setForm(defaultForm(d));
      if (!jobFromUrl()) {
        try {
          const list = await api("/jobs");
          const active = list.find((j) => j.status === "queued" || j.status === "running");
          if (active) setJobId(active.id);
        } catch { /* the toast above already says the API is down */ }
      }
    })();
  }, []); // eslint-disable-line

  // keep ?job= in the address bar so a refresh comes back to the same job
  useEffect(() => {
    const u = new URL(location.href);
    jobId ? u.searchParams.set("job", jobId) : u.searchParams.delete("job");
    history.replaceState(null, "", u);
  }, [jobId]);

  // ---- the selected job and its live progress (server-sent events)
  const refreshJob = useCallback(async (id) => {
    if (!id) return;
    try { setJob(await api(`/jobs/${id}`)); } catch (e) { if (e.status === 404) setJobId(null); }
  }, []);
  useEffect(() => { setJob(null); setProgress({}); if (jobId) refreshJob(jobId); }, [jobId, refreshJob]);
  useEffect(() => {
    if (!jobId) return undefined;
    const es = new EventSource(`${API}/jobs/${encodeURIComponent(jobId)}/events`);
    let t;
    es.onmessage = (m) => {
      const ev = JSON.parse(m.data);
      if (ev.type === "end") { es.close(); refreshJob(jobId); return; }
      if (ev.type === "step_progress") setProgress((p) => ({ ...p, [ev.step]: { done: ev.done, total: ev.total } }));
      else { clearTimeout(t); t = setTimeout(() => refreshJob(jobId), 250); }
    };
    return () => { clearTimeout(t); es.close(); };
  }, [jobId, refreshJob]);

  // ---- endpoints: test now, and again whenever an endpoint setting changes
  const testSeq = useRef(0);
  const testEndpoints = useCallback(async (f = formRef.current) => {
    if (!f.teacherUrl.trim()) { setEp({ state: "done", message: "Enter the teacher URL.", bad: true }); return; }
    const seq = ++testSeq.current;
    setEp((e) => ({ ...e, state: "testing" }));
    try {
      const res = await post("/endpoints/test", { endpoints: endpointsForTest(f) });
      if (seq !== testSeq.current) return;
      const t = res.find((x) => x.role === "teacher"), v = res.find((x) => x.role === "vision");
      setModels({ teacher: t?.models || [], vision: v?.models || [] });
      const pick = (list, re) => list.find((m) => re.test(m)) || list[0];
      setForm((cur) => {
        const next = { ...cur };
        if (t?.ok && t.models.length && !t.models.includes(cur.teacherModel)) next.teacherModel = pick(t.models, /^(?!.*(vl|vision)).*/i);
        if (v?.ok && v.models.length && !v.models.includes(cur.visionModel)) next.visionModel = pick(v.models, /vl|vision/i);
        return next;
      });
      const why = (x, role) => (x.ok ? (x.models.length ? "" : `${role}: no models served`) : `${role}: ${x.error || "not reachable"}`);
      const problems = [why(t, "Teacher"), v ? why(v, "Vision") : visionUsed(f) ? "Vision: enter the URL, or turn vision analysis off" : ""].filter(Boolean);
      setEp({ state: "done", teacher: t, vision: v, message: problems.join(" · "), bad: problems.length > 0 });
    } catch (e) {
      if (seq === testSeq.current) setEp({ state: "done", message: `Test failed: ${e.message}`, bad: true });
    }
  }, []);
  const epKey = [form.teacherUrl, form.visionUrl, form.apiKey, visionUsed(form)].join("|");
  const firstTest = useRef(true);
  useEffect(() => {
    if (!defaults) return undefined;
    const t = setTimeout(() => testEndpoints(), firstTest.current ? 0 : 700);
    firstTest.current = false;
    return () => clearTimeout(t);
  }, [epKey, defaults, testEndpoints]);

  // ---- what is missing before a job can start
  const folders = form.folders.split(/\n/).map((s) => s.trim()).filter(Boolean);
  const issues = [];
  if (!files.length && !folders.length) issues.push({ step: 0, text: "Add documents: upload files or enter a server folder." });
  try { buildRequest(form, ["check"]); } catch (e) {
    const s = /endpoint|vision/i.test(e.message) ? 1 : /question types|split|template|input|output|label|json|variety|checks/i.test(e.message) ? 2 : /decline|paraphrase|general/i.test(e.message) ? 3 : 4;
    issues.push({ step: s, text: e.message });
  }

  // ---- actions
  const startJob = async (preview) => {
    setStartErr("");
    if (issues.length) { setStep(issues[0].step); setStartErr(issues[0].text); return; }
    setStarting(true);
    try {
      const list = [...folders];
      if (files.length) list.unshift((await upload(files)).folder);
      const res = await post(`/jobs${preview ? "?preview=3" : ""}`, buildRequest(form, list));
      setFiles([]);
      toast(`Job ${res.job_id} started${preview ? " (preview on 3 chunks)" : ""}`);
      setJobId(res.job_id);
    } catch (e) {
      setStartErr(e.message);
    } finally { setStarting(false); }
  };
  const cancelJob = async () => { await post(`/jobs/${jobId}/cancel`); toast("Cancel requested: the job stops after the current item"); };
  const retryJob = async () => { await post(`/jobs/${jobId}/retry`); toast("Job re-queued: finished steps are skipped"); setProgress({}); refreshJob(jobId); };
  const newJob = () => { setJobId(null); setStep(0); setStartErr(""); };
  const reuseSettings = () => { if (job) { setForm(formFromRequest(job.request, defaults)); toast(`Settings of ${job.job_id} loaded`); } newJob(); };
  const resetForm = () => { setForm(defaultForm(defaults)); toast("Settings reset to the defaults"); };

  // ---- tell the sidebar and header what is going on, and listen to them
  const vUsed = visionUsed(form);
  useEffect(() => { emit("ep", { ...ep, teacherModel: form.teacherModel, visionModel: vUsed ? form.visionModel : "" }); }, [ep, form.teacherModel, form.visionModel, vUsed]);
  useEffect(() => { emit("job", { id: jobId, name: job?.name || job?.request?.name, status: job?.status, step, ready: issues.length === 0 }); });
  useEffect(() => { emit("jobs-changed"); }, [jobId, job?.status]);
  useBus("open-job", useCallback((id) => { setStartErr(""); setJobId(id); }, []));
  useBus("new-job", useCallback(() => { setJobId(null); setStep(0); setStartErr(""); }, []));
  useBus("retest", useCallback(() => testEndpoints(), [testEndpoints]));

  if (jobId) {
    return <RunProgress job={job} jobId={jobId} progress={progress} onCancel={cancelJob} onRetry={retryJob}
      onNew={newJob} onReuse={reuseSettings} openResults={openResults} />;
  }

  const stepProps = { form, set, setIn, setForm, files, setFiles, defaults, ep, models, testEndpoints, onReset: resetForm, tpl, reloadTemplates: loadTemplates, goStep: setStep };
  const cur = STEPS[step];
  const status = stepStatus(form, files, folders, ep);
  return (
    <div className="gen">
      <nav className="stepper" aria-label="Setup steps" ref={stepperRef} style={{ "--p": step / (STEPS.length - 1) }}>
        <span className="stp-progress" aria-hidden="true"><i style={ind ? { width: ind.end } : undefined} /></span>
        {ind && <span className="stp-ind" aria-hidden="true" style={{ transform: `translate(${ind.left}px, ${ind.top}px)`, width: ind.width, height: ind.height }} />}
        {STEPS.map((s, i) => {
          const st = status[s.key];
          return (
            <button type="button" key={s.key} className={`stp${i === step ? " on" : ""}${st.ok ? " ok" : ""}${st.warn ? " attn" : ""}`} onClick={() => setStep(i)}>
              <span className="stp-n">{st.ok && i !== step ? <Icon name="check" size={14} stroke={2.6} className="pop" /> : <span className="pop">{i + 1}</span>}</span>
              <span className="stp-t"><b>{s.title}</b><small>{st.note}</small></span>
            </button>
          );
        })}
      </nav>

      <div className="gen-grid">
        <section className="card gen-main">
          <header className={`gen-hd anim-${dir}`} key={`hd${step}`}>
            <span className="gen-hd-ico"><Icon name={cur.icon} size={20} /></span>
            <div className="gen-hd-t">
              <div className="gen-hd-meta">
                <span className="step-n">Step {step + 1} of {STEPS.length}</span>
                <span className="step-dots" aria-hidden="true">{STEPS.map((s2, i) => <i key={s2.key} className={i < step ? "done" : i === step ? "on" : ""} />)}</span>
              </div>
              <h2>{cur.title}</h2><p>{cur.lead}</p>
            </div>
          </header>
          <div className={`gen-bd anim-${dir}`} key={`bd${step}`}>
            {cur.key === "documents" && <DocumentsStep {...stepProps} />}
            {cur.key === "models" && <ModelsStep {...stepProps} />}
            {cur.key === "questions" && <DatasetStep {...stepProps} />}
            {cur.key === "mix" && <MixStep {...stepProps} />}
            {cur.key === "advanced" && <AdvancedStep {...stepProps} />}
          </div>
          <footer className="gen-ft">
            <button type="button" className="btn ghost" disabled={step === 0} onClick={() => setStep(step - 1)}><Icon name="back" size={15} />Back</button>
            <div className="grow" />
            {step < STEPS.length - 1
              ? <button type="button" className="btn dark" onClick={() => setStep(step + 1)}>Next: {STEPS[step + 1].title}<Icon name="arrow" size={15} /></button>
              : <button type="button" className="btn primary" disabled={starting} onClick={() => startJob(false)}><Icon name="play" size={13} />{starting ? "Starting…" : "Start job"}</button>}
          </footer>
        </section>

        <Summary form={form} files={files} folders={folders} ep={ep} issues={issues} starting={starting} startErr={startErr}
          onStart={startJob} goStep={setStep} />
      </div>
    </div>
  );
}

function stepStatus(form, files, folders, ep) {
  const n = files.length + folders.length;
  const split = form.split.reduce((a, b) => a + (Number(b) || 0), 0);
  const extras = [form.para.enabled, form.decline.enabled, form.general.enabled].filter(Boolean).length;
  const epOk = ep.state === "done" && !ep.bad;
  return {
    documents: n ? { ok: true, note: [files.length && `${files.length} file${files.length > 1 ? "s" : ""}`, folders.length && `${folders.length} folder${folders.length > 1 ? "s" : ""}`].filter(Boolean).join(" · ") } : { note: "Add your files" },
    models: ep.state === "testing" ? { note: "Testing…" } : epOk ? { ok: true, note: "Endpoints online" } : { warn: ep.state === "done", note: ep.state === "done" ? "Check the endpoints" : "Not tested yet" },
    questions: (() => {
      const needTypes = form.template?.input === "question" && !form.types.length;
      if (needTypes || split !== 100) return { warn: true, note: needTypes ? "Pick a question type" : "Split must total 100" };
      return { ok: true, note: isQa(form.template) ? `Q&A · ${form.types.length} type${form.types.length > 1 ? "s" : ""}` : form.template.name };
    })(),
    mix: { ok: true, note: extras ? `${extras} extra${extras > 1 ? "s" : ""} on` : "Document Q&A only" },
    advanced: { ok: false, note: "Optional" },
  };
}




























//steps.jsx
// The five setup steps of the Generate Dataset page.
import { useEffect, useRef, useState } from "react";
import { api, post } from "../api.js";
import {
  D_PRESETS, G_TOPICS, MIX, P_STYLES, cleanTemplate, fmtShort, host, inShort, isQa, mixEstimate, mixWarnings, outShort, sameTemplate, short, slug,
  templateError, visionUsed,
} from "../model.js";
import { Chips, Field, Icon, Modal, NumberInput, Segmented, Switch, useToast } from "../ui.jsx";

const ext = (n) => { const e = (n.split(".").pop() || "").toLowerCase(); return e === "htm" ? "html" : e === "markdown" ? "md" : e; };
const toggle = (list, k) => (list.includes(k) ? list.filter((x) => x !== k) : [...list, k]);
const mb = (b) => (b / 1048576 < 0.1 ? `${Math.max(1, Math.round(b / 1024))} KB` : `${(b / 1048576).toFixed(1)} MB`);

// ------------------------------------------------------------------ small building blocks
function Section({ title, hint, right, children }) {
  return (
    <section className="g-sec">
      <div className="g-sec-hd"><div><h3>{title}</h3>{hint && <p>{hint}</p>}</div>{right}</div>
      {children}
    </section>
  );
}

// a large selectable card: radio (single) or checkbox (multi) behaviour
function Choice({ on, onClick, icon, title, text, multi }) {
  return (
    <button type="button" role={multi ? "checkbox" : "radio"} aria-checked={on} className={`choice${on ? " on" : ""}`} onClick={onClick}>
      <span className="choice-top">
        {icon && <span className="choice-ico"><Icon name={icon} size={18} /></span>}
        <span className={`choice-mark${multi ? " sq" : ""}`}>{on && <Icon name="check" size={12} stroke={3} />}</span>
      </span>
      <b>{title}</b>
      {text && <small>{text}</small>}
    </button>
  );
}

function Toggle({ icon, title, text, on, onChange, children }) {
  return (
    <div className={`opt${on ? " on" : ""}`}>
      <div className="opt-top">
        <span className="opt-ico"><Icon name={icon} size={18} /></span>
        <div className="opt-t"><b>{title}</b><small>{text}</small></div>
        <Switch on={on} onChange={onChange} label={title} />
      </div>
      {on && children && <div className="opt-bd">{children}</div>}
    </div>
  );
}

function StatusBadge({ s, used = true, testing }) {
  let cls = "ok", text = "";
  if (!used) [cls, text] = ["muted", "Not used"];
  else if (testing) [cls, text] = ["run", "Testing…"];
  else if (!s) [cls, text] = ["muted", "Not tested"];
  else if (!s.ok) [cls, text] = ["bad", "Offline"];
  else if (!s.models?.length) [cls, text] = ["bad", "No models"];
  else text = `Online · ${s.latency_ms} ms`;
  return <span className={`badge ${cls}`}><i />{text}</span>;
}

export function MixBar({ mix }) {
  const parts = Object.entries(mix).filter(([, v]) => v > 0);
  const tot = parts.reduce((a, [, v]) => a + v, 0) || 1;
  return (
    <div className="mix">
      <div className="mixbar">{parts.map(([k, v]) => <i key={k} style={{ width: `${(100 * v) / tot}%`, background: MIX[k][1] }} title={`${MIX[k][0]} ${Math.round((100 * v) / tot)}%`} />)}</div>
      <div className="mixleg">{parts.map(([k, v]) => <span key={k} style={{ "--c": MIX[k][1] }}>{MIX[k][0]} <b>{Math.round((100 * v) / tot)}%</b></span>)}</div>
    </div>
  );
}

// ------------------------------------------------------------------ 1. documents
export function DocumentsStep({ form, set, files, setFiles, defaults }) {
  const input = useRef(null);
  const [drag, setDrag] = useState(false);
  const [showFolders, setShowFolders] = useState(!!form.folders.trim());
  // copy the FileList now: it is live, and clearing the input empties it
  const add = (list) => {
    const picked = Array.from(list || []);
    if (picked.length) setFiles((cur) => [...cur, ...picked.filter((f) => !cur.some((x) => x.name === f.name && x.size === f.size))]);
  };
  const roots = defaults?.input_roots || [];
  const total = files.reduce((a, f) => a + f.size, 0);
  return (
    <>
      <Section title="Upload files" hint="They are uploaded when the job starts."
        right={files.length > 0 && <span className="muted small">{files.length} file{files.length > 1 ? "s" : ""} · {mb(total)}</span>}>
        <div className={`dropzone${drag ? " drag" : ""}`} role="button" tabIndex={0} onClick={() => input.current.click()}
          onKeyDown={(e) => (e.key === "Enter" || e.key === " ") && (e.preventDefault(), input.current.click())}
          onDragOver={(e) => { e.preventDefault(); setDrag(true); }} onDragLeave={() => setDrag(false)}
          onDrop={(e) => { e.preventDefault(); setDrag(false); add(e.dataTransfer.files); }}>
          <span className="dz-art" aria-hidden="true">
            <span className="dz-sheet s1"><i /><i /><i /></span>
            <span className="dz-sheet s2"><i /><i /><i /></span>
            <span className="dz-sheet s3"><b>PDF</b><i /><i /><i /></span>
            <span className="dz-badge"><Icon name="upload" size={16} stroke={2.2} /></span>
          </span>
          <b>Drop files here, or <u>browse</u></b>
          <span className="dz-types">{["PDF", "DOCX", "TXT", "Markdown", "HTML"].map((t) => <span key={t}>{t}</span>)}</span>
          <input ref={input} type="file" multiple hidden accept=".pdf,.docx,.txt,.md,.markdown,.html,.htm" onChange={(e) => { add(e.target.files); e.target.value = ""; }} />
        </div>
        {files.length > 0 && (
          <div className="filegrid">
            {files.map((f, i) => (
              <div className="fcard" key={f.name + f.size}>
                <span className={`ext ${ext(f.name)}`}>{ext(f.name).toUpperCase()}</span>
                <span className="fcard-n" title={f.name}>{f.name}<small>{mb(f.size)}</small></span>
                <button type="button" className="icon-btn" aria-label={`Remove ${f.name}`} onClick={() => setFiles((x) => x.filter((_, k) => k !== i))}><Icon name="x" size={14} /></button>
              </div>
            ))}
          </div>
        )}
        {!showFolders
          ? <button type="button" className="link" onClick={() => setShowFolders(true)}><Icon name="folder" size={14} />Or use folders that are already on the server</button>
          : <Field label="Folders on the server (one per line)" hint={roots.length ? `Allowed: ${roots.join(" · ")}` : ""}>
            <textarea className="mono" rows={2} value={form.folders} onChange={(e) => set("folders", e.target.value)}
              placeholder={roots[0] ? `${roots[0]}${roots[0].includes("\\") ? "\\" : "/"}my_documents` : "/data/documents/my_documents"} />
          </Field>}
      </Section>

      <Section title="Dataset">
        <div className="grid2">
          <Field label="Dataset name" hint="Used for the job and the output files."><input value={form.name} onChange={(e) => set("name", e.target.value)} placeholder="e.g. hr_policies" /></Field>
        </div>
        <div className="choices two" role="radiogroup" aria-label="Dataset for">
          <Choice on={form.mode === "chat"} onClick={() => set("mode", "chat")} icon="chat" title="Chat fine-tuning" text="Question → answer. The usual choice for assistants." />
          <Choice on={form.mode === "reasoning"} onClick={() => set("mode", "reasoning")} icon="spark" title="Reasoning fine-tuning" text="Question → thinking trace → answer, for reasoning models." />
        </div>
      </Section>
    </>
  );
}

// ------------------------------------------------------------------ 2. models
export function ModelsStep({ form, set, ep, models, testEndpoints, goStep }) {
  const vUsed = visionUsed(form);
  const opts = (list, cur) => [...new Set([cur, ...(list || [])].filter(Boolean))];
  const testing = ep.state === "testing";
  return (
    <>
      <div className="epcards">
        <div className="epcard">
          <div className="epcard-hd">
            <span className="opt-ico on"><Icon name="server" size={18} /></span>
            <div className="opt-t"><b>Teacher model</b><small>Writes the questions and answers, then judges them</small></div>
          </div>
          <Field label="Endpoint URL"><input className="mono" value={form.teacherUrl} placeholder="http://gpu-host:8001/v1" onChange={(e) => set("teacherUrl", e.target.value)} /></Field>
          <Field label="Model"><select value={form.teacherModel} onChange={(e) => set("teacherModel", e.target.value)}>
            {opts(models.teacher, form.teacherModel).map((m) => <option key={m}>{m}</option>)}</select></Field>
          <div className="epcard-ft"><StatusBadge s={ep.teacher} testing={testing} /></div>
        </div>

        <div className={`epcard${vUsed ? "" : " off"}`}>
          <div className="epcard-hd">
            <span className={`opt-ico${vUsed ? " on" : ""}`}><Icon name="image" size={18} /></span>
            <div className="opt-t"><b>Vision model</b><small>Describes images, charts and screenshots</small></div>
            <span className={`badge ${vUsed ? "ok" : "muted"}`}><i />{vUsed ? "In use" : "Not used"}</span>
          </div>
          {!form.readImages ? <p className="note"><Icon name="alert" size={14} />Images are ignored, so the vision model is not used.{" "}
            <button type="button" className="link" onClick={() => goStep?.(2)}>Change in Dataset &amp; images</button></p>
            : !form.vision ? <p className="note"><Icon name="alert" size={14} />Vision analysis is off: OCR only, faster.{" "}
              <button type="button" className="link" onClick={() => goStep?.(2)}>Change in Dataset &amp; images</button></p>
              : <>
                <Field label="Endpoint URL"><input className="mono" value={form.visionUrl} placeholder="http://gpu-host:8002/v1" onChange={(e) => set("visionUrl", e.target.value)} /></Field>
                <Field label="Model"><select value={form.visionModel} onChange={(e) => set("visionModel", e.target.value)}>
                  {opts(models.vision, form.visionModel).map((m) => <option key={m}>{m}</option>)}</select></Field>
                <div className="epcard-ft"><StatusBadge s={ep.vision} used={vUsed} testing={testing} /></div>
              </>}
        </div>
      </div>

      <Section title="Connection" hint="OpenAI-compatible endpoints (for example vLLM). The tool only calls them; it never starts or loads a model.">
        <div className="grid2">
          <Field label="API key (optional)" hint="Sent as a Bearer token to both endpoints"><input type="password" autoComplete="off" value={form.apiKey} placeholder="none" onChange={(e) => set("apiKey", e.target.value)} /></Field>
          <Field label="Parallel requests" hint="How many calls run at the same time"><NumberInput value={form.maxParallel} min={1} max={64} onChange={(v) => set("maxParallel", v)} /></Field>
        </div>
        <div className={`testbar${ep.bad ? " bad" : ep.state === "done" ? " good" : ""}`}>
          <Icon name={ep.bad ? "alert" : ep.state === "done" ? "check" : "server"} size={16} />
          <span>{testing ? "Testing the endpoints…" : ep.message || (ep.state === "done"
            ? `All good: the teacher serves ${short(form.teacherModel)} at ${host(form.teacherUrl)}${vUsed ? `, vision serves ${short(form.visionModel)}` : ""}.`
            : "Endpoints are tested automatically when you change them.")}</span>
          <div className="grow" />
          <button type="button" className="btn small" disabled={testing} onClick={() => testEndpoints()}><Icon name="refresh" size={13} />Test again</button>
        </div>
      </Section>
    </>
  );
}

// ------------------------------------------------------------------ 3. dataset & images
const QTYPES = [
  ["factual", "Factual", "Facts stated in the text", "list"],
  ["explanation", "Explanation", "Why and how things work", "book"],
  ["how-to", "How-to", "Step-by-step instructions", "layers"],
  ["from_images", "From images", "About charts, screenshots and diagrams", "image"],
  ["comparison", "Comparison", "Compare things across documents", "repeat"],
];
const SPLIT_COLORS = ["var(--split-1)", "var(--split-2)", "var(--split-3)"];

export function DatasetStep({ form, set, tpl, reloadTemplates }) {
  const total = form.split.reduce((a, b) => a + (Number(b) || 0), 0);
  const [editor, setEditor] = useState(null); // null | "edit" | "new"
  const t = form.template, qa = isQa(t);
  const lengthUsed = !["json", "label", "continuation"].includes(t.output);
  const outW = qa ? "Answers" : outShort(t);
  return (
    <>
      <Section title="Dataset template" hint="What is generated from each part of your documents, and how each training row is written."
        right={<button type="button" className="btn small" onClick={() => setEditor("new")}><Icon name="plus" size={13} />New template</button>}>
        <TemplateCards form={form} set={set} tpl={tpl} onEdit={() => setEditor("edit")} />
      </Section>

      <Section title="Images" hint="Pictures, charts and screenshots inside your documents.">
        <div className="choices two" role="radiogroup" aria-label="Images in documents">
          <Choice on={form.readImages} onClick={() => set("readImages", true)} icon="image" title="Read images" text="Charts, screenshots and diagrams are used too." />
          <Choice on={!form.readImages} onClick={() => set("readImages", false)} icon="doc" title="Ignore images" text="Only the text of the documents is used." />
        </div>
        {form.readImages && (
          <Toggle icon="eye" title="Vision analysis" on={form.vision} onChange={(v) => set("vision", v)}
            text={form.vision ? "The vision model reads and describes every image (set it up in Models)." : "Off: OCR only. Text printed in images is kept; faster."} />
        )}
      </Section>

      {t.input === "question" && (
        <Section title="Question types" hint="Pick one or more. The teacher writes a mix of them for every part of your documents.">
          <div className="choices five">
            {QTYPES.map(([k, l, tx, ic]) => (
              <Choice multi key={k} on={form.types.includes(k)} onClick={() => set("types", toggle(form.types, k))} icon={ic} title={l} text={tx} />
            ))}
          </div>
        </Section>
      )}

      <Section title={outW}>
        {lengthUsed ? (
          <div className="choices two" role="radiogroup" aria-label={`${outW} length`}>
            <Choice on={form.answerLength === "short"} onClick={() => set("answerLength", "short")} title={qa ? "Short answers" : "Short"} text="One or two sentences, straight to the point." />
            <Choice on={form.answerLength === "detailed"} onClick={() => set("answerLength", "detailed")} title={qa ? "Detailed answers" : "Detailed"} text="Full explanations with the relevant context." />
          </div>
        ) : <p className="note"><Icon name="alert" size={14} />Output length: {t.output === "continuation" ? "the rest of the text" : "set by the template"}.</p>}
        <div className="grid2">
          <Field label={qa ? "Max Q&A pairs" : "Max rows"} hint={qa ? "Upper limit for document Q&A pairs." : "Upper limit for rows from the documents."}>
            <NumberInput value={form.maxPairs} min={1} onChange={(v) => set("maxPairs", v)} /></Field>
          <Field label="System prompt (optional)" hint="First message of every row. Use the same text when you serve the model.">
            <textarea rows={2} value={form.systemPrompt} onChange={(e) => set("systemPrompt", e.target.value)} placeholder="e.g. You are the HR assistant for Contoso." />
          </Field>
        </div>
      </Section>

      <Section title="Train / validation / test split" right={<span className={`badge ${total === 100 ? "ok" : "bad"}`}><i />{total === 100 ? "Totals 100%" : `Totals ${total}%, must be 100%`}</span>}>
        <div className="splitbar">{form.split.map((v, i) => (Number(v) > 0 ? <i key={i} style={{ flex: Number(v), background: SPLIT_COLORS[i] }} title={`${["Train", "Validation", "Test"][i]} ${v}%`}>{Number(v) >= 12 && <span>{["Train", "Validation", "Test"][i]} {v}%</span>}</i> : null))}</div>
        <div className="split4">
          {["Train %", "Validation %", "Test %"].map((l, i) => (
            <span className="inset" key={l}><small>{l}</small><NumberInput value={form.split[i]} min={0} max={100}
              onChange={(v) => set("split", form.split.map((x, k) => (k === i ? v : x)))} /></span>
          ))}
          <span className="inset"><small>Random seed</small><NumberInput value={form.seed} onChange={(v) => set("seed", v)} /></span>
        </div>
        <Field label="Split by" hint={form.splitBy === "fact" ? "Validation and test ask facts that were trained, in new words." : "Validation and test use whole sections that were never trained on."}>
          <Segmented value={form.splitBy} onChange={(v) => set("splitBy", v)} options={[["fact", "Fact"], ["section", "Section"]]} />
        </Field>
      </Section>

      <TemplateModal open={!!editor} mode={editor} onClose={() => setEditor(null)} form={form} set={set} tpl={tpl} reloadTemplates={reloadTemplates} />
    </>
  );
}

// ---- dataset templates: cards to pick one, and the editor
function Flow({ t }) {
  return <span className="tpl-flow"><span className="tag">{inShort(t)}</span><Icon name="arrow" size={12} /><span className="tag">{outShort(t)}</span><small>{fmtShort(t)}{t.context_in_row ? " · text in the row" : ""}</small></span>;
}

function TemplateCards({ form, set, tpl, onEdit }) {
  const t = form.template;
  const list = tpl.templates || [];
  const saved = list.find((x) => x.id === t.id);
  const changed = saved && !sameTemplate(saved, t);
  const shown = saved ? list : [...list, { ...t, unsaved: true }];
  return (
    <div className="tpl-cards" role="radiogroup" aria-label="Dataset template">
      {shown.map((x) => {
        const on = x.unsaved || x.id === t.id;
        const cur = on ? t : x;
        return (
          <div key={x.id + (x.unsaved ? "-new" : "")} role="radio" aria-checked={on} tabIndex={0} className={`tpl-card${on ? " on" : ""}`}
            onClick={() => !on && set("template", { ...x })} onKeyDown={(e) => (e.key === "Enter" || e.key === " ") && !on && (e.preventDefault(), set("template", { ...x }))}>
            <span className="tpl-card-top">
              <span className="choice-ico"><Icon name={isQa(cur) ? "chat" : "layers"} size={18} /></span>
              <span className={`tpl-kind${x.builtin ? "" : " own"}`}>{x.unsaved ? "Not saved" : x.builtin ? "Built-in" : "Yours"}{on && changed ? " · changed" : ""}</span>
              <span className="choice-mark">{on && <Icon name="check" size={12} stroke={3} />}</span>
            </span>
            <b>{cur.name}</b>
            {cur.description && <small className="tpl-card-d">{cur.description}</small>}
            <Flow t={cur} />
            {on && <button type="button" className="tpl-edit" onClick={(e) => { e.stopPropagation(); onEdit(); }}><Icon name="edit" size={13} />Edit</button>}
          </div>
        );
      })}
    </div>
  );
}

const asText = (list) => (list || []).join(", ");
const asList = (text) => [...new Set(String(text || "").split(",").map((x) => x.trim()).filter(Boolean))];
function toDraft(t) {
  const c = cleanTemplate(t);
  return { ...c, labelsText: asText(c.labels), fieldsText: asText(c.json_fields), variety: c.variety.map((v) => ({ name: v.name, text: asText(v.values) })) };
}
function fromDraft(d) {
  return cleanTemplate({ ...d, labels: asList(d.labelsText), json_fields: asList(d.fieldsText), variety: d.variety.map((v) => ({ name: v.name.trim(), values: asList(v.text) })) });
}

function TemplateModal({ open, mode, onClose, form, set, tpl, reloadTemplates }) {
  const toast = useToast();
  const [d, setD] = useState(null);
  const [busy, setBusy] = useState(false);
  useEffect(() => {
    if (!open) return;
    if (mode === "new") setD(toDraft({ id: "my-template", name: "My template", description: "", input: "instruction", output: "completion", format: "chat" }));
    else setD(toDraft(form.template));
  }, [open, mode]); // eslint-disable-line react-hooks/exhaustive-deps
  if (!open || !d) return null;
  const up = (k, v) => setD((x) => ({ ...x, [k]: v }));
  const upCheck = (k, v) => setD((x) => ({ ...x, checks: { ...x.checks, [k]: v } }));
  const t = fromDraft(d);
  const err = templateError(t);
  const list = tpl.templates || [];
  const isBuiltinId = list.some((x) => x.builtin && x.id === t.id);
  const isSaved = list.some((x) => !x.builtin && x.id === t.id);
  const textIn = ["text", "text_start"].includes(t.input);
  const vars = ["chunk", "heading", "doc_name", ...(textIn ? [] : ["n"]), ...t.variety.map((v) => v.name).filter(Boolean)];
  const outVars = ["input", "chunk", "heading", ...t.variety.map((v) => v.name).filter(Boolean)];
  const setInput = (v) => setD((x) => ({ ...x, input: v, output: v === "text_start" ? "continuation" : x.output === "continuation" ? "completion" : x.output,
    format: v === "text_start" && x.format === "chat" ? "completion" : x.format }));
  const setOutput = (v) => setD((x) => ({ ...x, output: v, input: v === "continuation" ? "text_start" : x.input === "text_start" ? "text" : x.input }));

  const use = () => { if (err) return; set("template", { ...t, builtin: isBuiltinId && sameTemplate(list.find((x) => x.id === t.id), t) }); onClose(); };
  const save = async () => {
    if (err) return;
    const body = isBuiltinId ? { ...t, id: slug(t.name) === t.id ? `${t.id}-copy` : slug(t.name) } : t;
    setBusy(true);
    try {
      const savedT = await post("/templates", body);
      await reloadTemplates();
      set("template", { ...savedT });
      toast(`Template "${savedT.name}" saved`);
      onClose();
    } catch (e) { toast(e.message, "error"); } finally { setBusy(false); }
  };
  const remove = async () => {
    setBusy(true);
    try {
      await api(`/templates/${encodeURIComponent(t.id)}`, { method: "DELETE" });
      await reloadTemplates();
      if (form.template.id === t.id) set("template", { ...(list.find((x) => x.id === "qa") || form.template) });
      toast(`Template "${t.name}" deleted`);
      onClose();
    } catch (e) { toast(e.message, "error"); } finally { setBusy(false); }
  };
  const opt = (obj) => Object.entries(obj || {}).map(([k, l]) => <option key={k} value={k}>{l}</option>);

  return (
    <Modal open={open} wide title={mode === "new" ? "New dataset template" : `Template: ${d.name}`} onClose={onClose}
      subtitle="What is generated from each part of your documents, how it is checked and how each training row is written."
      footer={<>
        {isSaved && mode !== "new" && <button type="button" className="btn ghost danger" onClick={remove} disabled={busy}><Icon name="trash" size={14} />Delete</button>}
        {err && <span className="tpl-err"><Icon name="alert" size={14} />{err}</span>}
        <div className="grow" />
        <button type="button" className="btn" onClick={save} disabled={!!err || busy} title={isBuiltinId ? "Built-in templates are saved as a copy" : ""}>
          <Icon name="download" size={14} />{isBuiltinId ? "Save as a copy" : isSaved ? "Save changes" : "Save template"}</button>
        <button type="button" className="btn primary" onClick={use} disabled={!!err}>Use for this job</button>
      </>}>
      <div className="tpl-form">
        <div className="tpl-main">
          <div className="grid2">
            <Field label="Name"><input value={d.name} onChange={(e) => { const v = e.target.value; setD((x) => ({ ...x, name: v, id: !isSaved && x.id === slug(x.name) ? slug(v) : x.id })); }} /></Field>
            <Field label="Id" hint="Used for the file name; letters, digits, - and _"><input className="mono" value={d.id} onChange={(e) => up("id", e.target.value.toLowerCase())} /></Field>
          </div>
          <Field label="Description (optional)"><input value={d.description} onChange={(e) => up("description", e.target.value)} placeholder="What this dataset teaches the model" /></Field>

          <div className="tpl-sec"><span className="tpl-n">1</span><b>Input</b><small>what the model is given</small></div>
          <div className="grid2">
            <Field label="Input"><select value={d.input} onChange={(e) => setInput(e.target.value)}>{opt(tpl.inputs)}</select></Field>
            <Field label="Put the source text into the row" hint={t.input === "text" ? "Always: the text is the input" : t.input === "text_start" ? "The first part of the text is the input" : "Yes teaches the model to answer from given context (RAG)"}>
              <Segmented value={textIn ? true : d.context_in_row} onChange={(v) => !textIn && up("context_in_row", v)} options={[[false, "No"], [true, "Yes"]]} />
            </Field>
          </div>
          {textIn && <Field label="Instruction" hint={`Shown with the text. You can use ${t.variety.map((v) => `{{ ${v.name} }}`).join(", ") || "variety fields"} here.`}>
            <input value={d.instruction} onChange={(e) => up("instruction", e.target.value)} placeholder={t.input === "text_start" ? "Continue the text." : "e.g. Summarise this section in 2-4 sentences."} /></Field>}
          {!textIn && <Field label={d.input === "custom" ? "Input prompt (required)" : "Extra guidance for the inputs (optional)"} hint={`Placeholders: ${vars.map((v) => `{{ ${v} }}`).join("  ")}`}>
            <textarea rows={3} value={d.input_prompt} onChange={(e) => up("input_prompt", e.target.value)}
              placeholder={d.input === "custom" ? "e.g. Write {{ n }} short customer emails asking about the topic of this passage, as a {{ persona }}." : "e.g. Focus on what a new employee would ask. Use the tone of a {{ persona }}."} /></Field>}

          <div className="tpl-sec"><span className="tpl-n">2</span><b>Output</b><small>what the model learns to produce</small></div>
          <div className="grid2">
            <Field label="Output"><select value={d.output} onChange={(e) => setOutput(e.target.value)}>{opt(tpl.outputs)}</select></Field>
            <Field label="Row format"><select value={d.format} onChange={(e) => up("format", e.target.value)}>{opt(tpl.formats)}</select></Field>
          </div>
          {d.output === "label" && <Field label="Labels (comma separated)" hint="The output must be exactly one of these"><input value={d.labelsText} onChange={(e) => up("labelsText", e.target.value)} placeholder="billing, access, hardware, other" /></Field>}
          {d.output === "json" && <Field label="JSON fields (comma separated)" hint="Every output must have these keys"><input value={d.fieldsText} onChange={(e) => up("fieldsText", e.target.value)} placeholder="topic, key_points, deadlines" /></Field>}
          {d.output !== "continuation" && <Field label={d.output === "custom" ? "Output prompt (required)" : "Extra guidance for the outputs (optional)"} hint={`Placeholders: ${outVars.map((v) => `{{ ${v} }}`).join("  ")}`}>
            <textarea rows={3} value={d.output_prompt} onChange={(e) => up("output_prompt", e.target.value)}
              placeholder={d.output === "custom" ? "e.g. Reply to {{ input }} as a friendly support agent, using only the passage." : "e.g. Use numbered steps when there is a procedure."} /></Field>}

          <div className="tpl-sec"><span className="tpl-n">3</span><b>Variety</b><small>a value is picked at random for each chunk and can be used as {"{{ name }}"}</small>
            <div className="grow" /><button type="button" className="link" disabled={d.variety.length >= 6} onClick={() => up("variety", [...d.variety, { name: `field${d.variety.length + 1}`, text: "" }])}><Icon name="plus" size={13} />Add field</button></div>
          {d.variety.length === 0 && <div className="muted small">No variety fields.</div>}
          {d.variety.map((v, i) => (
            <div className="tpl-var" key={i}>
              <input className="mono" aria-label="Field name" value={v.name} onChange={(e) => up("variety", d.variety.map((x, j) => (j === i ? { ...x, name: e.target.value.toLowerCase() } : x)))} />
              <input aria-label="Values" value={v.text} placeholder="values, comma separated" onChange={(e) => up("variety", d.variety.map((x, j) => (j === i ? { ...x, text: e.target.value } : x)))} />
              <button type="button" className="icon-btn" aria-label={`Remove ${v.name}`} onClick={() => up("variety", d.variety.filter((_, j) => j !== i))}><Icon name="x" size={14} /></button>
            </div>
          ))}

          <div className="tpl-sec"><span className="tpl-n">4</span><b>Checks</b><small>{d.output === "continuation" ? "no judge: the output is the document text" : "rows that fail are dropped, with the reason"}</small></div>
          {d.output !== "continuation" && <>
            <div className="grid2">
              <Field label="Judge: supported by the text"><Segmented value={d.checks.grounded} onChange={(v) => upCheck("grounded", v)} options={[[true, "On"], [false, "Off"]]} /></Field>
              {!["json", "label"].includes(d.output) && <Field label="Words in the output (min / max, 0 = any)">
                <div className="tpl-minmax"><span className="inset"><small>Min</small><NumberInput value={d.checks.min_words} min={0} onChange={(v) => upCheck("min_words", v)} /></span>
                  <span className="inset"><small>Max</small><NumberInput value={d.checks.max_words} min={0} onChange={(v) => upCheck("max_words", v)} /></span></div></Field>}
            </div>
            <Field label="Judge rubric (optional)" hint="Scored as 'follows' together with: does the output do what the input asks?">
              <textarea rows={2} value={d.checks.rubric} onChange={(e) => upCheck("rubric", e.target.value)} placeholder="e.g. Uses numbered steps for procedures and never gives legal advice." /></Field>
            {d.output === "json" && <p className="note"><Icon name="check" size={14} />Code check: the output must be valid JSON with every field.</p>}
            {d.output === "label" && <p className="note"><Icon name="check" size={14} />Code check: the output must be one of the labels.</p>}
          </>}
        </div>
        <aside className="tpl-side">
          <div className="tpl-sec first"><b>Training row</b><small>how one row will look</small></div>
          <Flow t={t} />
          <pre className="tpl-preview">{rowPreview(t)}</pre>
        </aside>
      </div>
    </Modal>
  );
}

function rowPreview(t) {
  const inText = { question: "How many leave days do new employees get?", instruction: "List the steps to request leave.", custom: "(what your input prompt writes)" }[t.input] || "";
  const text = "Employees get 25 days of annual leave. Requests go to the manager…";
  const instr = (t.instruction || "").replace(/{{\s*\w+\s*}}/g, "…") || (t.input === "text_start" ? "Continue the text." : "Summarise this text.");
  const out = { answer: "New employees get 25 days of annual leave.", completion: "1. Open the HR portal …", summary: "Staff have 25 leave days; …",
    json: `{${t.json_fields.map((f) => `"${f}": "…"`).join(", ")}}`, label: t.labels[0] || "label", rewrite: "Hi all, a quick note on leave …",
    continuation: "Requests go to the manager at least two weeks before …", custom: "(what your output prompt writes)" }[t.output];
  let user = inText;
  if (t.input === "text") user = `${instr}\n\n${text}`;
  else if (t.input === "text_start") user = "Employees get 25 days of annual leave.";
  else if (t.context_in_row) user = `Context:\n${text}\n\n${inText}`;
  if (t.format === "chat") {
    const u = t.input === "text_start" ? `${instr}\n\n${user}` : user;
    return JSON.stringify({ messages: [{ role: "user", content: u }, { role: "assistant", content: out }] }, null, 1);
  }
  if (t.format === "instruction") {
    const row = t.input === "text" ? { instruction: instr, input: text, output: out }
      : t.input === "text_start" ? { instruction: instr, input: user, output: out }
        : { instruction: inText, input: t.context_in_row ? text : "", output: out };
    return JSON.stringify(row, null, 1);
  }
  if (t.format === "completion") return JSON.stringify({ prompt: user, completion: out }, null, 1);
  return JSON.stringify({ text: `${user}${t.input === "text_start" ? " " : "\n\n"}${out}` }, null, 1);
}

// ------------------------------------------------------------------ 4. training mix
export function MixStep({ form, setIn, setForm }) {
  const [topic, setTopic] = useState("");
  const [open, setOpen] = useState(null);
  const g = form.general, d = form.decline, p = form.para;
  const own = g.topics.filter((t) => !G_TOPICS.some(([k]) => k === t));
  const addTopic = () => {
    const t = topic.trim();
    if (t && !g.topics.some((x) => x.toLowerCase() === t.toLowerCase())) setIn("general", "topics", [...g.topics, t]);
    setTopic("");
  };
  const setRules = (fn) => setForm((f) => ({ ...f, decline: { ...f.decline, rules: fn(f.decline.rules) } }));
  const upd = (i, k, v) => setRules((rs) => rs.map((r, j) => (j === i ? { ...r, [k]: v } : r)));
  const missing = D_PRESETS.filter((x) => !d.rules.some((r) => r.name.toLowerCase() === x.name.toLowerCase()));
  const mix = mixEstimate(form), warnings = mixWarnings(form);
  return (
    <>
      <div className="mixsum">
        <div className="mixsum-hd"><b>Expected mix</b><span className="muted small">{mix.paraphrases || mix.decline || mix.general ? "per 100 document Q&A pairs" : "document Q&A only"}</span></div>
        <MixBar mix={mix} />
      </div>
      {warnings.map((w) => <div className="warn" key={w}><Icon name="alert" size={15} />{w}</div>)}

      <Toggle icon="repeat" title="Paraphrases" text="The same question in new words, with the same answer" on={p.enabled} onChange={(v) => setIn("para", "enabled", v)}>
        <div className="inline"><span>New wordings per question</span><NumberInput className="mini" value={p.count} min={1} max={10} onChange={(v) => setIn("para", "count", v)} /></div>
        <Chips options={P_STYLES} value={p.styles} onToggle={(k) => setIn("para", "styles", toggle(p.styles, k))} />
      </Toggle>

      <Toggle icon="shield" title="Decline answers" text="Questions the model should politely refuse" on={d.enabled} onChange={(v) => setIn("decline", "enabled", v)}>
        <div className="inline"><span>Share of document pairs</span><NumberInput className="mini" value={d.share} min={1} max={30} onChange={(v) => setIn("decline", "share", v)} /><span>%</span></div>
        <div className="rules2">
          {d.rules.map((r, i) => (
            <div className={`rule2${r.on ? "" : " off"}${open === i ? " open" : ""}`} key={i}>
              <div className="rule2-top">
                <Switch on={r.on} onChange={(v) => upd(i, "on", v)} label={`Use ${r.name}`} />
                <button type="button" className="rule2-name" onClick={() => setOpen(open === i ? null : i)}>
                  <b>{r.name || "(no name)"}</b><small>{r.scope || "No scope yet"}</small>
                </button>
                <span className="tag">{r.reply_mode === "guided" ? "Teacher writes reply" : "Exact reply"}</span>
                <button type="button" className="icon-btn" aria-label={open === i ? "Close" : `Edit ${r.name}`} onClick={() => setOpen(open === i ? null : i)}><Icon name={open === i ? "x" : "edit"} size={14} /></button>
              </div>
              {open === i && (
                <div className="rule2-bd">
                  <Field label="Rule name"><input value={r.name} onChange={(e) => upd(i, "name", e.target.value)} /></Field>
                  <Field label="Decline questions like these"><textarea rows={2} value={r.scope} placeholder="e.g. Questions asking for the salary of a named colleague." onChange={(e) => upd(i, "scope", e.target.value)} /></Field>
                  <Field label="Reply">
                    <Segmented small value={r.reply_mode} onChange={(v) => upd(i, "reply_mode", v)} options={[["fixed", "Exact text"], ["guided", "Teacher writes it"]]} />
                    <textarea rows={2} value={r.reply} onChange={(e) => upd(i, "reply", e.target.value)}
                      placeholder={r.reply_mode === "guided" ? "How to reply, e.g. Politely refuse and point to the HR portal." : "The exact reply for every question of this rule."} />
                  </Field>
                  <div className="inline"><div className="grow" /><button type="button" className="btn small ghost danger" onClick={() => { setRules((rs) => rs.filter((_, j) => j !== i)); setOpen(null); }}><Icon name="trash" size={13} />Delete rule</button></div>
                </div>
              )}
            </div>
          ))}
        </div>
        <div className="inline">
          <button type="button" className="btn small" onClick={() => { setRules((rs) => [...rs, { name: `Rule ${rs.length + 1}`, scope: "", reply: "", reply_mode: "fixed", on: true }]); setOpen(d.rules.length); }}><Icon name="plus" size={13} />New rule</button>
          {missing.map((x) => <button type="button" key={x.name} className="btn small ghost" onClick={() => setRules((rs) => [...rs, { ...x, on: true }])}><Icon name="plus" size={13} />{x.name}</button>)}
        </div>
      </Toggle>

      <Toggle icon="globe" title="General knowledge" text="Keeps everyday answers: facts, how-tos, writing help" on={g.enabled} onChange={(v) => setIn("general", "enabled", v)}>
        <div className="inline"><span>Share of document pairs</span><NumberInput className="mini" value={g.share} min={1} max={50} onChange={(v) => setIn("general", "share", v)} /><span>%</span></div>
        <Chips options={[...G_TOPICS, ...own.map((t) => [t, t])]} value={g.topics} onToggle={(k) => setIn("general", "topics", toggle(g.topics, k))}
          onRemove={(k) => (own.includes(k) ? () => setIn("general", "topics", g.topics.filter((x) => x !== k)) : null)} />
        <div className="addtopic"><input value={topic} placeholder="Add your own topic…" onChange={(e) => setTopic(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && (e.preventDefault(), addTopic())} />
          <button type="button" className="btn small" onClick={addTopic} disabled={!topic.trim()}><Icon name="plus" size={13} />Add</button></div>
      </Toggle>
    </>
  );
}

// ------------------------------------------------------------------ 5. advanced
function Fold({ icon, title, hint, children, first }) {
  return (
    <details className="fold" open={first}>
      <summary><span className="opt-ico"><Icon name={icon} size={16} /></span><span className="opt-t"><b>{title}</b><small>{hint}</small></span><Icon name="chevron" size={16} className="fold-chev" /></summary>
      <div className="fold-bd">{children}</div>
    </details>
  );
}

export function AdvancedStep({ form, set, setIn, onReset }) {
  const onOff = (k) => <Segmented value={form[k]} onChange={(v) => set(k, v)} options={[[true, "On"], [false, "Off"]]} />;
  return (
    <>
      <div className="inline"><span className="muted">Open a section to change it.</span><div className="grow" />
        <button type="button" className="btn small ghost" onClick={onReset}><Icon name="refresh" size={13} />Reset everything to the defaults</button></div>
      <Fold first icon="image" title="Parsing & images" hint="OCR, image size limits">
        <div className="grid3">
          <Field label="OCR for scanned pages">{onOff("ocr")}</Field>
          <Field label="Skip images smaller than (px)"><NumberInput value={form.minImagePx} onChange={(v) => set("minImagePx", v)} /></Field>
          <Field label="Max images per document"><NumberInput value={form.maxImages} onChange={(v) => set("maxImages", v)} /></Field>
          {form.vision
            ? <Field label="Skip image types" hint="Types the vision model may return" className="span2"><input value={form.skipImageTypes} onChange={(e) => set("skipImageTypes", e.target.value)} /></Field>
            : <Field label="Keep image text with at least (characters)"><NumberInput value={form.minOcrChars} onChange={(v) => set("minOcrChars", v)} /></Field>}
        </div>
      </Fold>
      <Fold icon="book" title="Chunks & knowledge base" hint="Chunk sizes, summaries, glossary">
        <div className="grid3">
          <Field label="Chunk size min (tokens)"><NumberInput value={form.chunkMin} onChange={(v) => set("chunkMin", v)} /></Field>
          <Field label="Chunk size max (tokens)"><NumberInput value={form.chunkMax} onChange={(v) => set("chunkMax", v)} /></Field>
          <Field label="Merge pieces below (tokens)"><NumberInput value={form.mergeBelow} onChange={(v) => set("mergeBelow", v)} /></Field>
          <Field label="Related chunks per answer" hint="0 = no related-chunk search"><NumberInput value={form.related} min={0} max={8} onChange={(v) => set("related", v)} /></Field>
          <Field label="Document summary">{onOff("summary")}</Field>
          <Field label="Glossary">{onOff("glossary")}</Field>
          <Field label="Collection overview">{onOff("overview")}</Field>
        </div>
      </Fold>
      <Fold icon="spark" title="Generation" hint="Questions per chunk, temperatures, answer length">
        <div className="grid3">
          <Field label="One question per (tokens)"><NumberInput value={form.tpq} onChange={(v) => set("tpq", v)} /></Field>
          <Field label="Questions per chunk, min"><NumberInput value={form.perMin} onChange={(v) => set("perMin", v)} /></Field>
          <Field label="Questions per chunk, max"><NumberInput value={form.perMax} onChange={(v) => set("perMax", v)} /></Field>
          <Field label="Question temperature"><NumberInput step={0.1} value={form.tempQ} onChange={(v) => set("tempQ", v)} /></Field>
          <Field label="Answer temperature"><NumberInput step={0.1} value={form.tempA} onChange={(v) => set("tempA", v)} /></Field>
          <Field label="Max answer tokens"><NumberInput value={form.maxAnswerTokens} onChange={(v) => set("maxAnswerTokens", v)} /></Field>
          <Field label="Paraphrase temperature"><NumberInput step={0.1} value={form.para.temperature} onChange={(v) => setIn("para", "temperature", v)} /></Field>
          <Field label="General knowledge answers"><Segmented value={form.general.answer_length} onChange={(v) => setIn("general", "answer_length", v)} options={[["short", "Short"], ["detailed", "Detailed"]]} /></Field>
          <Field label="General answer temperature"><NumberInput step={0.1} value={form.general.temperature} onChange={(v) => setIn("general", "temperature", v)} /></Field>
        </div>
      </Fold>
      <Fold icon="check" title="Judge & duplicates" hint="Quality thresholds, near-duplicate removal">
        <div className="grid3">
          <Field label="Grounded score at least" hint="1–5"><NumberInput value={form.minGrounded} min={1} max={5} onChange={(v) => set("minGrounded", v)} /></Field>
          <Field label="Other scores at least" hint="1–5"><NumberInput value={form.minOther} min={1} max={5} onChange={(v) => set("minOther", v)} /></Field>
          <Field label="Max row length (tokens)"><NumberInput value={form.maxRowTokens} onChange={(v) => set("maxRowTokens", v)} /></Field>
          <Field label="Near-duplicate when Jaccard ≥"><NumberInput step={0.05} value={form.jaccard} onChange={(v) => set("jaccard", v)} /></Field>
          <Field label="Ask the teacher when Jaccard ≥" hint="1.0 = never ask"><NumberInput step={0.05} value={form.teacherJaccard} onChange={(v) => set("teacherJaccard", v)} /></Field>
        </div>
      </Fold>
      <Fold icon="download" title="Output" hint="Where the accepted dataset is saved">
        <Field label="Save the dataset as (on Accept)" hint="Default: the job folder's dataset/all.jsonl"><input className="mono" value={form.outPath} placeholder="<job folder>/dataset/all.jsonl" onChange={(e) => set("outPath", e.target.value)} /></Field>
      </Fold>
    </>
  );
}





























//Summary.jsx
// Right-hand panel of the Generate page: what the job will do, what is still missing, and Start / Preview.
import { inShort, isQa, mixEstimate, outShort, short, visionUsed } from "../model.js";
import { Icon } from "../ui.jsx";
import { MixBar } from "./steps.jsx";

const QT = { factual: "Factual", explanation: "Explanation", "how-to": "How-to", from_images: "From images", comparison: "Comparison" };

function Row({ icon, label, children, onEdit }) {
  return (
    <button type="button" className="sum-row" onClick={onEdit} title={`Edit ${label.toLowerCase()}`}>
      <span className="sum-ico"><Icon name={icon} size={15} /></span>
      <span className="sum-t"><small>{label}</small><span>{children}</span></span>
      <Icon name="edit" size={13} className="sum-edit" />
    </button>
  );
}

const dotOf = (s, testing) => (testing ? "run" : !s ? "" : s.ok && s.models?.length ? "ok" : "bad");

export default function Summary({ form, files, folders, ep, issues, starting, startErr, onStart, goStep }) {
  const testing = ep.state === "testing";
  const vUsed = visionUsed(form);
  const ready = issues.length === 0;
  const size = files.reduce((a, f) => a + f.size, 0) / 1048576;
  const docs = [files.length && `${files.length} file${files.length > 1 ? "s" : ""} (${size.toFixed(1)} MB)`, folders.length && `${folders.length} server folder${folders.length > 1 ? "s" : ""}`].filter(Boolean).join(" + ");
  return (
    <aside className="card summary">
      <div className="sum-hd"><span className="step-n">Summary</span><h3>{form.name.trim() || "Untitled dataset"}</h3>
        <span className="pill mode">{form.mode === "reasoning" ? "Reasoning SFT" : "Chat SFT"}</span></div>
      <div className="sum-rows">
        <Row icon="doc" label="Documents" onEdit={() => goStep(0)}>{docs || <em>None yet</em>}</Row>
        <Row icon="server" label="Models" onEdit={() => goStep(1)}>
          <span className="sum-ep"><span className={`dot ${dotOf(ep.teacher, testing)}`} />Teacher <b>{short(form.teacherModel) || "not set"}</b></span>
          <span className="sum-ep"><span className={`dot ${vUsed ? dotOf(ep.vision, testing) || "bad" : "muted"}`} />Vision <b>{vUsed ? short(form.visionModel) || "not set" : "off (OCR only)"}</b></span>
        </Row>
        <Row icon="layers" label="Dataset & images" onEdit={() => goStep(2)}>
          {form.template.name} <span className="sum-sub">{inShort(form.template)} → {outShort(form.template)}</span>
          {form.template.input === "question" && <span className="sum-sub">{form.types.map((t) => QT[t] || t).join(", ") || <em>No types picked</em>}</span>}
          <span className="sum-sub">{isQa(form.template) ? `${form.answerLength === "short" ? "Short" : "Detailed"} answers · up to ${Number(form.maxPairs || 0).toLocaleString()} pairs` : `Up to ${Number(form.maxPairs || 0).toLocaleString()} rows`}</span>
          <span className="sum-sub">{!form.readImages ? "Images ignored" : visionUsed(form) ? "Images read by the vision model" : "Images: OCR only"}</span>
          <span className="sum-sub">Split {form.split.join(" / ")} by {form.splitBy}</span>
        </Row>
        <Row icon="mix" label="Training mix" onEdit={() => goStep(3)}><MixBar mix={mixEstimate(form)} /></Row>
      </div>

      <div className="sum-ft">
        {ready
          ? <div className="sum-ready"><Icon name="check" size={15} />Ready to start</div>
          : <div className="sum-issues">
            <b><Icon name="alert" size={15} />{issues.length === 1 ? "One thing to fix" : `${issues.length} things to fix`}</b>
            {issues.map((x) => <button type="button" key={x.text} onClick={() => goStep(x.step)}>{x.text}<Icon name="arrow" size={13} /></button>)}
          </div>}
        {startErr && <div className="error">{startErr}</div>}
        <button type="button" className="btn primary lg wide" disabled={starting} onClick={() => onStart(false)}><Icon name="play" size={13} />{starting ? "Starting…" : "Start job"}</button>
        <button type="button" className="btn wide" disabled={starting} onClick={() => onStart(true)}><Icon name="eye" size={15} />Preview on 3 chunks first</button>
        <p className="sum-note">A preview runs the whole pipeline on 3 chunks, so you can check the output in a few minutes.</p>
      </div>
    </aside>
  );
}
























//RunProgress.jsx
// Live view of a job: overall progress, a timeline of the pipeline steps, and Cancel / Retry / Open results.
import { STEP_TITLES, fmtN, pct, secs, stepResult, stepTitle } from "../model.js";
import { Icon } from "../ui.jsx";

const TONE = { queued: "run", running: "run", done: "ok", failed: "bad", cancelled: "bad" };
const LABEL = { queued: "Queued", running: "Running", done: "Finished", failed: "Failed", cancelled: "Cancelled" };

export default function RunProgress({ job, jobId, progress, onCancel, onRetry, onNew, onReuse, openResults }) {
  if (!job) return <div className="card run-loading"><span className="dot run" />Loading job <span className="mono">{jobId}</span>…</div>;
  const steps = job.steps || STEP_TITLES.map(([key, title]) => ({ key, title, status: "waiting" }));
  const frac = (s) => (s.status === "done" || s.status === "skipped" ? 1 : s.status === "running" && progress[s.key]?.total ? progress[s.key].done / progress[s.key].total : 0);
  const overall = Math.round((100 * steps.reduce((a, s) => a + frac(s), 0)) / steps.length);
  const elapsed = steps.reduce((a, s) => a + (s.seconds || 0), 0);
  const curIdx = steps.findIndex((s) => s.status === "running");
  const cur = steps[curIdx];
  const st = job.status, tone = TONE[st] || "";
  const active = st === "queued" || st === "running";
  const failed = steps.find((s) => s.status === "failed");
  const headline = st === "done" ? `${fmtN(job.counts?.pairs_kept ?? 0)} rows are ready`
    : st === "running" && cur ? `Step ${curIdx + 1} of ${steps.length}: ${cur.title}`
      : st === "queued" ? "Waiting for the worker" : st === "failed" ? `Stopped at ${failed?.title || "a step"}` : st === "cancelled" ? "Cancelled" : st;

  return (
    <div className="runview">
      <section className={`card run-hero ${tone}`}>
        <div className="run-hero-top">
          <span className={`run-badge ${tone}`}><Icon name={st === "done" ? "check" : st === "failed" || st === "cancelled" ? "x" : "spark"} size={22} stroke={2.2} /></span>
          <div className="run-title">
            <span className="crumb"><span className="mono">{job.job_id}</span>{job.request?.preview_chunks ? " · preview on 3 chunks" : ""}</span>
            <h2>{job.name}</h2>
            <p>{headline}</p>
          </div>
          <div className="grow" />
          <span className={`pill ${tone}`}>{LABEL[st] || st}</span>
        </div>
        <div className="run-meter">
          <div className="run-bar"><i className={tone} style={{ width: `${st === "done" ? 100 : overall}%` }} /></div>
          <div className="run-meta"><b>{st === "done" ? 100 : overall}%</b><span><Icon name="clock" size={14} />{secs(elapsed)}</span></div>
        </div>
        {job.error && <div className="error">{job.error}</div>}
        <div className="run-actions">
          {st === "done" && <button type="button" className="btn primary lg" onClick={openResults}>Review the results<Icon name="arrow" size={15} /></button>}
          {active && <button type="button" className="btn danger" onClick={onCancel}><Icon name="stop" size={12} />Cancel job</button>}
          {(st === "failed" || st === "cancelled") && <button type="button" className="btn primary" onClick={onRetry}><Icon name="refresh" size={14} />Retry from the last step</button>}
          <div className="grow" />
          <button type="button" className="btn ghost" onClick={onReuse} disabled={active}><Icon name="repeat" size={14} />New job with these settings</button>
          <button type="button" className="btn" onClick={onNew} disabled={active}><Icon name="plus" size={14} />New job</button>
        </div>
      </section>

      <section className="card timeline">
        <div className="card-hd"><div className="card-t"><h2>Pipeline</h2><small>{steps.filter((s) => s.status === "done").length} of {steps.length} steps done</small></div></div>
        <ol className="tl">
          {steps.map((s, i) => {
            const cls = { done: "done", running: "now", skipped: "skip", failed: "fail" }[s.status] || "wait";
            const sp = progress[s.key];
            const res = stepResult(s.key, s.result);
            return (
              <li key={s.key} className={`tl-item ${cls}`}>
                <span className="tl-dot">{s.status === "done" ? <Icon name="check" size={13} stroke={2.8} /> : s.status === "failed" ? <Icon name="x" size={13} stroke={2.8} /> : s.status === "skipped" ? "–" : i + 1}</span>
                <div className="tl-body">
                  <div className="tl-row"><b>{stepTitle(s.key, s.title, job.request?.template)}</b><span className="tl-time">{s.status === "running" ? (sp?.total ? `${fmtN(sp.done)} / ${fmtN(sp.total)}` : "working…") : s.status === "done" && s.seconds != null ? secs(s.seconds) : s.status === "skipped" ? "skipped" : ""}</span></div>
                  {s.status === "running" && <div className="tl-bar"><i style={{ width: `${sp?.total ? pct(sp.done, sp.total) : 8}%` }} /></div>}
                  {res && s.status !== "running" && <small>{res}</small>}
                </div>
              </li>
            );
          })}
        </ol>
      </section>
    </div>
  );
}


























//generate.css
/* Generate Dataset page. Uses the tokens from styles.css (coral theme). */

.gen { display: grid; gap: 20px; max-width: 1360px; margin: 0 auto; }

/* ---- stepper */
.stepper { display: grid; grid-template-columns: repeat(5, minmax(0, 1fr)); gap: 6px; padding: 6px; background: var(--surface);
  border: 1px solid var(--line); border-radius: var(--radius-lg); box-shadow: var(--shadow); }
.stp { position: relative; display: flex; align-items: center; gap: 12px; padding: 10px 12px; border: 0; border-radius: 12px; background: none;
  cursor: pointer; text-align: left; min-width: 0; transition: background .12s; }
.stp:not(.on):hover { background: rgba(17, 24, 39, .025); }
.stp + .stp::before { content: ""; position: absolute; left: -6px; top: 50%; width: 6px; height: 2px; background: var(--line); }
.stp-n { width: 32px; height: 32px; border-radius: 50%; display: grid; place-items: center; flex: none; font-weight: 700; font-size: 14px;
  background: var(--surface-3); color: var(--muted); border: 1px solid var(--line); transition: all .15s; }
.stp-t { display: grid; min-width: 0; }
.stp-t b { font-size: 14.5px; font-weight: 600; color: var(--ink-2); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.stp-t small { font-size: 12.5px; color: var(--faint); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.stp.ok .stp-n { background: var(--surface); color: var(--green-600); border-color: var(--line-strong); }
.stp.ok .stp-t small { color: var(--muted); }
.stp.on { background: none; }
.stp.on .stp-n { background: var(--grad); color: #fff; border-color: transparent; box-shadow: none; }
.stp.on .stp-t b { color: var(--ink); }

/* ---- main card + summary */
.gen-grid { display: grid; grid-template-columns: minmax(0, 1fr) 340px; gap: 20px; align-items: start; }
.gen-main { min-height: 560px; }
.gen-hd { display: flex; gap: 16px; align-items: flex-start; padding: 22px 26px 18px; border-bottom: 1px solid var(--line-2); }
.gen-hd-ico { width: 44px; height: 44px; border-radius: 12px; display: grid; place-items: center; flex: none; background: var(--surface-2); color: var(--ink-2);
  border: 1px solid var(--line); }
.gen-hd h2 { margin: 2px 0 4px; font-size: 22px; font-weight: 700; letter-spacing: -.2px; }
.gen-hd p { margin: 0; color: var(--muted); font-size: 14.5px; max-width: 640px; }
.gen-bd { padding: 22px 26px 26px; display: grid; gap: 26px; align-content: start; }
.gen-ft { display: flex; align-items: center; gap: 10px; padding: 14px 26px; border-top: 1px solid var(--line-2); background: var(--surface-2);
  border-radius: 0 0 var(--radius-lg) var(--radius-lg); position: sticky; bottom: 0; }

/* sections inside a step */
.g-sec { display: grid; gap: 14px; }
.g-sec-hd { display: flex; align-items: flex-end; justify-content: space-between; gap: 12px; }
.g-sec-hd h3 { margin: 0; font-size: 15.5px; font-weight: 650; letter-spacing: -.1px; }
.g-sec-hd p { margin: 2px 0 0; font-size: 13.5px; color: var(--muted); }
.note { display: flex; gap: 8px; align-items: center; margin: 0; font-size: 13.5px; color: var(--muted); padding: 10px 12px; background: var(--surface-2);
  border: 1px dashed var(--line-strong); border-radius: 10px; }
.btn.lg { height: 44px; padding: 0 20px; font-size: 14.5px; border-radius: 10px; }

/* status badges */
.badge { display: inline-flex; align-items: center; gap: 6px; font-size: 12.5px; font-weight: 600; padding: 3px 10px; border-radius: 99px;
  background: var(--surface-3); color: var(--muted); white-space: nowrap; }
.badge i { width: 7px; height: 7px; border-radius: 50%; background: currentColor; }
.badge.ok { background: var(--green-50); color: var(--green-600); }
.badge.bad { background: var(--red-50); color: var(--red); }
.badge.run { background: var(--accent-50); color: var(--accent-600); }
.badge.run i { animation: pulse 1.3s infinite; }

/* choice cards */
.choices { display: grid; gap: 12px; }
.choices.two { grid-template-columns: repeat(2, minmax(0, 1fr)); }
.choices.five { grid-template-columns: repeat(auto-fill, minmax(170px, 1fr)); }
.choice { display: grid; gap: 4px; align-content: start; text-align: left; padding: 14px 16px; border: 1.5px solid var(--line); border-radius: 14px;
  background: var(--surface); cursor: pointer; transition: border-color .12s, box-shadow .12s, background .12s; }
.choice:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); }
.choice-top { display: flex; align-items: center; justify-content: space-between; margin-bottom: 6px; }
.choice-ico { width: 36px; height: 36px; border-radius: 10px; display: grid; place-items: center; background: var(--surface-3); color: var(--muted); transition: all .15s; }
.choice-mark { width: 20px; height: 20px; border-radius: 50%; border: 1.5px solid var(--line-strong); display: grid; place-items: center; color: #fff; margin-left: auto; transition: all .15s; }
.choice-mark.sq { border-radius: 6px; }
.choice b { font-size: 14.5px; font-weight: 600; color: var(--ink); }
.choice small { font-size: 13px; color: var(--muted); line-height: 1.45; }
.choice.on { border-color: var(--accent); background: var(--surface); box-shadow: 0 0 0 3px rgba(var(--accent-rgb), .07); }
.choice.on .choice-ico { background: var(--accent-50); color: var(--accent-600); }
.choice.on .choice-mark { background: var(--accent); border-color: transparent; }

/* documents */
.dropzone { display: grid; justify-items: center; gap: 10px; padding: 34px 20px; border: 1.5px dashed var(--line-strong); border-radius: 16px; text-align: center;
  background: var(--surface-2); cursor: pointer; transition: border-color .15s, background .15s, transform .15s; }
.dropzone:hover, .dropzone.drag { border-color: var(--accent-400); background: var(--tint-0); }
.dropzone.drag { transform: scale(1.005); }
.dz-ico { width: 56px; height: 56px; border-radius: 16px; display: grid; place-items: center; background: var(--surface); color: var(--accent-600);
  border: 1px solid var(--line); box-shadow: var(--shadow); }
.dropzone b { font-size: 16px; font-weight: 600; }
.dropzone u { text-decoration: none; color: var(--accent-600); }
.dz-types { display: flex; gap: 6px; flex-wrap: wrap; justify-content: center; }
.dz-types span { font-size: 12px; font-weight: 600; color: var(--muted); background: var(--surface); border: 1px solid var(--line); padding: 2px 9px; border-radius: 99px; }
.filegrid { display: grid; grid-template-columns: repeat(auto-fill, minmax(230px, 1fr)); gap: 8px; }
.fcard { display: flex; align-items: center; gap: 10px; padding: 9px 10px; border: 1px solid var(--line); border-radius: 12px; background: var(--surface); min-width: 0; }
.fcard-n { display: grid; flex: 1; min-width: 0; font-weight: 600; font-size: 14px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.fcard-n small { font-weight: 400; font-size: 12.5px; color: var(--muted); }

/* models */
.epcards { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 16px; }
.epcard { display: grid; gap: 14px; align-content: start; padding: 18px; border: 1px solid var(--line); border-radius: 16px; background: var(--surface); box-shadow: var(--shadow); }
.epcard.off { background: var(--surface-2); }
.epcard-hd { display: flex; align-items: center; gap: 12px; }
.epcard-ft { display: flex; justify-content: flex-end; }
.testbar { display: flex; align-items: center; gap: 10px; padding: 12px 14px; border-radius: 12px; border: 1px solid var(--line); background: var(--surface-2); font-size: 14px; color: var(--ink-2); }
.testbar.good { background: var(--green-50); border-color: var(--green-200); color: var(--green-ink); }
.testbar.bad { background: var(--red-50); border-color: var(--red-200); color: var(--red-ink); }

/* option toggles (mix) */
.opt-ico { width: 38px; height: 38px; border-radius: 11px; display: grid; place-items: center; flex: none; background: var(--surface-3); color: var(--muted); transition: all .15s; }
.opt-ico.on, .opt.on .opt-ico { background: var(--accent-50); color: var(--accent-600); }
.opt-t { display: grid; min-width: 0; flex: 1; }
.opt-t b { font-size: 15px; font-weight: 600; }
.opt-t small { font-size: 13px; color: var(--muted); }
.opt { border: 1px solid var(--line); border-radius: 16px; background: var(--surface); transition: border-color .15s, box-shadow .15s; }
.opt:hover { border-color: var(--line-strong); }
.opt.on { border-color: var(--line-strong); box-shadow: var(--shadow); }
.opt-top { display: flex; align-items: center; gap: 14px; padding: 14px 16px; }
.opt-bd { display: grid; gap: 12px; padding: 14px 16px 16px; border-top: 1px dashed var(--line); }
.mixsum { display: grid; gap: 10px; padding: 16px 18px; border-radius: 16px; background: var(--surface-2); border: 1px solid var(--line); }
.mixsum-hd { display: flex; justify-content: space-between; align-items: center; }
.addtopic { display: flex; gap: 8px; max-width: 420px; }
.rules2 { display: grid; gap: 8px; }
.rule2 { border: 1px solid var(--line); border-radius: 12px; background: var(--surface); transition: opacity .15s; }
.rule2.off { opacity: .6; background: var(--surface-2); }
.rule2.open { border-color: var(--line-strong); opacity: 1; }
.rule2-top { display: flex; align-items: center; gap: 12px; padding: 10px 12px; }
.rule2-name { display: grid; flex: 1; min-width: 0; text-align: left; border: 0; background: none; padding: 0; cursor: pointer; }
.rule2-name b { font-size: 14px; font-weight: 600; }
.rule2-name small { font-size: 12.5px; color: var(--muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.rule2-bd { display: grid; gap: 12px; padding: 4px 12px 12px; }

/* questions: split */
.splitbar { display: flex; height: 34px; border-radius: 10px; overflow: hidden; gap: 3px; background: var(--line-2); }
.splitbar i { display: grid; place-items: center; min-width: 0; font-style: normal; transition: flex .25s; }
.splitbar span { font-size: 12.5px; font-weight: 600; color: #fff; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; padding: 0 6px; text-shadow: 0 1px 2px rgba(0, 0, 0, .15); }
.split4 { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 10px; }

/* advanced: folds */
.fold { border: 1px solid var(--line); border-radius: 14px; background: var(--surface); }
.fold[open] { box-shadow: var(--shadow); }
.fold summary { display: flex; align-items: center; gap: 14px; padding: 14px 16px; cursor: pointer; list-style: none; }
.fold summary::-webkit-details-marker { display: none; }
.fold summary .opt-ico { width: 34px; height: 34px; border-radius: 10px; }
.fold[open] summary .opt-ico { background: var(--surface-3); color: var(--ink-2); }
.fold-chev { color: var(--faint); transition: transform .15s; }
.fold[open] .fold-chev { transform: rotate(90deg); }
.fold-bd { padding: 4px 16px 18px; }

/* ---- summary panel */
.summary { position: sticky; top: 0; overflow: hidden; }
.sum-hd { padding: 20px 20px 16px; display: grid; gap: 4px; justify-items: start; background: var(--surface); border-bottom: 1px solid var(--line-2); }
.sum-hd h3 { margin: 0 0 6px; font-size: 19px; font-weight: 700; letter-spacing: -.3px; word-break: break-word; }
.sum-rows { display: grid; padding: 6px 8px; }
.sum-row { display: flex; gap: 12px; align-items: flex-start; padding: 11px 12px; border: 0; background: none; border-radius: 10px; text-align: left; cursor: pointer; width: 100%; }
.sum-row:hover { background: var(--surface-2); }
.sum-row + .sum-row { border-top: 1px solid var(--line-2); border-radius: 0; }
.sum-ico { width: 30px; height: 30px; border-radius: 9px; display: grid; place-items: center; background: var(--surface-3); color: var(--muted); flex: none; }
.sum-t { display: grid; gap: 3px; flex: 1; min-width: 0; font-size: 14px; color: var(--ink); }
.sum-t > small { font-size: 11.5px; font-weight: 600; text-transform: uppercase; letter-spacing: .5px; color: var(--faint); }
.sum-t em { color: var(--muted); font-style: normal; }
.sum-sub { display: block; font-size: 13px; color: var(--muted); }
.sum-ep { display: flex; align-items: center; gap: 8px; font-size: 13.5px; color: var(--muted); }
.sum-ep b { color: var(--ink); font-weight: 600; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; min-width: 0; }
.sum-edit { color: var(--faint); opacity: 0; margin-top: 8px; transition: opacity .12s; }
.sum-row:hover .sum-edit { opacity: 1; }
.sum-ft { display: grid; gap: 10px; padding: 16px 20px 20px; border-top: 1px solid var(--line-2); background: var(--surface-2); }
.sum-ready { display: flex; align-items: center; gap: 8px; font-weight: 600; font-size: 14px; color: var(--green-600); }
.sum-issues { display: grid; gap: 6px; padding: 12px; border-radius: 12px; background: var(--amber-50); border: 1px solid var(--amber-200); }
.sum-issues b { display: flex; align-items: center; gap: 7px; font-size: 13.5px; color: var(--amber-ink); }
.sum-issues button { display: flex; align-items: center; justify-content: space-between; gap: 8px; text-align: left; border: 0; background: none; padding: 2px 0;
  font-size: 13px; color: var(--amber-ink); cursor: pointer; }
.sum-issues button:hover { color: var(--amber-ink-2); text-decoration: underline; }
.sum-note { margin: 0; font-size: 12.5px; color: var(--muted); text-align: center; }

/* ---- run view */
.runview { display: grid; gap: 20px; max-width: 1100px; margin: 0 auto; }
.run-loading { flex-direction: row; align-items: center; gap: 10px; padding: 24px; color: var(--muted); }
.run-hero { padding: 24px 26px; gap: 20px; }

.run-hero-top { display: flex; align-items: center; gap: 16px; }
.run-badge { width: 56px; height: 56px; border-radius: 16px; display: grid; place-items: center; color: #fff; flex: none; background: var(--faint); }
.run-badge.run { background: var(--accent-50); color: var(--accent-600); box-shadow: inset 0 0 0 1px var(--accent-100); }
.run-badge.run .ico { animation: spin 2.4s linear infinite; }
.run-badge.ok { background: var(--green-50); color: var(--green-600); box-shadow: inset 0 0 0 1px var(--green-200); }
.run-badge.bad { background: var(--red-50); color: var(--red); box-shadow: inset 0 0 0 1px var(--red-200); }
@keyframes spin { to { transform: rotate(360deg); } }
.run-title .crumb { font-size: 12.5px; color: var(--muted); }
.run-title .mono { background: var(--surface-3); padding: 1px 6px; border-radius: 5px; }
.run-title h2 { margin: 4px 0 2px; font-size: 24px; font-weight: 700; letter-spacing: -.25px; }
.run-title p { margin: 0; font-size: 15px; color: var(--ink-2); }
.run-meter { display: grid; gap: 8px; }
.run-bar { height: 12px; border-radius: 99px; background: rgba(17, 24, 39, .06); overflow: hidden; }
.run-bar i { display: block; height: 100%; border-radius: 99px; background: var(--grad); transition: width .4s; }
.run-bar i.ok { background: var(--green); } .run-bar i.bad { background: var(--red); }
.run-meta { display: flex; align-items: center; justify-content: space-between; font-size: 14px; color: var(--muted); }
.run-meta b { font-size: 16px; color: var(--ink); font-variant-numeric: tabular-nums; }
.run-meta span { display: inline-flex; align-items: center; gap: 6px; }
.run-actions { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }
.tl { list-style: none; margin: 0; padding: 14px 22px 20px; display: grid; }
.tl-item { position: relative; display: flex; gap: 14px; padding: 8px 0; }
.tl-item:not(:last-child)::after { content: ""; position: absolute; left: 14px; top: 38px; bottom: -8px; width: 2px; background: var(--line-2); }
.tl-item.done:not(:last-child)::after { background: var(--line); }
.tl-dot { width: 30px; height: 30px; border-radius: 50%; display: grid; place-items: center; flex: none; font-size: 12.5px; font-weight: 700;
  background: var(--surface); color: var(--faint); border: 1.5px solid var(--line); z-index: 1; }
.tl-item.done .tl-dot { background: var(--surface); border-color: var(--line-strong); color: var(--green-600); }
.tl-item.now .tl-dot { background: var(--accent); border-color: transparent; color: #fff; animation: pulse 1.3s infinite; }
.tl-item.fail .tl-dot { background: var(--red); border-color: var(--red); color: #fff; }
.tl-item.skip .tl-dot { background: var(--surface-3); }
.tl-body { flex: 1; min-width: 0; display: grid; gap: 4px; padding-top: 4px; }
.tl-row { display: flex; align-items: baseline; gap: 10px; }
.tl-row b { font-size: 14.5px; font-weight: 600; flex: 1; }
.tl-item.wait .tl-row b, .tl-item.skip .tl-row b { color: var(--muted); font-weight: 500; }
.tl-item.now .tl-row b { color: var(--accent-600); }
.tl-time { font-size: 13px; color: var(--muted); font-variant-numeric: tabular-nums; }
.tl-body small { font-size: 13px; color: var(--muted); }
.tl-bar { height: 6px; border-radius: 99px; background: var(--line-2); overflow: hidden; max-width: 520px; }
.tl-bar i { display: block; height: 100%; background: var(--accent); border-radius: 99px; transition: width .3s; }

/* ---- smaller screens */
@media (max-width: 1240px) {
  .gen-grid { grid-template-columns: minmax(0, 1fr); }
  .summary { position: static; }
  .stp-t small { display: none; }
}
@media (max-width: 900px) {
  .stepper { grid-template-columns: repeat(5, auto); overflow-x: auto; }
  .stp-t { display: none; }
  .stp.on .stp-t { display: grid; }
  .choices.two, .epcards, .grid2 { grid-template-columns: minmax(0, 1fr); }
  .split4 { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .run-hero-top { flex-wrap: wrap; }
}

/* ---- calm accents: the gradient is kept for the main action only */
.btn.dark { background: var(--dark-btn-b); border-color: var(--dark-btn-b); color: var(--dark-btn-ink); }
.btn.dark:hover:not(:disabled) { background: var(--dark-btn-a); border-color: var(--dark-btn-a); color: var(--dark-btn-ink); }
.gen .switch.on, .gen .switch.on:hover:not(:disabled) { background: var(--accent); }
.gen .chip.on { background: var(--surface); border-color: var(--line-strong); color: var(--ink); }
.gen .chip.on .ico { color: var(--accent); }
.gen .seg button.on { color: var(--ink); }

/* ---- JetBrains Mono for IDs, numbers and file names */
.fcard-n, .fcard-n small, .tl-time, .run-meta b, .mixleg b, .split4 input, .sum-ep b, .run-title .mono { font-family: var(--mono); letter-spacing: -.02em; }
.fcard-n { font-size: 13.5px; font-weight: 500; }

/* ================================================================ motion
   Subtle and quick: things fade and rise a few pixels, the active step and tab slide.
   Everything is switched off for people who ask their system for reduced motion. */
:root { --ease-out: cubic-bezier(.2, .8, .2, 1); --ease-spring: cubic-bezier(.34, 1.4, .64, 1); }

@keyframes rise-in { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: none; } }
@keyframes fade-in { from { opacity: 0; } to { opacity: 1; } }
@keyframes slide-fwd { from { opacity: 0; transform: translateX(18px); } to { opacity: 1; transform: none; } }
@keyframes slide-back { from { opacity: 0; transform: translateX(-18px); } to { opacity: 1; transform: none; } }
@keyframes pop-in { 0% { opacity: 0; transform: scale(.4); } 100% { opacity: 1; transform: scale(1); } }
@keyframes open-down { from { opacity: 0; transform: translateY(-6px); } to { opacity: 1; transform: none; } }

/* page enter: stepper, main card and summary rise in one after the other */
.gen > .stepper { animation: rise-in .45s var(--ease-out) both; }
.gen-grid > .gen-main { animation: rise-in .5s var(--ease-out) .06s both; }
.gen-grid > .summary { animation: rise-in .5s var(--ease-out) .12s both; }
main .content > * { animation: fade-in .3s ease both; }

/* stepper: a soft card slides behind the active step */
.stepper { position: relative; }
.stp { z-index: 1; }
.stp.on { background: none; }
.stp-ind { position: absolute; top: 0; left: 0; border-radius: 12px; background: var(--surface-2); pointer-events: none;
  box-shadow: inset 0 0 0 1px var(--line); transition: transform .42s var(--ease-out), width .42s var(--ease-out); }
.stp-n { transition: background .25s, color .25s, border-color .25s, transform .2s var(--ease-spring); }
.stp:hover .stp-n { transform: scale(1.06); }
.stp-n .pop { animation: pop-in .35s var(--ease-spring) both; display: inline-flex; }
.stp-t b, .stp-t small { transition: color .25s; }

/* step content: header and body slide in from the direction you moved, sections follow in a short cascade */
.gen-hd.anim-fwd { animation: slide-fwd .38s var(--ease-out) both; }
.gen-hd.anim-back { animation: slide-back .38s var(--ease-out) both; }
.gen-bd.anim-fwd > *, .gen-bd.anim-back > * { animation: rise-in .45s var(--ease-out) both; }
.gen-bd > *:nth-child(1) { animation-delay: .04s; }
.gen-bd > *:nth-child(2) { animation-delay: .09s; }
.gen-bd > *:nth-child(3) { animation-delay: .14s; }
.gen-bd > *:nth-child(4) { animation-delay: .19s; }
.gen-bd > *:nth-child(n+5) { animation-delay: .24s; }

/* cards and options: gentle lift on hover, the tick pops in when chosen */
.choice { transition: border-color .2s, box-shadow .2s, background .2s, transform .2s var(--ease-out); }
.choice:hover { transform: translateY(-2px); }
.choice:active { transform: translateY(0) scale(.995); }
.choice-ico { transition: background .25s, color .25s, transform .25s var(--ease-spring); }
.choice.on .choice-ico { transform: scale(1.05); }
.choice.on .choice-mark .ico { animation: pop-in .3s var(--ease-spring) both; }
.fcard { animation: rise-in .35s var(--ease-out) both; transition: border-color .2s, box-shadow .2s, transform .2s var(--ease-out); }
.fcard:hover { border-color: var(--line-strong); box-shadow: var(--shadow-md); transform: translateY(-1px); }
.dz-ico { transition: transform .3s var(--ease-spring), box-shadow .3s; }
.dropzone:hover .dz-ico, .dropzone.drag .dz-ico { transform: translateY(-3px); box-shadow: var(--shadow-md); }
.epcard, .opt, .fold, .rule2 { transition: border-color .2s, box-shadow .25s, background .2s, opacity .2s; }
.epcard:hover { box-shadow: var(--shadow-md); }
.opt-ico { transition: background .25s, color .25s, transform .25s var(--ease-spring); }
.opt.on .opt-ico { transform: scale(1.04); }
.opt-bd, .rule2-bd, .fold-bd { animation: open-down .3s var(--ease-out) both; }
.chip { transition: background .2s, border-color .2s, color .2s, transform .15s var(--ease-out); }
.chip:active { transform: scale(.96); }
.chip.on .ico { animation: pop-in .25s var(--ease-spring) both; }
.fold-chev { transition: transform .3s var(--ease-out); }
.splitbar i { transition: flex .45s var(--ease-out); }
.mixbar i { transition: width .45s var(--ease-out); }
.switch span { transition: left .25s var(--ease-spring), background .2s; }
.switch { transition: background .25s; }

/* buttons: small press feedback; the main action lifts and its arrow nudges forward */
.btn { transition: background .2s, border-color .2s, color .2s, box-shadow .2s, transform .15s var(--ease-out); }
.btn:active:not(:disabled) { transform: scale(.98); }
.btn.primary:hover:not(:disabled), .btn.dark:hover:not(:disabled) { transform: translateY(-1px); box-shadow: 0 8px 18px -8px rgba(16, 24, 40, .35); }
.btn.primary:hover:not(:disabled) { box-shadow: 0 10px 22px -8px rgba(var(--accent-rgb), .55); }
.btn .ico { transition: transform .2s var(--ease-out); }
.btn:hover:not(:disabled) > .ico:last-child:not(:first-child) { transform: translateX(2px); }
.gen-ft .btn.ghost:hover:not(:disabled) > .ico:first-child { transform: translateX(-2px); }

/* summary: the status box swaps smoothly, bars glide to their new size */
.sum-row { transition: background .2s; }
.sum-ico { transition: background .2s, color .2s; }
.sum-row:hover .sum-ico { background: var(--accent-50); color: var(--accent-600); }
.sum-ready, .sum-issues { animation: rise-in .35s var(--ease-out) both; }
.sum-issues button { transition: color .15s, transform .15s; }

/* run view: hero and timeline rise in, steps cascade */
.runview > .run-hero { animation: rise-in .45s var(--ease-out) both; }
.runview > .card:not(.run-hero) { animation: rise-in .5s var(--ease-out) .08s both; }
.tl-item { animation: rise-in .4s var(--ease-out) both; }
.tl-item:nth-child(1) { animation-delay: .10s; } .tl-item:nth-child(2) { animation-delay: .13s; } .tl-item:nth-child(3) { animation-delay: .16s; }
.tl-item:nth-child(4) { animation-delay: .19s; } .tl-item:nth-child(5) { animation-delay: .22s; } .tl-item:nth-child(6) { animation-delay: .25s; }
.tl-item:nth-child(7) { animation-delay: .28s; } .tl-item:nth-child(n+8) { animation-delay: .31s; }
.run-bar i { transition: width .6s var(--ease-out); }
.run-badge { animation: pop-in .45s var(--ease-spring) .1s both; }


@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: .01ms !important; animation-delay: 0s !important; transition-duration: .01ms !important; }
}

/* ---- Dataset & images: template cards and the template editor */
.tpl-cards { display: grid; grid-template-columns: repeat(auto-fill, minmax(230px, 1fr)); gap: 12px; }
.tpl-card { position: relative; display: grid; gap: 6px; align-content: start; padding: 14px 14px 12px; border-radius: 14px; border: 1px solid var(--line); background: var(--surface);
  cursor: pointer; text-align: left; transition: border-color .2s, box-shadow .2s, background .2s, transform .2s var(--ease-out); }
.tpl-card:hover { transform: translateY(-2px); border-color: var(--line-strong); }
.tpl-card:focus-visible { outline: none; box-shadow: var(--ring); }
.tpl-card.on { border-color: rgba(var(--accent-rgb), .55); background: linear-gradient(180deg, var(--tint-1), var(--surface) 70%); box-shadow: 0 0 0 1px rgba(var(--accent-rgb), .35), 0 10px 24px -18px rgba(var(--accent-rgb), .8); cursor: default; }
.tpl-card.on:hover { transform: none; }
.tpl-card-top { display: flex; align-items: center; gap: 8px; margin-bottom: 2px; }
.tpl-card.on .choice-ico { background: var(--grad); color: #fff; }
.tpl-card .choice-mark { margin-left: auto; }
.tpl-card.on .choice-mark { background: var(--grad); border-color: transparent; color: #fff; }
.tpl-kind { font-size: 11px; font-weight: 650; letter-spacing: .2px; padding: 2px 8px; border-radius: 99px; background: var(--surface-3); color: var(--muted); }
.tpl-kind.own { background: var(--accent-50); color: var(--accent-600); }
.tpl-card > b { font-size: 14.5px; font-weight: 650; color: var(--ink); }
.tpl-card-d { color: var(--muted); font-size: 12.5px; line-height: 1.4; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; }
.tpl-flow { display: flex; flex-wrap: wrap; align-items: center; gap: 6px; color: var(--muted); margin-top: 2px; }
.tpl-flow .tag { font-size: 11.5px; font-weight: 650; padding: 2px 8px; border-radius: 7px; background: var(--accent-50); color: var(--accent-600); border: 1px solid var(--accent-100); }
.tpl-flow small { font-size: 12px; }
.tpl-edit { justify-self: start; margin-top: 4px; display: inline-flex; align-items: center; gap: 6px; height: 28px; padding: 0 10px; border-radius: 8px; font: inherit; font-size: 12.5px;
  font-weight: 600; color: var(--accent-600); background: var(--surface); border: 1px solid rgba(var(--accent-rgb), .35); cursor: pointer; transition: background .2s; }
.tpl-edit:hover { background: var(--accent-50); }

.tpl-form { display: grid; grid-template-columns: minmax(0, 1fr) 300px; gap: 22px; align-items: start; }
.tpl-main { display: grid; gap: 12px; min-width: 0; }
.tpl-side { position: sticky; top: 0; display: grid; gap: 8px; }
.tpl-sec { display: flex; align-items: center; gap: 8px; padding-top: 12px; margin-top: 2px; border-top: 1px solid var(--line-2); }
.tpl-sec.first { border-top: 0; padding-top: 0; margin-top: 0; }
.tpl-sec b { font-size: 14px; color: var(--ink); } .tpl-sec small { color: var(--muted); font-size: 12.5px; }
.tpl-n { width: 22px; height: 22px; border-radius: 7px; display: grid; place-items: center; font-size: 12px; font-weight: 700; color: #fff; background: var(--grad); flex: none; }
.tpl-var { display: grid; grid-template-columns: 160px minmax(0, 1fr) auto; gap: 8px; align-items: center; }
.tpl-minmax { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; }
.tpl-preview { margin: 0; padding: 12px 14px; border-radius: 12px; background: #0f1320; color: #dfe3f0; font-family: var(--mono); font-size: 12px; line-height: 1.5;
  white-space: pre-wrap; word-break: break-word; max-height: 420px; overflow: auto; border: 1px solid var(--line); }
.tpl-err { display: inline-flex; align-items: center; gap: 6px; color: var(--red); font-size: 12.5px; }
@media (max-width: 900px) { .tpl-form { grid-template-columns: 1fr; } .tpl-side { position: static; } }
@media (max-width: 700px) { .tpl-var { grid-template-columns: 1fr; } }
