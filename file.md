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
import type { FileResult, WordCloudData, ImportanceComplexityData, GlossaryData, KeywordInsights } from './UploadInfer';
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


// ── Keyword Insights: shared node-link graph (used for Knowledge Graph & Prerequisites) ──
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
const LABEL_CHAR_W = 6.1; // ~px per character at the label's font size (tuned for Latin script)
const LABEL_CHAR_W_CJK = 11.5; // Korean/Japanese/Chinese glyphs render roughly full-width — much wider than Latin
const LABEL_LINE_H = 14;

// Korean (Hangul), Japanese (Hiragana/Katakana/Kanji), and Chinese ranges.
// CJK scripts have line-break opportunities BETWEEN EVERY CHARACTER by
// default (per Unicode UAX#14) — unlike Latin, which only breaks at
// whitespace/hyphens. Left unhandled, a narrow maxWidth wraps Korean
// labels one character per line, which visually looks like vertical text.
const CJK_RE = /[\uac00-\ud7a3\u3040-\u30ff\u3400-\u4dbf\u4e00-\u9fff]/;

function estimateNodeBox(label: string, circleSize: number) {
    const isCjk = CJK_RE.test(label);
    const hasBreakPoint = !isCjk && /[\s-]/.test(label);
    const charW = isCjk ? LABEL_CHAR_W_CJK : LABEL_CHAR_W;
    const rawTextWidth = label.length * charW + 8;
    const capWidth = hasBreakPoint ? LABEL_MAX_W : LABEL_HARD_MAX_W;
    const labelWidth = Math.min(capWidth, Math.max(rawTextWidth, 44));
    // CJK labels always render as a single line — never wrapped
    const lines = hasBreakPoint ? Math.max(1, Math.ceil(rawTextWidth / LABEL_MAX_W)) : 1;
    const w = Math.max(circleSize, labelWidth, 72) + 16;
    const h = circleSize + 16 + lines * LABEL_LINE_H;
    return { w, h, labelWidth };
}

const MindMapNode: React.FC<{ data: { label: string; value?: number; labelWidth?: number } }> = ({ data }) => {
    const size = 34 + Math.min(26, (data.value ?? 1) * 4);
    const isCjk = CJK_RE.test(data.label);
    // Korean/Japanese/Chinese labels never wrap — always a single line,
    // regardless of whether they contain spaces.
    const hasBreakPoint = !isCjk && /[\s-]/.test(data.label);
    return (
        <div className={styles.rfNode} style={{ width: size, height: size }} title={data.label}>
            <Handle type="target" position={Position.Top} id="top-target" className={styles.rfHandle} />
            <Handle type="source" position={Position.Bottom} id="bottom-source" className={styles.rfHandle} />
            <span
                className={styles.rfNodeLabel}
                style={{
                    maxWidth: data.labelWidth ?? LABEL_MAX_W,
                    whiteSpace: hasBreakPoint ? 'normal' : 'nowrap',
                    wordBreak: isCjk ? 'keep-all' : 'normal',
                    overflowWrap: isCjk ? 'normal' : undefined,
                }}
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
        g.setEdge(s, tg, label ? { width: Math.min(120, label.length * LABEL_CHAR_W + 12), height: LABEL_LINE_H + 8, labelpos: 'c' } : {});
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
            g.setEdge(e.source, e.target, label ? { width: Math.min(120, label.length * LABEL_CHAR_W + 12), height: LABEL_LINE_H + 8, labelpos: 'c' } : {});
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
const TimelineView: React.FC<{ data: TimelineData }> = ({ data }) => {
    const { t } = useTranslation();
    const labels = data.labels ?? [];
    const datasets = data.datasets ?? [];
    if (!labels.length || !datasets.length) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;

    const max = Math.max(1, ...datasets.flatMap(d => d.data.filter(v => typeof v === 'number')));
    const ROW_LABEL_W = 140;
    const COL_W = 56;

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
        <>
        <div style={{ overflowX: 'auto', paddingBottom: 4 }}>
            <div style={{ display: 'grid', gridTemplateColumns: `${ROW_LABEL_W}px repeat(${labels.length}, minmax(${COL_W}px, 1fr))`, width: 'max-content', minWidth: '100%' }}>
                {/* Corner cell */}
                <div />
                {/* Time labels — header row, horizontal across the top */}
                {labels.map((lab, li) => (
                    <div
                        key={li}
                        title={lab}
                        style={{
                            ...horizontalLabel,
                            fontSize: 10.5, fontWeight: 600, color: 'var(--t2)',
                            textAlign: 'center', padding: '0 4px 8px',
                        }}
                    >
                        {lab}
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
                                    title={`${ds.label} \u00b7 ${lab}: ${v}`}
                                    style={{
                                        height: 32, margin: 2, borderRadius: 4,
                                        background: `rgba(91, 164, 239, ${alpha})`,
                                        display: 'flex', alignItems: 'center', justifyContent: 'center',
                                        fontSize: 10, color: alpha > 0.5 ? '#fff' : 'var(--t2)',
                                    }}
                                >
                                    {v > 0 ? v : ''}
                                </div>
                            );
                        })}
                    </React.Fragment>
                ))}
            </div>
        </div>
        {/* Legend — same intensity scale as the cells above, GitHub contribution-graph style */}
        <div style={{ display: 'flex', alignItems: 'center', justifyContent: 'flex-end', gap: 5, marginTop: 10, fontSize: 11, color: 'var(--t2)' }}>
            <span>{t('uploadInfer.workspacePanel.legendLess', 'Less')}</span>
            {[0, 0.22, 0.4, 0.6, 0.78, 1].map((alpha, i) => (
                <div
                    key={i}
                    style={{
                        width: 12, height: 12, borderRadius: 3,
                        background: alpha === 0 ? 'var(--bg2)' : `rgba(91, 164, 239, ${alpha})`,
                        border: alpha === 0 ? '1px solid var(--bdr2)' : 'none',
                    }}
                />
            ))}
            <span>{t('uploadInfer.workspacePanel.legendMore', 'More')}</span>
        </div>
        </>
    );
};

// ── Keyword Insights: word cloud ──
const WordCloudView: React.FC<{ data: WordCloudData }> = ({ data }) => {
    const { t } = useTranslation();
    if (!data.wordcloud_image) return <div className={styles.tabEmpty}>{t('uploadInfer.workspacePanel.noData')}</div>;
    return (
        <div className={styles.wordCloudImageWrap} style={{ display: 'flex', justifyContent: 'center', alignItems: 'center', padding: 16 }}>
            <img
                src={data.wordcloud_image}
                alt={t('uploadInfer.workspacePanel.kiTabs.wordcloud')}
                className={styles.wordCloudImage}
                style={{ maxWidth: '100%', height: 'auto', borderRadius: 8 }}
            />
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
                                    <span className={styles.importanceValue}>{it.importance}/{IMPORTANCE_MAX}</span>
                                    <span
                                        className={styles.importanceFrequency}
                                        style={{ fontSize: 11, color: 'var(--t2)', flexShrink: 0, whiteSpace: 'nowrap' }}
                                        title={t('uploadInfer.workspacePanel.frequencyTitle')}
                                    >
                                        {t('uploadInfer.workspacePanel.frequencyCount', { count: it.frequency })}
                                    </span>
                                </div>
                            ))}
                        </div>
                    </div>
                );
            })}
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
const KI_SUBTABS = ['graph', 'wordcloud', 'timeline', 'prerequisites', 'importance', 'glossary'] as const;
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
                {sub === 'prerequisites' && (data.prerequisites
                    ? <NodeLinkGraph nodes={data.prerequisites.nodes} edges={data.prerequisites.edges.map(e => ({ source: e.prerequisite, target: e.enables, label: e.reason }))} />
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
