---
name: fluent-impl-locale-switching
description: >
  Use when implementing runtime locale or language switching in Fluent React applications.
  Prevents full page reloads and broken fallback chains during dynamic language changes.
  Covers React state re-localization, negotiateLanguages integration, localStorage persistence, and server-side detection.
  Keywords: language switcher, negotiateLanguages, localStorage,
  acceptedLanguages, re-render, fallback chain, change language at runtime,
  language dropdown, switch locale without reload, remember language.
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/react 0.15+, @fluent/langneg 0.7+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-impl-locale-switching

## Quick Reference

### Locale Switching Architecture

| Component | Role | Package |
|-----------|------|---------|
| `useState` | Holds current negotiated locales array | React |
| `negotiateLanguages` | Matches user preference against available locales | `@fluent/langneg` |
| `ReactLocalization` | Wraps bundles for the provider; recreated on locale change | `@fluent/react` |
| `LocalizationProvider` | Distributes translations via React Context | `@fluent/react` |
| `acceptedLanguages` | Parses HTTP Accept-Language header (server-side) | `@fluent/langneg` |
| `navigator.languages` | Browser-reported user locale preferences (client-side) | Web API |
| `localStorage` | Persists explicit user locale choice (client-side) | Web API |

### Switching Flow

```
User clicks locale ──> setState([userChoice]) ──> negotiateLanguages()
  ──> generateBundles(negotiated) ──> new ReactLocalization(bundles)
  ──> LocalizationProvider re-renders ──> all <Localized> update automatically
```

### Critical Warnings

**NEVER** create a new `ReactLocalization` instance inside the render body without memoization. ALWAYS use `useState` or `useMemo` to hold the instance. Creating it on every render triggers full re-rendering of all localized components.

**NEVER** hardcode locale identifiers scattered throughout the application. ALWAYS define a single `AVAILABLE_LOCALES` constant and derive all locale lists from it.

**NEVER** skip `negotiateLanguages` and pass raw user input directly as the locale list. ALWAYS negotiate to produce a valid fallback chain with a guaranteed `defaultLocale`.

**ALWAYS** persist the user's explicit locale choice to `localStorage` (client) or a cookie (server) so it survives page reloads and new sessions.

**ALWAYS** provide a `defaultLocale` to `negotiateLanguages`. Without it, the result array may be empty if no locales match.

---

## Decision Tree: Locale Detection Strategy

```
Where does the app run?
├── Browser only (SPA)
│   ├── Has user previously chosen a locale?
│   │   ├── YES ──> Read from localStorage, use as requestedLocales[0]
│   │   └── NO  ──> Use navigator.languages as requestedLocales
│   └── negotiateLanguages(requestedLocales, AVAILABLE_LOCALES, { defaultLocale })
├── Server-side (SSR / Next.js / Express)
│   ├── Has user cookie with locale preference?
│   │   ├── YES ──> Use cookie value as requestedLocales[0]
│   │   └── NO  ──> acceptedLanguages(req.headers["accept-language"])
│   └── negotiateLanguages(requestedLocales, AVAILABLE_LOCALES, { defaultLocale })
└── Hybrid (SSR + client hydration)
    ├── Server: detect via Accept-Language header or cookie
    ├── Client: override with localStorage preference on hydration
    └── Ensure server and client negotiate the SAME locale to avoid hydration mismatch
```

---

## Pattern 1: Complete Locale Switcher (Client-Side)

```typescript
// l10n.ts — Locale management module
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";

export const AVAILABLE_LOCALES = ["en-US", "fr", "de", "nl"] as const;
export const DEFAULT_LOCALE = "en-US";
const STORAGE_KEY = "fluent-locale-preference";

// Message store — in production, fetch these from .ftl files
const MESSAGES: Record<string, string> = {};

export function getSavedLocale(): string | null {
  try {
    return localStorage.getItem(STORAGE_KEY);
  } catch {
    return null; // localStorage unavailable (SSR, privacy mode)
  }
}

export function saveLocalePreference(locale: string): void {
  try {
    localStorage.setItem(STORAGE_KEY, locale);
  } catch {
    // Silently fail — persistence is optional
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

function generateBundles(locales: string[]): FluentBundle[] {
  return locales
    .filter((locale) => MESSAGES[locale])
    .map((locale) => {
      const bundle = new FluentBundle(locale);
      const errors = bundle.addResource(new FluentResource(MESSAGES[locale]));
      if (errors.length) {
        errors.forEach((e) => console.warn(`FTL error [${locale}]:`, e));
      }
      return bundle;
    });
}

export function createLocalization(locales: string[]): ReactLocalization {
  return new ReactLocalization(generateBundles(locales));
}
```

```tsx
// App.tsx — Root component with locale switching
import React, { useState, useCallback } from "react";
import { LocalizationProvider } from "@fluent/react";
import {
  detectInitialLocales,
  negotiateLocales,
  saveLocalePreference,
  createLocalization,
  AVAILABLE_LOCALES,
} from "./l10n";

function App() {
  const [currentLocales, setCurrentLocales] = useState(detectInitialLocales);

  const l10n = React.useMemo(
    () => createLocalization(currentLocales),
    [currentLocales]
  );

  const switchLocale = useCallback((locale: string) => {
    saveLocalePreference(locale);
    const negotiated = negotiateLocales([locale]);
    setCurrentLocales(negotiated);
  }, []);

  return (
    <LocalizationProvider l10n={l10n}>
      <LocaleSwitcher
        available={[...AVAILABLE_LOCALES]}
        current={currentLocales[0]}
        onSwitch={switchLocale}
      />
      <MainContent />
    </LocalizationProvider>
  );
}
```

```tsx
// LocaleSwitcher.tsx — UI component for locale selection
import React from "react";
import { Localized } from "@fluent/react";

interface LocaleSwitcherProps {
  available: string[];
  current: string;
  onSwitch: (locale: string) => void;
}

const LOCALE_NAMES: Record<string, string> = {
  "en-US": "English",
  fr: "Francais",
  de: "Deutsch",
  nl: "Nederlands",
};

function LocaleSwitcher({ available, current, onSwitch }: LocaleSwitcherProps) {
  return (
    <Localized id="locale-switcher-label" attrs={{ "aria-label": true }}>
      <select
        value={current}
        onChange={(e) => onSwitch(e.target.value)}
        aria-label="Select language"
      >
        {available.map((locale) => (
          <option key={locale} value={locale}>
            {LOCALE_NAMES[locale] ?? locale}
          </option>
        ))}
      </select>
    </Localized>
  );
}
```

---

## Pattern 2: Server-Side Locale Detection

```typescript
// server-l10n.ts — Express/Node.js locale detection
import { acceptedLanguages, negotiateLanguages } from "@fluent/langneg";
import type { Request } from "express";

const AVAILABLE_LOCALES = ["en-US", "fr", "de", "nl"];
const DEFAULT_LOCALE = "en-US";
const COOKIE_NAME = "locale-preference";

export function detectServerLocale(req: Request): string[] {
  // Priority 1: Explicit user cookie
  const cookieLocale = req.cookies?.[COOKIE_NAME];
  if (cookieLocale) {
    return negotiateLanguages([cookieLocale], AVAILABLE_LOCALES, {
      defaultLocale: DEFAULT_LOCALE,
    });
  }

  // Priority 2: Accept-Language header
  const headerValue = req.headers["accept-language"] || "";
  const requested = acceptedLanguages(headerValue);
  return negotiateLanguages(requested, AVAILABLE_LOCALES, {
    defaultLocale: DEFAULT_LOCALE,
  });
}
```

```typescript
// SSR rendering with detected locale
import { renderToString } from "react-dom/server";
import { JSDOM } from "jsdom";

function parseMarkup(str: string): Node[] {
  const dom = new JSDOM(`<body>${str}</body>`);
  return Array.from(dom.window.document.body.childNodes);
}

app.get("*", async (req, res) => {
  const locales = detectServerLocale(req);
  const bundles = await loadBundles(locales); // pre-load ALL bundles
  const l10n = new ReactLocalization(bundles, parseMarkup);

  const html = renderToString(
    <LocalizationProvider l10n={l10n}>
      <App />
    </LocalizationProvider>
  );

  // Pass detected locale to client for hydration
  res.send(`
    <html lang="${locales[0]}">
      <head><script>window.__LOCALES__=${JSON.stringify(locales)}</script></head>
      <body><div id="root">${html}</div></body>
    </html>
  `);
});
```

---

## Pattern 3: negotiateLanguages Integration

### Negotiation Strategies for Locale Switching

| Strategy | Use Case | Behavior |
|----------|----------|----------|
| `"filtering"` (default) | Locale switcher with fallback chain | Returns ALL matching locales, best for `ReactLocalization` |
| `"matching"` | One best-fit per user preference | Returns one match per requested locale |
| `"lookup"` | Single locale selection (date libraries) | Returns exactly one locale; requires `defaultLocale` or throws |

**ALWAYS** use `"filtering"` (the default) when building the bundle list for `ReactLocalization`. This produces the full fallback chain: if `de-AT` is requested and both `de-AT` and `de` are available, both appear in the result, giving maximum translation coverage.

```typescript
// Filtering: full fallback chain for ReactLocalization
const locales = negotiateLanguages(
  ["de-AT"],
  ["en-US", "de", "de-AT", "fr"],
  { defaultLocale: "en-US" }
);
// Result: ["de-AT", "de", "en-US"]
// de-AT first (exact match), de second (language fallback), en-US last (default)
```

---

## Pattern 4: Async Bundle Loading on Locale Switch

```typescript
// Async locale switching with loading state
function App() {
  const [currentLocales, setCurrentLocales] = useState(detectInitialLocales);
  const [l10n, setL10n] = useState<ReactLocalization | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  // Load bundles whenever locales change
  React.useEffect(() => {
    let cancelled = false;
    setIsLoading(true);

    loadBundlesAsync(currentLocales).then((bundles) => {
      if (!cancelled) {
        setL10n(new ReactLocalization(bundles));
        setIsLoading(false);
      }
    });

    return () => { cancelled = true; };
  }, [currentLocales]);

  const switchLocale = useCallback((locale: string) => {
    saveLocalePreference(locale);
    setCurrentLocales(negotiateLocales([locale]));
  }, []);

  if (!l10n) return <div>Loading translations...</div>;

  return (
    <LocalizationProvider l10n={l10n}>
      {isLoading && <div className="locale-loading-indicator" />}
      <LocaleSwitcher onSwitch={switchLocale} />
      <MainContent />
    </LocalizationProvider>
  );
}
```

**ALWAYS** use an effect cleanup function (`cancelled` flag) when loading bundles asynchronously. This prevents setting state on an unmounted component or applying stale locale data if the user switches locales rapidly.

---

## navigator.languages for Initial Detection

```typescript
// navigator.languages returns an array like ["en-US", "en", "fr"]
// ALWAYS use it as the initial requestedLocales when no saved preference exists
const initialLocales = negotiateLanguages(
  navigator.languages,     // readonly string[] — user's browser preferences
  AVAILABLE_LOCALES,
  { defaultLocale: DEFAULT_LOCALE }
);
```

**Key facts about `navigator.languages`:**
- Returns a frozen array of BCP 47 locale strings ordered by user preference
- Includes both language-region (`en-US`) and bare language (`en`) tags
- Available in all modern browsers; returns `["en-US"]` as fallback in older environments
- NEVER available on the server — use `acceptedLanguages()` for SSR

---

## Reference Links

- [references/methods.md](references/methods.md) -- API signatures for negotiateLanguages, acceptedLanguages, ReactLocalization, locale state management
- [references/examples.md](references/examples.md) -- Complete locale switcher component, server-side detection, preference persistence
- [references/anti-patterns.md](references/anti-patterns.md) -- What NOT to do when switching locales

### Official Sources

- https://github.com/projectfluent/fluent.js/wiki/React-Bindings
- https://github.com/projectfluent/fluent.js/wiki/React-Tutorial
- https://github.com/projectfluent/fluent.js/wiki/ReactLocalization
- https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/README.md
