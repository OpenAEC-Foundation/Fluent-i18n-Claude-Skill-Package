# Anti-Patterns — @fluent/langneg

## AP-1: Missing defaultLocale

### Problem

Omitting `defaultLocale` can result in an empty array when no matches are found, leaving the application with zero translations.

```typescript
// WRONG — no defaultLocale
const locales = negotiateLanguages(
  ["ko-KR"],           // user wants Korean
  ["en-US", "fr", "de"] // app does not have Korean
);
// Result: [] — empty array, no translations available
```

### Solution

ALWAYS provide `defaultLocale` in production code:

```typescript
// CORRECT — defaultLocale ensures at least one locale
const locales = negotiateLanguages(
  ["ko-KR"],
  ["en-US", "fr", "de"],
  { defaultLocale: "en-US" }
);
// Result: ["en-US"] — user gets English as fallback
```

---

## AP-2: Lookup Strategy Without defaultLocale

### Problem

The `"lookup"` strategy REQUIRES `defaultLocale`. Omitting it throws a runtime `Error`.

```typescript
// WRONG — throws Error at runtime
const locale = negotiateLanguages(
  navigator.languages,
  ["en-US", "fr"],
  { strategy: "lookup" }  // no defaultLocale!
);
// Error: Lookup strategy requires a defaultLocale
```

### Solution

ALWAYS provide `defaultLocale` when using lookup:

```typescript
// CORRECT
const locale = negotiateLanguages(
  navigator.languages,
  ["en-US", "fr"],
  { strategy: "lookup", defaultLocale: "en-US" }
);
```

---

## AP-3: Wrong Strategy for the Use Case

### Problem — Using Lookup for Fallback Chains

Using `"lookup"` when building a translation fallback chain results in only one locale, eliminating all fallback translations.

```typescript
// WRONG — only one locale, no fallback chain
const locales = negotiateLanguages(
  ["de-DE", "en-US"],
  ["de", "en-US", "fr"],
  { strategy: "lookup", defaultLocale: "en-US" }
);
// Result: ["de"] — if "de" is missing a translation, there is no fallback
```

### Solution

Use `"filtering"` (default) for fallback chains to maximize translation coverage:

```typescript
// CORRECT — full fallback chain
const locales = negotiateLanguages(
  ["de-DE", "en-US"],
  ["de", "en-US", "fr"],
  { defaultLocale: "en-US" }
);
// Result: ["de", "en-US"] — missing German translations fall through to English
```

### Problem — Using Filtering When One Locale is Needed

Using `"filtering"` when you need exactly one locale (e.g., for a date formatting library) returns multiple locales unnecessarily.

```typescript
// WRONG — filtering returns multiple locales when you only need one
const locales = negotiateLanguages(
  ["de-DE", "en-US"],
  ["de", "de-DE", "en-US"],
  { defaultLocale: "en-US" }
);
// Result: ["de-DE", "de", "en-US"] — which one do you pick for Intl.DateTimeFormat?
```

### Solution

Use `"lookup"` when you need exactly one locale:

```typescript
// CORRECT — single locale for non-Fluent APIs
const [locale] = negotiateLanguages(
  ["de-DE", "en-US"],
  ["de", "de-DE", "en-US"],
  { strategy: "lookup", defaultLocale: "en-US" }
);
// locale: "de-DE" — use directly with Intl.DateTimeFormat
```

---

## AP-4: Using navigator.languages on the Server

### Problem

`navigator.languages` does not exist in Node.js. Attempting to use it on the server causes a `ReferenceError`.

```typescript
// WRONG — navigator is undefined on the server
const locales = negotiateLanguages(
  navigator.languages,  // ReferenceError: navigator is not defined
  AVAILABLE_LOCALES,
  { defaultLocale: "en-US" }
);
```

### Solution

ALWAYS use `acceptedLanguages()` to parse the HTTP `Accept-Language` header on the server:

```typescript
import { acceptedLanguages, negotiateLanguages } from "@fluent/langneg";

// CORRECT — server-side locale detection
const requested = acceptedLanguages(req.headers["accept-language"] || "");
const locales = negotiateLanguages(requested, AVAILABLE_LOCALES, {
  defaultLocale: "en-US",
});
```

---

## AP-5: Hardcoding Locale Lists Instead of Negotiating

### Problem

Hardcoding a locale order bypasses the user's actual preferences and removes the ability to handle subtag matching.

```typescript
// WRONG — ignores user preferences entirely
const locales = ["en-US", "fr", "de"];
const bundles = locales.map((l) => createBundle(l));
```

### Solution

ALWAYS negotiate against the user's actual locale preferences:

```typescript
// CORRECT — respects user preferences with intelligent matching
const locales = negotiateLanguages(
  navigator.languages,
  ["en-US", "fr", "de"],
  { defaultLocale: "en-US" }
);
const bundles = locales.map((l) => createBundle(l));
```

---

## AP-6: Non-BCP 47 Locale Strings

### Problem

Passing locale strings that do not follow BCP 47 format produces no matches.

```typescript
// WRONG — non-standard locale identifiers
const locales = negotiateLanguages(
  ["english", "german"],          // not BCP 47
  ["en-US", "de"],
  { defaultLocale: "en-US" }
);
// Result: ["en-US"] — only defaultLocale, no actual matches

// ALSO WRONG — underscore instead of hyphen
const locales2 = negotiateLanguages(
  ["en_US"],                       // underscore format (POSIX, not BCP 47)
  ["en-US"],
  { defaultLocale: "en-US" }
);
// Result: ["en-US"] — matched only because it is defaultLocale, not by negotiation
```

### Solution

ALWAYS use BCP 47 format with hyphens:

```typescript
// CORRECT — standard BCP 47 identifiers
const locales = negotiateLanguages(
  ["en-US", "de-DE"],
  ["en-US", "de"],
  { defaultLocale: "en-US" }
);
```

---

## AP-7: Not Passing Empty String Fallback to acceptedLanguages

### Problem

If the `Accept-Language` header is `undefined` (e.g., missing from request), passing it directly to `acceptedLanguages()` throws a `TypeError`.

```typescript
// WRONG — header may be undefined
const requested = acceptedLanguages(req.headers["accept-language"]);
// TypeError if header is undefined
```

### Solution

ALWAYS provide an empty string fallback:

```typescript
// CORRECT — handles missing header gracefully
const requested = acceptedLanguages(req.headers["accept-language"] || "");
// Returns [] for empty string, which is safe to pass to negotiateLanguages
```

---

## Source References

- Strategy behavior: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/README.md
- acceptedLanguages: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/accepted_languages.ts
- negotiateLanguages: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/negotiate_languages.ts
