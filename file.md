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
          <button
            type="button"
            className={styles['sidebar__logout']}
            onClick={() => dispatch(logout())}
            title="Log out"
          >
            <LogOut size={16} />
          </button>
        </div>
      </div>
    </div>
  );
}





















@use '../../styles/_variables' as *;

// ===========================================================================
// Sidebar — matches History / Reports / Comparison / New Evaluation design
// system: ink/paper palette, ultramarine signal accent, mono instrument
// labels, hover-lift.
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
    background: $ink;
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
    background: $ink;
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
