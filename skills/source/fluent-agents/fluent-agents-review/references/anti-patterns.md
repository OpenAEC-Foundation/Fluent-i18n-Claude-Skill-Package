# Anti-Patterns — Consolidated Reference

All known anti-patterns for Project Fluent development, sourced from FTL syntax edge cases (Part 1, Section 1.12), @fluent/bundle API analysis (Part 2, Section 2.10), and @fluent/react integration issues (Part 3, Section 3.10).

---

## FTL Syntax Anti-Patterns

### AP-01: Tab Indentation

**What**: Using tab characters for FTL indentation.
**Why it fails**: Tabs (U+0009) are NOT whitespace for indentation. They become literal text content in the pattern output.
**Fix**: ALWAYS use space characters (U+0020) for indentation.

### AP-02: Missing Default Variant

**What**: Select expression without a `*[key]` default variant.
**Why it fails**: The grammar mandates exactly one `DefaultVariant`. Omitting it is a syntax error that produces Junk.
**Fix**: ALWAYS include exactly one default variant marked with `*`.

### AP-03: ICU MessageFormat Syntax

**What**: Using `{count, plural, one{...} other{...}}` in FTL files.
**Why it fails**: Fluent has its own syntax: `{ $count -> [one] ... *[other] ... }`. ICU syntax is parsed as a broken placeable.
**Fix**: Use Fluent's select expression syntax with `->` arrow and `$variable` references.

### AP-04: Invalid CLDR Plural Categories

**What**: Using invented category names like `none`, `single`, `multiple`, `some`.
**Why it fails**: Only CLDR categories are recognized: `zero`, `one`, `two`, `few`, `many`, `other`. Invented names never match and fall through to default.
**Fix**: Use only valid CLDR plural categories for the target language.

### AP-05: Variable Reference as Named Argument

**What**: `{ -term(case: $userCase) }` — passing a variable as a named argument value.
**Why it fails**: Named argument values MUST be string literals or number literals per the grammar: `NamedArgument ::= Identifier ":" (StringLiteral | NumberLiteral)`.
**Fix**: Use a literal value: `{ -term(case: "nominative") }`.

### AP-06: Special Character at Line Start

**What**: Starting a continuation line with `[`, `*`, or `.` without escaping.
**Why it fails**: These characters are reserved: `[` starts a variant key, `*` marks a default variant, `.` starts an attribute. The indented_char production excludes them.
**Fix**: Use quoted text placeables: `{"["}`, `{"*"}`, `{"."}`.

### AP-07: Comment Without Space

**What**: `#comment` instead of `# comment`.
**Why it fails**: The grammar requires a space (U+0020) after `#`, `##`, or `###`. Without it, the line is not a valid comment.
**Fix**: ALWAYS include a space: `# comment text`.

### AP-08: Attribute-Only Term

**What**: A term with only attributes and no value pattern.
**Why it fails**: Terms MUST have a value. The grammar: `Term ::= "-" Identifier "=" Pattern Attribute*`. Unlike Messages, the value is not optional.
**Fix**: ALWAYS give terms a value. Use attributes for metadata.

### AP-09: Wrong Variable Syntax

**What**: Using `{name}`, `{{name}}`, `%{name}`, or `${name}` instead of `{ $name }`.
**Why it fails**: `{name}` is a message reference (looks up message named "name"). Other syntaxes are not valid Fluent at all.
**Fix**: ALWAYS use `{ $name }` with `$` prefix and spaces inside braces.

---

## @fluent/bundle API Anti-Patterns

### AP-10: Raw String to addResource

**What**: `bundle.addResource("hello = Hello")` — passing a string instead of FluentResource.
**Why it fails**: `addResource()` expects a `FluentResource` instance. Passing a string throws TypeError.
**Fix**: `bundle.addResource(new FluentResource("hello = Hello"))`.

### AP-11: Removed API Methods

**What**: Calling `bundle.format()`, `bundle.addMessages()`, or `FluentResource.fromString()`.
**Why it fails**: These methods were removed in v0.14.0 (July 2019). They do not exist on current versions.
**Fix**: Use `getMessage()` + `formatPattern()` for formatting, `addResource(new FluentResource(...))` for loading, and the `FluentResource` constructor directly.

### AP-12: No Null Check on getMessage

**What**: `bundle.getMessage("id")!.value!` — force unwrapping without null check.
**Why it fails**: `getMessage()` returns `undefined` if the message does not exist. `msg.value` is `null` for attribute-only messages. Force unwrap crashes at runtime.
**Fix**: ALWAYS guard: `const msg = bundle.getMessage("id"); if (msg?.value) { ... }`.

### AP-13: Missing Errors Array in Production

**What**: `bundle.formatPattern(pattern, args)` — no third argument.
**Why it fails**: Without an errors array, `formatPattern` throws on the first resolution error (missing variable, broken reference, type mismatch). In production, this crashes the app for a translation issue.
**Fix**: ALWAYS pass an errors array in production: `bundle.formatPattern(pattern, args, errors)`.

### AP-14: Accessing Terms via getMessage

**What**: `bundle.getMessage("-brand-name")` — trying to read a term through the public API.
**Why it fails**: Terms are stored internally in `_terms`, not `_messages`. `getMessage()` only searches `_messages`. This always returns `undefined`.
**Fix**: Terms are resolved automatically when referenced in FTL messages: `about = About { -brand-name }.`. There is no public API for direct term access.

### AP-15: One FluentResource Per Message

**What**: Creating a `new FluentResource()` for each individual message in a loop.
**Why it fails**: Parser initialization overhead multiplied by message count. Unnecessary memory allocation and GC pressure.
**Fix**: Batch all messages into a single FTL string and create one `FluentResource` per file or logical group.

### AP-16: @fluent/syntax for Runtime

**What**: Using `@fluent/syntax` (FluentParser) to parse FTL at runtime in a production app.
**Why it fails**: `@fluent/syntax` produces a full AST — it is designed for tooling (linting, generation, editors). It is significantly larger and slower than the runtime parser in `@fluent/bundle`.
**Fix**: Use `@fluent/bundle`'s `FluentResource` for runtime. Reserve `@fluent/syntax` for build-time tools.

### AP-17: Invalid Variable Types

**What**: Passing objects, arrays, or booleans as FluentVariable values.
**Why it fails**: `FluentVariable` accepts only `string`, `number`, `Date`, `FluentType`, or `TemporalObject`. Other types cause TypeError or produce `{???}` output.
**Fix**: Convert to accepted types before passing: `String(value)`, `Number(value)`, etc.

### AP-18: Ignoring addResource Errors

**What**: Calling `bundle.addResource(resource)` without checking the return value.
**Why it fails**: Parse errors are returned as an `Error[]` array. Ignoring them means broken FTL is silently accepted, causing runtime resolution errors later.
**Fix**: ALWAYS check and log: `const errors = bundle.addResource(resource); if (errors.length) { ... }`.

### AP-19: Disabling useIsolating for RTL Locales

**What**: `new FluentBundle("ar", { useIsolating: false })` — disabling Unicode isolation for Arabic or other RTL languages.
**Why it fails**: Unicode isolation marks (U+2068/U+2069) prevent bidirectional text from corrupting surrounding layout. Disabling them for RTL locales causes LTR placeables (numbers, brand names) to break text flow.
**Fix**: ONLY disable `useIsolating` when content is unidirectional OR during testing where Unicode marks interfere with assertions.

---

## @fluent/react Anti-Patterns

### AP-20: Post-Translation String Manipulation

**What**: Concatenating, splitting, or modifying translated strings after formatting.
**Why it fails**: Translations may contain Unicode isolation marks, locale-specific formatting, or markup. Manipulation breaks the carefully constructed output.
**Fix**: ALWAYS treat translation output as opaque. Use FTL placeables and selectors for all string composition.

### AP-21: Changing IDs Without Semantic Change

**What**: Renaming a translation ID while keeping the same English text, or vice versa: changing the meaning without changing the ID.
**Why it fails**: Translation IDs are contracts with translators. Changed meaning with same ID means stale translations silently appear. Changed ID with same meaning triggers unnecessary re-translation.
**Fix**: ALWAYS change the ID when the message meaning changes. Keep the ID stable when only fixing typos.

### AP-22: Missing attrs Declaration

**What**: Using `<Localized id="input">` without `attrs={{ placeholder: true }}` when the FTL has `.placeholder`.
**Why it fails**: Translated attributes are ONLY applied when explicitly declared as `true` in the `attrs` object. This is a security measure — translations cannot inject arbitrary attributes.
**Fix**: ALWAYS declare every attribute that should be translated: `attrs={{ placeholder: true, "aria-label": true }}`.

### AP-23: ReactLocalization in Render

**What**: `const l10n = new ReactLocalization(bundles)` inside a component's render function without memoization.
**Why it fails**: Creates a new object on every render. React sees a new context value, causing ALL localized components in the tree to re-render.
**Fix**: Use `useState(() => new ReactLocalization(bundles))` or `useMemo()` for a stable reference.

### AP-24: Using withLocalization HOC

**What**: Wrapping components with `withLocalization(MyComponent)` instead of using the `useLocalization` hook.
**Why it fails**: The HOC does not support React ref forwarding (GitHub issue #598). This breaks parent-to-child ref patterns.
**Fix**: Use `const { l10n } = useLocalization()` in functional components.

### AP-25: Missing reportError Callback

**What**: `new ReactLocalization(bundles)` without a `reportError` function.
**Why it fails**: Missing translations are silently swallowed. The default behavior is `console.warn`, which may be acceptable in development but insufficient for production monitoring.
**Fix**: Provide a `reportError` callback for production: `new ReactLocalization(bundles, null, (err) => errorTracker.log(err))`.

---

## Summary Table

| ID | Area | Severity | Short Description |
|----|------|----------|-------------------|
| AP-01 | FTL | Critical | Tab indentation |
| AP-02 | FTL | Critical | Missing default variant |
| AP-03 | FTL | Critical | ICU syntax in FTL |
| AP-04 | FTL | High | Invalid CLDR categories |
| AP-05 | FTL | High | Variable ref as named arg |
| AP-06 | FTL | Medium | Special char at line start |
| AP-07 | FTL | Medium | Comment without space |
| AP-08 | FTL | High | Attribute-only term |
| AP-09 | FTL | Critical | Wrong variable syntax |
| AP-10 | API | Critical | Raw string to addResource |
| AP-11 | API | Critical | Removed API methods |
| AP-12 | API | Critical | No null check on getMessage |
| AP-13 | API | Critical | Missing errors array |
| AP-14 | API | High | Accessing terms via getMessage |
| AP-15 | API | Medium | One FluentResource per message |
| AP-16 | API | Medium | @fluent/syntax for runtime |
| AP-17 | API | High | Invalid variable types |
| AP-18 | API | High | Ignoring addResource errors |
| AP-19 | API | High | Disabling useIsolating for RTL |
| AP-20 | React | High | Post-translation string manipulation |
| AP-21 | React | Medium | Changing IDs without semantic change |
| AP-22 | React | High | Missing attrs declaration |
| AP-23 | React | High | ReactLocalization in render |
| AP-24 | React | Medium | Using withLocalization HOC |
| AP-25 | React | Medium | Missing reportError callback |
