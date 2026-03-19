# Term Anti-Patterns

## Anti-Pattern 1: Accessing Terms via getMessage()

### WRONG

```typescript
const bundle = new FluentBundle("en");
bundle.addResource(new FluentResource("-brand-name = Firefox"));

// WRONG — returns undefined, terms are NOT accessible via getMessage
const term = bundle.getMessage("-brand-name");
console.log(term); // undefined

// WRONG — hasMessage also returns false for terms
const exists = bundle.hasMessage("-brand-name");
console.log(exists); // false
```

### WHY

Terms are internal-only constructs. The `getMessage()` and `hasMessage()` methods operate exclusively on messages (identifiers without the hyphen prefix). There is NO public API to directly retrieve or format a term. Terms are resolved automatically when referenced inside message patterns.

### CORRECT

```ftl
# Define the term
-brand-name = Firefox

# Reference it in a message
about = About { -brand-name }.
```

```typescript
// Access the MESSAGE that references the term
const msg = bundle.getMessage("about");
const text = bundle.formatPattern(msg.value, {}, errors);
// Result: "About Firefox."
```

---

## Anti-Pattern 2: Variable References in Named Arguments

### WRONG

```ftl
-brand-name =
    { $case ->
       *[nominative] Firefox
        [locative] Firefoksie
    }

# WRONG — $userCase is a variable reference, NOT a literal
about = Informacje o { -brand-name(case: $userCase) }.
```

### WHY

The Fluent grammar restricts named argument values to string literals or number literals:

```ebnf
NamedArgument ::= Identifier blank? ":" blank? (StringLiteral | NumberLiteral)
```

Variable references (`$variable`) are NOT `StringLiteral` or `NumberLiteral`. This is a parse error. The FTL parser will reject this syntax.

### CORRECT

```ftl
# Named argument values MUST be literals
about = Informacje o { -brand-name(case: "locative") }.
download = Pobierz { -brand-name(case: "accusative") }.
```

Each message that references the term provides the appropriate case as a string literal. The case selection is determined at authoring time by the translator, not dynamically at runtime.

---

## Anti-Pattern 3: Treating Terms as Messages

### WRONG

```ftl
# Using a term where a message should be used
-welcome-message = Welcome to our application!

# Trying to display the term directly in application code
```

```typescript
// WRONG — this returns undefined because -welcome-message is a term
const msg = bundle.getMessage("-welcome-message");
// msg is undefined, formatPattern will fail
```

### WHY

Terms are designed for shared vocabulary and brand consistency referenced BY other messages. If text needs to be displayed directly to users, it MUST be a message (no hyphen prefix). Terms are invisible to the runtime API.

### CORRECT

```ftl
# User-facing text → message (no hyphen)
welcome-message = Welcome to our application!

# Shared vocabulary → term (hyphen prefix)
-brand-name = Firefox
```

```typescript
// Messages are accessible
const msg = bundle.getMessage("welcome-message");
const text = bundle.formatPattern(msg.value, {}, errors);
// Result: "Welcome to our application!"
```

---

## Anti-Pattern 4: Attribute-Only Terms

### WRONG

```ftl
# WRONG — terms MUST have a value
-brand-meta =
    .gender = masculine
    .animacy = inanimate
```

### WHY

The Fluent grammar requires terms to have a Pattern (value):

```ebnf
Term ::= "-" Identifier blank_inline? "=" blank_inline? Pattern Attribute*
```

Unlike messages (which allow `Attribute+` without a value), terms MUST have a value because they are designed to be interpolated in placeables like `{ -brand-meta }`. Without a value, the placeable has nothing to resolve to.

### CORRECT

```ftl
# Term with value AND attributes
-brand-name = Firefox
    .gender = masculine
    .animacy = inanimate
```

---

## Anti-Pattern 5: Expecting Term Attributes to Be Accessible via API

### WRONG

```typescript
// Assuming term attributes can be read programmatically
const term = bundle.getMessage("-brand-name"); // undefined
const gender = term.attributes.gender; // TypeError: cannot read property of undefined
```

### WHY

Term attributes are private to the FTL file. They are designed ONLY for use as selectors within FTL patterns. They are NOT exposed through any runtime API. This is by design: grammatical metadata is a localization concern, not an application concern.

### CORRECT

```ftl
# Term attributes drive FTL-internal branching
-brand-name = Firefox
    .gender = masculine

# The attribute controls text selection, not application logic
updated =
    { -brand-name.gender ->
        [masculine] { -brand-name } was updated.
        [feminine] { -brand-name } was updated.
       *[other] { -brand-name } was updated.
    }
```

The application simply formats the `updated` message. It never needs to know about gender — that logic is encapsulated in the FTL file where it belongs.

---

## Anti-Pattern 6: Forgetting the Hyphen Prefix

### WRONG

```ftl
# Missing hyphen — this creates a message, not a term
brand-name = Firefox

about = About { brand-name }.
```

### WHY

Without the hyphen prefix, `brand-name` is a regular message. This means:
1. It is accessible via `bundle.getMessage("brand-name")` (leaks internal vocabulary to the API)
2. It cannot receive named arguments from referencing messages
3. It does not benefit from term-specific resolution semantics

### CORRECT

```ftl
# Hyphen prefix makes it a term
-brand-name = Firefox

about = About { -brand-name }.
```

The hyphen prefix signals to both the Fluent runtime and human translators that this is shared vocabulary, not a user-facing message.

---

## Summary Table

| Anti-Pattern | Problem | Rule |
|-------------|---------|------|
| `getMessage("-term")` | Returns undefined | NEVER access terms via API |
| `{ -term(key: $var) }` | Parse error | NEVER use variable references in named args |
| Term for user-facing text | Invisible to API | ALWAYS use messages for displayed text |
| Attribute-only term | Grammar violation | ALWAYS give terms a value |
| Reading term attributes via API | Returns undefined | NEVER expect API access to term attributes |
| Missing hyphen prefix | Creates a message | ALWAYS use `-` prefix for terms |
