# Anti-Patterns Reference — fluent-impl-locale-switching

## AP-1: Recreating ReactLocalization on Every Render

### Problem

Creating a new `ReactLocalization` instance inside the render function without memoization causes every `<Localized>` component in the tree to re-render on every parent render, even when the locale has not changed.

### Wrong

```tsx
function App() {
  const [currentLocales, setCurrentLocales] = useState(["en-US"]);

  // BAD: new instance created on EVERY render
  const l10n = new ReactLocalization(generateBundles(currentLocales));

  return (
    <LocalizationProvider l10n={l10n}>
      <MainContent />
    </LocalizationProvider>
  );
}
```

### Correct

```tsx
function App() {
  const [currentLocales, setCurrentLocales] = useState(["en-US"]);

  // GOOD: only recreated when currentLocales changes
  const l10n = React.useMemo(
    () => new ReactLocalization(generateBundles(currentLocales)),
    [currentLocales]
  );

  return (
    <LocalizationProvider l10n={l10n}>
      <MainContent />
    </LocalizationProvider>
  );
}
```

### Why It Matters

`LocalizationProvider` uses React Context. When the `l10n` prop reference changes, React propagates the change to ALL consumers. Without memoization, this happens on every render — not just when the locale actually changes. In large applications this causes severe performance degradation.

---

## AP-2: Hardcoding Locale Lists in Multiple Places

### Problem

Scattering locale identifiers across different files makes it impossible to add or remove a supported locale without hunting through the entire codebase.

### Wrong

```typescript
// file1.ts
const locales = negotiateLanguages(requested, ["en-US", "fr", "de"], ...);

// file2.ts — different file, duplicated list
const LOCALES = ["en-US", "fr", "de"];

// file3.ts — someone forgot to add "de" here
const supportedLocales = ["en-US", "fr"];
```

### Correct

```typescript
// src/l10n/constants.ts — single source of truth
export const AVAILABLE_LOCALES = ["en-US", "fr", "de"] as const;
export const DEFAULT_LOCALE = "en-US";

// file1.ts
import { AVAILABLE_LOCALES, DEFAULT_LOCALE } from "./l10n/constants";
const locales = negotiateLanguages(requested, [...AVAILABLE_LOCALES], {
  defaultLocale: DEFAULT_LOCALE,
});
```

### Rule

**ALWAYS** define `AVAILABLE_LOCALES` and `DEFAULT_LOCALE` in exactly one module. Import everywhere else.

---

## AP-3: Skipping negotiateLanguages

### Problem

Passing raw user input directly to bundle generation without negotiation can produce invalid locale lists, empty fallback chains, or missing the default locale entirely.

### Wrong

```typescript
// BAD: raw user selection with no negotiation
const switchLocale = (locale: string) => {
  setCurrentLocales([locale]); // No fallback chain, no validation
};
```

### Correct

```typescript
// GOOD: negotiation produces valid fallback chain
const switchLocale = (locale: string) => {
  const negotiated = negotiateLanguages(
    [locale],
    [...AVAILABLE_LOCALES],
    { defaultLocale: DEFAULT_LOCALE }
  );
  setCurrentLocales(negotiated);
};
```

### Why It Matters

Without negotiation:
- If the user selects `"de-AT"` and you only have `"de"`, the locale list is `["de-AT"]` — no bundles match
- The default locale is never appended, so if the primary locale has missing translations, there is no fallback
- No subtag matching occurs, so `"de-AT"` does not fall back to `"de"`

---

## AP-4: Not Persisting User Preference

### Problem

If the user explicitly switches locale but the choice is not saved, every page reload or new tab reverts to the browser default. This frustrates users whose browser language differs from their preferred locale.

### Wrong

```typescript
// BAD: locale choice lost on page reload
const switchLocale = (locale: string) => {
  setCurrentLocales(negotiateLocales([locale]));
  // Nothing saved — choice is gone after reload
};
```

### Correct

```typescript
// GOOD: persist choice to localStorage
const switchLocale = (locale: string) => {
  saveLocalePreference(locale); // Write to localStorage
  setCurrentLocales(negotiateLocales([locale]));
};

// On initialization, check for saved preference
const initialLocales = (() => {
  const saved = getSavedLocale();
  const requested = saved ? [saved] : navigator.languages;
  return negotiateLocales(requested);
})();
```

### Rule

**ALWAYS** persist explicit user locale choices. Use `localStorage` for client-only apps, cookies for apps with SSR.

---

## AP-5: Using navigator.languages on the Server

### Problem

`navigator` does not exist in Node.js. Accessing `navigator.languages` during SSR throws a `ReferenceError` and crashes the server.

### Wrong

```typescript
// BAD: crashes on server
const requested = navigator.languages; // ReferenceError in Node.js
```

### Correct

```typescript
// GOOD: use acceptedLanguages() on server
import { acceptedLanguages } from "@fluent/langneg";

const requested = acceptedLanguages(req.headers["accept-language"] || "");
```

### Rule

**NEVER** reference `navigator` in server-side code. **ALWAYS** use `acceptedLanguages()` from `@fluent/langneg` to parse the HTTP `Accept-Language` header instead.

---

## AP-6: Hydration Mismatch Between Server and Client

### Problem

If the server detects locale from the `Accept-Language` header and the client immediately switches to a different saved localStorage preference, React hydration produces a mismatch warning and flickers.

### Wrong

```tsx
// Server renders with Accept-Language → "fr"
// Client immediately reads localStorage → "de"
// Result: hydration mismatch, visible flicker
function App() {
  const [locales] = useState(detectInitialLocales()); // reads localStorage
  // ...
}
```

### Correct

```tsx
// Client uses server-detected locale for initial hydration
function App() {
  const serverLocales: string[] = (window as any).__INITIAL_LOCALES__;
  const [locales, setLocales] = useState(serverLocales);

  // After hydration, apply user preference if different
  useEffect(() => {
    const saved = getSavedLocale();
    if (saved && saved !== locales[0]) {
      setLocales(negotiateLocales([saved]));
    }
  }, []);
}
```

### Rule

**ALWAYS** use server-detected locales for initial hydration. Apply the user's localStorage preference in a `useEffect` after hydration completes.

---

## AP-7: Not Handling Async Bundle Loading Races

### Problem

If the user switches locales rapidly, multiple async bundle fetches race. The last fetch to resolve may not correspond to the latest locale selection, causing the UI to display the wrong language.

### Wrong

```typescript
// BAD: no race condition handling
useEffect(() => {
  loadBundles(currentLocales).then((bundles) => {
    setL10n(new ReactLocalization(bundles)); // May apply stale bundles
  });
}, [currentLocales]);
```

### Correct

```typescript
// GOOD: cleanup cancels stale updates
useEffect(() => {
  let cancelled = false;

  loadBundles(currentLocales).then((bundles) => {
    if (!cancelled) {
      setL10n(new ReactLocalization(bundles));
    }
  });

  return () => {
    cancelled = true;
  };
}, [currentLocales]);
```

### Rule

**ALWAYS** use a `cancelled` flag in the effect cleanup to discard stale async bundle loads. This prevents the UI from flickering between locales during rapid switching.

---

## AP-8: Omitting defaultLocale from negotiateLanguages

### Problem

Without `defaultLocale`, `negotiateLanguages` may return an empty array if none of the requested locales match any available locale. This results in zero bundles and a completely untranslated UI.

### Wrong

```typescript
// BAD: no defaultLocale — may return []
const locales = negotiateLanguages(navigator.languages, AVAILABLE_LOCALES);
// If navigator.languages is ["ja"] and AVAILABLE_LOCALES has no Japanese: locales = []
```

### Correct

```typescript
// GOOD: defaultLocale guarantees at least one result
const locales = negotiateLanguages(navigator.languages, AVAILABLE_LOCALES, {
  defaultLocale: "en-US",
});
// Even if no Japanese match: locales = ["en-US"]
```

### Rule

**ALWAYS** provide `defaultLocale` to `negotiateLanguages`. An empty locale list means zero bundles and zero translations.
