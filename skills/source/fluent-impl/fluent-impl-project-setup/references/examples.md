# examples.md — Working Setup Examples

## Example 1: Minimal Working Setup

The smallest possible Fluent + React setup with one locale and one FTL file.

### FTL File

```ftl
# public/locales/en-US/main.ftl
app-title = Hello Fluent
greeting = Welcome to Project Fluent!
```

### Localization Module

```typescript
// src/l10n/index.ts
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { ReactLocalization } from "@fluent/react";

async function fetchMessages(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/main.ftl`);
  return response.text();
}

export async function initLocalization(): Promise<ReactLocalization> {
  const messages = await fetchMessages("en-US");
  const bundle = new FluentBundle("en-US");
  const errors = bundle.addResource(new FluentResource(messages));
  if (errors.length) {
    errors.forEach((e) => console.warn("FTL parse error:", e));
  }
  return new ReactLocalization([bundle]);
}
```

### React Entry Point

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

  if (!l10n) return <div>Loading...</div>;

  return (
    <LocalizationProvider l10n={l10n}>
      <Localized id="app-title">
        <h1>Hello Fluent</h1>
      </Localized>
      <Localized id="greeting">
        <p>Welcome to Project Fluent!</p>
      </Localized>
    </LocalizationProvider>
  );
}

export default App;
```

---

## Example 2: Complete Multi-Locale Scaffold

A production-ready setup with locale negotiation, multiple locales, and multiple FTL files.

### Directory Structure

```
my-app/
├── public/
│   └── locales/
│       ├── en-US/
│       │   ├── main.ftl
│       │   ├── errors.ftl
│       │   └── settings.ftl
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
│   │   ├── index.ts
│   │   └── bundles.ts
│   ├── components/
│   │   └── LocalizedApp.tsx
│   └── App.tsx
├── scripts/
│   └── validate-ftl.ts
└── package.json
```

### FTL Files

```ftl
# public/locales/en-US/main.ftl
app-title = My Application
nav-home = Home
nav-settings = Settings
nav-logout = Log Out
welcome-user = Welcome, { $userName }!
items-count = { $count ->
    [one] You have { $count } item.
   *[other] You have { $count } items.
}
```

```ftl
# public/locales/en-US/errors.ftl
error-not-found = Page not found.
error-network = Unable to connect. Please check your internet connection.
error-auth-expired = Your session has expired. Please log in again.
error-validation-required = This field is required.
error-validation-email = Please enter a valid email address.
```

```ftl
# public/locales/fr/main.ftl
app-title = Mon Application
nav-home = Accueil
nav-settings = Paramètres
nav-logout = Déconnexion
welcome-user = Bienvenue, { $userName } !
items-count = { $count ->
    [one] Vous avez { $count } élément.
   *[other] Vous avez { $count } éléments.
}
```

### Bundle Generation Module

```typescript
// src/l10n/bundles.ts
import { FluentBundle, FluentResource } from "@fluent/bundle";

const FTL_FILES = ["main.ftl", "errors.ftl", "settings.ftl"];

async function fetchMessages(locale: string, file: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/${file}`);
  if (!response.ok) {
    console.warn(`Missing FTL file: /locales/${locale}/${file}`);
    return "";
  }
  return response.text();
}

export async function createBundle(locale: string): Promise<FluentBundle> {
  const bundle = new FluentBundle(locale);

  for (const file of FTL_FILES) {
    const messages = await fetchMessages(locale, file);
    if (messages) {
      const errors = bundle.addResource(new FluentResource(messages));
      if (errors.length) {
        errors.forEach((e) =>
          console.warn(`FTL error in ${locale}/${file}:`, e)
        );
      }
    }
  }

  return bundle;
}
```

### Localization Init Module

```typescript
// src/l10n/index.ts
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";
import { createBundle } from "./bundles";

export const AVAILABLE_LOCALES = ["en-US", "fr", "de"];
export const DEFAULT_LOCALE = "en-US";

export async function initLocalization(): Promise<ReactLocalization> {
  const negotiated = negotiateLanguages(
    navigator.languages,
    AVAILABLE_LOCALES,
    { defaultLocale: DEFAULT_LOCALE }
  );

  const bundles = await Promise.all(negotiated.map(createBundle));
  return new ReactLocalization(bundles);
}

export async function switchLocale(
  locale: string
): Promise<ReactLocalization> {
  const negotiated = negotiateLanguages(
    [locale],
    AVAILABLE_LOCALES,
    { defaultLocale: DEFAULT_LOCALE }
  );

  const bundles = await Promise.all(negotiated.map(createBundle));
  return new ReactLocalization(bundles);
}
```

### LocalizationProvider Wrapper

```tsx
// src/components/LocalizedApp.tsx
import React, { useState, useEffect, useCallback } from "react";
import { LocalizationProvider } from "@fluent/react";
import { ReactLocalization } from "@fluent/react";
import { initLocalization, switchLocale } from "../l10n";

interface Props {
  children: React.ReactNode;
}

export function LocalizedApp({ children }: Props) {
  const [l10n, setL10n] = useState<ReactLocalization | null>(null);

  useEffect(() => {
    initLocalization().then(setL10n);
  }, []);

  const handleLocaleSwitch = useCallback(async (locale: string) => {
    const newL10n = await switchLocale(locale);
    setL10n(newL10n);
  }, []);

  if (!l10n) return <div>Loading translations...</div>;

  return (
    <LocalizationProvider l10n={l10n}>
      {children}
    </LocalizationProvider>
  );
}
```

### Root App Component

```tsx
// src/App.tsx
import React from "react";
import { Localized, useLocalization } from "@fluent/react";
import { LocalizedApp } from "./components/LocalizedApp";

function MainContent() {
  const { l10n } = useLocalization();

  const handleDelete = () => {
    const message = l10n.getString("confirm-delete", null, "Are you sure?");
    if (window.confirm(message)) {
      // proceed with deletion
    }
  };

  return (
    <div>
      <Localized id="app-title">
        <h1>My Application</h1>
      </Localized>
      <Localized id="welcome-user" vars={{ userName: "Anna" }}>
        <p>Welcome, Anna!</p>
      </Localized>
      <button onClick={handleDelete}>Delete</button>
    </div>
  );
}

function App() {
  return (
    <LocalizedApp>
      <MainContent />
    </LocalizedApp>
  );
}

export default App;
```

---

## Example 3: FTL Validation Script

Use `@fluent/syntax` to validate all FTL files in CI:

```typescript
// scripts/validate-ftl.ts
import { FluentParser } from "@fluent/syntax";
import { readFileSync, readdirSync, statSync } from "fs";
import { join, extname } from "path";

const parser = new FluentParser({ withSpans: true });
let hasErrors = false;

function validateFile(filePath: string): void {
  const source = readFileSync(filePath, "utf-8");
  const resource = parser.parse(source);

  for (const entry of resource.body) {
    if (entry.type === "Junk") {
      hasErrors = true;
      for (const annotation of entry.annotations) {
        console.error(
          `ERROR in ${filePath}: ${annotation.message} (code: ${annotation.code})`
        );
      }
    }
  }
}

function walkDir(dir: string): void {
  for (const item of readdirSync(dir)) {
    const fullPath = join(dir, item);
    if (statSync(fullPath).isDirectory()) {
      walkDir(fullPath);
    } else if (extname(fullPath) === ".ftl") {
      validateFile(fullPath);
    }
  }
}

walkDir("public/locales");

if (hasErrors) {
  console.error("FTL validation failed.");
  process.exit(1);
} else {
  console.log("All FTL files are valid.");
}
```

Run with:
```bash
npx tsx scripts/validate-ftl.ts
```

---

## Example 4: package.json Dependencies

```json
{
  "dependencies": {
    "@fluent/bundle": "^0.19.1",
    "@fluent/langneg": "^0.7.0",
    "@fluent/react": "^0.15.2",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@fluent/syntax": "^0.19.0",
    "typescript": "^5.0.0"
  }
}
```
