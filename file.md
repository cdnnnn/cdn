//Footer.module.scss
@use '../../styles/_variables' as *;

.footer {
  position: fixed;
  left: 0;
  right: 0;
  bottom: 0;
  height: $footer-height;
  z-index: 40;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 0 24px;
  color: $text-muted;
  font-size: 13px;
  border-top: 1px solid $border;
  background: $surface;
}

.footer__version {
  font-family: $font-mono;
  font-weight: 600;
  flex-shrink: 0;
}

.footer__copyright {
  text-align: right;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

// Offset past the sidebar so it doesn't sit underneath it, matching the
// sidebar's own mobile breakpoint (hidden below 768px — see Sidebar.module.scss).
.footer--app {
  left: $sidebar-width;

  @media (max-width: 768px) {
    left: 0;
  }
}
















//_variables.scss
// Design tokens — ported 1:1 from the original theme object (T)
$bg: #F7F8FC;
$surface: #FFFFFF;
$surface-alt: #F1F4F9;
$surface-hover: #F8F9FD;

$indigo: #1428A0;
$indigo-light: #4C63C7;
$indigo-dark: #0E1C74;
$violet: #2B45C9;
$indigo-pale: #E9EBF8;

$amber: #F59E0B;
$amber-dark: #D97706;
$amber-pale: #FFFBEB;

$emerald: #10B981;
$emerald-dark: #059669;
$emerald-pale: #ECFDF5;

$red: #EF4444;
$red-pale: #FEF2F2;
$sky: #0EA5E9;
$sky-pale: #F0F9FF;
$rose: #F43F5E;
$rose-pale: #FFF1F2;

$border: #E5E7EB;
$border-light: #F3F4F6;

$text-primary: #111827;
$text-secondary: #6B7280;
$text-muted: #9CA3AF;

$shadow-2: 0 2px 8px rgba(0, 0, 0, .06), 0 1px 2px rgba(0, 0, 0, .04);
$shadow-3: 0 8px 24px rgba(0, 0, 0, .08), 0 2px 6px rgba(0, 0, 0, .04);
$shadow-4: 0 16px 48px rgba(0, 0, 0, .1), 0 4px 12px rgba(0, 0, 0, .05);

$footer-height: 44px;
$sidebar-width: 256px;
// Small gap reclaimed above the fixed footer when a page's scroll shell
// pulls back the workspace content wrapper's bottom padding — see
// .pg-shell in global.scss.
$page-bottom-reclaim: 0.75rem;

$grad-primary: linear-gradient(135deg, #1428A0, #2B45C9);
$grad-warm: linear-gradient(135deg, #F59E0B, #F97316);
$grad-cool: linear-gradient(135deg, #10B981, #0EA5E9);

$font-display: 'Segoe UI', Roboto, Arial, sans-serif;
$font-body: 'Segoe UI', Roboto, Arial, sans-serif;
$font-mono: 'Segoe UI', Roboto, Arial, sans-serif;
