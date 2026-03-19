# Review Scenarios — Good Code, Bad Code, and Fixes

## Scenario 1: Basic Message with Variables

### Bad Code

```ftl
# FTL — wrong variable syntax (no $ prefix)
welcome = Welcome, { name }!
```

```typescript
// TypeScript — old API, no null check, no errors array
const text = bundle.format("welcome", { name: "Anna" });
```

### What's Wrong

1. FTL: `{ name }` is a message reference, not a variable. Missing `$` prefix.
2. TypeScript: `bundle.format()` was removed in v0.14.0. Must use two-step flow.
3. TypeScript: No null check on getMessage result.
4. TypeScript: No errors array — throws on first resolution error.

### Fixed Code

```ftl
# FTL — correct variable reference with $ prefix
welcome = Welcome, { $name }!
```

```typescript
// TypeScript — correct two-step flow with null check and errors
const msg = bundle.getMessage("welcome");
if (msg?.value) {
  const errors: Error[] = [];
  const text = bundle.formatPattern(msg.value, { name: "Anna" }, errors);
  if (errors.length > 0) {
    console.warn("Formatting errors:", errors);
  }
}
```

---

## Scenario 2: Select Expression (Plurals)

### Bad Code

```ftl
# FTL — ICU syntax, missing default variant
emails = {unreadEmails, plural,
    one {You have one email.}
    other {You have {unreadEmails} emails.}
}
```

### What's Wrong

1. Uses ICU MessageFormat syntax — Fluent has its own select syntax.
2. No `$` prefix on variable.
3. No `->` arrow operator.
4. No `*` default variant marker.
5. Variant patterns use `{...}` instead of being on separate lines.

### Fixed Code

```ftl
# FTL — correct Fluent select expression with default variant
emails =
    { $unreadEmails ->
        [one] You have one email.
       *[other] You have { $unreadEmails } emails.
    }
```

---

## Scenario 3: Term Usage

### Bad Code

```ftl
# FTL — term without hyphen prefix
brandName = Firefox

about = About { brandName }.
```

```typescript
// TypeScript — trying to access term via API
const brand = bundle.getMessage("-brand-name");
const brandText = bundle.formatPattern(brand!.value!);
```

### What's Wrong

1. FTL: `brandName` is a message, not a term. Terms MUST use `-` prefix.
2. FTL: Without term syntax, `brandName` is directly accessible via API (bad for brand consistency).
3. TypeScript: `getMessage("-brand-name")` always returns undefined — terms are not accessible via the public API.
4. TypeScript: Force unwrap `!` on potentially undefined values.

### Fixed Code

```ftl
# FTL — correct term with hyphen prefix
-brand-name = Firefox

about = About { -brand-name }.
```

```typescript
// TypeScript — terms are resolved via message references, not direct API access
const msg = bundle.getMessage("about");
if (msg?.value) {
  const errors: Error[] = [];
  const text = bundle.formatPattern(msg.value, null, errors);
  // text → "About Firefox."
}
```

---

## Scenario 4: Parameterized Term with Grammatical Cases

### Bad Code

```ftl
-brand-name =
    { $case ->
       *[nominative] Firefox
        [locative] Firefoksie
    }

# WRONG — variable reference as named argument value
about = About { -brand-name(case: $userCase) }.
```

### What's Wrong

1. Named argument values MUST be string literals or number literals.
2. `$userCase` is a variable reference — not allowed as a named argument value.

### Fixed Code

```ftl
-brand-name =
    { $case ->
       *[nominative] Firefox
        [locative] Firefoksie
    }

# CORRECT — string literal as named argument
about = About { -brand-name(case: "locative") }.
```

---

## Scenario 5: Attributes and Value-less Messages

### Bad Code

```ftl
login-input =
    .placeholder = email@example.com
    .aria-label = Login input value
```

```typescript
// TypeScript — not checking for null value
const msg = bundle.getMessage("login-input")!;
const text = bundle.formatPattern(msg.value!); // CRASH — value is null
```

```tsx
// React — missing attrs declaration
<Localized id="login-input">
  <input type="email" placeholder="Email" aria-label="Login" />
</Localized>
```

### What's Wrong

1. TypeScript: `msg.value` is null for attribute-only messages. Force unwrap crashes.
2. React: Missing `attrs` prop — translated attributes are NOT applied without explicit declaration.

### Fixed Code

```typescript
// TypeScript — handle null value, format attributes instead
const msg = bundle.getMessage("login-input");
if (msg) {
  // Value is null for this message — format attributes
  const errors: Error[] = [];
  const placeholder = bundle.formatPattern(
    msg.attributes["placeholder"], null, errors
  );
  const ariaLabel = bundle.formatPattern(
    msg.attributes["aria-label"], null, errors
  );
}
```

```tsx
// React — declare which attributes should be translated
<Localized
  id="login-input"
  attrs={{ placeholder: true, "aria-label": true }}
>
  <input type="email" placeholder="Email" aria-label="Login" />
</Localized>
```

---

## Scenario 6: Resource Creation and Error Handling

### Bad Code

```typescript
// WRONG — passing raw string to addResource
bundle.addResource(`
hello = Hello, world!
welcome = Welcome, {$name}!
`);

// WRONG — ignoring parse errors
bundle.addResource(new FluentResource(userInput));
```

### What's Wrong

1. `addResource()` expects a `FluentResource` instance, not a raw string. This throws TypeError.
2. Return value of `addResource()` contains parse errors but is ignored.

### Fixed Code

```typescript
// CORRECT — create FluentResource first
const resource = new FluentResource(`
hello = Hello, world!
welcome = Welcome, { $name }!
`);

// CORRECT — check and log parse errors
const errors = bundle.addResource(resource);
if (errors.length > 0) {
  errors.forEach(e => console.error("FTL parse error:", e.message));
}
```

---

## Scenario 7: React Provider Setup

### Bad Code

```tsx
function App() {
  // WRONG — creates new ReactLocalization on every render
  const l10n = new ReactLocalization(generateBundles(locales));

  return (
    <LocalizationProvider l10n={l10n}>
      <Content />
    </LocalizationProvider>
  );
}
```

### What's Wrong

1. Creating `new ReactLocalization()` on every render causes all localized components to re-render unnecessarily.
2. No stable reference — React sees a new object every time.

### Fixed Code

```tsx
function App() {
  // CORRECT — stable reference via useState
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

---

## Scenario 8: DOM Overlays with elems

### Bad Code

```ftl
confirm-action = <btn>Confirm</btn> or <lnk>cancel</lnk>.
```

```tsx
// WRONG — elems keys don't match FTL tag names
<Localized
  id="confirm-action"
  elems={{
    button: <button onClick={handleConfirm}></button>,
    link: <Link to="/"></Link>,
  }}
>
  <p>Confirm or cancel.</p>
</Localized>
```

### What's Wrong

1. FTL uses `<btn>` and `<lnk>` tags.
2. React `elems` uses `button` and `link` keys.
3. Keys MUST match exactly — `btn` not `button`, `lnk` not `link`.

### Fixed Code

```tsx
// CORRECT — elems keys match FTL tag names exactly
<Localized
  id="confirm-action"
  elems={{
    btn: <button onClick={handleConfirm}></button>,
    lnk: <Link to="/"></Link>,
  }}
>
  <p>Confirm or cancel.</p>
</Localized>
```

---

## Scenario 9: Whitespace and Indentation Errors

### Bad Code

```ftl
# WRONG — tab indentation (tabs become text content)
greeting =
	Hello, { $name }!
	Welcome to our platform.
```

### What's Wrong

1. Tab characters (U+0009) are NOT indentation. They are treated as regular text content.
2. The parser does not see these lines as continuation lines.
3. The tab character appears literally in the output.

### Fixed Code

```ftl
# CORRECT — space indentation only
greeting =
    Hello, { $name }!
    Welcome to our platform.
```

---

## Scenario 10: Complete Production Review

### Code Under Review

```ftl
### Application Messages

## Navigation
nav-home = Home
nav-settings = Settings

## Welcome Section
# Shown after login
# $userName (String) - The user's display name
welcome-back = Welcome back, { $userName }!

-app-name = MyApp
    .gender = neuter

emails =
    { $count ->
        [one] You have one new email in { -app-name }.
       *[other] You have { $count } new emails in { -app-name }.
    }

login-input =
    .placeholder = Enter your email
    .aria-label = Email login field
```

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

const bundle = new FluentBundle("en-US");
const resource = new FluentResource(ftlContent);
const errors = bundle.addResource(resource);
if (errors.length > 0) {
  console.error("Parse errors:", errors);
}

function translate(id: string, args?: Record<string, string | number>): string {
  const msg = bundle.getMessage(id);
  if (!msg) {
    console.warn(`Missing translation: ${id}`);
    return id;
  }
  if (msg.value) {
    const fmtErrors: Error[] = [];
    const text = bundle.formatPattern(msg.value, args, fmtErrors);
    if (fmtErrors.length > 0) {
      console.warn(`Format errors for ${id}:`, fmtErrors);
    }
    return text;
  }
  return id;
}
```

### Review Verdict: PASS

All checks satisfied:
- FTL syntax correct: spaces for indentation, valid identifiers, proper comment levels
- Select expression has `*[other]` default, uses valid CLDR `[one]` category
- Term has `-` prefix, has a value, attributes used correctly
- Resource comment (`###`), group comments (`##`), message comments (`#`) all correct
- TypeScript: two-step flow, null check, errors array, parse error logging
- Missing message handled gracefully with fallback to ID
- Attribute-only message (`login-input`) would need separate attribute formatting — the `translate` function correctly returns `id` when `msg.value` is null

### Improvement Suggestion

The `translate` helper could also expose attribute formatting:

```typescript
function translateAttr(
  id: string,
  attr: string,
  args?: Record<string, string | number>
): string {
  const msg = bundle.getMessage(id);
  if (!msg || !msg.attributes[attr]) {
    return `${id}.${attr}`;
  }
  const errors: Error[] = [];
  return bundle.formatPattern(msg.attributes[attr], args, errors);
}
```
