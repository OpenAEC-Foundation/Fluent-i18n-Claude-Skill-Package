# anti-patterns.md -- Scaffolding Mistakes to Avoid

## AP-1: Installing Only @fluent/bundle

**Wrong:**
```bash
npm install @fluent/bundle
```

**Why it fails:** Without `@fluent/langneg`, there is no way to match user locale preferences to available translations. Without `@fluent/react`, there is no way to distribute translations through the React component tree.

**Correct:**
```bash
# React projects: ALWAYS install all three
npm install @fluent/bundle @fluent/react @fluent/langneg

# Vanilla JS: at minimum two
npm install @fluent/bundle @fluent/langneg
```

---

## AP-2: Bundling FTL Strings Into JavaScript

**Wrong:**
```typescript
// Importing FTL as a string constant in JS
const messages = `
app-title = My Application
welcome-user = Welcome, { $userName }!
`;
```

**Why it fails:** This defeats locale-based code splitting. ALL locale strings are loaded regardless of which locale the user needs. The app bundle grows linearly with each new locale.

**Correct:** ALWAYS serve FTL files statically and load them via `fetch()`:
```typescript
const response = await fetch(`/locales/${locale}/main.ftl`);
const messages = await response.text();
```

---

## AP-3: Missing AVAILABLE_LOCALES Update When Adding Locales

**Wrong:**
```
public/locales/es/main.ftl  ← file exists
```
```typescript
// But AVAILABLE_LOCALES was never updated
export const AVAILABLE_LOCALES = ["en-US", "fr", "de"];
// "es" is never matched by negotiateLanguages
```

**Why it fails:** `negotiateLanguages` only considers locales listed in its `availableLocales` parameter. The FTL files exist but are never loaded.

**Correct:** ALWAYS update `AVAILABLE_LOCALES` when adding a new locale directory:
```typescript
export const AVAILABLE_LOCALES = ["en-US", "fr", "de", "es"];
```

---

## AP-4: Creating ReactLocalization Inside Render

**Wrong:**
```tsx
function App() {
  // NEW instance on EVERY render — causes full re-render cascade
  const l10n = new ReactLocalization(generateBundles(locales));

  return (
    <LocalizationProvider l10n={l10n}>
      <Content />
    </LocalizationProvider>
  );
}
```

**Why it fails:** Every render creates a new `ReactLocalization` instance. React detects a new context value and re-renders ALL `<Localized>` components and `useLocalization` consumers, even when translations have not changed.

**Correct:** Use `useState` to hold the instance; only update it when the locale actually changes:
```tsx
function App() {
  const [l10n, setL10n] = useState<ReactLocalization | null>(null);

  useEffect(() => {
    initLocalization().then(setL10n);
  }, []);

  if (!l10n) return <div>Loading...</div>;

  return (
    <LocalizationProvider l10n={l10n}>
      <Content />
    </LocalizationProvider>
  );
}
```

---

## AP-5: Hardcoding navigator.languages Without Negotiation

**Wrong:**
```typescript
const locale = navigator.languages[0]; // "de-AT"
const response = await fetch(`/locales/${locale}/main.ftl`); // 404!
```

**Why it fails:** The user's browser may report `de-AT` but the app only has `de`. Without `negotiateLanguages`, there is no fallback matching. The fetch fails with a 404.

**Correct:** ALWAYS pass browser locales through `negotiateLanguages`:
```typescript
const negotiated = negotiateLanguages(
  navigator.languages,      // ["de-AT", "en"]
  ["en-US", "de", "fr"],   // what the app supports
  { defaultLocale: "en-US" }
);
// Result: ["de", "en-US"] — de-AT matched to de, en matched to en-US
```

---

## AP-6: Ignoring addResource Errors

**Wrong:**
```typescript
const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(messages));
// No error check — malformed FTL silently produces missing translations
```

**Why it fails:** `addResource()` returns an array of parse errors. If the FTL has syntax issues, some messages are silently dropped. The app shows fallback text or message IDs with no indication of why.

**Correct:**
```typescript
const errors = bundle.addResource(new FluentResource(messages));
if (errors.length) {
  errors.forEach((e) => console.warn(`FTL parse error:`, e));
}
```

---

## AP-7: Missing Default Locale in negotiateLanguages

**Wrong:**
```typescript
const negotiated = negotiateLanguages(
  navigator.languages,
  AVAILABLE_LOCALES
  // No defaultLocale!
);
// If no matches found, negotiated is []
```

**Why it fails:** Without `defaultLocale`, `negotiateLanguages` may return an empty array when the user's browser locales do not match any available locale. This causes zero bundles to load, and the app shows only fallback text.

**Correct:** ALWAYS provide `defaultLocale`:
```typescript
const negotiated = negotiateLanguages(
  navigator.languages,
  AVAILABLE_LOCALES,
  { defaultLocale: "en-US" }
);
// ALWAYS returns at least ["en-US"]
```

---

## AP-8: Inconsistent FTL Files Across Locales

**Wrong:**
```
public/locales/en-US/main.ftl      ← 15 messages
public/locales/en-US/errors.ftl    ← 8 messages
public/locales/fr/main.ftl         ← 12 messages (3 missing!)
                                    ← errors.ftl missing entirely!
```

**Why it fails:** When the French bundle cannot find a message, it falls back to the next bundle in the chain. If ALL bundles are missing the message (or the fallback chain is empty), the user sees the raw message ID. Inconsistent file sets also cause `fetch()` 404 errors.

**Correct:** ALWAYS ensure every locale directory has the SAME set of FTL files with the SAME set of message IDs. Use the en-US files as the source of truth and verify completeness before deployment.

---

## AP-9: Using localStorage Without Try-Catch

**Wrong:**
```typescript
function getSavedLocale(): string | null {
  return localStorage.getItem("preferred-locale");
}
```

**Why it fails:** `localStorage` throws in private browsing mode on some browsers, in SSR environments, and when storage quota is exceeded. An unhandled exception crashes the localization initialization.

**Correct:**
```typescript
function getSavedLocale(): string | null {
  try {
    return localStorage.getItem("preferred-locale");
  } catch {
    return null;
  }
}
```

---

## AP-10: Forgetting the Loading State

**Wrong:**
```tsx
function App() {
  const [l10n, setL10n] = useState<ReactLocalization | null>(null);
  useEffect(() => { initLocalization().then(setL10n); }, []);

  // No loading check — crashes when l10n is null
  return (
    <LocalizationProvider l10n={l10n!}>
      <Content />
    </LocalizationProvider>
  );
}
```

**Why it fails:** `initLocalization` is async. On the first render, `l10n` is `null`. Passing `null!` to `LocalizationProvider` causes a runtime error because `ReactLocalization` methods are called on null.

**Correct:** ALWAYS render a loading state while translations are being fetched:
```tsx
if (!l10n) {
  return <div>Loading...</div>;
}

return (
  <LocalizationProvider l10n={l10n}>
    <Content />
  </LocalizationProvider>
);
```

---

## AP-11: Mutating ReactLocalization Instead of Replacing

**Wrong:**
```typescript
// Attempting to "update" an existing ReactLocalization
l10n.bundles = newBundles; // This does NOT trigger React re-renders
```

**Why it fails:** `ReactLocalization` does not have reactivity. Mutating its properties does not notify `LocalizationProvider` of changes. All `<Localized>` components continue showing the old translations.

**Correct:** ALWAYS create a NEW `ReactLocalization` instance and update React state:
```typescript
const newL10n = new ReactLocalization(newBundles);
setL10n(newL10n); // Triggers re-render of entire localized tree
```

---

## AP-12: Using @fluent/syntax for Runtime Formatting

**Wrong:**
```typescript
import { FluentParser } from "@fluent/syntax";
// Importing a 30KB+ tooling library into the app bundle
```

**Why it fails:** `@fluent/syntax` is a full AST parser/serializer intended for tooling (linting, validation, programmatic FTL generation). It adds significant bundle weight. `@fluent/bundle` already includes its own optimized runtime parser.

**Correct:** Install `@fluent/syntax` as a devDependency ONLY. Use it in CI scripts and build tools, NEVER in application runtime code:
```json
{
  "devDependencies": {
    "@fluent/syntax": "^0.19.0"
  }
}
```

---

## AP-13: Generating Locale Directories That Do Not Match BCP 47

**Wrong:**
```
public/locales/english/main.ftl
public/locales/french/main.ftl
```

**Why it fails:** `negotiateLanguages` uses BCP 47 identifiers (`en-US`, `fr`, `de-DE`). Directory names like `english` or `french` do not match. The fetch URL becomes `/locales/en-US/main.ftl` but the file is at `/locales/english/main.ftl`.

**Correct:** ALWAYS use BCP 47 identifiers as directory names:
```
public/locales/en-US/main.ftl
public/locales/fr/main.ftl
public/locales/de/main.ftl
```

---

## AP-14: No Error Handling for FTL Fetch Failures

**Wrong:**
```typescript
async function fetchMessages(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/main.ftl`);
  return response.text(); // No status check — 404 returns HTML error page as "FTL"
}
```

**Why it fails:** If the FTL file does not exist, `fetch()` returns a 404 response. Calling `.text()` on a 404 returns the HTML error page. Passing that HTML to `FluentResource` produces zero valid messages and fills the error array.

**Correct:**
```typescript
async function fetchMessages(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/main.ftl`);
  if (!response.ok) {
    throw new Error(`Failed to fetch FTL for ${locale}: ${response.status}`);
  }
  return response.text();
}
```
