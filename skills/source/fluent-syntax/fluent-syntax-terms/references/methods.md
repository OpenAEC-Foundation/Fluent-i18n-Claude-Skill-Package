# Term Grammar and Comparison Reference

## EBNF Grammar: Term

```ebnf
Term ::= "-" Identifier blank_inline? "=" blank_inline? Pattern Attribute*
```

A Term is defined by:
1. A hyphen (`-`) prefix
2. An `Identifier` (letters, digits, hyphens after first character)
3. An equals sign with optional surrounding whitespace
4. A **mandatory** `Pattern` (the term's value)
5. Zero or more `Attribute` entries

The Pattern is NOT optional. Every term MUST have a value. This differs from Message, where value is optional if attributes are present.

## EBNF Grammar: TermReference

```ebnf
TermReference ::= "-" Identifier AttributeAccessor? CallArguments?
```

A TermReference inside a placeable can:
- Reference the term's value: `{ -brand-name }`
- Access a term attribute: `{ -brand-name.gender }`
- Pass named arguments: `{ -brand-name(case: "locative") }`
- Combine attribute access and arguments: `{ -brand-name.gender(case: "locative") }`

## EBNF Grammar: CallArguments and NamedArgument

```ebnf
CallArguments ::= blank? "(" blank? argument_list blank? ")"
argument_list ::= (Argument blank? "," blank?)* Argument?
Argument      ::= NamedArgument | InlineExpression
NamedArgument ::= Identifier blank? ":" blank? (StringLiteral | NumberLiteral)
```

For term references, CallArguments contain ONLY NamedArgument entries (not positional InlineExpression). The grammar technically allows both, but the Fluent resolver only processes named arguments for terms.

### NamedArgument Value Restrictions

Named argument values MUST be one of:

| Type | Syntax | Example |
|------|--------|---------|
| StringLiteral | `"text"` | `case: "locative"` |
| NumberLiteral | digits with optional decimal | `count: 3` |

The following are NOT valid named argument values:
- Variable references: `case: $myVar` -- NOT valid
- Term references: `case: { -other-term }` -- NOT valid
- Function calls: `case: FUNC()` -- NOT valid

## EBNF Grammar: Attribute

```ebnf
Attribute         ::= line_end blank? "." Identifier blank_inline? "=" blank_inline? Pattern
AttributeAccessor ::= "." Identifier
```

Attributes on terms:
- Are defined on indented continuation lines after the term value
- Start with a dot (`.`) followed by an identifier
- Have their own Pattern (value)
- Are private: ONLY usable as selectors within FTL, NOT accessible via runtime API

## Term vs Message Grammar Comparison

```ebnf
Message ::= Identifier blank_inline? "=" blank_inline? ((Pattern Attribute*) | (Attribute+))
Term    ::= "-" Identifier blank_inline? "=" blank_inline? Pattern Attribute*
```

| Grammar Aspect | Message | Term |
|----------------|---------|------|
| Identifier prefix | None | `-` (hyphen) |
| Value (Pattern) | Optional if attributes present | MANDATORY |
| Attributes | Optional | Optional |
| Attribute-only (no value) | Valid | NOT valid |
| Referenced via | `MessageReference` | `TermReference` |
| Can receive named args | No (uses application variables) | Yes (from referencing message) |

### Why Terms MUST Have a Value

Terms are designed to be interpolated into messages via placeables like `{ -brand-name }`. A placeable resolves to a string value. If a term had no value (only attributes), the placeable would have nothing to resolve to. Messages can be attribute-only because they are accessed programmatically via `getMessage().attributes`, but terms have no such public accessor.

## MessageReference vs TermReference

```ebnf
MessageReference ::= Identifier AttributeAccessor?
TermReference    ::= "-" Identifier AttributeAccessor? CallArguments?
```

| Feature | MessageReference | TermReference |
|---------|-----------------|---------------|
| CallArguments | NOT allowed | Allowed |
| AttributeAccessor | Allowed | Allowed |
| Hyphen prefix | No | Yes |
| Passes data to target | No | Yes (via named args) |

Messages cannot receive parameters from other messages. Terms CAN receive parameters from referencing messages. This is the fundamental mechanism that enables grammatical case handling across languages.
