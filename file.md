//src/results/ImageViewer.jsx
// Expanded image view for the Source data drawer: the picked image large on a dark backdrop, its details beside it,
// prev/next between the images of the current list. Closes with the X, Esc or a click on the backdrop.
import { useCallback, useEffect, useRef, useState } from "react";
import { createPortal } from "react-dom";
import { Icon } from "../ui.jsx";
import "./image-viewer.css";

const Svg = ({ children, size = 18 }) => (
  <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.8" strokeLinecap="round" strokeLinejoin="round" aria-hidden="true">{children}</svg>
);
export const ExpandIcon = ({ size = 16 }) => <Svg size={size}><path d="M14 4h6v6M10 20H4v-6M20 4l-7 7M4 20l7-7" /></Svg>;
const ZoomIn = () => <Svg><circle cx="11" cy="11" r="6.5" /><path d="m20 20-4.2-4.2M11 8.5v5M8.5 11h5" /></Svg>;
const ZoomOut = () => <Svg><circle cx="11" cy="11" r="6.5" /><path d="m20 20-4.2-4.2M8.5 11h5" /></Svg>;
const External = () => <Svg><path d="M14 4h6v6M20 4l-9 9" /><path d="M18 14v5a1 1 0 0 1-1 1H5a1 1 0 0 1-1-1V7a1 1 0 0 1 1-1h5" /></Svg>;
const Prev = () => <Svg size={22}><path d="m15 6-6 6 6 6" /></Svg>;

/** images: the list shown in the drawer; index: the open one (null = closed); src(image) gives its URL. */
export default function ImageViewer({ images, index, onIndex, onClose, src }) {
  const open = index != null && images[index];
  const [zoom, setZoom] = useState(false);
  const [loaded, setLoaded] = useState(false);
  const [nat, setNat] = useState(null); // the image's natural size
  const [box, setBox] = useState(null); // the stage's size
  const dlg = useRef(null);
  const stage = useRef(null);
  const n = images.length;
  const go = useCallback((step) => { if (n > 1) onIndex((index + step + n) % n); }, [index, n, onIndex]);

  useEffect(() => { setZoom(false); setLoaded(false); setNat(null); }, [index]);

  // track the stage size, to scale the image to fit it (small images are enlarged, up to 4x)
  useEffect(() => {
    const el = stage.current;
    if (!el) return undefined;
    const ro = new ResizeObserver(([e]) => setBox({ w: e.contentRect.width, h: e.contentRect.height }));
    ro.observe(el);
    return () => ro.disconnect();
  }, [!!open]); // eslint-disable-line react-hooks/exhaustive-deps

  // keys: caught at the window's capture phase, so Esc closes only the viewer and not the drawer under it
  useEffect(() => {
    if (!open) return undefined;
    const h = (e) => {
      if (e.key === "Escape") { e.stopPropagation(); e.preventDefault(); onClose(); }
      else if (e.key === "ArrowLeft") { e.preventDefault(); go(-1); }
      else if (e.key === "ArrowRight") { e.preventDefault(); go(1); }
      else if (e.key === "+" || e.key === "=") setZoom(true);
      else if (e.key === "-") setZoom(false);
    };
    window.addEventListener("keydown", h, true);
    return () => window.removeEventListener("keydown", h, true);
  }, [open, go, onClose]);

  // focus moves into the viewer, and back to what opened it when it closes
  useEffect(() => {
    if (!open) return undefined;
    const back = document.activeElement;
    dlg.current?.focus();
    return () => back?.focus?.();
  }, [!!open]); // eslint-disable-line react-hooks/exhaustive-deps

  if (!open) return null;
  const i = images[index];
  const title = i.figure ? `Figure ${i.figure}` : "Image";
  const url = src(i);
  // fit: as large as the stage allows, never above 4x; zoom: actual size, or twice the fit when the fit is already bigger
  let size;
  if (nat && box) {
    const fit = Math.min(box.w / nat.w, box.h / nat.h, 4);
    const k = zoom ? Math.max(1, fit * 2) : fit;
    size = { width: Math.round(nat.w * k), height: Math.round(nat.h * k) };
  }
  return createPortal(
    <div className="iv">
      {/* a press on the empty backdrop around the image closes the viewer */}
      <div className="iv-dlg" role="dialog" aria-modal="true" aria-label={`${title}, expanded view`} tabIndex={-1} ref={dlg}
        onMouseDown={(e) => e.target.matches(".iv-dlg, .iv-bar, .iv-main, .iv-stage") && onClose()}>
        <div className="iv-bar">
          <div className="iv-t"><b>{title}{i.page ? ` · page ${Math.round(i.page)}` : ""}</b>
            {n > 1 && <span className="iv-count">{index + 1} / {n}</span>}</div>
          <div className="grow" />
          <button type="button" className="iv-btn" onClick={() => setZoom((z) => !z)} aria-pressed={zoom}
            title={zoom ? "Fit to screen (−)" : "Actual size (+)"} aria-label={zoom ? "Fit to screen" : "Actual size"}>{zoom ? <ZoomOut /> : <ZoomIn />}</button>
          <a className="iv-btn" href={url} target="_blank" rel="noreferrer" title="Open in a new tab" aria-label="Open in a new tab"><External /></a>
          <button type="button" className="iv-btn" onClick={onClose} title="Close (Esc)" aria-label="Close"><Icon name="x" size={18} /></button>
        </div>

        <div className="iv-main">
          <div className={`iv-stage${zoom ? " zoom" : ""}`} ref={stage}>
            {!loaded && <span className="iv-spin" aria-hidden="true" />}
            <img key={url} src={url} alt={i.caption || title} className={loaded ? "in" : ""} style={size}
              onLoad={(e) => { setNat({ w: e.currentTarget.naturalWidth || 800, h: e.currentTarget.naturalHeight || 600 }); setLoaded(true); }}
              onClick={() => setZoom((z) => !z)} draggable={false} />
          </div>
          <aside className="iv-info">
            <div className="iv-tags">
              {(i.type || i.source) && <span className="tag">{i.type || i.source}</span>}
              <span className={`tag ${i.status === "read" ? "ok" : "bad"}`}>{i.status === "read" ? `Read · ${i.source === "ocr" ? "OCR" : "vision model"}` : `Skipped${i.reason ? ` · ${i.reason}` : ""}`}</span>
            </div>
            {i.caption && <p><small>Caption</small>{i.caption}</p>}
            {i.visible_text && <p><small>Visible text</small>{i.visible_text}</p>}
            {i.description && <p><small>Description</small>{i.description}</p>}
            <p className="iv-meta"><small>Location</small>{i.heading_path || "(no heading)"}{i.width ? ` · ${i.width}×${i.height}px` : ""}</p>
          </aside>
        </div>

        {n > 1 && <>
          <button type="button" className="iv-nav prev" onClick={() => go(-1)} aria-label="Previous image" title="Previous (←)"><Prev /></button>
          <button type="button" className="iv-nav next" onClick={() => go(1)} aria-label="Next image" title="Next (→)"><Prev /></button>
        </>}
      </div>
    </div>,
    document.body,
  );
}























//src/results/image-viewer.css
/* Expanded image view (ImageViewer.jsx). The backdrop is dark in every theme so the image reads well;
   the details panel and buttons use the theme's tokens, so it follows each palette, light and dark. */

/* thumbnails in the drawer become buttons that open the viewer */
.thumb-btn { position: relative; padding: 0; width: 100%; cursor: zoom-in; font: inherit; color: inherit; }
.thumb-btn:focus-visible { outline: 2px solid var(--accent); outline-offset: 2px; }
.thumb-btn .thumb-x { position: absolute; right: 6px; bottom: 6px; display: grid; place-items: center; width: 26px; height: 26px; border-radius: 7px;
  background: var(--surface); color: var(--ink); border: 1px solid var(--line); box-shadow: 0 2px 6px rgba(0, 0, 0, .12);
  opacity: 0; transform: translateY(2px); transition: opacity .15s, transform .15s; }
.thumb-btn::after { content: ""; position: absolute; inset: 0; background: rgba(var(--accent-rgb), .06); opacity: 0; transition: opacity .15s; }
.thumb-btn:hover .thumb-x, .thumb-btn:focus-visible .thumb-x { opacity: 1; transform: none; }
.thumb-btn:hover::after { opacity: 1; }
.thumb-btn img { transition: transform .25s ease; }
.thumb-btn:hover img { transform: scale(1.03); }
@media (hover: none) { .thumb-btn .thumb-x { opacity: 1; transform: none; } }

/* the viewer */
.iv { position: fixed; inset: 0; z-index: 90; background: rgba(8, 10, 14, .86); backdrop-filter: blur(6px); animation: fade .16s ease-out; }
.iv-dlg { position: absolute; inset: 0; display: flex; flex-direction: column; outline: none; }
.iv-bar { display: flex; align-items: center; gap: 6px; padding: 12px 16px 12px 22px; color: #f3f4f6; }
.iv-t { display: flex; align-items: baseline; gap: 12px; min-width: 0; }
.iv-t b { font-size: 15px; font-weight: 600; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.iv-count { font-size: 12.5px; color: rgba(243, 244, 246, .65); font-variant-numeric: tabular-nums; }
.iv-btn { display: grid; place-items: center; width: 36px; height: 36px; border-radius: 9px; border: 0; background: rgba(255, 255, 255, .08);
  color: #f3f4f6; cursor: pointer; transition: background .15s; text-decoration: none; }
.iv-btn:hover { background: rgba(255, 255, 255, .18); }
.iv-btn[aria-pressed="true"] { background: rgba(var(--accent-rgb), .9); color: #fff; }
.iv-btn:focus-visible, .iv-nav:focus-visible { outline: 2px solid var(--accent-400); outline-offset: 2px; }

.iv-main { flex: 1; min-height: 0; display: grid; grid-template-columns: minmax(0, 1fr) 340px; gap: 16px; padding: 0 76px 22px; }
.iv-stage { position: relative; min-height: 0; display: grid; place-items: center; overflow: hidden; border-radius: 12px; }
.iv-stage > * { grid-area: 1 / 1; }
.iv-stage img { max-width: 100%; max-height: 100%; object-fit: contain; border-radius: 8px; background: #fff; cursor: zoom-in;
  box-shadow: 0 20px 50px -12px rgba(0, 0, 0, .55); opacity: 0; transform: scale(.98); transition: opacity .2s ease, transform .2s ease; user-select: none; }
.iv-stage img.in { opacity: 1; transform: none; }
.iv-stage.zoom { overflow: auto; place-items: safe center; }
.iv-stage.zoom img { max-width: none; max-height: none; cursor: zoom-out; }
.iv-spin { position: absolute; width: 28px; height: 28px; border-radius: 50%; border: 3px solid rgba(255, 255, 255, .2); border-top-color: var(--accent-400);
  animation: iv-spin .8s linear infinite; }
@keyframes iv-spin { to { transform: rotate(360deg); } }

.iv-info { align-self: start; max-height: 100%; min-height: 0; overflow: auto; display: flex; flex-direction: column; gap: 12px; padding: 16px 18px; border-radius: 12px;
  background: var(--surface); color: var(--ink); border: 1px solid var(--line); font-size: 13.5px; box-shadow: var(--shadow-lg); }
.iv-tags { display: flex; gap: 6px; flex-wrap: wrap; }
.iv-info p { margin: 0; color: var(--ink-2); white-space: pre-wrap; overflow-wrap: anywhere; line-height: 1.5; }
.iv-info p small { display: block; font-size: 11.5px; font-weight: 600; text-transform: uppercase; color: var(--faint); letter-spacing: .5px; margin-bottom: 2px; }
.iv-meta { color: var(--muted) !important; font-size: 12.5px; }

.iv-nav { position: absolute; top: 50%; width: 46px; height: 46px; margin-top: -10px; border-radius: 50%; border: 0; display: grid; place-items: center;
  background: rgba(255, 255, 255, .1); color: #fff; cursor: pointer; transition: background .15s, transform .15s; }
.iv-nav:hover { background: rgba(255, 255, 255, .22); }
.iv-nav.prev { left: 16px; }
.iv-nav.next { right: 16px; }
.iv-nav.next svg { transform: rotate(180deg); }
.iv-nav:active { transform: scale(.94); }

@media (max-width: 960px) {
  .iv-main { grid-template-columns: 1fr; grid-template-rows: minmax(0, 1fr) auto; padding: 0 16px 16px; }
  .iv-info { max-height: 34vh; }
  .iv-nav { top: auto; bottom: calc(34vh + 40px); }
}
@media (prefers-reduced-motion: reduce) {
  .iv, .iv-stage img, .thumb-btn img, .thumb-btn .thumb-x { animation: none; transition: none; }
}




















//src/results/SourceDrawer.jsx
// Source data drawer for the Results page: the parsed documents and images, and the knowledge base. Ported from ui-react.
import { useEffect, useRef, useState } from "react";
import { createPortal } from "react-dom";
import { api, imageUrl } from "./source.js";
import { fmtN } from "../model.js";
import { Drawer, Empty, Icon } from "../ui.jsx";
import ImageViewer, { ExpandIcon } from "./ImageViewer.jsx";

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
  const [view, setView] = useState(null); // index of the image open in the expanded view
  const mdRef = useRef(null);
  useEffect(() => {
    api(`/jobs/${jobId}/documents`).then((d) => { setDocs(d); setWait(false); setSel((s) => s || d.find((x) => x.status === "ok")?.doc_id || d[0]?.doc_id); })
      .catch((e) => { if (e.status === 409) setWait(true); });
  }, [jobId, job?.status]);
  const d = docs?.find((x) => x.doc_id === sel);
  useEffect(() => setView(null), [sel, imgTab]);
  useEffect(() => {
    setMd("");
    if (d?.status === "ok") api(`/jobs/${jobId}/documents/${encodeURIComponent(d.doc_id)}/markdown`).then(setMd).catch(() => setMd("Markdown is not ready yet."));
  }, [jobId, d?.doc_id, d?.status]);

  if (wait) return <Empty icon="doc" title="The documents are not parsed yet" />;
  if (!docs) return <div className="muted">Loading…</div>;
  const read = d ? d.images.filter((i) => i.status === "read") : [], skipped = d ? d.images.filter((i) => i.status !== "read") : [];
  const vision = job?.request?.parse?.vision_analysis && job?.request?.endpoints?.vision;
  const imgs = imgTab === "read" ? read : skipped;
  const shown = imgs.filter((i) => i.file); // the images the expanded view steps through
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
                {i.file
                  ? <button type="button" className="thumb thumb-btn" onClick={() => setView(shown.indexOf(i))} title="View larger" aria-label={`View ${i.figure ? `figure ${i.figure}` : "image"} larger`}>
                      <img loading="lazy" alt={`Figure ${i.figure}`} src={imageUrl(jobId, i.file)} /><span className="thumb-x"><ExpandIcon size={14} /></span></button>
                  : <div className="thumb" />}
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
            <ImageViewer images={shown} index={view} onIndex={setView} onClose={() => setView(null)} src={(i) => imageUrl(jobId, i.file)} />
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
