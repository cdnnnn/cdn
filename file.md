//Ticketselect.tsx
import { useEffect, useRef, useState, type KeyboardEvent } from 'react';
import { Check, ChevronDown } from 'lucide-react';
import styles from './CreateTicketDrawer.module.scss';

export interface TicketSelectOption {
  value: string;
  label: string;
  /** Optional accent hex — renders as a small dot next to the label (priority colors, avatar colors, etc). */
  accent?: string;
  /** Optional short text rendered inside the dot instead of a plain circle (e.g. avatar initials). */
  glyph?: string;
}

interface TicketSelectProps {
  value: string;
  options: TicketSelectOption[];
  placeholder?: string;
  onChange: (value: string) => void;
  disabled?: boolean;
  'aria-label'?: string;
}

/**
 * A small custom listbox styled to match the modal it lives in — replaces
 * the browser's native <select> (which can't be themed consistently across
 * platforms) with the app's own dropdown chrome: a bordered trigger button
 * and a floating menu with accent dots, a check on the active row, hover
 * states, and full arrow-key navigation.
 */
export default function TicketSelect({
  value,
  options,
  placeholder = 'Select…',
  onChange,
  disabled = false,
  ...aria
}: TicketSelectProps) {
  const [open, setOpen] = useState(false);
  const [highlight, setHighlight] = useState(0);
  const rootRef = useRef<HTMLDivElement | null>(null);
  const menuRef = useRef<HTMLDivElement | null>(null);
  const selected = options.find((o) => o.value === value) ?? null;

  const openMenu = () => {
    if (disabled) return;
    const idx = Math.max(
      0,
      options.findIndex((o) => o.value === value)
    );
    setHighlight(idx);
    setOpen(true);
  };

  useEffect(() => {
    if (!open) return;
    const onDoc = (e: MouseEvent) => {
      if (rootRef.current && !rootRef.current.contains(e.target as Node)) setOpen(false);
    };
    document.addEventListener('mousedown', onDoc);
    return () => document.removeEventListener('mousedown', onDoc);
  }, [open]);

  // Keep the highlighted row scrolled into view as it changes via keyboard.
  useEffect(() => {
    if (!open) return;
    const el = menuRef.current?.querySelectorAll('[role="option"]')[highlight] as
      | HTMLElement
      | undefined;
    el?.scrollIntoView({ block: 'nearest' });
  }, [open, highlight]);

  const choose = (v: string) => {
    onChange(v);
    setOpen(false);
  };

  const onTriggerKeyDown = (e: KeyboardEvent) => {
    if (disabled) return;
    if (!open && (e.key === 'ArrowDown' || e.key === 'ArrowUp' || e.key === 'Enter' || e.key === ' ')) {
      e.preventDefault();
      openMenu();
      return;
    }
    if (!open) return;
    if (e.key === 'Escape') {
      e.preventDefault();
      setOpen(false);
      return;
    }
    if (e.key === 'ArrowDown') {
      e.preventDefault();
      setHighlight((i) => (i + 1) % options.length);
      return;
    }
    if (e.key === 'ArrowUp') {
      e.preventDefault();
      setHighlight((i) => (i - 1 + options.length) % options.length);
      return;
    }
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      const opt = options[highlight];
      if (opt) choose(opt.value);
      return;
    }
    if (e.key === 'Home') {
      e.preventDefault();
      setHighlight(0);
      return;
    }
    if (e.key === 'End') {
      e.preventDefault();
      setHighlight(options.length - 1);
    }
  };

  return (
    <div className={styles['select']} ref={rootRef}>
      <button
        type="button"
        className={`${styles['select-trigger']} ${open ? styles['select-trigger--open'] : ''}`}
        onClick={() => (open ? setOpen(false) : openMenu())}
        onKeyDown={onTriggerKeyDown}
        disabled={disabled}
        aria-haspopup="listbox"
        aria-expanded={open}
        aria-label={aria['aria-label']}
      >
        <span className={styles['select-value']}>
          {selected ? (
            <>
              {selected.accent && (
                <span className={styles['select-dot']} style={{ background: selected.accent }}>
                  {selected.glyph}
                </span>
              )}
              {selected.label}
            </>
          ) : (
            <span className={styles['select-placeholder']}>{placeholder}</span>
          )}
        </span>
        <ChevronDown size={14} className={styles['select-caret']} />
      </button>

      {open && (
        <div className={styles['select-menu']} role="listbox" ref={menuRef}>
          {options.length === 0 && <div className={styles['select-empty']}>No options</div>}
          {options.map((o, i) => {
            const active = o.value === value;
            const isHighlighted = i === highlight;
            return (
              <button
                key={o.value}
                type="button"
                role="option"
                aria-selected={active}
                className={[
                  styles['select-option'],
                  active ? styles['select-option--active'] : '',
                  isHighlighted ? styles['select-option--highlight'] : '',
                ].join(' ')}
                onMouseEnter={() => setHighlight(i)}
                onClick={() => choose(o.value)}
              >
                {o.accent && (
                  <span className={styles['select-dot']} style={{ background: o.accent }}>
                    {o.glyph}
                  </span>
                )}
                <span className={styles['select-option-label']}>{o.label}</span>
                {active && <Check size={13} className={styles['select-check']} />}
              </button>
            );
          })}
        </div>
      )}
    </div>
  );
}


















//Createticketdrawer.tsx
import { useEffect, useMemo, useRef, useState } from 'react';
import { X, Loader2, ImagePlus, FileText } from 'lucide-react';
import type { Ticket, TicketAttachment, TicketPriority, TicketUser } from '../../types/tickets';
import { PRIORITY_META, initials, avatarAccent } from './ticketMeta';
import TicketSelect from './TicketSelect';
import styles from './CreateTicketDrawer.module.scss';

/** What the drawer hands back on submit — no id/status, the board adds those. */
export interface TicketSubmitPayload {
  title: string;
  description?: string;
  priority: TicketPriority;
  labels?: string[];
  assignee_id?: string | null;
  attachments?: TicketAttachment[];
}

interface CreateTicketDrawerProps {
  mode: 'create' | 'edit';
  initialTicket?: Ticket;
  members?: TicketUser[];
  submitting?: boolean;
  onClose: () => void;
  /** Return a Promise to keep the modal open on failure and close it on success. */
  onSubmit: (payload: TicketSubmitPayload) => Promise<unknown> | void;
}

const MAX_ATTACHMENT_BYTES = 5 * 1024 * 1024; // 5MB per image, static/mock build

const readAsDataUrl = (file: File) =>
  new Promise<string>((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => resolve(reader.result as string);
    reader.onerror = () => reject(reader.error);
    reader.readAsDataURL(file);
  });

const formatSize = (bytes: number) =>
  bytes < 1024 * 1024 ? `${Math.round(bytes / 1024)} KB` : `${(bytes / (1024 * 1024)).toFixed(1)} MB`;

// Centered modal, same shell as the rest of the app's dialogs (see
// .modal-overlay / .modal in this file's SCSS module). Rendered inline —
// no portal.
export default function CreateTicketDrawer({
  mode,
  initialTicket,
  members = [],
  submitting = false,
  onClose,
  onSubmit,
}: CreateTicketDrawerProps) {
  const [title, setTitle] = useState(initialTicket?.title ?? '');
  const [description, setDescription] = useState(initialTicket?.description ?? '');
  const [priority, setPriority] = useState<TicketPriority>(initialTicket?.priority ?? 'medium');
  const [labels, setLabels] = useState<string[]>(initialTicket?.labels ?? []);
  const [labelDraft, setLabelDraft] = useState('');
  const [assigneeId, setAssigneeId] = useState<string>(initialTicket?.assignee?.id ?? '');
  const [attachments, setAttachments] = useState<TicketAttachment[]>(initialTicket?.attachments ?? []);
  const [attachError, setAttachError] = useState('');
  const [touched, setTouched] = useState(false);

  const firstFieldRef = useRef<HTMLInputElement | null>(null);
  const fileInputRef = useRef<HTMLInputElement | null>(null);

  useEffect(() => {
    firstFieldRef.current?.focus();
  }, []);

  useEffect(() => {
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 'Escape' && !submitting) onClose();
    };
    document.addEventListener('keydown', onKey);
    return () => document.removeEventListener('keydown', onKey);
  }, [onClose, submitting]);

  const titleError = touched && !title.trim() ? 'Title is required' : '';
  const valid = title.trim().length > 0;

  const addLabel = () => {
    const v = labelDraft.trim();
    if (!v) return;
    if (!labels.includes(v)) setLabels((prev) => [...prev, v]);
    setLabelDraft('');
  };

  const handleFiles = async (fileList: FileList | null) => {
    if (!fileList || fileList.length === 0) return;
    setAttachError('');
    const files = Array.from(fileList);
    const oversized = files.find((f) => f.size > MAX_ATTACHMENT_BYTES);
    if (oversized) {
      setAttachError(`"${oversized.name}" is over 5MB — pick a smaller image.`);
      return;
    }
    const nonImage = files.find((f) => !f.type.startsWith('image/'));
    if (nonImage) {
      setAttachError('Only image files can be attached.');
      return;
    }
    const next: TicketAttachment[] = await Promise.all(
      files.map(async (f) => ({
        id: `a${Date.now()}${Math.random().toString(36).slice(2, 7)}`,
        name: f.name,
        url: await readAsDataUrl(f),
        size: f.size,
      }))
    );
    setAttachments((prev) => [...prev, ...next]);
  };

  const submit = () => {
    setTouched(true);
    if (!valid) return;
    onSubmit({
      title: title.trim(),
      description: description.trim() || undefined,
      priority,
      labels: labels.length ? labels : undefined,
      assignee_id: assigneeId || null,
      attachments: attachments.length ? attachments : undefined,
    });
  };

  const heading = mode === 'edit' ? 'Edit requirement' : 'New requirement';
  const cta = mode === 'edit' ? 'Save changes' : 'Create requirement';

  const priorityOptions = useMemo(
    () =>
      (Object.keys(PRIORITY_META) as TicketPriority[]).map((p) => ({
        value: p,
        label: PRIORITY_META[p].label,
        accent: PRIORITY_META[p].accent,
      })),
    []
  );

  const assigneeOptions = useMemo(
    () => [
      { value: '', label: 'Unassigned' },
      ...members.map((m) => ({
        value: m.id,
        label: m.name,
        accent: avatarAccent(m),
        glyph: initials(m),
      })),
    ],
    [members]
  );

  return (
    <div className={styles['modal-overlay']} onClick={() => !submitting && onClose()}>
      <div
        className={styles['modal']}
        role="dialog"
        aria-modal="true"
        aria-label={heading}
        onClick={(e) => e.stopPropagation()}
      >
        <header className={styles['modal-hdr']}>
          <div className={styles['modal-header-text']}>
            <span className={styles['modal-eyebrow']}>{mode === 'edit' ? 'Editing' : 'New'}</span>
            <span className={styles['modal-title']}>{heading}</span>
          </div>
          <button
            className={styles['modal-close']}
            onClick={onClose}
            disabled={submitting}
            aria-label="Close"
          >
            <X size={16} />
          </button>
        </header>

        <div className={styles['modal-body']}>
          {/* ---- main column: title, description, attachments ---- */}
          <div className={styles['modal-main']}>
            <label className={styles['field']}>
              <span className={styles['field-label']}>
                Title <span className={styles['field-req']}>*</span>
              </span>
              <input
                ref={firstFieldRef}
                className={`${styles['field-input']} ${styles['field-input--lg']} ${titleError ? styles['field-input--error'] : ''}`}
                value={title}
                onChange={(e) => setTitle(e.target.value)}
                onBlur={() => setTouched(true)}
                placeholder="Short summary of the requirement"
              />
              {titleError && <span className={styles['field-error']}>{titleError}</span>}
            </label>

            <label className={styles['field']}>
              <span className={styles['field-label']}>Description</span>
              <textarea
                className={styles['field-textarea']}
                value={description}
                onChange={(e) => setDescription(e.target.value)}
                rows={7}
                placeholder="Context, acceptance criteria, links…"
              />
            </label>

            <div className={styles['field']}>
              <span className={styles['field-label']}>Attachments</span>
              <input
                ref={fileInputRef}
                type="file"
                accept="image/*"
                multiple
                hidden
                onChange={(e) => {
                  handleFiles(e.target.files);
                  e.target.value = '';
                }}
              />
              <div className={styles['attach-zone']}>
                {attachments.map((a) => (
                  <div key={a.id} className={styles['attach-thumb']}>
                    <img src={a.url} alt={a.name} />
                    <button
                      type="button"
                      className={styles['attach-remove']}
                      onClick={() => setAttachments((prev) => prev.filter((x) => x.id !== a.id))}
                      aria-label={`Remove ${a.name}`}
                    >
                      <X size={11} />
                    </button>
                    <span className={styles['attach-meta']}>{formatSize(a.size)}</span>
                  </div>
                ))}
                <button
                  type="button"
                  className={styles['attach-add']}
                  onClick={() => fileInputRef.current?.click()}
                >
                  <ImagePlus size={18} />
                  Add image
                </button>
              </div>
              {attachError && <span className={styles['field-error']}>{attachError}</span>}
            </div>
          </div>

          {/* ---- meta column: priority, assignee, labels ---- */}
          <div className={styles['modal-rail']}>
            <div className={styles['field']}>
              <span className={styles['field-label']}>Priority</span>
              <TicketSelect
                value={priority}
                options={priorityOptions}
                onChange={(v) => setPriority(v as TicketPriority)}
                aria-label="Priority"
              />
            </div>

            <div className={styles['field']}>
              <span className={styles['field-label']}>Assignee</span>
              <TicketSelect
                value={assigneeId}
                options={assigneeOptions}
                placeholder="Unassigned"
                onChange={setAssigneeId}
                disabled={members.length === 0}
                aria-label="Assignee"
              />
            </div>

            <div className={styles['field']}>
              <span className={styles['field-label']}>Labels</span>
              <div className={styles['chip-input']}>
                {labels.map((l) => (
                  <span key={l} className={styles['chip']}>
                    {l}
                    <button
                      type="button"
                      onClick={() => setLabels((prev) => prev.filter((x) => x !== l))}
                      aria-label={`Remove ${l}`}
                    >
                      <X size={12} />
                    </button>
                  </span>
                ))}
                <input
                  className={styles['chip-field']}
                  value={labelDraft}
                  onChange={(e) => setLabelDraft(e.target.value)}
                  onKeyDown={(e) => {
                    if (e.key === 'Enter' || e.key === ',') {
                      e.preventDefault();
                      addLabel();
                    } else if (e.key === 'Backspace' && !labelDraft && labels.length) {
                      setLabels((prev) => prev.slice(0, -1));
                    }
                  }}
                  placeholder={labels.length ? 'Add another…' : 'Type, press Enter'}
                />
              </div>
            </div>

            {mode === 'edit' && initialTicket && (
              <div className={styles['field-note']}>
                <FileText size={12} />
                {initialTicket.key} · created{' '}
                {new Date(initialTicket.created_at).toLocaleDateString()}
              </div>
            )}
          </div>
        </div>

        <footer className={styles['modal-foot']}>
          <button className={styles['modal-cancel']} onClick={onClose} disabled={submitting}>
            Cancel
          </button>
          <button className={styles['modal-submit']} onClick={submit} disabled={submitting || !valid}>
            {submitting ? (
              <>
                <Loader2 size={15} className={styles['spin']} />
                Saving…
              </>
            ) : (
              cta
            )}
          </button>
        </footer>
      </div>
    </div>
  );
}




















//Createticketdrawwer.module.scss
@use '../../styles/_variables' as *;

// Centered modal — same shell convention used across the app (see
// Datasets.module.scss's __modal-overlay/__modal, and TicketBoard's
// .modal-overlay/.modal). Rendered inline, no portal: a full-viewport
// fixed overlay that flex-centers its panel.

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

@keyframes ticket-fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}
@keyframes ticket-modal-in {
  from { opacity: 0; transform: translateY(8px) scale(0.98); }
  to { opacity: 1; transform: none; }
}
@keyframes ticket-spin {
  to { transform: rotate(360deg); }
}

.modal-overlay {
  position: fixed;
  inset: 0;
  z-index: 200;
  background: rgba(20, 22, 27, 0.45);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
  animation: ticket-fade-in 0.15s ease;
}
.modal {
  width: min(780px, 100%);
  max-height: 88vh;
  display: flex;
  flex-direction: column;
  background: $card;
  border: 1px solid $line;
  border-radius: 18px;
  box-shadow: 0 24px 60px -20px rgba(20, 22, 27, 0.4);
  overflow: hidden;
  animation: ticket-modal-in 0.18s cubic-bezier(0.22, 1, 0.36, 1);
  // Own base size, a touch larger than the page base at very wide
  // viewports — a focused modal reads better slightly bigger than the
  // dense board sitting behind it.
  font-size: 0.8125rem;
  @media (min-width: 1800px) {
    font-size: 1.0625rem;
  }
}

// ---- header -----------------------------------------------------------
.modal-hdr {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.1em 1.25em;
  border-bottom: 1px solid $line;
}
.modal-header-text {
  display: flex;
  flex-direction: column;
  gap: 0.15em;
}
.modal-eyebrow {
  font-family: $mono;
  font-size: 0.68em;
  font-weight: 700;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  color: $signal;
}
.modal-title {
  font-family: $display;
  font-size: 1.2em;
  font-weight: 700;
  color: $ink;
}
.modal-close {
  flex-shrink: 0;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 8px;
  border: 1px solid $line;
  background: $paper;
  color: $ink-2;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease;
  &:hover:not(:disabled) { border-color: $ink-3; color: $ink; }
  &:disabled { opacity: 0.4; cursor: not-allowed; }
}

// ---- two-column body ----------------------------------------------------
.modal-body {
  flex: 1;
  min-height: 0;
  display: flex;
  overflow: hidden;
}
.modal-main {
  flex: 1;
  min-width: 0;
  overflow-y: auto;
  padding: 1.25em;
  display: flex;
  flex-direction: column;
  gap: 1.1em;
}
.modal-rail {
  flex: none;
  width: 260px;
  border-left: 1px solid $line;
  background: $paper;
  overflow-y: auto;
  padding: 1.25em;
  display: flex;
  flex-direction: column;
  gap: 1.1em;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 0.4em;
}
.field-label {
  font-size: 0.78em;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  color: $ink-2;
}
.field-req {
  color: $danger;
}
.field-input,
.field-textarea {
  width: 100%;
  border: 1px solid $line;
  border-radius: 8px;
  background: $card;
  color: $ink;
  font-size: 0.92em;
  font-family: inherit;
  padding: 0.65em 0.7em;
  outline: 0;
  transition: border-color 0.15s, box-shadow 0.15s;
  &:focus {
    border-color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
  &::placeholder {
    color: $ink-3;
  }
}
.field-input--lg {
  font-size: 1.15em;
  font-weight: 600;
  padding: 0.6em 0.7em;
}
.field-textarea {
  resize: vertical;
  line-height: 1.55;
  flex: 1;
}
.field-input--error {
  border-color: $danger;
  &:focus {
    box-shadow: 0 0 0 3px $danger-wash;
  }
}
.field-error {
  font-size: 0.78em;
  color: $danger;
}
.field-note {
  margin-top: auto;
  display: flex;
  align-items: center;
  gap: 0.4em;
  font-size: 0.76em;
  color: $ink-3;
  padding-top: 0.8em;
  border-top: 1px dashed $line;
}

// ---- attachments ----------------------------------------------------------
.attach-zone {
  display: flex;
  flex-wrap: wrap;
  gap: 0.6em;
}
.attach-thumb {
  position: relative;
  width: 84px;
  height: 84px;
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid $line;
  background: $paper;

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
  }
}
.attach-remove {
  position: absolute;
  top: 4px;
  right: 4px;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  border: 0;
  background: rgba(17, 24, 39, 0.65);
  color: #fff;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  &:hover {
    background: $danger;
  }
}
.attach-meta {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  padding: 0.15em 0.35em;
  font-size: 0.6em;
  color: #fff;
  background: rgba(17, 24, 39, 0.55);
  text-align: center;
}
.attach-add {
  width: 84px;
  height: 84px;
  border-radius: 8px;
  border: 1.5px dashed $line;
  background: $paper;
  color: $ink-3;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.3em;
  font-size: 0.72em;
  font-weight: 600;
  cursor: pointer;
  transition: border-color 0.15s, color 0.15s, background 0.15s;
  &:hover {
    border-color: $signal;
    color: $signal;
    background: $wash;
  }
}

// ---- chip input -----------------------------------------------------------
.chip-input {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 0.4em;
  border: 1px solid $line;
  border-radius: 8px;
  background: $card;
  padding: 0.45em 0.5em;
  min-height: 2.6em;
  &:focus-within {
    border-color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
}
.chip {
  display: inline-flex;
  align-items: center;
  gap: 0.3em;
  font-size: 0.8em;
  padding: 0.25em 0.3em 0.25em 0.55em;
  border-radius: 6px;
  background: $ink-wash;
  color: $ink-2;
  button {
    border: 0;
    background: transparent;
    color: $ink-3;
    display: inline-flex;
    cursor: pointer;
    padding: 0.1em;
    border-radius: 4px;
    &:hover {
      color: $danger;
      background: $danger-wash;
    }
  }
}
.chip-field {
  flex: 1;
  min-width: 6em;
  border: 0;
  outline: 0;
  background: transparent;
  color: $ink;
  font-size: 0.9em;
  &::placeholder {
    color: $ink-3;
  }
}

// ---- footer -----------------------------------------------------------
.modal-foot {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 0.6em;
  padding: 1em 1.25em;
  border-top: 1px solid $line;
}
.modal-cancel {
  padding: 9px 16px;
  border: 1px solid $line;
  border-radius: 10px;
  background: $card;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.9em;
  font-weight: 650;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;
  &:hover:not(:disabled) { border-color: $ink-3; color: $ink; background: $paper; }
  &:disabled { opacity: 0.5; cursor: not-allowed; }
}
.modal-submit {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 9px 18px;
  border: 1px solid transparent;
  border-radius: 10px;
  background: $signal;
  color: #fff;
  font-family: $sans;
  font-size: 0.9em;
  font-weight: 650;
  cursor: pointer;
  transition: background 0.15s ease;
  &:hover:not(:disabled) { background: $signal-2; }
  &:disabled { opacity: 0.6; cursor: not-allowed; }
}
.spin {
  animation: ticket-spin 0.8s linear infinite;
}

@media (max-width: 768px) {
  .modal-overlay { padding: 12px; }
  .modal { width: 100%; max-height: 94vh; border-radius: 14px; }
  .modal-body { flex-direction: column; overflow-y: auto; }
  .modal-rail { width: auto; border-left: 0; border-top: 1px solid $line; }
}

@media (prefers-reduced-motion: reduce) {
  .modal-overlay,
  .modal,
  .spin {
    animation: none;
  }
}

// ===========================================================================
// Custom select — replaces the native <select> with themed dropdown chrome
// (used for Priority / Assignee in the rail).
// ===========================================================================
.select {
  position: relative;
}
.select-trigger {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.5em;
  border: 1px solid $line;
  border-radius: 8px;
  background: $card;
  color: $ink;
  font-size: 0.92em;
  font-family: inherit;
  padding: 0.62em 0.7em;
  cursor: pointer;
  transition: border-color 0.15s, box-shadow 0.15s, background 0.15s;

  &:hover:not(:disabled) {
    border-color: $ink-3;
  }
  &:disabled {
    opacity: 0.55;
    cursor: not-allowed;
  }
}
.select-trigger--open {
  border-color: $signal;
  box-shadow: 0 0 0 3px $wash;
}
.select-value {
  display: flex;
  align-items: center;
  gap: 0.5em;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-weight: 500;
}
.select-placeholder {
  color: $ink-3;
  font-weight: 400;
}
.select-caret {
  flex-shrink: 0;
  color: $ink-3;
  transition: transform 0.15s;
  .select-trigger--open & {
    transform: rotate(180deg);
  }
}

.select-menu {
  position: absolute;
  top: calc(100% + 6px);
  left: 0;
  right: 0;
  z-index: 20;
  max-height: 15em;
  overflow-y: auto;
  background: $card;
  border: 1px solid $line;
  border-radius: 10px;
  box-shadow: 0 14px 32px -12px rgba(20, 22, 27, 0.35);
  padding: 0.35em;
  display: flex;
  flex-direction: column;
  gap: 0.1em;
  animation: ticket-modal-in 0.12s ease both;
}
.select-empty {
  padding: 0.6em 0.7em;
  font-size: 0.86em;
  color: $ink-3;
  text-align: center;
}
.select-option {
  display: flex;
  align-items: center;
  gap: 0.55em;
  width: 100%;
  border: 0;
  background: transparent;
  color: $ink;
  font-size: 0.9em;
  font-family: inherit;
  text-align: left;
  padding: 0.55em 0.6em;
  border-radius: 7px;
  cursor: pointer;
  transition: background 0.12s;

  &:hover {
    background: $paper;
  }
}
.select-option--active {
  background: $wash;
  font-weight: 600;
  color: $signal;
}
.select-option--highlight {
  background: $paper;
}
.select-option--active.select-option--highlight {
  background: $wash;
  box-shadow: inset 0 0 0 1px rgba(43, 43, 245, 0.25);
}
.select-option-label {
  flex: 1;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.select-check {
  flex-shrink: 0;
  color: $signal;
}
.select-dot {
  flex-shrink: 0;
  width: 1.5em;
  height: 1.5em;
  border-radius: 50%;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 0.62em;
  font-weight: 700;
  color: #fff;
  // used both as a plain priority-color dot (no glyph) and as a tiny
  // avatar-style badge (with initials as glyph) for the assignee list
}
