// ═══════════════════════════════════════════════
// pages/UploadInfer/WorkspacePanel.tsx
// LectureAI · Step-3 Workspace Result panel
// ═══════════════════════════════════════════════
import React, { useState, useMemo, useRef, useEffect, useCallback, forwardRef, useImperativeHandle } from 'react';
import { useTranslation } from 'react-i18next';
import { marked } from 'marked';
import {
    ReactFlow, Background, Controls, MiniMap, Handle, Position, applyNodeChanges,
    type Node as RFNode, type Edge as RFEdge, type NodeChange,
} from '@xyflow/react';
import '@xyflow/react/dist/style.css';
import dagre from 'dagre';
import api from '../../services/api';
import type { FileResult, WordCloudData, ImportanceComplexityData, GlossaryData, KeywordInsights, TimelineData, TimelineTimeUnit } from './UploadInfer';
import styles from './WorkspacePanel.module.scss';

marked.setOptions({ breaks: false, gfm: true });

interface Props {
    step2Visible?: boolean;
    step2Minimized?: boolean;
    fileResult: FileResult | null;
    fileLoading: boolean;
    activeFileId: number | null;
    onResultUpdate?: (patch: Partial<Pick<FileResult, 'summary' | 'keywords'>>) => void;
}

// TABS labels are now driven by i18n — see tabLabels() inside WorkspacePanel
const TAB_IDS = ['summary', 'keywords', 'assessment', 'shortAnswer', 'trueFalse', 'timestampedSummary', 'keywordInsights'] as const;
type TabId = typeof TAB_IDS[number];

// Exposed to the parent so the guided tour can drive which tab is showing
// without lifting activeTab into Redux just for that one use.
export interface WorkspacePanelHandle {
    setTab: (id: TabId) => void;
}


interface FaqItem {
    question: string;
    options: Record<string, string>;
    correct_answer: string;
    explanation: string;
}

function parseFaq(raw: string): FaqItem[] {
    if (!raw || raw === '[]') return [];
    try {
        // eslint-disable-next-line no-eval
        const result = eval(raw);
        if (Array.isArray(result)) return result as FaqItem[];
        return [];
    } catch {
        // Fallback: try direct JSON parse (if API returns valid JSON)
        try { return JSON.parse(raw) as FaqItem[]; } catch { }
        return [];
    }
}

// ── Permissive parser for the newer python-repr-style fields (short_answer,
//    true_false, timestamped_summary). Handles True/False/None literals that
//    plain eval() can't, and tries strict JSON first since that's cheaper
//    and safer whenever the backend does send valid JSON. ──
function parsePyList<T>(raw: string): T[] {
    if (!raw || raw === '[]') return [];
    try {
        const result = JSON.parse(raw);
        if (Array.isArray(result)) return result as T[];
    } catch { /* not valid JSON — fall through to python-ish eval */ }
    try {
        const normalized = raw
            .replace(/\bTrue\b/g, 'true')
            .replace(/\bFalse\b/g, 'false')
            .replace(/\bNone\b/g, 'null');
        // eslint-disable-next-line no-eval
        const result = eval('(' + normalized + ')');
        if (Array.isArray(result)) return result as T[];
    } catch { /* give up */ }
    return [];
}

interface ShortAnswerItem { question: string; answer: string; }
interface TrueFalseItem { statement: string; is_true: boolean; explanation: string; }
interface TimestampSegment { start_time: string; end_time: string; summary: string; }

const CHIP_COLORS = [
    'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
    'linear-gradient(135deg, #f093fb 0%, #f5576c 100%)',
    'linear-gradient(135deg, #4facfe 0%, #00f2fe 100%)',
    'linear-gradient(135deg, #43e97b 0%, #38f9d7 100%)',
    'linear-gradient(135deg, #fa709a 0%, #fee140 100%)',
    'linear-gradient(135deg, #30cfd0 0%, #330867 100%)',
    'linear-gradient(135deg, #a8edea 0%, #fed6e3 100%)',
    'linear-gradient(135deg, #ff9a56 0%, #ff6a88 100%)',
    'linear-gradient(135deg, #ffecd2 0%, #fcb69f 100%)',
    'linear-gradient(135deg, #a1c4fd 0%, #c2e9fb 100%)',
];

function seededColorIndex(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) hash = (hash * 31 + str.charCodeAt(i)) >>> 0;
    return hash % CHIP_COLORS.length;
}

// ── Copy / Download helpers ───────────────────────────────
function copyText(text: string) {
    if (navigator.clipboard) { navigator.clipboard.writeText(text).catch(() => fallbackCopy(text)); }
    else fallbackCopy(text);
}
function fallbackCopy(text: string) {
    const ta = document.createElement('textarea');
    ta.value = text; ta.style.cssText = 'position:fixed;top:0;left:0;opacity:0';
    document.body.appendChild(ta); ta.select();
    try { document.execCommand('copy'); } catch { }
    document.body.removeChild(ta);
}
function downloadFile(content: string, filename: string, mime: string) {
    const blob = new Blob([content], { type: mime });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = filename;
    document.body.appendChild(a); a.click();
    document.body.removeChild(a); URL.revokeObjectURL(url);
}
function wrapHtml(body: string, title: string): string {
    return `<!DOCTYPE html><html lang="en"><head><meta charset="UTF-8"><title>${title}</title><style>body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;max-width:900px;margin:40px auto;padding:20px;background:#f8f9fa;color:#333}</style></head><body>${body}</body></html>`;
}

// ── Format helpers ────────────────────────────────────────
type Formatted = { content: string; filename: string; mime: string };

function formatSummary(summary: string, fmt: string, base = 'summary'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    if (fmt === 'html') return { content: wrapHtml(marked.parse(summary) as string, 'Summary'), filename: `${base}_summary_${ts}.html`, mime: 'text/html' };
    if (fmt === 'json') return { content: JSON.stringify({ type: 'summary', content: summary, timestamp: new Date().toISOString() }, null, 2), filename: `${base}_summary_${ts}.json`, mime: 'application/json' };
    if (fmt === 'md') return { content: summary, filename: `${base}_summary_${ts}.md`, mime: 'text/markdown' };
    return { content: summary, filename: `${base}_summary_${ts}.txt`, mime: 'text/plain' };
}

function formatKeywords(keywords: string[], fmt: string, base = 'keywords'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    if (fmt === 'html') return { content: wrapHtml(`<div style="display:flex;flex-wrap:wrap;gap:10px">${keywords.map(k => `<span style="padding:6px 14px;border-radius:20px;background:linear-gradient(135deg,#8b5cf6,#a78bfa);color:#fff;font-weight:600">${k}</span>`).join('')}</div>`, 'Keywords'), filename: `${base}_keywords_${ts}.html`, mime: 'text/html' };
    if (fmt === 'json') return { content: JSON.stringify({ type: 'keywords', keywords, timestamp: new Date().toISOString() }, null, 2), filename: `${base}_keywords_${ts}.json`, mime: 'application/json' };
    if (fmt === 'md') return { content: '# Keywords\n\n' + keywords.map(k => `- ${k}`).join('\n'), filename: `${base}_keywords_${ts}.md`, mime: 'text/markdown' };
    return { content: keywords.join('\n'), filename: `${base}_keywords_${ts}.txt`, mime: 'text/plain' };
}

function formatAssessment(faqRaw: string, fmt: string, base = 'assessment'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    const items = parseFaq(faqRaw);
    if (fmt === 'json') return { content: JSON.stringify(items, null, 2), filename: `${base}_assessment_${ts}.json`, mime: 'application/json' };
    if (fmt === 'html') {
        const body = items.map((q, i) =>
            `<div style="background:#fff;border-radius:12px;padding:24px;margin-bottom:20px;box-shadow:0 2px 8px rgba(0,0,0,.08);border-left:4px solid #8b5cf6">
             <div style="font-weight:700;font-size:16px;margin-bottom:14px"><span style="background:linear-gradient(135deg,#8b5cf6,#a78bfa);color:#fff;padding:3px 10px;border-radius:6px;margin-right:10px;font-size:13px">Q${i + 1}</span>${q.question}</div>
             ${Object.entries(q.options).map(([k, v]) => `<div style="padding:10px 14px;margin:6px 0;border-radius:8px;border:1px solid ${k === q.correct_answer ? '#10b981' : '#e5e7eb'};background:${k === q.correct_answer ? 'rgba(16,185,129,.08)' : '#f9fafb'}"><b style="color:${k === q.correct_answer ? '#10b981' : '#8b5cf6'}">${k}.</b> ${v}${k === q.correct_answer ? ' <span style="background:#10b981;color:#fff;padding:1px 7px;border-radius:4px;font-size:11px">✓</span>' : ''}</div>`).join('')}
             <div style="margin-top:14px;padding:12px 16px;background:rgba(139,92,246,.06);border-left:3px solid #8b5cf6;border-radius:0 8px 8px 0;font-size:13px"><b style="color:#8b5cf6;font-size:11px;text-transform:uppercase">Explanation</b><br/>${q.explanation}</div>
             </div>`
        ).join('');
        return { content: wrapHtml(body, 'Assessment Questions'), filename: `${base}_assessment_${ts}.html`, mime: 'text/html' };
    }
    // txt
    const txt = items.map((q, i) =>
        `Q${i + 1}. ${q.question}\n` +
        Object.entries(q.options).map(([k, v]) => `  ${k}. ${v}${k === q.correct_answer ? ' ✓' : ''}`).join('\n') +
        `\n\nAnswer: ${q.correct_answer}\nExplanation: ${q.explanation}`
    ).join('\n\n' + '─'.repeat(60) + '\n\n');
    return { content: txt, filename: `${base}_assessment_${ts}.txt`, mime: 'text/plain' };
}

function formatShortAnswer(raw: string, fmt: string, base = 'short_answer'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    const items = parsePyList<ShortAnswerItem>(raw);
    if (fmt === 'json') return { content: JSON.stringify(items, null, 2), filename: `${base}_${ts}.json`, mime: 'application/json' };
    if (fmt === 'html') {
        const body = items.map((item, i) =>
            `<div style="background:#fff;border-radius:12px;padding:20px;margin-bottom:16px;box-shadow:0 2px 8px rgba(0,0,0,.08);border-left:4px solid #4facfe">
             <div style="font-weight:700;font-size:15px;margin-bottom:10px"><span style="background:#4facfe;color:#fff;padding:3px 10px;border-radius:6px;margin-right:10px;font-size:13px">Q${i + 1}</span>${item.question}</div>
             <div style="padding:10px 14px;background:rgba(79,172,254,.08);border-radius:8px;font-size:13px"><b style="color:#4facfe">Answer:</b> ${item.answer}</div>
             </div>`
        ).join('');
        return { content: wrapHtml(body, 'Short Answer Questions'), filename: `${base}_${ts}.html`, mime: 'text/html' };
    }
    const txt = items.map((item, i) => `Q${i + 1}. ${item.question}\nA: ${item.answer}`).join('\n\n' + '─'.repeat(60) + '\n\n');
    return { content: txt, filename: `${base}_${ts}.txt`, mime: 'text/plain' };
}

function formatTrueFalse(raw: string, fmt: string, base = 'true_false'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    const items = parsePyList<TrueFalseItem>(raw);
    if (fmt === 'json') return { content: JSON.stringify(items, null, 2), filename: `${base}_${ts}.json`, mime: 'application/json' };
    if (fmt === 'html') {
        const body = items.map((item, i) =>
            `<div style="background:#fff;border-radius:12px;padding:20px;margin-bottom:16px;box-shadow:0 2px 8px rgba(0,0,0,.08);border-left:4px solid #43e97b">
             <div style="font-weight:700;font-size:15px;margin-bottom:10px"><span style="background:#43e97b;color:#fff;padding:3px 10px;border-radius:6px;margin-right:10px;font-size:13px">${i + 1}</span>${item.statement}</div>
             <div style="padding:10px 14px;background:rgba(67,233,123,.08);border-radius:8px;font-size:13px"><b style="color:#43e97b">Answer:</b> ${item.is_true ? 'True' : 'False'}</div>
             <div style="margin-top:10px;font-size:13px;color:#666"><b>Explanation:</b> ${item.explanation}</div>
             </div>`
        ).join('');
        return { content: wrapHtml(body, 'True / False Questions'), filename: `${base}_${ts}.html`, mime: 'text/html' };
    }
    const txt = items.map((item, i) => `${i + 1}. ${item.statement}\nAnswer: ${item.is_true ? 'True' : 'False'}\nExplanation: ${item.explanation}`).join('\n\n' + '─'.repeat(60) + '\n\n');
    return { content: txt, filename: `${base}_${ts}.txt`, mime: 'text/plain' };
}

function formatTimestampedSummary(raw: string, fmt: string, base = 'timestamped_summary'): Formatted {
    const ts = new Date().toISOString().slice(0, 10);
    const items = parsePyList<TimestampSegment>(raw);
    if (fmt === 'json') return { content: JSON.stringify(items, null, 2), filename: `${base}_${ts}.json`, mime: 'application/json' };
    if (fmt === 'md') return { content: items.map(s => `### ${s.start_time} – ${s.end_time}\n\n${s.summary}`).join('\n\n'), filename: `${base}_${ts}.md`, mime: 'text/markdown' };
    const txt = items.map(s => `[${s.start_time} – ${s.end_time}]\n${s.summary}`).join('\n\n');
    return { content: txt, filename: `${base}_${ts}.txt`, mime: 'text/plain' };
}


const SUMMARY_FMTS = [{ k: 'txt', l: 'Text' }, { k: 'md', l: 'Markdown' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const KEYWORDS_FMTS = [{ k: 'txt', l: 'Text' }, { k: 'md', l: 'Markdown' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const ASSESSMENT_FMTS = [{ k: 'txt', l: 'Plain Text' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const SHORT_ANSWER_FMTS = [{ k: 'txt', l: 'Plain Text' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const TRUE_FALSE_FMTS = [{ k: 'txt', l: 'Plain Text' }, { k: 'html', l: 'HTML' }, { k: 'json', l: 'JSON' }];
const TIMESTAMPED_SUMMARY_FMTS = [{ k: 'txt', l: 'Text' }, { k: 'md', l: 'Markdown' }, { k: 'json', l: 'JSON' }];

// ── ActionBtn ─────────────────────────────────────────────
const ActionBtn: React.FC<{ title: string; onClick: () => void; active?: boolean; children: React.ReactNode }> = ({ title, onClick, active, children }) => (
    <button className={`${styles.actionBtn} ${active ? styles.actionBtnActive : ''}`} title={title} onClick={e => { e.stopPropagation(); onClick(); }}>
        {children}
    </button>
);

// ── FormatIcon ────────────────────────────────────────────
// Small colored glyph rendered next to each format option in the
// Copy / Download dropdowns. Each format has a recognisable hue.
const FORMAT_ICONS: Record<string, { color: string; bg: string; node: React.ReactNode }> = {
    txt: {
        color: '#64748b',
        bg: 'rgba(100, 116, 139, 0.12)',
        node: (
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <path d="M3 2.5h6.5L13 6v7.5A1 1 0 0112 14.5H3a1 1 0 01-1-1v-10a1 1 0 011-1z" />
                <path d="M9 2.5V6h4" />
                <path d="M5 9h6M5 11.5h4" />
            </svg>
        ),
    },
    md: {
        color: '#3b82f6',
        bg: 'rgba(59, 130, 246, 0.14)',
        node: (
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <rect x="1.5" y="3.5" width="13" height="9" rx="1.5" />
                <path d="M4 10.5V6l1.75 2.5L7.5 6v4.5" />
                <path d="M10.25 6v4.5M8.75 9l1.5 1.5L11.75 9" />
            </svg>
        ),
    },
    html: {
        color: '#e34f26',
        bg: 'rgba(227, 79, 38, 0.14)',
        node: (
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <path d="M2.5 2l1 12 4.5 1.3L12.5 14l1-12z" />
                <path d="M5 5.5h6L10.6 11l-2.6.8L5.4 11l-.15-2" />
            </svg>
        ),
    },
    json: {
        color: '#f59e0b',
        bg: 'rgba(245, 158, 11, 0.14)',
        node: (
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor"
                strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                <path d="M6 2.5C4.5 2.5 4 3.5 4 5v1.5C4 7.5 3 8 2 8c1 0 2 .5 2 1.5V11c0 1.5.5 2.5 2 2.5" />
                <path d="M10 2.5c1.5 0 2 1 2 2.5v1.5C12 7.5 13 8 14 8c-1 0-2 .5-2 1.5V11c0 1.5-.5 2.5-2 2.5" />
            </svg>
        ),
    },
};

const FormatIcon: React.FC<{ fmt: string }> = ({ fmt }) => {
    const def = FORMAT_ICONS[fmt];
    if (!def) return null;
    return (
        <span
            className={styles.fmtIcon}
            style={{ color: def.color, background: def.bg }}
            aria-hidden="true"
        >
            {def.node}
        </span>
    );
};

// ── Dropdown ──────────────────────────────────────────────
const Dropdown: React.FC<{
    icon: React.ReactNode; title: string; label: string;
    fmts: { k: string; l: string }[];
    onSelect: (k: string) => void; active?: boolean;
}> = ({ icon, title, label, fmts, onSelect, active }) => {
    const [open, setOpen] = useState(false);
    const ref = useRef<HTMLDivElement>(null);
    useEffect(() => {
        const h = (e: MouseEvent) => { if (ref.current && !ref.current.contains(e.target as Node)) setOpen(false); };
        document.addEventListener('mousedown', h); return () => document.removeEventListener('mousedown', h);
    }, []);
    return (
        <div className={styles.dropdownWrap} ref={ref}>
            <ActionBtn title={title} onClick={() => setOpen(o => !o)} active={open || active}>{icon}</ActionBtn>
            {open && (
                <div className={styles.dropdown}>
                    <div className={styles.dropdownLabel}>{label}</div>
                    {fmts.map(f => (
                        <button
                            key={f.k}
                            className={styles.dropdownItem}
                            onClick={() => { onSelect(f.k); setOpen(false); }}
                        >
                            <FormatIcon fmt={f.k} />
                            <span className={styles.dropdownItemLabel}>{f.l}</span>
                        </button>
                    ))}
                </div>
            )}
        </div>
    );
};

// ── Icons ─────────────────────────────────────────────────
const IcoEdit = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M7 2.5H3.5A1.5 1.5 0 002 4v8.5A1.5 1.5 0 003.5 14H12a1.5 1.5 0 001.5-1.5V9" />
        <path d="M11.5 1.5a1.414 1.414 0 012 2L8 9l-2.5.5.5-2.5 5.5-5.5z" />
    </svg>
);
const IcoCopy = ({ success }: { success: boolean }) => success ? (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" className={styles.successIcon}><path d="M3 8l3.5 3.5L13 4" /></svg>
) : (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <rect x="5" y="5" width="8" height="9" rx="1.5" />
        <path d="M10 5V4a1 1 0 00-1-1H4a1.5 1.5 0 00-1.5 1.5V11a1 1 0 001 1h1" />
    </svg>
);
const IcoDownload = () => (
    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
        <path d="M8 2v8M5 7l3 3 3-3" />
        <path d="M2 12v1.5A1.5 1.5 0 003.5 15h9a1.5 1.5 0 001.5-1.5V12" />
    </svg>
);

// ── TabToolbar: edit + copy + download ───────────────────
const TabToolbar: React.FC<{
    onEdit?: () => void;
    fmts: { k: string; l: string }[];
    onCopy: (k: string) => void;
    onDownload: (k: string) => void;
    copied: boolean;
}> = ({ onEdit, fmts, onCopy, onDownload, copied }) => {
    const { t } = useTranslation();
    return (
        <div className={styles.tabToolbar}>
            {onEdit && <ActionBtn title={t('uploadInfer.workspacePanel.edit')} onClick={onEdit}><IcoEdit /></ActionBtn>}
            <Dropdown icon={<IcoCopy success={copied} />} title={t('uploadInfer.workspacePanel.copy')} label={t('uploadInfer.workspacePanel.copyAs')} fmts={fmts} onSelect={onCopy} active={copied} />
            <Dropdown icon={<IcoDownload />} title={t('uploadInfer.workspacePanel.download')} label={t('uploadInfer.workspacePanel.downloadAs')} fmts={fmts} onSelect={onDownload} />
        </div>
    );
};

// ── Stable keyword chip ───────────────────────────────────
const KeywordChip: React.FC<{ kw: string; isNew: boolean }> = ({ kw, isNew }) => (
    <span className={`${styles.keywordPill} ${isNew ? styles.keywordPillNew : ''}`}
        style={{ background: CHIP_COLORS[seededColorIndex(kw)] }}>
        {kw}
    </span>
);

const ChipGrid: React.FC<{ kws: string[]; prevSet?: Set<string> }> = ({ kws, prevSet }) => {
    const { t } = useTranslation();
    return kws.length > 0 ? (
        <div className={styles.keywordGrid}>
            {kws.map(kw => <KeywordChip key={kw} kw={kw} isNew={prevSet ? !prevSet.has(kw) : false} />)}
        </div>
    ) : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noKeywords')}</div>;
};

// ── Tab: Summary ──────────────────────────────────────────
const TabSummary: React.FC<{ summary: string; fileId: number; onSaved: (s: string) => void }> = ({ summary, fileId, onSaved }) => {
    const { t } = useTranslation();
    const [editing, setEditing] = useState(false);
    const [draft, setDraft] = useState(summary);
    const [saving, setSaving] = useState(false);
    const [copied, setCopied] = useState(false);

    const html = useMemo(() => marked.parse(draft) as string, [draft]);
    const viewHtml = useMemo(() => marked.parse(summary) as string, [summary]);

    const handleSave = async () => {
        setSaving(true);
        try { await api.post('/files/update', { fileID: String(fileId), summary: draft }); onSaved(draft); setEditing(false); }
        finally { setSaving(false); }
    };
    const handleCopy = (fmt: string) => { copyText(formatSummary(summary, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatSummary(summary, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!summary && !editing) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noSummary')}</div>;

    return (
        <div className={`${styles.tabContent} ${editing ? styles.editMode : ''}`}>
            {!editing ? (
                <>
                    <TabToolbar onEdit={() => { setDraft(summary); setEditing(true); }} fmts={SUMMARY_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
                    <div className={styles.summaryMd} dangerouslySetInnerHTML={{ __html: viewHtml }} />
                </>
            ) : (
                <div className={styles.editLayout}>
                    <div className={styles.editCol}>
                        <div className={styles.editColHeader}><span className={styles.editColLabel}>{t('uploadInfer.workspacePanel.editColPreview')}</span><span className={styles.editColTag}>{t('uploadInfer.workspacePanel.markdownRenderer')}</span></div>
                        <div className={styles.editPreview}><div className={styles.summaryMd} dangerouslySetInnerHTML={{ __html: html }} /></div>
                    </div>
                    <div className={styles.editCol}>
                        <div className={styles.editColHeader}><span className={styles.editColLabel}>{t('uploadInfer.workspacePanel.editColEdit')}</span><span className={styles.editColTag}>{t('uploadInfer.workspacePanel.markdownTag')}</span></div>
                        <textarea className={styles.editTextarea} value={draft} onChange={e => setDraft(e.target.value)} spellCheck={false} />
                    </div>
                    <div className={styles.editFooter}>
                        <button className={styles.cancelBtn} onClick={() => setEditing(false)} disabled={saving}>{t('uploadInfer.workspacePanel.cancel')}</button>
                        <button className={styles.saveBtn} onClick={handleSave} disabled={saving}>
                            {saving ? <><span className={styles.inlineSpinner} />{t('uploadInfer.workspacePanel.saving')}</> : t('uploadInfer.workspacePanel.save')}
                        </button>
                    </div>
                </div>
            )}
        </div>
    );
};

// ── Tab: Keywords ─────────────────────────────────────────
const TabKeywords: React.FC<{ keywords: string[]; fileId: number; onSaved: (kws: string[]) => void }> = ({ keywords, fileId, onSaved }) => {
    const { t } = useTranslation();
    const [editing, setEditing] = useState(false);
    const [draft, setDraft] = useState(keywords.join('\n'));
    const [saving, setSaving] = useState(false);
    const [copied, setCopied] = useState(false);

    // prevSetSnapshot holds the keyword set from the PREVIOUS render of ChipGrid
    // so only truly new chips get the pop animation
    const prevSetSnapshot = useRef<Set<string>>(new Set(keywords));

    const draftKeywords = useMemo(() => draft.split('\n').map(k => k.trim()).filter(Boolean), [draft]);

    // After each render, schedule an update of the snapshot so the *next*
    // render can compare against what's currently visible
    useEffect(() => {
        const id = setTimeout(() => { prevSetSnapshot.current = new Set(draftKeywords); }, 0);
        return () => clearTimeout(id);
    }, [draftKeywords]);

    const handleSave = async () => {
        setSaving(true);
        try { await api.post('/files/update', { fileID: String(fileId), keywords: draftKeywords }); onSaved(draftKeywords); setEditing(false); }
        finally { setSaving(false); }
    };
    const handleCopy = (fmt: string) => { copyText(formatKeywords(keywords, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatKeywords(keywords, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!keywords.length && !editing) return <div className={styles.tabEmpty}>No keywords available.</div>;

    return (
        <div className={`${styles.tabContent} ${editing ? styles.editMode : ''}`}>
            {!editing ? (
                <>
                    <TabToolbar onEdit={() => { setDraft(keywords.join('\n')); prevSetSnapshot.current = new Set(keywords); setEditing(true); }} fmts={KEYWORDS_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
                    <ChipGrid kws={keywords} />
                </>
            ) : (
                <div className={styles.editLayout}>
                    <div className={styles.editCol}>
                        <div className={styles.editColHeader}><span className={styles.editColLabel}>{t('uploadInfer.workspacePanel.editColPreview')}</span><span className={styles.editColTag}>{t('uploadInfer.workspacePanel.chipView')}</span></div>
                        <div className={styles.editPreview}>
                            <ChipGrid kws={draftKeywords} prevSet={prevSetSnapshot.current} />
                        </div>
                    </div>
                    <div className={styles.editCol}>
                        <div className={styles.editColHeader}><span className={styles.editColLabel}>{t('uploadInfer.workspacePanel.editColEdit')}</span><span className={styles.editColTag}>{t('uploadInfer.workspacePanel.onePerLine')}</span></div>
                        <textarea className={styles.editTextarea} value={draft} onChange={e => setDraft(e.target.value)} spellCheck={false} placeholder={t('uploadInfer.workspacePanel.keywordPlaceholder')} />
                    </div>
                    <div className={styles.editFooter}>
                        <button className={styles.cancelBtn} onClick={() => setEditing(false)} disabled={saving}>{t('uploadInfer.workspacePanel.cancel')}</button>
                        <button className={styles.saveBtn} onClick={handleSave} disabled={saving}>
                            {saving ? <><span className={styles.inlineSpinner} />{t('uploadInfer.workspacePanel.saving')}</> : t('uploadInfer.workspacePanel.save')}
                        </button>
                    </div>
                </div>
            )}
        </div>
    );
};

// ── Tab: Assessment ───────────────────────────────────────
type AssessMode = 'mcq' | 'all';

const TabAssessment: React.FC<{ faq: string }> = ({ faq }) => {
    const { t } = useTranslation();
    const items = useMemo(() => parseFaq(faq), [faq]);

    // Always default to MCQ; reset when faq changes (new file)
    const [mode, setMode] = useState<AssessMode>('mcq');
    const [current, setCurrent] = useState(0);
    const [selected, setSelected] = useState<string | null>(null);
    const [revealed, setRevealed] = useState(false);
    const [complete, setComplete] = useState(false);
    const [copied, setCopied] = useState(false);

    const prevFaq = useRef(faq);
    useEffect(() => {
        if (faq !== prevFaq.current) {
            prevFaq.current = faq;
            setMode('mcq');
            setCurrent(0); setSelected(null); setRevealed(false); setComplete(false);
        }
    }, [faq]);

    const handleCopy = (fmt: string) => { copyText(formatAssessment(faq, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatAssessment(faq, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!items.length) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noQuestions')}</div>;

    // ── MCQ helpers ───────────────────────────────────────
    const q = items[current];
    const isLast = current === items.length - 1;
    const progress = Math.round(((current + (complete ? 1 : 0)) / items.length) * 100);

    const handleSelect = (key: string) => { if (revealed) return; setSelected(key); setRevealed(true); };
    const handleNext = () => { if (isLast) setComplete(true); else { setCurrent(c => c + 1); setSelected(null); setRevealed(false); } };
    const handleRestart = () => { setCurrent(0); setSelected(null); setRevealed(false); setComplete(false); };
    const getOptClass = (key: string) => {
        if (!revealed) return selected === key ? styles.faqOptSelected : '';
        if (key === q.correct_answer && selected === key) return styles.faqOptCorrect;
        if (key === q.correct_answer) return styles.faqOptCorrectAlt;
        if (key === selected) return styles.faqOptWrong;
        return '';
    };

    return (
        <div className={styles.tabContent}>
            {/* Toolbar row: mode tabs left, copy/download right */}
            <div className={styles.assessHeader}>
                <div className={styles.assessModeTabs}>
                    <button
                        className={`${styles.assessModeTab} ${mode === 'mcq' ? styles.assessModeTabActive : ''}`}
                        onClick={() => setMode('mcq')}
                    >
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                            <circle cx="8" cy="8" r="6" />
                            <path d="M6 8.5l1.5 1.5L10.5 6" />
                        </svg>
                        MCQ
                    </button>
                    <button
                        className={`${styles.assessModeTab} ${mode === 'all' ? styles.assessModeTabActive : ''}`}
                        onClick={() => setMode('all')}
                    >
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                            <path d="M2 4h12M2 8h12M2 12h8" />
                        </svg>
                        View All
                    </button>
                </div>
                <TabToolbar fmts={ASSESSMENT_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
            </div>

            {/* ── MCQ mode ── */}
            {mode === 'mcq' && (
                complete ? (
                    <div className={styles.assessComplete}>
                        <div className={styles.assessCompleteIcon}>
                            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <path d="M9 12.75L11.25 15 15 9.75M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
                            </svg>
                        </div>
                        <div className={styles.assessCompleteTitle}>{t('uploadInfer.workspacePanel.assessCompleteTitle')}</div>
                        <div className={styles.assessCompleteDesc}>{t('uploadInfer.workspacePanel.assessCompleteDesc', { count: items.length })}</div>
                        <button className={styles.assessRestartBtn} onClick={handleRestart}>{t('uploadInfer.workspacePanel.restart')}</button>
                    </div>
                ) : (
                    <>
                        <div className={styles.assessProgress}>
                            <div className={styles.assessProgressTrack}>
                                <div className={styles.assessProgressFill} style={{ width: `${progress}%` }} />
                            </div>
                            <span className={styles.assessProgressLabel}>{current + 1} / {items.length}</span>
                        </div>
                        <div className={styles.faqCard}>
                            <div className={styles.faqQ}>
                                <span className={styles.faqNum}>Q{current + 1}</span>
                                {q.question}
                            </div>
                            <div className={styles.faqOptions}>
                                {Object.entries(q.options).map(([key, val]) => (
                                    <button key={key} className={`${styles.faqOpt} ${getOptClass(key)}`} onClick={() => handleSelect(key)} disabled={revealed}>
                                        <span className={styles.faqOptKey}>{key}</span>
                                        <span className={styles.faqOptVal}>{val}</span>
                                        {revealed && key === q.correct_answer && <svg className={styles.faqCheckIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M3 8l3.5 3.5L13 4" /></svg>}
                                        {revealed && key === selected && key !== q.correct_answer && <svg className={styles.faqXIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M4 4l8 8M12 4l-8 8" /></svg>}
                                    </button>
                                ))}
                            </div>
                            {revealed && (
                                <div className={styles.faqExplain}>
                                    <div className={styles.faqExplainLabel}>
                                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" /><path d="M8 7v4M8 5.5v.5" /></svg>
                                        Explanation
                                    </div>
                                    <p className={styles.faqExplainText}>{q.explanation}</p>
                                </div>
                            )}
                        </div>
                        {revealed && (
                            <button className={`${styles.assessNextBtn} ${isLast ? styles.assessDoneBtn : ''}`} onClick={handleNext}>
                                {isLast ? t('uploadInfer.workspacePanel.finish') : t('uploadInfer.workspacePanel.next')}
                                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round">
                                    {isLast ? <path d="M3 8l3.5 3.5L13 4" /> : <path d="M3 8h10M9 4l4 4-4 4" />}
                                </svg>
                            </button>
                        )}
                    </>
                )
            )}

            {/* ── View All mode ── */}
            {mode === 'all' && (
                <div className={styles.viewAllList}>
                    {items.map((item, idx) => (
                        <div key={idx} className={styles.viewAllCard}>
                            {/* Question */}
                            <div className={styles.viewAllQ}>
                                <span className={styles.faqNum}>Q{idx + 1}</span>
                                {item.question}
                            </div>

                            {/* Options */}
                            <div className={styles.faqOptions}>
                                {Object.entries(item.options).map(([key, val]) => (
                                    <div
                                        key={key}
                                        className={`${styles.faqOpt} ${key === item.correct_answer ? styles.faqOptCorrectAlt : styles.viewAllOptNeutral}`}
                                    >
                                        <span className={`${styles.faqOptKey} ${key === item.correct_answer ? styles.faqOptKeyCorrect : ''}`}>{key}</span>
                                        <span className={styles.faqOptVal}>{val}</span>
                                        {key === item.correct_answer && (
                                            <svg className={styles.faqCheckIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round">
                                                <path d="M3 8l3.5 3.5L13 4" />
                                            </svg>
                                        )}
                                    </div>
                                ))}
                            </div>

                            {/* Explanation */}
                            <div className={styles.faqExplain}>
                                <div className={styles.faqExplainLabel}>
                                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round">
                                        <circle cx="8" cy="8" r="6" /><path d="M8 7v4M8 5.5v.5" />
                                    </svg>
                                    Explanation
                                </div>
                                <p className={styles.faqExplainText}>{item.explanation}</p>
                            </div>
                        </div>
                    ))}
                </div>
            )}
        </div>
    );
};

// ── Tab: Short Answer ──────────────────────────────────────
const TabShortAnswer: React.FC<{ raw: string }> = ({ raw }) => {
    const { t } = useTranslation();
    const items = useMemo(() => parsePyList<ShortAnswerItem>(raw), [raw]);
    const [revealedIds, setRevealedIds] = useState<Set<number>>(new Set());
    const [copied, setCopied] = useState(false);

    const prevRaw = useRef(raw);
    useEffect(() => { if (raw !== prevRaw.current) { prevRaw.current = raw; setRevealedIds(new Set()); } }, [raw]);

    const toggleReveal = (idx: number) => setRevealedIds(prev => {
        const next = new Set(prev);
        if (next.has(idx)) next.delete(idx); else next.add(idx);
        return next;
    });
    const allRevealed = items.length > 0 && revealedIds.size === items.length;
    const toggleAll = () => setRevealedIds(allRevealed ? new Set() : new Set(items.map((_, i) => i)));

    const handleCopy = (fmt: string) => { copyText(formatShortAnswer(raw, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatShortAnswer(raw, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!items.length) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noShortAnswer')}</div>;

    return (
        <div className={styles.tabContent}>
            <div className={styles.assessHeader}>
                <button className={styles.revealAllBtn} onClick={toggleAll}>
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M1.5 8s2.5-4.5 6.5-4.5S14.5 8 14.5 8s-2.5 4.5-6.5 4.5S1.5 8 1.5 8z" /><circle cx="8" cy="8" r="2" />
                    </svg>
                    {allRevealed ? t('uploadInfer.workspacePanel.hideAllAnswers') : t('uploadInfer.workspacePanel.showAllAnswers')}
                </button>
                <TabToolbar fmts={SHORT_ANSWER_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
            </div>
            <div className={styles.viewAllList}>
                {items.map((item, idx) => {
                    const revealed = revealedIds.has(idx);
                    return (
                        <div key={idx} className={styles.viewAllCard}>
                            <div className={styles.viewAllQ}>
                                <span className={styles.faqNum}>Q{idx + 1}</span>
                                {item.question}
                            </div>
                            {revealed ? (
                                <div className={styles.shortAnswerBox}>
                                    <div className={styles.shortAnswerLabel}>{t('uploadInfer.workspacePanel.answer')}</div>
                                    <p className={styles.shortAnswerText}>{item.answer}</p>
                                </div>
                            ) : (
                                <button className={styles.revealBtn} onClick={() => toggleReveal(idx)}>
                                    {t('uploadInfer.workspacePanel.revealAnswer')}
                                </button>
                            )}
                        </div>
                    );
                })}
            </div>
        </div>
    );
};

// ── Tab: True / False ────────────────────────────────────────
type TfMode = 'quiz' | 'all';

const TabTrueFalse: React.FC<{ raw: string }> = ({ raw }) => {
    const { t } = useTranslation();
    const items = useMemo(() => parsePyList<TrueFalseItem>(raw), [raw]);

    const [mode, setMode] = useState<TfMode>('quiz');
    const [current, setCurrent] = useState(0);
    const [selected, setSelected] = useState<'true' | 'false' | null>(null);
    const [revealed, setRevealed] = useState(false);
    const [complete, setComplete] = useState(false);
    const [copied, setCopied] = useState(false);

    const prevRaw = useRef(raw);
    useEffect(() => {
        if (raw !== prevRaw.current) {
            prevRaw.current = raw;
            setMode('quiz');
            setCurrent(0); setSelected(null); setRevealed(false); setComplete(false);
        }
    }, [raw]);

    const handleCopy = (fmt: string) => { copyText(formatTrueFalse(raw, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatTrueFalse(raw, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!items.length) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noTrueFalse')}</div>;

    const q = items[current];
    const correctKey: 'true' | 'false' = q.is_true ? 'true' : 'false';
    const isLast = current === items.length - 1;
    const progress = Math.round(((current + (complete ? 1 : 0)) / items.length) * 100);

    const handleSelect = (key: 'true' | 'false') => { if (revealed) return; setSelected(key); setRevealed(true); };
    const handleNext = () => { if (isLast) setComplete(true); else { setCurrent(c => c + 1); setSelected(null); setRevealed(false); } };
    const handleRestart = () => { setCurrent(0); setSelected(null); setRevealed(false); setComplete(false); };
    const getOptClass = (key: 'true' | 'false') => {
        if (!revealed) return selected === key ? styles.faqOptSelected : '';
        if (key === correctKey && selected === key) return styles.faqOptCorrect;
        if (key === correctKey) return styles.faqOptCorrectAlt;
        if (key === selected) return styles.faqOptWrong;
        return '';
    };

    return (
        <div className={styles.tabContent}>
            <div className={styles.assessHeader}>
                <div className={styles.assessModeTabs}>
                    <button className={`${styles.assessModeTab} ${mode === 'quiz' ? styles.assessModeTabActive : ''}`} onClick={() => setMode('quiz')}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                            <circle cx="8" cy="8" r="6" /><path d="M6 8.5l1.5 1.5L10.5 6" />
                        </svg>
                        {t('uploadInfer.workspacePanel.quizMode')}
                    </button>
                    <button className={`${styles.assessModeTab} ${mode === 'all' ? styles.assessModeTabActive : ''}`} onClick={() => setMode('all')}>
                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.4" strokeLinecap="round" strokeLinejoin="round">
                            <path d="M2 4h12M2 8h12M2 12h8" />
                        </svg>
                        {t('uploadInfer.workspacePanel.viewAll')}
                    </button>
                </div>
                <TabToolbar fmts={TRUE_FALSE_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
            </div>

            {mode === 'quiz' && (
                complete ? (
                    <div className={styles.assessComplete}>
                        <div className={styles.assessCompleteIcon}>
                            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                                <path d="M9 12.75L11.25 15 15 9.75M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
                            </svg>
                        </div>
                        <div className={styles.assessCompleteTitle}>{t('uploadInfer.workspacePanel.assessCompleteTitle')}</div>
                        <div className={styles.assessCompleteDesc}>{t('uploadInfer.workspacePanel.assessCompleteDesc', { count: items.length })}</div>
                        <button className={styles.assessRestartBtn} onClick={handleRestart}>{t('uploadInfer.workspacePanel.restart')}</button>
                    </div>
                ) : (
                    <>
                        <div className={styles.assessProgress}>
                            <div className={styles.assessProgressTrack}><div className={styles.assessProgressFill} style={{ width: `${progress}%` }} /></div>
                            <span className={styles.assessProgressLabel}>{current + 1} / {items.length}</span>
                        </div>
                        <div className={styles.faqCard}>
                            <div className={styles.faqQ}>
                                <span className={styles.faqNum}>{current + 1}</span>
                                {q.statement}
                            </div>
                            <div className={styles.tfOptions}>
                                {(['true', 'false'] as const).map(key => (
                                    <button key={key} className={`${styles.tfOpt} ${getOptClass(key)}`} onClick={() => handleSelect(key)} disabled={revealed}>
                                        <span className={styles.tfOptLabel}>{key === 'true' ? t('uploadInfer.workspacePanel.true') : t('uploadInfer.workspacePanel.false')}</span>
                                        {revealed && key === correctKey && <svg className={styles.faqCheckIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M3 8l3.5 3.5L13 4" /></svg>}
                                        {revealed && key === selected && key !== correctKey && <svg className={styles.faqXIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M4 4l8 8M12 4l-8 8" /></svg>}
                                    </button>
                                ))}
                            </div>
                            {revealed && (
                                <div className={styles.faqExplain}>
                                    <div className={styles.faqExplainLabel}>
                                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" /><path d="M8 7v4M8 5.5v.5" /></svg>
                                        {t('uploadInfer.workspacePanel.explanation')}
                                    </div>
                                    <p className={styles.faqExplainText}>{q.explanation}</p>
                                </div>
                            )}
                        </div>
                        {revealed && (
                            <button className={`${styles.assessNextBtn} ${isLast ? styles.assessDoneBtn : ''}`} onClick={handleNext}>
                                {isLast ? t('uploadInfer.workspacePanel.finish') : t('uploadInfer.workspacePanel.next')}
                                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round">
                                    {isLast ? <path d="M3 8l3.5 3.5L13 4" /> : <path d="M3 8h10M9 4l4 4-4 4" />}
                                </svg>
                            </button>
                        )}
                    </>
                )
            )}

            {mode === 'all' && (
                <div className={styles.viewAllList}>
                    {items.map((item, idx) => {
                        const ck: 'true' | 'false' = item.is_true ? 'true' : 'false';
                        return (
                            <div key={idx} className={styles.viewAllCard}>
                                <div className={styles.viewAllQ}>
                                    <span className={styles.faqNum}>{idx + 1}</span>
                                    {item.statement}
                                </div>
                                <div className={styles.tfOptions}>
                                    {(['true', 'false'] as const).map(key => (
                                        <div key={key} className={`${styles.tfOpt} ${key === ck ? styles.faqOptCorrectAlt : styles.viewAllOptNeutral}`}>
                                            <span className={styles.tfOptLabel}>{key === 'true' ? t('uploadInfer.workspacePanel.true') : t('uploadInfer.workspacePanel.false')}</span>
                                            {key === ck && <svg className={styles.faqCheckIcon} viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M3 8l3.5 3.5L13 4" /></svg>}
                                        </div>
                                    ))}
                                </div>
                                <div className={styles.faqExplain}>
                                    <div className={styles.faqExplainLabel}>
                                        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round"><circle cx="8" cy="8" r="6" /><path d="M8 7v4M8 5.5v.5" /></svg>
                                        {t('uploadInfer.workspacePanel.explanation')}
                                    </div>
                                    <p className={styles.faqExplainText}>{item.explanation}</p>
                                </div>
                            </div>
                        );
                    })}
                </div>
            )}
        </div>
    );
};

// ── Tab: Timestamped Summary ─────────────────────────────────
const TabTimestampedSummary: React.FC<{ raw: string }> = ({ raw }) => {
    const { t } = useTranslation();
    const items = useMemo(() => parsePyList<TimestampSegment>(raw), [raw]);
    const [copied, setCopied] = useState(false);

    const handleCopy = (fmt: string) => { copyText(formatTimestampedSummary(raw, fmt).content); setCopied(true); setTimeout(() => setCopied(false), 1500); };
    const handleDownload = (fmt: string) => { const f = formatTimestampedSummary(raw, fmt); downloadFile(f.content, f.filename, f.mime); };

    if (!items.length) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noTimestampedSummary')}</div>;

    return (
        <div className={styles.tabContent}>
            <TabToolbar fmts={TIMESTAMPED_SUMMARY_FMTS} onCopy={handleCopy} onDownload={handleDownload} copied={copied} />
            <div className={styles.tsList}>
                {items.map((seg, idx) => (
                    <div key={idx} className={styles.tsRow}>
                        <div className={styles.tsRail}>
                            <div className={styles.tsDot} />
                            {idx < items.length - 1 && <div className={styles.tsLine} />}
                        </div>
                        <div className={styles.tsCard}>
                            <div className={styles.tsRange}>{seg.start_time} <span className={styles.tsArrow}>→</span> {seg.end_time}</div>
                            <p className={styles.tsText}>{seg.summary}</p>
                        </div>
                    </div>
                ))}
            </div>
        </div>
    );
};


// ── Keyword Insights: shared node-link graph (used for Knowledge Graph) ──
interface GNode { id: string; label: string; value?: number; }
interface GEdge { source: string; target: string; label?: string; }

function buildGraphNodes(nodes: GNode[], edges: GEdge[]): GNode[] {
    const norm = (s: string) => s.trim().toLowerCase();
    const byKey = new Map<string, GNode>();
    const byId = new Map<string, GNode>();
    nodes.forEach(n => {
        byKey.set(norm(n.id), n); byKey.set(norm(n.label), n); byId.set(n.id, n);
    });
    edges.forEach(e => {
        [e.source, e.target].forEach(ref => {
            const k = norm(ref);
            if (!byKey.has(k)) {
                const synth: GNode = { id: ref, label: ref, value: 1 };
                byKey.set(k, synth); byId.set(ref, synth);
            }
        });
    });
    return Array.from(byId.values());
}

// Custom node — a labeled circle sized by `value`, with an invisible
// handle on all 4 sides (each doubling as source+target) so edges can
// connect from whichever side actually faces the other node.
// Multi-word labels wrap onto multiple lines (capped at LABEL_MAX_W).
// Single unbroken words (no spaces/hyphens — common for keyword nodes)
// are NOT force-wrapped: breaking a word with no natural break point
// inside a narrow box is what produced near-vertical stacks of 1-2
// characters per line. Those get their own width instead, up to
// LABEL_HARD_MAX_W.
const LABEL_MAX_W = 150;
const LABEL_HARD_MAX_W = 220;
const LABEL_CHAR_W = 6.1; // ~px per Latin character at the label's font size
const LABEL_CJK_CHAR_W = 12.5; // ~px per CJK character (Hangul/Kanji/Kana glyphs render roughly 2x as wide as Latin at the same font-size)
const LABEL_LINE_H = 14;

// Matches Hangul syllables/Jamo, CJK Unified Ideographs, and Hiragana/Katakana.
const CJK_CHAR_RE = /[\u3040-\u30ff\u3400-\u4dbf\u4e00-\u9fff\uac00-\ud7a3\uf900-\ufaff]/;

function estimateTextWidth(label: string): number {
    let width = 8;
    for (const ch of label) width += CJK_CHAR_RE.test(ch) ? LABEL_CJK_CHAR_W : LABEL_CHAR_W;
    return width;
}

function estimateNodeBox(label: string, circleSize: number) {
    const hasBreakPoint = /[\s-]/.test(label);
    const rawTextWidth = estimateTextWidth(label);
    const capWidth = hasBreakPoint ? LABEL_MAX_W : LABEL_HARD_MAX_W;
    const labelWidth = Math.min(capWidth, Math.max(rawTextWidth, 44));
    const lines = hasBreakPoint ? Math.max(1, Math.ceil(rawTextWidth / LABEL_MAX_W)) : 1;
    const w = Math.max(circleSize, labelWidth, 72) + 16;
    const h = circleSize + 16 + lines * LABEL_LINE_H;
    return { w, h, labelWidth };
}

const MindMapNode: React.FC<{ data: { label: string; value?: number; labelWidth?: number } }> = ({ data }) => {
    const size = 34 + Math.min(26, (data.value ?? 1) * 4);
    const hasBreakPoint = /[\s-]/.test(data.label);
    return (
        <div className={styles.rfNode} style={{ width: size, height: size }} title={data.label}>
            <Handle type="target" position={Position.Top} id="top-target" className={styles.rfHandle} />
            <Handle type="source" position={Position.Bottom} id="bottom-source" className={styles.rfHandle} />
            <span
                className={styles.rfNodeLabel}
                style={{ maxWidth: data.labelWidth ?? LABEL_MAX_W, whiteSpace: hasBreakPoint ? 'normal' : 'nowrap' }}
            >
                {data.label}
            </span>
        </div>
    );
};
const RF_NODE_TYPES = { mindmap: MindMapNode };

// ── Dagre auto-layout — replaces naive circular placement, which packed
//    nodes on top of each other once a graph had more than a handful of
//    them. Dagre lays nodes out in ranked layers with guaranteed spacing,
//    so nothing overlaps regardless of graph size. ──
function computeDagreLayout(nodes: GNode[], edges: GEdge[]) {
    const g = new dagre.graphlib.Graph();
    g.setGraph({ rankdir: 'TB', nodesep: 72, ranksep: 150, marginx: 60, marginy: 60, acyclicer: 'greedy' });
    g.setDefaultEdgeLabel(() => ({}));

    const dims = new Map<string, { w: number; h: number; labelWidth: number }>();
    nodes.forEach(n => {
        const size = 34 + Math.min(26, (n.value ?? 1) * 4);
        const dim = estimateNodeBox(n.label, size);
        dims.set(n.id, dim);
        g.setNode(n.id, dim);
    });

    const norm = (s: string) => s.trim().toLowerCase();
    const byKey = new Map<string, string>();
    nodes.forEach(n => { byKey.set(norm(n.id), n.id); byKey.set(norm(n.label), n.id); });

    // Dedupe edges — repeated source/target pairs don't add information but
    // do add extra crowded lines between the same two nodes.
    const seenEdges = new Set<string>();
    const resolvedEdges: { source: string; target: string; label?: string }[] = [];
    edges.forEach(e => {
        const s = byKey.get(norm(e.source));
        const tg = byKey.get(norm(e.target));
        if (!s || !tg || s === tg) return;
        const key = `${s}→${tg}`;
        if (seenEdges.has(key)) return;
        seenEdges.add(key);
        // Reserve space for the edge's own label too — without this, dagre
        // has no idea the label text exists and will happily route another
        // node right where that text needs to sit.
        const label = e.label?.trim();
        g.setEdge(s, tg, label ? { width: Math.min(120, estimateTextWidth(label) + 4), height: LABEL_LINE_H + 8, labelpos: 'c' } : {});
        resolvedEdges.push({ source: s, target: tg, label: e.label });
    });

    dagre.layout(g);

    const positioned = nodes.map(n => {
        const pos = g.node(n.id);
        const dim = dims.get(n.id)!;
        // Dagre positions are centers — React Flow expects top-left.
        return { ...n, x: pos.x - dim.w / 2, y: pos.y - dim.h / 2, labelWidth: dim.labelWidth };
    });
    return { positioned, resolvedEdges };
}

// ── Forest layout — each disconnected tree gets its own horizontal
//    (root → children growing left-to-right) dagre layout, then the trees
//    are stacked vertically underneath one another. Labels stay perfectly
//    horizontal either way (that's just CSS text direction, unaffected by
//    which way the tree branches), but growing each tree sideways instead
//    of downward gives multi-word labels far more horizontal room before
//    they need to wrap, and keeps unrelated trees visually separated
//    instead of interleaved in one shared vertical ranking. ──
function computeForestLayout(nodes: GNode[], edges: GEdge[]) {
    const norm = (s: string) => s.trim().toLowerCase();
    const byKey = new Map<string, string>();
    nodes.forEach(n => { byKey.set(norm(n.id), n.id); byKey.set(norm(n.label), n.id); });

    // Reduce to one parent per node — same rule as treeMode above, so each
    // node belongs to exactly one tree with no crossing multi-parent edges.
    const seenEdges = new Set<string>();
    const hasParent = new Set<string>();
    const treeEdges: { source: string; target: string; label?: string }[] = [];
    edges.forEach(e => {
        const s = byKey.get(norm(e.source));
        const tg = byKey.get(norm(e.target));
        if (!s || !tg || s === tg) return;
        const key = `${s}→${tg}`;
        if (seenEdges.has(key) || hasParent.has(tg)) return;
        seenEdges.add(key);
        hasParent.add(tg);
        treeEdges.push({ source: s, target: tg, label: e.label });
    });

    // Union-find to group nodes into connected components (one per tree,
    // including singleton nodes with no edges at all).
    const uf = new Map<string, string>();
    nodes.forEach(n => uf.set(n.id, n.id));
    const find = (x: string): string => {
        let root = x;
        while (uf.get(root) !== root) root = uf.get(root)!;
        while (uf.get(x) !== root) { const next = uf.get(x)!; uf.set(x, root); x = next; }
        return root;
    };
    const union = (a: string, b: string) => { const ra = find(a), rb = find(b); if (ra !== rb) uf.set(ra, rb); };
    treeEdges.forEach(e => union(e.source, e.target));

    const groups = new Map<string, GNode[]>();
    nodes.forEach(n => {
        const root = find(n.id);
        if (!groups.has(root)) groups.set(root, []);
        groups.get(root)!.push(n);
    });

    const dims = new Map<string, { w: number; h: number; labelWidth: number }>();
    nodes.forEach(n => {
        const size = 34 + Math.min(26, (n.value ?? 1) * 4);
        dims.set(n.id, estimateNodeBox(n.label, size));
    });

    const TREE_GAP = 56;
    let yOffset = 0;
    const positioned: (GNode & { x: number; y: number; labelWidth: number })[] = [];

    for (const groupNodes of groups.values()) {
        const g = new dagre.graphlib.Graph();
        g.setGraph({ rankdir: 'LR', nodesep: 28, ranksep: 130, marginx: 20, marginy: 20, acyclicer: 'greedy' });
        g.setDefaultEdgeLabel(() => ({}));
        groupNodes.forEach(n => g.setNode(n.id, dims.get(n.id)!));

        const idsInGroup = new Set(groupNodes.map(n => n.id));
        treeEdges.forEach(e => {
            if (!idsInGroup.has(e.source) || !idsInGroup.has(e.target)) return;
            const label = e.label?.trim();
            g.setEdge(e.source, e.target, label ? { width: Math.min(120, estimateTextWidth(label) + 4), height: LABEL_LINE_H + 8, labelpos: 'c' } : {});
        });

        dagre.layout(g);

        let minX = Infinity, maxX = -Infinity, minY = Infinity, maxY = -Infinity;
        groupNodes.forEach(n => {
            const pos = g.node(n.id);
            const dim = dims.get(n.id)!;
            minX = Math.min(minX, pos.x - dim.w / 2);
            maxX = Math.max(maxX, pos.x + dim.w / 2);
            minY = Math.min(minY, pos.y - dim.h / 2);
            maxY = Math.max(maxY, pos.y + dim.h / 2);
        });
        const shiftX = -minX;
        const shiftY = yOffset - minY;

        groupNodes.forEach(n => {
            const pos = g.node(n.id);
            const dim = dims.get(n.id)!;
            positioned.push({ ...n, x: pos.x - dim.w / 2 + shiftX, y: pos.y - dim.h / 2 + shiftY, labelWidth: dim.labelWidth });
        });

        yOffset += (maxY - minY) + TREE_GAP;
    }

    return { positioned, resolvedEdges: treeEdges };
}

const NodeLinkGraph: React.FC<{ nodes: GNode[]; edges: GEdge[]; treeMode?: boolean }> = ({ nodes, edges, treeMode = false }) => {
    const { t } = useTranslation();
    const allNodes = useMemo(() => buildGraphNodes(nodes, edges), [nodes, edges]);

    const initial = useMemo(() => {
        const { positioned, resolvedEdges } = treeMode
            ? computeForestLayout(allNodes, edges)
            : computeDagreLayout(allNodes, edges);

        const rfNodes: RFNode[] = positioned.map(n => ({
            id: n.id,
            type: 'mindmap',
            position: { x: n.x, y: n.y },
            data: { label: n.label, value: n.value, labelWidth: n.labelWidth },
        }));

        const rfEdges: RFEdge[] = resolvedEdges.map((e, i) => ({
            id: `e${i}-${e.source}-${e.target}`,
            source: e.source,
            target: e.target,
            sourceHandle: 'bottom-source',
            targetHandle: 'top-target',
            type: 'smoothstep',
            label: e.label,
            style: { stroke: 'var(--bdr2)' },
            labelStyle: { fill: 'var(--t2)', fontSize: 10 },
            labelBgStyle: { fill: 'var(--bg1)' },
        }));
        return { rfNodes, rfEdges };
    }, [allNodes, edges, treeMode]);

    const [rfNodes, setRfNodes] = useState<RFNode[]>(initial.rfNodes);
    useEffect(() => { setRfNodes(initial.rfNodes); }, [initial.rfNodes]);
    const onNodesChange = useCallback((changes: NodeChange[]) => setRfNodes(nds => applyNodeChanges(changes, nds)), []);

    if (allNodes.length === 0) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;

    return (
        <div className={styles.graphWrap}>
            <ReactFlow
                nodes={rfNodes}
                edges={initial.rfEdges}
                onNodesChange={onNodesChange}
                nodeTypes={RF_NODE_TYPES}
                fitView
                fitViewOptions={{ padding: 0.25 }}
                minZoom={0.1}
                maxZoom={1.5}
                proOptions={{ hideAttribution: true }}
            >
                <Background gap={16} size={1} color="var(--bdr)" />
                <Controls showInteractive={false} />
                {allNodes.length > 8 && <MiniMap pannable zoomable style={{ background: 'var(--bg0)' }} />}
            </ReactFlow>
        </div>
    );
};

// ── Keyword Insights: timeline — heatmap-style grid with time labels
//    running horizontally across the top header row (guaranteed via
//    inline styles: nowrap + horizontal writing-mode) and one row per
//    keyword running down the left. ──
const TIME_UNIT_LABEL: Record<TimelineTimeUnit, string> = {
    minutes: 'uploadInfer.workspacePanel.timeUnitMinutes',
    hours: 'uploadInfer.workspacePanel.timeUnitHours',
    seconds: 'uploadInfer.workspacePanel.timeUnitSeconds',
};
const TIME_UNIT_SHORT: Record<TimelineTimeUnit, string> = {
    minutes: 'uploadInfer.workspacePanel.timeUnitMinShort',
    hours: 'uploadInfer.workspacePanel.timeUnitHrShort',
    seconds: 'uploadInfer.workspacePanel.timeUnitSecShort',
};

const TimelineView: React.FC<{ data: TimelineData }> = ({ data }) => {
    const { t } = useTranslation();
    const labels = data.labels ?? [];
    const datasets = data.datasets ?? [];
    if (!labels.length || !datasets.length) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;

    const max = Math.max(1, ...datasets.flatMap(d => d.data.filter(v => typeof v === 'number')));
    const ROW_LABEL_W = 140;
    const COL_W = 56;
    const timeUnit = data.time_unit ?? 'minutes';
    const unitLabel = t(TIME_UNIT_LABEL[timeUnit]);
    // When labels are already actual clock timestamps (e.g. "0:00–5:00"),
    // the format itself conveys the unit — don't also append a suffix.
    // Only append the short unit to plain numeric bucket labels.
    const unitShort = t(TIME_UNIT_SHORT[timeUnit]);
    const formatTimeLabel = (lab: string) => data.timestamped ? lab : `${lab} ${unitShort}`;

    // Text guaranteed to stay horizontal, left-to-right, single line —
    // independent of whatever the surrounding stylesheet does elsewhere.
    const horizontalLabel: React.CSSProperties = {
        writingMode: 'horizontal-tb',
        textOrientation: 'mixed',
        whiteSpace: 'nowrap',
        overflow: 'hidden',
        textOverflow: 'ellipsis',
    };

    return (
        <div className={styles.tabContent}>
            <div className={styles.timelineAxisHint}>
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <circle cx="8" cy="8" r="6.25" />
                    <path d="M8 7.2v3.6M8 5.2v.1" />
                </svg>
                <span>
                    {t('uploadInfer.workspacePanel.timelineAxisHint', { unit: unitLabel })}
                </span>
            </div>
        <div style={{ overflowX: 'auto', paddingBottom: 4 }}>
            <div style={{ display: 'grid', gridTemplateColumns: `${ROW_LABEL_W}px repeat(${labels.length}, minmax(${COL_W}px, 1fr))`, width: 'max-content', minWidth: '100%' }}>
                {/* Corner cell — axis labels */}
                <div style={{ ...horizontalLabel, fontSize: 9.5, fontWeight: 700, color: 'var(--t2)', textTransform: 'uppercase', letterSpacing: '0.06em', display: 'flex', alignItems: 'flex-end', padding: '0 10px 8px 0' }}>
                    {t('uploadInfer.workspacePanel.timelineYAxis', 'Keyword')}
                </div>
                {/* Time labels — header row, horizontal across the top */}
                {labels.map((lab, li) => (
                    <div
                        key={li}
                        title={formatTimeLabel(lab)}
                        style={{
                            ...horizontalLabel,
                            fontSize: 10.5, fontWeight: 600, color: 'var(--t2)',
                            textAlign: 'center', padding: '0 4px 8px',
                        }}
                    >
                        {formatTimeLabel(lab)}
                    </div>
                ))}
                {/* One row per keyword */}
                {datasets.map((ds, di) => (
                    <React.Fragment key={di}>
                        <div style={{ ...horizontalLabel, fontSize: 12, fontWeight: 600, color: 'var(--t1)', padding: '6px 10px 6px 0', display: 'flex', alignItems: 'center' }} title={ds.label}>
                            {ds.label}
                        </div>
                        {labels.map((lab, li) => {
                            const v = ds.data[li] ?? 0;
                            const alpha = v > 0 ? Math.min(1, 0.22 + (v / max) * 0.78) : 0;
                            return (
                                <div
                                    key={li}
                                    title={`${ds.label} \u00b7 ${formatTimeLabel(lab)}`}
                                    style={{
                                        height: 32, margin: 2, borderRadius: 4,
                                        background: `rgba(91, 164, 239, ${alpha})`,
                                    }}
                                />
                            );
                        })}
                    </React.Fragment>
                ))}
            </div>
        </div>
        </div>
    );
};

// ── Keyword Insights: word cloud ──
// ── Pan/zoom image viewer — same interaction model as the Knowledge Graph
// tab (drag anywhere to pan, scroll to zoom in/out, buttons for
// zoom in/out/reset), but self-contained (no react-flow dependency needed
// for a single static image). ──
const ZOOM_MIN = 1;
const ZOOM_MAX = 4;
const ZOOM_STEP = 0.35;

const ZoomPanImage: React.FC<{ src: string; alt: string }> = ({ src, alt }) => {
    const { t } = useTranslation();
    const containerRef = useRef<HTMLDivElement>(null);
    const [scale, setScale] = useState(1);
    const [pan, setPan] = useState({ x: 0, y: 0 });
    const dragRef = useRef<{ startX: number; startY: number; startPanX: number; startPanY: number } | null>(null);
    const [dragging, setDragging] = useState(false);

    const clampScale = (s: number) => Math.min(ZOOM_MAX, Math.max(ZOOM_MIN, s));

    // Zoom while keeping the point under the cursor/center visually fixed —
    // same "zoom toward where you're pointing" feel as the graph view.
    const zoomAt = (nextScaleRaw: number, clientX?: number, clientY?: number) => {
        const nextScale = clampScale(nextScaleRaw);
        const el = containerRef.current;
        if (!el) { setScale(nextScale); return; }
        const rect = el.getBoundingClientRect();
        const cx = clientX ?? rect.left + rect.width / 2;
        const cy = clientY ?? rect.top + rect.height / 2;
        const originX = cx - rect.left - rect.width / 2;
        const originY = cy - rect.top - rect.height / 2;
        setPan(prev => {
            if (nextScale === ZOOM_MIN) return { x: 0, y: 0 };
            const ratio = nextScale / scale;
            return {
                x: originX - (originX - prev.x) * ratio,
                y: originY - (originY - prev.y) * ratio,
            };
        });
        setScale(nextScale);
    };

    const handleWheel = (e: React.WheelEvent) => {
        e.preventDefault();
        const delta = e.deltaY < 0 ? ZOOM_STEP : -ZOOM_STEP;
        zoomAt(scale + delta, e.clientX, e.clientY);
    };

    const handleMouseDown = (e: React.MouseEvent) => {
        if (scale <= ZOOM_MIN) return;
        dragRef.current = { startX: e.clientX, startY: e.clientY, startPanX: pan.x, startPanY: pan.y };
        setDragging(true);
    };
    useEffect(() => {
        if (!dragging) return;
        const onMove = (e: MouseEvent) => {
            const d = dragRef.current;
            if (!d) return;
            setPan({ x: d.startPanX + (e.clientX - d.startX), y: d.startPanY + (e.clientY - d.startY) });
        };
        const onUp = () => { setDragging(false); dragRef.current = null; };
        window.addEventListener('mousemove', onMove);
        window.addEventListener('mouseup', onUp);
        return () => {
            window.removeEventListener('mousemove', onMove);
            window.removeEventListener('mouseup', onUp);
        };
    }, [dragging]);

    const zoomIn = () => zoomAt(scale + ZOOM_STEP);
    const zoomOut = () => zoomAt(scale - ZOOM_STEP);
    const reset = () => { setScale(1); setPan({ x: 0, y: 0 }); };

    return (
        <div
            ref={containerRef}
            className={styles.zoomPanWrap}
            onWheel={handleWheel}
            onMouseDown={handleMouseDown}
            onDoubleClick={reset}
            style={{ cursor: scale > ZOOM_MIN ? (dragging ? 'grabbing' : 'grab') : 'default' }}
        >
            <img
                src={src}
                alt={alt}
                className={styles.wordCloudImage}
                draggable={false}
                style={{
                    transform: `translate(${pan.x}px, ${pan.y}px) scale(${scale})`,
                    transition: dragging ? 'none' : 'transform 0.15s ease-out',
                }}
            />
            <div className={styles.zoomControls}>
                <button type="button" className={styles.zoomControlBtn} onClick={zoomIn} disabled={scale >= ZOOM_MAX} title={t('uploadInfer.workspacePanel.zoomIn')}>
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="7" cy="7" r="5" /><path d="M13.5 13.5L10.8 10.8M7 4.5v5M4.5 7h5" />
                    </svg>
                </button>
                <button type="button" className={styles.zoomControlBtn} onClick={zoomOut} disabled={scale <= ZOOM_MIN} title={t('uploadInfer.workspacePanel.zoomOut')}>
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="7" cy="7" r="5" /><path d="M13.5 13.5L10.8 10.8M4.5 7h5" />
                    </svg>
                </button>
                <button type="button" className={styles.zoomControlBtn} onClick={reset} disabled={scale === ZOOM_MIN && pan.x === 0 && pan.y === 0} title={t('uploadInfer.workspacePanel.zoomReset')}>
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.6" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M13.5 8a5.5 5.5 0 11-1.6-3.9" /><path d="M13.5 2.5v3h-3" />
                    </svg>
                </button>
            </div>
        </div>
    );
};

const WordCloudView: React.FC<{ data: WordCloudData }> = ({ data }) => {
    const { t } = useTranslation();
    if (!data.wordcloud_image) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;
    return (
        <div className={styles.wordCloudImageWrap}>
            <ZoomPanImage src={data.wordcloud_image} alt={t('uploadInfer.workspacePanel.kiTabs.wordcloud')} />
        </div>
    );
};

// ── Keyword Insights: importance vs complexity scatter ──
// Complexity → accent color, reused for the column header, bars, and dots.
const COMPLEXITY_COLOR: Record<string, string> = {
    Easy: 'var(--green)',
    Medium: 'var(--amber)',
    Hard: 'var(--red)',
};
const complexityColor = (c: string) => COMPLEXITY_COLOR[c] ?? 'var(--blue)';

const ImportanceComplexityScatter: React.FC<{ data: ImportanceComplexityData }> = ({ data }) => {
    const { t } = useTranslation();
    const items = data.data ?? [];
    if (!items.length) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;

    const order = ['Easy', 'Medium', 'Hard'];
    const complexities = Array.from(new Set(items.map(it => it.complexity)))
        .sort((a, b) => {
            const ai = order.indexOf(a), bi = order.indexOf(b);
            if (ai === -1 && bi === -1) return a.localeCompare(b);
            if (ai === -1) return 1;
            if (bi === -1) return -1;
            return ai - bi;
        });

    // Importance is always on a fixed 0–10 scale from the backend, so the
    // bar fill is a direct out-of-10 progress value rather than scaled
    // against whatever the largest item in this particular file happens
    // to be — a 6 always fills 60% of the bar, on any file.
    const IMPORTANCE_MAX = 10;

    return (
        <div className={styles.tabContent}>
            <div className={styles.importanceHint}>
                <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
                    <circle cx="8" cy="8" r="6.25" />
                    <path d="M8 7.2v3.6M8 5.2v.1" />
                </svg>
                <span>{t('uploadInfer.workspacePanel.importanceHint')}</span>
            </div>
            <div className={styles.importanceColumns}>
            {complexities.map(c => {
                const colItems = items.filter(it => it.complexity === c).sort((a, b) => b.importance - a.importance);
                const color = complexityColor(c);
                return (
                    <div key={c} className={styles.importanceColumn}>
                        <div className={styles.importanceColumnHead} style={{ color, borderColor: color }}>
                            {c}
                            <span className={styles.importanceColumnCount}>{colItems.length}</span>
                        </div>
                        <div className={styles.importanceRows}>
                            {colItems.map((it, i) => (
                                <div key={i} className={styles.importanceRow} title={it.reason}>
                                    <span className={styles.importanceDot} style={{ background: color }} />
                                    <span className={styles.importanceKeyword}>{it.keyword}</span>
                                    <div className={styles.importanceBarTrack}>
                                        <div
                                            className={styles.importanceBarFill}
                                            style={{ width: `${Math.min(100, (it.importance / IMPORTANCE_MAX) * 100)}%`, background: color }}
                                        />
                                    </div>
                                    <span className={styles.importanceMeta}>
                                        <span className={styles.importanceValue}>{it.importance}/{IMPORTANCE_MAX}</span>
                                        <span
                                            className={styles.importanceFrequency}
                                            title={t('uploadInfer.workspacePanel.frequencyTitle')}
                                        >
                                            {t('uploadInfer.workspacePanel.frequencyCount', { count: it.frequency })}
                                        </span>
                                    </span>
                                </div>
                            ))}
                        </div>
                    </div>
                );
            })}
            </div>
        </div>
    );
};

// ── Keyword Insights: glossary ──
const GlossaryView: React.FC<{ data: GlossaryData }> = ({ data }) => {
    const { t } = useTranslation();
    const items = data.glossary ?? [];
    if (!items.length) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;
    const fmtTime = (ms: number) => {
        const totalSec = Math.floor(ms / 1000);
        const m = Math.floor(totalSec / 60), s = totalSec % 60;
        return `${m}:${String(s).padStart(2, '0')}`;
    };
    return (
        <div className={styles.glossaryList}>
            {items.map((it, i) => (
                <div key={i} className={styles.glossaryRow}>
                    <div className={styles.glossaryTerm}>{it.term}<span className={styles.glossaryTime}>{fmtTime(it.first_mentioned_ms)}</span></div>
                    <div className={styles.glossaryDef}>{it.definition}</div>
                </div>
            ))}
        </div>
    );
};

// ── Tab: Keyword Insights ────────────────────────────────────
const KI_SUBTABS = ['graph', 'wordcloud', 'timeline', 'importance', 'glossary'] as const;
type KiSubTab = typeof KI_SUBTABS[number];

const TabKeywordInsights: React.FC<{ data: KeywordInsights | null }> = ({ data }) => {
    const { t } = useTranslation();
    const [sub, setSub] = useState<KiSubTab>('graph');

    if (!data) return <div className={`${styles.tabContent} ${styles.tabEmpty}`}>{t('uploadInfer.workspacePanel.noKeywordInsights')}</div>;

    return (
        <div className={styles.tabContent}>
            <ScrollableTabRow
                ids={KI_SUBTABS}
                activeId={sub}
                onChange={setSub}
                labelFor={id => t(`uploadInfer.workspacePanel.kiTabs.${id}`)}
                itemClassName={styles.kiSubTab}
                activeClassName={styles.kiSubTabActive}
                wrapClassName={styles.kiSubNavWrap}
                trackClassName={styles.kiSubNav}
            />
            <div className={styles.kiBody}>
                {sub === 'graph' && (data.knowledge_graph
                    ? <NodeLinkGraph nodes={data.knowledge_graph.nodes} edges={data.knowledge_graph.edges.map(e => ({ source: e.source, target: e.target, label: e.type }))} treeMode />
                    : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'wordcloud' && (data.word_cloud ? <WordCloudView data={data.word_cloud} /> : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'timeline' && (data.timeline
                    ? <TimelineView data={data.timeline} />
                    : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'importance' && (data.importance_complexity ? <ImportanceComplexityScatter data={data.importance_complexity} /> : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
                {sub === 'glossary' && (data.glossary ? <GlossaryView data={data.glossary} /> : <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>)}
            </div>
        </div>
    );
};

// ── Scrollable tab row — generic horizontal-scroll tab strip with arrow
//    buttons + edge fades, reused for both the main tab bar and any
//    sub-navigation row (e.g. Keyword Insights) that could overflow. ──
function ScrollableTabRow<T extends string>({
    ids, activeId, onChange, labelFor,
    itemClassName, activeClassName, wrapClassName, trackClassName,
    dataTourFor,
}: {
    ids: readonly T[];
    activeId: T;
    onChange: (id: T) => void;
    labelFor: (id: T) => string;
    itemClassName: string;
    activeClassName: string;
    wrapClassName: string;
    trackClassName: string;
    // Optional — lets a specific caller (e.g. the main results tab bar)
    // tag each individual tab button for the guided tour, without
    // affecting other callers that reuse this same component (e.g. the
    // Keyword Insights sub-tab row).
    dataTourFor?: (id: T) => string | undefined;
}) {
    const trackRef = useRef<HTMLDivElement>(null);
    const [canScrollLeft, setCanScrollLeft] = useState(false);
    const [canScrollRight, setCanScrollRight] = useState(false);

    const updateScrollState = () => {
        const el = trackRef.current;
        if (!el) return;
        setCanScrollLeft(el.scrollLeft > 2);
        setCanScrollRight(el.scrollLeft < el.scrollWidth - el.clientWidth - 2);
    };

    useEffect(() => {
        updateScrollState();
        const el = trackRef.current;
        if (!el) return;
        const onResize = () => updateScrollState();
        window.addEventListener('resize', onResize);
        el.addEventListener('scroll', updateScrollState);
        return () => { window.removeEventListener('resize', onResize); el.removeEventListener('scroll', updateScrollState); };
    }, []); // eslint-disable-line

    // Keep the active item in view when it changes
    useEffect(() => {
        const el = trackRef.current;
        if (!el) return;
        const activeEl = el.querySelector<HTMLElement>(`[data-tab-id="${activeId}"]`);
        activeEl?.scrollIntoView({ behavior: 'smooth', block: 'nearest', inline: 'nearest' });
        const id = setTimeout(updateScrollState, 300);
        return () => clearTimeout(id);
    }, [activeId]); // eslint-disable-line

    const scrollBy = (dx: number) => trackRef.current?.scrollBy({ left: dx, behavior: 'smooth' });

    return (
        <div className={wrapClassName}>
            {canScrollLeft && (
                <button className={`${styles.tabScrollBtn} ${styles.tabScrollBtnLeft}`} onClick={() => scrollBy(-160)} aria-label="Scroll left">
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M10 3L6 8l4 5" /></svg>
                </button>
            )}
            <div className={trackClassName} ref={trackRef}>
                {ids.map(id => (
                    <button
                        key={id}
                        data-tab-id={id}
                        data-tour={dataTourFor?.(id)}
                        className={`${itemClassName} ${activeId === id ? activeClassName : ''}`}
                        onClick={() => onChange(id)}
                    >
                        {labelFor(id)}
                    </button>
                ))}
            </div>
            {canScrollLeft && <div className={`${styles.tabFade} ${styles.tabFadeLeft}`} />}
            {canScrollRight && <div className={`${styles.tabFade} ${styles.tabFadeRight}`} />}
            {canScrollRight && (
                <button className={`${styles.tabScrollBtn} ${styles.tabScrollBtnRight}`} onClick={() => scrollBy(160)} aria-label="Scroll right">
                    <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round"><path d="M6 3l4 5-4 5" /></svg>
                </button>
            )}
        </div>
    );
}

// ── Main tab bar — the fixed-width flex row silently clipped tabs once
//    there were more than ~3; ScrollableTabRow fixes that. ──
const ScrollableTabs: React.FC<{ activeTab: TabId; onChange: (id: TabId) => void }> = ({ activeTab, onChange }) => {
    const { t } = useTranslation();
    return (
        <div data-tour="results-tabbar">
            <ScrollableTabRow
                ids={TAB_IDS}
                activeId={activeTab}
                onChange={onChange}
                labelFor={id => t(`uploadInfer.workspacePanel.tabs.${id}`)}
                itemClassName={styles.tab}
                activeClassName={styles.active}
                wrapClassName={styles.wsptabsWrap}
                trackClassName={styles.wsptabs}
                dataTourFor={id => `results-tab-${id}`}
            />
        </div>
    );
};

// ── Main panel ────────────────────────────────────────────
const WorkspacePanel = forwardRef<WorkspacePanelHandle, Props>(({ step2Visible = true, step2Minimized = false, fileResult, fileLoading, activeFileId, onResultUpdate }, ref) => {
    const { t } = useTranslation();
    const [activeTab, setActiveTab] = useState<TabId>('summary');

    useImperativeHandle(ref, () => ({
        setTab: (id: TabId) => setActiveTab(id),
    }), []);

    return (
        <div className={`${styles.wspanel} ${(!step2Visible || step2Minimized) ? styles.wspanelExpanded : ''} ${step2Visible ? styles.wspanelWithStep2 : ''}`}>
            <div className={styles.wspanelHead}>
                <div className={styles.headLeft}>
                    {fileResult ? (
                        <>
                            <div className={styles.wsftitle}>{fileResult.fileName}</div>
                            <div className={styles.wsmeta}>
                                <span className={styles.wsmetaId}>#{fileResult.fileId}</span>
                                {fileResult.insertedAt && (<><span className={styles.wsmetaSep}>·</span><span className={styles.wsmetaDate}>{fileResult.insertedAt}</span></>)}
                            </div>
                        </>
                    ) : (
                        <>
                            <div className={styles.wsftitleEmpty}>—</div>
                            <div className={styles.wsmeta}><span className={styles.wsmetaHint}>{t('uploadInfer.workspacePanel.clickToView')}</span></div>
                        </>
                    )}
                </div>
            </div>

            <ScrollableTabs activeTab={activeTab} onChange={setActiveTab} />

            <div className={styles.wsbody}>
                {fileLoading && (
                    <div className={styles.wsSpinner}><div className={styles.spinner} /><span>{t('uploadInfer.workspacePanel.loadingFile')}</span></div>
                )}
                {!fileLoading && !fileResult && (
                    <div className={styles.wsEmpty}>
                        <svg viewBox="0 0 48 48" fill="none" stroke="currentColor" strokeWidth="1.2" strokeLinecap="round" strokeLinejoin="round">
                            <rect x="8" y="6" width="32" height="36" rx="3" /><path d="M16 16h16M16 23h16M16 30h10" />
                        </svg>
                        <div className={styles.wsEmptyTitle}>{t('uploadInfer.workspacePanel.noFileSelected')}</div>
                        <div className={styles.wsEmptyDesc}>{t('uploadInfer.workspacePanel.noFileDesc')}</div>
                    </div>
                )}
                {!fileLoading && fileResult && (
                    <>
                        {activeTab === 'summary' && <TabSummary summary={fileResult.summary} fileId={fileResult.fileId} onSaved={s => onResultUpdate?.({ summary: s })} />}
                        {activeTab === 'keywords' && <TabKeywords keywords={fileResult.keywords} fileId={fileResult.fileId} onSaved={kw => onResultUpdate?.({ keywords: kw })} />}
                        {activeTab === 'assessment' && <TabAssessment faq={fileResult.faq} />}
                        {activeTab === 'shortAnswer' && <TabShortAnswer raw={fileResult.shortAnswer} />}
                        {activeTab === 'trueFalse' && <TabTrueFalse raw={fileResult.trueFalse} />}
                        {activeTab === 'timestampedSummary' && <TabTimestampedSummary raw={fileResult.timestampedSummary} />}
                        {activeTab === 'keywordInsights' && <TabKeywordInsights data={fileResult.keywordInsights} />}
                    </>
                )}
            </div>
        </div>
    );
});

WorkspacePanel.displayName = 'WorkspacePanel';

export default WorkspacePanel;













// ═══════════════════════════════════════════════
// pages/UploadInfer/WorkspacePanel.module.scss
// LectureAI · Step-3 Workspace Result panel
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

// ── Keyframes ────────────────────────────────────────────
@keyframes fadein {
    from {
        opacity: 0;
        transform: translateY(2px);
    }

    to {
        opacity: 1;
        transform: translateY(0);
    }
}

@keyframes fadeInUp {
    from {
        opacity: 0;
        transform: translateY(10px);
    }

    to {
        opacity: 1;
        transform: translateY(0);
    }
}

@keyframes float {

    0%,
    100% {
        transform: translateY(0);
    }

    50% {
        transform: translateY(-8px);
    }
}

@keyframes wsSpin {
    to {
        transform: rotate(360deg);
    }
}

@keyframes chipFadeIn {
    from {
        opacity: 0;
        transform: scale(0.8) translateY(-6px);
    }

    to {
        opacity: 1;
        transform: scale(1) translateY(0);
    }
}

@keyframes explanationSlideIn {
    from {
        opacity: 0;
        transform: translateY(-6px);
    }

    to {
        opacity: 1;
        transform: translateY(0);
    }
}

@keyframes iconPop {
    0% {
        transform: scale(0);
        opacity: 0;
    }

    60% {
        transform: scale(1.2);
    }

    100% {
        transform: scale(1);
        opacity: 1;
    }
}

@keyframes completeFadeIn {
    from {
        opacity: 0;
        transform: scale(0.95);
    }

    to {
        opacity: 1;
        transform: scale(1);
    }
}

@keyframes iconBounce {

    0%,
    100% {
        transform: scale(1);
    }

    50% {
        transform: scale(1.1);
    }
}

// ── Panel shell ──────────────────────────────────────────
.wspanel {
    width: 360px;
    flex-shrink: 0;
    border-left: none;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    background: var(--bg0);
    transition: width 0.25s ease;
    position: relative;

    // Gradient border — only visible when step-2 panel is open beside it
    &::before {
        content: '';
        position: absolute;
        top: 0;
        left: 0;
        bottom: 0;
        width: 1px;
        background: linear-gradient(180deg,
                rgba(139, 92, 246, 0.0) 0%,
                rgba(139, 92, 246, 0.7) 20%,
                rgba(56, 196, 186, 0.8) 55%,
                rgba(240, 160, 48, 0.7) 85%,
                rgba(240, 160, 48, 0.0) 100%);
        pointer-events: none;
        opacity: 0;
        transition: opacity 0.32s ease;
    }
}

// Border fades in only when step-2 is visible next to workspace
.wspanelWithStep2::before {
    opacity: 1;
}

.wspanelExpanded {
    width: auto;
    flex: 1;
}

// ── Header — unchanged from original ────────────────────
.wspanelHead {
    padding: 12px 18px 9px;
    border-bottom: 1px solid var(--bdr);
    background: var(--bg1);
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
}

.headLeft {
    flex: 1;
    min-width: 0;
}

.wsftitle {
    font-size: 16px;
    font-weight: 700;
    color: var(--t0);
    letter-spacing: -0.2px;
    @include m.truncate;
    line-height: 1.3;
}

.wsftitleEmpty {
    font-size: 18px;
    font-weight: 300;
    color: var(--t2);
    opacity: 0.4;
}

.wsmeta {
    display: flex;
    align-items: center;
    gap: 5px;
    margin-top: 5px;
    flex-wrap: wrap;
}

.wsmetaId {
    display: inline-flex;
    align-items: center;
    font-size: 10px;
    font-weight: 700;
    font-family: var(--font-mono);
    color: #a78bfa;
    background: rgba(167, 139, 250, 0.12);
    border: 1px solid rgba(167, 139, 250, 0.3);
    border-radius: 4px;
    padding: 1px 6px;
    flex-shrink: 0;
    letter-spacing: 0.02em;
}

.wsmetaSep {
    color: var(--t2);
    opacity: 0.4;
    font-size: 13px;
}

.wsmetaDate {
    font-size: 13px;
    color: var(--t2);
    font-family: var(--font-mono);
}

.wsmetaHint {
    font-size: 13px;
    color: var(--t2);
    font-family: var(--font-mono);
    opacity: 0.6;
    font-style: italic;
}

// ── Tab bar ─────────────────────────────────────────────
.wsptabsWrap {
    position: relative;
    display: flex;
    align-items: center;
    border-bottom: 1px solid var(--bdr);
    background: var(--bg1);
    flex-shrink: 0;
}

.wsptabs {
    display: flex;
    padding: 0 16px;
    flex: 1;
    overflow-x: auto;
    scrollbar-width: none;
    -ms-overflow-style: none;

    &::-webkit-scrollbar {
        display: none;
    }
}

.tabFade {
    position: absolute;
    top: 0;
    bottom: 0;
    width: 28px;
    pointer-events: none;
    z-index: 1;
}

.tabFadeLeft {
    left: 28px;
    background: linear-gradient(90deg, var(--bg1), transparent);
}

.tabFadeRight {
    right: 28px;
    background: linear-gradient(270deg, var(--bg1), transparent);
}

.tabScrollBtn {
    flex-shrink: 0;
    align-self: stretch;
    width: 28px;
    display: flex;
    align-items: center;
    justify-content: center;
    border: none;
    background: var(--bg1);
    color: var(--t2);
    cursor: pointer;
    z-index: 2;
    transition: color 0.12s;

    svg {
        width: 13px;
        height: 13px;
    }

    &:hover {
        color: var(--t0);
    }
}

.tab {
    padding: 0 14px;
    height: 36px;
    display: flex;
    align-items: center;
    font-size: 14px;
    color: var(--t2);
    cursor: pointer;
    border: none;
    border-bottom: 2px solid transparent;
    background: transparent;
    font-family: var(--font-ui);
    transition: all 0.12s;
    user-select: none;
    white-space: nowrap;
    flex-shrink: 0;

    &:hover {
        color: var(--t1);
    }

    &.active {
        color: var(--blue);
        border-bottom-color: var(--blue);
        font-weight: 500;
    }
}

// ── Tab body ─────────────────────────────────────────────
.wsbody {
    flex: 1;
    overflow-y: auto;
    padding: 16px 18px;
    background: var(--bg0);
    @include m.scrollbar;
    animation: fadein 0.15s ease;
    display: flex;
    flex-direction: column;
}

// ── Empty state ──────────────────────────────────────────
.wsEmpty {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 10px;
    text-align: center;

    svg {
        width: 44px;
        height: 44px;
        color: var(--t2);
        opacity: 0.25;
        flex-shrink: 0;
        animation: float 3s ease-in-out infinite;
    }
}

.wsEmptyTitle {
    font-size: 16px;
    font-weight: 600;
    color: var(--t1);
}

.wsEmptyDesc {
    font-size: 14px;
    color: var(--t2);
    @include m.mono;
    line-height: 1.6;
    max-width: 200px;
}

// ── Loading spinner ──────────────────────────────────────
.wsSpinner {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 12px;
    color: var(--t2);
    font-size: 14px;
    @include m.mono;
}

.spinner {
    width: 28px;
    height: 28px;
    border: 2.5px solid var(--bdr2);
    border-top-color: var(--blue);
    border-radius: 50%;
    animation: wsSpin 0.7s linear infinite;
    flex-shrink: 0;
}

// ── Tab content wrapper ──────────────────────────────────
.tabContent {
    display: flex;
    flex-direction: column;
    gap: 10px;
    animation: fadein 0.15s ease;
    flex: 1;
    min-height: 0;
}

.tabEmpty {
    font-size: 14px;
    color: var(--t2);
    @include m.mono;
    padding: 24px 0;
    text-align: center;
}

// ── Summary — marked library output ─────────────────────
// Font sizes and spacing match Angular .markdown-content-inline exactly
.summaryMd {
    font-size: 14px;
    color: var(--t1);
    line-height: 1.7;

    // ── Paragraphs ──────────────────────────────────────
    // Angular: margin: 8px 0, line-height: 1.7
    p {
        margin: 8px 0;
        line-height: 1.7;

        &:first-child {
            margin-top: 0;
        }

        &:last-child {
            margin-bottom: 0;
        }
    }

    // ── Headings ─────────────────────────────────────────
    // Angular h1: 18px, margin 12px 0 8px 0, padding-bottom 6px
    h1 {
        font-size: 18px;
        font-weight: 700;
        color: var(--blue);
        margin: 12px 0 8px;
        padding-bottom: 6px;
        border-bottom: 2px solid rgba(139, 92, 246, 0.3);
        line-height: 1.3;
        position: relative;

        &:first-child {
            margin-top: 0;
        }

        &::before {
            content: '';
            position: absolute;
            bottom: -2px;
            left: 0;
            width: 60px;
            height: 2px;
            background: var(--blue);
        }
    }

    // Angular h2: 16px, margin 10px 0 6px 0, padding-left 12px
    h2 {
        font-size: 16px;
        font-weight: 700;
        color: #9f7aea;
        margin: 10px 0 6px;
        padding-left: 12px;
        line-height: 1.3;
        position: relative;

        &:first-child {
            margin-top: 0;
        }

        &::before {
            content: '';
            position: absolute;
            left: 0;
            top: 50%;
            transform: translateY(-50%);
            width: 4px;
            height: 18px;
            background: linear-gradient(180deg, var(--blue), #a78bfa);
            border-radius: 2px;
        }
    }

    // Angular h3: 14px, margin 8px 0 4px 0
    h3 {
        font-size: 14px;
        font-weight: 700;
        color: #a78bfa;
        margin: 8px 0 4px;
        line-height: 1.3;

        &:first-child {
            margin-top: 0;
        }
    }

    // Angular h4/h5/h6: 13px, margin 6px 0 4px 0
    h4,
    h5,
    h6 {
        font-size: 14px;
        font-weight: 600;
        color: #b794f4;
        margin: 6px 0 4px;
        line-height: 1.3;

        &:first-child {
            margin-top: 0;
        }
    }

    // ── Lists ────────────────────────────────────────────
    // Angular: margin 8px 0, padding-left 20px
    ul,
    ol {
        margin: 8px 0;
        padding-left: 20px;

        &:last-child {
            margin-bottom: 0;
        }
    }

    // Angular li: margin 4px 0, line-height 1.6, padding-left 8px
    li {
        margin: 4px 0;
        line-height: 1.6;
        padding-left: 8px;

        ul,
        ol {
            margin-top: 4px;
            margin-bottom: 4px;
        }
    }

    ul li::marker {
        color: var(--blue);
        font-weight: bold;
    }

    ol li::marker {
        color: var(--blue);
        font-weight: bold;
    }

    // ── Inline formatting ────────────────────────────────
    // Angular strong: color primary, background gradient, padding 2px 4px
    strong {
        font-weight: 700;
        color: var(--blue);
        background: linear-gradient(135deg,
                rgba(139, 92, 246, 0.1),
                rgba(167, 139, 250, 0.05));
        padding: 2px 4px;
        border-radius: 3px;
    }

    em {
        font-style: italic;
        color: #a78bfa;
    }

    // ── HR ───────────────────────────────────────────────
    // Angular: height 2px, gradient, margin 16px 0, opacity 0.3
    hr {
        border: none;
        height: 2px;
        background: linear-gradient(90deg, transparent, var(--blue), transparent);
        margin: 16px 0;
        opacity: 0.3;
    }

    // ── Blockquote ───────────────────────────────────────
    // Angular: border-left 3px, padding 12px 12px 12px 16px, margin 12px 0
    blockquote {
        border-left: 3px solid var(--blue);
        margin: 12px 0;
        padding: 12px 12px 12px 16px;
        background: linear-gradient(135deg,
                rgba(139, 92, 246, 0.08),
                rgba(167, 139, 250, 0.05));
        border-radius: 0 6px 6px 0;
        font-style: italic;
        opacity: 0.9;
        position: relative;

        &::before {
            content: '"';
            position: absolute;
            top: 8px;
            left: 8px;
            font-size: 32px;
            color: var(--blue);
            opacity: 0.3;
            font-family: Georgia, serif;
            line-height: 0;
        }

        // marked wraps blockquote text in <p>
        p {
            margin: 0;
            line-height: 1.6;
        }
    }

    // ── Inline code ──────────────────────────────────────
    // Angular: padding 3px 8px, font-size 0.9em (~11.7px), border rgba purple
    code {
        font-family: var(--font-mono);
        font-size: 0.9em;
        background: linear-gradient(135deg,
                rgba(139, 92, 246, 0.15),
                rgba(167, 139, 250, 0.1));
        border: 1px solid rgba(139, 92, 246, 0.3);
        border-radius: 4px;
        padding: 3px 8px;
        color: var(--blue);
        font-weight: 500;
    }

    // ── Code block ───────────────────────────────────────
    // Angular: padding 14px, margin 12px 0, border rgba purple
    pre {
        background: linear-gradient(135deg,
                rgba(139, 92, 246, 0.08),
                rgba(167, 139, 250, 0.05));
        border: 1px solid rgba(139, 92, 246, 0.25);
        border-radius: 8px;
        padding: 14px;
        overflow-x: auto;
        margin: 12px 0;
        position: relative;

        &:last-child {
            margin-bottom: 0;
        }

        // Angular pre::before — purple left accent bar
        &::before {
            content: '';
            position: absolute;
            top: 0;
            left: 0;
            width: 4px;
            height: 100%;
            background: linear-gradient(180deg, var(--blue), #a78bfa);
            border-radius: 8px 0 0 8px;
        }

        code {
            background: transparent;
            border: none;
            padding: 0;
            font-size: 0.9em;
            color: var(--t1);
            font-weight: normal;
        }
    }

    // ── Tables ───────────────────────────────────────────
    table {
        width: 100%;
        border-collapse: collapse;
        margin: 12px 0;
        border-radius: 6px;
        overflow: hidden;
    }

    th,
    td {
        padding: 10px 12px;
        border: 1px solid rgba(139, 92, 246, 0.2);
        text-align: left;
    }

    th {
        background: linear-gradient(135deg,
                rgba(139, 92, 246, 0.15),
                rgba(167, 139, 250, 0.1));
        font-weight: 700;
        color: var(--blue);
        border-bottom: 2px solid rgba(139, 92, 246, 0.3);
    }

    tr:hover {
        background: rgba(139, 92, 246, 0.03);
    }

    // ── Links ────────────────────────────────────────────
    a {
        color: var(--blue);
        text-decoration: none;
        font-weight: 500;
        border-bottom: 1px solid transparent;
        transition: border-color 0.2s, color 0.2s;

        &:hover {
            color: #9f7aea;
            border-bottom-color: var(--blue);
        }
    }

    // First child: no top margin
    >*:first-child {
        margin-top: 0;
    }

    // Last child: no bottom margin
    >*:last-child {
        margin-bottom: 0;
    }
}

// ── Keywords — gradient chips ────────────────────────────
.keywordGrid {
    display: flex;
    flex-wrap: wrap;
    gap: 7px;
}

.keywordPill {
    display: inline-flex;
    align-items: center;
    padding: 6px 12px;
    border-radius: 8px;
    font-size: 13px;
    font-weight: 600;
    @include m.mono;
    color: #fff;
    // background set inline via style prop (gradient)
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.15);
    animation: chipFadeIn 0.25s ease-out;
    position: relative;
    overflow: hidden;
    transition: transform 0.2s, box-shadow 0.2s;
    cursor: default;

    // shine on hover
    &::before {
        content: '';
        position: absolute;
        top: 0;
        left: -100%;
        width: 100%;
        height: 100%;
        background: linear-gradient(90deg,
                transparent,
                rgba(255, 255, 255, 0.25),
                transparent);
        transition: left 0.42s ease;
    }

    &:hover {
        transform: translateY(-2px);
        box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);

        &::before {
            left: 100%;
        }
    }
}

// ── Assessment — progress bar ────────────────────────────
.assessProgress {
    display: flex;
    align-items: center;
    gap: 10px;
}

.assessProgressTrack {
    flex: 1;
    height: 4px;
    background: var(--bg3);
    border-radius: 99px;
    overflow: hidden;
}

.assessProgressFill {
    height: 100%;
    background: linear-gradient(90deg, var(--blue), #a78bfa);
    border-radius: 99px;
    transition: width 0.3s ease;
}

.assessProgressLabel {
    font-size: 13px;
    font-family: var(--font-mono);
    font-weight: 700;
    color: var(--blue);
    white-space: nowrap;
    flex-shrink: 0;
}

// ── Assessment — question card ───────────────────────────
.faqCard {
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--r);
    padding: 12px 14px;
    display: flex;
    flex-direction: column;
    gap: 9px;
    animation: fadeInUp 0.2s ease-out;
    transition: border-color 0.15s, box-shadow 0.15s;

    &:hover {
        border-color: var(--blue-bdr);
        box-shadow: 0 2px 8px rgba(139, 92, 246, 0.07);
    }
}

.faqQ {
    font-size: 14px;
    font-weight: 600;
    color: var(--t0);
    line-height: 1.5;
    display: flex;
    gap: 8px;
    align-items: flex-start;
}

.faqNum {
    font-size: 13px;
    font-weight: 700;
    color: var(--blue);
    background: var(--blue-dim);
    border: 1px solid var(--blue-bdr);
    border-radius: 4px;
    padding: 1px 6px;
    flex-shrink: 0;
    margin-top: 2px;
    @include m.mono;
}

.faqOptions {
    display: flex;
    flex-direction: column;
    gap: 6px;
}

// ── Option button — base ─────────────────────────────────
.faqOpt {
    display: flex;
    align-items: center;
    gap: 9px;
    font-size: 13px;
    color: var(--t1);
    padding: 7px 10px;
    border-radius: 6px;
    border: 2px solid var(--bdr);
    background: var(--bg0);
    cursor: pointer;
    text-align: left;
    font-family: var(--font-ui);
    width: 100%;
    transition: border-color 0.15s, background 0.15s, transform 0.15s;

    &:hover:not(:disabled) {
        border-color: var(--blue-bdr);
        background: var(--blue-dim);
        transform: translateX(2px);
    }

    &:disabled {
        cursor: default;
    }
}

// Circle key badge
.faqOptKey {
    width: 22px;
    height: 22px;
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    background: var(--bg3);
    border: 1.5px solid var(--bdr2);
    font-size: 13px;
    font-weight: 700;
    color: var(--t1);
    @include m.mono;
    transition: background 0.15s, border-color 0.15s, color 0.15s;
}

.faqOptVal {
    flex: 1;
    line-height: 1.45;
}

// ── Option states ────────────────────────────────────────
.faqOptSelected {
    border-color: var(--blue-bdr);
    background: var(--blue-dim);

    .faqOptKey {
        background: var(--blue);
        border-color: var(--blue);
        color: #fff;
    }
}

.faqOptCorrect {
    border-color: var(--green-bdr);
    background: var(--green-dim);

    .faqOptKey {
        background: var(--green);
        border-color: var(--green);
        color: #fff;
    }

    .faqOptVal {
        color: var(--green);
        font-weight: 600;
    }
}

.faqOptCorrectAlt {
    border-color: var(--green-bdr);
    background: var(--green-dim);

    .faqOptKey {
        background: var(--green);
        border-color: var(--green);
        color: #fff;
    }

    .faqOptVal {
        color: var(--green);
        font-weight: 600;
    }
}

.faqOptWrong {
    border-color: var(--red-bdr);
    background: var(--red-dim);

    .faqOptKey {
        background: var(--red);
        border-color: var(--red);
        color: #fff;
    }

    .faqOptVal {
        color: var(--red);
        font-weight: 600;
    }
}

// ── Check / X icons ──────────────────────────────────────
.faqCheckIcon,
.faqXIcon {
    width: 13px;
    height: 13px;
    flex-shrink: 0;
    margin-left: auto;
    animation: iconPop 0.25s ease-out;
}

.faqCheckIcon {
    color: var(--green);
}

.faqXIcon {
    color: var(--red);
}

// ── Explanation box ──────────────────────────────────────
.faqExplain {
    background: var(--bg0);
    border: 1px solid var(--bdr);
    border-left: 3px solid var(--blue-bdr);
    border-radius: 0 5px 5px 0;
    padding: 8px 10px;
    display: flex;
    flex-direction: column;
    gap: 4px;
    animation: explanationSlideIn 0.2s ease-out;
}

.faqExplainLabel {
    display: flex;
    align-items: center;
    gap: 5px;
    font-size: 13px;
    font-weight: 600;
    color: var(--blue);
    text-transform: uppercase;
    letter-spacing: 0.06em;
    @include m.mono;

    svg {
        width: 12px;
        height: 12px;
        flex-shrink: 0;
    }
}

.faqExplainText {
    font-size: 13px;
    color: var(--t2);
    line-height: 1.6;
    @include m.mono;
    margin: 0;
}

// ── Next / Finish button ─────────────────────────────────
.assessNextBtn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 5px;
    padding: 6px 14px; // matches .btnSm padding scale
    border-radius: var(--r);
    border: none;
    background: var(--blue);
    color: #fff;
    font-size: 13px;
    font-weight: 600;
    font-family: var(--font-ui);
    cursor: pointer;
    align-self: flex-end;
    animation: fadein 0.15s ease;
    box-shadow: 0 2px 6px rgba(91, 164, 239, 0.28);
    transition: opacity 0.12s, transform 0.15s, box-shadow 0.15s;

    svg {
        width: 12px;
        height: 12px;
    }

    &:hover {
        opacity: 0.9;
        transform: translateY(-1px);
        box-shadow: 0 4px 10px rgba(91, 164, 239, 0.35);
    }
}

.assessDoneBtn {
    background: var(--green);
    box-shadow: 0 2px 6px rgba(74, 222, 128, 0.28);

    &:hover {
        box-shadow: 0 4px 10px rgba(74, 222, 128, 0.35);
    }
}

// ── Complete screen ──────────────────────────────────────
.assessComplete {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    padding: 28px 16px;
    text-align: center;
    animation: completeFadeIn 0.3s ease-out;
}

.assessCompleteIcon {
    width: 56px;
    height: 56px;
    background: var(--green-dim);
    border: 2px solid var(--green-bdr);
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    animation: iconBounce 0.5s ease-out;

    svg {
        width: 32px;
        height: 32px;
        color: var(--green);
    }
}

.assessCompleteTitle {
    font-size: 16px;
    font-weight: 700;
    color: var(--t0);
}

.assessCompleteDesc {
    font-size: 14px;
    color: var(--t2);
    @include m.mono;
}

.assessRestartBtn {
    margin-top: 4px;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 20px;
    border-radius: var(--r);
    border: 1px solid var(--bdr2);
    background: transparent;
    color: var(--t1);
    font-size: 14px;
    font-family: var(--font-ui);
    cursor: pointer;
    transition: all 0.12s;

    &:hover {
        background: var(--bg3);
        color: var(--t0);
        border-color: var(--bdr3);
    }
}

// (tabToolbar and editIconBtn replaced by actionBtn/dropdown system below)

// ── Edit mode layout ─────────────────────────────────────
.editMode {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
}

.editLayout {
    display: flex;
    gap: 12px;
    flex: 1;
    min-height: 0;
    position: relative;
    padding-bottom: 52px; // room for fixed footer
}

.editCol {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.editColHeader {
    display: flex;
    align-items: center;
    justify-content: space-between;
}

.editColLabel {
    font-size: 11px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.07em;
    color: var(--t2);
}

.editColTag {
    font-size: 10px;
    font-weight: 500;
    color: var(--t2);
    opacity: 0.55;
    font-style: italic;
    text-transform: none;
    letter-spacing: 0;
}

.editTextarea {
    flex: 1;
    resize: none;
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--r);
    color: var(--t1);
    font-size: 13px;
    font-family: var(--font-mono);
    line-height: 1.6;
    padding: 10px 12px;
    outline: none;
    min-height: 280px;
    transition: border-color 0.15s, box-shadow 0.15s;

    &:focus {
        border-color: var(--blue-bdr);
        box-shadow: 0 0 0 2px rgba(91, 164, 239, 0.12);
    }
}

.editPreview {
    flex: 1;
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--r);
    padding: 10px 12px;
    overflow-y: auto;
    min-height: 280px;
}

// ── Fixed footer inside edit layout ──────────────────────
.editFooter {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 48px;
    display: flex;
    align-items: center;
    justify-content: flex-end;
    gap: 8px;
    border-top: 1px solid var(--bdr);
    background: var(--bg1);
    padding: 0 4px;
    border-radius: 0 0 var(--r) var(--r);
}

.cancelBtn {
    padding: 6px 14px;
    border-radius: var(--r);
    border: 1px solid var(--bdr2);
    background: transparent;
    color: var(--t1);
    font-size: 13px;
    font-family: var(--font-ui);
    cursor: pointer;
    transition: all 0.12s;

    &:hover:not(:disabled) {
        background: var(--bg3);
        border-color: var(--bdr3);
    }

    &:disabled {
        opacity: 0.4;
        cursor: default;
    }
}

.saveBtn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 16px;
    border-radius: var(--r);
    border: none;
    background: var(--blue);
    color: #fff;
    font-size: 13px;
    font-weight: 600;
    font-family: var(--font-ui);
    cursor: pointer;
    transition: opacity 0.12s;
    box-shadow: 0 2px 6px rgba(91, 164, 239, 0.28);

    &:hover:not(:disabled) {
        opacity: 0.88;
    }

    &:disabled {
        opacity: 0.55;
        cursor: default;
    }
}

// ── Inline spinner for save button ───────────────────────
.inlineSpinner {
    width: 11px;
    height: 11px;
    border: 1.5px solid rgba(255, 255, 255, 0.35);
    border-top-color: #fff;
    border-radius: 50%;
    animation: wsSpin 0.65s linear infinite;
    flex-shrink: 0;
}

// ── Tab toolbar — icon-only buttons ──────────────────────
.tabToolbar {
    display: flex;
    align-items: center;
    gap: 4px;
    justify-content: flex-end;
    margin-bottom: 6px;
}

.actionBtn {
    width: 28px;
    height: 28px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 7px;
    border: 1px solid var(--bdr2);
    background: transparent;
    color: var(--t2);
    cursor: pointer;
    padding: 0;
    transition: all 0.15s;
    flex-shrink: 0;

    svg {
        width: 13px;
        height: 13px;
    }

    &:hover {
        background: var(--blue-dim);
        border-color: var(--blue-bdr);
        color: var(--blue);
    }
}

.actionBtnActive {
    background: var(--blue-dim);
    border-color: var(--blue-bdr);
    color: var(--blue);
}

.successIcon {
    color: var(--green) !important;
}

// ── Dropdown ──────────────────────────────────────────────
.dropdownWrap {
    position: relative;
}

.dropdown {
    position: absolute;
    top: calc(100% + 5px);
    right: 0;
    min-width: 168px;
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--r);
    box-shadow: 0 6px 20px rgba(0, 0, 0, 0.18);
    z-index: 50;
    overflow: hidden;
    animation: ddFade 0.12s ease;
}

@keyframes ddFade {
    from {
        opacity: 0;
        transform: translateY(-4px);
    }

    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.dropdownLabel {
    font-size: 10px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.07em;
    color: var(--t2);
    padding: 7px 11px 4px;
    opacity: 0.6;
}

.dropdownItem {
    display: flex;
    align-items: center;
    gap: 9px;
    width: 100%;
    text-align: left;
    padding: 7px 11px;
    font-size: 13px;
    color: var(--t1);
    background: transparent;
    border: none;
    cursor: pointer;
    font-family: var(--font-ui);
    transition: background 0.1s, color 0.1s;

    &:hover {
        background: var(--blue-dim);
        color: var(--blue);
    }
}

.dropdownItemLabel {
    flex: 1;
    min-width: 0;
}

// Per-format colored glyph in copy/download menus
.fmtIcon {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 22px;
    height: 22px;
    border-radius: 6px;
    flex-shrink: 0;
    transition: transform 0.12s ease;

    svg {
        width: 13px;
        height: 13px;
    }
}

.dropdownItem:hover .fmtIcon {
    transform: scale(1.06);
}

// ── Keyword pill — new chip animation ────────────────────
.keywordPillNew {
    animation: chipFadeIn 0.25s ease-out;
}

// ── Assessment header row (mode tabs + toolbar) ───────────
.assessHeader {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 10px;
    gap: 8px;
}

// ── Mode toggle tabs ──────────────────────────────────────
.assessModeTabs {
    display: flex;
    align-items: center;
    gap: 3px;
    background: var(--bg2);
    border: 1px solid var(--bdr);
    border-radius: 8px;
    padding: 3px;
}

.assessModeTab {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 4px 10px;
    border-radius: 6px;
    border: none;
    background: transparent;
    color: var(--t2);
    font-size: 12px;
    font-weight: 500;
    font-family: var(--font-ui);
    cursor: pointer;
    transition: all 0.15s;

    svg {
        width: 12px;
        height: 12px;
        flex-shrink: 0;
    }

    &:hover {
        color: var(--t1);
    }
}

.assessModeTabActive {
    background: var(--bg0);
    color: var(--blue);
    font-weight: 600;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.12);

    svg {
        stroke: var(--blue);
    }
}

// ── View All list ─────────────────────────────────────────
.viewAllList {
    display: flex;
    flex-direction: column;
    gap: 14px;
    animation: fadein 0.15s ease;
}

.viewAllCard {
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--r);
    padding: 14px;
    display: flex;
    flex-direction: column;
    gap: 10px;

    &:hover {
        border-color: var(--blue-bdr);
        box-shadow: 0 2px 8px rgba(139, 92, 246, 0.07);
    }
}

.viewAllQ {
    font-size: 14px;
    font-weight: 600;
    color: var(--t0);
    line-height: 1.5;
    display: flex;
    gap: 8px;
    align-items: flex-start;
}

// Neutral option style for View All (non-correct options)
.viewAllOptNeutral {
    cursor: default !important;
    opacity: 0.65;

    &:hover {
        transform: none !important;
        border-color: var(--bdr) !important;
        background: var(--bg0) !important;
    }
}

// Correct key badge in View All mode
.faqOptKeyCorrect {
    background: var(--green) !important;
    border-color: var(--green) !important;
    color: #fff !important;
}

// ── Large screen overrides (> 1900px) ────────────────────
@media (min-width: 1920px) {
    .wspanel {
        width: 400px;
    }

    .wsftitle {
        font-size: 18px;
    }

    .wsftitleEmpty {
        font-size: 20px;
    }

    .wsmetaId {
        font-size: 11px;
    }

    .wsmetaDate {
        font-size: 14px;
    }

    .wsmetaHint {
        font-size: 14px;
    }

    .tab {
        font-size: 15px;
    }

    .wsEmptyTitle {
        font-size: 18px;
    }

    .wsEmptyDesc {
        font-size: 15px;
    }

    .wsSpinner {
        font-size: 15px;
    }

    .tabEmpty {
        font-size: 15px;
    }

    .summaryMd {
        font-size: 15px;

        h1 {
            font-size: 20px;
        }

        h2 {
            font-size: 17px;
        }

        h3 {
            font-size: 15px;
        }

        h4,
        h5,
        h6 {
            font-size: 15px;
        }
    }

    .keywordPill {
        font-size: 14px;
    }

    .assessProgressLabel {
        font-size: 14px;
    }

    .faqQ {
        font-size: 15px;
    }

    .faqNum {
        font-size: 14px;
    }

    .faqOpt {
        font-size: 14px;
    }

    .faqOptKey {
        font-size: 14px;
    }

    .faqExplainLabel {
        font-size: 14px;
    }

    .faqExplainText {
        font-size: 14px;
    }

    .assessNextBtn {
        font-size: 14px;
    }

    .assessCompleteTitle {
        font-size: 18px;
    }

    .assessCompleteDesc {
        font-size: 15px;
    }

    .assessRestartBtn {
        font-size: 15px;
    }

    .dropdownItem {
        font-size: 14px;
    }

    .dropdownLabel {
        font-size: 11px;
    }

    .assessModeTab {
        font-size: 13px;
    }

    .viewAllQ {
        font-size: 15px;
    }

    .editColLabel {
        font-size: 12px;
    }

    .editColTag {
        font-size: 11px;
    }

    .editTextarea {
        font-size: 14px;
    }

    .cancelBtn {
        font-size: 14px;
    }

    .saveBtn {
        font-size: 14px;
    }
}
// ═══════════════════════════════════════════════
// New content types — Short Answer, True/False,
// Timestamped Summary, Keyword Insights
// ═══════════════════════════════════════════════

// ── Short Answer ──────────────────────────────────
.revealAllBtn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    border-radius: 6px;
    border: 1px solid var(--bdr2);
    background: var(--bg0);
    color: var(--t1);
    font-size: 12.5px;
    font-weight: 600;
    font-family: var(--font-ui);
    cursor: pointer;
    transition: all 0.13s;

    svg { width: 14px; height: 14px; }

    &:hover {
        border-color: var(--blue-bdr);
        color: var(--blue);
        background: var(--blue-dim);
    }
}

.revealBtn {
    margin-top: 10px;
    padding: 7px 14px;
    border-radius: 6px;
    border: 1px dashed var(--bdr2);
    background: transparent;
    color: var(--t2);
    font-size: 12.5px;
    font-weight: 600;
    font-family: var(--font-ui);
    cursor: pointer;
    transition: all 0.13s;

    &:hover {
        border-color: var(--blue-bdr);
        border-style: solid;
        color: var(--blue);
        background: var(--blue-dim);
    }
}

.shortAnswerBox {
    margin-top: 10px;
    padding: 12px 14px;
    background: rgba(79, 172, 254, 0.08);
    border-left: 3px solid #4facfe;
    border-radius: 0 8px 8px 0;
}

.shortAnswerLabel {
    font-size: 11px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    color: #4facfe;
    margin-bottom: 4px;
    @include m.mono;
}

.shortAnswerText {
    font-size: 13.5px;
    color: var(--t0);
    line-height: 1.55;
    margin: 0;
}

// ── True / False ──────────────────────────────────
.tfOptions {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 10px;
    margin-top: 12px;
}

.tfOpt {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    font-size: 14px;
    font-weight: 700;
    color: var(--t1);
    padding: 16px 10px;
    border-radius: 8px;
    border: 2px solid var(--bdr);
    background: var(--bg0);
    cursor: pointer;
    text-align: center;
    font-family: var(--font-ui);
    transition: border-color 0.15s, background 0.15s, transform 0.15s;

    &:hover:not(:disabled) {
        border-color: var(--blue-bdr);
        background: var(--blue-dim);
        transform: translateY(-1px);
    }

    &:disabled {
        cursor: default;
    }
}

.tfOptLabel {
    letter-spacing: 0.02em;
}

// ── Timestamped Summary ───────────────────────────
.tsList {
    display: flex;
    flex-direction: column;
    margin-top: 4px;
}

.tsRow {
    display: flex;
    gap: 14px;
}

.tsRail {
    display: flex;
    flex-direction: column;
    align-items: center;
    flex-shrink: 0;
    padding-top: 6px;
}

.tsDot {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    background: var(--blue);
    box-shadow: 0 0 0 3px var(--blue-dim);
    flex-shrink: 0;
}

.tsLine {
    width: 2px;
    flex: 1;
    background: var(--bdr2);
    margin-top: 2px;
}

.tsCard {
    flex: 1;
    padding-bottom: 20px;
}

.tsRange {
    font-size: 12px;
    font-weight: 700;
    color: var(--blue);
    margin-bottom: 4px;
    @include m.mono;
}

.tsArrow {
    color: var(--t2);
    margin: 0 2px;
}

.tsText {
    font-size: 13.5px;
    color: var(--t1);
    line-height: 1.6;
    margin: 0;
}

// ── Keyword Insights: sub-nav (scrollable, one line) ──
.kiSubNavWrap {
    position: relative;
    display: flex;
    align-items: center;
    margin-bottom: 16px;
    padding-bottom: 12px;
    border-bottom: 1px solid var(--bdr);

    .tabScrollBtn {
        background: var(--bg0);
    }

    .tabFadeLeft {
        background: linear-gradient(90deg, var(--bg0), transparent);
    }

    .tabFadeRight {
        background: linear-gradient(270deg, var(--bg0), transparent);
    }
}

.kiSubNav {
    display: flex;
    gap: 4px;
    flex: 1;
    overflow-x: auto;
    scrollbar-width: none;
    -ms-overflow-style: none;

    &::-webkit-scrollbar {
        display: none;
    }
}

.kiSubTab {
    padding: 5px 11px;
    border-radius: 99px;
    border: 1px solid var(--bdr2);
    background: var(--bg0);
    color: var(--t2);
    font-size: 12px;
    font-weight: 600;
    font-family: var(--font-ui);
    cursor: pointer;
    transition: all 0.13s;
    white-space: nowrap;
    flex-shrink: 0;

    &:hover {
        color: var(--t0);
        border-color: var(--bdr3, var(--bdr2));
    }
}

.kiSubTabActive {
    background: var(--blue-dim);
    border-color: var(--blue-bdr);
    color: var(--blue);
}

.kiBody {
    min-height: 200px;
    flex: 1;
    display: flex;
    flex-direction: column;
}

// ── Keyword Insights: node-link graph (React Flow mindmap) ──
.graphWrap {
    width: 100%;
    flex: 1;
    min-height: 460px;
    border-radius: 10px;
    overflow: hidden;
    border: 1px solid var(--bdr);
    background: var(--bg0);

    :global(.react-flow__attribution) {
        display: none;
    }

    :global(.react-flow__controls) {
        background: var(--bg1);
        border: 1px solid var(--bdr2);
        border-radius: 8px;
        overflow: hidden;
        box-shadow: none;
    }

    :global(.react-flow__controls-button) {
        background: var(--bg1);
        border-bottom: 1px solid var(--bdr2);
        color: var(--t1);

        &:hover {
            background: var(--bg3);
        }

        svg {
            fill: var(--t1);
        }
    }

    :global(.react-flow__minimap) {
        border-radius: 8px;
        border: 1px solid var(--bdr2);
        overflow: hidden;
    }

    :global(.react-flow__edge-path) {
        stroke: var(--bdr2);
    }

    :global(.react-flow__edge:hover) .react-flow__edge-path,
    :global(.react-flow__edge.selected) .react-flow__edge-path {
        stroke: var(--blue);
    }

    :global(.react-flow__edge-text) {
        fill: var(--t2);
    }
}

.rfNode {
    border-radius: 50%;
    background: var(--blue-dim);
    border: 1.6px solid var(--blue);
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: grab;
    position: relative;
    transition: background 0.15s;

    &:active {
        cursor: grabbing;
    }

    &:hover {
        background: var(--blue);

        .rfNodeLabel {
            color: var(--t0);
        }
    }
}

.rfNodeLabel {
    position: absolute;
    top: 100%;
    left: 50%;
    transform: translateX(-50%);
    margin-top: 5px;
    font-size: 11px;
    font-weight: 500;
    line-height: 14px;
    color: var(--t1);
    text-align: center;
    // keep-all (not overflow-wrap:break-word) — CJK text has no spaces
    // within a word, so break-word was breaking it between individual
    // characters whenever the estimated box was a little too narrow,
    // which renders as letters stacking vertically one per line.
    word-break: keep-all;
    overflow-wrap: normal;
    font-family: var(--font-ui);
    pointer-events: none;
}

.rfHandle {
    width: 6px !important;
    height: 6px !important;
    min-width: 0 !important;
    background: transparent !important;
    border: none !important;
    opacity: 0;
}

// ── Keyword Insights: matrix / heatmap grid ───────
.matrixScroll {
    overflow-x: auto;
    @include m.scrollbar;
}

.matrixGrid {
    display: grid;
    gap: 2px;
    width: max-content;
    min-width: 100%;
}

.matrixCorner {
    background: transparent;
}

.matrixColHead {
    font-size: 10px;
    color: var(--t2);
    writing-mode: vertical-rl;
    transform: rotate(180deg);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-height: 90px;
    padding: 4px 2px;
    text-align: left;
    @include m.mono;
}

.matrixRowHead {
    font-size: 11.5px;
    color: var(--t1);
    padding: 4px 8px 4px 0;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 140px;
    display: flex;
    align-items: center;
}

.matrixCell {
    min-width: 28px;
    aspect-ratio: 1;
    border-radius: 3px;
    border: 1px solid var(--bdr);
}

// ── Keyword Insights: word cloud ──────────────────
.wordCloud {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    justify-content: center;
    gap: 12px 18px;
    padding: 24px 12px;
}

// Server-rendered word cloud image, wrapped in the pan/zoom viewer below.
.wordCloudImageWrap {
    flex: 1;
    display: flex;
    min-height: 380px;
}

// Pan/zoom viewer — drag to pan (once zoomed in), scroll to zoom, double
// click to reset. Same interaction model as the Knowledge Graph tab.
.zoomPanWrap {
    position: relative;
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
    border-radius: 10px;
    border: 1px solid var(--bdr);
    background: var(--bg0);
    touch-action: none;
    user-select: none;
}

.wordCloudImage {
    max-width: 100%;
    max-height: 100%;
    border-radius: 8px;
    pointer-events: none;
}

.zoomControls {
    position: absolute;
    right: 10px;
    bottom: 10px;
    display: flex;
    flex-direction: column;
    background: var(--bg1);
    border: 1px solid var(--bdr2);
    border-radius: 8px;
    overflow: hidden;
    z-index: 2;
}

.zoomControlBtn {
    width: 28px;
    height: 28px;
    display: flex;
    align-items: center;
    justify-content: center;
    border: none;
    border-bottom: 1px solid var(--bdr2);
    background: var(--bg1);
    color: var(--t1);
    cursor: pointer;
    transition: background 0.12s;

    svg {
        width: 14px;
        height: 14px;
    }

    &:last-child {
        border-bottom: none;
    }

    &:hover:not(:disabled) {
        background: var(--bg3);
    }

    &:disabled {
        opacity: 0.35;
        cursor: not-allowed;
    }
}

.wcWord {
    font-weight: 700;
    font-family: var(--font-ui);
    line-height: 1;
    cursor: default;
    transition: transform 0.15s;

    &:hover {
        transform: scale(1.08);
    }
}

// ── Keyword Insights: clusters ────────────────────
.clusterGrid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 12px;
}

.clusterCard {
    padding: 12px 14px;
    background: var(--bg0);
    border: 1px solid var(--bdr);
    border-radius: 10px;
}

.clusterTitle {
    font-size: 11px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: var(--t2);
    margin-bottom: 8px;
    @include m.mono;
}

.clusterChips {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
}

.clusterChip {
    padding: 3px 9px;
    border-radius: 99px;
    background: var(--blue-dim);
    border: 1px solid var(--blue-bdr);
    color: var(--blue);
    font-size: 11.5px;
    font-weight: 600;
}

// ── Keyword Insights: frequency bars ──────────────
.freqList {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.freqRow {
    display: grid;
    grid-template-columns: 130px 1fr 130px;
    align-items: center;
    gap: 12px;
}

.freqLabel {
    font-size: 12.5px;
    color: var(--t1);
    font-weight: 600;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.freqBarTrack {
    height: 10px;
    background: var(--bg0);
    border: 1px solid var(--bdr);
    border-radius: 99px;
    overflow: hidden;
}

.freqBarFill {
    height: 100%;
    background: linear-gradient(90deg, var(--blue), #a78bfa);
    border-radius: 99px;
}

.freqMeta {
    display: flex;
    gap: 8px;
    align-items: center;
    font-size: 11px;
    color: var(--t1);
    justify-content: flex-end;
    @include m.mono;
}

.freqDim {
    color: var(--t2);
}

// ── Keyword Insights: importance/complexity columns ──
// Sorted rows in normal document flow, grouped by complexity — this can
// never overlap the way freely-positioned dots + floating labels could.

// Explains what "m/n" (importance score) and "f× freq" (occurrence count)
// mean, shown once above the columns.
.importanceHint {
    display: flex;
    align-items: flex-start;
    gap: 7px;
    font-size: 12px;
    color: var(--t2);
    line-height: 1.5;
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--r);
    padding: 8px 10px;
    margin-bottom: 12px;

    svg {
        width: 14px;
        height: 14px;
        flex-shrink: 0;
        margin-top: 1px;
        color: var(--blue);
    }
}

// Explains what the X-axis (time, in whatever unit the backend sends)
// and Y-axis (keyword) represent, shown once above the timeline grid.
.timelineAxisHint {
    display: flex;
    align-items: flex-start;
    gap: 7px;
    font-size: 12px;
    color: var(--t2);
    line-height: 1.5;
    background: var(--bg1);
    border: 1px solid var(--bdr);
    border-radius: var(--r);
    padding: 8px 10px;
    margin-bottom: 12px;

    svg {
        width: 14px;
        height: 14px;
        flex-shrink: 0;
        margin-top: 1px;
        color: var(--blue);
    }
}

.importanceColumns {
    display: flex;
    gap: 20px;
    flex-wrap: wrap;
    align-items: flex-start;
}

.importanceColumn {
    flex: 1;
    min-width: 200px;
}

.importanceColumnHead {
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 12px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    padding-bottom: 8px;
    margin-bottom: 10px;
    border-bottom: 2px solid;
    @include m.mono;
}

.importanceColumnCount {
    font-size: 11px;
    font-weight: 600;
    color: var(--t2);
    background: var(--bg0);
    border: 1px solid var(--bdr2);
    border-radius: 99px;
    padding: 1px 7px;
}

.importanceRows {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.importanceRow {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
    row-gap: 4px;
}

.importanceDot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    flex-shrink: 0;
}

.importanceKeyword {
    font-size: 12.5px;
    color: var(--t0);
    font-weight: 600;
    width: 84px;
    flex-shrink: 0;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.importanceBarTrack {
    flex: 1 1 60px;
    height: 7px;
    background: var(--bg0);
    border: 1px solid var(--bdr);
    border-radius: 99px;
    overflow: hidden;
    min-width: 0;
}

.importanceBarFill {
    height: 100%;
    border-radius: 99px;
}

// Value + frequency grouped together so they size to their own content
// and never fight the bar track for space — the bar shrinks first, this
// block never wraps or overlaps its neighbors.
.importanceMeta {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-shrink: 0;
    white-space: nowrap;
    margin-left: auto;
}

.importanceValue {
    font-size: 11px;
    font-weight: 600;
    color: var(--t2);
    flex-shrink: 0;
    white-space: nowrap;
    @include m.mono;
}

.importanceFrequency {
    font-size: 11px;
    color: var(--t2);
    flex-shrink: 0;
    white-space: nowrap;
    @include m.mono;
}

// ── Keyword Insights: glossary ────────────────────
.glossaryList {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.glossaryRow {
    padding: 10px 14px;
    background: var(--bg0);
    border: 1px solid var(--bdr);
    border-radius: 8px;
}

.glossaryTerm {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 13.5px;
    font-weight: 700;
    color: var(--t0);
    margin-bottom: 4px;
}

.glossaryTime {
    font-size: 10.5px;
    font-weight: 600;
    color: var(--t2);
    background: var(--bg1);
    border: 1px solid var(--bdr2);
    border-radius: 99px;
    padding: 1px 7px;
    @include m.mono;
}

.glossaryDef {
    font-size: 13px;
    color: var(--t1);
    line-height: 1.55;
}


