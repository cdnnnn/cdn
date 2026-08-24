@use '../../styles/_variables' as *;

// ===========================================================================
// Custom Metrics — Dashboard / Create Metric / Upload Dataset.
// Mirrors the History/Reports/Comparison/Sidebar design system: ink/paper
// palette, ultramarine signal accent, mono instrument labels, hover-lift.
// Shared by all three sub-pages via CSS module composition.
// ===========================================================================

$ink:      var(--ink-1);
$ink-2:    var(--ink-2);
$ink-3:    var(--ink-3);
$paper:    var(--paper);
$card:     var(--card);
$line:     var(--line);
$line-2:   var(--line-2);
$signal:   #2B2BF5;
$signal-2: #1C1CC7;
$wash:     var(--signal-wash);
$ok:       #0FA968;
$ok-wash:  var(--ok-wash);
$amber:    #E08600;
$amber-wash: var(--amber-wash);
$danger:   #DC2626;
$danger-wash: var(--danger-wash);
$violet:   #6D28D9;
$violet-wash: rgba(109, 40, 217, 0.1);
$sky:      #0369A1;
$sky-wash: var(--sky-wash);
$ink-wash: var(--ink-wash);

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

// Font scaling: matches the Sidebar's pattern — `.cm` sets a single base
// font-size, and every descendant font-size below is expressed in `em`
// (relative to that base), so bumping `.cm`'s font-size on wide screens
// scales the whole feature proportionally from one place.
$cm-base-font: 0.875rem;

%micro {
  font-family: $mono;
  font-size: 0.7857em;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

@keyframes cm-spin { to { transform: rotate(360deg); } }
@keyframes cm-toast-in {
  from { opacity: 0; transform: translate(-50%, 8px); }
  to   { opacity: 1; transform: translate(-50%, 0); }
}

// ---- shared page header -----------------------------------------------
.cm {
  // master scale control — every em-based font-size in this module
  // responds to this (mirrors Sidebar.module.scss)
  font-size: $cm-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 20px;
    border-bottom: 1px solid $line;
    background: $card;

    h1 {
      font-family: $display;
      font-size: 1.7143em;
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $ink;
      line-height: 1.2;
    }
  }

  &__header-eyebrow {
    @extend %micro;
    display: flex;
    align-items: center;
    gap: 8px;
    color: $signal;
    margin-bottom: 6px;

    &::before {
      content: '';
      width: 16px;
      height: 2px;
      border-radius: 2px;
      background: $signal;
    }
  }

  &__header-sub {
    margin-top: 4px;
    font-size: 0.9643em;
    color: $ink-2;
  }
}

.pg-body-scroll {
  overflow-y: auto;
  padding: 20px 32px 32px;
}

// ---- buttons -------------------------------------------------------------
.btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 9px 16px;
  border-radius: 10px;
  border: 1px solid $line;
  background: $card;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.9286em;
  font-weight: 650;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease, box-shadow 0.15s ease, transform 0.15s ease;

  &:hover:not(:disabled) { border-color: $ink-3; color: $ink; box-shadow: $soft; }
  &:disabled { opacity: 0.45; cursor: not-allowed; }
}

.btn-sm { padding: 6px 11px; font-size: 0.8571em; border-radius: 8px; }

.btn-primary {
  border-color: $signal;
  background: $signal;
  color: #fff;

  &:hover:not(:disabled) { background: $signal-2; border-color: $signal-2; color: #fff; transform: translateY(-1px); box-shadow: $lift; }
}

.btn-secondary {
  background: $paper;
}

.btn-ai {
  border-color: rgba($violet, 0.3);
  background: $violet-wash;
  color: $violet;

  &:hover:not(:disabled) { border-color: $violet; background: rgba($violet, 0.16); color: $violet; }
}

.btn-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border-radius: 7px;
  border: 1px solid transparent;
  background: transparent;
  color: $ink-3;
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease, border-color 0.15s ease;

  &:hover { background: $danger-wash; border-color: rgba($danger, 0.2); color: $danger; }
}

.spin { animation: cm-spin 0.8s linear infinite; }

// ---- toggle groups ---------------------------------------------------------
.btn-group {
  display: inline-flex;
  padding: 3px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 11px;
  gap: 2px;
}

.btn-toggle {
  padding: 7px 16px;
  border-radius: 8px;
  border: none;
  background: transparent;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.9286em;
  font-weight: 650;
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease, box-shadow 0.15s ease;

  &:hover { color: $ink; }

  &.active {
    background: $card;
    color: $signal;
    box-shadow: $soft;
    font-weight: 700;
  }
}

.toggle-container {
  display: inline-flex;
  border: 1px solid $line;
  border-radius: 11px;
  overflow: hidden;
}

.toggle-btn {
  padding: 9px 18px;
  border: none;
  background: $paper;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.9286em;
  font-weight: 650;
  cursor: pointer;
  transition: background 0.15s ease, color 0.15s ease;

  &:hover { color: $ink; }

  &.active {
    background: $signal;
    color: #fff;
  }

  &:not(:last-child) { border-right: 1px solid $line; }
}

// ---- cards (dashboard) ------------------------------------------------------
.cards-row {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 16px;
}

.card {
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $soft;
  overflow: hidden;
}

.card-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 18px;
  border-bottom: 1px solid $line;

  h3 {
    font-family: $display;
    font-size: 1.0714em;
    font-weight: 700;
    color: $ink;
  }
}

.card-body { padding: 4px 0 8px; }

// ---- generic table -----------------------------------------------------
.table-wrap {
  overflow-x: auto;
}

.table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.9286em;

  thead th {
    text-align: left;
    background: $paper;
    @extend %micro;
    font-size: 0.6429em;
    color: $ink-3;
    padding: 10px 18px;
    white-space: nowrap;
  }

  tbody tr {
    border-top: 1px solid $line-2;
    transition: background 0.13s ease;
    &:hover { background: $paper; }
  }

  tbody td {
    padding: 11px 18px;
    color: $ink;
    vertical-align: middle;
  }
}

.badge {
  display: inline-flex;
  align-items: center;
  font-family: $mono;
  font-size: 0.7143em;
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  border-radius: 6px;
  padding: 3px 8px;
  white-space: nowrap;
  color: $signal;
  background: $wash;

  &--code  { color: $violet; background: $violet-wash; }
  &--model { color: $signal; background: $wash; }
  &--rag   { color: $sky; background: $sky-wash; }
  &--agent { color: $amber; background: $amber-wash; }
}

// ---- forms ---------------------------------------------------------------
.form-group {
  margin-bottom: 20px;

  label {
    display: block;
    @extend %micro;
    font-size: 0.7857em;
    color: $ink-2;
    margin-bottom: 8px;
  }
}

.input,
.select {
  width: 100%;
  border: 1.5px solid $line;
  border-radius: 9px;
  padding: 9px 12px;
  font-size: 0.9286em;
  font-family: $sans;
  color: $ink;
  background: $card;

  &::placeholder { color: $ink-3; }
  &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
}

.select { cursor: pointer; }

.hint {
  font-size: 0.8929em;
  color: $ink-3;
  margin: -4px 0 16px;
}

.panel {
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  box-shadow: $soft;
  padding: 24px;
}

.panel + .panel { margin-top: 16px; }

// ---- metric template cards --------------------------------------------
.metric-templates {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
  margin-bottom: 24px;
}

.metric-card {
  position: relative;
  border: 1.5px solid $line;
  border-radius: 14px;
  padding: 14px;
  cursor: pointer;
  background: $paper;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, background 0.15s ease;

  &:hover { border-color: $ink-3; }

  &.selected {
    border-color: $signal;
    background: $wash;
    box-shadow: 0 0 0 1px $signal inset;
  }
}

.metric-card-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 6px;
  font-family: $display;
  font-weight: 700;
  font-size: 0.9643em;
  color: $ink;

  input[type='radio'] { accent-color: $signal; }
}

.metric-card p {
  font-size: 0.8571em;
  color: $ink-2;
  line-height: 1.45;
  margin-bottom: 8px;
}

.metric-card code {
  display: block;
  font-family: $mono;
  font-size: 0.7857em;
  color: $signal;
  background: $card;
  border: 1px solid $line;
  border-radius: 7px;
  padding: 6px 8px;
  overflow-x: auto;
  white-space: nowrap;
}

// ---- custom rule builder ---------------------------------------------
.custom-rules-section {
  border-top: 1px solid $line;
  padding-top: 20px;
  margin-bottom: 20px;

  h4 {
    font-family: $display;
    font-size: 1em;
    font-weight: 700;
    color: $ink;
    margin-bottom: 12px;
  }
}

.optional {
  @extend %micro;
  font-size: 0.7143em;
  color: $ink-3;
  margin-left: 6px;
}

.rule-item {
  margin-bottom: 8px;
}

.rule-fields {
  display: flex;
  align-items: center;
  gap: 8px;

  select, input {
    flex-shrink: 0;
  }
}

.rule-field-select { width: 150px; }
.rule-operator { width: 140px; }
.rule-compare-type { width: 140px; }
.rule-value { flex: 1; min-width: 160px; }

.add-rule-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 10px 0 16px;
}

.gate-select { width: 90px; font-weight: 700; color: $signal; }

.rule-preview {
  label {
    display: block;
    @extend %micro;
    font-size: 0.7143em;
    color: $ink-3;
    margin-bottom: 6px;
  }

  code {
    display: block;
    font-family: $mono;
    font-size: 0.8571em;
    color: $ink;
    background: $paper;
    border: 1px solid $line;
    border-radius: 9px;
    padding: 10px 12px;
  }
}

.threshold-row {
  display: flex;
  gap: 24px;
  margin-top: 16px;

  .rule-field {
    label {
      display: block;
      @extend %micro;
      font-size: 0.7143em;
      color: $ink-2;
      margin-bottom: 6px;
    }

    input {
      width: 100px;
    }
  }
}

// ---- code editor -----------------------------------------------------
.code-editor {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
  margin-bottom: 24px;
}

.editor-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 14px;
  background: $paper;
  border-bottom: 1px solid $line;
  @extend %micro;
  font-size: 0.7857em;
  color: $ink-2;
}

.code-area {
  width: 100%;
  min-height: 320px;
  border: none;
  resize: vertical;
  padding: 16px;
  font-family: $mono;
  font-size: 0.8929em;
  line-height: 1.6;
  color: $ink;
  background: $card;

  &:focus { outline: none; }
}

// ---- validation ---------------------------------------------------------
.validation-section {
  border-top: 1px solid $line;
  padding-top: 20px;
  margin-bottom: 24px;
}

.validation-header {
  margin-bottom: 12px;

  h3 {
    font-family: $display;
    font-size: 1.0714em;
    font-weight: 700;
    color: $ink;
  }

  p {
    font-size: 0.8929em;
    color: $ink-3;
    margin-top: 2px;
  }
}

.validation-controls {
  display: flex;
  gap: 10px;
  margin-bottom: 16px;

  select { max-width: 280px; }
}

.validation-results {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
}

.validation-summary {
  display: flex;
  gap: 24px;
  padding: 12px 18px;
  background: $paper;
  border-top: 1px solid $line;
  font-size: 0.9286em;
  color: $ink-2;

  strong { color: $ink; font-family: $mono; }
}

.cell-pass { font-family: $mono; font-weight: 700; color: $ok; }
.cell-fail { font-family: $mono; font-weight: 700; color: $danger; }
.cell-num { font-family: $mono; font-weight: 700; color: $ink; }

// ---- form actions ---------------------------------------------------------
.form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  padding-top: 8px;
}

// ---- upload wizard: steps -------------------------------------------------
.steps {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 28px;
}

.step {
  display: flex;
  align-items: center;
  gap: 8px;
  opacity: 0.55;
  transition: opacity 0.15s ease;

  &.active, &.done { opacity: 1; }
}

.step-num {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: $mono;
  font-size: 0.8571em;
  font-weight: 700;
  color: $ink-3;
  background: $paper;
  border: 1.5px solid $line;
}

.step.active .step-num {
  color: #fff;
  background: $signal;
  border-color: $signal;
}

.step.done .step-num {
  color: $signal;
  background: $wash;
  border-color: $signal;
}

.step-label {
  font-size: 0.9286em;
  font-weight: 650;
  color: $ink-2;
}

.step.active .step-label { color: $ink; font-weight: 700; }

.step-line {
  width: 32px;
  height: 1.5px;
  background: $line;
}

// ---- dropzone --------------------------------------------------------
.dropzone {
  border: 1.5px dashed $line;
  border-radius: 16px;
  padding: 40px 20px;
  text-align: center;
  cursor: pointer;
  background: $paper;
  transition: border-color 0.15s ease, background 0.15s ease;
  margin-bottom: 16px;

  &:hover, &.drag { border-color: $signal; background: $wash; }
}

.dropzone-icon {
  width: 44px;
  height: 44px;
  margin: 0 auto 12px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $card;
  border: 1px solid $line;
  color: $signal;
}

.dropzone p { font-size: 1em; color: $ink; font-weight: 650; margin-bottom: 2px; }
.dropzone-hint { font-size: 0.8929em; color: $ink-3 !important; font-weight: 500 !important; }
.dropzone-formats { font-family: $mono; font-size: 0.7857em !important; color: $ink-3 !important; margin-top: 8px !important; font-weight: 500 !important; }

.file-info {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 14px;
  border: 1px solid $line;
  border-radius: 12px;
  background: $card;
  margin-bottom: 16px;

  .file-name { font-weight: 700; color: $ink; font-size: 0.9286em; }
  .file-size { font-family: $mono; font-size: 0.8214em; color: $ink-3; }
}

// ---- detected columns / chips ------------------------------------------
.detected-columns {
  margin-top: 20px;

  h4 {
    @extend %micro;
    font-size: 0.7857em;
    color: $ink-2;
    margin-bottom: 10px;
  }
}

.column-chips {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.chip {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-family: $mono;
  font-size: 0.7857em;
  font-weight: 700;
  color: $signal;
  background: $wash;
  border: 1px solid rgba($signal, 0.18);
  border-radius: 999px;
  padding: 4px 10px;

  span { color: $ink-3; font-weight: 500; text-transform: none; letter-spacing: 0; }
}

// ---- field mapping --------------------------------------------------------
.mapping-row {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 12px 0;
  border-bottom: 1px solid $line-2;

  &:last-child { border-bottom: none; }
}

.mapping-target {
  flex: 0 0 220px;
  font-family: $display;
  font-weight: 700;
  font-size: 0.9643em;
  color: $ink;
}

.required { color: $danger; margin-right: 3px; }

.mapping-hint {
  display: block;
  font-family: $sans;
  font-weight: 500;
  font-size: 0.8214em;
  color: $ink-3;
  margin-top: 2px;
}

.mapping-arrow { color: $ink-3; flex-shrink: 0; }
.mapping-source { flex: 1; }

.ai-assist {
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 20px 0;
}

.ai-hint { font-size: 0.8571em; color: $ink-3; }

// ---- json preview / summary --------------------------------------------
.json-preview {
  background: $paper;
  border: 1px solid $line;
  border-radius: 14px;
  padding: 16px;
  margin-bottom: 20px;
  max-height: 340px;
  overflow: auto;

  pre {
    font-family: $mono;
    font-size: 0.8571em;
    line-height: 1.6;
    color: $ink;
    white-space: pre-wrap;
    word-break: break-word;
  }
}

.transform-summary {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 12px;
  margin-bottom: 24px;
}

.summary-item {
  display: flex;
  flex-direction: column;
  gap: 4px;
  padding: 14px 16px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 14px;

  .summary-label { @extend %micro; font-size: 0.6429em; color: $ink-3; }
  .summary-value { font-family: $mono; font-size: 1.2143em; font-weight: 700; color: $ink; }
  .summary-value--danger { color: $danger; }
  .summary-value--ok { color: $ok; }
}

.empty {
  padding: 24px;
  text-align: center;
  color: $ink-3;
  font-size: 0.9286em;
}

// ---- toast --------------------------------------------------------------
.toast {
  position: fixed;
  left: 50%;
  bottom: 28px;
  transform: translateX(-50%);
  z-index: 200;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 11px 18px;
  border-radius: 11px;
  background: #14161B;
  color: #fff;
  font-size: 0.9286em;
  font-weight: 650;
  box-shadow: $lift;
  animation: cm-toast-in 0.18s ease;

  &--ok::before { content: ''; width: 6px; height: 6px; border-radius: 50%; background: $ok; flex-shrink: 0; }
  &--error::before { content: ''; width: 6px; height: 6px; border-radius: 50%; background: #FF6B6B; flex-shrink: 0; }
}

@media (max-width: 900px) {
  .cards-row { grid-template-columns: 1fr; }
  .metric-templates { grid-template-columns: 1fr; }
  .transform-summary { grid-template-columns: repeat(2, 1fr); }
}

@media (max-width: 640px) {
  .cm__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .pg-body-scroll { padding: 16px 18px 22px; }
  .rule-fields { flex-wrap: wrap; }
  .mapping-row { flex-wrap: wrap; }
  .mapping-target { flex: 1 1 100%; }
}
// ---- Add to CustomMetrics.module.scss -----------------------------------
// Paste at the end of the file, before the closing @media blocks (or after
// them — order doesn't matter since these are new, non-overlapping classes).

// ---- wizard step rail (supports 8 steps — scrolls on narrow screens) ----
.steps {
  overflow-x: auto;
  padding-bottom: 2px;

  &::-webkit-scrollbar { height: 4px; }
  &::-webkit-scrollbar-thumb { background: $line; border-radius: 2px; }
}

// ---- section heading inside a wizard step --------------------------------
.step-heading {
  font-family: $display;
  font-size: 1.0714em; // 0.9375rem / 0.875rem
  font-weight: 700;
  color: $ink;
  margin-bottom: 4px;
}

.step-sub {
  font-size: 0.8929em; // 0.78125rem / 0.875rem
  color: $ink-3;
  margin-bottom: 20px;
}

.textarea {
  width: 100%;
  min-height: 96px;
  border: 1.5px solid $line;
  border-radius: 9px;
  padding: 9px 12px;
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  font-family: $sans;
  color: $ink;
  background: $card;
  resize: vertical;

  &::placeholder { color: $ink-3; }
  &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
}

// ---- multi-select checkbox list (built-in checks) --------------------
.checkbox-list {
  display: flex;
  flex-direction: column;
  gap: 2px;
  border: 1px solid $line;
  border-radius: 12px;
  overflow: hidden;
}

.checkbox-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 14px;
  background: $card;
  cursor: pointer;
  transition: background 0.13s ease;

  &:not(:last-child) { border-bottom: 1px solid $line-2; }
  &:hover { background: $paper; }

  input[type='checkbox'] { accent-color: $signal; flex-shrink: 0; }
}

.checkbox-item__name {
  font-weight: 650;
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  color: $ink;
}

.checkbox-item__logic {
  font-family: $mono;
  font-size: 0.7857em; // 0.6875rem / 0.875rem
  color: $ink-3;
  margin-left: auto;
}

// ---- code editor language badge ------------------------------------------
.lang-badge {
  display: inline-flex;
  align-items: center;
  font-family: $mono;
  font-size: 0.7143em; // 0.625rem / 0.875rem
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  color: $ok;
  background: $ok-wash;
  border-radius: 6px;
  padding: 3px 8px;
}

.editor-header-left {
  display: flex;
  align-items: center;
  gap: 8px;
}

// ---- judge model list ----------------------------------------------------
.model-list {
  display: flex;
  flex-direction: column;
  gap: 2px;
  border: 1px solid $line;
  border-radius: 12px;
  overflow: hidden;
}

.model-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 11px 14px;
  background: $card;
  cursor: pointer;
  transition: background 0.13s ease, opacity 0.13s ease;

  &:not(:last-child) { border-bottom: 1px solid $line-2; }
  &:hover:not(&--disabled) { background: $paper; }

  &--selected { background: $wash; }
  &--disabled { cursor: not-allowed; opacity: 0.5; }

  input[type='radio'] { accent-color: $signal; flex-shrink: 0; }
}

.model-item__name {
  font-weight: 650;
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  color: $ink;
}

.model-item__provider {
  font-family: $mono;
  font-size: 0.7143em; // 0.625rem / 0.875rem
  color: $ink-3;
}

.model-item__status {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  margin-left: auto;
  font-family: $mono;
  font-size: 0.7143em; // 0.625rem / 0.875rem
  font-weight: 700;
  text-transform: uppercase;
}

.health-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  flex-shrink: 0;

  &--healthy { background: $ok; }
  &--unhealthy { background: $danger; }
  &--checking { background: $ink-3; animation: cm-spin 1s linear infinite; border-radius: 2px; }
}

.status-healthy { color: $ok; }
.status-unhealthy { color: $danger; }
.status-checking { color: $ink-3; }

.badge--simple { color: $signal; background: $wash; }
.badge--active { color: $ok; background: $ok-wash; }
.badge--inactive { color: $ink-3; background: $ink-wash; }

// ---- dataset preview list ---------------------------------------------
.preview-list {
  border: 1px solid $line;
  border-radius: 14px;
  overflow: hidden;
  max-height: 360px;
  overflow-y: auto;
}

.preview-item {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 12px 14px;
  border-bottom: 1px solid $line-2;
  transition: background 0.13s ease;

  &:last-child { border-bottom: none; }
  &:hover { background: $paper; }

  input[type='checkbox'] { accent-color: $signal; margin-top: 3px; flex-shrink: 0; }
}

.preview-item__body { min-width: 0; flex: 1; }
.preview-item__q { font-size: 0.9286em; color: $ink; font-weight: 650; margin-bottom: 3px; }
.preview-item__a { font-size: 0.8214em; color: $ink-2; }
.preview-item__a-label { font-family: $mono; font-size: 0.7143em; color: $ink-3; margin-right: 5px; }

// ---- success modal ---------------------------------------------------
.modal-overlay {
  position: fixed;
  inset: 0;
  z-index: 300;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(10, 12, 18, 0.5);
  padding: 20px;
}

.modal {
  width: 100%;
  max-width: 380px;
  background: $card;
  border-radius: 18px;
  box-shadow: $lift;
  padding: 28px 24px 22px;
  text-align: center;
}

.modal-icon {
  width: 48px;
  height: 48px;
  margin: 0 auto 14px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $ok-wash;
  color: $ok;
}

.modal h3 {
  font-family: $display;
  font-size: 1.1429em; // 1rem / 0.875rem
  font-weight: 800;
  color: $ink;
  margin-bottom: 6px;
}

.modal p {
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  color: $ink-2;
  margin-bottom: 4px;
}

.modal-id {
  display: inline-block;
  font-family: $mono;
  font-size: 0.7857em; // 0.6875rem / 0.875rem
  font-weight: 700;
  color: $signal;
  background: $wash;
  border-radius: 7px;
  padding: 3px 8px;
  margin-bottom: 20px;
}

.modal-actions {
  display: flex;
  gap: 10px;
  justify-content: center;
}

// ---- generic banners -------------------------------------------------
.error-banner {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px;
  border-radius: 10px;
  background: $danger-wash;
  border: 1px solid rgba($danger, 0.2);
  color: $danger;
  font-size: 0.8929em; // 0.78125rem / 0.875rem
  margin-bottom: 16px;
}

.success-banner {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px;
  border-radius: 10px;
  background: $ok-wash;
  border: 1px solid rgba($ok, 0.2);
  color: $ok;
  font-size: 0.8929em; // 0.78125rem / 0.875rem
  margin-bottom: 16px;
  font-weight: 650;
}

.loading-row {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 20px;
  justify-content: center;
  color: $ink-3;
  font-size: 0.8929em; // 0.78125rem / 0.875rem
}
// ---- Add to CustomMetrics.module.scss ------------------------------------

// ---- section wrapper -------------------------------------------------
.section {
  margin-bottom: 28px;

  &:last-child { margin-bottom: 0; }
}

.section-title {
  font-family: $display;
  font-size: 1.0714em; // 0.9375rem / 0.875rem
  font-weight: 700;
  color: $ink;
  margin-bottom: 2px;
}

.section-sub {
  font-size: 0.8929em; // 0.78125rem / 0.875rem
  color: $ink-3;
  margin-bottom: 16px;
}

// ---- selectable card grids (eval type / metric type) ---------------------
.card-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;

  &--4 { grid-template-columns: repeat(4, 1fr); }
}

.select-card {
  position: relative;
  border: 1.5px solid $line;
  border-radius: 14px;
  padding: 16px 14px;
  cursor: pointer;
  background: $paper;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, background 0.15s ease, opacity 0.15s ease;

  &:hover:not(&--disabled) { border-color: $ink-3; }

  &--selected {
    border-color: $signal;
    background: $wash;
    box-shadow: 0 0 0 1px $signal inset;
  }

  &--disabled {
    cursor: not-allowed;
    opacity: 0.5;
  }
}

.select-card__title {
  font-family: $display;
  font-weight: 700;
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  color: $ink;
  margin-bottom: 5px;
}

.select-card__desc {
  font-size: 0.8214em; // 0.71875rem / 0.875rem
  color: $ink-2;
  line-height: 1.45;
}

.select-card__badge {
  position: absolute;
  top: 12px;
  right: 12px;
  font-family: $mono;
  font-size: 0.6429em; // 0.5625rem / 0.875rem
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  color: $ink-3;
  background: $card;
  border: 1px solid $line;
  border-radius: 6px;
  padding: 2px 6px;
}

// ---- rule builder with AND/OR gates between rows -------------------------
.rule-row-wrap {
  display: flex;
  flex-direction: column;
  align-items: stretch;
}

.rule-gate {
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 6px 0;

  &::before, &::after {
    content: '';
    flex: 1;
    height: 1px;
    background: $line;
  }
}

.rule-gate-btn {
  display: inline-flex;
  padding: 2px;
  background: $paper;
  border: 1px solid $line;
  border-radius: 8px;
  gap: 1px;
  flex-shrink: 0;
}

.rule-gate-opt {
  padding: 4px 10px;
  border-radius: 6px;
  border: none;
  background: transparent;
  color: $ink-2;
  font-family: $mono;
  font-size: 0.7143em; // 0.625rem / 0.875rem
  font-weight: 700;
  cursor: pointer;

  &.active { background: $signal; color: #fff; }
}

.rule-summary {
  margin-top: 14px;

  label {
    display: block;
    @extend %micro;
    font-size: 0.625rem;
    color: $ink-3;
    margin-bottom: 6px;
  }

  code {
    display: block;
    font-family: $mono;
    font-size: 0.8214em; // 0.71875rem / 0.875rem
    color: $ink;
    background: $paper;
    border: 1px solid $line;
    border-radius: 9px;
    padding: 10px 12px;
    line-height: 1.6;
  }
}

// ---- prompt builder ----------------------------------------------------
.prompt-template-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 16px;
}

.prompt-template-item {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 12px 14px;
  border: 1.5px solid $line;
  border-radius: 12px;
  cursor: pointer;
  background: $paper;
  transition: border-color 0.15s ease, background 0.15s ease;

  &:hover { border-color: $ink-3; }
  &--selected { border-color: $signal; background: $wash; }

  input[type='radio'] { accent-color: $signal; margin-top: 3px; flex-shrink: 0; }
}

.prompt-template-item__label { font-weight: 700; font-size: 0.9286em; color: $ink; }
.prompt-template-item__desc { font-size: 0.8214em; color: $ink-2; margin-top: 2px; }
.prompt-template-item__placeholders {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  margin-top: 6px;
}

// ---- dataset preview toolbar --------------------------------------------
.preview-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 8px;
}

.preview-toolbar__actions {
  display: flex;
  gap: 6px;
}

.link-btn {
  border: none;
  background: none;
  padding: 0;
  color: $signal;
  font-size: 0.8214em; // 0.71875rem / 0.875rem
  font-weight: 650;
  cursor: pointer;

  &:hover { text-decoration: underline; }
}

// ---- sticky footer -------------------------------------------------------
// True viewport-fixed (not scroll-sticky) so it stays put regardless of how
// far the form is scrolled — offset by the sidebar width on desktop.
.sticky-footer {
  position: fixed;
  bottom: 0;
  left: $sidebar-width;
  right: 0;
  z-index: 20;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 10px;
  padding: 14px 32px;
  background: $card;
  border-top: 1px solid $line;
  box-shadow: 0 -8px 20px -12px rgba(20, 22, 27, 0.15);

  @media (max-width: 768px) {
    left: 0;
  }
}

// applied to .pg-body-scroll on pages that render a .sticky-footer, so the
// last section isn't hidden behind it
.pg-body-scroll--with-footer {
  padding-bottom: 90px;
}

.sticky-footer__info {
  margin-right: auto;
  font-size: 0.8214em; // 0.71875rem / 0.875rem
  color: $ink-3;
}

@media (max-width: 900px) {
  .card-grid, .card-grid--4 { grid-template-columns: 1fr; }
}
// ---- Add to CustomMetrics.module.scss ------------------------------------

// ---- custom dropdown (replaces native <select> in Create Metric) -------
.dropdown {
  position: relative;
  width: 100%;

  &--disabled { opacity: 0.5; pointer-events: none; }
}

.dropdown__trigger {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  border: 1.5px solid $line;
  border-radius: 9px;
  padding: 9px 11px;
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  font-family: $sans;
  color: $ink;
  background: $card;
  cursor: pointer;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  text-align: left;

  &:hover { border-color: $ink-3; }

  &--open {
    border-color: $signal;
    box-shadow: 0 0 0 3px $wash;
  }
}

.dropdown__value { color: $ink; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.dropdown__placeholder { color: $ink-3; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

.dropdown__chevron {
  flex-shrink: 0;
  color: $ink-3;
  transition: transform 0.15s ease;

  &--open { transform: rotate(180deg); }
}

.dropdown__menu {
  position: absolute;
  top: calc(100% + 6px);
  left: 0;
  // No `right: 0` here on purpose — that forced the menu to exactly match
  // the trigger's width, so a squeezed flex-item trigger (e.g. the 4th
  // rule column, which sits in a `flex: 1` slot) produced an unreadably
  // narrow menu. `min-width: 100%` keeps it at least as wide as the
  // trigger; `width: max-content` lets it grow to fit the longest option
  // label instead of being clipped to the trigger's shrunk width.
  min-width: 100%;
  width: max-content;
  max-width: 320px;
  z-index: 40;
  max-height: 260px;
  overflow-y: auto;
  background: $card;
  border: 1px solid $line;
  border-radius: 12px;
  box-shadow: $lift;
  padding: 4px;
}

.dropdown__option {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  padding: 8px 10px;
  border: none;
  border-radius: 8px;
  background: transparent;
  color: $ink-2;
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  font-weight: 550;
  text-align: left;
  cursor: pointer;
  transition: background 0.12s ease, color 0.12s ease;

  &:hover { background: $paper; color: $ink; }

  &--selected {
    background: $wash;
    color: $signal;
    font-weight: 700;
  }

  svg { flex-shrink: 0; color: $signal; }
}

.dropdown__option-sub {
  display: block;
  font-family: $mono;
  font-size: 0.8571em; // relative to option's own font-size
  font-weight: 500;
  color: $ink-3;
  margin-top: 1px;
}

.dropdown__empty {
  padding: 12px;
  text-align: center;
  color: $ink-3;
  font-size: 0.8214em; // 0.71875rem / 0.875rem
}

// ---- dataset selection cards --------------------------------------------
.dataset-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
  gap: 12px;
  margin-bottom: 16px;
}

.dataset-card {
  position: relative;
  border: 1.5px solid $line;
  border-radius: 14px;
  padding: 14px 16px;
  cursor: pointer;
  background: $paper;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, background 0.15s ease;
  display: flex;
  align-items: center;
  gap: 12px;

  &:hover { border-color: $ink-3; }

  &--selected {
    border-color: $signal;
    background: $wash;
    box-shadow: 0 0 0 1px $signal inset;
  }
}

.dataset-card__radio {
  flex-shrink: 0;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  border: 1.5px solid $line-2;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: border-color 0.15s ease;

  .dataset-card--selected & { border-color: $signal; }

  &::after {
    content: '';
    width: 9px;
    height: 9px;
    border-radius: 50%;
    background: $signal;
    opacity: 0;
    transition: opacity 0.15s ease;
  }

  .dataset-card--selected &::after { opacity: 1; }
}

.dataset-card__body { min-width: 0; flex: 1; }
.dataset-card__name {
  font-family: $display;
  font-weight: 700;
  font-size: 0.9286em; // 0.8125rem / 0.875rem
  color: $ink;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.dataset-card__count {
  font-family: $mono;
  font-size: 0.7857em; // 0.6875rem / 0.875rem
  color: $ink-3;
  margin-top: 2px;
}
