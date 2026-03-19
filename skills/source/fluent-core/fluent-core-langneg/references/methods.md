# API Methods Reference — @fluent/langneg

## negotiateLanguages()

### Full Signature

```typescript
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

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `requestedLocales` | `Readonly<Array<string>>` | Yes | — | User's preferred locales in BCP 47 format |
| `availableLocales` | `Readonly<Array<string>>` | Yes | — | Locales the application has translations for |
| `options` | `object` | No | `{}` | Configuration object |
| `options.strategy` | `"filtering" \| "matching" \| "lookup"` | No | `"filtering"` | Negotiation algorithm to use |
| `options.defaultLocale` | `string` | No | `undefined` | Fallback locale appended if not in results |

### Return Value

`Array<string>` — Matched locale identifiers ordered by preference. May be empty if no matches found and no `defaultLocale` provided.

### Implementation Logic

1. Calls `filterMatches(requestedLocales, availableLocales, strategy)` internally
2. For `"lookup"` strategy: throws `Error` if `defaultLocale` is `undefined`
3. For `"lookup"` strategy: pushes `defaultLocale` if no matches found
4. For `"filtering"` and `"matching"`: appends `defaultLocale` to results if provided and not already present

### Error Conditions

| Condition | Error Type | Message |
|-----------|-----------|---------|
| `strategy === "lookup"` and `defaultLocale` is `undefined` | `Error` | Lookup strategy requires a defaultLocale |

---

## acceptedLanguages()

### Full Signature

```typescript
function acceptedLanguages(header: string): Array<string>
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `header` | `string` | Yes | Raw HTTP `Accept-Language` header value |

### Return Value

`Array<string>` — Locale identifiers sorted by descending quality value (`q=`). Equal quality values preserve their original order.

### Parsing Rules

- Default quality value is `1.0` when `q=` is omitted
- Quality values range from `0.0` to `1.0`
- Wildcard `*` is included in results (represents "any language")
- Whitespace around values is trimmed

### Error Conditions

| Condition | Error Type |
|-----------|-----------|
| Argument is not a string | `TypeError` |

### Examples

```typescript
import { acceptedLanguages } from "@fluent/langneg";

// Full header with quality values
acceptedLanguages("fr-CH, fr;q=0.9, en;q=0.8, de;q=0.7, *;q=0.5");
// → ["fr-CH", "fr", "en", "de", "*"]

// Simple header without quality values (all default to q=1.0)
acceptedLanguages("en-US, en, nl");
// → ["en-US", "en", "nl"]

// Single locale
acceptedLanguages("de-DE");
// → ["de-DE"]

// Empty string
acceptedLanguages("");
// → []
```

---

## filterMatches()

### Full Signature

```typescript
function filterMatches(
  requestedLocales: Array<string>,
  availableLocales: Array<string>,
  strategy: "filtering" | "matching" | "lookup"
): Array<string>
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `requestedLocales` | `Array<string>` | Yes | User's preferred locales |
| `availableLocales` | `Array<string>` | Yes | Application's available locales |
| `strategy` | `"filtering" \| "matching" \| "lookup"` | Yes | Algorithm to use |

### Return Value

`Array<string>` — Matched locale identifiers. Does NOT handle `defaultLocale` logic — that is done by `negotiateLanguages()`.

### Usage Notes

This is the low-level matching engine exported for advanced use cases. In most applications, ALWAYS use `negotiateLanguages()` instead, which wraps `filterMatches()` and adds `defaultLocale` handling.

### Strategy Behaviors

**Filtering:**
- Iterates all requested locales
- For each requested locale, finds ALL available locales that match
- Includes subtag matches (e.g., requested `"de-DE"` matches available `"de"`)
- Results are ordered: exact matches first, then subtag matches

**Matching:**
- Iterates all requested locales
- For each requested locale, finds the SINGLE best available locale
- Prefers exact matches over subtag matches
- Each available locale can only be matched once

**Lookup:**
- Iterates requested locales in order
- Returns the FIRST available locale that matches ANY requested locale
- Returns at most one result
- Uses subtag fallback: strips subtags progressively (e.g., `"de-DE"` → `"de"`)

---

## Source References

- `negotiateLanguages`: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/negotiate_languages.ts
- `acceptedLanguages`: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/accepted_languages.ts
- `filterMatches`: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/matches.ts
