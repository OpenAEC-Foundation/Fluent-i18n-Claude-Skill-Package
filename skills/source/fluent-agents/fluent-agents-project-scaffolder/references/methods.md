# methods.md -- File Templates, Dependencies, Configuration Reference

## npm Dependency Reference

### Core Stack (React Projects)

```json
{
  "dependencies": {
    "@fluent/bundle": "^0.18.0",
    "@fluent/react": "^0.15.0",
    "@fluent/langneg": "^0.7.0"
  }
}
```

**Install command:**
```bash
npm install @fluent/bundle @fluent/react @fluent/langneg
```

### Minimal Stack (Vanilla JS / Node.js)

```json
{
  "dependencies": {
    "@fluent/bundle": "^0.18.0",
    "@fluent/langneg": "^0.7.0"
  }
}
```

**Install command:**
```bash
npm install @fluent/bundle @fluent/langneg
```

### Optional Development Dependency

```json
{
  "devDependencies": {
    "@fluent/syntax": "^0.19.0"
  }
}
```

### Peer Dependency Matrix

| Package | Peer Dependency | Minimum Version | Notes |
|---------|----------------|-----------------|-------|
| `@fluent/react` | `react` | 16.8.0 | Hooks support required |
| `@fluent/react` | `@fluent/bundle` | 0.16.0 | Must be installed alongside |
| All `@fluent/*` | Node.js | ^20.19 or ^22.12 or >=24 | Engine requirement |

---

## File Template: src/l10n/index.ts

```typescript
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";
import { createBundles } from "./bundles";

// ALWAYS list every locale that has a corresponding directory under public/locales/
export const AVAILABLE_LOCALES = ["en-US", "fr", "de"];

// ALWAYS set the default to a locale that has complete translations
export const DEFAULT_LOCALE = "en-US";

// FTL files to load per locale. ALWAYS match the actual filenames in public/locales/{locale}/
export const FTL_FILES = ["main.ftl"];

export async function initLocalization(): Promise<ReactLocalization> {
  const negotiated = negotiateLanguages(
    navigator.languages,
    AVAILABLE_LOCALES,
    { defaultLocale: DEFAULT_LOCALE }
  );

  const bundles = await createBundles(negotiated, FTL_FILES);
  return new ReactLocalization(bundles);
}
```

### Configuration Constants

| Constant | Type | Purpose |
|----------|------|---------|
| `AVAILABLE_LOCALES` | `string[]` | BCP 47 identifiers for all supported locales |
| `DEFAULT_LOCALE` | `string` | Fallback locale; MUST exist in `AVAILABLE_LOCALES` |
| `FTL_FILES` | `string[]` | FTL filenames to load per locale |

---

## File Template: src/l10n/bundles.ts

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

async function fetchMessages(locale: string, file: string): Promise<string> {
  const url = `/locales/${locale}/${file}`;
  const response = await fetch(url);
  if (!response.ok) {
    throw new Error(`Failed to fetch ${url}: ${response.status}`);
  }
  return response.text();
}

function createBundle(locale: string, messages: string[]): FluentBundle {
  const bundle = new FluentBundle(locale);
  for (const ftl of messages) {
    const resource = new FluentResource(ftl);
    const errors = bundle.addResource(resource);
    if (errors.length) {
      errors.forEach((e) =>
        console.warn(`FTL parse error in ${locale}:`, e)
      );
    }
  }
  return bundle;
}

export async function createBundles(
  locales: string[],
  files: string[]
): Promise<FluentBundle[]> {
  return Promise.all(
    locales.map(async (locale) => {
      const messages = await Promise.all(
        files.map((file) => fetchMessages(locale, file))
      );
      return createBundle(locale, messages);
    })
  );
}
```

### API Signatures

| Function | Signature | Returns |
|----------|-----------|---------|
| `fetchMessages` | `(locale: string, file: string) => Promise<string>` | Raw FTL text |
| `createBundle` | `(locale: string, messages: string[]) => FluentBundle` | Configured bundle with all resources loaded |
| `createBundles` | `(locales: string[], files: string[]) => Promise<FluentBundle[]>` | Array of bundles in preference order |

---

## File Template: src/l10n/switch.ts

```typescript
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";
import { AVAILABLE_LOCALES, DEFAULT_LOCALE, FTL_FILES } from "./index";
import { createBundles } from "./bundles";

const LOCALE_STORAGE_KEY = "preferred-locale";

export function getSavedLocale(): string | null {
  try {
    return localStorage.getItem(LOCALE_STORAGE_KEY);
  } catch {
    // localStorage unavailable (SSR, privacy mode)
    return null;
  }
}

export function saveLocalePreference(locale: string): void {
  try {
    localStorage.setItem(LOCALE_STORAGE_KEY, locale);
  } catch {
    // Silently fail if localStorage is unavailable
  }
}

export async function switchLocale(
  locale: string
): Promise<ReactLocalization> {
  saveLocalePreference(locale);
  const negotiated = negotiateLanguages(
    [locale],
    AVAILABLE_LOCALES,
    { defaultLocale: DEFAULT_LOCALE }
  );
  const bundles = await createBundles(negotiated, FTL_FILES);
  return new ReactLocalization(bundles);
}

export function getInitialLocales(): string[] {
  const saved = getSavedLocale();
  const requested = saved ? [saved] : Array.from(navigator.languages);
  return negotiateLanguages(requested, AVAILABLE_LOCALES, {
    defaultLocale: DEFAULT_LOCALE,
  });
}
```

### API Signatures

| Function | Signature | Returns |
|----------|-----------|---------|
| `getSavedLocale` | `() => string \| null` | Stored locale preference or null |
| `saveLocalePreference` | `(locale: string) => void` | Persists locale to localStorage |
| `switchLocale` | `(locale: string) => Promise<ReactLocalization>` | New localization instance for the chosen locale |
| `getInitialLocales` | `() => string[]` | Negotiated locales respecting saved preference |

---

## File Template: src/components/LocalizedApp.tsx

```tsx
import React, { useState, useEffect, useCallback } from "react";
import { LocalizationProvider } from "@fluent/react";
import { ReactLocalization } from "@fluent/react";
import { initLocalization, AVAILABLE_LOCALES } from "../l10n";
import { switchLocale, getSavedLocale } from "../l10n/switch";

interface LocalizedAppProps {
  children: React.ReactNode;
}

export function LocalizedApp({ children }: LocalizedAppProps) {
  const [l10n, setL10n] = useState<ReactLocalization | null>(null);

  useEffect(() => {
    const saved = getSavedLocale();
    if (saved) {
      switchLocale(saved).then(setL10n);
    } else {
      initLocalization().then(setL10n);
    }
  }, []);

  const handleLocaleSwitch = useCallback(async (locale: string) => {
    const newL10n = await switchLocale(locale);
    setL10n(newL10n);
  }, []);

  if (!l10n) {
    return <div>Loading...</div>;
  }

  return (
    <LocalizationProvider l10n={l10n}>
      {children}
    </LocalizationProvider>
  );
}
```

---

## File Template: FTL Baseline Messages

### en-US/main.ftl

```ftl
# Application
app-title = My Application

# Navigation
nav-home = Home
nav-settings = Settings
nav-about = About

# Common actions
action-save = Save
action-cancel = Cancel
action-delete = Delete
action-confirm = Confirm

# Welcome
welcome-user = Welcome, { $userName }!

# Plurals
items-count = { $count ->
    [one] You have { $count } item.
   *[other] You have { $count } items.
}

# Status
loading = Loading...
error-generic = Something went wrong. Please try again.
error-not-found = Page not found.
```

### fr/main.ftl

```ftl
# Application
app-title = Mon Application

# Navigation
nav-home = Accueil
nav-settings = Paramètres
nav-about = À propos

# Common actions
action-save = Enregistrer
action-cancel = Annuler
action-delete = Supprimer
action-confirm = Confirmer

# Welcome
welcome-user = Bienvenue, { $userName } !

# Plurals
items-count = { $count ->
    [one] Vous avez { $count } élément.
   *[other] Vous avez { $count } éléments.
}

# Status
loading = Chargement...
error-generic = Un problème est survenu. Veuillez réessayer.
error-not-found = Page introuvable.
```

---

## Configuration: negotiateLanguages Options

| Option | Type | Default | When to Use |
|--------|------|---------|-------------|
| `strategy: "filtering"` | string | Default | Maximum coverage; returns all matching locales as fallback chain |
| `strategy: "matching"` | string | -- | One best match per requested locale |
| `strategy: "lookup"` | string | -- | Single best locale; requires `defaultLocale` or throws |
| `defaultLocale` | string | -- | ALWAYS provide; appended as ultimate fallback |

**ALWAYS** use `"filtering"` strategy (default) for React apps -- it produces the longest fallback chain, maximizing translation coverage.

**ALWAYS** provide `defaultLocale` -- without it, `negotiateLanguages` may return an empty array if no matches are found.

---

## Configuration: FluentBundle Options

| Option | Type | Default | When to Set |
|--------|------|---------|-------------|
| `useIsolating` | boolean | `true` | Set `false` ONLY for testing or purely LTR content |
| `functions` | Record | `{}` | When custom FTL functions (beyond NUMBER/DATETIME) are needed |
| `transform` | TextTransform | identity | When applying global text transforms (e.g., brand name injection) |

**NEVER** disable `useIsolating` in production unless all content is guaranteed unidirectional.
