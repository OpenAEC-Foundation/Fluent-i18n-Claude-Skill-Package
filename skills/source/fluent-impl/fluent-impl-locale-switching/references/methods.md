# Methods Reference — fluent-impl-locale-switching

## negotiateLanguages()

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

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `requestedLocales` | `Readonly<Array<string>>` | Yes | User's preferred locales (e.g., `navigator.languages` or `[userChoice]`) |
| `availableLocales` | `Readonly<Array<string>>` | Yes | Locales the application supports |
| `options.strategy` | `"filtering" \| "matching" \| "lookup"` | No | Negotiation algorithm. Default: `"filtering"` |
| `options.defaultLocale` | `string` | No | Appended to results if not already present. Required for `"lookup"` strategy |

**Return value:** `Array<string>` — locale identifiers ordered by preference.

**Behavior by strategy:**

| Strategy | Input requested | Input available | Output |
|----------|----------------|-----------------|--------|
| `"filtering"` | `["de-DE", "fr"]` | `["en-US", "de", "de-DE", "fr-CA", "fr"]` | `["de-DE", "de", "fr", "fr-CA", "en-US"]` |
| `"matching"` | `["de-DE", "fr"]` | `["en-US", "de", "de-DE", "fr-CA", "fr"]` | `["de-DE", "fr", "en-US"]` |
| `"lookup"` | `["de-DE", "fr"]` | `["en-US", "de", "de-DE", "fr-CA", "fr"]` | `["de-DE"]` |

**ALWAYS** provide `defaultLocale` when using `"lookup"` strategy — the function throws `Error` if it is missing.

---

## acceptedLanguages()

```typescript
import { acceptedLanguages } from "@fluent/langneg";

function acceptedLanguages(header: string): Array<string>
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `header` | `string` | Yes | Raw HTTP `Accept-Language` header value |

**Return value:** `Array<string>` — locale identifiers sorted by quality value (descending).

**Example:**
```typescript
const header = "fr-CH, fr;q=0.9, en;q=0.8, de;q=0.7, *;q=0.5";
const locales = acceptedLanguages(header);
// Result: ["fr-CH", "fr", "en", "de", "*"]
```

**Throws:** `TypeError` if `header` is not a string.

**ALWAYS** guard against undefined headers on the server:
```typescript
const requested = acceptedLanguages(req.headers["accept-language"] || "");
```

---

## ReactLocalization

```typescript
import { ReactLocalization } from "@fluent/react";

class ReactLocalization {
  constructor(
    bundles: Iterable<FluentBundle>,
    parseMarkup?: MarkupParser | null,
    reportError?: (error: Error) => void
  );

  bundles: Iterable<FluentBundle>;
  getString(id: string, vars?: Record<string, FluentVariable> | null, fallback?: string): string;
  getBundle(id: string): FluentBundle | null;
  getElement(sourceElement: ReactElement, id: string, args?: { vars?, elems?, attrs? }): ReactElement;
  areBundlesEmpty(): boolean;
}
```

### Locale Switching Integration

To switch locales, create a NEW `ReactLocalization` instance with new bundles. The `LocalizationProvider` detects the new instance and triggers re-rendering of all `<Localized>` components.

```typescript
// On locale change:
const newBundles = generateBundles(negotiatedLocales);
const newL10n = new ReactLocalization(newBundles);
// Pass newL10n to LocalizationProvider via state update
```

**NEVER** mutate the `bundles` property directly — ALWAYS create a new `ReactLocalization` instance.

---

## React State Management for Locale Switching

### useState Pattern

```typescript
const [currentLocales, setCurrentLocales] = useState<string[]>(() =>
  detectInitialLocales()
);
```

**ALWAYS** use lazy initialization (function form) for `useState` when the initial value requires computation (negotiation, localStorage reads).

### useMemo for ReactLocalization

```typescript
const l10n = React.useMemo(
  () => new ReactLocalization(generateBundles(currentLocales)),
  [currentLocales]
);
```

**ALWAYS** memoize `ReactLocalization` creation with `useMemo` keyed on the `currentLocales` array. This prevents recreating the instance on every render while still updating when locales change.

### useCallback for Switch Handler

```typescript
const switchLocale = useCallback((locale: string) => {
  saveLocalePreference(locale);
  const negotiated = negotiateLocales([locale]);
  setCurrentLocales(negotiated);
}, []);
```

**ALWAYS** wrap the switch handler in `useCallback` to maintain stable identity for child components that accept it as a prop.

---

## localStorage API for Preference Persistence

```typescript
// Save preference
localStorage.setItem("fluent-locale-preference", locale);

// Read preference
const saved = localStorage.getItem("fluent-locale-preference");

// Clear preference (revert to browser default)
localStorage.removeItem("fluent-locale-preference");
```

**ALWAYS** wrap localStorage access in try/catch — it throws in private browsing mode on some browsers and is unavailable in SSR environments.

---

## Cookie-Based Persistence (Server-Side)

```typescript
// Client: set cookie on locale switch
document.cookie = `locale-preference=${locale}; path=/; max-age=31536000; SameSite=Lax`;

// Server (Express): read cookie
const locale = req.cookies?.["locale-preference"];
```

**ALWAYS** set `SameSite=Lax` and `path=/` on locale cookies. **NEVER** set `HttpOnly` on the locale cookie — client-side JavaScript must be able to read it for hydration.
