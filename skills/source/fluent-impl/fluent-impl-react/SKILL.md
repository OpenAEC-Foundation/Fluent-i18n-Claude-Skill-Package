---
name: fluent-impl-react
description: >
  Use when integrating Project Fluent with React via @fluent/react for translated UI components.
  Prevents misconfigured LocalizationProvider, broken DOM overlays, and incorrect Localized prop usage.
  Covers provider setup, Localized component, useLocalization hook, DOM overlays, and bundle generation.
  Keywords: LocalizationProvider, Localized, useLocalization, DOM overlay, ReactLocalization, @fluent/react.
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/react 0.15+, @fluent/bundle 0.16+, React 16.8+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-impl-react

## Quick Reference

### Package Stack

| Package | Version | Role |
|---------|---------|------|
| `@fluent/react` | 0.15+ | React bindings (Provider, Localized, hooks) |
| `@fluent/bundle` | 0.16+ (peer dep) | FluentBundle + FluentResource for parsing FTL |
| `react` | 16.8+ (peer dep) | Hooks support required |

```bash
npm install @fluent/bundle @fluent/react
```

### Public API Exports

| Export | Type | Purpose |
|--------|------|---------|
| `LocalizationProvider` | Component | Distributes translations via React Context |
| `Localized` | Component | Declarative translation rendering with overlays |
| `useLocalization` | Hook | Imperative access to `ReactLocalization` instance |
| `ReactLocalization` | Class | Core engine: stores, caches, queries bundles |
| `withLocalization` | HOC | Legacy wrapper -- does NOT support ref forwarding |
| `MarkupParser` | Type | `(str: string) => Array<Node>` for custom parsing |

### Critical Warnings

**ALWAYS** declare `attrs` explicitly on `<Localized>` when translating HTML attributes. Omitting `attrs` causes translated attributes to be silently ignored -- this is a security measure preventing translations from injecting arbitrary attributes.

**NEVER** manipulate translated strings after formatting. Treat translation output as opaque. Let FTL handle all string composition through placeables and selectors.

**NEVER** create a new `ReactLocalization` instance inside a render function without memoization. This causes every `<Localized>` component to re-render on every parent render cycle.

**ALWAYS** use the `useLocalization` hook instead of the `withLocalization` HOC in functional components. The HOC does NOT support ref forwarding (open issue #598).

---

## LocalizationProvider

The provider distributes a `ReactLocalization` instance to all descendant components via React Context. Every `<Localized>` and `useLocalization()` call MUST be inside a `<LocalizationProvider>`.

```tsx
import { LocalizationProvider } from "@fluent/react";
import { ReactLocalization } from "@fluent/react";

<LocalizationProvider l10n={l10n}>
  <App />
</LocalizationProvider>
```

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `l10n` | `ReactLocalization` | Yes | Localization instance with bundles |

**Behavior:** Manages subscriptions for all nested `<Localized>` elements. Establishes a fallback chain -- if a translation is missing in one bundle, the system proceeds to the next bundle in sequence.

---

## Localized Component

The primary declarative API for rendering translations. Wraps a React element and replaces its text content (and optionally attributes) with translations.

### Props

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `id` | `string` | Yes | FTL message identifier |
| `vars` | `Record<string, FluentVariable>` | No | Variables passed to the translation |
| `elems` | `Record<string, ReactElement>` | No | React elements for DOM overlay markup |
| `attrs` | `Record<string, boolean>` | No | Attributes to translate (explicit opt-in) |
| `children` | `ReactElement` | No | Fallback element shown if translation missing |

### Basic Usage

```ftl
hello-world = Hello, World!
```

```tsx
<Localized id="hello-world">
  <p>Hello, World!</p>
</Localized>
```

The child element serves as the fallback -- displayed when the translation is not found in any bundle.

### Variables

```ftl
welcome-user = Welcome, { $userName }!
```

```tsx
<Localized id="welcome-user" vars={{ userName: currentUser.name }}>
  <p>Welcome, {currentUser.name}!</p>
</Localized>
```

### DOM Overlays (elems)

The `elems` prop maps custom element names in FTL to real React components:

```ftl
create-account = <confirm>Create account</confirm> or <cancel>go back</cancel>.
```

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

**How it works internally:**
1. The library parses the translation using a `<template>` element, creating an inert `DocumentFragment`
2. Unknown elements (like `<confirm>`) become `HTMLUnknownElement` instances
3. The library matches these by name against the `elems` prop keys
4. Provided React elements are cloned with the translated text content as children

### Attribute Mapping (attrs)

```ftl
my-link = Something
    .title = Link to something
```

```tsx
<Localized id="my-link" attrs={{ title: true }}>
  <a href="/something" title="Link to something">Something</a>
</Localized>
```

Translated attributes are ONLY applied when explicitly declared as `true` in the `attrs` object. This is a security measure -- translations cannot inject arbitrary attributes onto DOM elements.

---

## useLocalization Hook

Provides imperative access to translations within functional components. Returns the `ReactLocalization` instance from context.

```typescript
import { useLocalization } from "@fluent/react";

function AlertButton() {
  const { l10n } = useLocalization();

  const handleClick = () => {
    alert(l10n.getString("confirm-delete", { itemName: "Report" }));
  };

  return <button onClick={handleClick}>Delete</button>;
}
```

**Return value:** `{ l10n: ReactLocalization }`

**Primary use cases:** `window.alert()`, `window.confirm()`, logging, validation messages, `document.title`, or any scenario where JSX rendering via `<Localized>` is not possible.

---

## ReactLocalization Class

The core engine that stores, caches, and queries a sequence of `FluentBundle` instances.

### Constructor

```typescript
new ReactLocalization(
  bundles: Iterable<FluentBundle>,
  parseMarkup?: MarkupParser | null,
  reportError?: (error: Error) => void
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `bundles` | -- | Bundle sequence (wrapped in `CachedSyncIterable` internally) |
| `parseMarkup` | `createParseMarkup()` | Custom parser for DOM overlay markup |
| `reportError` | `console.warn` | Error reporting callback |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `getBundle` | `(id: string) => FluentBundle \| null` | Find first bundle containing message `id` |
| `getString` | `(id: string, vars?: Record<string, FluentVariable> \| null, fallback?: string) => string` | Format message to string; returns fallback or id if not found |
| `getElement` | `(sourceElement: ReactElement, id: string, args?: { vars?, elems?, attrs? }) => ReactElement` | Format message into a React element with overlays and attributes |
| `areBundlesEmpty` | `() => boolean` | Check if bundle iterable is exhausted/empty |

**Bundle resolution:** Iterates through bundles in order (representing locale preference chain). The first bundle containing the requested message `id` wins. This enables graceful degradation -- missing translations fall through to the next locale.

---

## Bundle Generation Patterns

### Array-Based (Eager)

```typescript
function generateBundles(locales: string[]): FluentBundle[] {
  return locales.map((locale) => {
    const bundle = new FluentBundle(locale);
    bundle.addResource(new FluentResource(MESSAGES[locale]));
    return bundle;
  });
}
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
```

ALWAYS prefer the generator pattern for large locale sets -- fallback bundles are only created when needed. The `ReactLocalization` constructor wraps generators in `CachedSyncIterable.from()` so iteration happens only once.

### Async Loading

```typescript
async function fetchMessages(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/messages.ftl`);
  return response.text();
}

async function createLocalization(locales: string[]): Promise<ReactLocalization> {
  const bundles = await Promise.all(
    locales.map(async (locale) => {
      const messages = await fetchMessages(locale);
      const bundle = new FluentBundle(locale);
      bundle.addResource(new FluentResource(messages));
      return bundle;
    })
  );
  return new ReactLocalization(bundles);
}
```

**Critical limitation:** As of v0.15, all translations (including fallback locales) MUST be fetched before rendering `<LocalizationProvider>`. There is no built-in support for async/lazy fallback locale loading during render.

---

## SSR: Custom parseMarkup

When `<template>` is unavailable (server-side rendering, React Native), ALWAYS provide a custom `parseMarkup` function:

```typescript
import { JSDOM } from "jsdom";

function parseMarkup(str: string): Node[] {
  const dom = new JSDOM(`<body>${str}</body>`);
  return Array.from(dom.window.document.body.childNodes);
}

const l10n = new ReactLocalization(bundles, parseMarkup);
```

Each returned `Node` MUST have `nodeName` and `textContent` properties.

---

## withLocalization HOC (Legacy)

The `withLocalization` HOC exists but has a known limitation: it does NOT support ref forwarding (issue #598). ALWAYS prefer the `useLocalization` hook in functional components.

```tsx
// Legacy pattern -- avoid in new code
import { withLocalization, WithLocalizationProps } from "@fluent/react";

function MyComponent({ getString }: WithLocalizationProps) {
  return <p>{getString("my-message")}</p>;
}

export default withLocalization(MyComponent);
```

---

## Decision Trees

### Choosing a Translation API

```
Need translation in JSX?
├── Yes → Use <Localized> component
│   ├── Need rich markup? → Use elems prop for DOM overlays
│   ├── Need translated attributes? → Use attrs prop with explicit opt-in
│   └── Need variables? → Use vars prop
└── No → Use useLocalization() hook
    └── Call l10n.getString(id, vars, fallback)
```

### Choosing Bundle Generation Strategy

```
How many locales?
├── 2-3 locales → Array-based (eager) is fine
├── 4+ locales → Generator-based (lazy) to avoid unnecessary work
└── Remote FTL files? → Async fetch + Promise.all before provider mount
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- LocalizationProvider props, Localized props, useLocalization return, ReactLocalization constructor and methods, MarkupParser type
- [references/examples.md](references/examples.md) -- Provider setup, Localized basic/vars/elems/attrs, useLocalization imperative, bundle generation patterns
- [references/anti-patterns.md](references/anti-patterns.md) -- Creating ReactLocalization in render, missing attrs, withLocalization vs hook, post-translation manipulation

### Official Sources

- https://github.com/projectfluent/fluent.js/wiki/LocalizationProvider
- https://github.com/projectfluent/fluent.js/wiki/Localized
- https://github.com/projectfluent/fluent.js/wiki/useLocalization
- https://github.com/projectfluent/fluent.js/wiki/ReactLocalization
- https://github.com/projectfluent/fluent.js/wiki/React-Overlays
- https://github.com/projectfluent/fluent.js/wiki/React-Tutorial
