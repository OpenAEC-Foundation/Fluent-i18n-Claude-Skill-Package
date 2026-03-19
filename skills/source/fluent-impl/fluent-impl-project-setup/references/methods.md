# methods.md — Package Details and API Signatures

## Package Registry

### @fluent/bundle

| Property | Value |
|----------|-------|
| npm | `@fluent/bundle` |
| Current version | 0.19.1 (April 2, 2025) |
| License | Apache-2.0 OR MIT |
| Entry (CJS) | `./index.js` |
| Entry (ESM) | `./esm/index.js` |
| Types | `./esm/index.d.ts` |
| Node.js | `^20.19 \|\| ^22.12 \|\| >=24` |
| Dependencies | None |
| Requires `Intl` APIs | `Intl.DateTimeFormat`, `Intl.NumberFormat`, `Intl.PluralRules` |

**Exports:**
- Classes: `FluentBundle`, `FluentResource`, `FluentType`, `FluentNone`, `FluentNumber`, `FluentDateTime`
- Types: `FluentValue`, `FluentVariable`, `FluentFunction`, `TextTransform`, `Message`, `Scope`

### @fluent/react

| Property | Value |
|----------|-------|
| npm | `@fluent/react` |
| Current version | 0.15.2 |
| License | Apache-2.0 |
| Entry (CJS) | `./index.js` |
| Entry (ESM) | `./esm/index.js` |
| Types | `./esm/index.d.ts` |
| Node.js | `^20.19 \|\| ^22.12 \|\| >=24` |
| Dependencies | `@fluent/sequence` ^0.8.0, `cached-iterable` ^0.3.0 |
| Peer dependencies | `@fluent/bundle` >=0.16.0, `react` >=16.8.0 |

**Exports:**
- Components: `LocalizationProvider`, `Localized`
- Hook: `useLocalization`
- HOC: `withLocalization`
- Class: `ReactLocalization`
- Types: `LocalizedProps`, `WithLocalizationProps`, `MarkupParser`

### @fluent/langneg

| Property | Value |
|----------|-------|
| npm | `@fluent/langneg` |
| Current version | 0.7.0 |
| License | Apache-2.0 |
| Dependencies | None |
| Node.js | `^20.19 \|\| ^22.12 \|\| >=24` |

**Exports:**
- `negotiateLanguages(requested, available, options?)` — locale matching
- `acceptedLanguages(header)` — parse HTTP Accept-Language header
- `filterMatches(requested, available, strategy)` — low-level matching engine

### @fluent/syntax (optional, devDependency)

| Property | Value |
|----------|-------|
| npm | `@fluent/syntax` |
| Current version | 0.19.0 |
| License | Apache-2.0 |
| Node.js | `^20.19 \|\| ^22.12 \|\| >=24` |

**Exports:**
- `FluentParser` — parse FTL source to AST (`Resource`)
- `FluentSerializer` — serialize AST back to FTL string
- `serializeExpression`, `serializeVariantKey` — standalone serializers
- Full AST types: `Resource`, `Message`, `Term`, `Pattern`, `Identifier`, etc.

---

## Core API Signatures

### FluentBundle

```typescript
class FluentBundle {
  constructor(
    locales: string | Array<string>,
    options?: {
      functions?: Record<string, FluentFunction>;
      useIsolating?: boolean;  // default: true
      transform?: TextTransform;
    }
  );

  addResource(
    res: FluentResource,
    options?: { allowOverrides?: boolean }  // default: false
  ): Array<Error>;

  getMessage(id: string): Message | undefined;
  hasMessage(id: string): boolean;

  formatPattern(
    pattern: Pattern,
    args?: Record<string, FluentVariable> | null,
    errors?: Array<Error> | null
  ): string;
}
```

### FluentResource

```typescript
class FluentResource {
  body: Array<Message | Term>;
  constructor(source: string);
}
```

### ReactLocalization

```typescript
class ReactLocalization {
  constructor(
    bundles: Iterable<FluentBundle>,
    parseMarkup?: MarkupParser | null,
    reportError?: (error: Error) => void
  );

  getBundle(id: string): FluentBundle | null;
  getString(
    id: string,
    vars?: Record<string, FluentVariable> | null,
    fallback?: string
  ): string;
  getElement(
    sourceElement: ReactElement,
    id: string,
    args?: { vars?; elems?; attrs? }
  ): ReactElement;
  areBundlesEmpty(): boolean;
}
```

### negotiateLanguages

```typescript
function negotiateLanguages(
  requestedLocales: Readonly<Array<string>>,
  availableLocales: Readonly<Array<string>>,
  options?: {
    strategy?: "filtering" | "matching" | "lookup";
    defaultLocale?: string;
  }
): Array<string>;
```

### FluentParser (from @fluent/syntax)

```typescript
class FluentParser {
  constructor(options?: { withSpans?: boolean });  // default: true
  parse(source: string): Resource;
  parseEntry(source: string): Entry;
}
```

---

## Version Compatibility Matrix

| @fluent/bundle | @fluent/react | React | Node.js |
|----------------|---------------|-------|---------|
| 0.19.x | 0.15.x | >=16.8.0 | ^20.19 \|\| ^22.12 \|\| >=24 |
| 0.18.x | 0.15.x | >=16.8.0 | >=14 |
| 0.17.x | 0.15.x | >=16.8.0 | >=12 |
| 0.16.x | 0.14.x | >=16.8.0 | >=10 |

**ALWAYS** use @fluent/bundle 0.18+ with @fluent/react 0.15+ for current projects.

---

## Type Reference

```typescript
type FluentVariable = FluentValue | TemporalObject | string | number | Date;
type FluentValue = FluentType<unknown> | string;
type FluentFunction = (
  positional: Array<FluentValue>,
  named: Record<string, FluentValue>
) => FluentValue;
type TextTransform = (text: string) => string;
type MarkupParser = (str: string) => Array<Node>;

interface Message {
  value: Pattern | null;
  attributes: Record<string, Pattern>;
}
```
