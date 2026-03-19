---
name: fluent-core-langneg
description: >
  Use when implementing language negotiation or locale fallback chains with @fluent/langneg. Prevents wrong strategy selection and broken fallback behavior.
  Covers negotiateLanguages() strategies, BCP 47 locale handling, acceptedLanguages() for HTTP headers, and fallback chain construction.
  Keywords: negotiateLanguages, @fluent/langneg, BCP 47, Accept-Language, locale fallback, filtering, matching, lookup.
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/langneg 0.7+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-core-langneg

## Quick Reference

### Package Overview

| Property | Value |
|----------|-------|
| Package | `@fluent/langneg` |
| Version | 0.7.0 |
| License | Apache-2.0 |
| Dependencies | None (zero runtime dependencies) |
| Install | `npm install @fluent/langneg` |

### Exports

| Export | Type | Purpose |
|--------|------|---------|
| `negotiateLanguages` | Function | Match user-requested locales against available locales |
| `acceptedLanguages` | Function | Parse HTTP `Accept-Language` header into sorted locale array |
| `filterMatches` | Function | Low-level matching engine (exported but rarely used directly) |

### Strategy Comparison

Given requested `["de-DE", "fr-FR"]` and available `["it", "de", "en-US", "fr-CA", "de-DE", "fr", "de-AU"]`:

| Strategy | Result | Behavior |
|----------|--------|----------|
| `"filtering"` (default) | `["de-DE", "de", "fr", "fr-CA"]` | ALL matches for ALL requested locales |
| `"matching"` | `["de-DE", "fr"]` | BEST single match per requested locale |
| `"lookup"` | `["de-DE"]` | SINGLE best match across all requested locales |

### Critical Warnings

> **NEVER** use the `"lookup"` strategy without providing `defaultLocale` — the function throws an `Error` if `defaultLocale` is undefined with lookup strategy.

> **NEVER** pass non-BCP 47 locale strings (e.g., `"english"`, `"ENUS"`) — the negotiation algorithm expects standard BCP 47 tags like `"en-US"`, `"fr-CA"`, `"sr-Latn"`.

> **ALWAYS** provide `defaultLocale` in production code — without it, negotiation can return an empty array if no matches are found, leaving the user with no translations.

---

## negotiateLanguages()

### Full Signature

```typescript
import { negotiateLanguages } from "@fluent/langneg";

function negotiateLanguages(
  requestedLocales: Readonly<Array<string>>,
  availableLocales: Readonly<Array<string>>,
  options?: {
    strategy?: "filtering" | "matching" | "lookup";
    defaultLocale?: string;
  }
): Array<string>
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `requestedLocales` | `Readonly<Array<string>>` | User's preferred locales (e.g., `navigator.languages`) |
| `availableLocales` | `Readonly<Array<string>>` | Locales the application supports |
| `options.strategy` | `"filtering" \| "matching" \| "lookup"` | Negotiation algorithm (default: `"filtering"`) |
| `options.defaultLocale` | `string` | Fallback locale; appended to results if not already present |

### Return Value

`Array<string>` — locale identifiers ordered by preference.

### defaultLocale Behavior

- For `"filtering"` and `"matching"`: `defaultLocale` is appended to results if provided and not already present in the match results
- For `"lookup"`: `defaultLocale` is **REQUIRED** — the function throws `Error` if omitted; pushed to results if no matches found

### Client-Side Usage

```typescript
import { negotiateLanguages } from "@fluent/langneg";

const AVAILABLE_LOCALES = ["en-US", "fr", "de", "ja"];

const userLocales = negotiateLanguages(
  navigator.languages,        // e.g., ["de-DE", "en-US"]
  AVAILABLE_LOCALES,
  { defaultLocale: "en-US" }
);
// With filtering: ["de", "en-US"] — "de-DE" matched "de" via subtag matching
```

### Server-Side Usage

```typescript
import { acceptedLanguages, negotiateLanguages } from "@fluent/langneg";

const requested = acceptedLanguages(req.headers["accept-language"] || "");
const supported = negotiateLanguages(requested, AVAILABLE_LOCALES, {
  defaultLocale: "en-US",
});
```

---

## Strategies In Depth

### Filtering (Default)

Returns ALL available locales that match ANY requested locale. Most permissive — produces the longest fallback chain.

```typescript
const result = negotiateLanguages(
  ["de-DE", "fr-FR"],
  ["it", "de", "en-US", "fr-CA", "de-DE", "fr", "de-AU"],
  { strategy: "filtering", defaultLocale: "en-US" }
);
// Result: ["de-DE", "de", "fr", "fr-CA", "en-US"]
// "de-AU" excluded — no "de-AU" in requested; "en-US" appended as defaultLocale
```

ALWAYS use filtering when building a fallback chain for `FluentBundle` sequences — it maximizes translation coverage.

### Matching

Returns the BEST single match for EACH requested locale. One result per request entry.

```typescript
const result = negotiateLanguages(
  ["de-DE", "fr-FR"],
  ["it", "de", "en-US", "fr-CA", "de-DE", "fr", "de-AU"],
  { strategy: "matching", defaultLocale: "en-US" }
);
// Result: ["de-DE", "fr", "en-US"]
// One match per requested locale; defaultLocale appended
```

ALWAYS use matching when you need a concise preference list without redundant variants.

### Lookup

Returns the SINGLE best match across ALL requested locales. Most restrictive — exactly one result.

```typescript
const result = negotiateLanguages(
  ["de-DE", "fr-FR"],
  ["it", "de", "en-US", "fr-CA", "de-DE", "fr", "de-AU"],
  { strategy: "lookup", defaultLocale: "en-US" }
);
// Result: ["de-DE"]
// Single best match; defaultLocale used only if zero matches found
```

ALWAYS use lookup when you need exactly one locale (e.g., selecting a date format library, choosing a single UI direction).

---

## Strategy Selection Decision Tree

```
Need to select locales?
├── Need a full fallback chain for translations?
│   └── YES → Use "filtering" (default)
│       Returns all matching locales for maximum coverage
│
├── Need one best locale per user preference?
│   └── YES → Use "matching"
│       Returns best match per requested locale
│
└── Need exactly one locale for the entire app?
    └── YES → Use "lookup"
        Returns single best match
        ⚠ REQUIRES defaultLocale or throws Error
```

---

## acceptedLanguages()

Parses HTTP `Accept-Language` headers into a sorted array of locale strings.

```typescript
import { acceptedLanguages } from "@fluent/langneg";

const locales = acceptedLanguages("fr-CH, fr;q=0.9, en;q=0.8, de;q=0.7, *;q=0.5");
// Result: ["fr-CH", "fr", "en", "de", "*"] — sorted by q-value descending
```

- Parses quality values (`q=`) from the header string
- Sorts by descending quality, preserving order for equal weights
- ALWAYS use on the server instead of `navigator.languages` (which is unavailable in Node.js)
- Throws `TypeError` if the argument is not a string

---

## BCP 47 Locale Identifiers

All locale strings in `@fluent/langneg` follow BCP 47 (IETF language tag standard):

| Format | Example | Components |
|--------|---------|------------|
| Language | `en`, `fr`, `de` | ISO 639 language code |
| Language + Region | `en-US`, `fr-CA`, `de-DE` | Language + ISO 3166 country code |
| Language + Script | `sr-Latn`, `zh-Hans` | Language + ISO 15924 script code |

### Subtag Matching

The negotiation algorithms perform subtag matching — a requested `"de-DE"` will match an available `"de"` as a fallback. The library includes minimal likely-subtags data to resolve generic locales (e.g., requesting `"en"` can match both `"en-GB"` and `"en-US"`).

Input locales are coerced to strings internally via `Array.from(...).map(String)`.

---

## Integration with @fluent/react

ALWAYS use `negotiateLanguages` to determine the locale order before creating `FluentBundle` instances:

```typescript
import { negotiateLanguages } from "@fluent/langneg";
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { ReactLocalization } from "@fluent/react";

const negotiated = negotiateLanguages(
  navigator.languages,
  ["en-US", "fr", "de"],
  { defaultLocale: "en-US" }
);

const bundles = negotiated.map((locale) => {
  const bundle = new FluentBundle(locale);
  bundle.addResource(new FluentResource(MESSAGES[locale]));
  return bundle;
});

const l10n = new ReactLocalization(bundles);
```

The order returned by `negotiateLanguages` determines the fallback chain — the first bundle is the preferred locale, subsequent bundles serve as fallbacks for missing translations.

---

## Reference Links

- [references/methods.md](references/methods.md) — Full API signatures for negotiateLanguages, acceptedLanguages, filterMatches
- [references/examples.md](references/examples.md) — Client-side, server-side, and per-strategy usage examples
- [references/anti-patterns.md](references/anti-patterns.md) — Common mistakes and what to do instead

### Official Sources

- https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/README.md
- https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/negotiate_languages.ts
- https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/accepted_languages.ts
- https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/index.ts
