// ═══════════════════════════════════════════════
// components/AuthSpinner/AuthSpinner.tsx
// SemcoEval · Full-screen auth overlay
// ═══════════════════════════════════════════════
import type { FC } from 'react';
import styles from './AuthSpinner.module.scss';

const AuthSpinner: FC = () => (
  <div className={styles.authSpinner} role="status" aria-live="polite">
    <div className={styles.authSpinnerCard}>
      <div className={styles.authSpinnerMark}>
        <img src="/assets/logo.png" alt="SemcoEval" className={styles.authSpinnerLogo} />
      </div>
      <div className={styles.authSpinnerRing}>
        <div className={styles.authSpinnerArc} />
      </div>
      <div className={styles.authSpinnerLabel}>Authenticating…</div>
      <div className={styles.authSpinnerSub}>Connecting to Knox SSO</div>
    </div>
  </div>
);

export default AuthSpinner;
















@use '../../styles/_variables' as *;

.authSpinner {
  position: fixed;
  inset: 0;
  z-index: 1000;
  display: grid;
  place-items: center;
  background: $bg;
}

.authSpinnerCard {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1.125rem;
  padding: 2.5rem 3rem;
  background: $surface;
  border: 1px solid $border-light;
  border-radius: 1.25rem;
  box-shadow: $shadow-4;
}

.authSpinnerMark {
  width: 52px;
  height: 52px;
  border-radius: 1rem;
  display: grid;
  place-items: center;
  color: #fff;
  background: linear-gradient(155deg, $indigo 0%, $indigo-dark 100%);
  box-shadow: $shadow-3, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.14);
  animation: auth-spinner-pulse 2.2s ease-in-out infinite;
  overflow: hidden;
}

.authSpinnerLogo {
  width: 28px;
  height: 28px;
  object-fit: contain;
  display: block;
}

.authSpinnerRing {
  position: relative;
  width: 32px;
  height: 32px;
}

.authSpinnerArc {
  position: absolute;
  inset: 0;
  border-radius: 50%;
  border: 3px solid $border;
  border-top-color: $indigo;
  animation: auth-spinner-spin 0.8s linear infinite;
}

.authSpinnerLabel {
  font-family: $font-display;
  font-size: 0.9375rem;
  font-weight: 700;
  color: $text-primary;
  letter-spacing: -0.01em;
}

.authSpinnerSub {
  margin-top: -0.75rem;
  font-size: 0.8125rem;
  color: $text-muted;
}

@media (min-width: 1800px) {
  .authSpinnerLabel {
    font-size: 1.03125rem;
  }
  .authSpinnerSub {
    font-size: 0.90625rem;
  }
}

@keyframes auth-spinner-spin {
  to {
    transform: rotate(360deg);
  }
}

@keyframes auth-spinner-pulse {
  0%,
  100% {
    transform: scale(1);
    box-shadow: $shadow-3, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.14);
  }
  50% {
    transform: scale(1.05);
    box-shadow: $shadow-4, inset 0 0 0 0.0625rem rgba(255, 255, 255, 0.18);
  }
}
