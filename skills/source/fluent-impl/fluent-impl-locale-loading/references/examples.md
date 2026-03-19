# examples.md — Locale Loading Working Examples

## Example 1: Async Loading with React

Complete setup that fetches FTL files from a server and renders when ready.

### FTL Files

```ftl
# public/locales/en-US/messages.ftl
app-title = My Application
welcome = Welcome, { $userName }!
items-count = { $count ->
    [one] You have one item.
   *[other] You have { $count } items.
}
```

```ftl
# public/locales/fr/messages.ftl
app-title = Mon Application
welcome = Bienvenue, { $userName } !
items-count = { $count ->
    [one] Vous avez un article.
   *[other] Vous avez { $count } articles.
}
```

### TypeScript Loading Module

```typescript
// src/l10n.ts
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";

const AVAILABLE_LOCALES = ["en-US", "fr", "de"];
const DEFAULT_LOCALE = "en-US";

async function fetchFtl(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/messages.ftl`);
  if (!response.ok) {
    console.warn(`Missing locale file for ${locale}`);
    return "";
  }
  return response.text();
}

function createBundle(locale: string, ftlText: string): FluentBundle {
  const bundle = new FluentBundle(locale);
  const resource = new FluentResource(ftlText);
  const errors = bundle.addResource(resource);
  if (errors.length > 0) {
    errors.forEach((e) => console.warn(`FTL parse error [${locale}]:`, e));
  }
  return bundle;
}

export async function initLocalization(): Promise<ReactLocalization> {
  const negotiated = negotiateLanguages(
    navigator.languages,
    AVAILABLE_LOCALES,
    { defaultLocale: DEFAULT_LOCALE }
  );

  const bundles = await Promise.all(
    negotiated.map(async (locale) => {
      const ftlText = await fetchFtl(locale);
      return createBundle(locale, ftlText);
    })
  );

  return new ReactLocalization(bundles);
}
```

### React App Component

```tsx
// src/App.tsx
import React, { useState, useEffect } from "react";
import { LocalizationProvider, Localized } from "@fluent/react";
import { ReactLocalization } from "@fluent/react";
import { initLocalization } from "./l10n";

function App() {
  const [l10n, setL10n] = useState<ReactLocalization | null>(null);

  useEffect(() => {
    initLocalization().then(setL10n);
  }, []);

  if (!l10n) {
    return <div>Loading translations...</div>;
  }

  return (
    <LocalizationProvider l10n={l10n}>
      <Localized id="app-title">
        <h1>My Application</h1>
      </Localized>
      <Localized id="welcome" vars={{ userName: "Anna" }}>
        <p>Welcome, Anna!</p>
      </Localized>
    </LocalizationProvider>
  );
}

export default App;
```

---

## Example 2: Lazy Generator Pattern

Bundles are created on demand. The primary locale is always available; fallback locales are only instantiated when a message lookup misses.

```typescript
// src/l10n-lazy.ts
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { ReactLocalization } from "@fluent/react";

// Pre-fetched FTL texts stored in a Map
const ftlCache = new Map<string, string>();

export async function prefetchLocales(locales: string[]): Promise<void> {
  await Promise.all(
    locales.map(async (locale) => {
      if (ftlCache.has(locale)) return;
      const response = await fetch(`/locales/${locale}/messages.ftl`);
      if (response.ok) {
        ftlCache.set(locale, await response.text());
      }
    })
  );
}

function* generateBundles(locales: string[]): Generator<FluentBundle> {
  for (const locale of locales) {
    const ftlText = ftlCache.get(locale);
    if (!ftlText) {
      console.warn(`No cached FTL for ${locale}, skipping`);
      continue;
    }

    const bundle = new FluentBundle(locale);
    const resource = new FluentResource(ftlText);
    const errors = bundle.addResource(resource);
    if (errors.length > 0) {
      errors.forEach((e) => console.warn(`FTL error [${locale}]:`, e));
    }
    yield bundle;
  }
}

export function createLocalization(locales: string[]): ReactLocalization {
  // ReactLocalization wraps the generator in CachedSyncIterable automatically
  return new ReactLocalization(generateBundles(locales));
}
```

Usage:
```typescript
// At app startup
await prefetchLocales(["en-US", "fr", "de"]);
const l10n = createLocalization(["fr", "en-US"]); // fr preferred, en-US fallback
```

---

## Example 3: Multiple FTL Files Per Locale (Feature-Split)

Each feature/page has its own FTL file. All files for a locale are loaded into a single bundle.

```typescript
// src/l10n-feature.ts
import { FluentBundle, FluentResource } from "@fluent/bundle";

interface FeatureConfig {
  locale: string;
  files: string[];
}

async function loadFeatureBundle(config: FeatureConfig): Promise<FluentBundle> {
  const bundle = new FluentBundle(config.locale);

  const fetchResults = await Promise.all(
    config.files.map(async (file) => {
      const url = `/locales/${config.locale}/${file}`;
      const response = await fetch(url);
      if (!response.ok) {
        console.warn(`Missing FTL: ${url}`);
        return null;
      }
      return { file, text: await response.text() };
    })
  );

  for (const result of fetchResults) {
    if (!result) continue;
    const resource = new FluentResource(result.text);
    const errors = bundle.addResource(resource);
    if (errors.length > 0) {
      errors.forEach((e) =>
        console.warn(`FTL error [${config.locale}/${result.file}]:`, e)
      );
    }
  }

  return bundle;
}

// Usage: load core + page-specific FTL
const bundle = await loadFeatureBundle({
  locale: "en-US",
  files: ["main.ftl", "errors.ftl", "dashboard.ftl"],
});
```

### Adding Features On Demand

```typescript
// src/l10n-dynamic.ts
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { ReactLocalization } from "@fluent/react";

class LocaleManager {
  private bundles = new Map<string, FluentBundle>();
  private loadedFeatures = new Map<string, Set<string>>();

  async ensureBundle(locale: string): Promise<FluentBundle> {
    if (!this.bundles.has(locale)) {
      this.bundles.set(locale, new FluentBundle(locale));
      this.loadedFeatures.set(locale, new Set());
    }
    return this.bundles.get(locale)!;
  }

  async loadFeature(locale: string, feature: string): Promise<void> {
    const loaded = this.loadedFeatures.get(locale);
    if (loaded?.has(feature)) return; // Already loaded

    const bundle = await this.ensureBundle(locale);
    const response = await fetch(`/locales/${locale}/${feature}.ftl`);
    if (!response.ok) {
      console.warn(`Missing feature FTL: ${locale}/${feature}.ftl`);
      return;
    }

    const ftlText = await response.text();
    const resource = new FluentResource(ftlText);
    const errors = bundle.addResource(resource);
    if (errors.length > 0) {
      errors.forEach((e) =>
        console.warn(`FTL error [${locale}/${feature}]:`, e)
      );
    }

    loaded?.add(feature);
  }

  getLocalization(locales: string[]): ReactLocalization {
    const bundles = locales
      .map((l) => this.bundles.get(l))
      .filter((b): b is FluentBundle => b !== undefined);
    return new ReactLocalization(bundles);
  }
}
```

Usage with route-based loading:
```typescript
const manager = new LocaleManager();

// On app startup: load core translations
await manager.loadFeature("en-US", "main");
await manager.loadFeature("en-US", "errors");

// On navigation to settings page:
await manager.loadFeature("en-US", "settings");
const l10n = manager.getLocalization(["en-US"]);
```

---

## Example 4: Bundle Caching with Locale Switching

```typescript
// src/l10n-cached.ts
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";

const AVAILABLE_LOCALES = ["en-US", "fr", "de", "es"];
const DEFAULT_LOCALE = "en-US";

const bundleCache = new Map<string, FluentBundle>();

async function getBundle(locale: string): Promise<FluentBundle> {
  if (bundleCache.has(locale)) {
    return bundleCache.get(locale)!;
  }

  const response = await fetch(`/locales/${locale}/messages.ftl`);
  const ftlText = response.ok ? await response.text() : "";
  const bundle = new FluentBundle(locale);
  const resource = new FluentResource(ftlText);
  const errors = bundle.addResource(resource);
  if (errors.length > 0) {
    errors.forEach((e) => console.warn(`FTL error [${locale}]:`, e));
  }

  bundleCache.set(locale, bundle);
  return bundle;
}

export async function switchLocale(
  userLocale: string
): Promise<ReactLocalization> {
  const negotiated = negotiateLanguages(
    [userLocale],
    AVAILABLE_LOCALES,
    { defaultLocale: DEFAULT_LOCALE }
  );

  const bundles = await Promise.all(negotiated.map(getBundle));
  return new ReactLocalization(bundles);
}
```

Usage in React:
```tsx
function App() {
  const [l10n, setL10n] = useState<ReactLocalization | null>(null);

  const handleLocaleChange = async (locale: string) => {
    const newL10n = await switchLocale(locale);
    setL10n(newL10n);
  };

  // ... render with LocalizationProvider
}
```

The `bundleCache` ensures that switching back to a previously loaded locale does NOT re-fetch or re-parse the FTL content.

---

## Example 5: Server-Side Loading (Node.js)

```typescript
// server/l10n.ts
import { readFileSync } from "fs";
import { join } from "path";
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { acceptedLanguages, negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";

const LOCALES_DIR = join(__dirname, "../public/locales");
const AVAILABLE_LOCALES = ["en-US", "fr", "de"];
const DEFAULT_LOCALE = "en-US";

function loadBundleFromDisk(locale: string): FluentBundle {
  const ftlPath = join(LOCALES_DIR, locale, "messages.ftl");
  const ftlText = readFileSync(ftlPath, "utf-8");
  const bundle = new FluentBundle(locale);
  const resource = new FluentResource(ftlText);
  const errors = bundle.addResource(resource);
  if (errors.length > 0) {
    errors.forEach((e) => console.warn(`FTL error [${locale}]:`, e));
  }
  return bundle;
}

export function createServerLocalization(
  acceptLanguageHeader: string
): ReactLocalization {
  const requested = acceptedLanguages(acceptLanguageHeader);
  const negotiated = negotiateLanguages(requested, AVAILABLE_LOCALES, {
    defaultLocale: DEFAULT_LOCALE,
  });

  const bundles = negotiated.map(loadBundleFromDisk);
  return new ReactLocalization(bundles);
}
```

Usage in Express:
```typescript
app.get("*", (req, res) => {
  const l10n = createServerLocalization(
    req.headers["accept-language"] || ""
  );
  // Use l10n for server-side rendering
});
```
