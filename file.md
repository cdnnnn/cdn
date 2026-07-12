// ═══════════════════════════════════════════════
// PromptTemplates.module.scss
// Content Analytics · Prompt Template management
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

// ── Page shell ──────────────────────────────────
.page {
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

// ── Page header ─────────────────────────────────
.ph {
  padding: 14px 24px 12px;
  background: var(--bg1);
  border-bottom: 1px solid var(--bdr);
  flex-shrink: 0;
}

.phRow {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 16px;
}

.phTitle {
  font-size: 18px;
  font-weight: 600;
  color: var(--t0);
  letter-spacing: -0.3px;
  font-family: var(--font-display);
}

.phSub {
  font-size: 11px;
  color: var(--t2);
  margin-top: 3px;
  @include m.mono;
}

.phActs {
  display: flex;
  align-items: center;
  gap: 10px;
  flex-shrink: 0;
}

// ── Scrollable body ──────────────────────────────
.body {
  flex: 1;
  overflow-y: auto;
  @include m.scrollbar;
}

.viewBody {
  padding: 20px 24px 40px;
  display: flex;
  flex-direction: column;
  gap: 14px;
}

// ── Banners ───────────────────────────────────────
.successBanner {
  padding: 9px 14px;
  border-radius: var(--r);
  background: var(--green-dim);
  border: 1px solid var(--green-bdr);
  color: var(--green);
  font-size: 12px;
  font-weight: 500;
}

.errorBanner {
  padding: 9px 14px;
  border-radius: var(--r);
  background: var(--red-dim);
  border: 1px solid var(--red-bdr);
  color: var(--red);
  font-size: 12px;
  font-weight: 500;
}

// ── Table ────────────────────────────────────────
.tableWrap {
  overflow-x: auto;
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  background: var(--bg1);
  @include m.scrollbar;
}

.table {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px;

  th {
    padding: 10px 16px;
    text-align: left;
    font-size: 10px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    color: var(--t2);
    background: var(--bg0);
    border-bottom: 1px solid var(--bdr);
    white-space: nowrap;
    @include m.mono;
  }

  td {
    padding: 11px 16px;
    border-bottom: 1px solid var(--bdr);
    color: var(--t0);
    vertical-align: middle;

    &:last-child {
      text-align: right;
    }
  }

  tr:last-child td {
    border-bottom: none;
  }

  tr:hover td {
    background: var(--bg2);
  }
}

.nameCell {
  font-weight: 600;
  color: var(--t0);
}

.descCell {
  max-width: 320px;
  @include m.truncate;
}

.rowActs {
  display: inline-flex;
  gap: 6px;
}

.emptyRow {
  text-align: center !important;
  color: var(--t2);
  padding: 32px 16px !important;
  font-size: 12px;
}

.muted {
  color: var(--t2) !important;
}

.mono {
  @include m.mono;
  color: var(--t1) !important;
}

// ── Buttons ───────────────────────────────────────
.btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 6px 13px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.12s;
  white-space: nowrap;

  &:hover:not(:disabled) {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.5;
    cursor: default;
  }
}

.btnSm {
  padding: 4px 10px;
  font-size: 11px;
}

.btnPrimary {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 7px 14px;
  border-radius: var(--r);
  border: 1px solid var(--blue-bdr);
  background: var(--blue);
  color: #fff;
  font-family: var(--font-ui);
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.12s;
  white-space: nowrap;

  &:hover:not(:disabled) {
    filter: brightness(1.08);
  }

  &:disabled {
    opacity: 0.6;
    cursor: default;
  }
}

.btnDanger {
  color: var(--red);
  border-color: var(--red-bdr);

  &:hover:not(:disabled) {
    background: var(--red-dim);
    color: var(--red);
    border-color: var(--red-bdr);
  }
}

.btnDangerSolid {
  background: var(--red);
  border-color: var(--red-bdr);

  &:hover:not(:disabled) {
    filter: brightness(1.08);
  }
}

// ── Spinner ───────────────────────────────────────
.spinnerWrap {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 12px;
  padding: 80px 0;
  color: var(--t2);
  font-size: 12px;
  @include m.mono;
}

.spinner {
  width: 22px;
  height: 22px;
  border: 2px solid var(--bdr2);
  border-top-color: var(--blue);
  border-radius: 50%;
  animation: ptSpin 0.7s linear infinite;
}

// ── Modal ─────────────────────────────────────────
.overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.55);
  backdrop-filter: blur(2px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 24px;
  animation: ptFadeIn 0.15s ease-out;

  // Give a little more breathing room on short viewports
  @media (max-height: 700px) {
    padding: 12px;
    align-items: flex-start;
    overflow-y: auto;
    @include m.scrollbar;
  }
}

.modal {
  width: 100%;
  max-width: 560px;
  // Cap the modal at viewport height minus overlay padding
  max-height: calc(100vh - 48px);
  display: flex;
  flex-direction: column;
  // Without this the flex children can push past max-height
  min-height: 0;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--rxl);
  box-shadow: var(--shadow);
  animation: ptScaleIn 0.16s cubic-bezier(0.4, 0, 0.2, 1);

  // The <form> inside must also be a flex column so modalBody can flex-grow
  form {
    display: flex;
    flex-direction: column;
    flex: 1;
    min-height: 0;
  }
}

.modalSm {
  max-width: 380px;
}

.modalHead {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 18px;
  border-bottom: 1px solid var(--bdr);
  flex-shrink: 0;
}

.modalTitle {
  font-size: 14px;
  font-weight: 600;
  color: var(--t0);
  font-family: var(--font-display);
}

.closeBtn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border-radius: 7px;
  border: none;
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  transition: background 0.12s, color 0.12s;

  &:hover {
    background: var(--bg3);
    color: var(--t0);
  }
}

.modalBody {
  padding: 16px 18px;
  flex: 1;        // fill all space between header and footer
  min-height: 0;  // allows shrinking below content size so overflow-y kicks in
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 14px;
  @include m.scrollbar;
}

.modalFoot {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 8px;
  padding: 14px 18px;
  border-top: 1px solid var(--bdr);
  flex-shrink: 0;
}

.confirmText {
  font-size: 13px;
  color: var(--t1);
  line-height: 1.5;

  strong {
    color: var(--t0);
  }
}

// ── Form ──────────────────────────────────────────
.formGroup {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.label {
  font-size: 11px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  @include m.mono;
}

.required {
  color: var(--red);
}

.input,
.textarea {
  width: 100%;
  background: var(--bg3);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 13px;
  padding: 8px 10px;
  outline: none;
  transition: border-color 0.12s, box-shadow 0.12s;

  &::placeholder {
    color: var(--t2);
  }

  &:focus {
    border-color: var(--blue-bdr);
    box-shadow: 0 0 0 2px var(--blue-dim);
  }
}

.textarea {
  resize: vertical;
  min-height: 64px;
  line-height: 1.5;
  @include m.mono;
  font-size: 12px;
}

// ── Animations ────────────────────────────────────
@keyframes ptFadeIn {
  from {
    opacity: 0;
  }

  to {
    opacity: 1;
  }
}

@keyframes ptScaleIn {
  from {
    opacity: 0;
    transform: translateY(6px) scale(0.98);
  }

  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

@keyframes ptSpin {
  to {
    transform: rotate(360deg);
  }
}
