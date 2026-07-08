@use '../../../styles/tokens' as t;

.db-analytics-results-panel {
  background: t.$surface-2;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  flex: 1;
  min-height: 0;
}

.db-analytics-results-panel__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 53px;
  padding: 0 16px;
  border-bottom: 1px solid t.$border-strong;
  background: t.$surface-1;
  flex-shrink: 0;
}

.db-analytics-results-panel__header-text {
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 1px;
  min-width: 0;
}

.db-analytics-results-panel__title {
  font-size: 14px;
  font-weight: 500;
  margin: 0;
  color: t.$text-primary;
  line-height: 1.3;
}

.db-analytics-results-panel__subtitle {
  font-size: 11px;
  color: t.$text-muted;
  margin: 0;
  line-height: 1.3;
}

.db-analytics-results-panel__body {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
}

.db-analytics-results-panel__error {
  margin: 12px;
  font-size: 12px;
  color: #791f1f;
  background: #fbe9e9;
  padding: 8px 11px;
  border-radius: t.$radius;
}

.db-analytics-chart-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
  padding: 12px;
}

.db-analytics-chart-card {
  border: 1px solid t.$border-strong;
  border-radius: t.$radius-lg;
  padding: 14px 16px 16px;
  min-width: 0;
  background: t.$surface-2;
  box-shadow: 0 1px 3px rgba(11, 11, 11, 0.05);
  transition: border-color 0.18s ease, box-shadow 0.18s ease, transform 0.18s ease;

  &:hover {
    border-color: t.$border-stronger;
    box-shadow: 0 6px 18px rgba(11, 11, 11, 0.08);
    transform: translateY(-1px);
  }
}

.db-analytics-chart-card--span-2 {
  grid-column: 1 / -1;
}

.db-analytics-chart-card__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 14px;
  padding-bottom: 11px;
  border-bottom: 0.5px solid t.$border;
}

.db-analytics-chart-card__heading {
  display: flex;
  align-items: center;
  gap: 9px;
  min-width: 0;
}

.db-analytics-chart-card__icon {
  width: 28px;
  height: 28px;
  border-radius: 8px;
  background: t.$gradient-primary;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  box-shadow: 0 2px 6px rgba(79, 70, 229, 0.25);
}

.db-analytics-chart-card__title {
  font-size: 13.5px;
  font-weight: 600;
  color: t.$text-primary;
  margin: 0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-chart-card__badge {
  font-size: 10px;
  font-weight: 500;
  color: t.$text-secondary;
  background: t.$surface-0;
  border: 0.5px solid t.$border;
  padding: 2px 7px;
  border-radius: 999px;
  text-transform: capitalize;
  flex-shrink: 0;
}

.db-analytics-chart-legend {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  row-gap: 8px;
  margin-top: 12px;
  padding-top: 12px;
  border-top: 0.5px solid t.$border;
}

.db-analytics-chart-legend__divider {
  width: 1px;
  align-self: stretch;
  min-height: 16px;
  background: t.$border-strong;
  margin: 0 12px;
  flex-shrink: 0;
}

.db-analytics-chart-legend__item {
  display: flex;
  align-items: center;
  gap: 7px;
  font-size: 12.5px;
  color: t.$text-secondary;
  white-space: nowrap;
}

.db-analytics-chart-legend__swatch {
  width: 9px;
  height: 9px;
  border-radius: 3px;
  flex-shrink: 0;
}

.db-analytics-chart-legend__name {
  max-width: 140px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  color: t.$text-primary;
  font-weight: 500;
}

.db-analytics-chart-legend__pct {
  font-weight: 700;
  color: t.$text-primary;
  font-variant-numeric: tabular-nums;
  flex-shrink: 0;
}

.db-analytics-empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 48px 16px;
  color: t.$text-muted;
  text-align: center;
  min-height: 100%;
  box-sizing: border-box;

  p {
    font-size: 13px;
    margin: 0;
    max-width: 220px;
  }
}

.db-analytics-empty-state__icon--generating {
  color: t.$accent;
  animation: db-analytics-empty-state-draw 1.6s ease-in-out infinite;
}

@keyframes db-analytics-empty-state-draw {
  0% {
    transform: scale(0.9) rotate(-4deg);
    opacity: 0.55;
  }
  50% {
    transform: scale(1.08) rotate(3deg);
    opacity: 1;
  }
  100% {
    transform: scale(0.9) rotate(-4deg);
    opacity: 0.55;
  }
}

.db-analytics-kpi-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
  gap: 12px;
}

.db-analytics-kpi-card {
  position: relative;
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 16px 16px 14px;
  border-radius: t.$radius-lg;
  background: t.$surface-0;
  border: 1px solid t.$border;
  overflow: hidden;
  transition: transform 0.15s ease, box-shadow 0.15s ease, border-color 0.15s ease;

  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 16px rgba(11, 11, 11, 0.07);
    border-color: t.$border-strong;
  }
}

.db-analytics-kpi-card__label {
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.01em;
  text-transform: uppercase;
  color: t.$text-secondary;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.db-analytics-kpi-card__value {
  font-size: 25px;
  font-weight: 700;
  letter-spacing: -0.01em;
  line-height: 1.15;
  color: t.$text-primary;
}

.db-analytics-table-wrap {
  overflow-x: auto;
  max-height: 260px;
  overflow-y: auto;
}

.db-analytics-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px;

  th {
    text-align: left;
    font-weight: 500;
    color: t.$text-secondary;
    padding: 6px 10px;
    border-bottom: 1px solid t.$border-strong;
    background: t.$surface-1;
    position: sticky;
    top: 0;
    white-space: nowrap;
  }

  td {
    padding: 6px 10px;
    border-bottom: 0.5px solid t.$border;
    color: t.$text-primary;
    white-space: nowrap;
  }

  tr:last-child td {
    border-bottom: none;
  }
}

.db-analytics-chart-card__no-data {
  padding: 24px 12px;
  text-align: center;
  font-size: 12px;
  color: t.$text-muted;
}

.db-analytics-chart-scroll {
  overflow-x: auto;
  overflow-y: hidden;
  margin: 0 -4px;
  padding: 0 4px 8px;

  // Firefox: request a thin, always-visible scrollbar with explicit colors.
  scrollbar-width: thin;
  scrollbar-color: t.$border-stronger transparent;

  // Chrome/Safari/Edge: styling the pseudo-elements alone isn't enough —
  // by default WebKit only paints "overlay" scrollbars on hover/scroll and
  // they disappear immediately after, which is what looked broken. Forcing
  // classic (non-overlay) scrollbars keeps the thumb always rendered.
  &::-webkit-scrollbar {
    height: 6px;
    -webkit-appearance: none;
  }

  &::-webkit-scrollbar-track {
    background: t.$surface-0;
    border-radius: 999px;
  }

  &::-webkit-scrollbar-thumb {
    background: t.$border-stronger;
    border-radius: 999px;
    border: 1px solid t.$surface-0;
  }

  &::-webkit-scrollbar-thumb:hover {
    background: t.$text-muted;
  }
}
