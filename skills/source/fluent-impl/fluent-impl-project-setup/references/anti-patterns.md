# anti-patterns.md — What NOT To Do

## AP-1: Installing the Legacy `fluent` Package

```bash
# WRONG — the `fluent` package was renamed to @fluent/bundle in v0.13.0
npm install fluent

# CORRECT
npm install @fluent/bundle
```

**Why:** The `fluent` package on npm is the pre-rename version (last release: 0.12.x). It has a completely different API surface (`MessageContext`, `addMessages()`, `format()`). All documentation, tutorials, and community support target `@fluent/bundle`.

---

## AP-2: Missing Peer Dependencies

```bash
# WRONG — installing @fluent/react without its peer dependencies
npm install @fluent/react
# Results in: "peer dep missing: @fluent/bundle >=0.16.0"

# CORRECT — always install the complete stack
npm install @fluent/bundle @fluent/react @fluent/langneg
```

**Why:** `@fluent/react` requires `@fluent/bundle` as a peer dependency. Without it, `FluentBundle` and `FluentResource` are unavailable, and no translations can be created. npm v7+ auto-installs peers, but explicit installation ensures version control.

---

## AP-3: Using @fluent/syntax for Runtime Formatting

```typescript
// WRONG — @fluent/syntax is a tooling library, not a runtime formatter
import { FluentParser } from "@fluent/syntax";

const parser = new FluentParser();
const ast = parser.parse(ftlSource);
// ...manually walking the AST to extract translations

// CORRECT — use @fluent/bundle for runtime
import { FluentBundle, FluentResource } from "@fluent/bundle";

const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(ftlSource));
const msg = bundle.getMessage("hello");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value);
}
```

**Why:** `@fluent/syntax` produces a full AST for tooling purposes. It adds significant bundle size and provides no formatting capability. `@fluent/bundle` includes its own optimized runtime parser that is smaller and faster.

---

## AP-4: Flat FTL Files Without Locale Directories

```
# WRONG — all locales in one directory without structure
public/
├── en-US.ftl      # 500+ messages in one file
├── fr.ftl
└── de.ftl

# CORRECT — locale directories with feature-based files
public/locales/
├── en-US/
│   ├── main.ftl
│   ├── errors.ftl
│   └── settings.ftl
├── fr/
│   ├── main.ftl
│   ├── errors.ftl
│   └── settings.ftl
└── de/
    ├── main.ftl
    ├── errors.ftl
    └── settings.ftl
```

**Why:** A single FTL file per locale becomes unmanageable beyond ~50 messages. Feature-based splitting enables code-splitting (load only the FTL files needed for the current route), parallel translator workflow (different translators can work on different files), and easier review (changes are scoped to a feature).

---

## AP-5: Component-Based FTL Files

```
# WRONG — one FTL file per component
public/locales/en-US/
├── Header.ftl
├── Footer.ftl
├── LoginForm.ftl
├── SignupForm.ftl
├── UserProfile.ftl
├── SettingsPanel.ftl
└── ... (dozens of tiny files)

# CORRECT — one FTL file per feature area
public/locales/en-US/
├── main.ftl        # navigation, headers, common UI
├── auth.ftl        # login, signup, password reset
├── settings.ftl    # settings page
└── errors.ftl      # error messages
```

**Why:** Component-based splitting creates excessive HTTP requests (one per component), fragments related translations across many files, and couples translation organization to UI component structure (which changes more frequently than feature areas).

---

## AP-6: Non-Semantic Message IDs

```ftl
# WRONG — generic, meaningless IDs
msg1 = Welcome
btn2 = Submit
text = Enter your email
label = Password

# CORRECT — semantic component-element-action pattern
login-title = Welcome
login-button-submit = Submit
login-input-email-placeholder = Enter your email
login-input-password-label = Password
```

**Why:** Generic IDs (`msg1`, `btn2`) are impossible to maintain at scale. When a translator sees `msg1`, they have no context about where or how it appears. Semantic IDs convey location and purpose, enabling translators to provide appropriate translations without seeing the UI.

---

## AP-7: Hardcoding Locale Lists Without Negotiation

```typescript
// WRONG — hardcoded locale without negotiation
const bundle = new FluentBundle("en-US");
// User who speaks fr-FR gets English regardless

// CORRECT — negotiate against user preferences
import { negotiateLanguages } from "@fluent/langneg";

const negotiated = negotiateLanguages(
  navigator.languages,           // user preferences
  ["en-US", "fr", "de"],        // available locales
  { defaultLocale: "en-US" }    // fallback
);
// negotiated = ["fr", "en-US"] for a fr-FR user
```

**Why:** Without negotiation, the app ignores user locale preferences. `@fluent/langneg` handles subtag matching (fr-FR matches fr), preference ordering, and fallback chains automatically.

---

## AP-8: Placing FTL Files in src/ Instead of public/

```
# WRONG — FTL files bundled with source code
src/
├── l10n/
│   ├── en-US.ftl    # gets processed by bundler
│   └── fr.ftl
└── App.tsx

# CORRECT — FTL files served statically
public/
└── locales/
    ├── en-US/
    │   └── main.ftl  # served as-is via HTTP
    └── fr/
        └── main.ftl
src/
├── l10n/
│   └── index.ts      # fetches FTL via fetch()
└── App.tsx
```

**Why:** FTL files are plain text loaded at runtime via `fetch()`. Placing them in `src/` causes bundlers (Webpack, Vite) to process them incorrectly or require custom loader configuration. The `public/` directory serves files as-is, which is exactly what FTL loading requires.

---

## AP-9: Creating ReactLocalization on Every Render

```tsx
// WRONG — new ReactLocalization on every render
function App() {
  // This runs on EVERY render, re-creating the entire localization context
  const l10n = new ReactLocalization(generateBundles(locales));

  return (
    <LocalizationProvider l10n={l10n}>
      <Content />
    </LocalizationProvider>
  );
}

// CORRECT — use useState to persist across renders
function App() {
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

**Why:** Creating a new `ReactLocalization` instance triggers re-rendering of ALL `<Localized>` descendants. Using `useState` with an initializer function ensures the instance is created once and persisted across renders.

---

## AP-10: Skipping the Loading State

```tsx
// WRONG — rendering before translations are loaded
function App() {
  const [l10n, setL10n] = useState<ReactLocalization | null>(null);

  useEffect(() => {
    initLocalization().then(setL10n);
  }, []);

  // Renders LocalizationProvider with null l10n — crashes
  return (
    <LocalizationProvider l10n={l10n!}>
      <Content />
    </LocalizationProvider>
  );
}

// CORRECT — show loading state until translations are ready
function App() {
  const [l10n, setL10n] = useState<ReactLocalization | null>(null);

  useEffect(() => {
    initLocalization().then(setL10n);
  }, []);

  if (!l10n) return <div>Loading translations...</div>;

  return (
    <LocalizationProvider l10n={l10n}>
      <Content />
    </LocalizationProvider>
  );
}
```

**Why:** `LocalizationProvider` requires a valid `ReactLocalization` instance. Passing `null` or `undefined` causes a runtime crash. ALWAYS gate rendering behind a loading check when translations are fetched asynchronously.

---

## AP-11: Ignoring addResource Errors

```typescript
// WRONG — silently swallowing parse errors
const resource = new FluentResource(userProvidedFtl);
bundle.addResource(resource);
// Broken FTL silently fails → missing translations at runtime

// CORRECT — check and log errors
const resource = new FluentResource(userProvidedFtl);
const errors = bundle.addResource(resource);
if (errors.length > 0) {
  errors.forEach((e) => console.error("FTL parse error:", e.message));
}
```

**Why:** `addResource()` returns an array of `Error` objects for any messages with syntax errors. These messages are silently skipped. Without error checking, broken FTL results in missing translations with no diagnostic information.

---

## AP-12: Version Mismatch Between Bundle and React

```json
// WRONG — @fluent/bundle too old for @fluent/react
{
  "@fluent/bundle": "^0.15.0",
  "@fluent/react": "^0.15.2"
}
// @fluent/react requires @fluent/bundle >=0.16.0

// CORRECT — compatible versions
{
  "@fluent/bundle": "^0.19.1",
  "@fluent/react": "^0.15.2"
}
```

**Why:** `@fluent/react` declares `@fluent/bundle >=0.16.0` as a peer dependency. Version 0.16.0 renamed `FluentArgument` to `FluentVariable` and changed the ES target to ES2018. Using an older bundle version causes type mismatches and potential runtime errors.
