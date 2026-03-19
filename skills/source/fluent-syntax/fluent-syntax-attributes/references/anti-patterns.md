# Attributes -- Anti-Patterns

## Anti-Pattern 1: Not Checking for null Value

Value-less messages (attribute-only) return `null` for `msg.value`. Calling `formatPattern(null!)` causes a runtime error.

```typescript
// WRONG -- assumes every message has a value
const msg = bundle.getMessage("search-input")!;
const text = bundle.formatPattern(msg.value!); // Runtime error if value is null

// CORRECT -- always check for null value
const msg = bundle.getMessage("search-input");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value);
}
```

**Why this happens**: Developers assume every message has visible text. But messages like `search-input =\n    .placeholder = Search...` have ONLY attributes and `msg.value` is `null`.

---

## Anti-Pattern 2: Missing attrs Declaration in React

Translated attributes are ONLY applied when explicitly declared as `true` in the `attrs` prop. Forgetting `attrs` causes attributes to silently not translate.

```tsx
// WRONG -- attributes will NOT be translated
<Localized id="search-input">
  <input placeholder="Search..." aria-label="Search" />
</Localized>

// CORRECT -- explicitly opt in to each attribute
<Localized id="search-input" attrs={{ placeholder: true, "aria-label": true }}>
  <input placeholder="Search..." aria-label="Search" />
</Localized>
```

**Why this is enforced**: This is a security measure. Translations cannot inject arbitrary attributes (like `onclick`, `href`, or `style`) onto DOM elements. Every translated attribute MUST be explicitly allowed by the developer.

---

## Anti-Pattern 3: Trying to Access Term Attributes at Runtime

Term attributes are private -- they exist ONLY for use as selectors within FTL files. There is no public API to retrieve them.

```typescript
// WRONG -- terms are not accessible via getMessage()
const term = bundle.getMessage("-brand-name"); // returns undefined
// Even if it worked, term.attributes["gender"] is not exposed

// CORRECT -- term attributes are used as selectors in FTL only
// FTL:
// -brand-name = Aurora
//     .gender = feminine
//
// update-msg =
//     { -brand-name.gender ->
//         [feminine] { -brand-name } has been updated.
//        *[other] { -brand-name } software has been updated.
//     }
const msg = bundle.getMessage("update-msg");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value);
}
```

---

## Anti-Pattern 4: Using Non-Boolean Values in attrs

The `attrs` prop expects boolean values. Using truthy non-boolean values may work but is not the intended API.

```tsx
// WRONG -- string values instead of booleans
<Localized id="input" attrs={{ placeholder: "true", title: 1 }}>
  <input />
</Localized>

// CORRECT -- use boolean true
<Localized id="input" attrs={{ placeholder: true, title: true }}>
  <input />
</Localized>
```

---

## Anti-Pattern 5: Assuming All Attributes Exist

Not all locales may define all attributes. ALWAYS check that an attribute pattern exists before formatting.

```typescript
// WRONG -- crashes if "title" attribute is not defined in the FTL
const msg = bundle.getMessage("button")!;
const title = bundle.formatPattern(msg.attributes["title"]);

// CORRECT -- check attribute existence
const msg = bundle.getMessage("button");
if (msg) {
  const titlePattern = msg.attributes["title"];
  if (titlePattern) {
    const title = bundle.formatPattern(titlePattern);
  }
}
```

**Why this matters**: During development, one locale may have all attributes defined while another does not. The fallback bundle may not include the same attributes as the primary locale.

---

## Anti-Pattern 6: Using getString() for Attribute Access

`ReactLocalization.getString()` formats the message **value** only. It provides NO access to attributes.

```typescript
// WRONG -- getString only returns the value, not attributes
const { l10n } = useLocalization();
const placeholder = l10n.getString("search-input");
// Returns the message ID "search-input" as fallback because value is null

// CORRECT -- use getBundle + getMessage for attributes
const bundle = l10n.getBundle("search-input");
if (bundle) {
  const msg = bundle.getMessage("search-input");
  if (msg) {
    const pattern = msg.attributes["placeholder"];
    if (pattern) {
      const placeholder = bundle.formatPattern(pattern);
    }
  }
}
```

---

## Anti-Pattern 7: Duplicating Strings Instead of Using Attributes

When a UI element needs multiple translated strings, NEVER create separate messages for each string. Use attributes to group them.

```ftl
# WRONG -- separate messages for the same UI element
login-placeholder = email@example.com
login-aria-label = Login input value
login-title = Type your login email

# CORRECT -- grouped under one message with attributes
login-input =
    .placeholder = email@example.com
    .aria-label = Login input value
    .title = Type your login email
```

**Why this matters**: Grouping related strings under one message helps translators understand context. Localization tools display attributes together, making it easier to maintain consistency.

---

## Anti-Pattern 8: Using Attributes for Unrelated Strings

Attributes are for strings that belong to the SAME UI element. NEVER use attributes to group unrelated strings.

```ftl
# WRONG -- these are unrelated UI elements
page-strings =
    .header = Welcome
    .footer = Copyright 2024
    .sidebar = Navigation

# CORRECT -- separate messages for separate elements
page-header = Welcome
page-footer = Copyright 2024
page-sidebar = Navigation
```

---

## Anti-Pattern 9: Forgetting to Pass Variables to Attribute Formatting

When attributes contain variables, the same `args` object MUST be passed to `formatPattern()` for both the value and the attributes.

```typescript
// WRONG -- variables not passed when formatting attributes
const msg = bundle.getMessage("download-link");
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, { fileName: "report.pdf" });
}
const title = bundle.formatPattern(msg!.attributes["title"]);
// title contains "{$fileName}" literally because args were not passed

// CORRECT -- pass the same args to attribute formatting
const args = { fileName: "report.pdf", fileSize: 2.4 };
const errors: Error[] = [];
if (msg?.value) {
  const text = bundle.formatPattern(msg.value, args, errors);
}
const title = bundle.formatPattern(msg!.attributes["title"], args, errors);
```

---

## Anti-Pattern 10: Creating Value-less Terms

Unlike messages, terms MUST have a value. An attribute-only term is a syntax error.

```ftl
# WRONG -- terms must have a value
-brand =
    .gender = feminine

# CORRECT -- term has a value and an attribute
-brand = Aurora
    .gender = feminine
```

The grammar enforces this: `Term ::= "-" Identifier ... Pattern Attribute*` -- the `Pattern` is required, not optional.
