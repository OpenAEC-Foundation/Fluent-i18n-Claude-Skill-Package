# fluent-impl-react: Anti-Patterns

## AP-1: Creating ReactLocalization in Render Without Memoization

### Problem

Creating a new `ReactLocalization` instance on every render causes all `<Localized>` components and `useLocalization` consumers to re-render unnecessarily.

### Wrong

```tsx
function App() {
  // WRONG: new instance every render
  const l10n = new ReactLocalization(generateBundles(locales));

  return (
    <LocalizationProvider l10n={l10n}>
      <Content />
    </LocalizationProvider>
  );
}
```

### Correct

```tsx
function App() {
  // CORRECT: stable instance via state
  const [l10n] = useState(
    () => new ReactLocalization(generateBundles(locales))
  );

  return (
    <LocalizationProvider l10n={l10n}>
      <Content />
    </LocalizationProvider>
  );
}
```

Or when locales change at runtime, tie the instance to locale state:

```tsx
function App() {
  const [locales, setLocales] = useState(["en-US"]);

  // Recreated only when locales change
  const l10n = useMemo(
    () => new ReactLocalization(generateBundles(locales)),
    [locales]
  );

  return (
    <LocalizationProvider l10n={l10n}>
      <Content />
    </LocalizationProvider>
  );
}
```

**Rule:** ALWAYS use `useState` or `useMemo` for `ReactLocalization` instances. NEVER create them as unguarded expressions in render.

---

## AP-2: Missing attrs Declaration

### Problem

Translated attributes are ONLY applied when explicitly declared as `true` in the `attrs` prop. Omitting `attrs` causes translated attributes to be silently ignored -- the component renders without errors but attributes remain untranslated.

### Wrong

```ftl
submit-btn = Submit
    .title = Click to submit
    .aria-label = Submit the form
```

```tsx
// WRONG: no attrs prop -- title and aria-label are NOT translated
<Localized id="submit-btn">
  <button title="Click to submit" aria-label="Submit the form">Submit</button>
</Localized>
```

### Correct

```tsx
// CORRECT: explicitly opt in to each attribute
<Localized id="submit-btn" attrs={{ title: true, "aria-label": true }}>
  <button title="Click to submit" aria-label="Submit the form">Submit</button>
</Localized>
```

**Rule:** ALWAYS declare every attribute you want translated in the `attrs` prop. This is a security measure -- translations cannot inject arbitrary attributes onto DOM elements.

---

## AP-3: Using withLocalization Instead of useLocalization

### Problem

The `withLocalization` HOC does NOT support ref forwarding (open issue #598). Components wrapped with it cannot receive refs, breaking patterns like `React.forwardRef`, focus management, and integration with third-party libraries.

### Wrong

```tsx
// WRONG: no ref forwarding support
import { withLocalization, WithLocalizationProps } from "@fluent/react";

function Input({ getString, ...props }: WithLocalizationProps & InputProps) {
  return <input placeholder={getString("search-placeholder")} {...props} />;
}

// Ref will NOT reach the <input> element
export default withLocalization(Input);
```

### Correct

```tsx
// CORRECT: hook-based, ref works naturally
import { useLocalization } from "@fluent/react";
import { forwardRef } from "react";

const Input = forwardRef<HTMLInputElement, InputProps>((props, ref) => {
  const { l10n } = useLocalization();

  return (
    <input
      ref={ref}
      placeholder={l10n.getString("search-placeholder")}
      {...props}
    />
  );
});
```

**Rule:** ALWAYS use the `useLocalization` hook in functional components. NEVER use `withLocalization` in new code.

---

## AP-4: Post-Translation String Manipulation

### Problem

Concatenating, splitting, or otherwise manipulating translated strings breaks localization. Different languages have different word orders, grammar rules, and sentence structures. String manipulation assumes English word order.

### Wrong

```tsx
// WRONG: concatenation assumes English word order
const { l10n } = useLocalization();
const greeting = l10n.getString("greeting") + ", " + userName + "!";

// WRONG: template literal with translated fragment
const message = `${l10n.getString("welcome")} ${l10n.getString("back")}`;

// WRONG: splitting translated string
const parts = l10n.getString("full-name").split(" ");
```

### Correct

```ftl
greeting = Welcome, { $userName }!
welcome-back = Welcome back!
```

```tsx
// CORRECT: let FTL handle all composition
const { l10n } = useLocalization();
const greeting = l10n.getString("greeting", { userName });

// CORRECT: single message with all context
const message = l10n.getString("welcome-back");
```

**Rule:** NEVER manipulate translated strings after formatting. Treat translation output as opaque. ALWAYS let FTL handle string composition through placeables (`{ $var }`) and selectors.

---

## AP-5: Ignoring addResource Errors

### Problem

`bundle.addResource()` returns an array of errors for malformed FTL. Ignoring these leads to silent translation failures -- messages appear as their IDs instead of translated text, with no indication of why.

### Wrong

```typescript
// WRONG: errors silently discarded
const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(ftlContent));
```

### Correct

```typescript
// CORRECT: errors logged for debugging
const bundle = new FluentBundle("en-US");
const errors = bundle.addResource(new FluentResource(ftlContent));
if (errors.length) {
  errors.forEach((e) => console.warn(`FTL parse error in en-US:`, e));
}
```

**Rule:** ALWAYS check and log the return value of `bundle.addResource()`. Partial translations are acceptable, but errors MUST be visible during development.

---

## AP-6: Not Providing parseMarkup for SSR

### Problem

The default DOM overlay parser uses the browser's `<template>` element, which does not exist in Node.js. Server-side rendering with `elems` props fails silently or throws errors.

### Wrong

```typescript
// WRONG: default parseMarkup fails on server
const l10n = new ReactLocalization(bundles);
// renderToString() with <Localized elems={...}> will break
```

### Correct

```typescript
import { JSDOM } from "jsdom";

function parseMarkup(str: string): Node[] {
  const dom = new JSDOM(`<body>${str}</body>`);
  return Array.from(dom.window.document.body.childNodes);
}

// CORRECT: custom parser for server environment
const l10n = new ReactLocalization(bundles, parseMarkup);
```

**Rule:** ALWAYS provide a custom `parseMarkup` function when using `@fluent/react` in a server-side rendering context.

---

## AP-7: Changing Translation ID Without Semantic Change

### Problem

Translation IDs are contracts with translators. Changing an ID without changing the meaning forces unnecessary re-translation. Conversely, keeping the same ID when the meaning changes means translators are never notified that re-translation is needed.

### Wrong

```ftl
# WRONG: renamed ID, same meaning -- forces unnecessary re-translation
# Before:
welcome = Welcome!
# After:
welcome-message = Welcome!

# WRONG: same ID, different meaning -- translators not notified
# Before:
delete-confirm = Delete this item?
# After (meaning changed but ID unchanged):
delete-confirm = Permanently delete this item and all its data?
```

### Correct

```ftl
# CORRECT: ID unchanged, meaning unchanged
welcome = Welcome!

# CORRECT: new ID signals meaning change to translators
delete-confirm-permanent = Permanently delete this item and all its data?
```

**Rule:** ALWAYS change the message ID when the meaning changes. NEVER change the ID when the meaning stays the same.
