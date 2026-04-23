---
source: (our own — synthesized from `~/git/Mentci/CLAUDE.md` and the practiced
  style across `~/git/Mentci/components/{aski-core,corec,askic,askicc,arbor,bibliotheca,...}/src/`)
fetched: 2026-04-23
trimmed: dropped framework-specific concepts that aren't actually used in
  practice (actor frameworks, esoteric naming conventions, language-internal
  terminology); kept rules that the live code follows
---

# Rust style

The shape Rust takes in our repos. Follow it for new code; new files in
existing crates should look like the existing files.

## The rules in one sentence

**Behavior lives on types. Domain values are typed. Boundaries take and
return one object. Errors are enums you implement by hand.**

## Methods on types, not free functions

The only free function in a binary crate is `main`. Reusable behavior is
a method on a type or a trait impl. Test helpers are methods on a fixture
struct.

```rust
// Wrong
pub fn parse_item_details(json: &str, md5: &str) -> Result<ItemDetails, Error> { … }

// Right
impl ItemDetails {
    pub fn from_json(json: &str, md5: &str) -> Result<Self, Error> { … }
}
```

A small private helper inside one module is fine if it is genuinely local
(`fn hex(h: &Hash) -> String` next to a single Display impl). Anything
that smells reusable becomes a method.

## Domain values are types, not primitives

If a value has identity beyond its bits, it gets a newtype. A content
hash is not a `String`. A node name is not a `String`. A file path used
as an identifier is not a `Path`.

```rust
// Wrong
pub fn details(&self, md5: &str) -> Result<ItemDetails, Error> { … }

// Right
pub struct Md5([u8; 16]);
pub fn details(&self, md5: &Md5) -> Result<ItemDetails, Error> { … }
```

When you have many name-shaped newtypes, write a one-arm macro for them.
Pattern from `aski-core/src/lib.rs:22`:

```rust
macro_rules! name_newtype {
    ($name:ident) => {
        #[derive(Archive, Serialize, Deserialize, Debug, Clone, PartialEq, Eq, Hash)]
        pub struct $name(pub String);
    };
}
name_newtype!(ModuleName);
name_newtype!(StructName);
name_newtype!(FieldName);   // distinct from ModuleName, StructName
```

## One object in, one object out

Method signatures take at most one explicit object argument and return
exactly one object. When inputs or outputs need more, define a struct.
Naked tuples are not return types.

```rust
// Wrong — multiple primitives at the boundary
fn download_url(&self, md5: &str, path_index: Option<u32>,
                domain_index: Option<u32>) -> Result<DownloadInfo, Error> { … }

// Wrong — naked tuple return
fn parse_results(html: &str) -> Result<(Vec<SearchResult>, bool), Error> { … }

// Right — typed in, typed out
struct DownloadRequest { md5: Md5, path_index: Option<u32>, domain_index: Option<u32> }
fn download_url(&self, request: DownloadRequest) -> Result<DownloadInfo, Error> { … }

// Right — the type knows how to construct itself
impl SearchResponse {
    pub fn from_html(html: &str, page: u32) -> Result<Self, Error> { … }
}
```

`&self` doesn't count as the object argument; it's the receiver.

## Constructors are associated functions

`new`, `with_*`, `from_*`, `build` — never module-level free functions.

| Name           | Use when                                                       |
|----------------|----------------------------------------------------------------|
| `new`          | default / minimal construction.                                |
| `with_<thing>` | ergonomic alt with one extra knob (`Tree::with_bits`).         |
| `from_<src>`   | conversion from a specific source type or representation.      |
| `from_input`   | conversion from a typed input struct (single-object-in style). |
| `build`        | multi-step construction with clearly-named primitive args.     |
| `Default`      | when "empty / zero" is meaningful for the type.                |
| `From<T>`      | infallible conversion from another type.                       |
| `TryFrom<T>`   | fallible conversion. Pair with `Error` enum.                   |

We don't have a settled choice between `from_*` returning `Result` and
implementing `TryFrom`. Prefer `TryFrom` when the conversion has one
canonical source type; prefer `from_<src>(…) -> Result<Self, Error>`
when there are several plausible sources or extra args.

## Use existing trait domains

If `core::str::FromStr` already names what you do, implement `FromStr`,
not an inherent `parse` method. Same for `Display`, `From`, `TryFrom`,
`AsRef`, `Default`, `Iterator`. Don't reach for an inherent method just
because it's quicker.

```rust
use core::str::FromStr;

impl FromStr for Message {
    type Err = MessageParseError;
    fn from_str(input: &str) -> Result<Self, Self::Err> { … }
}
```

Inherent methods that bypass an obvious trait domain are a smell.

## Direction-encoded names

Prefer `from_*`, `to_*`, `into_*`, `as_*`. Avoid `read`, `write`, `load`,
`save` when a direction word already conveys the meaning. `as_str` over
`get_string`. `to_hex` over `format_hex`. `from_bytes` over
`parse_bytes`.

`get` / `put` are fine for storage interfaces (`ChunkStore::get`); they
name the storage operation, not a conversion.

## Errors: enum + manual impls, no thiserror, no anyhow

Each crate defines its own `Error` enum in `src/error.rs`. Variants are
structured (carry the data needed to render a useful message).
`fmt::Display` and `std::error::Error` are implemented by hand. `From`
impls handle conversions from foreign error types.

```rust
// arbor/src/error.rs (pattern)
#[derive(Debug, Clone)]
pub enum Error {
    ChunkNotFound(Hash),
    DeserializationFailed(String),
    InvalidNode(String),
    MergeConflict { key: Vec<u8> },
}

impl fmt::Display for Error {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Error::ChunkNotFound(h)        => write!(f, "chunk not found: {}", hex(h)),
            Error::DeserializationFailed(m) => write!(f, "deserialization failed: {m}"),
            Error::InvalidNode(m)           => write!(f, "invalid node: {m}"),
            Error::MergeConflict { key }    => write!(f, "merge conflict on key ({} bytes)", key.len()),
        }
    }
}
impl std::error::Error for Error {}

// When wrapping foreign errors:
impl From<reqwest::Error> for Error {
    fn from(e: reqwest::Error) -> Self { Error::Network(e) }
}
```

Public APIs return `Result<T, Error>` with the crate's own enum.
**Never** `Result<T, Box<dyn Error>>` and **never** `anyhow::Result`.

`Result<T, String>` is acceptable only inside a bootstrap tool with no
dependencies (corec). Don't reach for it elsewhere — if you find
yourself wanting it, write the enum.

## Module layout

One concern per file. Typical crate:

```
src/
├── lib.rs        # re-exports + crate-level doc (//!)
├── error.rs      # Error enum + impls
├── types.rs      # domain newtypes + small structs
├── <thing>.rs    # one file per major type / subsystem (client.rs, store.rs, …)
└── main.rs       # only if the crate is a binary; only free fn lives here
```

Impls live in the same file as the type they're for. We don't split
types and impls across files (no `tree.rs` + `tree_impl.rs`).

## Cargo.toml

- `edition = "2021"`. (No 2024 edition use yet in core crates.)
- No workspace root. Each repo is standalone; meta-repos use symlinks
  / submodules, not Cargo workspaces.
- Default deps for serialization: `rkyv` (for binary contracts inside
  the aski pipeline), `serde` + `serde_json` (only at JSON boundaries
  where consumers need it).
- `tokio` only where async I/O actually matters (HTTP). Don't introduce
  it speculatively. `#[tokio::main]` is fine in a binary that needs it.
- Forbidden by convention: `thiserror`, `anyhow`, `eyre`. We write
  errors by hand.

## Documentation

Doc comments are impersonal, timeless, precise. Document the why and
the contract; don't restate the signature.

```rust
/// Build a balanced prolly tree from sorted key-value pairs.
///
/// Bits controls fanout: 2^bits children per chunk. Larger bits → flatter
/// tree, fewer rewrites on update, more bytes per chunk.
pub fn build<I>(store: S, bits: u32, sorted: I) -> Result<Self, Error> where … { … }
```

Module-level docs go in `//!` at the top of `lib.rs` (crate-level) or
`///` at the top of a single-purpose module file. Skip docs on obvious
boilerplate: getters, `From` impls, internal helpers.

No examples in doc comments unless the API is non-obvious. No personal
voice ("we do X"). No tense ("will return", "is going to"). Present
indicative only.

## Anti-patterns called out by name

- **Hand-maintained enum-shaped lists.** If a list of names lives in
  source as `[("U8", 0), ("U16", 0), …]`, you've smuggled data into
  code. Source it from a data file or generate it. (Acknowledged
  violation: `aski-core/src/lib.rs:81-105` — the `Primitive::all()`
  list.)
- **Naked tuple returns.** Always wrap in a named struct.
- **`Result<T, Box<dyn Error>>`.** Define the enum.
- **`anyhow::Error`, `thiserror::Error`.** Same — define the enum.
- **Inherent methods that shadow trait domains.** If `FromStr` fits,
  implement `FromStr`.
- **Free functions outside `main`.** Make it a method.
- **Re-stating signatures in doc comments.** "Returns the foo." Cut it.
