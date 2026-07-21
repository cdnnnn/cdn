// ═══════════════════════════════════════════════
// src/i18n/index.ts
// react-i18next configuration
// Default language: Korean (ko)
// ═══════════════════════════════════════════════
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';

import ko from './locales/ko.json';
import en from './locales/en.json';

i18n
    .use(LanguageDetector)
    .use(initReactI18next)
    .init({
        resources: {
            ko: { translation: ko },
            en: { translation: en },
        },
        // Default to Korean when nothing is saved yet.
        // Do NOT set `lng` here — that would override the detector entirely.
        fallbackLng: 'ko',
        supportedLngs: ['ko', 'en'],
        interpolation: {
            escapeValue: false, // React already handles XSS
        },
        detection: {
            // Only look at localStorage — ignore browser locale (navigator),
            // so first-time visitors always start at 'ko' regardless of
            // their browser/OS language.
            order: ['localStorage'],
            caches: ['localStorage'],
        },
    });

export default i18n;
