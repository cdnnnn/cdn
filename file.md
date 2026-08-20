@use '../../styles/_variables' as *;

// Font scaling: `.drawer` sets a single base font-size. All descendant
// font-sizes are expressed in `em` (relative to that base), so bumping
// `.drawer`'s font-size on wide screens scales the whole drawer
// proportionally from one place — same convention as Sidebar, Providers,
// and Model Catalog.

// base font-size the drawer's internal `em` scale is built on
$drawer-base-font: 13px;

.drawer-overlay {
  position: fixed; top: 0; left: 0; right: 0; bottom: $footer-height;
  background: rgba(17, 24, 39, .4); z-index: 100;
  display: flex; justify-content: flex-end;
}
.drawer {
  width: 420px; max-width: 100%; height: calc(100% - 30px); background: $surface; box-shadow: $shadow-4;
  display: flex; flex-direction: column; animation: drawerIn .25s ease both;

  // master scale control — every em-based font-size below responds to this
  font-size: $drawer-base-font;

  @media (min-width: 1800px) {
    font-size: 16px;
  }
}
@keyframes drawerIn { from { transform: translateX(24px); opacity: 0; } to { transform: translateX(0); opacity: 1; } }
.drawer__hdr {
  display: flex; justify-content: space-between; align-items: center; padding: 20px 24px; border-bottom: 1px solid $border-light;
  h2 { font-size: 1.3846em; font-weight: 700; } // 18px / 13px
}
.drawer__close { background: none; border: none; cursor: pointer; color: $text-muted; }
.drawer__body { flex: 1; overflow-y: auto; padding: 24px; }
.drawer__foot { display: flex; justify-content: flex-end; gap: 10px; padding: 16px 24px; border-top: 1px solid $border-light; }

// ---- category custom dropdown ------------------------------------------------
.combo {
  position: relative;
}
.combo-trigger {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  width: 100%;
  padding: 10px 12px;
  border: 1.5px solid $border;
  border-radius: 10px;
  background: $surface;
  cursor: pointer;
  text-align: left;
  transition: border-color 0.15s ease, box-shadow 0.15s ease, background 0.15s ease;

  &:hover:not(:disabled) { border-color: $indigo; }
  &:disabled { cursor: not-allowed; opacity: 0.6; }

  &--open {
    border-color: $indigo;
    box-shadow: 0 0 0 3px $indigo-pale;
  }
}
.combo-value {
  display: flex;
  align-items: center;
  min-width: 0;
}
.combo-value-label {
  font-size: 1em; // 13px / 13px (base)
  font-weight: 650;
  color: $text-primary;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.combo-placeholder {
  font-size: 1em; // 13px / 13px (base)
  color: $text-muted;
}
.combo-caret {
  flex-shrink: 0;
  color: $text-muted;
  transition: transform 0.18s ease;

  &--open { transform: rotate(180deg); color: $indigo; }
}
.combo-panel {
  position: absolute;
  z-index: 20;
  top: calc(100% + 6px);
  left: 0;
  right: 0;
  max-height: 280px;
  overflow-y: auto;
  padding: 6px;
  background: $surface;
  border: 1px solid $border;
  border-radius: 12px;
  box-shadow: $shadow-3;
  animation: combo-panel-in 0.14s ease both;
}
@keyframes combo-panel-in {
  from { opacity: 0; transform: translateY(-4px); }
  to { opacity: 1; transform: translateY(0); }
}
.combo-empty {
  padding: 14px 10px;
  font-size: 0.9615em; // 12.5px / 13px
  color: $text-muted;
  text-align: center;
}
.combo-option {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  width: 100%;
  padding: 9px 10px;
  border: none;
  border-radius: 8px;
  background: none;
  cursor: pointer;
  text-align: left;
  transition: background 0.12s ease;

  &:hover { background: $surface-alt; }

  &--selected {
    background: $indigo-pale;

    &:hover { background: $indigo-pale; }
  }

  & + & { margin-top: 1px; }
}
.combo-option-check {
  flex-shrink: 0;
  width: 14px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: $indigo;
}
.combo-option-text {
  display: flex;
  flex-direction: column;
  gap: 2px;
  min-width: 0;
}
.combo-option-label {
  font-size: 1em; // 13px / 13px (base)
  font-weight: 650;
  color: $text-primary;
}
.combo-option-desc {
  font-size: 0.8846em; // 11.5px / 13px
  line-height: 1.4;
  color: $text-secondary;
}

// ---- hint / error text -------------------------------------------------------
.field-hint {
  margin-top: 6px;
  font-size: 0.9231em; // 12px / 13px
  line-height: 1.45;
  color: $text-secondary;
}
.field-hint--error {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-top: 8px;
  font-size: 0.9231em; // 12px / 13px
  color: #DC2626;
}
.locked-tag {
  margin-left: 6px;
  padding: 1px 7px;
  border-radius: 999px;
  background: $emerald-pale;
  color: $emerald-dark;
  font-size: 0.8077em; // 10.5px / 13px
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.03em;
  vertical-align: middle;
}

// ---- optional-discovery panel -------------------------------------------------
.discover-panel {
  display: flex;
  gap: 12px;
  margin-bottom: 18px;
  padding: 16px;
  border-radius: 14px;
  background: linear-gradient(165deg, $indigo-pale 0%, $surface-alt 65%);
  border: 1px solid rgba($indigo, 0.16);
  position: relative;
  overflow: hidden;
}
.discover-panel-icon {
  flex-shrink: 0;
  width: 32px;
  height: 32px;
  border-radius: 9px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: $indigo;
  color: #fff;
  box-shadow: 0 4px 10px -3px rgba($indigo, 0.5);
}
.discover-panel-body {
  min-width: 0;
  flex: 1;
}
.discover-panel-title {
  font-size: 1.0385em; // 13.5px / 13px
  font-weight: 750;
  color: $text-primary;
  margin-bottom: 3px;
}
.discover-panel-text {
  font-size: 0.9231em; // 12px / 13px
  line-height: 1.5;
  color: $text-secondary;
  margin: 0 0 12px;
}
.discover-panel-actions {
  display: flex;
  align-items: center;
  gap: 14px;
  flex-wrap: wrap;
}
.discover-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 14px;
  border-radius: 9px;
  border: 1px solid $indigo;
  background: $indigo;
  color: #fff;
  font-size: 0.9615em; // 12.5px / 13px
  font-weight: 700;
  cursor: pointer;
  box-shadow: 0 2px 6px -2px rgba($indigo, 0.5);
  transition: background 0.15s ease, border-color 0.15s ease, transform 0.12s ease, box-shadow 0.15s ease;

  &:hover:not(:disabled) { background: $indigo-dark; border-color: $indigo-dark; transform: translateY(-1px); box-shadow: 0 4px 10px -2px rgba($indigo, 0.55); }
  &:disabled { opacity: 0.5; cursor: not-allowed; transform: none; box-shadow: none; }
}
.discover-link {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 0;
  border: none;
  background: none;
  font-size: 0.9231em; // 12px / 13px
  font-weight: 600;
  color: $text-secondary;
  cursor: pointer;
  transition: color 0.15s ease;

  &:hover { color: $indigo; }
}
.spin { animation: add-custom-model-spin 0.8s linear infinite; }
@keyframes add-custom-model-spin { to { transform: rotate(360deg); } }

// ---- discovery success banner --------------------------------------------------
.success-banner {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 16px;
  padding: 10px 13px;
  border-radius: 10px;
  background: $emerald-pale;
  border: 1px solid rgba($emerald, 0.25);
  color: $emerald-dark;
  font-size: 0.9615em; // 12.5px / 13px
  font-weight: 650;

  svg { flex-shrink: 0; }
}

// ---- discovered model picker -------------------------------------------------
.discovered-list {
  display: flex;
  flex-direction: column;
  gap: 6px;
  max-height: 220px;
  overflow-y: auto;
  padding: 2px;
}
.discovered-row {
  position: relative;
  display: flex;
  flex-direction: column;
  gap: 4px;
  width: 100%;
  padding: 10px 34px 10px 12px;
  border: 1.5px solid $border;
  border-radius: 9px;
  background: $surface;
  cursor: pointer;
  text-align: left;
  transition: border-color 0.14s ease, background 0.14s ease;

  &:hover { border-color: $indigo; background: $indigo-pale; }

  &--selected {
    border-color: $indigo;
    background: $indigo-pale;
  }
}
.discovered-row-main {
  display: flex;
  align-items: baseline;
  gap: 8px;
  min-width: 0;
}
.discovered-row-name {
  font-weight: 700;
  font-size: 1em; // 13px / 13px (base)
  color: $text-primary;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.discovered-row-id {
  flex-shrink: 0;
  font-family: monospace;
  font-size: 0.8462em; // 11px / 13px
  color: $text-muted;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.discovered-row-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  font-size: 0.8462em; // 11px / 13px
  color: $text-secondary;
}
.already-added-badge {
  padding: 1px 7px;
  border-radius: 999px;
  background: rgba(245, 158, 11, 0.16);
  color: #B45309;
  font-weight: 700;
}
.discovered-row-check {
  position: absolute;
  top: 50%;
  right: 10px;
  transform: translateY(-50%);
  color: $indigo;
}

// ---- reset / mode-switch link -------------------------------------------------
.reset-link {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  margin: -6px 0 16px;
  padding: 0;
  border: none;
  background: none;
  font-size: 0.9231em; // 12px / 13px
  font-weight: 650;
  color: $indigo;
  cursor: pointer;

  &:hover { text-decoration: underline; }
}































@use '../../styles/_variables' as *;

// Font scaling: `.drawer` sets a single base font-size. All descendant
// font-sizes are expressed in `em` (relative to that base), so bumping
// `.drawer`'s font-size on wide screens scales the whole drawer
// proportionally from one place — same convention as Sidebar, Providers,
// and Model Catalog.

// base font-size the drawer's internal `em` scale is built on
$drawer-base-font: 13px;

.drawer-overlay {
  position: fixed; top: 0; left: 0; right: 0; bottom: $footer-height;
  background: rgba(17, 24, 39, .4); z-index: 100;
  display: flex; justify-content: flex-end;
}
.drawer {
  width: 420px; max-width: 100%; height: calc(100% - 30px); background: $surface; box-shadow: $shadow-4;
  display: flex; flex-direction: column; animation: drawerIn .25s ease both;

  // master scale control — every em-based font-size below responds to this
  font-size: $drawer-base-font;

  @media (min-width: 1800px) {
    font-size: 16px;
  }
}
@keyframes drawerIn { from { transform: translateX(24px); opacity: 0; } to { transform: translateX(0); opacity: 1; } }
.drawer__hdr {
  display: flex; justify-content: space-between; align-items: center; padding: 20px 24px; border-bottom: 1px solid $border-light;
  h2 { font-size: 1.3846em; font-weight: 700; } // 18px / 13px
}
.drawer__close { background: none; border: none; cursor: pointer; color: $text-muted; }
.drawer__body { flex: 1; overflow-y: auto; padding: 24px; }
.drawer__foot { display: flex; justify-content: flex-end; gap: 10px; padding: 16px 24px; border-top: 1px solid $border-light; }
