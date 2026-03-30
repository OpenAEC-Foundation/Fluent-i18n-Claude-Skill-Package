---
name: fluent-impl-locale-loading
description: >
  Use when loading FTL locale files at runtime via fetch, dynamic import, or lazy bundle generation.
  Prevents stale translation caches and bundle loading race conditions in multi-locale apps.
  Covers async fetch loading, generator patterns, CachedSyncIterable, file organization, and code-splitting.
  Keywords: fetch FTL, FluentResource, CachedSyncIterable, lazy loading,
  bundle generation, code-splitting, load translations at runtime,
  async locale loading, dynamic import translations.
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-impl-locale-loading

## Quick Reference

### Loading Pipeline

| Step | API | Input | Output |
|------|-----|-------|--------|
| 1. Fetch FTL | `fetch(url)` | URL string | `Response` |
| 2. Extract text | `response.text()` | `Response` | FTL string |
| 3. Parse resource | `new FluentResource(text)` | FTL string | `FluentResource` |
| 4. Create bundle | `new FluentBundle(locale)` | Locale string | `FluentBundle` |
| 5. Add resource | `bundle.addResource(resource)` | `FluentResource` | `Error[]` |
| 6. Check errors | Inspect returned array | `Error[]` | Log warnings |

### File Organization Patterns

| Pattern | Structure | Use When |
|---------|-----------|----------|
| Flat files | `/locales/{locale}.ftl` | < 50 messages total |
| Directory per locale | `/locales/{locale}/messages.ftl` | 50-200 messages |
| Feature-split | `/locales/{locale}/{feature}.ftl` | > 200 messages or code-splitting needed |

### Key Dependencies

| Package | Role | Version |
|---------|------|---------|
| `@fluent/bundle` | FluentBundle, FluentResource | 0.18+ |
| `@fluent/react` | ReactLocalization, LocalizationProvider | 0.15+ |
| `cached-iterable` | CachedSyncIterable (bundled with @fluent/react) | 0.3+ |

### Critical Warnings

**NEVER** create a `new FluentResource()` for each individual message. ALWAYS batch all messages from an FTL file into a single `FluentResource`. Creating one resource per message adds unnecessary parser overhead.

**NEVER** ignore the error array returned by `bundle.addResource()`. ALWAYS log errors to detect malformed FTL before it causes silent translation failures in production.

**NEVER** block initial render by synchronously loading FTL files. ALWAYS load translations asynchronously and show a loading state until bundles are ready.

**ALWAYS** pass an `errors` array to `bundle.addResource()` and log the results. Without this, parse errors are silently swallowed and translations fail without explanation.

**ALWAYS** use `CachedSyncIterable` (or `ReactLocalization` which wraps it automatically) when using generator-based bundle creation. Without caching, the generator is consumed on first iteration and subsequent lookups return nothing.

---

## Decision Tree: File Organization

```
How many translation messages does your app have?
├─ < 50 messages
│  └─ USE flat files: /locales/en-US.ftl, /locales/fr.ftl
│     One file per locale, all messages together
│
├─ 50-200 messages
│  └─ USE directory per locale: /locales/en-US/messages.ftl
│     Single file per locale in its own directory
│     Room to split later without changing paths
│
└─ > 200 messages OR need code-splitting
   └─ USE feature-split: /locales/en-US/main.ftl, settings.ftl, errors.ftl
      ├─ Split by page or feature
      ├─ Load only what the current route needs
      └─ Multiple addResource() calls per bundle
```

## Decision Tree: Loading Strategy

```
When do you need translations available?
├─ At app startup (all locales needed immediately)
│  └─ USE eager loading with Promise.all()
│     Fetch all negotiated locales in parallel
│     Wait for all before rendering
│
├─ At app startup (only primary locale needed immediately)
│  └─ USE lazy generator with CachedSyncIterable
│     Primary locale loaded eagerly
│     Fallback locales loaded on demand when a message is missing
│
└─ On route change (different pages need different translations)
   └─ USE feature-split loading
      Load FTL file matching current route
      Add to existing bundle via addResource()
      Cache loaded resources in a Map
```

---

## Core Pattern: Async FTL Loading

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

async function fetchMessages(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/messages.ftl`);
  if (!response.ok) {
    throw new Error(`Failed to fetch FTL for ${locale}: ${response.status}`);
  }
  return response.text();
}

function createBundle(locale: string, ftlText: string): FluentBundle {
  const bundle = new FluentBundle(locale);
  const resource = new FluentResource(ftlText);
  const errors = bundle.addResource(resource);
  if (errors.length > 0) {
    errors.forEach((e) => console.warn(`FTL error in ${locale}:`, e));
  }
  return bundle;
}

async function loadBundles(locales: string[]): Promise<FluentBundle[]> {
  return Promise.all(
    locales.map(async (locale) => {
      const ftlText = await fetchMessages(locale);
      return createBundle(locale, ftlText);
    })
  );
}
```

## Core Pattern: Lazy Generator with CachedSyncIterable

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { ReactLocalization } from "@fluent/react";

function* generateBundles(
  locales: string[],
  ftlCache: Map<string, string>
): Generator<FluentBundle> {
  for (const locale of locales) {
    const ftlText = ftlCache.get(locale);
    if (!ftlText) continue;

    const bundle = new FluentBundle(locale);
    const resource = new FluentResource(ftlText);
    const errors = bundle.addResource(resource);
    if (errors.length > 0) {
      errors.forEach((e) => console.warn(`FTL error in ${locale}:`, e));
    }
    yield bundle;
  }
}

// ReactLocalization wraps the generator in CachedSyncIterable automatically
const l10n = new ReactLocalization(generateBundles(locales, cache));
```

The generator pattern is preferable for large locale sets because fallback bundles are only created when needed. `ReactLocalization` wraps any iterable in `CachedSyncIterable.from()` internally, so the generator is iterated once and then cached.

## Core Pattern: Multiple FTL Files Per Locale

```typescript
async function loadMultipleFiles(
  locale: string,
  files: string[]
): Promise<FluentBundle> {
  const bundle = new FluentBundle(locale);

  for (const file of files) {
    const response = await fetch(`/locales/${locale}/${file}`);
    const ftlText = await response.text();
    const resource = new FluentResource(ftlText);
    const errors = bundle.addResource(resource);
    if (errors.length > 0) {
      errors.forEach((e) => console.warn(`FTL error in ${locale}/${file}:`, e));
    }
  }

  return bundle;
}

// Usage: one bundle holds multiple resources
const bundle = await loadMultipleFiles("en-US", [
  "main.ftl",
  "errors.ftl",
  "settings.ftl",
]);
```

A single `FluentBundle` accepts multiple `FluentResource` instances via repeated `addResource()` calls. All messages from all resources become available in the same bundle.

---

## File Organization Convention

### Feature-Split Structure

```
public/
└── locales/
    ├── en-US/
    │   ├── main.ftl          # Core UI: navigation, common buttons
    │   ├── auth.ftl           # Login, registration, password reset
    │   ├── settings.ftl       # Settings page
    │   ├── errors.ftl         # Error messages
    │   └── dashboard.ftl      # Dashboard-specific messages
    ├── fr/
    │   ├── main.ftl
    │   ├── auth.ftl
    │   ├── settings.ftl
    │   ├── errors.ftl
    │   └── dashboard.ftl
    └── de/
        ├── main.ftl
        ├── auth.ftl
        ├── settings.ftl
        ├── errors.ftl
        └── dashboard.ftl
```

### FTL File Naming Rules

- **ALWAYS** use lowercase filenames: `main.ftl`, not `Main.ftl`
- **ALWAYS** use BCP 47 locale identifiers for directory names: `en-US`, `fr-CA`, `zh-Hans`
- **ALWAYS** keep the same file names across all locales — every locale directory MUST mirror the same set of `.ftl` files
- **NEVER** mix messages from different features in one FTL file when using feature-split

---

## Bundle Caching Strategy

```typescript
const bundleCache = new Map<string, FluentBundle>();

async function getOrCreateBundle(locale: string): Promise<FluentBundle> {
  if (bundleCache.has(locale)) {
    return bundleCache.get(locale)!;
  }

  const ftlText = await fetchMessages(locale);
  const bundle = createBundle(locale, ftlText);
  bundleCache.set(locale, bundle);
  return bundle;
}
```

**ALWAYS** cache `FluentBundle` instances in a `Map<string, FluentBundle>` when locales may be re-requested (e.g., locale switching). Creating bundles and parsing FTL is not free — avoid repeated parsing of the same content.

---

## Reference Links

- [references/methods.md](references/methods.md) -- API methods for fetch, FluentResource, addResource, and bundle creation
- [references/examples.md](references/examples.md) -- Complete working examples for async loading, lazy generators, caching, and feature-split loading
- [references/anti-patterns.md](references/anti-patterns.md) -- What NOT to do when loading FTL files

### Official Sources

- https://github.com/projectfluent/fluent.js/wiki/React-Tutorial
- https://github.com/projectfluent/fluent.js/wiki/ReactLocalization
- https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/bundle.ts
- https://github.com/projectfluent/fluent.js/blob/main/fluent-react/src/localization.ts
