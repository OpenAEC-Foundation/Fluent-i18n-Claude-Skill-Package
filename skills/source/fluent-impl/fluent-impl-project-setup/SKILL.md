---
name: fluent-impl-project-setup
description: "Guides complete Fluent project setup including npm dependency installation, directory structure for FTL files and TypeScript code, minimal working FTL+TypeScript+React configuration, semantic message ID conventions, @fluent/syntax for FTL validation tooling, and file naming best practices. Activates when starting a new Fluent project, adding Fluent to an existing app, or scaffolding the localization infrastructure."
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/bundle 0.18+, @fluent/react 0.15+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-impl-project-setup

## Quick Reference

### Required Packages

| Package | Purpose | Version |
|---------|---------|---------|
| `@fluent/bundle` | Core runtime: parse FTL, format messages | 0.18+ (current: 0.19.1) |
| `@fluent/react` | React bindings: `LocalizationProvider`, `Localized`, `useLocalization` | 0.15+ (current: 0.15.2) |
| `@fluent/langneg` | Locale negotiation: match user preferences to available locales | 0.7.0 |

### Optional Packages

| Package | Purpose | When to Install |
|---------|---------|-----------------|
| `@fluent/syntax` | Full AST parser/serializer for FTL validation and tooling | CI linting, editor tooling, programmatic FTL generation |

### Peer Dependencies

| Package | Required By | Minimum Version |
|---------|-------------|-----------------|
| `react` | `@fluent/react` | 16.8.0 (hooks support) |
| `@fluent/bundle` | `@fluent/react` | 0.16.0 |

### Critical Warnings

**NEVER** use `@fluent/syntax` for runtime message formatting -- it is a tooling library. ALWAYS use `@fluent/bundle` for runtime formatting.

**NEVER** install the legacy `fluent` package -- it was renamed to `@fluent/bundle` in v0.13.0. The old package is unmaintained.

**NEVER** skip installing `@fluent/langneg` -- without locale negotiation, your app cannot match user preferences to available translations.

**ALWAYS** install all three core packages together -- they form an inseparable stack for any React-based Fluent project.

**ALWAYS** check `addResource()` return value for parse errors -- silent failures cause missing translations at runtime.

---

## Setup Workflow

### Step 1: Install Dependencies

```bash
# Core stack (ALWAYS install all three)
npm install @fluent/bundle @fluent/react @fluent/langneg

# Optional: FTL validation tooling (for CI/linting scripts)
npm install --save-dev @fluent/syntax
```

### Step 2: Create Directory Structure

```
my-app/
├── public/
│   └── locales/                    # FTL files served statically
│       ├── en-US/
│       │   ├── main.ftl            # Core UI messages
│       │   ├── errors.ftl          # Error messages
│       │   └── settings.ftl        # Settings page messages
│       ├── fr/
│       │   ├── main.ftl
│       │   ├── errors.ftl
│       │   └── settings.ftl
│       └── de/
│           ├── main.ftl
│           ├── errors.ftl
│           └── settings.ftl
├── src/
│   ├── l10n/
│   │   ├── index.ts                # initLocalization(), locale constants
│   │   └── bundles.ts              # Bundle generation logic
│   ├── components/
│   │   └── LocalizedApp.tsx        # LocalizationProvider wrapper
│   └── App.tsx
└── package.json
```

**ALWAYS** place FTL files under `public/locales/{locale}/` for static serving via `fetch()`.

**ALWAYS** create a dedicated `src/l10n/` directory for localization setup code.

**ALWAYS** organize FTL files by feature, NOT by component -- one FTL file per feature area per locale.

### Step 3: Create FTL Files

```ftl
# public/locales/en-US/main.ftl
app-title = My Application
nav-home = Home
nav-settings = Settings
welcome-user = Welcome, { $userName }!
items-count = { $count ->
    [one] You have { $count } item.
   *[other] You have { $count } items.
}
```

### Step 4: Create Localization Module

```typescript
// src/l10n/index.ts
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";

export const AVAILABLE_LOCALES = ["en-US", "fr", "de"];
export const DEFAULT_LOCALE = "en-US";

async function fetchMessages(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/main.ftl`);
  return response.text();
}

function createBundle(locale: string, messages: string): FluentBundle {
  const bundle = new FluentBundle(locale);
  const resource = new FluentResource(messages);
  const errors = bundle.addResource(resource);
  if (errors.length) {
    errors.forEach((e) => console.warn(`FTL error in ${locale}:`, e));
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

### Step 5: Wire Up React

```tsx
// src/App.tsx
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
    return <div>Loading translations...</div>;
  }

  return (
    <LocalizationProvider l10n={l10n}>
      <Localized id="app-title">
        <h1>My Application</h1>
      </Localized>
    </LocalizationProvider>
  );
}

export default App;
```

---

## Decision Trees

### Which Packages Do I Need?

```
Is this a React project?
├── YES → Install @fluent/bundle + @fluent/react + @fluent/langneg
│         Do you need FTL linting or validation in CI?
│         ├── YES → Also install @fluent/syntax (devDependency)
│         └── NO  → Core three packages are sufficient
└── NO  → Install @fluent/bundle + @fluent/langneg only
          (React bindings are not needed for vanilla JS/Node.js)
```

### How Should I Organize FTL Files?

```
How many messages does the app have?
├── < 50 messages → Single file per locale: locales/en-US.ftl
├── 50-200 messages → Feature-based: locales/en-US/{feature}.ftl
└── > 200 messages → Feature-based + code-splitting per route
```

### FTL File Naming Convention

| File Name | Content |
|-----------|---------|
| `main.ftl` | Core UI: navigation, headers, common buttons |
| `errors.ftl` | Error messages, validation feedback |
| `settings.ftl` | Settings/preferences page |
| `auth.ftl` | Login, registration, password reset |
| `dashboard.ftl` | Dashboard-specific messages |

---

## Semantic Message ID Conventions

ALWAYS use the `component-element-action` pattern for message IDs:

| Pattern | Example | When to Use |
|---------|---------|-------------|
| `{page}-{element}` | `login-title` | Static text elements |
| `{page}-{element}-{action}` | `login-button-submit` | Interactive elements |
| `{component}-{descriptor}` | `nav-home` | Reusable components |
| `{feature}-{state}` | `upload-progress` | State descriptions |
| `{feature}-error-{type}` | `auth-error-invalid-email` | Error messages |

**ALWAYS** use kebab-case for message IDs: `welcome-user`, NOT `welcomeUser` or `welcome_user`.

**NEVER** use generic IDs like `button1`, `text3`, or `message` -- they are impossible to maintain.

**ALWAYS** change the message ID when the meaning changes -- this signals translators that re-translation is needed.

---

## FTL Validation with @fluent/syntax

Use `@fluent/syntax` in CI pipelines or development scripts to validate FTL files:

```typescript
// scripts/validate-ftl.ts
import { FluentParser } from "@fluent/syntax";
import { readFileSync, readdirSync } from "fs";
import { join } from "path";

const parser = new FluentParser({ withSpans: true });

function validateFtlFile(filePath: string): boolean {
  const source = readFileSync(filePath, "utf-8");
  const resource = parser.parse(source);
  let valid = true;

  for (const entry of resource.body) {
    if (entry.type === "Junk") {
      console.error(`Parse error in ${filePath}:`, entry.annotations);
      valid = false;
    }
  }
  return valid;
}
```

**NEVER** import `@fluent/syntax` in application runtime code -- it adds unnecessary bundle weight. Use it ONLY in build scripts, CI, and development tooling.

---

## Reference Links

- [references/methods.md](references/methods.md) -- npm packages, versions, peer dependencies, API signatures
- [references/examples.md](references/examples.md) -- Minimal working setup, complete scaffold, directory structures
- [references/anti-patterns.md](references/anti-patterns.md) -- Wrong versions, missing peer deps, wrong structure

### Official Sources

- https://github.com/projectfluent/fluent.js/wiki/React-Tutorial
- https://github.com/projectfluent/fluent.js/tree/main/fluent-bundle
- https://github.com/projectfluent/fluent.js/tree/main/fluent-react
- https://github.com/projectfluent/fluent.js/tree/main/fluent-langneg
- https://github.com/projectfluent/fluent.js/tree/main/fluent-syntax
