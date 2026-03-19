---
name: fluent-agents-project-scaffolder
description: >
  Use when scaffolding a new Fluent i18n project or generating boilerplate for React localization.
  Prevents incomplete setups by generating all required files with correct configuration.
  Covers FTL files per locale, FluentBundle config, LocalizationProvider, locale negotiation, and async loading.
  Keywords: scaffold, boilerplate, LocalizationProvider, @fluent/langneg, locale switching, code generation.
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/bundle 0.18+, @fluent/react 0.15+, @fluent/langneg 0.7+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-agents-project-scaffolder

## Quick Reference

### Generated File Manifest

| File | Purpose |
|------|---------|
| `public/locales/en-US/main.ftl` | Default locale FTL messages |
| `public/locales/{locale}/main.ftl` | Additional locale FTL messages (one per locale) |
| `src/l10n/index.ts` | Localization init, constants, async loader |
| `src/l10n/bundles.ts` | FluentBundle creation, resource loading, error handling |
| `src/l10n/switch.ts` | Locale switching with preference persistence |
| `src/components/LocalizedApp.tsx` | Root wrapper with LocalizationProvider and loading state |
| `package.json` | Updated with Fluent dependencies |

### Required npm Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `@fluent/bundle` | ^0.18.0 | Core runtime: parse FTL, format messages |
| `@fluent/react` | ^0.15.0 | React bindings: LocalizationProvider, Localized, useLocalization |
| `@fluent/langneg` | ^0.7.0 | Locale negotiation: match user preferences to available locales |

### Peer Dependencies (must already exist)

| Package | Minimum | Required By |
|---------|---------|-------------|
| `react` | 16.8.0 | `@fluent/react` |
| `react-dom` | 16.8.0 | `@fluent/react` |

### Critical Warnings

**ALWAYS** install all three core packages together -- they form an inseparable stack for React-based Fluent projects.

**ALWAYS** generate FTL files with a trailing newline -- the FTL parser expects it.

**ALWAYS** check `addResource()` return value for parse errors -- silent failures cause missing translations at runtime.

**NEVER** generate FTL files with concatenated strings or template literals at runtime -- ALWAYS serve `.ftl` files statically via `fetch()`.

**NEVER** create `ReactLocalization` inside a render function without memoization -- this causes full re-render of all localized components on every state change.

**NEVER** skip the async loading pattern -- bundling FTL strings into JavaScript defeats the purpose of locale-based code splitting.

**NEVER** hardcode `navigator.languages` as the only locale source -- ALWAYS pass it through `negotiateLanguages()` to handle fallbacks correctly.

---

## Decision Tree: Which Scaffold?

```
What type of project?
├── React app (Vite, CRA, Next.js client)
│   ├── New project → Full scaffold (all files)
│   └── Existing project → Integration scaffold (l10n/ + FTL files only)
│
├── Vanilla JS (no framework)
│   └── Skip: LocalizedApp.tsx, @fluent/react
│       Generate: l10n/index.ts (vanilla), FTL files, bundle setup
│       Install: @fluent/bundle + @fluent/langneg only
│
└── Node.js server (Express, Fastify)
    └── Skip: LocalizedApp.tsx, @fluent/react, localStorage
        Generate: l10n/index.ts (server), FTL files, bundle setup
        Install: @fluent/bundle + @fluent/langneg only
        Use: acceptedLanguages() from @fluent/langneg for HTTP headers
```

---

## Scaffolding Steps

### Step 1: Install Dependencies

```bash
# React project (ALWAYS all three)
npm install @fluent/bundle @fluent/react @fluent/langneg

# Vanilla JS or Node.js (no React bindings needed)
npm install @fluent/bundle @fluent/langneg
```

### Step 2: Create Directory Structure

```
project-root/
├── public/
│   └── locales/
│       ├── en-US/
│       │   └── main.ftl
│       ├── fr/
│       │   └── main.ftl
│       └── de/
│           └── main.ftl
└── src/
    ├── l10n/
    │   ├── index.ts
    │   ├── bundles.ts
    │   └── switch.ts
    └── components/
        └── LocalizedApp.tsx
```

**ALWAYS** use BCP 47 locale identifiers for directory names: `en-US`, `fr`, `de-DE`.

**ALWAYS** place FTL files under `public/locales/` for static serving via `fetch()`.

### Step 3: Generate FTL Files

ALWAYS seed each locale with these baseline messages:

```ftl
# public/locales/en-US/main.ftl
app-title = My Application
nav-home = Home
nav-settings = Settings
welcome-user = Welcome, { $userName }!
items-count = { $count ->
    [one] You have { $count } item.
   *[other] You have { $count } items.
}
loading = Loading...
error-generic = Something went wrong. Please try again.
```

**ALWAYS** use kebab-case for message IDs.

**ALWAYS** include a plural selector example to establish the pattern for translators.

**ALWAYS** include `loading` and `error-generic` messages -- every app needs them.

### Step 4: Generate l10n/index.ts

This is the entry point for localization. It exports constants and the initialization function:

```typescript
// src/l10n/index.ts
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";
import { createBundles } from "./bundles";

export const AVAILABLE_LOCALES = ["en-US", "fr", "de"];
export const DEFAULT_LOCALE = "en-US";

export async function initLocalization(): Promise<ReactLocalization> {
  const negotiated = negotiateLanguages(
    navigator.languages,
    AVAILABLE_LOCALES,
    { defaultLocale: DEFAULT_LOCALE }
  );

  const bundles = await createBundles(negotiated);
  return new ReactLocalization(bundles);
}
```

### Step 5: Generate l10n/bundles.ts

Handles FTL fetching, FluentBundle creation, and error logging:

```typescript
// src/l10n/bundles.ts
import { FluentBundle, FluentResource } from "@fluent/bundle";

async function fetchMessages(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/main.ftl`);
  if (!response.ok) {
    throw new Error(`Failed to fetch FTL for ${locale}: ${response.status}`);
  }
  return response.text();
}

function createBundle(locale: string, messages: string): FluentBundle {
  const bundle = new FluentBundle(locale);
  const resource = new FluentResource(messages);
  const errors = bundle.addResource(resource);
  if (errors.length) {
    errors.forEach((e) => console.warn(`FTL parse error in ${locale}:`, e));
  }
  return bundle;
}

export async function createBundles(
  locales: string[]
): Promise<FluentBundle[]> {
  return Promise.all(
    locales.map(async (locale) => {
      const messages = await fetchMessages(locale);
      return createBundle(locale, messages);
    })
  );
}
```

### Step 6: Generate l10n/switch.ts

Provides locale switching with localStorage persistence:

```typescript
// src/l10n/switch.ts
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";
import { AVAILABLE_LOCALES, DEFAULT_LOCALE } from "./index";
import { createBundles } from "./bundles";

const LOCALE_STORAGE_KEY = "preferred-locale";

export function getSavedLocale(): string | null {
  return localStorage.getItem(LOCALE_STORAGE_KEY);
}

export function saveLocalePreference(locale: string): void {
  localStorage.setItem(LOCALE_STORAGE_KEY, locale);
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
  const bundles = await createBundles(negotiated);
  return new ReactLocalization(bundles);
}
```

### Step 7: Generate LocalizedApp.tsx

The root wrapper component with async init and loading state:

```tsx
// src/components/LocalizedApp.tsx
import React, { useState, useEffect, useCallback } from "react";
import { LocalizationProvider } from "@fluent/react";
import { ReactLocalization } from "@fluent/react";
import { initLocalization } from "../l10n";
import { switchLocale, getSavedLocale } from "../l10n/switch";
import { negotiateLanguages } from "@fluent/langneg";
import { AVAILABLE_LOCALES, DEFAULT_LOCALE } from "../l10n";

interface Props {
  children: React.ReactNode;
}

export function LocalizedApp({ children }: Props) {
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

### Step 8: Wire Into App Entry Point

```tsx
// src/App.tsx or src/main.tsx
import React from "react";
import { LocalizedApp } from "./components/LocalizedApp";

function App() {
  return (
    <LocalizedApp>
      {/* Your application components here */}
    </LocalizedApp>
  );
}

export default App;
```

---

## Vanilla JS Scaffold

For projects without React, skip `@fluent/react` and use the bundle API directly:

```typescript
// src/l10n/index.ts (vanilla JS variant)
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { negotiateLanguages } from "@fluent/langneg";

const AVAILABLE_LOCALES = ["en-US", "fr", "de"];
const DEFAULT_LOCALE = "en-US";

let currentBundle: FluentBundle | null = null;

export async function initLocalization(): Promise<FluentBundle> {
  const negotiated = negotiateLanguages(
    navigator.languages,
    AVAILABLE_LOCALES,
    { defaultLocale: DEFAULT_LOCALE }
  );

  const locale = negotiated[0];
  const response = await fetch(`/locales/${locale}/main.ftl`);
  const messages = await response.text();
  const bundle = new FluentBundle(locale);
  const errors = bundle.addResource(new FluentResource(messages));
  if (errors.length) {
    errors.forEach((e) => console.warn(`FTL error:`, e));
  }
  currentBundle = bundle;
  return bundle;
}

export function t(id: string, args?: Record<string, string | number>): string {
  if (!currentBundle) return id;
  const msg = currentBundle.getMessage(id);
  if (!msg?.value) return id;
  const errors: Error[] = [];
  return currentBundle.formatPattern(msg.value, args, errors);
}
```

---

## Node.js Server Scaffold

For Express/Fastify, use `acceptedLanguages()` instead of `navigator.languages`:

```typescript
// src/l10n/server.ts
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { negotiateLanguages, acceptedLanguages } from "@fluent/langneg";
import { readFileSync } from "fs";
import { join } from "path";

const AVAILABLE_LOCALES = ["en-US", "fr", "de"];
const DEFAULT_LOCALE = "en-US";

const bundleCache = new Map<string, FluentBundle>();

function loadBundle(locale: string): FluentBundle {
  if (bundleCache.has(locale)) return bundleCache.get(locale)!;
  const ftlPath = join(__dirname, `../../locales/${locale}/main.ftl`);
  const messages = readFileSync(ftlPath, "utf-8");
  const bundle = new FluentBundle(locale);
  const errors = bundle.addResource(new FluentResource(messages));
  if (errors.length) {
    errors.forEach((e) => console.warn(`FTL error in ${locale}:`, e));
  }
  bundleCache.set(locale, bundle);
  return bundle;
}

export function getLocalization(acceptLanguageHeader: string): FluentBundle {
  const requested = acceptedLanguages(acceptLanguageHeader);
  const negotiated = negotiateLanguages(requested, AVAILABLE_LOCALES, {
    defaultLocale: DEFAULT_LOCALE,
  });
  return loadBundle(negotiated[0]);
}
```

---

## Checklist: Verify Scaffold Completeness

After generating all files, verify:

- [ ] `npm install` succeeds with no peer dependency warnings
- [ ] All FTL files end with a trailing newline
- [ ] `AVAILABLE_LOCALES` array matches directory names under `public/locales/`
- [ ] `DEFAULT_LOCALE` is included in `AVAILABLE_LOCALES`
- [ ] Every locale directory has the same set of FTL files
- [ ] `addResource()` errors are logged, not silently ignored
- [ ] `LocalizedApp` renders a loading state while translations load
- [ ] Locale switching creates a NEW `ReactLocalization` instance (not mutating)

---

## Reference Links

- [references/methods.md](references/methods.md) -- File templates, dependency list, configuration reference
- [references/examples.md](references/examples.md) -- Complete scaffolded project output with all files
- [references/anti-patterns.md](references/anti-patterns.md) -- Scaffolding mistakes to avoid

### Official Sources

- https://github.com/projectfluent/fluent.js/wiki/React-Tutorial
- https://github.com/projectfluent/fluent.js/tree/main/fluent-bundle
- https://github.com/projectfluent/fluent.js/tree/main/fluent-react
- https://github.com/projectfluent/fluent.js/tree/main/fluent-langneg
