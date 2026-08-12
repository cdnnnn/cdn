@use '../../styles/_variables' as *;

// ===========================================================================
// Providers — matches the History/Reports/Comparison/Sidebar design system:
// ink/paper palette, ultramarine signal accent, mono instrument labels,
// hover-lift cards. Neutrals resolve to theme CSS vars for dark mode.
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
$ink-wash: var(--ink-wash);

$mono:    $font-mono;
$sans:    $font-body;
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

// ---- header -----------------------------------------------------------
.providers__header {
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
    font-size: 1.5rem;
    font-weight: 800;
    letter-spacing: -0.02em;
    color: $ink;
    line-height: 1.2;
  }
}

.providers__header-eyebrow {
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

.providers__header-sub {
  margin-top: 4px;
  font-size: 0.84375rem;
  color: $ink-2;
}

.providers__header-meta {
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
.providers__toolbar {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  flex-wrap: wrap;
  padding: 16px 32px;
  border-bottom: 1px solid $line;
  background: $card;
}

.providers__search {
  position: relative;
  flex: 1;
  min-width: 220px;
  max-width: 320px;

  svg {
    position: absolute;
    top: 50%;
    left: 12px;
    transform: translateY(-50%);
    color: $ink-3;
    pointer-events: none;
  }

  input {
    width: 100%;
    border: 1.5px solid $line;
    border-radius: 10px;
    padding: 9px 12px 9px 36px;
    font-size: 0.8125rem;
    font-family: $sans;
    color: $ink;
    background: $paper;
    transition: border-color 0.15s ease, box-shadow 0.15s ease, background 0.15s ease;

    &::placeholder { color: $ink-3; }
    &:focus {
      outline: none;
      border-color: $signal;
      background: $card;
      box-shadow: 0 0 0 3px $wash;
    }
  }
}

.providers__toolbar-right {
  display: flex;
  align-items: center;
  gap: 14px;
  flex-wrap: wrap;
}

.providers__filter-group {
  display: flex;
  align-items: center;
  gap: 6px;
}

.providers__toolbar-label {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  @extend %micro;
  font-size: 0.625rem;
  color: $ink-3;
  padding-right: 4px;
}

.providers__filter-pill {
  padding: 6px 12px;
  border: 1px solid $line;
  border-radius: 999px;
  background: $card;
  color: $ink-2;
  font-size: 0.75rem;
  font-weight: 650;
  cursor: pointer;
  transition: all 0.14s ease;

  &:hover { border-color: $ink-3; color: $ink; }

  &--on {
    border-color: $signal;
    background: $signal;
    color: #fff;
  }
}

.providers__toolbar-divider {
  width: 1px;
  height: 20px;
  background: $line;
}

.providers__add-btn {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 9px 15px;
  border-radius: 10px;
  border: 1px solid $signal;
  background: $signal;
  color: #fff;
  font-family: $sans;
  font-size: 0.8125rem;
  font-weight: 700;
  cursor: pointer;
  white-space: nowrap;
  transition: background 0.15s ease, border-color 0.15s ease, transform 0.12s ease;

  &:hover { background: $signal-2; border-color: $signal-2; transform: translateY(-1px); }
}

// ---- grid -----------------------------------------------------------------
.providers__grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(380px, 1fr));
  gap: 12px;
}

.providers__empty {
  grid-column: 1 / -1;
  padding: 40px;
  text-align: center;
  color: $ink-3;
  font-size: 0.8125rem;
}

// ---- provider card ----------------------------------------------------
.providers__card {
  display: flex;
  flex-direction: column;
  padding: 18px;
  background: $card;
  border: 1.5px solid $line;
  border-radius: 16px;
  box-shadow: $soft;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, transform 0.15s ease;

  &:hover { border-color: $ink-3; box-shadow: $lift; transform: translateY(-2px); }
}

.providers__card-hdr {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 12px;
}

.providers__card-id {
  display: flex;
  align-items: center;
  gap: 12px;
  min-width: 0;
}

.providers__icon {
  flex-shrink: 0;
  width: 40px;
  height: 40px;
  border-radius: 11px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $paper;
  border: 1px solid $line;
  color: $ink;
  font-family: $display;
  font-weight: 800;
  font-size: 1rem;
  overflow: hidden;

  img { width: 100%; height: 100%; object-fit: contain; }
}

.providers__name {
  font-family: $display;
  font-size: 0.9375rem;
  font-weight: 700;
  color: $ink;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.providers__count {
  font-family: $mono;
  font-size: 0.71875rem;
  color: $ink-3;
  margin-top: 2px;
}

.providers__card-top-actions {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 6px;
}

.providers__icon-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  border-radius: 8px;
  border: 1px solid $line;
  background: $card;
  color: $ink-2;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  &:hover { border-color: $signal; color: $signal; background: $wash; }
}

.providers__badge-connected {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 9px 4px 7px;
  border-radius: 999px;
  font-family: $mono;
  font-size: 0.625rem;
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  color: $ok;
  background: $ok-wash;
  white-space: nowrap;
}

.providers__badge-idle {
  display: inline-flex;
  align-items: center;
  padding: 4px 9px;
  border-radius: 999px;
  font-family: $mono;
  font-size: 0.625rem;
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  color: $ink-3;
  background: $ink-wash;
  white-space: nowrap;
}

.providers__desc {
  font-size: 0.8125rem;
  color: $ink-2;
  line-height: 1.55;
  margin-bottom: 14px;
  flex: 1;
}

// ---- key entry form -------------------------------------------------------
.providers__key-form {
  margin-top: auto;
  padding-top: 4px;
}

.providers__key-input {
  width: 100%;
  border: 1.5px solid $line;
  border-radius: 9px;
  padding: 9px 11px;
  font-size: 0.8125rem;
  font-family: $mono;
  color: $ink;
  background: $paper;
  margin-bottom: 8px;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;

  &::placeholder { color: $ink-3; font-family: $sans; }
  &:focus { outline: none; border-color: $signal; box-shadow: 0 0 0 3px $wash; }
}

.providers__key-actions {
  display: flex;
  gap: 8px;
}

// ---- foot actions -----------------------------------------------------
.providers__foot-actions {
  display: flex;
  flex-wrap: nowrap;
  align-items: center;
  gap: 6px;
  margin-top: auto;
  padding-top: 6px;
}

.providers__foot-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 5px;
  flex: 1 1 0;
  min-width: 0;
  padding: 8px 8px;
  border-radius: 8px;
  border: 1px solid $line;
  background: $card;
  color: $ink-2;
  font-family: $sans;
  font-size: 0.71875rem;
  font-weight: 650;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  cursor: pointer;
  transition: border-color 0.15s ease, color 0.15s ease, background 0.15s ease;

  svg { flex-shrink: 0; }

  &:disabled { opacity: 0.5; cursor: not-allowed; }

  &--primary {
    border-color: $signal;
    background: $signal;
    color: #fff;
    &:hover:not(:disabled) { background: $signal-2; border-color: $signal-2; }
  }

  &--accent {
    border-color: $signal;
    color: $signal;
    background: $wash;
    &:hover:not(:disabled) { background: rgba(43, 43, 245, 0.14); }
  }

  &--ghost {
    &:hover:not(:disabled) { border-color: $ink-3; color: $ink; background: $paper; }
  }

  &--danger {
    color: $danger;
    border-color: rgba(220, 38, 38, 0.25);
    &:hover:not(:disabled) { background: $danger-wash; }
  }
}

.providers__spin {
  animation: providers-spin 0.8s linear infinite;
}

@keyframes providers-spin {
  to { transform: rotate(360deg); }
}

@media (max-width: 640px) {
  .providers__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .providers__toolbar { padding: 14px 18px; }
  .providers__grid { grid-template-columns: 1fr; }

  .providers__foot-btn {
    padding: 8px 6px;
    font-size: 0;
    gap: 0;

    svg { font-size: initial; }
  }
}
