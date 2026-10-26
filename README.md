![Build](https://img.shields.io/github/actions/workflow/status/oxlip-lang/oal/ci.yml?branch=master)
[![License](https://img.shields.io/badge/license-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

# The Oxlip API Language
Oxlip is a high-level functional programming language for designing [OpenAPI](https://www.openapis.org/what-is-openapi) definitions.
As an [Interface Description Language](https://en.wikipedia.org/wiki/Interface_description_language), it is not general purpose.
The motivation is to alleviate the pain of managing OpenAPI in JSON or YAML by hand and at scale.
Oxlip defines algebraic abstractions over [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) concepts, not too dissimilar to [Sass/SCSS over CSS](https://sass-lang.com/).

There are pros and cons to both _API-design-first_ and OpenAPI generated from implementation.
As OpenAPI is better produced or consumed by machines rather than humans, Oxlip tries to help _API-design-first_ teams with better tooling.

## [Documentation](https://www.oxlip-lang.org/)

## [Playground](https://oxlip-lang.github.io/oxlip-playground)

## Installation
This step requires a [local Rust and Cargo installation](https://doc.rust-lang.org/cargo/getting-started/installation.html).

```
make install
```
Optional: a [VSCode language extension](https://github.com/oxlip-lang/oal-vscode) is available for syntax highlighting and IDE capabilities.

## Usage
```
    oal-cli [OPTIONS]

OPTIONS:
    -b, --base <BASE>        The relative URL to a base OpenAPI description
    -c, --conf <CONFIG>      The path to the configuration file
    -h, --help               Print help information
    -m, --main <MAIN>        The relative URL to the main program
    -t, --target <TARGET>    The relative URL to the target OpenAPI description
```

### Property names

Bare names such as `'user_id` are unchanged. Use an apostrophe followed by a
JSON string for names containing dots, spaces, Unicode or other characters:

```oal
res /users on get -> {
  '"user.name"! str,
  '"display name" str,
  '"quote\" and backslash\\" str
};
```

JSON escapes (including Unicode surrogate pairs) are decoded; malformed escapes
are reported as lexical errors. The required/optional suffix (`!`/`?`) follows
the closing quote. Quoted names work in object properties, query/path parameters
and headers, subject to the naming rules of the target protocol. This does not
change OAL string literal or annotation syntax.

### Compiling the example program
```
oal-cli --conf examples/oal.toml
```

### Media-type examples

The `examples` annotation maps names to external URLs or explicit inline values:

```oal
res /items on post : <{}> `examples: {request: {value: {name: "sample"}}}`
    -> <status=200, {}> `examples: {inline: {value: {id: 1}}, remote: "examples/item.json"}`;
```

A URL string retains the existing shorthand and produces OpenAPI `externalValue`.
A mapping containing only `value` produces OpenAPI `value`, preserving objects,
arrays, strings, numbers, booleans and explicit `null`. To embed a URL as a string
value, write `{value: "https://example.test/item"}` rather than the URL shorthand.
Example names are strings; quote names containing whitespace or punctuation,
and names such as `"422"`, `"true"` or `"null"` that YAML would otherwise parse
as non-string values. Previously ignored malformed entries now produce errors.
Within backtick-delimited annotations, encode a literal backtick in a quoted
string as `\u0060`; use `\n` for an embedded newline.

Examples may annotate a schema or a content expression. Content-level examples
take precedence; when absent, the schema's examples are used. An explicit empty
mapping on the content suppresses that fallback. Malformed entries, YAML tags,
non-finite numbers and non-string object keys produce diagnostics instead of
silently losing example data. Inline values must be representable as JSON.

## Mixed URI Segments

URI segments can combine literal text and typed parameters:

```oal
res /api/db/{'dataset str}.{'table str} on get -> {};
res /v{'version int}/files/{'name str}.json on get -> {};

let id = 'id int;
res /items/item-{id} on get -> {};
```

These produce `/api/db/{dataset}.{table}`, `/v{version}/files/{name}.json`
and `/items/item-{id}` without post-processing. Every parameter still has its
own required OpenAPI path-parameter declaration and primitive type. Literal
prefixes/suffixes, multiple parameters, URI `concat`, query parameters, URI
schema examples and generated operation IDs use the same segment structure.

Fragments within a segment must be adjacent; whitespace and comments are still
allowed inside parameter braces. Existing slash-separated syntax, including
whitespace between `/` and its parameter braces, is retained. Literal fragments
use ASCII letters/digits, `_`, `-`, `.`, and percent-encoded data, which is
preserved verbatim. Encode reserved literal braces as `%7B`/`%7D` and Unicode
as UTF-8 percent escapes; the compiler does not decode or validate those escapes.
After a parameter, `~` and `->` retain their OAL operator meaning, and the token
`on` still begins a relation even without whitespace. Use `%7E`, `%2D%3E` or
`%6Fn` if those sequences are intended as literal suffixes. A literal `/` starts
another segment; `%2F` remains part of the current segment.

### YAML output

The CLI and WASM compiler use the same YAML emitter. Unicode NEL (U+0085),
LINE SEPARATOR (U+2028) and PARAGRAPH SEPARATOR (U+2029) are escaped in keys
and values so YAML 1.1 and 1.2 readers do not fold them or add indentation to
their contents. Multiline strings are quoted without automatic line wrapping.
This changes presentation, not the OpenAPI data model or compilation interfaces.
OAL annotations and CLI base files still use the existing parser.

## Experimental: WebAssembly support
Release to WebAssembly requires the installation of [`wasm-pack`](https://rustwasm.github.io/wasm-pack/installer/).

```
make wasm
```

## Operation Security

Declare OpenAPI security requirements on a transfer:

```oal
# security: [{api_key: []}, {oauth: [read, write]}]
let protected = get -> {};
res /items on protected;

# security: []
let public = get -> {};
res /health on public;
```

Requirements in the array are alternatives (OR). Schemes within one requirement
are all required (AND), for example `security: [{api_key: [], oauth: [read]}]`.
An empty requirement permits anonymous access: `security: [{}, {oauth: [read]}]`.
Omitting the annotation inherits document-level security; `security: []`
explicitly disables it for the operation. A later security annotation replaces
earlier requirements rather than concatenating alternatives, including when
the later annotation is empty.

Scheme names and scopes must be strings, with a scope array for every scheme.
Malformed values produce diagnostics; YAML-only tags are rejected. Use empty
scope arrays for schemes without OAuth/OpenID scopes. Scheme definitions remain
in `components.securitySchemes` of the CLI's base OpenAPI document; this feature
does not add scheme declarations to OAL or a base-document argument to WASM.
References and scopes are emitted as supplied, not checked against that base.
These annotations describe authentication requirements; they do not enforce it.

### Base document input

The historic JavaScript `compile(oalText)` function is unchanged. A separate
WASM entry point accepts two JSON-compatible objects:

```js
const result = compileDocument(
  { source: "res /items on get -> {};" },
  {
    openapi: "3.0.3",
    info: { title: "Items API", version: "1.0.0" },
    paths: {},
    components: {
      securitySchemes: {
        bearer: { type: "http", scheme: "bearer" }
      }
    },
    security: [{ bearer: [] }]
  }
);
try {
  if (result.error) throw new Error(result.error);
  console.log(result.api);
} finally {
  result.free();
}
```

Both arguments are required. The first must contain only the string `source`;
the second is a full OpenAPI 3.0 base document, not a partial patch or YAML text.
The required fields include `openapi`, `info.title`, `info.version`, and `paths`.
Malformed objects or OAL return the existing `{api: "", error: "..."}` result.
Inputs are serialized with JSON semantics and are not modified. Cyclic objects
and serialization errors produce diagnostics. Use ordinary JSON-compatible data,
not functions, BigInts, custom getters or `toJSON` implementations.

The existing CLI builder defines ownership: generated `paths` and
`components.schemas` **replace** those entire base collections. Other metadata,
document security, non-schema components and supported `x-` extensions are
retained through the typed OpenAPI model. This is not a deep merge and does not
preserve arbitrary unknown fields or original formatting; normal serialization
defaults may appear. A supplied base does not inherit the default title/server.
Reference resolution and complete OpenAPI semantic validation are not performed.

Rust callers retain `oal_wasm::compile(input: &str)` and can also call
`oal_wasm::compile_with_document(input: &str, base: oal_wasm::OpenAPI)`.
The CLI and its `--base` interface are unchanged. The new WASM entry point uses
the same builder without appending YAML fragments. Always deploy the generated
JavaScript bindings together with the matching WASM binary.
