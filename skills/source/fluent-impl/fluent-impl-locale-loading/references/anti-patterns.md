# anti-patterns.md — FTL Loading Anti-Patterns

## AP-01: Creating FluentResource Per Message

### WRONG

```typescript
// Creates a new FluentResource for each individual message — massive overhead
for (const [id, text] of Object.entries(translations)) {
  const resource = new FluentResource(`${id} = ${text}`);
  bundle.addResource(resource);
}
```

### WHY THIS IS WRONG

Each `new FluentResource()` call invokes the FTL parser. The parser has initialization overhead (regex compilation, state setup) that is amortized over many messages in a single resource. Creating one resource per message means paying that cost hundreds of times instead of once.

### CORRECT

```typescript
// Batch all messages into a single FTL string and parse once
const ftlText = Object.entries(translations)
  .map(([id, text]) => `${id} = ${text}`)
  .join("\n");
const resource = new FluentResource(ftlText);
const errors = bundle.addResource(resource);
if (errors.length > 0) {
  errors.forEach((e) => console.warn("FTL parse error:", e));
}
```

Or better: load messages from a single `.ftl` file that already contains all messages.

---

## AP-02: Ignoring addResource Errors

### WRONG

```typescript
// Silently discarding parse errors — broken translations go unnoticed
bundle.addResource(new FluentResource(ftlText));
// No error handling at all
```

### WHY THIS IS WRONG

`addResource()` returns an array of `Error` objects for messages that had syntax errors. These errors indicate that specific messages failed to parse and will NOT be available for formatting. Without logging, you have no way to know which translations are broken until a user reports missing text.

### CORRECT

```typescript
const errors = bundle.addResource(new FluentResource(ftlText));
if (errors.length > 0) {
  errors.forEach((error) => {
    console.warn("FTL parse error:", error.message);
  });
  // In development, you may want to throw:
  // if (process.env.NODE_ENV === "development") {
  //   throw new Error(`FTL has ${errors.length} parse error(s)`);
  // }
}
```

ALWAYS log `addResource()` errors. In development environments, consider making them fatal to catch issues early.

---

## AP-03: Blocking Render with Synchronous Loading

### WRONG

```typescript
// Using XMLHttpRequest synchronously — blocks the main thread and freezes the UI
function loadFtlSync(locale: string): string {
  const xhr = new XMLHttpRequest();
  xhr.open("GET", `/locales/${locale}/messages.ftl`, false); // false = synchronous
  xhr.send();
  return xhr.responseText;
}

// This blocks rendering until ALL locales are loaded
const bundles = locales.map((locale) => {
  const ftlText = loadFtlSync(locale);
  const bundle = new FluentBundle(locale);
  bundle.addResource(new FluentResource(ftlText));
  return bundle;
});
```

### WHY THIS IS WRONG

Synchronous network requests block the browser's main thread. The UI freezes completely — no rendering, no interaction, no animations — until the request completes. If the server is slow or the network drops, the page appears broken. Modern browsers actively warn against or restrict synchronous XHR on the main thread.

### CORRECT

```typescript
async function loadBundles(locales: string[]): Promise<FluentBundle[]> {
  return Promise.all(
    locales.map(async (locale) => {
      const response = await fetch(`/locales/${locale}/messages.ftl`);
      const ftlText = await response.text();
      const bundle = new FluentBundle(locale);
      const errors = bundle.addResource(new FluentResource(ftlText));
      if (errors.length > 0) {
        errors.forEach((e) => console.warn(`FTL error [${locale}]:`, e));
      }
      return bundle;
    })
  );
}

// Show a loading state while translations load
function App() {
  const [l10n, setL10n] = useState<ReactLocalization | null>(null);

  useEffect(() => {
    loadBundles(["en-US", "fr"]).then((bundles) => {
      setL10n(new ReactLocalization(bundles));
    });
  }, []);

  if (!l10n) return <div>Loading...</div>;

  return (
    <LocalizationProvider l10n={l10n}>
      <MainContent />
    </LocalizationProvider>
  );
}
```

ALWAYS load FTL files asynchronously with `fetch()`. Show a loading indicator until translations are ready. NEVER use synchronous XHR.

---

## AP-04: Not Caching Bundles on Locale Switch

### WRONG

```typescript
// Re-fetches and re-parses FTL every time the user switches locales
async function switchLocale(locale: string) {
  const response = await fetch(`/locales/${locale}/messages.ftl`);
  const ftlText = await response.text();
  const bundle = new FluentBundle(locale);
  bundle.addResource(new FluentResource(ftlText));
  setL10n(new ReactLocalization([bundle]));
}
```

### WHY THIS IS WRONG

If a user switches from English to French and back to English, this code fetches and parses the English FTL file again. Network requests and FTL parsing are not free — this creates unnecessary latency and bandwidth usage, especially on slow connections.

### CORRECT

```typescript
const bundleCache = new Map<string, FluentBundle>();

async function switchLocale(locale: string) {
  if (!bundleCache.has(locale)) {
    const response = await fetch(`/locales/${locale}/messages.ftl`);
    const ftlText = await response.text();
    const bundle = new FluentBundle(locale);
    const errors = bundle.addResource(new FluentResource(ftlText));
    if (errors.length > 0) {
      errors.forEach((e) => console.warn(`FTL error [${locale}]:`, e));
    }
    bundleCache.set(locale, bundle);
  }

  setL10n(new ReactLocalization([bundleCache.get(locale)!]));
}
```

ALWAYS cache `FluentBundle` instances in a `Map` when locale switching is supported. FTL content does not change at runtime — once parsed, the bundle can be reused indefinitely.

---

## AP-05: Using a Bare Generator Without CachedSyncIterable

### WRONG

```typescript
function* generateBundles(): Generator<FluentBundle> {
  yield createBundle("en-US");
  yield createBundle("fr");
}

// Passing a bare generator to manual iteration code
const bundles = generateBundles();

// First lookup — consumes the generator
for (const bundle of bundles) {
  if (bundle.hasMessage("hello")) break;
}

// Second lookup — generator is exhausted, finds NOTHING
for (const bundle of bundles) {
  if (bundle.hasMessage("goodbye")) break; // Never executes
}
```

### WHY THIS IS WRONG

JavaScript generators are single-use iterators. Once a generator is iterated (partially or fully), it cannot be restarted. A second `for...of` loop over the same generator object iterates zero elements. This means the first message lookup works, but all subsequent lookups silently fail.

### CORRECT

```typescript
import { CachedSyncIterable } from "cached-iterable";

function* generateBundles(): Generator<FluentBundle> {
  yield createBundle("en-US");
  yield createBundle("fr");
}

// Wrap in CachedSyncIterable — results are cached for re-iteration
const bundles = CachedSyncIterable.from(generateBundles());

// Both lookups work correctly
for (const bundle of bundles) {
  if (bundle.hasMessage("hello")) break;
}
for (const bundle of bundles) {
  if (bundle.hasMessage("goodbye")) break;
}
```

Note: `ReactLocalization` applies `CachedSyncIterable.from()` automatically in its constructor. This anti-pattern only applies when you use generators with the `@fluent/bundle` API directly, without `@fluent/react`.

---

## AP-06: Fetching FTL Without Checking Response Status

### WRONG

```typescript
async function loadFtl(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/messages.ftl`);
  return response.text(); // Returns HTML error page on 404!
}
```

### WHY THIS IS WRONG

`fetch()` does NOT throw on HTTP errors (404, 500). It resolves with a `Response` where `ok` is `false`. Calling `.text()` on a 404 response returns the server's error page HTML, which `FluentResource` then tries to parse as FTL. This produces zero valid messages with no obvious error — translations silently vanish.

### CORRECT

```typescript
async function loadFtl(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/messages.ftl`);
  if (!response.ok) {
    console.warn(`Failed to load FTL for ${locale}: HTTP ${response.status}`);
    return ""; // Return empty string — FluentResource handles it gracefully
  }
  return response.text();
}
```

ALWAYS check `response.ok` before calling `.text()`. Return an empty string on failure so that `new FluentResource("")` produces an empty but valid resource.

---

## AP-07: Loading All Locales Eagerly When Only One Is Needed

### WRONG

```typescript
// Fetches ALL available locales on startup, even though only 1-2 are needed
const ALL_LOCALES = ["en-US", "fr", "de", "es", "it", "ja", "ko", "zh-Hans"];

const bundles = await Promise.all(
  ALL_LOCALES.map(async (locale) => {
    const ftlText = await fetch(`/locales/${locale}/messages.ftl`).then(r => r.text());
    return createBundle(locale, ftlText);
  })
);
```

### WHY THIS IS WRONG

If the user's browser is set to `fr`, you typically only need `fr` and `en-US` (as fallback). Loading all 8 locales wastes bandwidth and delays initial render. Each additional FTL file adds a network request and parse time.

### CORRECT

```typescript
import { negotiateLanguages } from "@fluent/langneg";

const ALL_LOCALES = ["en-US", "fr", "de", "es", "it", "ja", "ko", "zh-Hans"];

// Only load negotiated locales (typically 1-3)
const needed = negotiateLanguages(navigator.languages, ALL_LOCALES, {
  defaultLocale: "en-US",
});

const bundles = await Promise.all(
  needed.map(async (locale) => {
    const ftlText = await fetch(`/locales/${locale}/messages.ftl`).then(r => r.text());
    return createBundle(locale, ftlText);
  })
);
```

ALWAYS use `negotiateLanguages()` to determine which locales the user actually needs before fetching FTL files. Load only the negotiated subset.
