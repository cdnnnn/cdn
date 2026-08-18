@use '../../styles/_variables' as *;

// ===========================================================================
// Dashboard — matches the Evaluation Run Console design system:
// ink/paper palette, ultramarine signal accent, mono instrument labels,
// hover-lift cards with a subtle inset ring on emphasis.
//
// Font scaling: `.dashboard` sets a single base font-size. All descendant
// font-sizes are expressed in `em` (relative to that base), so bumping
// `.dashboard`'s font-size (e.g. on wide screens) scales the whole component
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
$ok:       #0FA968;
$ok-wash:  #E7F7EF;
$amber:    #E08600;
$amber-wash: #FDF3E3;
$danger:   #DC2626;
$danger-wash: #FDECEC;

$mono:    $font-mono;
$sans:    $font-body;
$display: $font-display;

$soft: 0 1px 2px rgba(20, 22, 27, 0.05);
$lift: 0 14px 30px -14px rgba(20, 22, 27, 0.22);

// base font-size the dashboard's internal `em` scale is built on
$dashboard-base-font: 0.875rem;

%micro {
  font-family: $mono;
  font-size: 0.6875rem;
  font-weight: 700;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}

// ===========================================================================
// Root scaling wrapper
// ===========================================================================
.dashboard {
  // master scale control — every em-based font-size below responds to this
  font-size: $dashboard-base-font;

  @media (min-width: 1800px) {
    font-size: 1rem;
  }
}

// ===========================================================================
// Header
// ===========================================================================
.db {
  &__header {
    flex-shrink: 0;
    display: flex;
    align-items: flex-end;
    justify-content: space-between;
    gap: 1rem;
    padding: 24px 32px 20px;
    margin-bottom: 24px;
    border-bottom: 1px solid $line;
    background: $card;

    h1 {
      font-family: $display;
      font-size: 1.7143em; // 1.5rem / 0.875rem
      font-weight: 800;
      letter-spacing: -0.02em;
      color: $ink;
      line-height: 1.2;
    }
  }

  &__eyebrow {
    @extend %micro;
    font-size: 0.7857em; // 0.6875rem / 0.875rem
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

  &__sub {
    margin-top: 4px;
    font-size: 0.9643em; // 0.84375rem / 0.875rem
    color: $ink-2;
  }

  &__meta {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    gap: 7px;
    padding: 7px 13px;
    border-radius: 999px;
    border: 1px solid $line;
    background: $paper;
    font-family: $mono;
    font-size: 0.8214em; // 0.71875rem / 0.875rem
    font-weight: 700;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    color: $ink-2;
    white-space: nowrap;
    margin-bottom: 3px;
  }
}

// ===========================================================================
// Body
// ===========================================================================
.db-body {
  padding: 0 32px 32px;
}

// ---- stat cards -----------------------------------------------------------
.d-stats {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 14px;
  margin-bottom: 18px;
}

.d-stat {
  position: relative;
  background: $card;
  border: 1px solid $line;
  border-radius: 16px;
  padding: 20px 22px;
  transition: border-color 0.16s ease, box-shadow 0.16s ease, transform 0.16s ease;

  &:hover {
    border-color: $ink-3;
    box-shadow: $lift;
    transform: translateY(-2px);
  }
}

.d-stat-top {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

.d-stat-label {
  @extend %micro;
  font-size: 0.7143em; // 0.625rem / 0.875rem
  color: $ink-3;
}

.d-stat-val {
  font-family: $mono;
  font-size: 2.2857em; // 2rem / 0.875rem
  font-weight: 700;
  letter-spacing: -0.02em;
  line-height: 1;
  margin-top: 10px;
  color: $ink;
}

.d-stat-change {
  font-size: 0.8214em; // 0.71875rem / 0.875rem
  color: $ok;
  font-weight: 600;
  margin-top: 8px;
  display: flex;
  align-items: center;
  gap: 5px;
}

// ---- main grid --------------------------------------------------------------
.dash {
  &__grid {
    display: grid;
    grid-template-columns: 1.2fr 1fr;
    gap: 14px;
    margin-bottom: 14px;
  }

  &__panel {
    background: $card;
    border: 1px solid $line;
    border-radius: 16px;
    overflow: hidden;
  }

  &__panel-hdr {
    padding: 17px 20px;
    border-bottom: 1px solid $line-2;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  &__panel-title {
    display: flex;
    align-items: center;
    gap: 8px;
    font-family: $display;
    font-weight: 700;
    font-size: 1em; // 0.875rem / 0.875rem
    color: $ink;

    svg { color: $signal; }
  }

  &__panel-sub {
    font-size: 0.8571em; // 0.75rem / 0.875rem
    color: $ink-3;
    margin-top: 2px;
  }

  &__link {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-family: $sans;
    font-size: 0.8929em; // 0.78125rem / 0.875rem
    font-weight: 650;
    color: $ink-2;
    background: transparent;
    border: 0;
    cursor: pointer;
    padding: 0;
    transition: color 0.15s ease;

    &:hover { color: $signal; }
  }

  &__run-row {
    padding: 14px 20px;
    border-bottom: 1px solid $line-2;
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 14px;
    cursor: pointer;
    transition: background 0.15s ease;

    &:last-child { border-bottom: 0; }
    &:hover { background: $paper; }
  }

  &__run-main {
    display: flex;
    align-items: center;
    gap: 13px;
    min-width: 0;
  }

  &__run-name {
    font-family: $display;
    font-weight: 700;
    font-size: 0.9643em; // 0.84375rem / 0.875rem
    color: $ink;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  &__run-meta {
    font-size: 0.8214em; // 0.71875rem / 0.875rem
    color: $ink-3;
    margin-top: 2px;
  }

  &__spinner {
    flex-shrink: 0;
    width: 40px;
    height: 40px;
    border-radius: 11px;
    background: $wash;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  &__fail-icon {
    flex-shrink: 0;
    width: 40px;
    height: 40px;
    border-radius: 11px;
    background: $danger-wash;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  &__empty {
    padding: 30px 20px;
    text-align: center;
    color: $ink-3;
    font-size: 0.9286em; // 0.8125rem / 0.875rem
  }

  &__legend {
    display: flex;
    justify-content: center;
    gap: 18px;
    margin-top: 6px;
    flex-wrap: wrap;
  }

  &__legend span {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 0.8214em; // 0.71875rem / 0.875rem
    font-weight: 600;
    color: $ink-2;
  }

  &__dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    display: inline-block;
  }

  &__actions {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 12px;
  }
}

// ---- status pill (mono, colored per run status) ----------------------------
.status-pill {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 4px 10px 4px 8px;
  border-radius: 999px;
  font-family: $mono;
  font-size: 0.7143em; // 0.625rem / 0.875rem
  font-weight: 700;
  letter-spacing: 0.06em;
  text-transform: uppercase;

  &::before {
    content: '';
    width: 5px;
    height: 5px;
    border-radius: 50%;
  }

  &--completed {
    color: $ok;
    background: $ok-wash;
    &::before { background: $ok; }
  }

  &--failed {
    color: $danger;
    background: $danger-wash;
    &::before { background: $danger; }
  }

  &--running {
    color: $signal;
    background: $wash;
    &::before { background: $signal; animation: db-pulse 1.1s ease-in-out infinite; }
  }
}

// ---- quick action tiles -----------------------------------------------------
.qa-btn {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  padding: 22px 18px;
  border: 1.5px solid $line;
  border-radius: 16px;
  background: $card;
  cursor: pointer;
  text-align: center;
  transition: border-color 0.18s ease, box-shadow 0.18s ease, transform 0.18s ease;

  &:hover {
    border-color: $ink-3;
    box-shadow: $lift;
    transform: translateY(-3px);
  }
}

.qa-btn__icon {
  width: 46px;
  height: 46px;
  border-radius: 13px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  background: $ink;
  position: relative;
  overflow: hidden;

  &::after {
    content: '';
    position: absolute;
    inset: 0;
    background: linear-gradient(140deg, transparent 45%, rgba(255, 255, 255, 0.18) 140%);
  }

  svg { position: relative; z-index: 1; }
}

.qa-btn--ind .qa-btn__icon { background: $signal; }
.qa-btn--em  .qa-btn__icon { background: $ok; }
.qa-btn--amb .qa-btn__icon { background: $amber; }
.qa-btn--sky .qa-btn__icon { background: #0369A1; }

.qa-btn__label {
  font-size: 0.9643em; // 0.84375rem / 0.875rem
  font-weight: 700;
  font-family: $display;
  color: $ink;
}

.qa-btn__desc {
  font-size: 0.8214em; // 0.71875rem / 0.875rem
  color: $ink-3;
  margin-top: 2px;
}

// ---- panel section title used for Quick Actions card -----------------------
.section-title {
  display: flex;
  align-items: center;
  gap: 8px;
  font-family: $display;
  font-weight: 700;
  font-size: 1em; // 0.875rem / 0.875rem
  color: $ink;
  margin-bottom: 4px;

  svg { color: $signal; }
}

.section-sub {
  font-size: 0.8571em; // 0.75rem / 0.875rem
  color: $ink-3;
  margin-bottom: 18px;
}

@keyframes db-pulse {
  0%, 100% { box-shadow: 0 0 0 0 rgba(43, 43, 245, 0.5); }
  50% { box-shadow: 0 0 0 4px rgba(43, 43, 245, 0); }
}

@media (max-width: 768px) {
  .d-stats { grid-template-columns: repeat(2, 1fr); }
  .dash__grid { grid-template-columns: 1fr; }
  .dash__actions { grid-template-columns: 1fr 1fr; }
  .db__header { padding: 20px 18px 16px; flex-direction: column; align-items: flex-start; gap: 10px; }
  .db-body { padding: 0 18px 24px; }
}
