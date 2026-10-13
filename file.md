//RagPanel.module.scss
@use 'sass:color';
@use '../../styles/_variables' as *;

// ===========================================================================
// RAG upload + processing panel — embedded as the second column of the
// dataset detail view for RAG datasets (replaces the old RagStudio modal).
// Sections (Upload / Processing / Files) are the same cards as before; only
// the modal chrome (overlay, gradient header, footer) is gone.
//
// Reuses the shared "ink" theme tokens from _variables.scss so dark mode
// keeps working without anything hardcoded beyond the amber queued tint.
// ===========================================================================

$queued-amber: #E08600;
$queued-amber-wash: #FDF3E3;

// Base font-size — same em-scaling convention as Datasets.
$rag-panel-base-font: 0.8125rem;

.rag-panel {
  font-size: $rag-panel-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  display: flex;
  flex-direction: column;
  gap: 16px;

  // ---- header ---------------------------------------------------------------
  &__hdr {
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 10px 12px;
  }

  &__hdr-actions {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-left: auto;
    // Room for the stamp's entry scale-up so it isn't clipped by the scroll container.
    padding: 4px 4px 4px 0;
  }

  &__hdr-info {
    display: flex;
    align-items: center;
    gap: 10px;
    min-width: 0;

    h2 {
      font-family: $font-display;
      font-size: 1.2308em; // 1rem / 0.8125rem
      font-weight: 700;
      letter-spacing: -0.01em;
      color: $ink;
      margin: 2px 0 0;
    }
  }

  &__hdr-icon {
    flex-shrink: 0;
    width: 32px;
    height: 32px;
    border-radius: 10px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    background: $wash;
    color: $signal;
  }

  &__eyebrow {
    font-family: $font-mono;
    font-size: 0.6923em; // 0.5625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    color: $ink-3;
    margin: 0;
  }

  &__refresh {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 13px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $card;
    color: $ink-2;
    font-family: $font-body;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

    &:hover { border-color: $ink-3; color: $ink; }
  }

  // ---- phase marker (header) ---------------------------------------------------
  &__phase {
    position: relative;
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-family: $font-mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 800;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    white-space: nowrap;
    user-select: none;
  }

  // Upload — calm, dashed "action needed" tag in the app's signal color.
  &__phase--upload {
    padding: 6px 12px;
    border: 1.5px dashed $signal;
    border-radius: 999px;
    color: $signal;
    background: $wash;
    animation: rag-panel-phase-in 0.3s ease both;
  }

  // Processing — solid highlighter with a light sweep passing across it.
  &__phase--processing {
    padding: 6px 13px;
    border-radius: 999px;
    color: #fff;
    background: $signal;
    overflow: hidden;
    box-shadow: 0 0 0 3px $wash, 0 4px 12px rgba(43, 43, 245, 0.25);
    animation: rag-panel-phase-in 0.3s ease both;

    &::after {
      content: '';
      position: absolute;
      inset: 0;
      width: 50%;
      background: linear-gradient(100deg, transparent, rgba(255, 255, 255, 0.45), transparent);
      transform: translateX(-120%);
      animation: rag-panel-sweep 1.8s ease-in-out infinite;
      pointer-events: none;
    }
  }

  // Completed — stamp look: double border, upright, "stamped" onto the page
  // with a quick scale-down on entry.
  &__phase--completed {
    padding: 6px 14px 6px 11px;
    color: $ok;
    background: rgba($ok, 0.06);
    border: 2px solid $ok;
    border-radius: 7px;
    font-size: 0.8462em;
    letter-spacing: 0.18em;
    // inner hairline = the second ring of the stamp
    box-shadow: inset 0 0 0 2px $card, inset 0 0 0 3px $ok;
    animation: rag-panel-stamp 0.45s cubic-bezier(0.2, 0.9, 0.3, 1.25) both;
  }

  // ---- sections -------------------------------------------------------------
  &__section {
    background: $card;
    border: 1px solid $line;
    border-radius: 16px;
    padding: 18px;
  }

  &__section-hdr {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    margin-bottom: 14px;

    h3 {
      font-family: $font-display;
      font-size: 1.0769em; // 0.875rem / 0.8125rem
      font-weight: 700;
      color: $ink;
      margin: 0;
      display: flex;
      align-items: center;
      gap: 8px;

      em {
        font-family: $font-mono;
        font-style: normal;
        font-size: 0.7143em;
        font-weight: 700;
        color: $ink-3;
        background: $paper;
        border: 1px solid $line;
        border-radius: 99px;
        padding: 1px 8px;
      }
    }
  }

  &__section-hint {
    font-family: $font-mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
    text-align: right;
  }

  &__state-tag {
    font-family: $font-mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.05em;
    text-transform: uppercase;
    color: $signal;
    background: $wash;
    border-radius: 999px;
    padding: 3px 10px;
    white-space: nowrap;
  }

  // ---- dropzone --------------------------------------------------------------
  &__dropzone {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 6px;
    padding: 30px 16px;
    border: 1.5px dashed $line;
    border-radius: 14px;
    background: $paper;
    color: $ink-3;
    cursor: pointer;
    text-align: center;
    transition: border-color 0.15s ease, background 0.15s ease, transform 0.1s ease;

    &:hover { border-color: $signal; background: $wash; }

    &--over {
      border-color: $signal;
      background: $wash;
      transform: scale(1.005);
    }

    svg { color: $signal; margin-bottom: 2px; }
  }

  &__dropzone-text {
    font-family: $font-body;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 650;
    color: $ink;
  }

  &__dropzone-hint {
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    color: $ink-3;
  }

  &__file-input {
    display: none;
  }

  &__locked-note {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 13px 15px;
    border: 1px dashed $line;
    border-radius: 12px;
    background: $paper;
    color: $ink-2;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    line-height: 1.5;

    svg { flex-shrink: 0; color: $ink-3; }
  }

  &__inline-warn,
  &__inline-error {
    display: flex;
    align-items: center;
    gap: 7px;
    margin-top: 10px;
    padding: 8px 11px;
    border-radius: 9px;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    font-weight: 600;
    line-height: 1.5;

    svg { flex-shrink: 0; }
  }

  &__inline-warn {
    background: $queued-amber-wash;
    color: $queued-amber;
  }

  &__inline-error {
    background: $danger-wash;
    color: $danger;
  }

  // ---- staged files ------------------------------------------------------------
  &__staged-list {
    margin-top: 14px;
  }

  &__staged-chip {
    display: flex;
    align-items: center;
    gap: 6px;
    height: 100%;
    box-sizing: border-box;
    max-width: 100%;
    padding: 5px 6px 5px 10px;
    border: 1px solid $line;
    border-radius: 999px;
    background: $paper;
    font-size: 0.8462em; // 0.6875rem / 0.8125rem
    color: $ink-2;

    svg:first-child { flex-shrink: 0; color: $ink-3; }

    span {
      flex: 1;
      min-width: 0;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
      font-weight: 600;
      color: $ink;
    }

    em {
      flex-shrink: 0;
      font-family: $font-mono;
      font-style: normal;
      font-size: 0.85em;
      color: $ink-3;
    }

    button {
      flex-shrink: 0;
      display: inline-flex;
      align-items: center;
      justify-content: center;
      width: 18px;
      height: 18px;
      border: 0;
      border-radius: 999px;
      background: $line-2;
      color: $ink-2;
      cursor: pointer;
      transition: background 0.13s ease, color 0.13s ease;

      &:hover:not(:disabled) { background: $danger-wash; color: $danger; }
      &:disabled { opacity: 0.5; cursor: not-allowed; }
    }
  }

  &__upload-btn {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    margin-top: 14px;
    padding: 9px 16px;
    border: 1px solid transparent;
    border-radius: 10px;
    background: $signal;
    color: #fff;
    font-family: $font-body;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    font-weight: 650;
    cursor: pointer;
    transition: background 0.15s ease;

    &:hover:not(:disabled) { background: $signal-2; }
    &:disabled { opacity: 0.6; cursor: not-allowed; }
  }

  // ---- processing stats --------------------------------------------------------
  &__stats {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(84px, 1fr));
    gap: 8px;
    margin-bottom: 14px;
  }

  &__stat {
    display: flex;
    flex-direction: column;
    gap: 2px;
    padding: 10px 12px;
    border-radius: 12px;
    background: $paper;
    border: 1px solid $line;
  }

  &__stat-val {
    font-family: $font-display;
    font-size: 1.3846em; // 1.125rem / 0.8125rem
    font-weight: 700;
    color: $ink;
    line-height: 1;
  }

  &__stat-label {
    font-family: $font-mono;
    font-size: 0.6154em; // 0.5rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-3;
    margin-top: 4px;
  }

  &__stat--uploaded .rag-panel__stat-val { color: $ink-2; }
  &__stat--queued .rag-panel__stat-val { color: $queued-amber; }
  &__stat--processing .rag-panel__stat-val { color: $signal; }
  &__stat--ready .rag-panel__stat-val { color: $ok; }
  &__stat--failed .rag-panel__stat-val { color: $danger; }

  &__progress-track {
    height: 8px;
    border-radius: 999px;
    background: $line-2;
    overflow: hidden;
    margin-bottom: 14px;
  }

  &__progress-fill {
    height: 100%;
    border-radius: 999px;
    background: linear-gradient(90deg, $signal, $ok);
    transition: width 0.4s ease;
  }

  &__process-btn {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 10px 20px;
    border: 1px solid transparent;
    border-radius: 10px;
    background: $ok;
    color: #fff;
    font-family: $font-body;
    font-size: 0.9615em; // 0.78125rem / 0.8125rem
    font-weight: 700;
    cursor: pointer;
    transition: background 0.15s ease, transform 0.15s ease;

    &:hover:not(:disabled) { background: color.scale($ok, $lightness: -10%); transform: translateY(-1px); }
    &:disabled { opacity: 0.45; cursor: not-allowed; transform: none; }
  }

  // ---- file list -----------------------------------------------------------
  &__empty {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    padding: 28px 12px;
    color: $ink-3;
    font-size: 0.9231em; // 0.75rem / 0.8125rem
    text-align: center;
  }

  // The list element itself is the VirtualList scroller (height/overflow are
  // set by it); rows are absolutely positioned at a fixed height.
  &__file-list {
    padding-right: 2px;
  }

  &__file-row {
    height: 100%;
    box-sizing: border-box;
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 9px 11px;
    border: 1px solid $line;
    border-radius: 10px;
    background: $paper;
  }

  &__file-icon {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 28px;
    height: 28px;
    border-radius: 8px;
    background: $card;
    border: 1px solid $line;
    color: $ink-3;
  }

  &__file-info {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  &__file-name {
    font-size: 0.8846em; // 0.71875rem / 0.8125rem
    font-weight: 650;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__file-meta {
    font-family: $font-mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    color: $ink-3;
  }

  &__file-status {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 3px 9px;
    border-radius: 999px;
    font-family: $font-mono;
    font-size: 0.7692em; // 0.625rem / 0.8125rem
    font-weight: 700;
    letter-spacing: 0.03em;
    text-transform: uppercase;
    white-space: nowrap;

    &--uploaded { color: $ink-2; background: $ink-wash; }
    &--queued { color: $queued-amber; background: $queued-amber-wash; }
    &--processing { color: $signal; background: $wash; }
    &--ready { color: $ok; background: $ok-wash; }
    &--failed { color: $danger; background: $danger-wash; }
    &--other { color: $ink-3; background: $ink-wash; }
  }

  &__file-error-icon {
    flex-shrink: 0;
    display: inline-flex;
    color: $danger;
    cursor: help;
  }

  &__spin {
    animation: rag-panel-spin 0.8s linear infinite;
  }
}

@keyframes rag-panel-spin {
  to { transform: rotate(360deg); }
}

@keyframes rag-panel-phase-in {
  from { opacity: 0; transform: translateY(-4px); }
  to { opacity: 1; transform: translateY(0); }
}

@keyframes rag-panel-sweep {
  0% { transform: translateX(-120%); }
  60%, 100% { transform: translateX(320%); }
}

@keyframes rag-panel-stamp {
  0% { opacity: 0; transform: scale(1.9); }
  60% { opacity: 1; transform: scale(0.94); }
  100% { opacity: 1; transform: scale(1); }
}

@media (prefers-reduced-motion: reduce) {
  .rag-panel__spin,
  .rag-panel__phase,
  .rag-panel__phase--processing::after { animation: none; }
  .rag-panel__phase--processing::after { display: none; }
  .rag-panel__dropzone,
  .rag-panel__upload-btn,
  .rag-panel__process-btn,
  .rag-panel__refresh { transition: none; }
}
