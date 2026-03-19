# Examples Reference — fluent-impl-locale-switching

## Example 1: Complete Locale Switcher Component

A production-ready locale switcher with persistence, negotiation, and accessible UI.

### FTL Messages

```ftl
# locales/en-US/ui.ftl
locale-switcher-label = Select language
locale-switcher-current = Current language: { $locale }

# locales/fr/ui.ftl
locale-switcher-label = Choisir la langue
locale-switcher-current = Langue actuelle : { $locale }

# locales/de/ui.ftl
locale-switcher-label = Sprache auswahlen
locale-switcher-current = Aktuelle Sprache: { $locale }
```

### Localization Module

```typescript
// src/l10n/index.ts
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";

export const AVAILABLE_LOCALES = ["en-US", "fr", "de"] as const;
export type AppLocale = (typeof AVAILABLE_LOCALES)[number];
export const DEFAULT_LOCALE: AppLocale = "en-US";

const STORAGE_KEY = "fluent-locale-preference";

export function getSavedLocale(): string | null {
  try {
    return localStorage.getItem(STORAGE_KEY);
  } catch {
    return null;
  }
}

export function saveLocalePreference(locale: string): void {
  try {
    localStorage.setItem(STORAGE_KEY, locale);
  } catch {
    // Silently fail in restricted environments
  }
}

export function clearLocalePreference(): void {
  try {
    localStorage.removeItem(STORAGE_KEY);
  } catch {
    // Silently fail
  }
}

export function negotiateLocales(requested: readonly string[]): string[] {
  return negotiateLanguages(requested, [...AVAILABLE_LOCALES], {
    defaultLocale: DEFAULT_LOCALE,
  });
}

export function detectInitialLocales(): string[] {
  const saved = getSavedLocale();
  const requested = saved ? [saved] : navigator.languages;
  return negotiateLocales(requested);
}

async function fetchMessages(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/messages.ftl`);
  if (!response.ok) {
    console.warn(`Failed to load translations for ${locale}`);
    return "";
  }
  return response.text();
}

export async function createBundles(
  locales: string[]
): Promise<FluentBundle[]> {
  return Promise.all(
    locales.map(async (locale) => {
      const messages = await fetchMessages(locale);
      const bundle = new FluentBundle(locale);
      const errors = bundle.addResource(new FluentResource(messages));
      if (errors.length) {
        errors.forEach((e) =>
          console.warn(`FTL parse error in ${locale}:`, e)
        );
      }
      return bundle;
    })
  );
}

export async function createLocalization(
  locales: string[]
): Promise<ReactLocalization> {
  const bundles = await createBundles(locales);
  return new ReactLocalization(bundles);
}
```

### Root App Component

```tsx
// src/App.tsx
import React, { useState, useCallback, useEffect } from "react";
import { LocalizationProvider } from "@fluent/react";
import { ReactLocalization } from "@fluent/react";
import {
  detectInitialLocales,
  negotiateLocales,
  saveLocalePreference,
  createLocalization,
  AVAILABLE_LOCALES,
} from "./l10n";
import { LocaleSwitcher } from "./components/LocaleSwitcher";

export function App() {
  const [currentLocales, setCurrentLocales] = useState(detectInitialLocales);
  const [l10n, setL10n] = useState<ReactLocalization | null>(null);

  // Load translations on locale change
  useEffect(() => {
    let cancelled = false;
    createLocalization(currentLocales).then((newL10n) => {
      if (!cancelled) setL10n(newL10n);
    });
    return () => {
      cancelled = true;
    };
  }, [currentLocales]);

  const switchLocale = useCallback((locale: string) => {
    saveLocalePreference(locale);
    setCurrentLocales(negotiateLocales([locale]));
  }, []);

  if (!l10n) return <div>Loading translations...</div>;

  return (
    <LocalizationProvider l10n={l10n}>
      <header>
        <LocaleSwitcher
          available={[...AVAILABLE_LOCALES]}
          current={currentLocales[0]}
          onSwitch={switchLocale}
        />
      </header>
      <main>{/* Application content */}</main>
    </LocalizationProvider>
  );
}
```

### Locale Switcher UI Component

```tsx
// src/components/LocaleSwitcher.tsx
import React from "react";
import { Localized } from "@fluent/react";

interface LocaleSwitcherProps {
  available: string[];
  current: string;
  onSwitch: (locale: string) => void;
}

// Display names in their own language for immediate recognition
const NATIVE_NAMES: Record<string, string> = {
  "en-US": "English (US)",
  fr: "Francais",
  de: "Deutsch",
  nl: "Nederlands",
  es: "Espanol",
  it: "Italiano",
  ja: "Japanese",
  "zh-Hans": "Simplified Chinese",
  "pt-BR": "Portugues (Brasil)",
};

export function LocaleSwitcher({
  available,
  current,
  onSwitch,
}: LocaleSwitcherProps) {
  return (
    <nav aria-label="Language selection">
      <Localized id="locale-switcher-label" attrs={{ "aria-label": true }}>
        <select
          value={current}
          onChange={(e) => onSwitch(e.target.value)}
          aria-label="Select language"
        >
          {available.map((locale) => (
            <option key={locale} value={locale}>
              {NATIVE_NAMES[locale] ?? locale}
            </option>
          ))}
        </select>
      </Localized>
    </nav>
  );
}
```

### Button-Based Locale Switcher (Alternative)

```tsx
// src/components/LocaleButtons.tsx
import React from "react";

interface LocaleButtonsProps {
  available: string[];
  current: string;
  onSwitch: (locale: string) => void;
}

const NATIVE_NAMES: Record<string, string> = {
  "en-US": "English",
  fr: "Francais",
  de: "Deutsch",
};

export function LocaleButtons({
  available,
  current,
  onSwitch,
}: LocaleButtonsProps) {
  return (
    <div role="radiogroup" aria-label="Language selection">
      {available.map((locale) => (
        <button
          key={locale}
          role="radio"
          aria-checked={locale === current}
          onClick={() => onSwitch(locale)}
          disabled={locale === current}
        >
          {NATIVE_NAMES[locale] ?? locale}
        </button>
      ))}
    </div>
  );
}
```

---

## Example 2: Server-Side Locale Detection (Express + SSR)

```typescript
// server/locale-middleware.ts
import { acceptedLanguages, negotiateLanguages } from "@fluent/langneg";
import type { Request, Response, NextFunction } from "express";

const AVAILABLE_LOCALES = ["en-US", "fr", "de"];
const DEFAULT_LOCALE = "en-US";
const COOKIE_NAME = "locale-preference";

export function localeMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  // Priority 1: Explicit cookie preference
  const cookieLocale = req.cookies?.[COOKIE_NAME];
  if (cookieLocale) {
    req.locales = negotiateLanguages([cookieLocale], AVAILABLE_LOCALES, {
      defaultLocale: DEFAULT_LOCALE,
    });
    next();
    return;
  }

  // Priority 2: Accept-Language header
  const headerValue = req.headers["accept-language"] || "";
  const requested = acceptedLanguages(headerValue);
  req.locales = negotiateLanguages(requested, AVAILABLE_LOCALES, {
    defaultLocale: DEFAULT_LOCALE,
  });
  next();
}

// Locale switch endpoint
export function handleLocaleSwitch(req: Request, res: Response): void {
  const { locale } = req.body;
  if (!AVAILABLE_LOCALES.includes(locale)) {
    res.status(400).json({ error: "Unsupported locale" });
    return;
  }

  // Set cookie for 1 year
  res.cookie(COOKIE_NAME, locale, {
    maxAge: 365 * 24 * 60 * 60 * 1000,
    path: "/",
    sameSite: "lax",
  });

  res.json({ locale, negotiated: negotiateLanguages([locale], AVAILABLE_LOCALES, { defaultLocale: DEFAULT_LOCALE }) });
}
```

```typescript
// server/ssr-render.ts — SSR with locale detection
import { renderToString } from "react-dom/server";
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { ReactLocalization, LocalizationProvider } from "@fluent/react";
import { JSDOM } from "jsdom";
import * as fs from "fs";
import * as path from "path";

function parseMarkup(str: string): Node[] {
  const dom = new JSDOM(`<body>${str}</body>`);
  return Array.from(dom.window.document.body.childNodes);
}

function loadBundlesSync(locales: string[]): FluentBundle[] {
  return locales.map((locale) => {
    const ftlPath = path.join(__dirname, `../locales/${locale}/messages.ftl`);
    const messages = fs.readFileSync(ftlPath, "utf-8");
    const bundle = new FluentBundle(locale);
    bundle.addResource(new FluentResource(messages));
    return bundle;
  });
}

export function renderApp(locales: string[]): string {
  const bundles = loadBundlesSync(locales);
  const l10n = new ReactLocalization(bundles, parseMarkup);

  const html = renderToString(
    <LocalizationProvider l10n={l10n}>
      <App />
    </LocalizationProvider>
  );

  return `<!DOCTYPE html>
<html lang="${locales[0]}" dir="${getDirection(locales[0])}">
<head>
  <meta charset="utf-8" />
  <script>window.__INITIAL_LOCALES__=${JSON.stringify(locales)};</script>
</head>
<body>
  <div id="root">${html}</div>
  <script src="/bundle.js"></script>
</body>
</html>`;
}

function getDirection(locale: string): string {
  const rtlLocales = ["ar", "he", "fa", "ur"];
  const lang = locale.split("-")[0];
  return rtlLocales.includes(lang) ? "rtl" : "ltr";
}
```

### Client Hydration with Server-Detected Locale

```tsx
// src/client-entry.tsx
import React from "react";
import { hydrateRoot } from "react-dom/client";
import { LocalizationProvider } from "@fluent/react";
import { getSavedLocale, negotiateLocales, createLocalization } from "./l10n";

async function hydrate() {
  // Use server-detected locales for initial hydration to avoid mismatch
  const serverLocales: string[] = (window as any).__INITIAL_LOCALES__;

  // After hydration, check if user has a different saved preference
  const saved = getSavedLocale();
  const clientLocales = saved ? negotiateLocales([saved]) : serverLocales;

  const l10n = await createLocalization(clientLocales);

  hydrateRoot(
    document.getElementById("root")!,
    <LocalizationProvider l10n={l10n}>
      <App />
    </LocalizationProvider>
  );
}

hydrate();
```

---

## Example 3: Preference Persistence Patterns

### localStorage Wrapper with Validation

```typescript
// src/l10n/storage.ts
const STORAGE_KEY = "fluent-locale-preference";

export function getSavedLocale(
  availableLocales: readonly string[]
): string | null {
  try {
    const saved = localStorage.getItem(STORAGE_KEY);
    // Validate: only return if the saved locale is still supported
    if (saved && availableLocales.includes(saved)) {
      return saved;
    }
    // Clean up invalid saved preference
    if (saved) {
      localStorage.removeItem(STORAGE_KEY);
    }
    return null;
  } catch {
    return null;
  }
}

export function saveLocalePreference(locale: string): void {
  try {
    localStorage.setItem(STORAGE_KEY, locale);
  } catch {
    // localStorage unavailable — silently degrade
  }
}

export function clearLocalePreference(): void {
  try {
    localStorage.removeItem(STORAGE_KEY);
  } catch {
    // Silently fail
  }
}
```

### Reset to Browser Default

```tsx
// "Use browser language" button pattern
function ResetLocaleButton({ onReset }: { onReset: () => void }) {
  const handleReset = () => {
    clearLocalePreference();
    onReset();
  };

  return (
    <Localized id="locale-reset-button">
      <button onClick={handleReset}>Use browser language</button>
    </Localized>
  );
}

// In parent component:
const resetToBrowserDefault = useCallback(() => {
  clearLocalePreference();
  const negotiated = negotiateLocales(navigator.languages);
  setCurrentLocales(negotiated);
}, []);
```

---

## Example 4: Locale-Aware Document Attributes

```tsx
// Update <html> element when locale changes
useEffect(() => {
  const primaryLocale = currentLocales[0];
  document.documentElement.lang = primaryLocale;

  // Set text direction for RTL languages
  const rtlLanguages = ["ar", "he", "fa", "ur"];
  const langBase = primaryLocale.split("-")[0];
  document.documentElement.dir = rtlLanguages.includes(langBase)
    ? "rtl"
    : "ltr";
}, [currentLocales]);
```
