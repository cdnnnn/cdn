import { Link, NavLink } from 'react-router-dom';
import { Home, Link2, Cpu, BookOpen, Play, FlaskConical, GitCompare, FileText, MessageSquare, LogOut } from 'lucide-react';
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

// Assumed routes/icon — adjust to match the actual VoC pages once they exist
const vocItems = [
  { to: '/app/voc', icon: <MessageSquare size={18} />, label: 'Voice of Customer' },
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
        <div className={styles['sidebar__section']}>VoCs</div>
        {vocItems.map((item) => (
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
