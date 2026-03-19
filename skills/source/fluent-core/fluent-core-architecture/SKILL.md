---
name: fluent-core-architecture
description: >
  Use when setting up or understanding Project Fluent architecture. Prevents misconceptions about the bundle-resource-message model and asymmetric localization.
  Covers design philosophy, FTL format overview, and how Fluent differs from ICU MessageFormat.
  Keywords: Project Fluent, FTL, FluentBundle, FluentResource, asymmetric localization, i18n architecture.
license: MIT
compatibility: "Designed for Claude Code. Requires @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-core-architecture

## Quick Reference

### Fluent Package Ecosystem

| Package | Version | Purpose |
|---------|---------|---------|
| `@fluent/bundle` | 0.19.1 | Core runtime: parse FTL, format messages |
| `@fluent/react` | 0.15.2 | React bindings: `<Localized>`, `useLocalization()` |
| `@fluent/langneg` | 0.7.0 | Locale negotiation between user prefs and available locales |
| `@fluent/syntax` | 0.19.0 | Full AST parser/serializer for tooling (linting, codegen) |

### Runtime Model (data flow)

```
FTL string ──► FluentResource ──► FluentBundle ──► getMessage(id) ──► formatPattern(pattern, args) ──► string
                  (parse)           (store)          (lookup)              (resolve + format)
```

### Key Terminology

| Term | Definition |
|------|-----------|
| **Message** | A named unit of translation: identifier + value (pattern) + optional attributes |
| **Term** | A reusable fragment prefixed with `-`; NEVER accessible via `getMessage()` |
| **Pattern** | The translatable content of a message or attribute |
| **Placeable** | `{ }` syntax that embeds variables, references, or function calls into text |
| **Variant** | One branch of a select expression, keyed by string or plural category |
| **Attribute** | A `.name = value` sub-entry on a message or term |
| **Selector** | A `->` expression that chooses a variant based on a runtime value |
| **Resource** | A parsed FTL file containing messages and terms |
| **Bundle** | A locale-bound container that holds resources and formats messages |

### Critical Warnings

**NEVER** confuse Fluent with ICU MessageFormat. They are fundamentally different systems. Fluent uses asymmetric localization (each locale develops independently); ICU uses symmetric patterns (source language grammar leaks to all translations). See the comparison table below.

**NEVER** use `@fluent/syntax` for runtime message formatting. It is a tooling library for linting, AST manipulation, and codegen. ALWAYS use `@fluent/bundle` for runtime formatting.

**NEVER** skip the two-step lookup: ALWAYS call `getMessage(id)` first, then `formatPattern(pattern, args)`. There is no single-call format method.

**NEVER** pass raw FTL strings to `addResource()`. It requires a `FluentResource` instance: `bundle.addResource(new FluentResource(ftl))`.

**ALWAYS** pass an `errors` array to `formatPattern()` in production. Without it, the method throws on the first resolution error instead of returning a best-effort string.

---

## Design Philosophy

Project Fluent is Mozilla's localization system built on one core principle: **asymmetric localization**. Translators can use the full expressive power of their language without asking developers for permission.

### Asymmetric Localization

Traditional i18n systems (gettext, Java properties, ICU MessageFormat) enforce a one-to-one mapping between source strings and translations. The grammar of the source language limits what translators can express. Fluent eliminates this constraint:

- A simple English string can map to a complex multi-variant Polish translation with grammatical cases.
- The Polish translator adds selectors independently -- no code changes, no English file modifications.
- Locale-specific logic is isolated. Complexity in one locale NEVER leaks to another.

### Progressive Enhancement

Each locale develops independently. English may use `welcome = Welcome, { $name }!` while Polish uses a selector on grammatical gender. The developer writes one `getMessage("welcome")` call. Fluent handles the rest.

### Simplicity for Simple Cases

The basic syntax is as simple as any properties file:

```ftl
hello = Hello, world!
```

Complexity (selectors, terms, attributes) is added only when a locale needs it.

---

## FTL Format Overview

FTL (Fluent Translation List) is the file format for Fluent translations. Syntax version: **Fluent Syntax 1.0** (released April 2019).

### Messages

The basic unit of translation. An identifier, `=`, and a value (pattern):

```ftl
hello = Hello, world!
welcome = Welcome, { $name }!
```

### Terms

Reusable fragments prefixed with `-`. ALWAYS use terms for brand names and shared vocabulary:

```ftl
-brand-name = Firefox
about = About { -brand-name }.
```

Terms accept parameters and can store grammatical metadata as attributes. Terms are NEVER accessible via the public `getMessage()` API.

### Attributes

Multiple translatable strings grouped under one message using `.attribute` syntax:

```ftl
login-input = Predefined value
    .placeholder = email@example.com
    .aria-label = Login input value
```

### Selectors

Choose translation variants based on runtime values. Every select expression MUST have exactly one default variant (marked with `*`):

```ftl
emails =
    { $count ->
        [one] You have one unread email.
       *[other] You have { $count } unread emails.
    }
```

### Comments

Three levels: `#` (message-bound), `##` (section group), `###` (file-level resource comment).

---

## The Runtime Model

### FluentBundle -> FluentResource -> Message -> Pattern -> formatPattern

The runtime model follows a strict pipeline:

**Step 1: Create a bundle** for a specific locale:

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";
const bundle = new FluentBundle("en-US");
```

**Step 2: Parse FTL into a resource** and add it to the bundle:

```typescript
const resource = new FluentResource(`
welcome = Welcome, { $name }!
-brand = Acme Corp
`);
const errors = bundle.addResource(resource);
```

**Step 3: Look up a message** by its identifier:

```typescript
const msg = bundle.getMessage("welcome");
```

The returned `Message` object has this shape:

```typescript
interface Message {
  value: Pattern | null;
  attributes: Record<string, Pattern>;
}
```

**Step 4: Format the pattern** with runtime arguments:

```typescript
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, { name: "Anna" });
  // -> "Welcome, Anna!"
}
```

### Key API Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `constructor` | `new FluentBundle(locales, options?)` | Create a locale-bound bundle |
| `addResource` | `(resource, options?) => Error[]` | Add parsed FTL; returns parse errors |
| `getMessage` | `(id) => Message \| undefined` | Look up a message by ID |
| `hasMessage` | `(id) => boolean` | Check if a message exists |
| `formatPattern` | `(pattern, args?, errors?) => string` | Format a pattern to a string |

### Constructor Options

| Option | Type | Default | Purpose |
|--------|------|---------|---------|
| `functions` | `Record<string, FluentFunction>` | `{}` | Custom functions available in FTL |
| `useIsolating` | `boolean` | `true` | Wrap placeables in Unicode bidi isolation marks |
| `transform` | `(text: string) => string` | identity | Transform applied to all text elements |

---

## Fluent vs ICU MessageFormat

| Aspect | ICU MessageFormat | Project Fluent |
|--------|-------------------|----------------|
| **Localization model** | Symmetric (1:1 mapping) | Asymmetric (1:N mapping) |
| **Complexity leakage** | Source language grammar leaks to all translations | Locale logic is isolated per language |
| **File format** | Embedded in code or properties files | Dedicated `.ftl` files |
| **Terms/brands** | No built-in concept | First-class `-term` syntax |
| **Attributes** | Not supported | `.attribute` syntax for multi-value messages |
| **Comments** | Not standardized | Three-level comment system (`#`, `##`, `###`) |
| **Error handling** | Typically throws errors | Graceful fallback (shows message ID or `{???}`) |
| **Plural syntax** | `{count, plural, one{...} other{...}}` | `{ $count -> [one] ... *[other] ... }` |
| **Gender syntax** | `{gender, select, male{...} female{...} other{...}}` | Uses term attributes with selectors |
| **Nesting** | Deep nesting common and hard to read | Flat structure, complexity in variants |
| **Translator autonomy** | Often requires developer coordination | Translators add complexity independently |

The most significant difference: in ICU, if Polish needs grammatical cases, the developer MUST modify source code to pass case parameters and update the English source string. In Fluent, the Polish translator adds a selector independently -- the English file stays simple and no code changes are needed.

---

## Decision Tree: Which Package Do I Need?

```
Need to format messages at runtime in an app?
├── YES -> @fluent/bundle (core runtime)
│   ├── Building a React app? -> ALSO add @fluent/react
│   └── Need locale negotiation? -> ALSO add @fluent/langneg
└── NO
    ├── Linting or validating FTL files? -> @fluent/syntax
    ├── Programmatically generating FTL? -> @fluent/syntax
    └── Building editor tooling? -> @fluent/syntax
```

**NEVER** use `@fluent/syntax` for runtime formatting. It produces a full AST and is significantly heavier than the optimized runtime parser in `@fluent/bundle`.

---

## Reference Links

- [references/methods.md](references/methods.md) -- Key types: FluentBundle, FluentResource, Message, Pattern
- [references/examples.md](references/examples.md) -- Full FTL + TypeScript flow examples
- [references/anti-patterns.md](references/anti-patterns.md) -- Common mistakes including ICU confusion

### Official Sources

- https://projectfluent.org -- Design philosophy, syntax guide
- https://github.com/projectfluent/fluent.js -- JavaScript implementation
- https://github.com/projectfluent/fluent -- FTL specification and EBNF grammar
- https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf -- Formal grammar
