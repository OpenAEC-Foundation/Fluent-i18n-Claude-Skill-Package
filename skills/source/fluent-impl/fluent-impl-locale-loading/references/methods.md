# methods.md — Locale Loading API Reference

## Fetch Pattern

The standard browser `fetch()` API is used to load FTL files from a server. There is no Fluent-specific loader — FTL files are plain text.

### fetchMessages()

```typescript
async function fetchMessages(locale: string): Promise<string> {
  const url = `/locales/${locale}/messages.ftl`;
  const response = await fetch(url);
  if (!response.ok) {
    throw new Error(`Failed to fetch ${url}: ${response.status}`);
  }
  return response.text();
}
```

**Key points:**
- `fetch()` returns a `Response` object
- `response.text()` extracts the FTL content as a plain string
- ALWAYS check `response.ok` before calling `.text()` — a 404 returns an empty bundle with no warning otherwise
- The URL pattern `/locales/{locale}/messages.ftl` is a convention, not a requirement

### fetchMultipleFiles()

```typescript
async function fetchMultipleFiles(
  locale: string,
  fileNames: string[]
): Promise<Map<string, string>> {
  const results = new Map<string, string>();
  await Promise.all(
    fileNames.map(async (fileName) => {
      const url = `/locales/${locale}/${fileName}`;
      const response = await fetch(url);
      if (response.ok) {
        results.set(fileName, await response.text());
      } else {
        console.warn(`Missing FTL file: ${url}`);
      }
    })
  );
  return results;
}
```

---

## FluentResource

```typescript
class FluentResource {
  body: Array<Message | Term>;
  constructor(source: string);
}
```

`FluentResource` parses an FTL string into an internal representation optimized for runtime use.

**Constructor:**
- Accepts a single `string` argument containing FTL syntax
- Parsing is synchronous and happens immediately in the constructor
- Parse errors are per-message — one bad message does NOT prevent others from being parsed
- The internal parser is optimized for runtime (smaller and faster than `@fluent/syntax`)

**Example:**
```typescript
import { FluentResource } from "@fluent/bundle";

const resource = new FluentResource(`
hello = Hello, world!
welcome = Welcome, { $userName }!
-brand-name = Acme Corp
`);

console.log(resource.body.length); // 3 (2 messages + 1 term)
```

**NEVER** use `FluentResource.fromString()` — this static method was removed in v0.14.0. ALWAYS use the constructor directly.

---

## FluentBundle Constructor

```typescript
class FluentBundle {
  constructor(
    locales: string | Array<string>,
    options?: {
      functions?: Record<string, FluentFunction>;
      useIsolating?: boolean;
      transform?: TextTransform;
    }
  );
}
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `locales` | `string \| Array<string>` | (required) | BCP 47 locale identifier(s) |
| `functions` | `Record<string, FluentFunction>` | `{}` | Custom functions available in FTL |
| `useIsolating` | `boolean` | `true` | Wrap placeables in Unicode isolation marks |
| `transform` | `TextTransform` | `(v) => v` | Transform for all text elements |

**Example:**
```typescript
import { FluentBundle } from "@fluent/bundle";

const bundle = new FluentBundle("en-US");
const bundleMultiLocale = new FluentBundle(["sr-Latn", "sr-Cyrl"]);
```

---

## addResource()

```typescript
addResource(
  res: FluentResource,
  options?: { allowOverrides?: boolean }
): Array<Error>
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `res` | `FluentResource` | (required) | A parsed FTL resource |
| `allowOverrides` | `boolean` | `false` | If `true`, duplicate message IDs overwrite existing ones |

**Return value:** An array of `Error` objects for messages with syntax errors. An empty array means all messages were added successfully.

**Behavior with duplicates:**
- `allowOverrides: false` (default): duplicate IDs produce an error in the returned array, and the original message is kept
- `allowOverrides: true`: duplicate IDs silently replace the original message

**Example — adding multiple resources to one bundle:**
```typescript
const bundle = new FluentBundle("en-US");

// First resource: core messages
const coreResource = new FluentResource(`
app-title = My Application
nav-home = Home
nav-settings = Settings
`);
const coreErrors = bundle.addResource(coreResource);

// Second resource: page-specific messages
const pageResource = new FluentResource(`
settings-theme = Theme
settings-language = Language
settings-save = Save Changes
`);
const pageErrors = bundle.addResource(pageResource);

// All messages from both resources are now in the same bundle
```

**Example — overriding messages:**
```typescript
const bundle = new FluentBundle("en-US");

const base = new FluentResource(`greeting = Hello`);
bundle.addResource(base);

const override = new FluentResource(`greeting = Hi there`);
bundle.addResource(override, { allowOverrides: true });
// "greeting" now resolves to "Hi there"
```

---

## ReactLocalization Constructor

```typescript
class ReactLocalization {
  constructor(
    bundles: Iterable<FluentBundle>,
    parseMarkup?: MarkupParser | null,
    reportError?: (error: Error) => void
  );
}
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `bundles` | `Iterable<FluentBundle>` | (required) | Array or generator of bundles |
| `parseMarkup` | `MarkupParser \| null` | `createParseMarkup()` | Custom markup parser for SSR |
| `reportError` | `(error: Error) => void` | `console.warn` | Error reporting callback |

The constructor wraps the `bundles` iterable in `CachedSyncIterable.from()`. This means:
- Arrays are passed through unchanged
- Generators are iterated once, and results are cached for subsequent lookups
- The first bundle in the iterable is the preferred locale; subsequent bundles are fallbacks

**Example:**
```typescript
import { ReactLocalization } from "@fluent/react";
import { FluentBundle } from "@fluent/bundle";

// From an array
const l10n = new ReactLocalization(bundleArray);

// From a generator (automatically cached)
function* generateBundles(): Generator<FluentBundle> {
  yield createBundle("en-US");
  yield createBundle("fr");
}
const l10n = new ReactLocalization(generateBundles());
```

---

## CachedSyncIterable

```typescript
import { CachedSyncIterable } from "cached-iterable";

class CachedSyncIterable<T> implements Iterable<T> {
  static from<T>(iterable: Iterable<T>): CachedSyncIterable<T>;
}
```

`CachedSyncIterable` wraps a generator or iterable so that:
1. First iteration consumes the generator and caches each yielded value
2. Subsequent iterations read from the cache without re-executing the generator
3. If the first iteration stops early (message found in first bundle), remaining generator values are not consumed until needed

This is critical for the lazy bundle pattern — without `CachedSyncIterable`, a generator-based bundle sequence would be exhausted after the first message lookup, and all subsequent lookups would find no bundles.

`ReactLocalization` applies `CachedSyncIterable.from()` automatically. You do NOT need to wrap generators manually when using `@fluent/react`.

**Direct usage (without @fluent/react):**
```typescript
import { CachedSyncIterable } from "cached-iterable";

function* generateBundles(): Generator<FluentBundle> {
  yield createBundle("en-US");
  yield createBundle("fr");
  yield createBundle("de");
}

const bundles = CachedSyncIterable.from(generateBundles());

// First iteration: creates en-US bundle, finds message, stops
for (const bundle of bundles) {
  if (bundle.hasMessage("hello")) break;
}

// Second iteration: reads en-US from cache, does NOT re-create it
for (const bundle of bundles) {
  if (bundle.hasMessage("goodbye")) break;
}
```
