---
name: fluent-syntax-attributes
description: "Guides FTL attributes including the .attribute syntax on messages and terms, value-less messages with only attributes, compound messages with value and attributes, accessing attributes via getMessage().attributes in TypeScript, and the Localized attrs prop in React. Activates when working with HTML element attributes, form field labels, or multi-value messages in FTL."
license: MIT
compatibility: "Designed for Claude Code. Requires Fluent Syntax 1.0, @fluent/bundle 0.18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# fluent-syntax-attributes

## Quick Reference

### Attribute Syntax Overview

| Concept | Syntax | Example |
|---------|--------|---------|
| Message attribute | `.attr = Value` (indented) | `.placeholder = Enter email` |
| Term attribute | `.attr = Value` on `-term` | `.gender = feminine` |
| Value-less message | `msg =` followed by attributes only | `login =\n    .aria-label = Login` |
| Compound message | Value + attributes | `login = Log in\n    .title = Click to log in` |
| Attribute reference (FTL) | `{ msg.attr }` or `{ -term.attr }` | `{ -brand.gender }` |
| TypeScript access | `getMessage(id).attributes["attr"]` | Returns `Pattern \| undefined` |
| React access | `<Localized attrs={{ title: true }}>` | Explicit opt-in per attribute |

### EBNF Grammar

```ebnf
Attribute         ::= line_end blank? "." Identifier blank_inline? "=" blank_inline? Pattern
AttributeAccessor ::= "." Identifier
Message           ::= Identifier blank_inline? "=" blank_inline? ((Pattern Attribute*) | (Attribute+))
Term              ::= "-" Identifier blank_inline? "=" blank_inline? Pattern Attribute*
```

### Critical Warnings

**ALWAYS** check `msg.value` for `null` before calling `formatPattern()` -- value-less messages (attributes-only) are valid FTL. Calling `formatPattern(null!)` causes a runtime error.

**ALWAYS** declare `attrs` explicitly in React's `<Localized>` component -- translations CANNOT inject arbitrary attributes onto DOM elements. Only attributes listed as `true` in the `attrs` prop are applied.

**NEVER** try to access term attributes at runtime via `getMessage()` -- term attributes are private and can ONLY be used as selectors inside FTL files. There is no public API to retrieve them.

**NEVER** assume a message has a value -- the grammar allows `(Attribute+)` without a `Pattern`. ALWAYS use the pattern `if (msg?.value) { ... }` before formatting.

---

## Attribute Syntax

### Message Attributes

Attributes are indented lines starting with a dot (`.`) that belong to the preceding message or term. They group multiple translatable strings under one identifier.

```ftl
login-input = Predefined value
    .placeholder = email@example.com
    .aria-label = Login input value
    .title = Type your login email
```

This creates one message (`login-input`) with a value and three attributes. Each attribute has its own `Pattern` that can contain placeables, variables, and references.

### Value-less Messages (Attributes Only)

A message MAY have ONLY attributes and no value. This is valid FTL:

```ftl
login-input =
    .placeholder = email@example.com
    .aria-label = Login input value
```

Here `login-input` has no value (`msg.value` is `null`), only attributes. Use this when a UI element has no visible text content but has accessible labels or HTML attributes that need translation.

### Compound Messages (Value + Attributes)

A message MAY have both a value and attributes:

```ftl
login-button = Log In
    .title = Click to log in to your account
    .aria-label = Log in button
```

The value (`Log In`) is the primary text. The attributes (`.title`, `.aria-label`) are secondary strings associated with the same UI element.

---

## Term Attributes

Terms can also have attributes, but term attributes are **private** -- they are NOT accessible to the localization runtime and can ONLY be used as selectors within FTL.

```ftl
-brand-name = Aurora
    .gender = feminine

update-successful =
    { -brand-name.gender ->
        [masculine] { -brand-name } has been updated.
        [feminine] { -brand-name } has been updated.
       *[other] { -brand-name } software has been updated.
    }
```

The `.gender` attribute on `-brand-name` stores grammatical metadata. It is referenced as a selector via `{ -brand-name.gender -> ... }` but CANNOT be retrieved by application code.

### Key Difference: Message vs. Term Attributes

| Aspect | Message Attributes | Term Attributes |
|--------|-------------------|-----------------|
| Runtime access | YES -- via `getMessage().attributes` | NO -- private, selector-only |
| Purpose | Translatable UI strings (placeholder, title, aria-label) | Grammatical metadata (gender, animacy, case) |
| Value requirement | Message can be attribute-only (no value) | Term MUST have a value |
| Use in selectors | YES -- via `{ msg.attr -> ... }` | YES -- via `{ -term.attr -> ... }` |

---

## TypeScript Access

### getMessage() Returns Attributes

The `getMessage()` method returns an object with `value` and `attributes`:

```typescript
interface Message {
  value: Pattern | null;
  attributes: Record<string, Pattern>;
}
```

### Accessing Attributes in TypeScript

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(`
login-input =
    .placeholder = email@example.com
    .aria-label = Login input value
    .title = Type your login email
`));

const msg = bundle.getMessage("login-input");
if (msg) {
  // Value is null for this attribute-only message
  if (msg.value) {
    const text = bundle.formatPattern(msg.value);
  }

  // Access attributes by name
  const placeholder = bundle.formatPattern(msg.attributes["placeholder"]);
  // -> "email@example.com"

  const ariaLabel = bundle.formatPattern(msg.attributes["aria-label"]);
  // -> "Login input value"
}
```

### Compound Message Access

```typescript
const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(`
submit-button = Submit Form
    .title = Click to submit
    .aria-label = Submit the registration form
`));

const msg = bundle.getMessage("submit-button");
if (msg) {
  // Format the value
  if (msg.value) {
    const buttonText = bundle.formatPattern(msg.value);
    // -> "Submit Form"
  }

  // Format an attribute
  const title = bundle.formatPattern(msg.attributes["title"]);
  // -> "Click to submit"
}
```

### Safe Access Pattern

ALWAYS use this pattern when accessing messages that may or may not have values:

```typescript
function getMessageValue(
  bundle: FluentBundle,
  id: string,
  args?: Record<string, FluentVariable>
): string | null {
  const msg = bundle.getMessage(id);
  if (!msg) return null;
  if (!msg.value) return null;
  const errors: Error[] = [];
  return bundle.formatPattern(msg.value, args, errors);
}

function getMessageAttribute(
  bundle: FluentBundle,
  id: string,
  attr: string,
  args?: Record<string, FluentVariable>
): string | null {
  const msg = bundle.getMessage(id);
  if (!msg) return null;
  const pattern = msg.attributes[attr];
  if (!pattern) return null;
  const errors: Error[] = [];
  return bundle.formatPattern(pattern, args, errors);
}
```

---

## React Access

### The attrs Prop

In `@fluent/react`, the `<Localized>` component uses the `attrs` prop to control which FTL attributes are applied to the wrapped element:

```tsx
import { Localized } from "@fluent/react";

<Localized id="login-input" attrs={{ placeholder: true, "aria-label": true }}>
  <input
    type="email"
    placeholder="email@example.com"
    aria-label="Login input"
  />
</Localized>
```

Corresponding FTL:

```ftl
login-input =
    .placeholder = email@example.com
    .aria-label = Login input value
```

### Compound Messages in React

When a message has both a value and attributes:

```tsx
<Localized id="submit-button" attrs={{ title: true }}>
  <button title="Click to submit">Submit Form</button>
</Localized>
```

```ftl
submit-button = Submit Form
    .title = Click to submit
```

The value replaces the button's text content. The `.title` attribute replaces the `title` prop -- but ONLY because `attrs={{ title: true }}` explicitly allows it.

### Imperative Attribute Access

Use the `useLocalization` hook for imperative access:

```tsx
import { useLocalization } from "@fluent/react";

function DynamicInput() {
  const { l10n } = useLocalization();

  // getString formats the message value, not attributes
  // For attribute access, use getBundle + getMessage directly
  const bundle = l10n.getBundle("login-input");
  if (bundle) {
    const msg = bundle.getMessage("login-input");
    if (msg) {
      const placeholder = bundle.formatPattern(
        msg.attributes["placeholder"]
      );
    }
  }

  return <input placeholder={placeholder} />;
}
```

---

## Decision Trees

### When to Use Attributes vs. Separate Messages

```
Is the string associated with the SAME UI element?
├── YES → Use attributes on one message
│   ├── Element has visible text + HTML attributes → Compound message
│   └── Element has ONLY HTML attributes (no text) → Value-less message
└── NO → Use separate messages
```

### When to Use Term Attributes

```
Is the metadata needed for grammatical agreement?
├── YES → Use term attribute (.gender, .animacy, .case)
│   └── Reference as selector: { -term.attr -> ... }
└── NO → Is it a translatable string?
    ├── YES → Use message attribute
    └── NO → Store as a comment instead
```

---

## Reference Links

- [references/methods.md](references/methods.md) -- EBNF grammar for Attribute and AttributeAccessor; getMessage().attributes API
- [references/examples.md](references/examples.md) -- Complete examples for message attributes, term attributes, value-less messages, compound messages, TypeScript and React access
- [references/anti-patterns.md](references/anti-patterns.md) -- Common mistakes with attributes: null value checks, missing attrs in React, term attribute access

### Official Sources

- https://projectfluent.org/fluent/guide/attributes.html
- https://raw.githubusercontent.com/projectfluent/fluent/master/spec/fluent.ebnf
- https://github.com/projectfluent/fluent.js/blob/main/fluent-bundle/src/bundle.ts
- https://github.com/projectfluent/fluent.js/wiki/Localized
