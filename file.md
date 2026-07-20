// ═══════════════════════════════════════════════
// InferencePanel.module.scss
// ═══════════════════════════════════════════════
@use '../../styles/mixins' as m;

// ── Panel shell ──
.infpanel {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;

  // On very large screens, everything (text, padding, borders, icons)
  // was tuned for ~1440-1900px and starts to look small. `zoom` scales
  // the whole subtree proportionally and reflows layout correctly
  // (unlike transform: scale, which doesn't affect layout/hit-testing).
  @media (min-width: 1900px) {
    zoom: 1.25;
  }
}

// ── Inference Running label in step header ──
.inferenceRunningLabel {
  background: linear-gradient(90deg, var(--blue), #a78bfa, var(--amber));
  background-size: 200% auto;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  animation: labelShimmer 2.5s linear infinite;
  font-weight: 700;
}

@keyframes labelShimmer {
  0% {
    background-position: 0% center;
  }

  100% {
    background-position: 200% center;
  }
}

// ── Panel header ──
.infpanelHead {
  padding: 13px 18px 10px;
  border-bottom: none;
  background: var(--bg1);
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  position: relative;

  &::after {
    content: '';
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 1px;
    background: linear-gradient(90deg,
        rgba(139, 92, 246, 0.0) 0%,
        rgba(139, 92, 246, 0.6) 20%,
        rgba(56, 196, 186, 0.7) 50%,
        rgba(240, 160, 48, 0.6) 80%,
        rgba(240, 160, 48, 0.0) 100%);
    pointer-events: none;
  }
}

.slbl {
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  @include m.mono;
  margin-bottom: 2px;
}

.selSummary {
  font-size: 11px;
  color: var(--t1);
}

.headActions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.closeStepBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.14s;

  svg {
    width: 13px;
    height: 13px;
  }

  &:hover {
    background: rgba(239, 68, 68, 0.08);
    border-color: rgba(239, 68, 68, 0.3);
    color: #ef4444;
  }
}

// ══════════════════════════════════════
// RUN INFERENCE BUTTON
// ══════════════════════════════════════

.runBtn {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 6px 13px 6px 8px;
  border-radius: 8px;
  border: 1px solid transparent;
  background: var(--blue);
  color: #ffffff;
  font-family: var(--font-ui);
  cursor: pointer;
  transition: background 0.15s, box-shadow 0.15s, opacity 0.15s, border-color 0.15s;
  white-space: nowrap;
  user-select: none;
  flex-shrink: 0;

  &:hover:not(:disabled) {
    background: #6bb3f5;
  }

  &:active:not(:disabled) {
    background: #4a90d9;
  }
}

// Icon chip inside the button
.runBtnIconWrap {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  border-radius: 6px;
  background: rgba(255, 255, 255, 0.18);
  flex-shrink: 0;
  transition: background 0.14s;

  svg {
    display: block;
  }

  .runBtn:hover:not(:disabled) & {
    background: rgba(255, 255, 255, 0.25);
  }
}

// Two-line text stack
.runBtnText {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  line-height: 1.2;
  gap: 2px;
}

.runBtnTitle {
  font-size: 11px;
  font-weight: 700;
  letter-spacing: -0.1px;
}

.runBtnSub {
  font-size: 8.5px;
  font-weight: 500;
  opacity: 0.82;
  font-family: var(--font-mono);
}

// Ready state — glowing outline pulse to draw the eye
.runBtnReady {
  box-shadow:
    0 0 0 1px rgba(91, 164, 239, 0.7),
    0 2px 12px rgba(91, 164, 239, 0.35);
  animation: runReadyPulse 2.4s ease-in-out infinite;
}

@keyframes runReadyPulse {

  0%,
  100% {
    box-shadow:
      0 0 0 1px rgba(91, 164, 239, 0.7),
      0 2px 12px rgba(91, 164, 239, 0.30);
  }

  50% {
    box-shadow:
      0 0 0 2px rgba(91, 164, 239, 0.55),
      0 4px 20px rgba(91, 164, 239, 0.50);
  }
}

// Disabled state — muted, clearly not actionable, sub-label explains why
.runBtnDisabled {
  opacity: 0.48;
  cursor: not-allowed;
  pointer-events: none;
  box-shadow: none;
}

// Large screen tweaks
@media (min-width: 1920px) {
  .runBtnTitle {
    font-size: 12px;
  }

  .runBtnSub {
    font-size: 9px;
  }

  .runBtnIconWrap {
    width: 26px;
    height: 26px;
  }

  .runBtn {
    padding: 7px 14px 7px 9px;
    gap: 9px;
    border-radius: 9px;
  }
}

// ── Main body ──
.infpanelBody {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  padding: 14px 18px 0;
}

// Configuration mode — body itself scrolls, batch section is absent
.infpanelBodyConfig {
  overflow-y: auto;
  @include m.scrollbar;
}

// Running mode — no scroll on body; each batch column scrolls independently
.infpanelBodyRunning {
  overflow: hidden;
}

// ── Selection banner ──
.selBanner {
  background: var(--bg2);
  border: 1px solid var(--blue-bdr);
  border-radius: var(--rl);
  padding: 9px 13px;
  margin-bottom: 13px;
  display: flex;
  align-items: center;
  gap: 9px;
}

.selCt {
  font-size: 11px;
  color: var(--blue);
  font-weight: 600;
  @include m.mono;
  flex-shrink: 0;
}

.selNm {
  font-size: 10px;
  color: var(--t2);
  @include m.mono;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  flex: 1;
}

// ── Collapsible settings wrap ──
.infSettingsWrap {
  padding-bottom: 16px;
  flex-shrink: 0;
}

.settingsGrid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 11px;
  margin-bottom: 12px;
}

// 2 fields per row inside the merged Generate content card
.fieldsGrid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 4px 20px;
  align-items: start;
}

.fieldCell {
  min-width: 0;
}

// Model dropdown spans both columns
.modelFieldTop {
  width: 260px;
  max-width: 100%;
  padding-bottom: 14px;
  margin-bottom: 14px;
  border-bottom: 1px solid var(--bdr);
}

.promptGrid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 11px;
}

.promptStack {
  display: flex;
  flex-direction: column;
  gap: 11px;
}

.noPromptHint {
  font-size: 11px;
  color: var(--t2);
  font-family: var(--font-mono);
  text-align: center;
  padding: 10px 0 2px;
}

// ── Card ──
.card {
  background: var(--bg1);
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  padding: 14px 16px;
}

.cardT {
  font-size: 9px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color: var(--t2);
  margin-bottom: 12px;
  @include m.mono;
}

.promptMode {
  font-size: 10px;
  margin-left: 8px;
  font-weight: 400;
  text-transform: none;
  letter-spacing: 0;
  @include m.mono;
}

// ── Form ──
.fg {
  margin-bottom: 11px;
}

// Indents a prompt block under the checkbox it belongs to (same 20px
// indent convention as .nestedField, used for the timestamp interval).
.promptInline {
  padding-left: 20px;
  margin-bottom: 4px;
  cursor: default;
}

.mb0 {
  margin-bottom: 0 !important;
}

.mb8 {
  margin-bottom: 8px;
}

.mb12 {
  margin-bottom: 12px !important;
}

.flRow {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 4px;
}

.fl {
  display: block;
  font-size: 11px;
  color: var(--t1);
  margin-bottom: 0;
  font-weight: 500;
}

.fc {
  width: 100%;
  padding: 7px 11px;
  background: var(--bg0);
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  color: var(--t0);
  font-family: var(--font-ui);
  font-size: 11px;
  transition: border-color 0.12s;
  outline: none;
  appearance: none;

  &:focus {
    border-color: var(--blue);
    box-shadow: 0 0 0 3px rgba(91, 164, 239, 0.07);
  }

  &::placeholder {
    color: var(--t2);
  }
}

select.fc {
  cursor: pointer;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='7'%3E%3Cpath d='M1 1l4 4 4-4' stroke='%234e5668' stroke-width='1.5' fill='none' stroke-linecap='round'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 10px center;
  padding-right: 26px;
}

textarea.fc {
  resize: vertical;
  min-height: 56px;
  line-height: 1.6;
}

.optTag {
  font-size: 10px;
  color: var(--t2);
  margin-left: 4px;
  font-weight: 400;
  @include m.mono;
}

// ── Checkboxes ──
.cr {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 3px 0;
  cursor: pointer;
  user-select: none;

  label {
    font-size: 11px;
    color: var(--t1);
    cursor: pointer;
  }
}

.crDisabled {
  cursor: default;
  opacity: 0.45;

  label {
    cursor: default;
  }

  .cb {
    cursor: default;
  }
}

.cb {
  width: 14px;
  height: 14px;
  border: 1.5px solid var(--bdr2);
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: all 0.12s;
}

.ck {
  .cb {
    background: var(--blue);
    border-color: var(--blue);

    &::after {
      content: '';
      width: 7px;
      height: 4px;
      border-left: 1.5px solid #0b0d10;
      border-bottom: 1.5px solid #0b0d10;
      transform: rotate(-45deg) translate(0, -1px);
      display: block;
    }
  }

  label {
    color: var(--t0);
  }
}

// Nested option, e.g. "Keyword Insights" under Keywords — indented, only
// rendered while its parent checkbox is on.
.crNested {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 3px 0 3px 20px;
  cursor: pointer;
  user-select: none;
  position: relative;

  &::before {
    content: '';
    position: absolute;
    left: 6px;
    top: 0;
    bottom: 50%;
    width: 8px;
    border-left: 1.5px solid var(--bdr2);
    border-bottom: 1.5px solid var(--bdr2);
    border-radius: 0 0 0 4px;
  }

  label {
    font-size: 10.5px;
    color: var(--t1);
    cursor: pointer;
  }

  &.ck label {
    color: var(--t0);
  }
}

// Nested field, e.g. the Timestamp Summary Interval dropdown — indented to
// visually sit under its parent checkbox.
.nestedField {
  padding-left: 20px;
  margin-bottom: 8px;

  label {
    display: block;
    font-size: 9.5px;
    color: var(--t2);
    margin-bottom: 4px;
  }

  select {
    max-width: 200px;
  }
}

// ── Range sliders ──
.rangeInput {
  -webkit-appearance: none;
  width: 100%;
  height: 3px;
  background: var(--bg4);
  border-radius: 99px;
  outline: none;
  cursor: pointer;
  display: block;
  margin-top: 4px;

  &::-webkit-slider-thumb {
    -webkit-appearance: none;
    width: 14px;
    height: 14px;
    background: var(--blue);
    border-radius: 50%;
    border: 2px solid var(--bg0);
    cursor: pointer;
    box-shadow: 0 0 0 2px rgba(91, 164, 239, 0.25);
  }
}

.slim {
  display: flex;
  justify-content: space-between;
  font-size: 10px;
  color: var(--t2);
  margin-top: 3px;
  @include m.mono;
}

.sv {
  @include m.mono;
  font-size: 11px;
  color: var(--blue);
}

.qTypeLabel {
  font-size: 11px;
  color: var(--t1);
  font-weight: 500;
  margin-bottom: 5px;
}

.dimLabel {
  color: var(--t2) !important;
}

.feasibilityTag {
  font-size: 9px;
  background: var(--amber-dim);
  color: var(--amber);
  padding: 1px 5px;
  border-radius: 99px;
  margin-left: 3px;
}

// ── Divider ──
.divider {
  border: none;
  height: 1px;
  margin: 12px 0;
  background: linear-gradient(90deg,
      rgba(139, 92, 246, 0.0) 0%,
      rgba(139, 92, 246, 0.35) 25%,
      rgba(56, 196, 186, 0.4) 50%,
      rgba(240, 160, 48, 0.35) 75%,
      rgba(240, 160, 48, 0.0) 100%);
}

// ── Callout ──
.callout {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  padding: 9px 12px;
  border-radius: var(--r);
  font-size: 11px;
  margin-bottom: 10px;
}

.cInfo {
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
  color: var(--blue);
}

// ── Generic utility buttons (used for save/cancel etc.) ──
.btn {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 6px 13px;
  border-radius: var(--r);
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t1);
  font-family: var(--font-ui);
  font-size: 10px;
  cursor: pointer;
  transition: all 0.12s;
  white-space: nowrap;
  user-select: none;

  &:hover {
    background: var(--bg3);
    color: var(--t0);
    border-color: var(--bdr3);
  }
}

.btnP {
  background: var(--blue);
  color: #ffffff;
  border-color: var(--blue);
  font-weight: 600;

  &:hover {
    background: #a78bfa;
    border-color: #a78bfa;
    color: #ffffff;
  }

  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
    pointer-events: none;
  }
}

// ── Batch running pill in header ─────────────────
.batchRunPill {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 10px;
  color: var(--amber);
  background: var(--amber-dim);
  border: 1px solid var(--amber-bdr);
  padding: 1px 8px;
  border-radius: 99px;
  margin-left: 10px;
  font-family: var(--font-mono);
  vertical-align: middle;
}

.batchRunDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: var(--amber);
  animation: breathe 1.2s ease-in-out infinite;
}

@keyframes breathe {

  0%,
  100% {
    opacity: 1;
  }

  50% {
    opacity: 0.35;
  }
}

// ── Model loading indicator ───────────────────────
.loadingDot {
  font-size: 11px;
  color: var(--t2);
  font-family: var(--font-mono);
}

// ══════════════════════════════════════
// 3-COLUMN BATCH STATUS
// ══════════════════════════════════════
.batchSection {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  margin-top: 16px;
  animation: fadeIn 0.2s ease;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(4px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.batchSectionTitle {
  font-size: 10px;
  font-weight: 600;
  color: var(--t2);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  font-family: var(--font-mono);
  margin-bottom: 10px;
  display: flex;
  align-items: center;
  gap: 10px;
  flex-shrink: 0;
  flex-wrap: wrap;
}

.liveLabel {
  flex-shrink: 0;
}

// ── Live status pill ─────────────────────────────
.liveStatus {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 10px;
  font-weight: 500;
  color: var(--green);
  background: var(--green-dim);
  border: 1px solid var(--green-bdr);
  padding: 2px 9px 2px 7px;
  border-radius: 99px;
  font-family: var(--font-mono);
  text-transform: none;
  letter-spacing: 0;
  white-space: nowrap;
}

.liveDot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: var(--green);
  flex-shrink: 0;
  animation: livePulse 1.4s ease-in-out infinite;
}

@keyframes livePulse {

  0%,
  100% {
    opacity: 1;
    transform: scale(1);
  }

  50% {
    opacity: 0.4;
    transform: scale(0.75);
  }
}

.liveSep {
  color: var(--green);
  opacity: 0.4;
  font-size: 10px;
}

.liveCountdown {
  color: var(--green);
  font-weight: 700;
  font-variant-numeric: tabular-nums;
  min-width: 2ch;
  display: inline-block;
  text-align: right;
}

.liveFiles {
  color: var(--green);
  opacity: 0.85;
}

.batchColumns {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
  flex: 1;
  overflow: hidden;
  min-height: 0;
}

.batchCol {
  display: flex;
  flex-direction: column;
  gap: 6px;
  min-width: 0;
  min-height: 0;
  overflow: hidden;
}

// Column headers
.batchColHead {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 10px;
  font-weight: 700;
  padding: 6px 10px;
  border-radius: 8px;
  font-family: var(--font-mono);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  flex-shrink: 0;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }

  &.headQueued {
    background: rgba(240, 160, 48, 0.08);
    color: var(--amber);
    border: 1px solid var(--amber-bdr);

    svg {
      stroke: var(--amber);
    }
  }

  &.headRunning {
    background: var(--blue-dim);
    color: var(--blue);
    border: 1px solid var(--blue-bdr);

    svg {
      stroke: var(--blue);
    }
  }

  &.headCompleted {
    background: var(--green-dim);
    color: var(--green);
    border: 1px solid var(--green-bdr);

    svg {
      stroke: var(--green);
    }
  }
}

.batchColCount {
  margin-left: auto;
  font-size: 11px;
  font-weight: 700;
  min-width: 16px;
  text-align: center;
}

.batchColBody {
  display: flex;
  flex-direction: column;
  gap: 5px;
  flex: 1;
  overflow-y: auto;
  padding-bottom: 14px;
  @include m.scrollbar;
}

.batchEmpty {
  text-align: center;
  font-size: 11px;
  color: var(--t2);
  font-family: var(--font-mono);
  padding: 12px 0;
}

// ── Status card wrapper (handles enter/exit animation) ──
.statusCardWrap {
  position: relative;
  border-radius: 10px;
}

// ── Queued: slow breathing amber dashed-style border ──
.queuedWrap {
  padding: 2px;
  background: conic-gradient(from 0deg,
      rgba(240, 160, 48, 0.7) 0deg,
      rgba(240, 160, 48, 0.15) 60deg,
      rgba(240, 160, 48, 0.05) 120deg,
      rgba(240, 160, 48, 0.15) 180deg,
      rgba(240, 160, 48, 0.05) 240deg,
      rgba(240, 160, 48, 0.15) 300deg,
      rgba(240, 160, 48, 0.7) 360deg);
  animation: queuedBreath 2.8s ease-in-out infinite;
}

@keyframes queuedBreath {

  0%,
  100% {
    opacity: 0.5;
  }

  50% {
    opacity: 1;
  }
}

// ── Completed: static clean green gradient border, no animation ──
.completedWrap {
  padding: 2px;
  background: linear-gradient(135deg,
      rgba(74, 222, 128, 0.9) 0%,
      rgba(74, 222, 128, 0.3) 40%,
      rgba(74, 222, 128, 0.6) 70%,
      rgba(74, 222, 128, 0.9) 100%);
}

// ── Animated gradient border for running cards ──
.runningWrap {
  border-radius: 10px;
  padding: 2px;
  background: conic-gradient(from var(--border-angle, 0deg),
      transparent 0deg,
      transparent 30deg,
      #5ba4ef 50deg,
      #a78bfa 65deg,
      #f0a030 80deg,
      transparent 100deg,
      transparent 360deg);
  animation: borderBeamSpin 2.4s linear infinite;
}

@property --border-angle {
  syntax: '<angle>';
  initial-value: 0deg;
  inherits: false;
}

@keyframes borderBeamSpin {
  to {
    --border-angle: 360deg;
  }
}

// Status cards
.statusCard {
  display: flex;
  align-items: flex-start;
  gap: 7px;
  padding: 8px 10px;
  border-radius: 8px;
  border: 1px solid;
  min-width: 0;
  position: relative;

  &.queued {
    background: var(--bg1);
    border-color: transparent;
    border-radius: 8px;
  }

  &.running {
    background: var(--bg1);
    border-color: transparent;
    border-radius: 8px;
  }

  &.completed {
    background: var(--bg1);
    border-color: transparent;
    border-radius: 8px;
  }
}

.statusExt {
  width: 26px;
  height: 26px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 9px;
  font-weight: 800;
  font-family: var(--font-mono);
  flex-shrink: 0;

  &.vtt {
    background: var(--blue-dim);
    color: var(--blue);
  }

  &.srt {
    background: var(--green-dim);
    color: var(--green);
  }
}

// ── Completed check icon ──
.completedCheck {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  background: var(--green-dim);
  border: 1.5px solid var(--green-bdr);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  color: var(--green);

  svg {
    width: 13px;
    height: 13px;
  }
}

// ── Queued "waiting" meta line ──
.queuedMeta {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 9px;
  color: var(--amber);
  font-family: var(--font-mono);
  opacity: 0.8;
}

.queuedDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: var(--amber);
  flex-shrink: 0;
  animation: queuedDotPulse 1.6s ease-in-out infinite;
}

@keyframes queuedDotPulse {

  0%,
  100% {
    opacity: 0.3;
    transform: scale(0.8);
  }

  50% {
    opacity: 1;
    transform: scale(1.1);
  }
}

// ── Completed date meta ──
.completedMeta {
  font-size: 9px;
  color: var(--green);
  font-family: var(--font-mono);
  opacity: 0.75;
}

.statusInfo {
  flex: 1;
  min-width: 0;
}

.statusNameRow {
  display: flex;
  align-items: center;
  gap: 5px;
  min-width: 0;
  margin-bottom: 3px;
  overflow: hidden;
}

.statusName {
  font-size: 11px;
  font-weight: 500;
  color: var(--t0);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  flex: 1;
  min-width: 0;
}

.statusIdBadge {
  font-size: 9px;
  font-weight: 700;
  font-family: var(--font-mono);
  border-radius: 4px;
  padding: 1px 5px;
  flex-shrink: 0;
  white-space: nowrap;
  letter-spacing: 0.02em;
  color: var(--blue);
  background: var(--blue-dim);
  border: 1px solid var(--blue-bdr);
}

.statusIdBadge_queued {
  color: var(--amber);
  background: rgba(240, 160, 48, 0.12);
  border-color: rgba(240, 160, 48, 0.3);
}

.statusIdBadge_running {
  color: var(--blue);
  background: var(--blue-dim);
  border-color: var(--blue-bdr);
}

.statusIdBadge_completed {
  color: var(--green);
  background: var(--green-dim);
  border-color: var(--green-bdr);
}

.statusDate {
  font-size: 9px;
  color: var(--t2);
  font-family: var(--font-mono);
}

.statusProgress {
  display: flex;
  align-items: center;
  gap: 5px;
}

.statusBar {
  flex: 1;
  height: 3px;
  background: var(--bg4);
  border-radius: 99px;
  overflow: hidden;
}

.statusFill {
  height: 100%;
  background: var(--blue);
  border-radius: 99px;
  transition: width 0.4s ease;
}

.statusPct {
  font-size: 9px;
  color: var(--blue);
  font-family: var(--font-mono);
  flex-shrink: 0;
}

// ── Model row (label + refresh) ──────────────────
.modelLabelRow {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 4px;

  .fl {
    margin-bottom: 0;
  }
}

.refreshBtn {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 8px;
  border-radius: 99px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-family: var(--font-ui);
  font-size: 10px;
  cursor: pointer;
  transition: all 0.12s;
  flex-shrink: 0;

  svg {
    width: 10px;
    height: 10px;
    flex-shrink: 0;
    transition: transform 0.12s;
  }

  &:hover:not(:disabled) {
    background: var(--bg3);
    color: var(--t1);
    border-color: var(--bdr3);
  }

  &:disabled {
    opacity: 0.45;
    cursor: not-allowed;
  }
}

.spinning {
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }

  to {
    transform: rotate(360deg);
  }
}

// ── Model status messages ─────────────────────────
.modelWarn {
  display: flex;
  align-items: flex-start;
  gap: 6px;
  margin-top: 7px;
  padding: 7px 10px;
  border-radius: var(--r);
  background: var(--amber-dim);
  border: 1px solid var(--amber-bdr);
  color: var(--amber);
  font-size: 11px;
  line-height: 1.5;

  svg {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
    margin-top: 1px;
    stroke: var(--amber);
  }
}

.modelOk {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 6px;
  font-size: 10px;
  color: var(--t2);
  font-family: var(--font-mono);

  svg {
    width: 11px;
    height: 11px;
    flex-shrink: 0;
    stroke: var(--green);
  }
}

// ══════════════════════════════════════
// SUBMITTING OVERLAY
// ══════════════════════════════════════
.submittingOverlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(var(--bg0-rgb, 11, 13, 16), 0.82);
  backdrop-filter: blur(4px);
  z-index: 20;
  animation: fadeIn 0.18s ease;
}

.submittingCard {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  padding: 28px 32px;
  background: var(--bg1);
  border: 1px solid var(--bdr2);
  border-radius: var(--rxl, 12px);
  max-width: 320px;
  width: 90%;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.28);
  text-align: center;
}

.submittingSpinner {
  width: 36px;
  height: 36px;
  border: 3px solid var(--bdr2);
  border-top-color: var(--blue);
  border-right-color: rgba(139, 92, 246, 0.5);
  border-radius: 50%;
  animation: spin 0.75s linear infinite;
  flex-shrink: 0;
}

.submittingTitle {
  font-size: 12px;
  font-weight: 600;
  color: var(--t0);
  letter-spacing: -0.2px;
}

.submittingDesc {
  font-size: 11px;
  color: var(--t2);
  font-family: var(--font-mono);
  line-height: 1.65;
}

.submittingFiles {
  display: flex;
  flex-direction: column;
  gap: 5px;
  width: 100%;
  margin-top: 2px;
}

.submittingFile {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 11px;
  color: var(--t1);
  font-family: var(--font-mono);
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--r);
  padding: 5px 10px;
  text-align: left;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.submittingDot {
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: var(--blue);
  flex-shrink: 0;
  animation: breathe 1.2s ease-in-out infinite;
}

.submittingMore {
  font-size: 10px;
  color: var(--t2);
  font-family: var(--font-mono);
  text-align: center;
  padding: 2px 0;
}

// ── Stop button on queued file cards ─────────────────────
.stopBtn {
  width: 24px;
  height: 24px;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  border: 1px solid rgba(239, 68, 68, 0.35);
  background: rgba(239, 68, 68, 0.08);
  color: #ef4444;
  cursor: pointer;
  padding: 0;
  transition: all 0.15s;
  margin-left: auto;

  svg {
    width: 9px;
    height: 9px;
  }

  &:hover:not(:disabled) {
    background: rgba(239, 68, 68, 0.18);
    border-color: rgba(239, 68, 68, 0.6);
    box-shadow: 0 0 6px rgba(239, 68, 68, 0.2);
  }

  &:disabled {
    opacity: 0.5;
    cursor: default;
  }
}

.stopSpinner {
  width: 10px;
  height: 10px;
  border: 1.5px solid rgba(239, 68, 68, 0.3);
  border-top-color: #ef4444;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}

// ── Large screen overrides (> 1900px) ────────────────────
@media (min-width: 1920px) {
  .slbl {
    font-size: 11px;
  }

  .selSummary {
    font-size: 12px;
  }

  .selCt {
    font-size: 12px;
  }

  .selNm {
    font-size: 11px;
  }

  .cardT {
    font-size: 9px;
  }

  .fl {
    font-size: 12px;
  }

  .fc {
    font-size: 12px;
  }

  .optTag {
    font-size: 11px;
  }

  .cr label {
    font-size: 12px;
  }

  .slim {
    font-size: 11px;
  }

  .sv {
    font-size: 12px;
  }

  .qTypeLabel {
    font-size: 12px;
  }

  .noPromptHint {
    font-size: 12px;
  }

  .btn {
    font-size: 12px;
  }

  .batchSectionTitle {
    font-size: 11px;
  }

  .batchColHead {
    font-size: 11px;
  }

  .batchColCount {
    font-size: 12px;
  }

  .batchEmpty {
    font-size: 12px;
  }

  .statusName {
    font-size: 12px;
  }

  .statusIdBadge {
    font-size: 9px;
  }

  .statusDate {
    font-size: 9px;
  }

  .statusPct {
    font-size: 9px;
  }

  .queuedMeta {
    font-size: 9px;
  }

  .completedMeta {
    font-size: 9px;
  }

  .loadingDot {
    font-size: 12px;
  }

  .refreshBtn {
    font-size: 11px;
  }

  .modelWarn {
    font-size: 12px;
  }

  .modelOk {
    font-size: 11px;
  }

  .submittingTitle {
    font-size: 13px;
  }

  .submittingDesc {
    font-size: 12px;
  }

  .submittingFile {
    font-size: 12px;
  }

  .submittingMore {
    font-size: 11px;
  }

  .liveStatus {
    font-size: 11px;
  }
}

// ── Per-file prompt preview ───────────────────
.perFileBtn {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  margin-left: 8px;
  padding: 2px 8px 2px 6px;
  border-radius: 5px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  font-size: 9px;
  font-family: var(--font-ui);
  cursor: pointer;
  vertical-align: middle;
  transition: all 0.14s;

  svg:first-child {
    width: 12px;
    height: 12px;
    flex-shrink: 0;
  }
}

.perFileBtn:hover {
  background: var(--bg3);
  border-color: var(--bdr3);
  color: var(--t0);
}

.perFileBtnActive {
  background: var(--blue-dim);
  border-color: var(--blue-bdr);
  color: var(--blue);
}

.perFileBtnActive:hover {
  background: var(--blue-dim);
  border-color: var(--blue);
  color: var(--blue);
}

.perFileChevron {
  width: 10px;
  height: 10px;
  flex-shrink: 0;
  transition: transform 0.18s ease;
}

.perFileChevronOpen {
  transform: rotate(180deg);
}

// Animated expand container
.perFilePanel {
  display: grid;
  grid-template-rows: 0fr;
  opacity: 0;
  transition:
    grid-template-rows 0.22s cubic-bezier(0.4, 0, 0.2, 1),
    opacity 0.18s ease,
    margin 0.2s ease;
  margin-bottom: 0;
}

.perFilePanelOpen {
  grid-template-rows: 1fr;
  opacity: 1;
  margin-bottom: 6px;
}

.perFilePanelInner {
  min-height: 0;
  overflow: hidden;
  border: 1px solid var(--bdr2);
  border-radius: var(--r);
  background: var(--bg1);
  max-height: 200px;
  overflow-y: auto;
}

.perFileRow {
  padding: 8px 10px;
  border-bottom: 1px solid var(--bdr1);
  display: flex;
  flex-direction: column;
  gap: 3px;

  &:last-child {
    border-bottom: none;
  }
}

.perFileName {
  font-size: 9px;
  font-family: var(--font-mono);
  color: var(--t2);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.perFilePrompt {
  font-size: 10px;
  color: var(--t0);
  line-height: 1.5;
  white-space: pre-wrap;
  word-break: break-word;
}

.perFileEmpty {
  font-size: 10px;
  color: var(--t2);
  font-style: italic;
  opacity: 0.6;
}

@media (min-width: 1920px) {
  .perFileBtn {
    font-size: 10px;
  }

  .perFileName {
    font-size: 10px;
  }

  .perFilePrompt {
    font-size: 11px;
  }

  .perFileEmpty {
    font-size: 11px;
  }
}

// ── Prompt save / cancel bar ──────────────────
.promptActions {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 8px;
  padding: 10px 12px 12px;
  border-top: 1px solid var(--bdr1);
  margin-top: 4px;
}

.promptSaveError {
  flex: 1;
  font-size: 10px;
  color: var(--red, #ef4444);
  margin-right: 4px;
}

.btnSm {
  height: 30px;
  padding: 0 14px;
  font-size: 10px;
  border-radius: 6px;
  border: 1px solid transparent;
  font-family: var(--font-ui);
  font-weight: 500;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  transition: all 0.14s;

  &:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }
}

.btnGhost {
  background: transparent;
  border-color: var(--bdr2);
  color: var(--t1);

  &:hover:not(:disabled) {
    background: var(--bg3);
    border-color: var(--bdr3);
  }
}

.btnPrimary {
  background: var(--blue);
  border-color: var(--blue);
  color: #fff;

  &:hover:not(:disabled) {
    opacity: 0.88;
  }
}

.btnSpinner {
  width: 11px;
  height: 11px;
  border: 1.5px solid rgba(255, 255, 255, 0.35);
  border-top-color: #fff;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  flex-shrink: 0;
}

@media (min-width: 1920px) {
  .btnSm {
    height: 34px;
    font-size: 11px;
    padding: 0 16px;
  }

  .promptSaveError {
    font-size: 11px;
  }
}

// ─────────────────────────────────────────────
// Minimize / minimized rail
// ─────────────────────────────────────────────

.minimizeStepBtn {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  background: transparent;
  color: var(--t2);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.14s;

  svg {
    width: 13px;
    height: 13px;
  }

  &:hover {
    background: var(--bg3);
    border-color: var(--bdr3);
    color: var(--t0);
  }
}

.infpanelMinimized {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.minRail {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  gap: 14px;
  padding: 14px 0;
  width: 100%;
  background: var(--bg2);
  border: 1px solid var(--bdr);
  border-radius: var(--rl);
  color: var(--t1);
  cursor: pointer;
  transition: background 0.14s, border-color 0.14s, color 0.14s;

  &:hover {
    background: var(--bg3);
    border-color: var(--bdr2);
    color: var(--t0);
  }
}

.minRailIcon {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 7px;
  border: 1px solid var(--bdr2);
  color: var(--t2);
  flex-shrink: 0;

  svg {
    width: 14px;
    height: 14px;
  }

  .minRail:hover & {
    color: var(--t0);
    border-color: var(--bdr3);
  }
}

.minRailLabel {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  writing-mode: vertical-rl;
  transform: rotate(180deg);
  font-size: 10px;
  font-weight: 500;
  letter-spacing: 0.04em;
  color: var(--t1);
  white-space: nowrap;
  user-select: none;
}

.minRailDot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--amber);
  animation: minRailPulse 1.2s infinite;
}

@keyframes minRailPulse {

  0%,
  100% {
    opacity: 1;
  }

  50% {
    opacity: 0.4;
  }
}
