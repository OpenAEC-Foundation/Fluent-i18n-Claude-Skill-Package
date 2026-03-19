# fluent-impl-react: API Reference

## LocalizationProvider Props

```typescript
interface LocalizationProviderProps {
  l10n: ReactLocalization;
  children: ReactNode;
}
```

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `l10n` | `ReactLocalization` | Yes | Localization instance containing bundles |
| `children` | `ReactNode` | Yes | Component tree that consumes translations |

---

## LocalizedProps

```typescript
interface LocalizedProps {
  id: string;
  attrs?: Record<string, boolean>;
  vars?: Record<string, FluentVariable>;
  elems?: Record<string, ReactElement>;
  children?: ReactElement;
}
```

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `id` | `string` | Yes | FTL message identifier to look up |
| `attrs` | `Record<string, boolean>` | No | Map of attribute names to translate. ONLY attributes set to `true` are applied. |
| `vars` | `Record<string, FluentVariable>` | No | Variables passed to the FTL message (maps to `$varName` in FTL) |
| `elems` | `Record<string, ReactElement>` | No | React elements for DOM overlay markup (maps to `<elemName>` in FTL) |
| `children` | `ReactElement` | No | Fallback element displayed when translation is not found |

`FluentVariable` is `string | number | Date | FluentType`.

---

## useLocalization Return Type

```typescript
function useLocalization(): { l10n: ReactLocalization }
```

Returns an object with a single property `l10n` containing the `ReactLocalization` instance from the nearest `<LocalizationProvider>` ancestor.

---

## ReactLocalization Class

### Constructor

```typescript
class ReactLocalization {
  constructor(
    bundles: Iterable<FluentBundle>,
    parseMarkup?: MarkupParser | null,
    reportError?: (error: Error) => void
  )
}
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `bundles` | `Iterable<FluentBundle>` | -- | Iterable of bundles in locale preference order |
| `parseMarkup` | `MarkupParser \| null` | `createParseMarkup()` | Custom markup parser for DOM overlays |
| `reportError` | `(error: Error) => void` | `console.warn` | Error reporting callback for missing messages and format errors |

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `bundles` | `Iterable<FluentBundle>` | Bundle sequence (wrapped in `CachedSyncIterable`) |
| `parseMarkup` | `MarkupParser \| null` | Active markup parser |
| `reportError` | `(error: Error) => void` | Active error reporter |

### Methods

#### getBundle

```typescript
getBundle(id: string): FluentBundle | null
```

Returns the first `FluentBundle` in the sequence that contains a message with the given `id`. Returns `null` if no bundle contains the message.

#### getString

```typescript
getString(
  id: string,
  vars?: Record<string, FluentVariable> | null,
  fallback?: string
): string
```

Formats a message to a plain string. Resolution order:
1. First bundle containing `id` formats the message with `vars`
2. If no bundle contains `id`, returns `fallback` if provided
3. If no `fallback`, returns `id` itself

ALWAYS reports an error via `reportError` when the message is not found.

#### getElement

```typescript
getElement(
  sourceElement: ReactElement,
  id: string,
  args?: {
    vars?: Record<string, FluentVariable>;
    elems?: Record<string, ReactElement>;
    attrs?: Record<string, boolean>;
  }
): ReactElement
```

Formats a message into a React element. Applies DOM overlays (via `elems`), variable substitution (via `vars`), and attribute mapping (via `attrs`). Used internally by the `<Localized>` component.

#### areBundlesEmpty

```typescript
areBundlesEmpty(): boolean
```

Returns `true` if the bundle iterable contains zero bundles. Useful for conditional rendering while translations are loading.

---

## MarkupParser Type

```typescript
type MarkupParser = (str: string) => Array<Node>;
```

Each returned `Node` MUST have at minimum:
- `nodeName: string` -- element tag name or `#text` for text nodes
- `textContent: string | null` -- text content of the node

The default parser uses the browser's `<template>` element. For SSR or React Native, provide a custom parser using `jsdom` or `cheerio`.

---

## WithLocalizationProps (Legacy)

```typescript
interface WithLocalizationProps {
  getString: (id: string, vars?: Record<string, FluentVariable>, fallback?: string) => string;
  getElement: (sourceElement: ReactElement, id: string, args?: object) => ReactElement;
}
```

Injected by the `withLocalization` HOC. Does NOT support ref forwarding. ALWAYS prefer `useLocalization` hook instead.
