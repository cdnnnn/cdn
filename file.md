@use '../../styles/_variables' as *;

// ===========================================================================
// Providers — matches the Run Console / Dashboard design system:
// ink/paper palette, ultramarine signal accent, mono instrument labels,
// hover-lift cards. Sidebar block keys are kept stable (shared with
// ProviderModelsSidebar) but recolored to the same tokens.
// ===========================================================================

$ink: #14161B;
$ink-2: #565B66;
$ink-3: #8A909B;
$paper: #F5F6F8;
$card: #FFFFFF;
$line: #E6E8EC;
$line-2: #EEF0F3;
$signal: #2B2BF5;
$signal-2: #1C1CC7;
$wash: #ECEDFF;
$ok: #0FA968;
$ok-wash: #E7F7EF;
$amber: #E08600;
$danger: #DC2626;
$danger-wash: #FDECEC;

$mono: $font-mono;
$sans: $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

%micro {
  font-family: $mono;
  font-size: 0.6875rem;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.providers {

  // ---- header -----------------------------------------------------------
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 20px;
    margin-bottom: 20px;
    border-bottom: 1px solid $line;
    background: $card;

    h1 {
      font-family: $display;
      font-size: 1.5rem;
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
    font-size: 0.84375rem;
    color: $ink-2;
  }

  &__header-meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 7px 13px;
    border-radius: 999px;
    border: 1px solid $line;
    background: $paper;
    font-family: $mono;
    font-size: 0.71875rem;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-2;
    white-space: nowrap;
    margin-bottom: 3px;
  }

  // ---- toolbar ------------------------------------------------------------
  &__toolbar {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 14px;
    padding: 14px 32px;
    background: $card;
    border-bottom: 1px solid $line;
    flex-wrap: wrap;
  }

  &__search {
    position: relative;
    flex: 1;
    max-width: 340px;
    min-width: 200px;

    svg {
      position: absolute;
      top: 50%;
      left: 13px;
      transform: translateY(-50%);
      color: $ink-3;
      pointer-events: none;
    }

    input {
      width: 100%;
      border: 1.5px solid $line;
      border-radius: 10px;
      padding: 9px 12px 9px 38px;
      font-size: 0.84375rem;
      font-family: $sans;
      color: $ink;
      background: $paper;
      transition: border-color 0.15s ease, background 0.15s ease;

      &::placeholder {
        color: $ink-3;
      }

      &:focus {
        outline: none;
        border-color: $signal;
        background: $card;
      }
    }
  }

  &__toolbar-right {
    display: flex;
    align-items: center;
    gap: 14px;
    flex-wrap: wrap;
  }

  &__filter-group {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 4px;
    background: $paper;
    border: 1px solid $line;
    border-radius: 999px;
  }

  &__toolbar-label {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 5px;
    padding: 5px 10px 5px 11px;
    @extend %micro;
    font-size: 0.625rem;
    color: $ink-3;
    white-space: nowrap;
  }

  &__filter-pill {
    padding: 6px 13px;
    border: 0;
    border-radius: 999px;
    background: transparent;
    color: $ink-2;
    font-size: 0.78125rem;
    font-weight: 650;
    cursor: pointer;
    transition: all 0.15s ease;

    &:hover {
      color: $ink;
    }

    &--on {
      background: $card;
      color: $signal;
      box-shadow: $soft;
    }
  }

  &__toolbar-divider {
    flex-shrink: 0;
    width: 1px;
    height: 26px;
    background: $line;
  }

  &__add-btn {
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 9px 15px;
    border: 0;
    border-radius: 10px;
    background: $ink;
    color: #fff;
    font-family: $sans;
    font-size: 0.8125rem;
    font-weight: 650;
    cursor: pointer;
    box-shadow: $soft;
    transition: background 0.16s ease, transform 0.16s ease, box-shadow 0.16s ease;

    &:hover {
      background: #000;
      transform: translateY(-1px);
      box-shadow: $lift;
    }
  }

  // ---- provider card grid --------------------------------------------------
  &__grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(380px, 1fr));
    gap: 12px;
  }

  &__card {
    position: relative;
    display: flex;
    flex-direction: column;
    height: 100%;
    padding: 17px 18px;
    border: 1.5px solid $line;
    border-radius: 16px;
    background: $card;
    transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease;

    &:hover {
      border-color: $ink-3;
      box-shadow: $lift;
      transform: translateY(-2px);
    }
  }

  &__card-hdr {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 10px;
  }

  &__card-id {
    display: flex;
    align-items: center;
    gap: 13px;
    min-width: 0;
  }

  &__icon {
    flex-shrink: 0;
    width: 42px;
    height: 42px;
    border-radius: 12px;
    display: grid;
    place-items: center;
    background: $paper;
    border: 1px solid $line;
    color: $ink;
    font-family: $display;
    font-weight: 800;
    font-size: 1.0625rem;

    img {
      width: 24px;
      height: 24px;
      object-fit: contain;
    }
  }

  &__name {
    font-family: $display;
    font-size: 0.9375rem;
    font-weight: 700;
    color: $ink;
    line-height: 1.25;
  }

  &__count {
    font-size: 0.75rem;
    color: $ink-3;
    margin-top: 2px;
  }

  &__card-top-actions {
    display: flex;
    align-items: center;
    gap: 6px;
    flex-shrink: 0;
  }

  &__icon-btn {
    display: grid;
    place-items: center;
    width: 28px;
    height: 28px;
    border: 1px solid $line;
    border-radius: 8px;
    background: $card;
    color: $ink-2;
    cursor: pointer;
    transition: border-color 0.14s ease, color 0.14s ease, background 0.14s ease;

    &:hover {
      border-color: $ink-3;
      color: $ink;
      background: $paper;
    }
  }

  &__badge-connected {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 10px 4px 8px;
    border-radius: 999px;
    font-family: $mono;
    font-size: 0.625rem;
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ok;
    background: $ok-wash;
    white-space: nowrap;

    &::before {
      content: '';
      width: 5px;
      height: 5px;
      border-radius: 50%;
      background: $ok;
    }
  }

  &__badge-idle {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 11px;
    border-radius: 999px;
    font-size: 0.6875rem;
    font-weight: 600;
    color: $ink-3;
    background: transparent;
    border: 1px dashed $line;
    white-space: nowrap;

    &::before {
      content: '';
      flex-shrink: 0;
      width: 6px;
      height: 6px;
      border-radius: 50%;
      background: $ink-3;
      opacity: 0.7;
    }
  }

  &__desc {
    flex: 1;
    margin-top: 11px;
    font-size: 0.8125rem;
    color: $ink-2;
    line-height: 1.5;
  }

  // ---- inline API key form -------------------------------------------------
  &__key-form {
    display: flex;
    gap: 8px;
    margin-top: 12px;
  }

  &__key-input {
    flex: 1;
    border: 1.5px solid $line;
    border-radius: 9px;
    padding: 8px 11px;
    font-size: 0.8125rem;
    font-family: $sans;
    color: $ink;
    background: $paper;
    transition: border-color 0.15s ease, background 0.15s ease;

    &::placeholder {
      color: $ink-3;
    }

    &:focus {
      outline: none;
      border-color: $signal;
      background: $card;
    }
  }

  &__key-actions {
    display: flex;
    gap: 6px;
    flex-shrink: 0;
  }

  // ---- footer action row ---------------------------------------------------
  &__foot-actions {
    display: flex;
    flex-wrap: nowrap;
    align-items: center;
    justify-content: flex-start;
    gap: 6px;
    margin-top: 13px;
  }

  &__foot-btn {
    flex: 0 0 auto;
    width: 82px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 4px;
    padding: 6px 4px;
    border-radius: 8px;
    border: 1px solid transparent;
    font-family: $sans;
    font-size: 0.71875rem;
    font-weight: 650;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    cursor: pointer;
    transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease, transform 0.12s ease;

    &:disabled {
      cursor: not-allowed;
      opacity: 0.55;
    }

    &--primary {
      background: $ink;
      color: #fff;

      &:not(:disabled):hover {
        background: #000;
        transform: translateY(-1px);
      }
    }

    &--accent {
      background: $signal;
      color: #fff;

      &:not(:disabled):hover {
        background: $signal-2;
        transform: translateY(-1px);
      }
    }

    &--ghost {
      background: $card;
      border-color: $line;
      color: $ink-2;

      &:not(:disabled):hover {
        border-color: $ink-3;
        color: $ink;
        background: $paper;
      }
    }

    &--danger {
      background: $danger-wash;
      color: $danger;

      &:not(:disabled):hover {
        background: rgba($danger, 0.16);
      }
    }
  }

  &__spin {
    animation: providers-spin 0.8s linear infinite;
  }

  &__empty {
    grid-column: 1 / -1;
    padding: 40px 20px;
    text-align: center;
    color: $ink-3;
    font-size: 0.84375rem;
    border: 1px dashed $line;
    border-radius: 14px;
  }

  // ===========================================================================
  // Provider models sidebar — keys kept stable for ProviderModelsSidebar,
  // recolored to the ink/paper/signal system.
  // ===========================================================================
  &__sidebar-overlay {
    position: fixed;
    inset: 0;
    background: rgba(20, 22, 27, 0.45);
    z-index: 40;
  }

  &__sidebar {
    position: fixed;
    top: 0;
    right: 0;
    bottom: 0;
    width: min(420px, 100vw);
    background: $card;
    border-left: 1px solid $line;
    box-shadow: -20px 0 40px -16px rgba(20, 22, 27, 0.28);
    z-index: 41;
    display: flex;
    flex-direction: column;
    animation: providers-sidebar-in 0.22s cubic-bezier(0.22, 0.72, 0.16, 1);
  }

  &__sidebar-header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    padding: 20px 20px 16px;
    border-bottom: 1px solid $line;
  }

  &__sidebar-title {
    font-family: $display;
    font-size: 1.0625rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;
  }

  &__sidebar-subtitle {
    margin-top: 3px;
    font-size: 0.75rem;
    color: $ink-3;
  }

  &__sidebar-body {
    flex: 1;
    overflow-y: auto;
    padding: 16px 20px 24px;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  &__sidebar-empty {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 8px;
    padding: 48px 12px;
    color: $ink-3;
    font-size: 0.8125rem;
    text-align: center;
  }

  &__model-row {
    border: 1px solid $line;
    border-radius: 12px;
    padding: 14px;
    background: $paper;
  }

  &__model-row-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
  }

  &__model-row-name {
    font-family: $display;
    font-weight: 700;
    font-size: 0.875rem;
    color: $ink;
  }

  &__model-row-tags {
    margin-top: 8px;
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  &__model-row-meta {
    margin-top: 10px;
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 8px 12px;

    >div {
      display: flex;
      flex-direction: column;
      gap: 2px;
      font-size: 0.8125rem;
      color: $ink;
    }
  }

  &__model-row-meta-label {
    @extend %micro;
    font-size: 0.625rem;
    color: $ink-3;
  }

  &__model-row-url {
    margin-top: 10px;
    padding-top: 10px;
    border-top: 1px dashed $line;
    font-family: $mono;
    font-size: 0.75rem;
    color: $ink-2;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
}

@keyframes providers-spin {
  to {
    transform: rotate(360deg);
  }
}

@keyframes providers-sidebar-in {
  from {
    transform: translateX(100%);
  }

  to {
    transform: translateX(0);
  }
}

@media (max-width: 768px) {
  .providers__header {
    padding: 20px 18px 16px;
    flex-direction: column;
    align-items: flex-start;
    gap: 10px;
  }

  .providers__toolbar {
    padding: 12px 18px;
  }

  .providers__grid {
    grid-template-columns: 1fr;
  }
}
