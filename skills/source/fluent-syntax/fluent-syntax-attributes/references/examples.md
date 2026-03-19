# Attributes -- Complete Examples

## FTL Message Attributes

### Basic Message with Attributes

```ftl
# A text input with placeholder and accessible label
login-input = Predefined value
    .placeholder = email@example.com
    .aria-label = Login input value
    .title = Type your login email
```

Four translatable strings under one identifier. The value is "Predefined value"; the three attributes provide HTML-level metadata.

### Value-less Message (Attributes Only)

```ftl
# An input with no visible text, only HTML attributes
search-input =
    .placeholder = Search...
    .aria-label = Search the site
    .title = Enter your search query
```

`search-input` has no value (`msg.value` is `null`). This is valid FTL per the grammar: `Message ::= ... (Attribute+)`.

### Compound Message (Value + Attributes)

```ftl
# A button with visible text and a tooltip
submit-button = Submit
    .title = Click to submit your form
    .aria-label = Submit the registration form
```

The value "Submit" is the button text. The `.title` and `.aria-label` are HTML attributes.

### Attributes with Variables

```ftl
# Attributes can contain placeables and variables
download-link = Download { $fileName }
    .title = Click to download { $fileName } ({ $fileSize } MB)
    .aria-label = Download file { $fileName }
```

### Attributes with Message References

```ftl
app-name = My Application

about-link = About
    .title = Learn more about { app-name }
```

### Multiline Attribute Values

```ftl
tooltip =
    .title =
        This is a long tooltip text
        that spans multiple lines.
        Each line is indented.
```

Attribute values follow the same multiline rules as message values: continuation lines MUST be indented by at least one space.

---

## FTL Term Attributes

### Gender Metadata on Terms

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

The `.gender` attribute is private -- it is ONLY usable as a selector, not retrievable at runtime.

### Animacy Metadata on Terms

```ftl
-robot = RoboHelper
    .animacy = inanimate

-assistant = Alice
    .animacy = animate

greeted-by =
    { -assistant.animacy ->
        [animate] You were greeted by { -assistant }.
       *[inanimate] { -assistant } displayed a greeting.
    }
```

### Multiple Term Attributes

```ftl
-product = Thunderbird
    .gender = masculine
    .starts-with-vowel = no

product-description =
    { -product.starts-with-vowel ->
        [yes] An { -product } update is available.
       *[no] A { -product } update is available.
    }
```

---

## TypeScript -- Accessing Attributes

### Value-less Message Access

```typescript
import { FluentBundle, FluentResource } from "@fluent/bundle";

const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(`
search-input =
    .placeholder = Search...
    .aria-label = Search the site
`));

const msg = bundle.getMessage("search-input");
if (msg) {
  // msg.value is null -- this is an attribute-only message
  console.log(msg.value); // null

  // Access attributes
  const errors: Error[] = [];
  const placeholder = bundle.formatPattern(
    msg.attributes["placeholder"], null, errors
  );
  // -> "Search..."

  const ariaLabel = bundle.formatPattern(
    msg.attributes["aria-label"], null, errors
  );
  // -> "Search the site"
}
```

### Compound Message Access

```typescript
const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(`
submit-button = Submit
    .title = Click to submit your form
    .aria-label = Submit the registration form
`));

const msg = bundle.getMessage("submit-button");
if (msg) {
  const errors: Error[] = [];

  // Format the value
  if (msg.value) {
    const buttonText = bundle.formatPattern(msg.value, null, errors);
    // -> "Submit"
  }

  // Format attributes
  const title = bundle.formatPattern(
    msg.attributes["title"], null, errors
  );
  // -> "Click to submit your form"
}
```

### Attributes with Variables

```typescript
const bundle = new FluentBundle("en-US");
bundle.addResource(new FluentResource(`
download-link = Download { $fileName }
    .title = Click to download { $fileName } ({ $fileSize } MB)
`));

const msg = bundle.getMessage("download-link");
if (msg) {
  const args = { fileName: "report.pdf", fileSize: 2.4 };
  const errors: Error[] = [];

  if (msg.value) {
    const text = bundle.formatPattern(msg.value, args, errors);
    // -> "Download report.pdf"
  }

  const title = bundle.formatPattern(
    msg.attributes["title"], args, errors
  );
  // -> "Click to download report.pdf (2.4 MB)"
}
```

### Iterating Over All Attributes

```typescript
const msg = bundle.getMessage("login-input");
if (msg) {
  const errors: Error[] = [];
  for (const [name, pattern] of Object.entries(msg.attributes)) {
    const value = bundle.formatPattern(pattern, null, errors);
    console.log(`${name}: ${value}`);
  }
}
```

### Safe Utility Functions

```typescript
import { FluentBundle, FluentVariable } from "@fluent/bundle";

function formatAttribute(
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
  const result = bundle.formatPattern(pattern, args ?? null, errors);
  if (errors.length > 0) {
    console.warn(`Errors formatting ${id}.${attr}:`, errors);
  }
  return result;
}

// Usage:
const placeholder = formatAttribute(bundle, "search-input", "placeholder");
// -> "Search..." or null if missing
```

---

## React -- Attribute Mapping

### Basic Attribute Mapping

```tsx
import { Localized } from "@fluent/react";

function SearchBox() {
  return (
    <Localized
      id="search-input"
      attrs={{ placeholder: true, "aria-label": true }}
    >
      <input
        type="search"
        placeholder="Search..."
        aria-label="Search the site"
      />
    </Localized>
  );
}
```

```ftl
search-input =
    .placeholder = Search...
    .aria-label = Search the site
```

### Compound Message in React

```tsx
function SubmitButton() {
  return (
    <Localized
      id="submit-button"
      attrs={{ title: true, "aria-label": true }}
    >
      <button
        title="Click to submit"
        aria-label="Submit form"
      >
        Submit
      </button>
    </Localized>
  );
}
```

```ftl
submit-button = Submit
    .title = Click to submit your form
    .aria-label = Submit the registration form
```

The value ("Submit") replaces the button's text content. The `.title` and `.aria-label` attributes replace the corresponding HTML attributes.

### Attributes with Variables in React

```tsx
function DownloadLink({ fileName, fileSize }: Props) {
  return (
    <Localized
      id="download-link"
      attrs={{ title: true }}
      vars={{ fileName, fileSize }}
    >
      <a href={`/files/${fileName}`} title={`Download ${fileName}`}>
        {`Download ${fileName}`}
      </a>
    </Localized>
  );
}
```

```ftl
download-link = Download { $fileName }
    .title = Click to download { $fileName } ({ $fileSize } MB)
```

### Image with alt Text

```tsx
function ProductImage({ productName }: Props) {
  return (
    <Localized
      id="product-image"
      attrs={{ alt: true }}
      vars={{ productName }}
    >
      <img src="/product.jpg" alt={`Photo of ${productName}`} />
    </Localized>
  );
}
```

```ftl
product-image =
    .alt = Photo of { $productName }
```

This is a value-less message -- the image has no text content, only an `alt` attribute.

### Imperative Attribute Access in React

```tsx
import { useLocalization } from "@fluent/react";

function DynamicForm() {
  const { l10n } = useLocalization();

  // getString only formats the value, not attributes.
  // For attributes, access the bundle directly:
  const bundle = l10n.getBundle("form-email");
  let placeholder = "Email";
  if (bundle) {
    const msg = bundle.getMessage("form-email");
    if (msg) {
      const pattern = msg.attributes["placeholder"];
      if (pattern) {
        placeholder = bundle.formatPattern(pattern);
      }
    }
  }

  return <input type="email" placeholder={placeholder} />;
}
```

```ftl
form-email =
    .placeholder = Enter your email address
    .aria-label = Email input
```
