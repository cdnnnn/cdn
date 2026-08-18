import { Link, NavLink } from 'react-router-dom';
import { Home, Link2, Cpu, BookOpen, Play, FlaskConical, GitCompare, FileText, LogOut } from 'lucide-react';
import { useAppDispatch, useAppSelector } from '../../hooks/redux';
import { logout } from '../../store/slices/authSlice';
import ThemeToggle from '../common/ThemeToggle';
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
  { to: '/app/reports', icon: <FileText size={18} />, label: 'Reports' },
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
        <div className={styles['sidebar__theme-row']}>
          <span>Theme</span>
          <ThemeToggle />
        </div>
        <div className={styles['sidebar__user']}>
          <div className={styles['sidebar__avatar']}>{(user?.username || 'U').slice(0, 2).toUpperCase()}</div>
          <div className={styles['sidebar__user-info']}>
            <div className={styles['sidebar__user-name']}>{user?.profileName || user?.username || 'Guest'}</div>
            <div className={styles['sidebar__user-email']}>{user?.email || ''}</div>
          </div>
          <LogOut size={16} style={{ color: 'var(--text-muted)', cursor: 'pointer' }} onClick={() => dispatch(logout())} />
        </div>
      </div>
    </div>
  );
}























@use '../../styles/_variables' as *;

.sidebar {
  width: 256px;
  background: $surface;
  border-right: 1px solid $border;
  display: flex;
  flex-direction: column;
  flex-shrink: 0;

  &__logo {
    padding: 24px;
    display: flex;
    align-items: center;
    gap: 10px;
    font-family: $font-display;
    font-size: 18px;
    font-weight: 700;
    color: $text-primary;
    border-bottom: 1px solid $border-light;
    text-decoration: none;
    cursor: pointer;
    transition: opacity .15s;
  }

  &__logo:hover {
    opacity: .8;
  }

  &__mark {
    width: 30px;
    height: 30px;
    background: $grad-primary;
    border-radius: 9px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #fff;
    font-size: 14px;
    box-shadow: 0 2px 8px rgba(20, 40, 160, .25);
  }

  &__nav {
    flex: 1;
    padding: 14px 12px;
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  &__section {
    font-size: 11px;
    font-weight: 700;
    color: $text-muted;
    text-transform: uppercase;
    letter-spacing: 1.5px;
    padding: 20px 14px 8px;
    font-family: $font-display;
  }

  &__foot {
    padding: 16px;
    border-top: 1px solid $border-light;
  }

  &__theme-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 6px 4px 14px;
    font-size: 12px;
    font-weight: 600;
    color: $text-secondary;
  }

  &__user {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 10px;
    border-radius: 12px;
    transition: background .15s;
    cursor: pointer;
  }

  &__user:hover {
    background: $surface-alt;
  }

  &__avatar {
    width: 36px;
    height: 36px;
    border-radius: 10px;
    background: $grad-primary;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 14px;
    font-weight: 700;
    color: #fff;
  }

  &__user-info {
    flex: 1;
  }

  &__user-name {
    font-size: 13px;
    font-weight: 600;
    color: $text-primary;
  }

  &__user-email {
    font-size: 11px;
    color: $text-muted;
  }
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 11px 14px;
  border-radius: 12px;
  font-size: 14px;
  font-weight: 500;
  color: $text-secondary;
  cursor: pointer;
  transition: all .2s;
  border: none;
  background: none;
  width: 100%;
  text-align: left;
  text-decoration: none;
}

.nav-item:hover {
  color: $text-primary;
  background: $surface-alt;
}

.nav-item.active {
  color: $indigo;
  background: $indigo-pale;
  font-weight: 600;
  box-shadow: inset 3px 0 0 $indigo;

  svg {
    color: $indigo;
  }
}

@media (max-width: 768px) {
  .sidebar {
    display: none;
  }
}
