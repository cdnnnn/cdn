@use '../../styles/_variables' as *;

// ===========================================================================
// Sidebar — matches History / Reports / Comparison / New Evaluation design
// system: ink/paper palette, ultramarine signal accent, mono instrument
// labels, hover-lift.
//
// Font scaling: `.sidebar` sets a single base font-size. All descendant
// font-sizes are expressed in `em` (relative to that base), so bumping
// `.sidebar`'s font-size (e.g. on wide screens) scales the whole component
// proportionally from one place.
// ===========================================================================

$ink:      #14161B;
$ink-2:    #565B66;
$ink-3:    #8A909B;
$paper:    #F5F6F8;
$card:     #FFFFFF;
$line:     #E6E8EC;
$line-2:   #EEF0F3;
$signal:   #2B2BF5;
$signal-2: #1C1CC7;
$wash:     #ECEDFF;
$danger:   #DC2626;
$danger-wash: #FDECEC;

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);

// base font-size the sidebar's internal `em` scale is built on
$sidebar-base-font: 0.875rem;

%micro {
  font-family: $mono;
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

  // master scale control — every em-based font-size below responds to this
  font-size: $sidebar-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }

  &__logo {
    padding: 22px 24px;
    display: flex;
    align-items: center;
    gap: 11px;
    font-family: $display;
    font-size: 1.2143em; // 1.0625rem / 0.875rem
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
    background: $ink;
    font-size: 0.9286em; // 0.875rem / 0.875rem base → matches original 0.875rem abs size
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
    font-size: 0.7143em; // 0.625rem / 0.875rem
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
    font-size: 0.8571em; // 0.75rem / 0.875rem
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
    background: $ink;
    color: #fff;
    font-family: $display;
    font-size: 0.9286em; // 0.8125rem / 0.875rem
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
    font-size: 0.9286em; // 0.8125rem / 0.875rem
    font-weight: 700;
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__user-email {
    font-family: $mono;
    font-size: 0.7857em; // 0.6875rem / 0.875rem
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

.nav-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 14px;
  border-radius: 10px;
  font-size: 0.9643em; // 0.84375rem / 0.875rem
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

  &.active {
    color: $signal;
    background: $wash;
    font-weight: 700;
    box-shadow: inset 2.5px 0 0 $signal;

    svg { color: $signal; }
  }
}

@media (max-width: 768px) {
  .sidebar { display: none; }
}
