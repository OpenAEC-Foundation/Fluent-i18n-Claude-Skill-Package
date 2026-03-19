# fluent-impl-react: Examples

## Provider Setup

### Minimal Setup

```ftl
# locales/en-US/messages.ftl
app-title = My Application
welcome-user = Welcome, { $userName }!
```

```tsx
import React from "react";
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { ReactLocalization, LocalizationProvider, Localized } from "@fluent/react";

const ftl = `
app-title = My Application
welcome-user = Welcome, { $userName }!
`;

const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(ftl));

const l10n = new ReactLocalization([bundle]);

function App() {
  return (
    <LocalizationProvider l10n={l10n}>
      <Localized id="app-title">
        <h1>My Application</h1>
      </Localized>
    </LocalizationProvider>
  );
}
```

### Async Loading with Multiple Locales

```ftl
# locales/en-US/messages.ftl
app-title = My Application

# locales/fr/messages.ftl
app-title = Mon Application
```

```tsx
import React, { useState, useEffect } from "react";
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { ReactLocalization, LocalizationProvider } from "@fluent/react";

async function fetchMessages(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/messages.ftl`);
  return response.text();
}

async function createLocalization(locales: string[]): Promise<ReactLocalization> {
  const bundles = await Promise.all(
    locales.map(async (locale) => {
      const messages = await fetchMessages(locale);
      const bundle = new FluentBundle(locale);
      const errors = bundle.addResource(new FluentResource(messages));
      if (errors.length) {
        errors.forEach((e) => console.warn(`FTL error in ${locale}:`, e));
      }
      return bundle;
    })
  );
  return new ReactLocalization(bundles);
}

function App() {
  const [l10n, setL10n] = useState<ReactLocalization | null>(null);

  useEffect(() => {
    createLocalization(["en-US", "fr"]).then(setL10n);
  }, []);

  if (!l10n) {
    return <div>Loading translations...</div>;
  }

  return (
    <LocalizationProvider l10n={l10n}>
      <MainContent />
    </LocalizationProvider>
  );
}
```

---

## Localized Component: Basic

```ftl
hello-world = Hello, World!
```

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

The child `<p>` is the fallback -- displayed if `hello-world` is not found in any bundle.

---

## Localized Component: Variables

```ftl
welcome-user = Welcome, { $userName }!
items-count = { $count } items in your cart.
```

```tsx
<Localized id="welcome-user" vars={{ userName: "Alice" }}>
  <p>Welcome, Alice!</p>
</Localized>

<Localized id="items-count" vars={{ count: 5 }}>
  <p>5 items in your cart.</p>
</Localized>
```

Variables in FTL use `$varName` syntax. The `vars` prop keys MUST match the FTL variable names (without the `$` prefix).

---

## Localized Component: DOM Overlays (elems)

```ftl
privacy-notice = Read our <privacy-link>privacy policy</privacy-link> for details.
create-account = <confirm>Create account</confirm> or <cancel>go back</cancel>.
```

```tsx
<Localized
  id="privacy-notice"
  elems={{
    "privacy-link": <a href="/privacy"></a>,
  }}
>
  <p>{"Read our <privacy-link>privacy policy</privacy-link> for details."}</p>
</Localized>

<Localized
  id="create-account"
  elems={{
    confirm: <button onClick={handleCreate}></button>,
    cancel: <Link to="/"></Link>,
  }}
>
  <p>{"<confirm>Create account</confirm> or <cancel>go back</cancel>."}</p>
</Localized>
```

**Result HTML:**
```html
<p>Read our <a href="/privacy">privacy policy</a> for details.</p>
<p><button>Create account</button> or <a href="/">go back</a>.</p>
```

The `elems` prop keys MUST match the custom element names used in the FTL message. The React elements provided are cloned with the translated text as children.

---

## Localized Component: Attribute Mapping (attrs)

```ftl
submit-button = Submit
    .title = Click to submit the form
    .aria-label = Submit form

search-input = Search
    .placeholder = Type to search...
```

```tsx
<Localized id="submit-button" attrs={{ title: true, "aria-label": true }}>
  <button title="Click to submit the form" aria-label="Submit form">
    Submit
  </button>
</Localized>

<Localized id="search-input" attrs={{ placeholder: true }}>
  <input type="text" placeholder="Type to search..." />
</Localized>
```

ALWAYS explicitly list every attribute you want translated. Attributes NOT listed in `attrs` are silently ignored, even if they exist in the FTL message.

---

## useLocalization: Imperative Formatting

```ftl
confirm-delete = Are you sure you want to delete "{ $itemName }"?
document-title = { $pageName } — My App
validation-required = This field is required.
```

```tsx
import { useLocalization } from "@fluent/react";

function DeleteButton({ itemName }: { itemName: string }) {
  const { l10n } = useLocalization();

  const handleClick = () => {
    const message = l10n.getString("confirm-delete", { itemName });
    if (window.confirm(message)) {
      deleteItem(itemName);
    }
  };

  return <button onClick={handleClick}>Delete</button>;
}

function PageTitle({ pageName }: { pageName: string }) {
  const { l10n } = useLocalization();

  useEffect(() => {
    document.title = l10n.getString("document-title", { pageName });
  }, [l10n, pageName]);

  return null;
}

function FormField() {
  const { l10n } = useLocalization();
  const [error, setError] = useState<string | null>(null);

  const validate = (value: string) => {
    if (!value) {
      setError(l10n.getString("validation-required"));
    }
  };

  return (
    <div>
      <input onChange={(e) => validate(e.target.value)} />
      {error && <span className="error">{error}</span>}
    </div>
  );
}
```

---

## Bundle Generation Patterns

### Array-Based (Eager)

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { ReactLocalization } from "@fluent/react";

const MESSAGES: Record<string, string> = {
  "en-US": `hello = Hello!`,
  "fr": `hello = Bonjour !`,
  "de": `hello = Hallo!`,
};

function generateBundles(locales: string[]): FluentBundle[] {
  return locales.map((locale) => {
    const bundle = new FluentBundle(locale);
    bundle.addResource(new FluentResource(MESSAGES[locale]));
    return bundle;
  });
}

const l10n = new ReactLocalization(generateBundles(["en-US", "fr", "de"]));
```

### Generator-Based (Lazy)

```typescript
function* generateBundles(locales: string[]): Generator<FluentBundle> {
  for (const locale of locales) {
    const bundle = new FluentBundle(locale);
    bundle.addResource(new FluentResource(MESSAGES[locale]));
    yield bundle;
  }
}

// Fallback bundles are only created when needed
const l10n = new ReactLocalization(generateBundles(["en-US", "fr", "de"]));
```

The generator pattern is preferable for 4+ locales. `ReactLocalization` wraps the iterable in `CachedSyncIterable.from()` so it is only iterated once.

---

## Locale Switching at Runtime

```ftl
# locales/en-US/messages.ftl
greeting = Hello!

# locales/fr/messages.ftl
greeting = Bonjour !
```

```tsx
import React, { useState, useCallback } from "react";
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization, LocalizationProvider, Localized } from "@fluent/react";

const AVAILABLE = ["en-US", "fr", "de"];
const MESSAGES: Record<string, string> = { /* ... */ };

function App() {
  const [currentLocales, setCurrentLocales] = useState(() =>
    negotiateLanguages(navigator.languages, AVAILABLE, {
      defaultLocale: "en-US",
    })
  );

  const l10n = new ReactLocalization(
    currentLocales.map((locale) => {
      const bundle = new FluentBundle(locale);
      bundle.addResource(new FluentResource(MESSAGES[locale]));
      return bundle;
    })
  );

  const switchLocale = useCallback((locale: string) => {
    setCurrentLocales(
      negotiateLanguages([locale], AVAILABLE, { defaultLocale: "en-US" })
    );
  }, []);

  return (
    <LocalizationProvider l10n={l10n}>
      <button onClick={() => switchLocale("fr")}>Fran&ccedil;ais</button>
      <button onClick={() => switchLocale("en-US")}>English</button>
      <Localized id="greeting">
        <p>Hello!</p>
      </Localized>
    </LocalizationProvider>
  );
}
```

When `currentLocales` changes, React re-renders the provider with a new `ReactLocalization` instance, and all `<Localized>` components automatically re-translate.

---

## SSR with Custom parseMarkup

```typescript
import { JSDOM } from "jsdom";
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { ReactLocalization } from "@fluent/react";

function parseMarkup(str: string): Node[] {
  const dom = new JSDOM(`<body>${str}</body>`);
  return Array.from(dom.window.document.body.childNodes);
}

const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(`welcome = Hello <em>World</em>!`));

const l10n = new ReactLocalization([bundle], parseMarkup);
```

ALWAYS provide a custom `parseMarkup` when rendering on the server. The default parser relies on `<template>` which does not exist in Node.js.
