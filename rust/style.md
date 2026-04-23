---
source: (our own)
fetched: 2026-04-23
---

# Rust style

## The rules in one sentence

**Behavior lives on types. Domain values are typed. Boundaries take and
return one object. Errors are enums you implement by hand.**

## Methods on types, not free functions

The only free function in a binary crate is `main`. Reusable behavior is
a method on a type or a trait impl. Test helpers are methods on a fixture
struct.

```rust
// Wrong
pub fn parse_cert(pem: &str) -> Result<Cert, Error> { … }

// Right
impl Cert {
    pub fn from_pem(pem: &str) -> Result<Self, Error> { … }
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
pub fn details(&self, md5: &str) -> Result<Item, Error> { … }

// Right
pub struct Md5([u8; 16]);
pub fn details(&self, md5: &Md5) -> Result<Item, Error> { … }
```

**The wrapped field is private.** A `pub` field exposes the primitive
and defeats every reason to wrap it: callers can construct unchecked
values and read the raw bytes back out.

```rust
// Wrong — pub field, the type is just a label
pub struct NodeName(pub String);

// Right — field private; construction and access go through methods
pub struct NodeName(String);

impl NodeName {
    pub fn new(s: impl Into<String>) -> Self { Self(s.into()) }   // or TryFrom if validated
}

impl AsRef<str> for NodeName {
    fn as_ref(&self) -> &str { &self.0 }
}
```

Construction with validation goes through `TryFrom<&str>` (or
`from_str`) returning the crate's `Error`.

## One type per concept — no `-Details` / `-Info` companions

If you find yourself defining `Item` *and* `ItemDetails`, stop. The
"-Details" or "-Info" suffix paired with a base type is one concept
fragmented across two types because the base was designed too thin.
Fix the base type. The same applies to `-Extra`, `-Meta`, `-Full`,
`-Extended`, `-Raw`/`-Parsed` pairs, and any other suffix that means
"the real version of the thing next door."

```rust
// Wrong — two types for one concept
struct Item { md5: Md5, name: String }
struct ItemDetails { md5: Md5, name: String, size: u64, mirrors: Vec<Url>, … }

// Right — one Item, complete
struct Item {
    md5: Md5,
    name: String,
    size: u64,
    mirrors: Vec<Url>,
    …
}
```

If different *call sites* genuinely need different *projections*, model
that with a method that returns a smaller view (`item.summary()`), not
with a parallel type.

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

Prefer `TryFrom` when the conversion has one canonical source type;
prefer `from_<src>(…) -> Result<Self, Error>` when there are several
plausible sources or extra args.

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

## Errors: typed enum per crate via thiserror

Each crate defines its own `Error` enum in `src/error.rs`, derived with
`thiserror`. Variants are structured — carry the data needed to render a
useful message. Foreign error types convert via `#[from]`.

```rust
use thiserror::Error;

#[derive(Debug, Clone, Error)]
pub enum Error {
    #[error("chunk not found: {0}")]
    ChunkNotFound(Hash),

    #[error("deserialization failed: {0}")]
    DeserializationFailed(String),

    #[error("invalid node: {0}")]
    InvalidNode(String),

    #[error("merge conflict on key ({} bytes)", key.len())]
    MergeConflict { key: Vec<u8> },

    #[error("network: {0}")]
    Network(#[from] reqwest::Error),
}
```

Public APIs return `Result<T, Error>` with the crate's own enum.
**Never** `anyhow::Result`, `eyre::Result`, or `Result<T, Box<dyn Error>>`
— they erase the error type at the boundary, which loses the typed-failure
discipline the rest of the rules build up. Callers can no longer pattern-
match on what went wrong.

## Concurrency: supervised actors with ractor

Multi-step transformations, long-running orchestrations, and concurrent
task execution go through supervised actors. The framework is
[`ractor`](https://crates.io/crates/ractor).

- **Messages are typed.** Each actor's message type is its own enum
  (one variant per request kind). No untyped channels, no
  `Box<dyn Any>`.
- **Supervision is recursive.** An actor that spawns sub-actors
  supervises them — failures escalate; the parent decides restart vs
  shutdown. Don't spawn detached tasks.
- **State is private.** An actor's state is owned by the actor and
  mutated only inside its message handlers. Don't share state across
  actors via `Arc<Mutex<T>>` — pass a message instead.
- **Async I/O elsewhere is fine.** A method that just awaits an HTTP
  call doesn't need an actor. The bar is *multi-step* or *concurrent*
  work that needs lifecycle and failure handling.

## Module layout

One concern per file. Typical crate:

```
src/
├── lib.rs        # re-exports + crate-level doc (//!)
├── error.rs      # Error enum + impls
├── types.rs      # domain newtypes + small structs
├── <thing>.rs    # one file per major type / subsystem
└── main.rs       # only if the crate is a binary; only free fn lives here
```

Impls live in the same file as the type they're for. Don't split types
and impls across files.

## Cargo.toml

- `edition = "2024"`.
- No Cargo workspaces. Each repo is standalone.
- Serialization: `rkyv` for binary contracts inside the aski pipeline;
  `serde` + `serde_json` only at JSON boundaries where consumers need it.
- `tokio` only where async I/O actually matters (HTTP).
- Standard for errors: `thiserror`. Forbidden: `anyhow`, `eyre` —
  they erase error types at boundaries.

## Documentation

Doc comments are impersonal, timeless, precise. Document the contract;
don't restate the signature.

```rust
impl Cert {
    /// Issue a server certificate against this CA.
    ///
    /// The CA's signing key must be an Ed25519 key resolvable via the
    /// local GPG agent. The server keypair is ECDSA P-256, generated fresh.
    pub fn issue_server(&self, request: ServerCertRequest) -> Result<Self, Error> { … }
}
```

Module-level docs go in `//!` at the top of `lib.rs` or `///` at the top
of a single-purpose module file. Skip docs on obvious boilerplate:
getters, `From` impls, internal helpers.

No examples in doc comments unless the API is non-obvious. No personal
voice. No future tense. Present indicative only.
