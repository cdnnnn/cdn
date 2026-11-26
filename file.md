//steps.jsx
// The six setup steps of the Generate Dataset page.
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
// what happens to pictures: one plain choice, stored as readImages + vision
const PICS = [
  ["describe", "eye", "Describe them", "A vision model writes what each picture shows. Best results; needs the vision model."],
  ["ocr", "doc", "Read the words only", "Only text printed in pictures is kept (OCR). Faster; no vision model."],
  ["ignore", "x", "Skip pictures", "Only the text of the documents is used."],
];
export const picMode = (f) => (!f.readImages ? "ignore" : f.vision ? "describe" : "ocr");
const setPics = (set, k) => { set("readImages", k !== "ignore"); if (k !== "ignore") set("vision", k === "describe"); };

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

      <Section title="Pictures inside your documents" hint="Charts, diagrams and screenshots. Choose what happens to them.">
        <div className="choices three" role="radiogroup" aria-label="Pictures inside the documents">
          {PICS.map(([k, ic, title, text]) => (
            <Choice key={k} on={picMode(form) === k} onClick={() => setPics(set, k)} icon={ic} title={title} text={text} />
          ))}
        </div>
        <div className="pic-ex" aria-label="Example">
          <span className="pic-ex-h"><Icon name="image" size={14} />Example: a bar chart called “Leave days per year”</span>
          <span className={picMode(form) === "describe" ? "on" : ""}><b>Describe</b>“Bar chart: 22 leave days in 2023, 25 in 2024.” Questions can be asked about it.</span>
          <span className={picMode(form) === "ocr" ? "on" : ""}><b>Words only</b>“Leave days per year 2023 2024”: the numbers in the bars are lost.</span>
          <span className={picMode(form) === "ignore" ? "on" : ""}><b>Skip</b>Nothing from the chart is used.</span>
        </div>
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
            <button type="button" className="link" onClick={() => goStep?.(0)}>Change in Documents</button></p>
            : !form.vision ? <p className="note"><Icon name="alert" size={14} />Vision analysis is off: OCR only, faster.{" "}
              <button type="button" className="link" onClick={() => goStep?.(0)}>Change in Documents</button></p>
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

// ------------------------------------------------------------------ 3. dataset type
const QTYPES = [
  ["factual", "Factual", "“How many leave days do I get?”", "list"],
  ["explanation", "Explanation", "“Why must leave be booked two weeks ahead?”", "book"],
  ["how-to", "How-to", "“How do I request leave?”", "layers"],
  ["from_images", "From pictures", "“What does the leave chart show for 2024?”", "image"],
  ["comparison", "Comparison", "“How does sick leave differ from annual leave?”", "repeat"],
];
const SPLIT_COLORS = ["var(--split-1)", "var(--split-2)", "var(--split-3)"];

export function DatasetTypeStep({ form, set, tpl, reloadTemplates }) {
  const [editor, setEditor] = useState(null); // null | "edit" | "new"
  const t = form.template, qa = isQa(t);
  const lengthUsed = !["json", "label", "continuation"].includes(t.output);
  return (
    <>
      <Section title="What one training row looks like" hint="Each card shows an example made from an HR leave-policy document. Pick the one closest to what your model should do."
        right={<button type="button" className="btn small" onClick={() => setEditor("new")}><Icon name="plus" size={13} />New template</button>}>
        <TemplateCards form={form} set={set} tpl={tpl} onEdit={() => setEditor("edit")} />
      </Section>

      {t.input === "question" && (
        <Section title="Kinds of questions" hint="Pick one or more. The teacher model writes a mix of them for every part of your documents.">
          <div className="choices five">
            {QTYPES.map(([k, l, tx, ic]) => (
              <Choice multi key={k} on={form.types.includes(k)} onClick={() => set("types", toggle(form.types, k))} icon={ic} title={l} text={tx} />
            ))}
          </div>
        </Section>
      )}

      <Section title={qa ? "Answer length" : "Output length"}>
        {lengthUsed ? (
          <div className="choices two" role="radiogroup" aria-label="Length">
            <Choice on={form.answerLength === "short"} onClick={() => set("answerLength", "short")} title="Short" text="One or two sentences. e.g. “25 days a year.”" />
            <Choice on={form.answerLength === "detailed"} onClick={() => set("answerLength", "detailed")} title="Detailed"
              text="A full explanation. e.g. “New staff get 25 days a year. Book them in the HR portal at least two weeks ahead…”" />
          </div>
        ) : <p className="note"><Icon name="alert" size={14} />{t.output === "continuation" ? "The output is the rest of the text, so there is no length to choose." : "This template sets the length itself."}</p>}
      </Section>

      <Fold icon="gear" title="More options" hint="System prompt">
        <Field label="System prompt (optional)" hint="Put at the start of every row. If you use one, send the same text when you use the trained model.">
          <textarea rows={2} value={form.systemPrompt} onChange={(e) => set("systemPrompt", e.target.value)} placeholder="e.g. You are the HR assistant for Contoso." />
        </Field>
      </Fold>

      <TemplateModal open={!!editor} mode={editor} onClose={() => setEditor(null)} form={form} set={set} tpl={tpl} reloadTemplates={reloadTemplates} />
    </>
  );
}

// ------------------------------------------------------------------ 4. size & split
const SPLIT_INFO = [["Train", "teaches the model"], ["Validation", "checks progress"], ["Test", "gives the final score"]];
export function SizeStep({ form, set }) {
  const total = form.split.reduce((a, b) => a + (Number(b) || 0), 0);
  const qa = isQa(form.template);
  return (
    <>
      <Section title="How many rows" hint="The job stops when it reaches this number. 2,000 is a good first try; more documents can give more.">
        <Field label={qa ? "Most Q&A pairs to create" : "Most rows to create"}>
          <NumberInput value={form.maxPairs} min={1} onChange={(v) => set("maxPairs", v)} /></Field>
      </Section>

      <Section title="Train / validation / test split" hint="Most rows teach the model. A few are kept aside to check how well it learned."
        right={<span className={`badge ${total === 100 ? "ok" : "bad"}`}><i />{total === 100 ? "Totals 100%" : `Totals ${total}%, must be 100%`}</span>}>
        <div className="splitbar">{form.split.map((v, i) => (Number(v) > 0 ? <i key={i} style={{ flex: Number(v), background: SPLIT_COLORS[i] }} title={`${SPLIT_INFO[i][0]} ${v}%`}>{Number(v) >= 12 && <span>{SPLIT_INFO[i][0]} {v}%</span>}</i> : null))}</div>
        <div className="split3">
          {SPLIT_INFO.map(([l, what], i) => (
            <div className="split-item" key={l}>
              <span className="inset"><small>{l} %</small><NumberInput value={form.split[i]} min={0} max={100}
                onChange={(v) => set("split", form.split.map((x, k) => (k === i ? v : x)))} /></span>
              <span className="split-what" style={{ "--c": SPLIT_COLORS[i] }}>{what}</span>
            </div>
          ))}
        </div>
      </Section>

      <Fold icon="gear" title="More options" hint="How rows are split, random seed">
        <div className="grid2">
          <Field label="Split by" hint={form.splitBy === "fact" ? "Fact: test rows ask about facts the model trained on, in new words." : "Section: test rows come from whole sections the model never saw."}>
            <Segmented value={form.splitBy} onChange={(v) => set("splitBy", v)} options={[["fact", "Fact"], ["section", "Section"]]} />
          </Field>
          <Field label="Random seed" hint="Same seed and settings give the same split.">
            <NumberInput value={form.seed} onChange={(v) => set("seed", v)} /></Field>
        </div>
      </Fold>
    </>
  );
}

// ---- dataset templates: cards to pick one, and the editor
function Flow({ t }) {
  return <span className="tpl-flow"><span className="tag">{inShort(t)}</span><Icon name="arrow" size={12} /><span className="tag">{outShort(t)}</span><small>{fmtShort(t)}{t.context_in_row ? " · text in the row" : ""}</small></span>;
}

// one short example row per template, from an HR leave-policy document
function exampleRow(t) {
  const instr = (t.instruction || "").replace(/{{\s*\w+\s*}}/g, "…") || "Summarise this text.";
  const inp = { question: "How many leave days do new staff get?", instruction: "List the steps to request leave.", text: `${instr} + a passage of the document`,
    text_start: "“Employees get 25 days of annual…”", custom: "A customer email about leave" }[t.input] || "…";
  const out = { answer: "25 days a year.", completion: "1. Open the HR portal  2. Pick the dates  3. Send to your manager", summary: "Staff get 25 leave days; book them two weeks ahead.",
    json: `{ ${(t.json_fields || []).slice(0, 2).map((f) => `"${f}": …`).join(", ") || "…"} }`, label: (t.labels || [])[0] || "a label",
    rewrite: "Hi team, a quick note on leave…", continuation: "“…leave. Requests go to the manager.”", custom: "Written by your output prompt" }[t.output] || "…";
  return [inp, out];
}
function Example({ t }) {
  const [a, b] = exampleRow(t);
  return (
    <span className="tpl-ex">
      <span><em>{inShort(t)}</em>{a}</span>
      <span><em>{outShort(t)}</em>{b}</span>
    </span>
  );
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
            <Example t={cur} />
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




























//GeneratePage.jsx
// Generate Dataset page: a guided, step-by-step setup with a live summary, and a progress view once a job runs.
// Same API as the original UI (/api/datagen); the form <-> JobRequest logic is in ../model.js.
import { useCallback, useEffect, useLayoutEffect, useRef, useState } from "react";
import { useStepper } from "../stepper.js";
import { API, api, post, upload } from "../api.js";
import { QA_TEMPLATE, T_FORMATS, T_INPUTS, T_OUTPUTS, buildRequest, defaultForm, endpointsForTest, formFromRequest, isQa, visionUsed } from "../model.js";
import { Icon, useToast } from "../ui.jsx";
import { emit, useBus } from "../bus.js";
import { AdvancedStep, DatasetTypeStep, DocumentsStep, MixStep, ModelsStep, SizeStep } from "./steps.jsx";
import Summary from "./Summary.jsx";
import RunProgress from "./RunProgress.jsx";
import "./generate.css";

export const STEPS = [
  { key: "documents", title: "Documents", icon: "doc", lead: "Upload the files the model should learn from, and choose what happens to the pictures inside them." },
  { key: "models", title: "Models", icon: "server", lead: "The teacher model writes and judges the Q&A pairs; the vision model reads images. Both must already be running." },
  { key: "dataset", title: "Dataset type", icon: "layers", lead: "Choose what one training row looks like. Each card shows an example made from an HR leave-policy document." },
  { key: "size", title: "Size & split", icon: "chart", lead: "Choose how many rows to create, and how they are shared between training and testing." },
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
    const m = e.message;
    const key = /endpoint|vision/i.test(m) ? "models" : /split/i.test(m) ? "size" : /question types|template|input|output|label|json|variety|checks/i.test(m) ? "dataset"
      : /decline|paraphrase|general/i.test(m) ? "mix" : "advanced";
    issues.push({ step: STEPS.findIndex((x) => x.key === key), text: m });
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
            {cur.key === "dataset" && <DatasetTypeStep {...stepProps} />}
            {cur.key === "size" && <SizeStep {...stepProps} />}
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
    dataset: form.template?.input === "question" && !form.types.length ? { warn: true, note: "Pick a kind of question" }
      : { ok: true, note: isQa(form.template) ? `Q&A · ${form.types.length} kind${form.types.length > 1 ? "s" : ""}` : form.template.name },
    size: split !== 100 ? { warn: true, note: "Split must total 100" }
      : { ok: true, note: `Up to ${Number(form.maxPairs || 0).toLocaleString()} ${isQa(form.template) ? "pairs" : "rows"}` },
    mix: { ok: true, note: extras ? `${extras} extra${extras > 1 ? "s" : ""} on` : "No extras" },
    advanced: { ok: false, note: "Optional" },
  };
}






























//Summary.jsx
// Right-hand panel of the Generate page: what the job will do, what is still missing, and Start / Preview.
import { inShort, isQa, mixEstimate, outShort, short, visionUsed } from "../model.js";
import { Icon } from "../ui.jsx";
import { MixBar, picMode } from "./steps.jsx";

const QT = { factual: "Factual", explanation: "Explanation", "how-to": "How-to", from_images: "From pictures", comparison: "Comparison" };
const PIC = { describe: "Pictures described by the vision model", ocr: "Pictures: words only (OCR)", ignore: "Pictures skipped" };

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
        <Row icon="doc" label="Documents" onEdit={() => goStep(0)}>{docs || <em>None yet</em>}<span className="sum-sub">{PIC[picMode(form)]}</span></Row>
        <Row icon="server" label="Models" onEdit={() => goStep(1)}>
          <span className="sum-ep"><span className={`dot ${dotOf(ep.teacher, testing)}`} />Teacher <b>{short(form.teacherModel) || "not set"}</b></span>
          <span className="sum-ep"><span className={`dot ${vUsed ? dotOf(ep.vision, testing) || "bad" : "muted"}`} />Vision <b>{vUsed ? short(form.visionModel) || "not set" : "off (OCR only)"}</b></span>
        </Row>
        <Row icon="layers" label="Dataset type" onEdit={() => goStep(2)}>
          {form.template.name} <span className="sum-sub">{inShort(form.template)} → {outShort(form.template)}</span>
          {form.template.input === "question" && <span className="sum-sub">{form.types.map((t) => QT[t] || t).join(", ") || <em>No kinds picked</em>}</span>}
          {!["json", "label", "continuation"].includes(form.template.output) && <span className="sum-sub">{form.answerLength === "short" ? "Short" : "Detailed"} {isQa(form.template) ? "answers" : "outputs"}</span>}
        </Row>
        <Row icon="chart" label="Size & split" onEdit={() => goStep(3)}>
          Up to {Number(form.maxPairs || 0).toLocaleString()} {isQa(form.template) ? "pairs" : "rows"}
          <span className="sum-sub">Train {form.split[0]}% · Validation {form.split[1]}% · Test {form.split[2]}%</span>
        </Row>
        <Row icon="mix" label="Training mix" onEdit={() => goStep(4)}><MixBar mix={mixEstimate(form)} /></Row>
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





















//generate.css
/* Generate Dataset page. Uses the tokens from styles.css (coral theme). */

.gen { display: grid; gap: 20px; max-width: 1360px; margin: 0 auto; }

/* ---- stepper */
.stepper { display: grid; grid-auto-flow: column; grid-auto-columns: minmax(0, 1fr); gap: 6px; padding: 6px; background: var(--surface);
  border: 1px solid var(--line); border-radius: var(--radius-lg); box-shadow: var(--shadow); }
.stp { position: relative; display: flex; align-items: center; gap: 10px; padding: 10px 10px; border: 0; border-radius: 12px; background: none;
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
  .stepper { grid-auto-columns: auto; overflow-x: auto; }
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

/* ---- simpler dataset steps: pictures choice with an example, template example rows, split legend */
.choices.three { grid-template-columns: repeat(3, minmax(0, 1fr)); }
.pic-ex { display: grid; gap: 6px; margin-top: 12px; padding: 12px 14px; border: 1px dashed var(--line-strong); border-radius: 10px; background: var(--surface-2); font-size: 13px; color: var(--muted); }
.pic-ex-h { display: flex; align-items: center; gap: 6px; font-weight: 600; color: var(--ink-2); }
.pic-ex > span:not(.pic-ex-h) { display: grid; grid-template-columns: 96px minmax(0, 1fr); gap: 10px; padding: 3px 6px; border-radius: 6px; transition: background .2s, color .2s; }
.pic-ex > span b { font-weight: 600; color: var(--ink-2); }
.pic-ex > span.on { background: var(--accent-50); color: var(--ink); }
.pic-ex > span.on b { color: var(--accent-600); }

.tpl-ex { display: grid; gap: 4px; margin-top: 4px; padding: 8px 10px; border-radius: 8px; background: var(--surface-2); border: 1px solid var(--line-2); font-size: 12.5px; line-height: 1.35; color: var(--ink-2); }
.tpl-ex > span { display: grid; gap: 1px; }
.tpl-ex > span + span { padding-top: 5px; border-top: 1px dashed var(--line); }
.tpl-ex em { font-style: normal; font-size: 10.5px; font-weight: 700; letter-spacing: .4px; color: var(--accent-600); text-transform: uppercase; }
.tpl-card.on .tpl-ex { background: var(--surface); }

.split3 { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 10px; max-width: 640px; }
.split-item { display: grid; gap: 6px; }
.split-what { display: flex; align-items: center; gap: 6px; font-size: 12.5px; color: var(--muted); }
.split-what::before { content: ""; width: 8px; height: 8px; border-radius: 2px; background: var(--c); flex: none; }
.split3 input { font-family: var(--mono); letter-spacing: -.02em; }
.gen-bd > .fold { margin-top: 4px; }

@media (max-width: 900px) {
  .choices.three, .split3 { grid-template-columns: minmax(0, 1fr); }
}
[data-theme="dark"] .pic-ex > span.on { background: rgba(var(--accent-rgb), .12); }
[data-theme="dark"] .tpl-ex { background: var(--bg); }
