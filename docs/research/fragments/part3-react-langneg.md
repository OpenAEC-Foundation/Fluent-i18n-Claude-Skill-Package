# Part 3: @fluent/react and @fluent/langneg

> Research date: 2026-03-19
> Sources: GitHub wiki, source code, package.json, GitHub issues — all fetched via WebFetch

---

## 3.1 @fluent/react Overview and Version

**Current version:** 0.15.2 (from `fluent-react/package.json` on GitHub main branch)

**Description:** "Fluent bindings for React" — provides the React integration layer for Project Fluent, exposing translations via the React Context API and component patterns.

**License:** Apache-2.0

**Entry points:**
- CommonJS: `./index.js`
- ESM: `./esm/index.js`
- TypeScript definitions: `./esm/index.d.ts`

**Dependencies:**
- `@fluent/sequence`: ^0.8.0
- `cached-iterable`: ^0.3.0

**Peer dependencies:**
- `@fluent/bundle`: >=0.16.0
- `react`: >=16.8.0 (hooks support required)

**Node engine requirement:** ^20.19 || ^22.12 || >=24

**Complete public API exports** (from `src/index.ts`):

| Export | Type | Source module |
|--------|------|---------------|
| `ReactLocalization` | Class | `./localization.js` |
| `LocalizationProvider` | Component | `./provider.js` |
| `withLocalization` | HOC | `./with_localization.js` |
| `WithLocalizationProps` | TypeScript type | `./with_localization.js` |
| `Localized` | Component | `./localized.js` |
| `LocalizedProps` | TypeScript type | `./localized.js` |
| `MarkupParser` | Type | `./markup.js` |
| `useLocalization` | Hook | `./use_localization.js` |

**Installation (complete stack):**
```bash
npm install @fluent/bundle @fluent/langneg @fluent/react
```

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-react/package.json, https://github.com/projectfluent/fluent.js/blob/main/fluent-react/src/index.ts

---

## 3.2 LocalizationProvider

The `LocalizationProvider` component uses React's Context API to distribute translations throughout the component tree. Every `<Localized>` component and every `useLocalization()` call must be a descendant of a `<LocalizationProvider>`.

### 3.2.1 Props and Configuration

The provider accepts an `l10n` prop (a `ReactLocalization` instance) in the wiki documentation pattern, but the React Tutorial shows a `bundles` prop pattern. Based on the source code and tutorial, the canonical usage is:

```tsx
import { LocalizationProvider } from "@fluent/react";

<LocalizationProvider l10n={l10n}>
  <App />
</LocalizationProvider>
```

Where `l10n` is a `ReactLocalization` instance. The provider makes this instance available to all descendant components via React Context.

**Key behavior:**
- Manages subscriptions for all nested `<Localized>` elements
- Establishes a fallback chain — if a translation is missing in one bundle, the system proceeds to the next bundle
- Context-based: eliminates manual prop-threading of translation objects

Source: https://github.com/projectfluent/fluent.js/wiki/LocalizationProvider

### 3.2.2 Bundle Generation Pattern

Bundles can be generated synchronously (array) or lazily (generator). Both approaches yield `FluentBundle` instances:

**Array-based (eager, synchronous):**
```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

function generateBundles(currentLocales: string[]): FluentBundle[] {
  return currentLocales.map((locale) => {
    const bundle = new FluentBundle(locale);
    for (const file of FILES[locale]) {
      const resource = new FluentResource(file);
      bundle.addResource(resource);
    }
    return bundle;
  });
}
```

**Generator-based (lazy):**
```typescript
function* generateBundles(currentLocales: string[]): Generator<FluentBundle> {
  for (const locale of currentLocales) {
    const bundle = new FluentBundle(locale);
    for (const file of FILES[locale]) {
      const resource = new FluentResource(file);
      bundle.addResource(resource);
    }
    yield bundle;
  }
}
```

The generator pattern is preferable for large locale sets because fallback bundles are only created when needed (lazy evaluation). The `ReactLocalization` constructor wraps the iterable in `CachedSyncIterable.from()` so it is only iterated once and then cached.

Source: https://github.com/projectfluent/fluent.js/wiki/ReactLocalization, https://github.com/projectfluent/fluent.js/blob/main/fluent-react/src/localization.ts

### 3.2.3 Async Bundle Loading

Translations are typically stored in `.ftl` files on the server and loaded asynchronously. The recommended pattern fetches all needed translations before passing them to the provider:

```typescript
async function getMessages(locale: string): Promise<string> {
  const url = `/static/locale/${locale}/content.ftl`;
  const response = await fetch(url);
  return await response.text();
}

async function createBundles(locales: string[]): Promise<FluentBundle[]> {
  return Promise.all(
    locales.map(async (locale) => {
      const messages = await getMessages(locale);
      const bundle = new FluentBundle(locale);
      bundle.addResource(new FluentResource(messages));
      return bundle;
    })
  );
}

// In your app initialization:
const bundles = await createBundles(negotiatedLocales);
const l10n = new ReactLocalization(bundles);
```

**Critical limitation:** As of v0.15.2, all translations (including fallback locales) must be fetched before rendering `<LocalizationProvider>`. There is no built-in support for async/lazy fallback locale loading during render. Future versions may add this capability.

Source: https://github.com/projectfluent/fluent.js/wiki/React-Tutorial, https://github.com/projectfluent/fluent.js/wiki/ReactLocalization

---

## 3.3 Localized Component

The `<Localized>` component is the primary declarative API for displaying translated content. It wraps a React element and replaces its text content (and optionally attributes) with translations.

### 3.3.1 Basic Usage

```tsx
import { Localized } from "@fluent/react";

function Greeting() {
  return (
    <Localized id="hello-world">
      <p>Hello, World!</p>
    </Localized>
  );
}
```

The child element (`<p>`) serves as the fallback — it is displayed if the translation for `"hello-world"` is not found in any bundle.

**Props (from `LocalizedProps` type):**

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `id` | `string` | Yes | Translation message identifier |
| `attrs` | `Record<string, boolean>` | No | Map of attribute names to translate |
| `vars` | `Record<string, FluentVariable>` | No | Variables passed to the translation |
| `elems` | `Record<string, ReactElement>` | No | React elements for DOM overlay markup |
| `children` | `ReactElement` | No | Fallback element |

**With variables:**
```tsx
<Localized id="welcome-user" vars={{ userName: currentUser.name }}>
  <p>Welcome, {currentUser.name}!</p>
</Localized>
```

Corresponding FTL:
```ftl
welcome-user = Welcome, { $userName }!
```

Source: https://github.com/projectfluent/fluent.js/wiki/Localized, https://github.com/projectfluent/fluent.js/wiki/React-Tutorial

### 3.3.2 DOM Overlays

DOM overlays enable translations to contain markup that maps to real React components. This is the `elems` prop mechanism:

```tsx
<Localized
  id="create-account"
  elems={{
    confirm: <button onClick={createAccount}></button>,
    cancel: <Link to="/"></Link>,
  }}
>
  <p>{"<confirm>Create account</confirm> or <cancel>go back</cancel>."}</p>
</Localized>
```

Corresponding FTL:
```ftl
create-account = <confirm>Create account</confirm> or <cancel>go back</cancel>.
```

**How it works internally:**
1. `@fluent/react` parses the translation using a `<template>` element, creating an inert `DocumentFragment`
2. Unknown elements (like `<confirm>`, `<cancel>`) become `HTMLUnknownElement` instances
3. The library matches these by name against the `elems` prop keys
4. Provided React elements are cloned with the translated text content as children

**Result:**
```html
<p>
  <button onClick={createAccount}>Create account</button> or
  <Link to="/">go back</Link>.
</p>
```

**Custom markup parser (for SSR / React Native):**
When `<template>` is unavailable (server-side rendering, React Native), pass a custom `parseMarkup` function to `ReactLocalization`:

```typescript
type MarkupParser = (str: string) => Array<Node>;
// Each Node must have: nodeName, textContent

const l10n = new ReactLocalization(bundles, customParseMarkup);
```

For SSR, `jsdom` or `cheerio` can be used as the parser backend.

Source: https://github.com/projectfluent/fluent.js/wiki/React-Overlays

### 3.3.3 Attribute Mapping

The `attrs` prop controls which FTL message attributes are applied to the wrapped element:

```tsx
<Localized id="my-link" attrs={{ title: true }}>
  <a href="/something" title="Link to something">
    Something
  </a>
</Localized>
```

Corresponding FTL:
```ftl
my-link = Something
    .title = Link to something
```

**Key rule:** Translated attributes are ONLY applied when explicitly declared as `true` in the `attrs` object. This is a security measure — translations cannot inject arbitrary attributes onto DOM elements.

Source: https://github.com/projectfluent/fluent.js/wiki/React-Tutorial

---

## 3.4 useLocalization Hook

The `useLocalization` hook provides imperative access to translations within functional components. It returns the current `ReactLocalization` instance from context.

```typescript
import { useLocalization } from "@fluent/react";

function AlertButton() {
  const { l10n } = useLocalization();

  const handleClick = () => {
    alert(l10n.getString("confirm-delete"));
  };

  return <button onClick={handleClick}>Delete</button>;
}
```

**Return value:** `{ l10n: ReactLocalization }`

**Primary use case:** Imperative translation formatting — `window.alert()`, `window.confirm()`, logging, validation messages, or any scenario where JSX rendering via `<Localized>` is not possible.

**`getString` method signature:**
```typescript
getString(
  id: string,
  vars?: Record<string, FluentVariable> | null,
  fallback?: string
): string
```

If the translation is not found, `getString` returns the `fallback` parameter, or the `id` itself if no fallback is provided.

Source: https://github.com/projectfluent/fluent.js/wiki/useLocalization

---

## 3.5 ReactLocalization Class

The `ReactLocalization` class is the core engine of `@fluent/react`. It stores, caches, and queries a sequence of `FluentBundle` instances.

**Constructor:**
```typescript
constructor(
  bundles: Iterable<FluentBundle>,
  parseMarkup?: MarkupParser | null,   // defaults to createParseMarkup()
  reportError?: (error: Error) => void  // defaults to console.warn
)
```

**Properties:**
| Property | Type | Description |
|----------|------|-------------|
| `bundles` | `Iterable<FluentBundle>` | Bundle sequence (wrapped in `CachedSyncIterable`) |
| `parseMarkup` | `MarkupParser \| null` | Parser for DOM overlay markup |
| `reportError` | `(error: Error) => void` | Error reporting callback |

**Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `getBundle` | `(id: string) => FluentBundle \| null` | Find first bundle containing message `id` |
| `getString` | `(id: string, vars?: Record<string, FluentVariable> \| null, fallback?: string) => string` | Format message to string; returns fallback or id if not found |
| `getElement` | `(sourceElement: ReactElement, id: string, args?: { vars?, elems?, attrs? }) => ReactElement` | Format message into a React element with overlays and attributes |
| `areBundlesEmpty` | `() => boolean` | Check if bundle iterable is exhausted/empty |

**Bundle resolution:** The class iterates through bundles in order (representing the user's locale preference chain). The first bundle that contains the requested message `id` wins. This enables graceful degradation — missing translations fall through to the next locale.

**Caching:** The constructor wraps the bundle iterable with `CachedSyncIterable.from()`, ensuring the potentially expensive iteration (especially from generators) only happens once.

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-react/src/localization.ts

---

## 3.6 Locale Switching Patterns

To switch locales at runtime, create a new `ReactLocalization` instance with re-ordered bundles and update the provider. The typical React pattern uses state:

```tsx
import React, { useState, useCallback } from "react";
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization, LocalizationProvider } from "@fluent/react";

const AVAILABLE_LOCALES = ["en-US", "fr", "de"];

function generateBundles(locales: string[]): FluentBundle[] {
  return locales.map((locale) => {
    const bundle = new FluentBundle(locale);
    bundle.addResource(new FluentResource(MESSAGES[locale]));
    return bundle;
  });
}

function App() {
  const [currentLocales, setCurrentLocales] = useState(() =>
    negotiateLanguages(navigator.languages, AVAILABLE_LOCALES, {
      defaultLocale: "en-US",
    })
  );

  const l10n = new ReactLocalization(generateBundles(currentLocales));

  const switchLocale = useCallback((locale: string) => {
    const negotiated = negotiateLanguages(
      [locale],
      AVAILABLE_LOCALES,
      { defaultLocale: "en-US" }
    );
    setCurrentLocales(negotiated);
  }, []);

  return (
    <LocalizationProvider l10n={l10n}>
      <LocaleSwitcher onSwitch={switchLocale} />
      <MainContent />
    </LocalizationProvider>
  );
}
```

When `currentLocales` state changes, React re-renders the provider with a new `ReactLocalization` instance, and all `<Localized>` components automatically re-translate.

Source: https://github.com/projectfluent/fluent.js/wiki/React-Bindings, https://github.com/projectfluent/fluent.js/wiki/React-Tutorial

---

## 3.7 @fluent/langneg Overview

**Current version:** 0.7.0

**Description:** "Language Negotiation API for Fluent"

**License:** Apache-2.0

**Dependencies:** None (zero runtime dependencies)

**Node engine requirement:** ^20.19 || ^22.12 || >=24

**Exports (from `src/index.ts`):**

| Export | Type | Description |
|--------|------|-------------|
| `negotiateLanguages` | Function | Main negotiation algorithm |
| `acceptedLanguages` | Function | Parse HTTP Accept-Language header |
| `filterMatches` | Function | Low-level matching engine |

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/package.json, https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/index.ts

### 3.7.1 negotiateLanguages()

The primary function that matches user-requested locales against application-available locales.

**Full TypeScript signature:**
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

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `requestedLocales` | `Readonly<Array<string>>` | User's preferred locales (e.g., `navigator.languages`) |
| `availableLocales` | `Readonly<Array<string>>` | Locales the app supports |
| `options.strategy` | `"filtering" \| "matching" \| "lookup"` | Negotiation algorithm (default: `"filtering"`) |
| `options.defaultLocale` | `string` | Fallback locale; appended if not already in results |

**Return value:** `Array<string>` — locale identifiers ordered by preference.

**Implementation logic:**
1. Calls `filterMatches()` with the chosen strategy
2. For `"lookup"` strategy: throws `Error` if `defaultLocale` is undefined; pushes `defaultLocale` if no matches found
3. For other strategies: appends `defaultLocale` to results if provided and not already present

**Usage example:**
```typescript
import { negotiateLanguages } from "@fluent/langneg";

const supported = negotiateLanguages(
  navigator.languages,        // e.g., ["de-DE", "fr-FR"]
  ["it", "de", "en-US", "fr-CA", "de-DE", "fr", "de-AU"],
  { defaultLocale: "en-US" }
);
```

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/negotiate_languages.ts, https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/README.md

### 3.7.2 Strategies

Given requested `["de-DE", "fr-FR"]` and available `["it", "de", "en-US", "fr-CA", "de-DE", "fr", "de-AU"]`:

**Filtering (default):** Returns ALL available locales that match ANY requested locale. Most permissive.
- Result: `["de-DE", "de", "fr", "fr-CA"]`
- Use when: you want maximum coverage with multiple fallbacks per language

**Matching:** Returns the BEST single match for EACH requested locale. One result per request.
- Result: `["de-DE", "fr"]`
- Use when: you want one best-fit per user preference

**Lookup:** Returns the SINGLE best match across ALL requested locales. Most restrictive.
- Result: `["de-DE"]`
- Use when: you need exactly one locale (e.g., selecting a date format library)
- **Requirement:** `defaultLocale` MUST be provided or the function throws an `Error`

**Likely subtags:** The module includes minimal likely-subtags data to resolve generic locales. Requesting `"en"` can match both `"en-GB"` and `"en-US"`.

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/README.md

### 3.7.3 BCP 47 Compliance

All locale identifiers in `@fluent/langneg` follow BCP 47 (IETF language tag standard):
- Language: `en`, `fr`, `de`
- Language + region: `en-US`, `fr-CA`, `de-DE`
- Language + script: `sr-Latn`, `zh-Hans`

The negotiation algorithms perform subtag matching — `"de-DE"` will match `"de"` as a fallback. Input locales are coerced to strings via `Array.from(...).map(String)`.

**acceptedLanguages() helper:**
For server-side usage, parse the HTTP `Accept-Language` header:

```typescript
import { acceptedLanguages } from "@fluent/langneg";

// Parse: "fr-CH, fr;q=0.9, en;q=0.8, de;q=0.7, *;q=0.5"
const locales = acceptedLanguages(req.headers["accept-language"]);
// Result: ["fr-CH", "fr", "en", "de", "*"] (sorted by q-value)
```

**Implementation:** Parses quality values (`q=`) from the header, sorts by descending quality while preserving order for equal weights. Throws `TypeError` if argument is not a string.

Source: https://github.com/projectfluent/fluent.js/blob/main/fluent-langneg/src/accepted_languages.ts

---

## 3.8 Complete Integration Pattern (Full Example)

A production-ready setup combining all packages:

```typescript
// l10n.ts — Localization setup module
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";

const AVAILABLE_LOCALES = ["en-US", "fr", "it"];
const DEFAULT_LOCALE = "en-US";

async function fetchMessages(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/messages.ftl`);
  return response.text();
}

function createBundle(locale: string, messages: string): FluentBundle {
  const bundle = new FluentBundle(locale);
  const resource = new FluentResource(messages);
  const errors = bundle.addResource(resource);
  if (errors.length) {
    // Log but do not throw — partial translations are acceptable
    errors.forEach((e) => console.warn(`FTL parse error in ${locale}:`, e));
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
      const messages = await fetchMessages(locale);
      return createBundle(locale, messages);
    })
  );

  return new ReactLocalization(bundles);
}
```

```tsx
// App.tsx — Root component
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
    return <div>Loading...</div>;
  }

  return (
    <LocalizationProvider l10n={l10n}>
      <Localized id="app-title">
        <h1>My Application</h1>
      </Localized>
    </LocalizationProvider>
  );
}
```

```ftl
# locales/en-US/messages.ftl
app-title = My Application
welcome-user = Welcome, { $userName }!
confirm-delete = Are you sure you want to delete { $itemName }?
create-account = <confirm>Create account</confirm> or <cancel>go back</cancel>.

# locales/fr/messages.ftl
app-title = Mon Application
welcome-user = Bienvenue, { $userName } !
confirm-delete = Voulez-vous vraiment supprimer { $itemName } ?
create-account = <confirm>Créer un compte</confirm> ou <cancel>revenir</cancel>.
```

Source: Composite example based on patterns from https://github.com/projectfluent/fluent.js/wiki/React-Tutorial, https://github.com/projectfluent/fluent.js/wiki/ReactLocalization

---

## 3.9 SSR Considerations

Server-side rendering with `@fluent/react` has known challenges:

1. **No `<template>` element on server:** The DOM overlay system relies on `<template>` to parse HTML in translations. On the server (Node.js), this element does not exist. You MUST provide a custom `parseMarkup` function using `jsdom` or `cheerio`:

```typescript
import { JSDOM } from "jsdom";

function parseMarkup(str: string): Node[] {
  const dom = new JSDOM(`<body>${str}</body>`);
  return Array.from(dom.window.document.body.childNodes);
}

const l10n = new ReactLocalization(bundles, parseMarkup);
```

2. **Locale detection on server:** Use `acceptedLanguages()` to parse the HTTP `Accept-Language` header instead of `navigator.languages`:

```typescript
import { acceptedLanguages, negotiateLanguages } from "@fluent/langneg";

const requested = acceptedLanguages(req.headers["accept-language"] || "");
const supported = negotiateLanguages(requested, AVAILABLE_LOCALES, {
  defaultLocale: "en-US",
});
```

3. **No async fallback:** All bundles must be ready before rendering. This means server-side code must pre-load all translations synchronously or await them before calling `renderToString()`.

4. **Open issue #646:** There is an open issue requesting documented SSR examples for Next.js and other React-based frameworks, indicating this is an area where the community needs more guidance.

Source: https://github.com/projectfluent/fluent.js/wiki/React-Overlays, https://github.com/projectfluent/fluent.js/issues (issue #646)

---

## 3.10 Common Issues and Anti-Patterns

Based on GitHub issues analysis (https://github.com/projectfluent/fluent.js/issues?q=is%3Aissue+react):

### Open Issues (React-specific)

| # | Issue | Impact |
|---|-------|--------|
| #646 | No SSR documentation for Next.js | Developers struggle with server-side setup |
| #598 | `withLocalization` does not support ref forwarding | Cannot use refs on wrapped components |
| #571 | Insufficient `<Localized>` documentation | Developers confused about prop behavior |
| #550 | Cannot use `elems` without `children` | Limits flexibility of overlay API |
| #538 | React Overlays do not work as documented | Documentation-implementation mismatch |
| #537 | Tutorial has compile-time errors | Onboarding friction |
| #522 | No RTL/LTR direction support | Missing bidirectional text handling |
| #500 | No `setL10n` in `useLocalization` | Cannot trigger re-localization imperatively |
| #498 | Unclear behavior with text node children | API ambiguity |

### Known Anti-Patterns

1. **Post-translation string manipulation:** NEVER concatenate or manipulate translated strings. Treat translation output as opaque. Let FTL handle all string composition through placeables and selectors.

2. **Changing IDs without semantic change:** Translation IDs are contracts with translators. ALWAYS change the ID when the meaning of the message changes — this signals translators that re-translation is needed.

3. **Missing `attrs` declaration:** Translated attributes are ONLY applied when explicitly whitelisted in `attrs={{ attributeName: true }}`. Forgetting this causes attributes to silently not translate.

4. **Creating ReactLocalization in render:** Avoid creating a new `ReactLocalization` instance on every render. Use `useState` or `useMemo` to prevent unnecessary re-initialization and re-rendering of all localized components.

5. **Not handling `addResource` errors:** `bundle.addResource()` returns an array of errors for malformed FTL. Ignoring these leads to silent translation failures.

6. **Using `withLocalization` instead of `useLocalization`:** The HOC does not support ref forwarding (issue #598). Prefer the `useLocalization` hook in functional components.

### Resolved Issues Worth Noting

| # | Issue | Resolution |
|---|-------|------------|
| #546 | React Native compatibility | Works but needs custom `parseMarkup` (no DOM) |
| #519 | No error reporting for missing IDs | Added `reportError` callback to `ReactLocalization` constructor |
| #473 | Rules of Hooks violations | Fixed in library code |
| #476 | Loading `.ftl` files in Create React App | Requires custom webpack config or fetch-based loading |

Source: https://github.com/projectfluent/fluent.js/issues?q=is%3Aissue+react

---

## 3.11 File Organization Best Practices

### Recommended directory structure

```
src/
├── l10n/
│   ├── index.ts              # initLocalization(), locale switching
│   └── bundles.ts            # Bundle generation logic
├── components/
│   └── LocalizedApp.tsx      # LocalizationProvider wrapper
public/
└── locales/
    ├── en-US/
    │   ├── main.ftl          # Core UI messages
    │   ├── errors.ftl        # Error messages
    │   └── settings.ftl      # Settings page messages
    ├── fr/
    │   ├── main.ftl
    │   ├── errors.ftl
    │   └── settings.ftl
    └── de/
        ├── main.ftl
        ├── errors.ftl
        └── settings.ftl
```

### Key principles

1. **One FTL file per feature/page per locale** — keeps translations manageable and enables code-splitting
2. **Flat structure for small apps** — `locales/en-US.ftl`, `locales/fr.ftl` is acceptable for fewer than 50 messages
3. **Multiple resources per bundle** — a single `FluentBundle` can hold multiple `FluentResource` instances via repeated `addResource()` calls
4. **Semantic message IDs** — use `component-element-action` pattern: `login-button-submit`, `header-nav-home`
5. **Co-locate fallback strings** — the English text inside `<Localized>` children serves as both fallback and source for translators grepping the codebase

Source: https://github.com/projectfluent/fluent.js/wiki/React-Tutorial

---

## Summary of Key Facts for Skill Writing

| Fact | Value |
|------|-------|
| @fluent/react version | 0.15.2 |
| @fluent/langneg version | 0.7.0 |
| React peer dep | >=16.8.0 |
| @fluent/bundle peer dep | >=0.16.0 |
| License | Apache-2.0 |
| Primary components | `LocalizationProvider`, `Localized` |
| Primary hook | `useLocalization` → `{ l10n }` |
| Primary class | `ReactLocalization(bundles, parseMarkup?, reportError?)` |
| Negotiation strategies | filtering (default), matching, lookup |
| SSR | Requires custom `parseMarkup`, `acceptedLanguages()` |
| Known gaps | No async fallback loading, no RTL support, no ref forwarding in HOC |
