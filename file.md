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
  font-size: 11px;
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











//Footer.tsx
// Shared footer used on both the public Landing page and every /app route
// (rendered from AppShell). Fixed to the bottom of the viewport.
//
// variant="full"   → spans the full viewport width (Landing — no sidebar)
// variant="app"     → offset past the sidebar width (/app/* — has a sidebar)
import packageJson from '../../../package.json';
import styles from './Footer.module.scss';

interface FooterProps {
  variant?: 'full' | 'app';
}

export default function Footer({ variant = 'full' }: FooterProps) {
  const year = new Date().getFullYear();
  return (
    <footer className={`${styles.footer} ${variant === 'app' ? styles['footer--app'] : ''}`}>
      <span className={styles.footer__version}>v{packageJson.version}</span>
      <span className={styles.footer__copyright}>
        &copy; {year} SemcoEval &middot; Privacy &middot; Terms &middot; Documentation
      </span>
    </footer>
  );
}











//Sidebar.module.scss
@use '../../styles/_variables' as *;

.sidebar {
  width: 256px; background: $surface; border-right: 1px solid $border;
  display: flex; flex-direction: column; flex-shrink: 0;

  &__logo {
    padding: 24px; display: flex; align-items: center; gap: 10px;
    font-family: $font-display; font-size: 18px; font-weight: 700;
    color: $text-primary; border-bottom: 1px solid $border-light;
    text-decoration: none; cursor: pointer; transition: opacity .15s;
  }
  &__logo:hover { opacity: .8; }
  &__mark {
    width: 30px; height: 30px; background: $grad-primary; border-radius: 9px;
    display: flex; align-items: center; justify-content: center; color: #fff;
    font-size: 14px; box-shadow: 0 2px 8px rgba(20, 40, 160, .25);
  }
  &__nav { flex: 1; padding: 14px 12px; display: flex; flex-direction: column; gap: 2px; }
  &__section {
    font-size: 11px; font-weight: 700; color: $text-muted; text-transform: uppercase;
    letter-spacing: 1.5px; padding: 20px 14px 8px; font-family: $font-display;
  }
  &__foot { padding: 16px; border-top: 1px solid $border-light; }
  &__user {
    display: flex; align-items: center; gap: 10px; padding: 10px; border-radius: 12px;
    transition: background .15s; cursor: pointer;
  }
  &__user:hover { background: $surface-alt; }
  &__avatar {
    width: 36px; height: 36px; border-radius: 10px; background: $grad-primary;
    display: flex; align-items: center; justify-content: center; font-size: 14px; font-weight: 700; color: #fff;
  }
  &__user-info { flex: 1; }
  &__user-name { font-size: 13px; font-weight: 600; color: $text-primary; }
  &__user-email { font-size: 11px; color: $text-muted; }
}

.nav-item {
  display: flex; align-items: center; gap: 12px; padding: 11px 14px; border-radius: 12px;
  font-size: 14px; font-weight: 500; color: $text-secondary; cursor: pointer; transition: all .2s;
  border: none; background: none; width: 100%; text-align: left; text-decoration: none;
}
.nav-item:hover { color: $text-primary; background: $surface-alt; }
.nav-item.active {
  color: $indigo; background: $indigo-pale; font-weight: 600; box-shadow: inset 3px 0 0 $indigo;
  svg { color: $indigo; }
}

@media (max-width: 768px) {
  .sidebar { display: none; }
}

















//Sidebar.tsx
import { Link, NavLink } from 'react-router-dom';
import { Home, Link2, Cpu, BookOpen, Play, FlaskConical, GitCompare, LogOut } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { logout } from '../../store/slices/authSlice';
import styles from './Sidebar.module.scss';

const navItems = [
  { to: '/app/dashboard', icon: <Home size={18} />, label: 'Dashboard' },
  { to: '/app/providers', icon: <Link2 size={18} />, label: 'Providers' },
  { to: '/app/models', icon: <Cpu size={18} />, label: 'Models' },
  { to: '/app/datasets', icon: <BookOpen size={18} />, label: 'Datasets' },
];

const workflowItems = [
  { to: '/app/run-evaluation', icon: <Play size={18} />, label: 'New Evaluation' },
  { to: '/app/history', icon: <FlaskConical size={18} />, label: 'History' },
  { to: '/app/comparison', icon: <GitCompare size={18} />, label: 'Comparison' },
];

export default function Sidebar() {
  const dispatch = useAppDispatch();
  const user = useAppSelector((s) => s.auth.user);

  const navLinkClass = ({ isActive }: { isActive: boolean }) =>
    `${styles['nav-item']} ${isActive ? styles.active : ''}`;

  return (
    <div className={styles.sidebar}>
      <Link to="/" className={styles['sidebar__logo']}>
        <div className={styles['sidebar__mark']}>&#9670;</div>
        SemcoEval
      </Link>
      <nav className={styles['sidebar__nav']}>
        {navItems.map((item) => (
          <NavLink key={item.to} to={item.to} className={navLinkClass}>
            {item.icon}
            {item.label}
          </NavLink>
        ))}
        <div className={styles['sidebar__section']}>Workflow</div>
        {workflowItems.map((item) => (
          <NavLink key={item.to} to={item.to} className={navLinkClass}>
            {item.icon}
            {item.label}
          </NavLink>
        ))}
      </nav>
      <div className={styles['sidebar__foot']}>
        <div className={styles['sidebar__user']}>
          <div className={styles['sidebar__avatar']}>{(user?.username || 'U').slice(0, 2).toUpperCase()}</div>
          <div className={styles['sidebar__user-info']}>
            <div className={styles['sidebar__user-name']}>{user?.profileName || user?.username || 'Guest'}</div>
            <div className={styles['sidebar__user-email']}>{user?.email || ''}</div>
          </div>
          <LogOut size={16} style={{ color: '#9CA3AF', cursor: 'pointer' }} onClick={() => dispatch(logout())} />
        </div>
      </div>
    </div>
  );
}














//AddCustomModelDrawer.module.scss
@use '../../styles/_variables' as *;

.drawer-overlay {
  position: fixed; top: 0; left: 0; right: 0; bottom: $footer-height;
  background: rgba(17, 24, 39, .4); z-index: 100;
  display: flex; justify-content: flex-end;
}
.drawer {
  width: 420px; max-width: 100%; height: 100%; background: $surface; box-shadow: $shadow-4;
  display: flex; flex-direction: column; animation: drawerIn .25s ease both;
}
@keyframes drawerIn { from { transform: translateX(24px); opacity: 0; } to { transform: translateX(0); opacity: 1; } }
.drawer__hdr {
  display: flex; justify-content: space-between; align-items: center; padding: 20px 24px; border-bottom: 1px solid $border-light;
  h2 { font-size: 18px; font-weight: 700; }
}
.drawer__close { background: none; border: none; cursor: pointer; color: $text-muted; }
.drawer__body { flex: 1; overflow-y: auto; padding: 24px; }
.drawer__foot { display: flex; justify-content: flex-end; gap: 10px; padding: 16px 24px; border-top: 1px solid $border-light; }










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

$footer-height: 32px;
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
