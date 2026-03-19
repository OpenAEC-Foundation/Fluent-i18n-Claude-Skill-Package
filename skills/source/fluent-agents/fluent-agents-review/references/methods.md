# Validation Checklist — Complete Methods Reference

## Area 1: FTL Syntax Compliance

### 1.1 Whitespace Rules

| # | Check Item | Pass Criteria | Fail Indicator |
|---|-----------|---------------|----------------|
| W-01 | Indentation uses spaces only | All indent chars are U+0020 | Tab characters (U+0009) found in indent position |
| W-02 | Continuation lines indented | Every pattern continuation has >= 1 leading space | Line at column 0 inside a multiline pattern |
| W-03 | No special chars at line start | First char after indent is not `[`, `*`, or `.` | Parser treats line as variant/attribute instead of text |
| W-04 | Dedentation is consistent | Common indent stripped from all block lines | Mixed indent levels produce unexpected leading spaces |
| W-05 | Leading blank lines ignored | Blank lines between `=` and first content are stripped | Assuming blank lines appear in output |
| W-06 | Interior blank lines preserved | Blank line with content above AND below is kept | Expecting all blank lines to be stripped |
| W-07 | Trailing blank lines stripped | No trailing whitespace in pattern output | Assuming trailing blanks appear in output |

### 1.2 Identifier Format

| # | Check Item | Pass Criteria | Fail Indicator |
|---|-----------|---------------|----------------|
| I-01 | Message ID starts with letter | First char matches `[a-zA-Z]` | Starts with number, hyphen, underscore, or special char |
| I-02 | ID contains valid chars only | After first char: `[a-zA-Z0-9_-]*` | Dots, spaces, or other special chars in ID |
| I-03 | Term ID has hyphen prefix | Term identifiers start with `-` | Term without `-` prefix (becomes a message) |
| I-04 | Attribute ID valid | `.identifier` after message/term, valid identifier chars | Space between `.` and attribute name |

### 1.3 Comment Syntax

| # | Check Item | Pass Criteria | Fail Indicator |
|---|-----------|---------------|----------------|
| C-01 | Space after hash | `# text`, not `#text` | No space after `#`, `##`, or `###` |
| C-02 | Correct comment level | `#` for message, `##` for group, `###` for resource | Wrong level for the scope of the comment |
| C-03 | Message binding | `#` comment directly above message (no blank line) binds | Blank line breaks binding; standalone when binding expected |
| C-04 | Group/resource standalone | `##` and `###` NEVER bind to messages | Expecting `##` comment to attach to next message |

### 1.4 Special Characters

| # | Check Item | Pass Criteria | Fail Indicator |
|---|-----------|---------------|----------------|
| S-01 | Literal curly braces escaped | `{"{"}` and `{"}"}` used | Raw `{` or `}` in text breaks parsing |
| S-02 | Escape sequences in strings only | `\"`, `\\`, `\uHHHH`, `\UHHHHHH` only inside `"..."` | Escape sequences in unquoted text (backslash is literal) |
| S-03 | Non-breaking space via escape | `{"\u00A0"}` for NBSP | Raw NBSP character (invisible, hard to maintain) |
| S-04 | Leading spaces via quoted text | `{"    "}Text` to preserve leading spaces | Raw leading spaces stripped by dedentation |

---

## Area 2: Select Expressions

| # | Check Item | Pass Criteria | Fail Indicator |
|---|-----------|---------------|----------------|
| SE-01 | Default variant present | Exactly one `*[key]` in each select | No default — syntax error |
| SE-02 | Valid CLDR plural categories | `zero`, `one`, `two`, `few`, `many`, `other` | Invented: `none`, `single`, `multiple`, `some` |
| SE-03 | No ICU syntax | Uses `{ $var -> [key] pattern }` | Uses `{var, plural, one{...} other{...}}` |
| SE-04 | Arrow operator correct | `->` (hyphen, greater-than) | `=>` or `-->` or other variants |
| SE-05 | Variants on separate lines | Each variant starts on new indented line | Multiple variants on one line |
| SE-06 | Exact match before category | Number literals `[1]` checked before CLDR `[one]` | Relying on CLDR for exact values |
| SE-07 | Ordinal uses NUMBER type | `NUMBER($pos, type: "ordinal")` for ordinals | Plain variable in selector for ordinal matching |
| SE-08 | Formatted selector consistency | Same NUMBER options in selector and display | Different formatting in selector vs. placeable |

---

## Area 3: Terms

| # | Check Item | Pass Criteria | Fail Indicator |
|---|-----------|---------------|----------------|
| T-01 | Hyphen prefix present | All terms start with `-` | Term without prefix — treated as message |
| T-02 | Term has value | Every term has a Pattern (not attribute-only) | Attribute-only term — grammar violation |
| T-03 | Named args are literals | `{ -term(case: "nominative") }` | `{ -term(case: $var) }` — variable not allowed |
| T-04 | No API access to terms | Terms resolve only via FTL references | `getMessage("-brand")` in TypeScript — returns undefined |
| T-05 | Term attributes are private | Attributes used only as selectors in FTL | Attempting to display `-brand.gender` value |
| T-06 | Parameterized terms correct | Args passed from referencing message | Expecting term to receive app-level variables |

---

## Area 4: Attributes

| # | Check Item | Pass Criteria | Fail Indicator |
|---|-----------|---------------|----------------|
| A-01 | Attribute syntax correct | `.attr-name = value` on indented new line | Attribute on same line as message value |
| A-02 | Value-less message handled | Code checks `msg.value` for null before formatting | `msg.value!` — crashes on attribute-only messages |
| A-03 | Attribute access in code | `bundle.formatPattern(msg.attributes["placeholder"])` | Trying to access attributes as `msg.placeholder` |
| A-04 | React attrs declared | `attrs={{ title: true }}` on `<Localized>` | Missing attrs — attributes silently not applied |
| A-05 | Attribute names match FTL | React attrs keys match FTL `.attribute` names | Mismatch between FTL and React attr names |

---

## Area 5: API Usage (@fluent/bundle)

| # | Check Item | Pass Criteria | Fail Indicator |
|---|-----------|---------------|----------------|
| API-01 | FluentResource constructor | `new FluentResource(ftlString)` | Raw string to `addResource()` — TypeError |
| API-02 | addResource error check | `const errors = bundle.addResource(res)` + log | Return value ignored |
| API-03 | Two-step formatting | `getMessage()` then `formatPattern()` | Calling `bundle.format()` — removed in v0.14.0 |
| API-04 | getMessage null check | `if (msg?.value)` guard | Force unwrap `msg!.value!` |
| API-05 | Errors array in production | `formatPattern(pattern, args, errors)` | No third argument — throws on first error |
| API-06 | Valid variable types | `string`, `number`, `Date`, `FluentType`, `TemporalObject` | Objects, arrays, booleans passed as variables |
| API-07 | No term via getMessage | Terms not accessible through public API | `getMessage("-term-id")` — always undefined |
| API-08 | Batch resources | One FluentResource per FTL file, not per message | `new FluentResource()` inside a loop per message |
| API-09 | Correct imports | `from "@fluent/bundle"` (not `from "fluent"`) | Old package name `fluent` (pre-0.13.0) |
| API-10 | allowOverrides explicit | `{ allowOverrides: true }` when intentional | Duplicate IDs without flag produce errors |

---

## Area 6: React Integration (@fluent/react)

| # | Check Item | Pass Criteria | Fail Indicator |
|---|-----------|---------------|----------------|
| R-01 | Provider wraps app | `<LocalizationProvider l10n={l10n}>` at root | Localized components outside provider — context missing |
| R-02 | Stable l10n reference | `useState` or `useMemo` for ReactLocalization | Creating new instance on every render |
| R-03 | Localized id prop | `<Localized id="message-id">` | Missing or empty `id` prop |
| R-04 | Localized attrs prop | `attrs={{ title: true }}` for translatable attrs | Missing attrs — silent translation failure |
| R-05 | Localized vars prop | `vars={{ userName: "Anna" }}` matches FTL `$userName` | Variable name mismatch between React and FTL |
| R-06 | Localized elems prop | Keys match FTL `<tag>` names | Element key doesn't match FTL markup tag |
| R-07 | Fallback child present | Meaningful English text as child | Empty or missing child element |
| R-08 | useLocalization hook | `const { l10n } = useLocalization()` | Wrong destructuring or calling outside provider |
| R-09 | getString for imperative | `l10n.getString(id, vars, fallback)` | Using nonexistent `l10n.format()` |
| R-10 | Prefer hook over HOC | `useLocalization` over `withLocalization` | HOC used — no ref forwarding support |
| R-11 | SSR parseMarkup | Custom `parseMarkup` for server-side rendering | Relying on `<template>` element on server |
| R-12 | Bundles ready before render | All translations loaded before provider mounts | Rendering provider before async fetch completes |

---

## Area 7: Error Handling

| # | Check Item | Pass Criteria | Fail Indicator |
|---|-----------|---------------|----------------|
| E-01 | Parse errors logged | `addResource()` return value checked and logged | Errors silently discarded |
| E-02 | Format errors collected | `errors[]` array passed to `formatPattern()` | No array — throws on resolution error |
| E-03 | Missing message handled | `hasMessage()` check or undefined guard | Calling formatPattern on undefined pattern |
| E-04 | Fallback strategy | Graceful degradation with message ID or fallback text | App crash on missing translation |
| E-05 | reportError configured | `new ReactLocalization(bundles, null, reportError)` | Missing translations silently ignored |
| E-06 | Cyclic reference protection | Errors array catches RangeError from cycles | Unhandled crash from circular message references |
| E-07 | Excessive placeables | Aware that >100 placeables is fatal | No protection against user-provided FTL with many placeables |
