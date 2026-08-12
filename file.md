@use '../../styles/_variables' as *;

// ===========================================================================
// Sidebar — matches History / Reports / Comparison / New Evaluation design
// system: ink/paper palette, ultramarine signal accent, mono instrument
// labels, hover-lift.
//
// Neutrals resolve to theme CSS vars for dark mode. $solid is a FIXED
// near-black used for "always dark" chips (logo mark, avatar) — using
// themed $ink there would turn them near-white in dark mode.
//
// Active nav item: light mode keeps the original wash-tint look; dark mode
// overrides to a solid signal pill (low-opacity wash reads washed-out on a
// dark surface).
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
$danger:   #DC2626;
$danger-wash: var(--danger-wash);

$solid: #14161B;

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);

%micro {
  font-family: $mono;
  font-size: 0.6875rem;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

.sidebar {
  width: 256px;
  background: $card;
  border-right: 1px solid $line;
  display: flex;
  flex-direction: column;
  flex-shrink: 0;

  &__logo {
    padding: 22px 24px;
    display: flex;
    align-items: center;
    gap: 11px;
    font-family: $display;
    font-size: 1.0625rem;
    font-weight: 800;
    letter-spacing: -0.01em;
    color: $ink;
    border-bottom: 1px solid $line;
    text-decoration: none;
    cursor: pointer;
    transition: opacity 0.15s ease;

    &:hover { opacity: 0.82; }
  }

  &__mark {
    flex-shrink: 0;
    width: 30px;
    height: 30px;
    border-radius: 9px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #fff;
    background: $solid;
    font-size: 0.875rem;
    position: relative;
    overflow: hidden;

    &::after {
      content: '';
      position: absolute;
      inset: 0;
      background: linear-gradient(140deg, transparent 40%, rgba($signal, 0.9) 140%);
    }
  }

  &__nav {
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    padding: 14px 12px;
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  &__section {
    @extend %micro;
    font-size: 0.625rem;
    color: $ink-3;
    padding: 20px 14px 8px;
  }

  &__foot {
    flex-shrink: 0;
    padding: 16px;
    border-top: 1px solid $line;
  }

  &__theme-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 6px 4px 14px;
    font-size: 0.75rem;
    font-weight: 650;
    color: $ink-2;
  }

  &__user {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 9px 10px;
    border-radius: 12px;
    border: 1px solid transparent;
    transition: background 0.15s ease, border-color 0.15s ease;

    &:hover { background: $paper; border-color: $line; }
  }

  &__avatar {
    flex-shrink: 0;
    width: 36px;
    height: 36px;
    border-radius: 10px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: $solid;
    color: #fff;
    font-family: $display;
    font-size: 0.8125rem;
    font-weight: 700;
    position: relative;
    overflow: hidden;

    &::after {
      content: '';
      position: absolute;
      inset: 0;
      background: linear-gradient(140deg, transparent 40%, rgba($signal, 0.9) 140%);
    }
  }

  &__user-info {
    flex: 1;
    min-width: 0;
  }

  &__user-name {
    font-size: 0.8125rem;
    font-weight: 700;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__user-email {
    font-family: $mono;
    font-size: 0.6875rem;
    color: $ink-3;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__logout {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 28px;
    height: 28px;
    border-radius: 8px;
    border: 1px solid transparent;
    background: transparent;
    color: $ink-3;
    cursor: pointer;
    transition: background 0.15s ease, color 0.15s ease, border-color 0.15s ease;

    &:hover {
      background: $danger-wash;
      border-color: rgba($danger, 0.2);
      color: $danger;
    }
  }
}

// ---- nav item ---------------------------------------------------------
.nav-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 14px;
  border-radius: 10px;
  font-size: 0.84375rem;
  font-weight: 550;
  color: $ink-2;
  cursor: pointer;
  transition: color 0.15s ease, background 0.15s ease;
  border: none;
  background: none;
  width: 100%;
  text-align: left;
  text-decoration: none;

  svg { flex-shrink: 0; color: $ink-3; transition: color 0.15s ease; }

  &:hover {
    color: $ink;
    background: $paper;
    svg { color: $ink-2; }
  }

  // Light mode (default): original wash-tint active state.
  &.active {
    color: $signal;
    background: $wash;
    font-weight: 700;
    box-shadow: inset 2.5px 0 0 $signal;

    svg { color: $signal; }
  }
}

// Dark mode override: solid signal pill with white text/icon — a
// low-opacity wash tint reads washed-out against a dark surface.
:global([data-theme='dark']) .nav-item.active {
  color: #fff;
  background: $signal;
  box-shadow: 0 4px 12px -4px rgba($signal, 0.55);

  svg { color: #fff; }

  &:hover { background: $signal-2; }
}

@media (max-width: 768px) {
  .sidebar { display: none; }
}
