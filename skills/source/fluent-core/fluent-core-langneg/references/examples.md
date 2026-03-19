# Examples — @fluent/langneg

## Client-Side Negotiation (Browser)

### Basic Setup with navigator.languages

```typescript
import { negotiateLanguages } from "@fluent/langneg";

const AVAILABLE_LOCALES = ["en-US", "fr", "de", "ja", "zh-Hans"];
const DEFAULT_LOCALE = "en-US";

// navigator.languages returns the user's browser language preferences
// e.g., ["de-DE", "en-US", "en"]
const userLocales = negotiateLanguages(
  navigator.languages,
  AVAILABLE_LOCALES,
  { defaultLocale: DEFAULT_LOCALE }
);
// Result with filtering: ["de", "en-US"] + defaultLocale appended if not present
```

### Locale Switching in React

```typescript
import { useState, useCallback } from "react";
import { negotiateLanguages } from "@fluent/langneg";

const AVAILABLE_LOCALES = ["en-US", "fr", "de"];

function App() {
  const [currentLocales, setCurrentLocales] = useState(() =>
    negotiateLanguages(navigator.languages, AVAILABLE_LOCALES, {
      defaultLocale: "en-US",
    })
  );

  const switchLocale = useCallback((locale: string) => {
    const negotiated = negotiateLanguages(
      [locale],
      AVAILABLE_LOCALES,
      { defaultLocale: "en-US" }
    );
    setCurrentLocales(negotiated);
  }, []);

  // Pass currentLocales to bundle generation and ReactLocalization
  // ...
}
```

---

## Server-Side Negotiation (Node.js / Express)

### Parsing Accept-Language Header

```typescript
import { acceptedLanguages, negotiateLanguages } from "@fluent/langneg";

const AVAILABLE_LOCALES = ["en-US", "fr", "de", "ja"];
const DEFAULT_LOCALE = "en-US";

function getServerLocales(req: Request): string[] {
  // Parse the Accept-Language header
  // e.g., "fr-CH, fr;q=0.9, en;q=0.8, de;q=0.7, *;q=0.5"
  const requested = acceptedLanguages(req.headers["accept-language"] || "");
  // → ["fr-CH", "fr", "en", "de", "*"]

  // Negotiate against available locales
  return negotiateLanguages(requested, AVAILABLE_LOCALES, {
    defaultLocale: DEFAULT_LOCALE,
  });
  // → ["fr", "de", "en-US"]
}
```

### Express Middleware Pattern

```typescript
import { acceptedLanguages, negotiateLanguages } from "@fluent/langneg";

const AVAILABLE_LOCALES = ["en-US", "fr", "de"];

function localeMiddleware(req, res, next) {
  const header = req.headers["accept-language"] || "";
  const requested = acceptedLanguages(header);
  req.locales = negotiateLanguages(requested, AVAILABLE_LOCALES, {
    defaultLocale: "en-US",
  });
  next();
}

app.use(localeMiddleware);

app.get("/", (req, res) => {
  // req.locales is now ["fr", "en-US"] etc.
  const bundles = createBundles(req.locales);
  // ...
});
```

### SSR with React (Server-Side Rendering)

```typescript
import { acceptedLanguages, negotiateLanguages } from "@fluent/langneg";
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { ReactLocalization } from "@fluent/react";
import { renderToString } from "react-dom/server";

async function handleSSR(req, res) {
  // 1. Negotiate locales from HTTP header
  const requested = acceptedLanguages(req.headers["accept-language"] || "");
  const locales = negotiateLanguages(requested, AVAILABLE_LOCALES, {
    defaultLocale: "en-US",
  });

  // 2. Load all bundles BEFORE rendering (no async fallback support)
  const bundles = await Promise.all(
    locales.map(async (locale) => {
      const ftl = await loadFTLFile(locale);
      const bundle = new FluentBundle(locale);
      bundle.addResource(new FluentResource(ftl));
      return bundle;
    })
  );

  // 3. Create localization with custom parseMarkup for server
  const l10n = new ReactLocalization(bundles, parseMarkup);

  // 4. Render
  const html = renderToString(
    <LocalizationProvider l10n={l10n}>
      <App />
    </LocalizationProvider>
  );
  res.send(html);
}
```

---

## All Three Strategies — Side by Side

### Setup

```typescript
import { negotiateLanguages } from "@fluent/langneg";

const requested = ["de-DE", "fr-FR"];
const available = ["it", "de", "en-US", "fr-CA", "de-DE", "fr", "de-AU"];
const defaultLocale = "en-US";
```

### Filtering (Default)

```typescript
const result = negotiateLanguages(requested, available, {
  strategy: "filtering",
  defaultLocale,
});
// → ["de-DE", "de", "fr", "fr-CA", "en-US"]
//
// Explanation:
// - "de-DE" requested → matches "de-DE" (exact) and "de" (subtag)
// - "fr-FR" requested → matches "fr" (subtag) and "fr-CA" (same language)
// - "en-US" appended as defaultLocale (not in matches)
// - "it" and "de-AU" not matched by any requested locale
```

### Matching

```typescript
const result = negotiateLanguages(requested, available, {
  strategy: "matching",
  defaultLocale,
});
// → ["de-DE", "fr", "en-US"]
//
// Explanation:
// - "de-DE" requested → best match is "de-DE" (exact)
// - "fr-FR" requested → best match is "fr" (closest)
// - "en-US" appended as defaultLocale
```

### Lookup

```typescript
const result = negotiateLanguages(requested, available, {
  strategy: "lookup",
  defaultLocale,
});
// → ["de-DE"]
//
// Explanation:
// - Finds first match across all requested: "de-DE" (exact match)
// - Returns single result only
// - defaultLocale would be used if zero matches found
```

---

## Subtag Matching Examples

### Language-only matches region variant

```typescript
negotiateLanguages(["en"], ["en-US", "en-GB"], { defaultLocale: "en-US" });
// → ["en-US", "en-GB"] (filtering — both match "en")
```

### Region-specific falls back to language

```typescript
negotiateLanguages(["de-AT"], ["de", "fr"], { defaultLocale: "en-US" });
// → ["de", "en-US"] (filtering — "de-AT" matches "de" via subtag)
```

### Script subtag matching

```typescript
negotiateLanguages(["sr-Latn"], ["sr-Latn", "sr-Cyrl", "sr"], {
  defaultLocale: "en-US",
});
// → ["sr-Latn", "sr", "en-US"] (filtering — exact + language match)
```

---

## Building a Complete Fallback Chain

The recommended pattern for production applications:

```typescript
import { negotiateLanguages } from "@fluent/langneg";
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { ReactLocalization } from "@fluent/react";

const AVAILABLE_LOCALES = ["en-US", "fr", "de", "ja"];
const DEFAULT_LOCALE = "en-US";

// Step 1: Negotiate with filtering for maximum fallback coverage
const locales = negotiateLanguages(
  navigator.languages,
  AVAILABLE_LOCALES,
  { defaultLocale: DEFAULT_LOCALE }
);

// Step 2: Create bundles in negotiated order
// First bundle = preferred locale, last = ultimate fallback
function* generateBundles(locales: string[]) {
  for (const locale of locales) {
    const bundle = new FluentBundle(locale);
    bundle.addResource(new FluentResource(MESSAGES[locale]));
    yield bundle;
  }
}

// Step 3: Pass to ReactLocalization
// Missing translations in preferred locale fall through to next bundle
const l10n = new ReactLocalization(generateBundles(locales));
```

---

## Source References

- Client-side pattern: https://github.com/projectfluent/fluent.js/wiki/React-Tutorial
- Server-side pattern: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/accepted_languages.ts
- Strategy examples: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/README.md
