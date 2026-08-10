import { Routes, Route, Navigate } from 'react-router-dom';
import Landing from '../components/landing/Landing';
import SsoLogin from '../components/auth/SsoLogin';
import AuthGuard from '../components/AuthGuard/AuthGuard';
import AppShell from '../components/layout/AppShell';
import Dashboard from '../components/dashboard/Dashboard';
import Providers from '../components/providers/Providers';
import ModelCatalog from '../components/models/ModelCatalog';
import Datasets from '../components/datasets/Datasets';
import NewEvaluation from '../components/evaluations/NewEvaluation';
import History from '../components/history/History';
import Comparison from '../components/comparison/Comparison';
import Reports from '../components/reports/Reports';

export default function AppRoutes() {
  return (
    <Routes>
      {/* AuthGuard redirects here with { from, errorMessage } on error/logged_out.
          This is the only route not gated by AuthGuard, so it must stay outside it. */}
      <Route path="/sso-login" element={<SsoLogin />} />

      {/* AuthGuard now wraps BOTH "/" and "/app/*" — the landing page's content
          only renders once authenticated, same as the rest of the app. It
          triggers the SSO WebSocket handshake (useSsoAuth) as soon as it
          mounts, which happens on every fresh page load / refresh (auth
          state is in-memory only, never persisted), and shows AuthSpinner
          while that's in flight. */}
      <Route element={<AuthGuard />}>
        <Route path="/" element={<Landing />} />

        <Route path="/app" element={<AppShell />}>
          <Route index element={<Navigate to="dashboard" replace />} />
          <Route path="dashboard" element={<Dashboard />} />
          <Route path="providers" element={<Providers />} />
          <Route path="models" element={<ModelCatalog />} />
          <Route path="datasets" element={<Datasets />} />
          <Route path="run-evaluation" element={<NewEvaluation />} />
          <Route path="history" element={<History />} />
          <Route path="comparison" element={<Comparison />} />
          <Route path="reports" element={<Reports />} />
        </Route>
      </Route>

      <Route path="*" element={<Navigate to="/" replace />} />
    </Routes>
  );
}
