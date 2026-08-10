import { Routes, Route, Navigate } from 'react-router-dom';
import Landing from '../components/landing/Landing';
import AppShell from '../components/layout/AppShell';
import ProtectedRoute from './ProtectedRoute';
import Dashboard from '../components/dashboard/Dashboard';
import Providers from '../components/providers/Providers';
import ModelCatalog from '../components/models/ModelCatalog';
import TestSuites from '../components/suites/TestSuites';
import NewEvaluation from '../components/evaluations/NewEvaluation';
import Evaluations from '../components/evaluations/Evaluations';
import EvaluationDetail from '../components/evaluations/EvaluationDetail';
import Comparison from '../components/comparison/Comparison';
import Reports from '../components/reports/Reports';

const MOCKS_ENABLED = import.meta.env.VITE_ENABLE_MOCKS === 'true';

export default function AppRoutes() {
  return (
    <Routes>
      {/* In mock mode, main.tsx already seeds a session before render, so
          skip the landing/SSO page entirely and land straight in the app. */}
      <Route path="/" element={MOCKS_ENABLED ? <Navigate to="/app/dashboard" replace /> : <Landing />} />

      <Route element={<ProtectedRoute />}>
        <Route path="/app" element={<AppShell />}>
          <Route index element={<Navigate to="dashboard" replace />} />
          <Route path="dashboard" element={<Dashboard />} />
          <Route path="providers" element={<Providers />} />
          <Route path="models" element={<ModelCatalog />} />
          <Route path="suites" element={<TestSuites />} />
          <Route path="new-eval" element={<NewEvaluation />} />
          <Route path="evaluations" element={<Evaluations />} />
          <Route path="evaluations/:id" element={<EvaluationDetail />} />
          <Route path="comparison" element={<Comparison />} />
          <Route path="reports" element={<Reports />} />
        </Route>
      </Route>

      <Route path="*" element={<Navigate to="/" replace />} />
    </Routes>
  );
}
