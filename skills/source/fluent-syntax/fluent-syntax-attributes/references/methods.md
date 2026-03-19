# Attributes -- Grammar and API Methods

## EBNF Grammar

### Attribute Production

```ebnf
Attribute ::= line_end blank? "." Identifier blank_inline? "=" blank_inline? Pattern
```

An attribute:
1. Starts on a new line (`line_end`)
2. Is optionally indented (`blank?`)
3. Begins with a dot (`.`)
4. Followed by an `Identifier` (the attribute name)
5. Optional whitespace, then `=`, then optional whitespace
6. Then a `Pattern` (the attribute value)

### AttributeAccessor Production

```ebnf
AttributeAccessor ::= "." Identifier
```

Used in `MessageReference` and `TermReference` to access a specific attribute:

```ebnf
MessageReference ::= Identifier AttributeAccessor?
TermReference    ::= "-" Identifier AttributeAccessor? CallArguments?
```

### Message Production (Attribute Context)

```ebnf
Message ::= Identifier blank_inline? "=" blank_inline? ((Pattern Attribute*) | (Attribute+))
```

A message MUST have:
- A value (Pattern) with zero or more attributes: `(Pattern Attribute*)`
- OR one or more attributes without a value: `(Attribute+)`

This means value-less messages are grammatically valid.

### Term Production (Attribute Context)

```ebnf
Term ::= "-" Identifier blank_inline? "=" blank_inline? Pattern Attribute*
```

A term MUST have a value (Pattern). Attributes are optional. Unlike messages, a term CANNOT be attribute-only.

### Identifier Production

```ebnf
Identifier ::= [a-zA-Z] [a-zA-Z0-9_-]*
```

Attribute names follow the same `Identifier` rules as message names. They MUST start with a letter and MAY contain letters, digits, underscores, and hyphens. Common attribute names: `placeholder`, `aria-label`, `title`, `alt`, `value`, `gender`, `animacy`.

---

## getMessage() API

### Signature

```typescript
getMessage(id: string): Message | undefined
```

### Message Interface

```typescript
interface Message {
  value: Pattern | null;
  attributes: Record<string, Pattern>;
}
```

| Property | Type | Description |
|----------|------|-------------|
| `value` | `Pattern \| null` | The main message pattern. `null` when the message has only attributes. |
| `attributes` | `Record<string, Pattern>` | A record of attribute patterns keyed by attribute name. Empty object `{}` when no attributes exist. |

### formatPattern() for Attributes

```typescript
formatPattern(
  pattern: Pattern,
  args?: Record<string, FluentVariable> | null,
  errors?: Array<Error> | null
): string
```

Attribute patterns are formatted the same way as message values. Pass `msg.attributes["attr-name"]` as the `pattern` argument.

### hasMessage()

```typescript
hasMessage(id: string): boolean
```

Returns `true` if the bundle contains a message with the given identifier. Does NOT distinguish between messages with values and value-less (attribute-only) messages -- both return `true`.

---

## React API for Attributes

### Localized Component -- attrs Prop

```typescript
interface LocalizedProps {
  id: string;
  attrs?: Record<string, boolean>;
  vars?: Record<string, FluentVariable>;
  elems?: Record<string, ReactElement>;
  children?: ReactElement;
}
```

The `attrs` prop is a `Record<string, boolean>` where:
- Keys are FTL attribute names (matching `.attr-name` in FTL)
- Values MUST be `true` to enable translation of that attribute
- Attributes NOT listed or set to `false` are NOT translated

### ReactLocalization.getElement()

```typescript
getElement(
  sourceElement: ReactElement,
  id: string,
  args?: {
    vars?: Record<string, FluentVariable>;
    elems?: Record<string, ReactElement>;
    attrs?: Record<string, boolean>;
  }
): ReactElement
```

This is the internal method used by `<Localized>`. It resolves both the message value and the declared attributes, applying them to the source element.

### ReactLocalization.getString()

```typescript
getString(
  id: string,
  vars?: Record<string, FluentVariable> | null,
  fallback?: string
): string
```

`getString()` formats the message **value** only. It does NOT provide access to attributes. For imperative attribute access, use `getBundle()` followed by `getMessage()` and `formatPattern()`.
