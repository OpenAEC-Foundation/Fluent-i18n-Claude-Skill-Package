# examples.md -- Complete Scaffolded Project Output

## Example 1: Full React Project Scaffold

### Project Structure After Scaffolding

```
my-react-app/
├── public/
│   └── locales/
│       ├── en-US/
│       │   └── main.ftl
│       ├── fr/
│       │   └── main.ftl
│       └── de/
│           └── main.ftl
├── src/
│   ├── l10n/
│   │   ├── index.ts
│   │   ├── bundles.ts
│   │   └── switch.ts
│   ├── components/
│   │   ├── LocalizedApp.tsx
│   │   └── LocaleSwitcher.tsx
│   └── App.tsx
├── package.json
└── tsconfig.json
```

### package.json (relevant section)

```json
{
  "dependencies": {
    "@fluent/bundle": "^0.18.0",
    "@fluent/langneg": "^0.7.0",
    "@fluent/react": "^0.15.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@fluent/syntax": "^0.19.0"
  }
}
```

### public/locales/en-US/main.ftl

```ftl
# Application
app-title = My Application

# Navigation
nav-home = Home
nav-settings = Settings
nav-about = About

# Common actions
action-save = Save
action-cancel = Cancel
action-delete = Delete
action-confirm = Confirm

# Welcome
welcome-user = Welcome, { $userName }!

# Plurals
items-count = { $count ->
    [one] You have { $count } item.
   *[other] You have { $count } items.
}

# Status
loading = Loading...
error-generic = Something went wrong. Please try again.
error-not-found = Page not found.

# Locale switcher
locale-switcher-label = Language
```

### public/locales/fr/main.ftl

```ftl
# Application
app-title = Mon Application

# Navigation
nav-home = Accueil
nav-settings = Paramètres
nav-about = À propos

# Common actions
action-save = Enregistrer
action-cancel = Annuler
action-delete = Supprimer
action-confirm = Confirmer

# Welcome
welcome-user = Bienvenue, { $userName } !

# Plurals
items-count = { $count ->
    [one] Vous avez { $count } élément.
   *[other] Vous avez { $count } éléments.
}

# Status
loading = Chargement...
error-generic = Un problème est survenu. Veuillez réessayer.
error-not-found = Page introuvable.

# Locale switcher
locale-switcher-label = Langue
```

### public/locales/de/main.ftl

```ftl
# Application
app-title = Meine Anwendung

# Navigation
nav-home = Startseite
nav-settings = Einstellungen
nav-about = Über uns

# Common actions
action-save = Speichern
action-cancel = Abbrechen
action-delete = Löschen
action-confirm = Bestätigen

# Welcome
welcome-user = Willkommen, { $userName }!

# Plurals
items-count = { $count ->
    [one] Sie haben { $count } Element.
   *[other] Sie haben { $count } Elemente.
}

# Status
loading = Wird geladen...
error-generic = Etwas ist schiefgelaufen. Bitte versuchen Sie es erneut.
error-not-found = Seite nicht gefunden.

# Locale switcher
locale-switcher-label = Sprache
```

### src/l10n/index.ts

```typescript
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";
import { createBundles } from "./bundles";

export const AVAILABLE_LOCALES = ["en-US", "fr", "de"];
export const DEFAULT_LOCALE = "en-US";
export const FTL_FILES = ["main.ftl"];

export async function initLocalization(): Promise<ReactLocalization> {
  const negotiated = negotiateLanguages(
    navigator.languages,
    AVAILABLE_LOCALES,
    { defaultLocale: DEFAULT_LOCALE }
  );

  const bundles = await createBundles(negotiated, FTL_FILES);
  return new ReactLocalization(bundles);
}
```

### src/l10n/bundles.ts

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

async function fetchMessages(locale: string, file: string): Promise<string> {
  const url = `/locales/${locale}/${file}`;
  const response = await fetch(url);
  if (!response.ok) {
    throw new Error(`Failed to fetch ${url}: ${response.status}`);
  }
  return response.text();
}

function createBundle(locale: string, messages: string[]): FluentBundle {
  const bundle = new FluentBundle(locale);
  for (const ftl of messages) {
    const resource = new FluentResource(ftl);
    const errors = bundle.addResource(resource);
    if (errors.length) {
      errors.forEach((e) =>
        console.warn(`FTL parse error in ${locale}:`, e)
      );
    }
  }
  return bundle;
}

export async function createBundles(
  locales: string[],
  files: string[]
): Promise<FluentBundle[]> {
  return Promise.all(
    locales.map(async (locale) => {
      const messages = await Promise.all(
        files.map((file) => fetchMessages(locale, file))
      );
      return createBundle(locale, messages);
    })
  );
}
```

### src/l10n/switch.ts

```typescript
import { negotiateLanguages } from "@fluent/langneg";
import { ReactLocalization } from "@fluent/react";
import { AVAILABLE_LOCALES, DEFAULT_LOCALE, FTL_FILES } from "./index";
import { createBundles } from "./bundles";

const LOCALE_STORAGE_KEY = "preferred-locale";

export function getSavedLocale(): string | null {
  try {
    return localStorage.getItem(LOCALE_STORAGE_KEY);
  } catch {
    return null;
  }
}

export function saveLocalePreference(locale: string): void {
  try {
    localStorage.setItem(LOCALE_STORAGE_KEY, locale);
  } catch {
    // Silently fail if localStorage is unavailable
  }
}

export async function switchLocale(
  locale: string
): Promise<ReactLocalization> {
  saveLocalePreference(locale);
  const negotiated = negotiateLanguages(
    [locale],
    AVAILABLE_LOCALES,
    { defaultLocale: DEFAULT_LOCALE }
  );
  const bundles = await createBundles(negotiated, FTL_FILES);
  return new ReactLocalization(bundles);
}

export function getInitialLocales(): string[] {
  const saved = getSavedLocale();
  const requested = saved ? [saved] : Array.from(navigator.languages);
  return negotiateLanguages(requested, AVAILABLE_LOCALES, {
    defaultLocale: DEFAULT_LOCALE,
  });
}
```

### src/components/LocalizedApp.tsx

```tsx
import React, { useState, useEffect, useCallback } from "react";
import { LocalizationProvider } from "@fluent/react";
import { ReactLocalization } from "@fluent/react";
import { initLocalization } from "../l10n";
import { switchLocale, getSavedLocale } from "../l10n/switch";

interface LocalizedAppProps {
  children: React.ReactNode;
}

export function LocalizedApp({ children }: LocalizedAppProps) {
  const [l10n, setL10n] = useState<ReactLocalization | null>(null);

  useEffect(() => {
    const saved = getSavedLocale();
    if (saved) {
      switchLocale(saved).then(setL10n);
    } else {
      initLocalization().then(setL10n);
    }
  }, []);

  const handleLocaleSwitch = useCallback(async (locale: string) => {
    const newL10n = await switchLocale(locale);
    setL10n(newL10n);
  }, []);

  if (!l10n) {
    return <div>Loading...</div>;
  }

  return (
    <LocalizationProvider l10n={l10n}>
      {children}
    </LocalizationProvider>
  );
}
```

### src/components/LocaleSwitcher.tsx

```tsx
import React from "react";
import { Localized, useLocalization } from "@fluent/react";
import { AVAILABLE_LOCALES } from "../l10n";

const LOCALE_LABELS: Record<string, string> = {
  "en-US": "English",
  "fr": "Français",
  "de": "Deutsch",
};

interface LocaleSwitcherProps {
  onSwitch: (locale: string) => void;
}

export function LocaleSwitcher({ onSwitch }: LocaleSwitcherProps) {
  const { l10n } = useLocalization();

  return (
    <div>
      <Localized id="locale-switcher-label">
        <label>Language</label>
      </Localized>
      <select
        onChange={(e) => onSwitch(e.target.value)}
        defaultValue={AVAILABLE_LOCALES[0]}
      >
        {AVAILABLE_LOCALES.map((locale) => (
          <option key={locale} value={locale}>
            {LOCALE_LABELS[locale] || locale}
          </option>
        ))}
      </select>
    </div>
  );
}
```

### src/App.tsx

```tsx
import React from "react";
import { Localized } from "@fluent/react";
import { LocalizedApp } from "./components/LocalizedApp";

function App() {
  return (
    <LocalizedApp>
      <Localized id="app-title">
        <h1>My Application</h1>
      </Localized>
      <Localized id="welcome-user" vars={{ userName: "World" }}>
        <p>Welcome, World!</p>
      </Localized>
    </LocalizedApp>
  );
}

export default App;
```

---

## Example 2: Vanilla JS Scaffold

### Project Structure

```
my-vanilla-app/
├── public/
│   └── locales/
│       ├── en-US/
│       │   └── main.ftl
│       └── fr/
│           └── main.ftl
├── src/
│   └── l10n/
│       └── index.ts
└── package.json
```

### package.json (relevant section)

```json
{
  "dependencies": {
    "@fluent/bundle": "^0.18.0",
    "@fluent/langneg": "^0.7.0"
  }
}
```

### src/l10n/index.ts

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { negotiateLanguages } from "@fluent/langneg";

const AVAILABLE_LOCALES = ["en-US", "fr"];
const DEFAULT_LOCALE = "en-US";

let currentBundle: FluentBundle | null = null;

async function fetchMessages(locale: string): Promise<string> {
  const response = await fetch(`/locales/${locale}/main.ftl`);
  if (!response.ok) {
    throw new Error(`Failed to fetch FTL for ${locale}: ${response.status}`);
  }
  return response.text();
}

export async function initLocalization(): Promise<FluentBundle> {
  const negotiated = negotiateLanguages(
    navigator.languages,
    AVAILABLE_LOCALES,
    { defaultLocale: DEFAULT_LOCALE }
  );

  const locale = negotiated[0];
  const messages = await fetchMessages(locale);
  const bundle = new FluentBundle(locale);
  const errors = bundle.addResource(new FluentResource(messages));
  if (errors.length) {
    errors.forEach((e) => console.warn(`FTL error:`, e));
  }
  currentBundle = bundle;
  return bundle;
}

export function t(
  id: string,
  args?: Record<string, string | number>
): string {
  if (!currentBundle) return id;
  const msg = currentBundle.getMessage(id);
  if (!msg?.value) return id;
  const errors: Error[] = [];
  const result = currentBundle.formatPattern(msg.value, args, errors);
  if (errors.length) {
    errors.forEach((e) => console.warn(`Format error for ${id}:`, e));
  }
  return result;
}

export function getCurrentBundle(): FluentBundle | null {
  return currentBundle;
}
```

### Usage in vanilla JS

```typescript
import { initLocalization, t } from "./l10n";

async function main() {
  await initLocalization();

  document.querySelector("h1")!.textContent = t("app-title");
  document.querySelector("#welcome")!.textContent = t("welcome-user", {
    userName: "World",
  });
}

main();
```

---

## Example 3: Node.js Server Scaffold

### Project Structure

```
my-server-app/
├── locales/
│   ├── en-US/
│   │   └── main.ftl
│   └── fr/
│       └── main.ftl
├── src/
│   ├── l10n/
│   │   └── server.ts
│   └── server.ts
└── package.json
```

### src/l10n/server.ts

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";
import { negotiateLanguages, acceptedLanguages } from "@fluent/langneg";
import { readFileSync } from "fs";
import { join } from "path";

const AVAILABLE_LOCALES = ["en-US", "fr"];
const DEFAULT_LOCALE = "en-US";
const LOCALES_DIR = join(__dirname, "../../locales");

const bundleCache = new Map<string, FluentBundle>();

function loadBundle(locale: string): FluentBundle {
  if (bundleCache.has(locale)) {
    return bundleCache.get(locale)!;
  }

  const ftlPath = join(LOCALES_DIR, locale, "main.ftl");
  const messages = readFileSync(ftlPath, "utf-8");
  const bundle = new FluentBundle(locale);
  const errors = bundle.addResource(new FluentResource(messages));
  if (errors.length) {
    errors.forEach((e) => console.warn(`FTL error in ${locale}:`, e));
  }
  bundleCache.set(locale, bundle);
  return bundle;
}

export function getBundle(acceptLanguageHeader: string): FluentBundle {
  const requested = acceptedLanguages(acceptLanguageHeader || "");
  const negotiated = negotiateLanguages(requested, AVAILABLE_LOCALES, {
    defaultLocale: DEFAULT_LOCALE,
  });
  return loadBundle(negotiated[0]);
}

export function formatMessage(
  bundle: FluentBundle,
  id: string,
  args?: Record<string, string | number>
): string {
  const msg = bundle.getMessage(id);
  if (!msg?.value) return id;
  const errors: Error[] = [];
  const result = bundle.formatPattern(msg.value, args, errors);
  if (errors.length) {
    errors.forEach((e) => console.warn(`Format error for ${id}:`, e));
  }
  return result;
}
```

### Usage in Express

```typescript
import express from "express";
import { getBundle, formatMessage } from "./l10n/server";

const app = express();

app.get("/", (req, res) => {
  const bundle = getBundle(req.headers["accept-language"] || "");
  const title = formatMessage(bundle, "app-title");
  const welcome = formatMessage(bundle, "welcome-user", { userName: "World" });
  res.send(`<h1>${title}</h1><p>${welcome}</p>`);
});

app.listen(3000);
```

---

## Example 4: Adding a New Locale to Existing Scaffold

When adding a new locale (e.g., `es`) to an existing scaffold:

### Step 1: Create FTL directory and file

```
public/locales/es/main.ftl
```

### Step 2: Copy en-US/main.ftl as starting point and translate

```ftl
# public/locales/es/main.ftl
app-title = Mi Aplicación
nav-home = Inicio
nav-settings = Configuración
welcome-user = Bienvenido, { $userName }!
items-count = { $count ->
    [one] Tienes { $count } elemento.
   *[other] Tienes { $count } elementos.
}
loading = Cargando...
error-generic = Algo salió mal. Inténtalo de nuevo.
```

### Step 3: Update AVAILABLE_LOCALES

```typescript
// src/l10n/index.ts
export const AVAILABLE_LOCALES = ["en-US", "fr", "de", "es"];
```

### Step 4: Update locale labels (if using LocaleSwitcher)

```typescript
const LOCALE_LABELS: Record<string, string> = {
  "en-US": "English",
  "fr": "Français",
  "de": "Deutsch",
  "es": "Español",
};
```

**ALWAYS** ensure the new locale directory has ALL the same FTL files as en-US.

**ALWAYS** update `AVAILABLE_LOCALES` -- forgetting this means the new locale is never matched by `negotiateLanguages`.
