# Askama Reference

Compile-time template engine for Rust based on Jinja syntax. Templates are parsed and
validated at compile time, producing type-safe Rust code with zero runtime overhead.

Latest version: **0.15.x** (as of late 2025).

---

## 1. Installation

### Core Crate

```toml
[dependencies]
askama = "0.15"
```

### With Web Framework Integration (askama_web)

The old per-framework crates (`askama_axum`, `askama_actix`, `askama_rocket`) are
deprecated since askama 0.13. Use `askama_web` instead:

```toml
[dependencies]
askama = "0.15"
askama_web = { version = "0.15", features = ["axum-0.8"] }
```

Available framework features for `askama_web`:

| Feature          | Framework      | Trait Implemented |
|------------------|----------------|-------------------|
| `axum-0.8`       | Axum 0.8       | `IntoResponse`    |
| `axum-0.7`       | Axum 0.7       | `IntoResponse`    |
| `actix-web-4`    | Actix-web 4    | `Responder`       |
| `rocket-0.5`     | Rocket 0.5     | `Responder`       |
| `warp-0.4`       | Warp 0.4       | `Reply`           |
| `warp-0.3`       | Warp 0.3       | `Reply`           |
| `poem-3`         | Poem 3         | `IntoResponse`    |
| `trillium-0.2`   | Trillium 0.2   | `Handler`         |

Logging features: `eprintln`, `log-0.4`, `tracing-0.1`.

### Optional askama Features

```toml
[dependencies]
askama = { version = "0.15", features = ["serde_json"] }
```

- `serde_json` -- enables the `json` / `tojson` filter
- `alloc` -- `render()` returns `String` (enabled by default)
- `std` -- `write_into()` for `io::Write` (enabled by default)
- `blocks` -- block fragment rendering
- `code-in-doc` -- `in_doc = true` for doc-embedded templates
- `config` -- custom `askama.toml` configuration

---

## 2. Setup

### Template Directory

By default, askama looks for templates in `templates/` relative to your crate root
(where `Cargo.toml` lives).

```
my-crate/
  Cargo.toml
  askama.toml          # optional config
  templates/
    base.html
    index.html
    partials/
      header.html
  src/
    main.rs
```

### askama.toml Configuration

Place `askama.toml` in the crate root. All settings are optional.

```toml
[general]
# Template directories (relative to crate root). Default: ["templates"]
dirs = ["templates"]

# Global whitespace handling: "preserve" (default), "suppress", "minimize"
whitespace = "preserve"

# Default syntax name (if custom syntaxes defined)
# default_syntax = "custom"

# Custom syntax delimiters
[[syntax]]
name = "custom"
block_start = "{%"
block_end = "%}"
comment_start = "{#"
comment_end = "#}"
expr_start = "{{"
expr_end = "}}"

# Custom escapers
[[escaper]]
path = "askama::filters::Html"
extensions = ["html", "htm", "xml"]

[[escaper]]
path = "askama::filters::Text"
extensions = ["txt", "md", "yml"]
```

### Default Escapers

| Extensions                            | Escaper              |
|---------------------------------------|----------------------|
| `html`, `htm`, `xml`, `j2`, `jinja`  | HTML auto-escaping   |
| `txt`, `md`, `yml`, `none`, `""`     | No escaping (text)   |

---

## 3. Template Syntax Overview

Templates consist of three element types:

| Syntax       | Purpose              | Example                        |
|--------------|----------------------|--------------------------------|
| `{{ ... }}`  | Expression output    | `{{ user.name }}`              |
| `{# ... #}`  | Comments             | `{# this is ignored #}`       |
| `{% ... %}`  | Tags / control flow  | `{% if logged_in %}`           |

### Raw Blocks

Disable template processing for a section:

```
{% raw %}
  {{ this is not evaluated }}
  {% neither is this %}
{% endraw %}
```

---

## 4. Expressions

### Variables

Top-level variables come from struct fields. Access nested fields with dot notation:

```
{{ user.name }}
{{ user.address.city }}
```

### Method Calls

```
{{ name.len() }}
{{ items.is_empty() }}
{{ value.to_string() }}
```

### Operators

| Category    | Operators                              |
|-------------|----------------------------------------|
| Arithmetic  | `+`, `-`, `*`, `/`, `%`               |
| Comparison  | `==`, `!=`, `<`, `>`, `<=`, `>=`      |
| Logical     | `&&`, `\|\|`, `!`                      |
| Bitwise     | `bitand`, `bitor`, `xor` (named)       |
| String cat  | `~` (spaces required: `a ~ b`)        |
| Cast        | `as` (primitive types: `val as i32`)   |
| Reference   | `&val`, `*val`                         |
| Error prop  | `?` (on `Result` types only)          |

### Literals

- Strings: `"hello"`
- Integers: `42`, `-5`
- Booleans: `true`, `false`
- Struct creation: `{{ MyStruct { field: value } }}`

### Filters

Pipe syntax, chainable:

```
{{ name | lower }}
{{ name | lower | truncate(20) }}
{{ items | join(", ") }}
```

### Constants and Paths

```
{{ crate::MY_CONSTANT }}
{{ self::helper_function() }}
{{ crate::module::function(arg) }}
```

---

## 5. Control Flow

### If / Else If / Else

```
{% if users.len() == 0 %}
  No users
{% else if users.len() == 1 %}
  1 user
{% elif users.len() == 2 %}
  2 users
{% else %}
  {{ users.len() }} users
{% endif %}
```

`else if` and `elif` are interchangeable.

### If-Let Pattern Matching

```
{% if let Some(user) = current_user %}
  Hello, {{ user.name }}
{% else %}
  Not logged in
{% endif %}
```

### Existence Checks

```
{% if x is defined %}
  x exists: {{ x }}
{% endif %}

{% if y is not defined %}
  y is undefined
{% endif %}
```

### For Loops

```
{% for user in users %}
  <li>{{ user.name }}</li>
{% endfor %}
```

With else block (when collection is empty):

```
{% for item in items %}
  <li>{{ item }}</li>
{% else %}
  <li>No items found</li>
{% endfor %}
```

#### Loop Variables

Inside a `{% for %}` block, the `loop` object is available:

| Variable       | Type   | Description                     |
|----------------|--------|---------------------------------|
| `loop.index`   | `usize`| 1-based iteration count         |
| `loop.index0`  | `usize`| 0-based iteration count         |
| `loop.first`   | `bool` | `true` on first iteration       |
| `loop.last`    | `bool` | `true` on last iteration        |

```
{% for user in users %}
  {% if loop.first %}
    <li>First: {{ user.name }}</li>
  {% else %}
    <li>User #{{ loop.index }}: {{ user.name }}</li>
  {% endif %}
{% endfor %}
```

#### Loop Control

```
{% for item in items %}
  {% if item.hidden %}
    {% continue %}
  {% endif %}
  {% if loop.index > 10 %}
    {% break %}
  {% endif %}
  {{ item.name }}
{% endfor %}
```

### Match

Type-safe pattern matching (mirrors Rust `match`):

```
{% match result %}
  {% when Ok(val) %}
    Success: {{ val }}
  {% when Err(e) %}
    Error: {{ e }}
{% endmatch %}
```

Option handling:

```
{% match item %}
  {% when Some with ("foo") %}
    Found literal foo
  {% when Some with (val) %}
    Found {{ val }}
  {% when None %}
    Nothing
{% endmatch %}
```

Multiple patterns and wildcards:

```
{% match code %}
  {% when 1 | 4 | 86 %}
    Special code
  {% when _ %}
    Other code
{% endmatch %}
```

Struct patterns:

```
{% match point %}
  {% when Point { x, y: 0 } %}
    On x-axis at {{ x }}
  {% when Point { x, y } %}
    At ({{ x }}, {{ y }})
{% endmatch %}
```

### Variable Assignment

```
{% let name = user.name %}
{% let len = name.len() %}
{% let mut counter = 0 %}
{% set name = "alternate" %}     {# Jinja-compatible alias for let #}
```

Scoped assignment:

```
{% let val -%}
{% if condition -%}
  {% let val = "yes" -%}
{% else -%}
  {% let val = "no" -%}
{% endif -%}
{{ val }}
```

---

## 6. Built-in Filters

### String Filters

| Filter                     | Description                                 | Example                          |
|----------------------------|---------------------------------------------|----------------------------------|
| `lower` / `lowercase`     | Lowercase                                   | `{{ name \| lower }}`            |
| `upper` / `uppercase`     | Uppercase                                   | `{{ name \| upper }}`            |
| `capitalize`               | First char upper, rest lower                | `{{ name \| capitalize }}`       |
| `title` / `titlecase`     | Capitalize each word                        | `{{ s \| title }}`               |
| `trim`                     | Strip leading/trailing whitespace           | `{{ s \| trim }}`                |
| `truncate(len)`            | Limit length, append "..." if truncated     | `{{ s \| truncate(20) }}`        |
| `center(width)`            | Center in field of given width              | `{{ s \| center(40) }}`         |
| `indent(n)`                | Indent newlines with n spaces               | `{{ s \| indent(4) }}`          |
| `wordcount`                | Count words                                 | `{{ s \| wordcount }}`          |
| `pluralize(s, p)`          | Singular/plural form based on integer       | `{{ count \| pluralize("item", "items") }}` |

### HTML Filters

| Filter                  | Description                                    |
|-------------------------|------------------------------------------------|
| `escape` / `e`         | Escape HTML chars (`<`, `>`, `&`, `"`, `'`)    |
| `safe`                  | Mark as safe, skip auto-escaping               |
| `linebreaks`            | Newlines to `<br>`, double newlines to `<p>`   |
| `linebreaksbr`          | All newlines to `<br>`                         |
| `paragraphbreaks`       | Double newlines to `<p>` tags only             |

### Collection Filters

| Filter              | Description                                    | Example                           |
|---------------------|------------------------------------------------|-----------------------------------|
| `join(sep)`         | Join iterable with separator                   | `{{ items \| join(", ") }}`       |
| `unique`            | Remove duplicates (requires `std`)             | `{{ items \| unique }}`           |
| `reject(val)`       | Filter out matching values                     | `{{ items \| reject("") }}`       |

### Format Filters

| Filter                    | Description                                   | Example                                  |
|---------------------------|-----------------------------------------------|------------------------------------------|
| `format(fmt)`             | Rust `format!` syntax                         | `{{ val \| format("{:.2}") }}`            |
| `fmt(fmt)`                | Same as format, supports chaining             | `{{ val \| fmt("{:#x}") }}`              |
| `filesizeformat`          | Bytes to human-readable (KB, MB, etc.)        | `{{ bytes \| filesizeformat }}`          |

### Encoding Filters

| Filter                  | Description                                    |
|-------------------------|------------------------------------------------|
| `urlencode`             | Percent-encode (preserves `/`)                 |
| `urlencode_strict`      | Percent-encode (also encodes `/`)              |

### JSON Filter (feature-gated)

Requires `features = ["serde_json"]`:

```
{{ data | json }}
{{ data | tojson }}
{{ data | json(2) }}    {# pretty-print with 2-space indent #}
```

### Utility Filters

| Filter             | Description                                 |
|--------------------|---------------------------------------------|
| `default(val)`     | Fallback for undefined/unassigned values    |
| `defined_or(val)`  | Fallback if identifier is undefined         |
| `assigned_or(val)` | Fallback if value is default state          |
| `deref`            | Dereference the argument                    |
| `ref`              | Create reference to argument                |

### Filter Blocks

Apply a filter to an entire block:

```
{% filter lower | capitalize %}
  {{ some_content }}
{% endfilter %}
```

---

## 7. Custom Filters

Define a `filters` module in the same scope as your template struct. The module must
be named `filters` exactly.

### Basic Custom Filter

```rust
use askama::Template;

#[derive(Template)]
#[template(source = "{{ value | double }}", ext = "txt")]
struct MyTemplate {
    value: i32,
}

mod filters {
    pub fn double<T: std::fmt::Display>(
        value: T,
        _: &dyn askama::Values,
    ) -> askama::Result<String> {
        let n: i32 = value.to_string().parse().unwrap();
        Ok((n * 2).to_string())
    }
}
```

### Custom Filter with Extra Arguments

```rust
use askama::Template;

#[derive(Template)]
#[template(source = "{{ s | myfilter(4) }}", ext = "txt")]
struct MyFilterTemplate<'a> {
    s: &'a str,
}

mod filters {
    pub fn myfilter<T: std::fmt::Display>(
        s: T,
        _: &dyn askama::Values,
        n: usize,
    ) -> askama::Result<String> {
        let s = s.to_string();
        let mut replace = String::with_capacity(n);
        replace.extend((0..n).map(|_| "a"));
        Ok(s.replace("oo", &replace))
    }
}

// {{ "foo" | myfilter(4) }}  =>  "faaaa"
```

**Signature rules:**
1. First parameter: the piped value (generic over `Display`)
2. Second parameter: `&dyn askama::Values` (runtime values store)
3. Additional parameters: extra arguments from the template call
4. Returns `askama::Result<String>`

---

## 8. Template Inheritance

### Base Template (`templates/base.html`)

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>{% block title %}{{ title }} - My Site{% endblock %}</title>
    {% block head %}{% endblock %}
  </head>
  <body>
    <div id="content">
      {% block content %}<p>Placeholder</p>{% endblock %}
    </div>
  </body>
</html>
```

### Child Template (`templates/index.html`)

```html
{% extends "base.html" %}

{% block title %}Index{% endblock %}

{% block head %}
  <style>.content { color: blue; }</style>
{% endblock %}

{% block content %}
  <h1>Index</h1>
  <p>Hello, world!</p>
  {% call super() %}
{% endblock %}
```

**Rules:**
- `{% extends "..." %}` must be the first tag in the template
- Blocks defined in the base can be overridden by child templates
- `{% call super() %}` renders the parent block's content
- `{% endblock %}` can optionally include the block name: `{% endblock content %}`
- Multiple levels of inheritance are supported

---

## 9. Includes

Insert another template inline:

```
{% include "partials/header.html" %}

<main>
  {{ content }}
</main>

{% include "partials/footer.html" %}
```

The included template has access to all variables in the current scope.

---

## 10. Macros

### Definition

```
{% macro heading(text, level = "1") %}
  <h{{ level }}>{{ text }}</h{{ level }}>
{% endmacro %}
```

Default arguments are supported.

### Invocation

Positional:
```
{% call heading("Welcome") %}
{% call heading("Subtitle", "2") %}
```

Named:
```
{% call heading(level = "3", text = "Section") %}
```

### Call Blocks (Macros with Body)

```
{% macro card(title) %}
  <div class="card">
    <h2>{{ title }}</h2>
    <div class="body">{{ caller() }}</div>
  </div>
{% endmacro %}

{% call card("My Card") %}
  <p>This is the card body content.</p>
{% endcall %}
```

`{{ caller() }}` renders the body passed between `{% call %}` and `{% endcall %}`.

### Call Blocks with Arguments

```
{% call(item) list_items(items) %}
  <li>{{ item.name }}</li>
{% endcall %}
```

### Importing Macros

```
{% import "macros.html" as m %}

{{ m::heading("Hello") }}
```

---

## 11. Whitespace Control

Three whitespace control markers can be placed on either side of any tag:

| Marker | Behavior                                         |
|--------|--------------------------------------------------|
| `-`    | **Suppress** -- remove all adjacent whitespace   |
| `+`    | **Preserve** -- keep one whitespace character     |
| `~`    | **Minimize** -- collapse to single space/newline  |

### Examples

```
{{- expr -}}          {# suppress whitespace on both sides #}
{%- if cond -%}       {# suppress around tag #}
{#- comment -#}       {# suppress around comment #}

{{+ expr +}}          {# preserve whitespace #}
{%~ if cond ~%}       {# minimize whitespace #}
```

Mixed example:

```
{% if foo %}
  {{- bar -}}
{% else if -%}
  nothing
{%- endif %}
```

### Priority

When markers conflict (inner `-` vs outer `+`): suppress > minimize > preserve.

### Global Whitespace Setting

Per-template:
```rust
#[derive(Template)]
#[template(path = "page.html", whitespace = "suppress")]
struct Page;
```

In `askama.toml`:
```toml
[general]
whitespace = "suppress"
```

Modes:
- `"preserve"` (default) -- use `-` to suppress
- `"suppress"` -- use `+` to preserve
- `"minimize"` -- collapse whitespace, use `~`

---

## 12. Escaping

### Auto-Escaping

Enabled by default for `.html`, `.htm`, `.xml` extensions. Escapes: `<`, `>`, `&`,
`"`, `'`.

```
{{ user_input }}           {# auto-escaped in .html templates #}
```

### Disable for Single Expression

```
{{ trusted_html | safe }}
```

### Force Escaping

```
{{ value | escape }}
{{ value | e }}
```

### Override Escape Mode

```rust
#[derive(Template)]
#[template(path = "page.html", escape = "none")]  // disable
struct Page;

#[derive(Template)]
#[template(path = "data.txt", escape = "html")]   // force enable
struct Data;
```

### Raw Blocks

```
{% raw %}
  This {{ is not }} evaluated.
  {% tags are %} literal text.
{% endraw %}
```

---

## 13. Struct Integration

### Basic Template Struct

```rust
use askama::Template;

#[derive(Template)]
#[template(path = "hello.html")]
struct HelloTemplate<'a> {
    name: &'a str,
}
```

Field names must match template variable names.

### `#[template()]` Attribute Options

| Attribute    | Description                                              |
|--------------|----------------------------------------------------------|
| `path`       | Template file path relative to `templates/` dir          |
| `source`     | Inline template string (mutually exclusive with `path`)  |
| `ext`        | File extension (required with `source`, for escape mode) |
| `escape`     | Override escaper: `"html"`, `"none"`, or custom path     |
| `block`      | Render only a named block (for fragment rendering)       |
| `blocks`     | Generate sub-templates for all blocks                    |
| `whitespace` | Per-template whitespace mode                             |
| `syntax`     | Custom syntax name (from `askama.toml`)                  |
| `config`     | Path to config file                                      |
| `print`      | Debug: `"ast"`, `"code"`, `"all"`                        |
| `in_doc`     | Enable template in doc comments (`true`/`false`)         |

### Inline Source

```rust
#[derive(Template)]
#[template(source = "Hello {{ name }}", ext = "txt")]
struct Greeting {
    name: String,
}
```

### Template Trait Methods

```rust
let tmpl = HelloTemplate { name: "World" };

// Render to new String (best performance)
let html: String = tmpl.render().unwrap();

// Render into fmt::Write (e.g., existing String)
let mut buf = String::new();
tmpl.render_into(&mut buf).unwrap();

// Write into io::Write (e.g., Vec<u8>, File)
let mut bytes = Vec::new();
tmpl.write_into(&mut bytes).unwrap();

// Associated constants
let ext = HelloTemplate::EXTENSION;    // e.g., Some("html")
let mime = HelloTemplate::MIME_TYPE;   // e.g., "text/html; charset=utf-8"
```

**Performance note:** Prefer `render()` over `to_string()`. The `Display` impl
(`to_string()` / `format!()`) is 100-200% slower than `render()`.

---

## 14. Axum Integration

### Option A: askama_web (Recommended)

```toml
[dependencies]
askama = "0.15"
askama_web = { version = "0.15", features = ["axum-0.8"] }
axum = "0.8"
tokio = { version = "1", features = ["full"] }
```

```rust
use askama::Template;
use askama_web::WebTemplate;
use axum::{Router, routing::get};

#[derive(Template, WebTemplate)]
#[template(path = "index.html")]
struct IndexTemplate {
    title: String,
}

async fn index() -> IndexTemplate {
    IndexTemplate {
        title: "Home".to_string(),
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(index));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

`WebTemplate` auto-implements `IntoResponse`. Returns 200 with
`Content-Type: text/html; charset=utf-8`. On render error, returns 500.

### Option B: Manual IntoResponse

```rust
use askama::Template;
use axum::response::{Html, IntoResponse, Response};
use axum::http::StatusCode;

#[derive(Template)]
#[template(path = "index.html")]
struct IndexTemplate {
    title: String,
}

impl IntoResponse for IndexTemplate {
    fn into_response(self) -> Response {
        match self.render() {
            Ok(html) => Html(html).into_response(),
            Err(e) => {
                eprintln!("Template render error: {e}");
                (StatusCode::INTERNAL_SERVER_ERROR, "Render error").into_response()
            }
        }
    }
}
```

### Error Handling Pattern

Define an app error type for better error responses:

```rust
use askama::Template;
use axum::response::{Html, IntoResponse, Response};
use axum::http::StatusCode;

enum AppError {
    Template(askama::Error),
    NotFound,
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, msg) = match self {
            AppError::Template(e) => {
                eprintln!("Template error: {e}");
                (StatusCode::INTERNAL_SERVER_ERROR, "Internal error")
            }
            AppError::NotFound => (StatusCode::NOT_FOUND, "Not found"),
        };
        (status, msg).into_response()
    }
}

impl From<askama::Error> for AppError {
    fn from(e: askama::Error) -> Self {
        AppError::Template(e)
    }
}

async fn page() -> Result<Html<String>, AppError> {
    let tmpl = IndexTemplate { title: "Page".into() };
    Ok(Html(tmpl.render()?))
}
```

---

## 15. Partials Pattern (htmx)

### Strategy: Separate Fragment Templates

Create small, dedicated templates for htmx partial responses:

**`templates/todos/list.html`** (full page):
```html
{% extends "base.html" %}

{% block content %}
<div id="todo-list">
  {% include "todos/_items.html" %}
</div>
<form hx-post="/todos" hx-target="#todo-list" hx-swap="innerHTML">
  <input name="title" />
  <button type="submit">Add</button>
</form>
{% endblock %}
```

**`templates/todos/_items.html`** (htmx fragment):
```html
{% for todo in todos %}
  <div class="todo-item">
    <span>{{ todo.title }}</span>
    <button hx-delete="/todos/{{ todo.id }}" hx-target="#todo-list" hx-swap="innerHTML">
      Delete
    </button>
  </div>
{% endfor %}
```

### Rust Side: Full Page vs Fragment

```rust
use askama::Template;

// Full page (extends base.html)
#[derive(Template)]
#[template(path = "todos/list.html")]
struct TodoListPage {
    todos: Vec<Todo>,
}

// Fragment only (for htmx responses)
#[derive(Template)]
#[template(path = "todos/_items.html")]
struct TodoItemsFragment {
    todos: Vec<Todo>,
}

async fn list_todos(
    headers: axum::http::HeaderMap,
) -> Result<Html<String>, AppError> {
    let todos = get_todos().await;

    // Check if htmx request
    let html = if headers.contains_key("hx-request") {
        TodoItemsFragment { todos }.render()?
    } else {
        TodoListPage { todos }.render()?
    };
    Ok(Html(html))
}
```

### Block Fragment Rendering

Askama supports rendering a single named block with the `block` attribute:

```rust
// Full page template
#[derive(Template)]
#[template(path = "page.html")]
struct FullPage {
    name: String,
    items: Vec<String>,
}

// Render only the "content" block from page.html
#[derive(Template)]
#[template(path = "page.html", block = "content")]
struct ContentFragment {
    name: String,
    items: Vec<String>,
}
```

This avoids duplicating template markup -- define blocks in the full template and
render individual blocks as fragments for htmx responses.

---

## 16. Advanced

### Enums as Templates

```rust
#[derive(Template)]
#[template(path = "area.txt")]
enum Area {
    #[template(block = "square")]
    Square(f32),
    #[template(block = "rectangle")]
    Rectangle { a: f32, b: f32 },
    #[template(block = "circle")]
    Circle { radius: f32 },
}
```

**`templates/area.txt`:**
```
{%- block square -%}
    {{ self.0 }}^2
{%- endblock -%}

{%- block rectangle -%}
    {{ a }} * {{ b }}
{%- endblock -%}

{%- block circle -%}
    pi * {{ radius }}^2
{%- endblock -%}
```

### Option Handling

Use `match`:

```
{% match user.email %}
  {% when Some with (email) %}
    <a href="mailto:{{ email }}">{{ email }}</a>
  {% when None %}
    <span>No email</span>
{% endmatch %}
```

Or `if let`:

```
{% if let Some(email) = user.email %}
  {{ email }}
{% endif %}
```

Fallback (less idiomatic):

```
{% if user.email.is_some() %}
  {{ user.email.as_ref().unwrap() }}
{% endif %}
```

### Nested Structs (Render in Place)

Inner structs that derive `Template` render automatically when used as expressions:

```rust
#[derive(Template)]
#[template(source = "Section: {{ section }}", ext = "txt")]
struct Page<'a> {
    section: SectionTemplate<'a>,
}

#[derive(Template)]
#[template(source = "A={{ a }}, B={{ b }}", ext = "txt")]
struct SectionTemplate<'a> {
    a: &'a str,
    b: &'a str,
}

// Renders: "Section: A=foo, B=bar"
```

**Important:** If the inner template produces HTML, use `{{ section | safe }}` to
prevent double-escaping. Or implement `askama::filters::HtmlSafe` for the inner type:

```rust
impl askama::filters::HtmlSafe for SectionTemplate<'_> {}
```

### References and Lifetimes

Templates commonly borrow data:

```rust
#[derive(Template)]
#[template(path = "user.html")]
struct UserTemplate<'a> {
    name: &'a str,
    items: &'a [Item],
}
```

Multiple lifetimes work as expected:

```rust
#[derive(Template)]
#[template(path = "detail.html")]
struct DetailTemplate<'a, 'b> {
    header: &'a str,
    body: &'b str,
}
```

### Generic Templates

```rust
#[derive(Template)]
#[template(source = "{{ value }}", ext = "txt")]
struct GenericTemplate<T: std::fmt::Display> {
    value: T,
}
```

### Rust Macros in Templates

Standard Rust macros can be called:

```
{% let formatted = format!("{:.2}", price) %}
{{ formatted }}
```

### Doc-Embedded Templates

With the `code-in-doc` feature, embed templates in doc comments:

```rust
#[derive(Template)]
#[template(in_doc = true, ext = "html")]
/// ```askama
/// <h1>{{ title }}</h1>
/// ```
struct DocTemplate {
    title: String,
}
```

### Debugging Generated Code

```rust
#[derive(Template)]
#[template(path = "page.html", print = "code")]  // "ast", "code", or "all"
struct DebugTemplate;
```

Compile output shows the generated Rust code for inspection.
