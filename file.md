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
        // No explicit `lng` — LanguageDetector decides based on order below.
        // First-time visitors (nothing in localStorage) fall back to English.
        fallbackLng: 'en',
        supportedLngs: ['ko', 'en'],
        interpolation: {
            escapeValue: false,
        },
        detection: {
            order: ['localStorage', 'navigator'],
            caches: ['localStorage'],
        },
    });

export default i18n;
