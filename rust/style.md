---
source: (our own)
fetched: 2026-04-23
---

# Rust style

## The rules in one sentence

**Behavior lives on types. Domain values are typed. Boundaries take and
return one object. Errors are enums you implement by hand.**

## Beauty is the criterion

Every rule below is downstream of one: **if it isn't beautiful, it
isn't done.** Ugly code is evidence that the underlying problem is
unsolved — special cases stacked on the normal case, untyped
quantities, dead branches, hidden coupling, abbreviations that don't
read as English. Each is a *signal* that the right structure has not
yet been found. Slow down and find it. The aesthetic discomfort
*is* the diagnostic reading.

The full case for this — the philosophical foundation, the
practitioners' explicit defense (Hardy / Hoare / Dijkstra / Brooks /
Hickey / Torvalds), and the catastrophic record of what happens when
ugly engineering ships (Therac-25 / Ariane 5 / Mars Climate Orbiter
/ Heartbleed / Boeing MCAS) — lives in
[`programming/beauty.md`](../programming/beauty.md). Read it
before pushing back on a rule below as "verbose" or "ceremonial."
The objection is almost always training-data drift, not informed
judgment.

Per Li (2026-04-27): *"Fuck ugliness and non-conciseness. Who knows
how many people were put to death and tortured because someone
wasn't concise and explicit enough."* The rules below are how we
make the codebase beautiful in the operative sense — not pretty,
but right.

## Methods on types, not free functions

The only free function in a binary crate is `main`. Reusable behavior is
a method on a type or a trait impl. Test helpers are methods on a fixture
struct.

```rust
// Wrong
pub fn parse_cert(pem: &str) -> Result<Cert, Error> { … }

// Right
impl Cert {
    pub fn from_pem(pem: &str) -> Result<Self, Self::Error> { … }
}
```

A small private helper inside one module is fine if it is genuinely local
(`fn hex(h: &Hash) -> String` next to a single Display impl). Anything
that smells reusable becomes a method.

### Why this matters — affordances, not operations

For the cross-language version of this discipline (the same idea
stated language-neutrally, plus the LLM-codegen reasoning and the
principled exceptions), see
[`programming/abstractions.md`](../programming/abstractions.md).
This section is the Rust-specific enforcement.

Methods encode **affordances** — what kinds of things a value of this
type can do. Free functions encode **operations** that happen to take
some arguments. The distinction is load-bearing.

In the real world, fruits can be eaten and clouds cannot. Code that
models the world correctly says `fruit.eat()`, not `eat(fruit)`. The
method form binds the verb to the type that owns it. The free-function
form lets the verb float — and `eat(cloud)` becomes thinkable, type-
checked only if you happen to have given `Cloud` an explicit
"missing eat" marker.

**The deeper consequence**: free functions let the agent SKIP creating
the type that should own the behavior. If you're tempted to write

```rust
// Wrong
pub fn parse_query(text: &str) -> Result<QueryOp, Error> { … }
```

the rule forces you to ask: *what type owns query parsing?* The answer
is `QueryParser`, and the rule's pressure makes that type exist:

```rust
// Right
pub struct QueryParser<'input> { lexer: Lexer<'input> }

impl<'input> QueryParser<'input> {
    pub fn new(input: &'input str) -> Self { … }
    pub fn into_query(self) -> Result<QueryOp, Error> { … }
}
```

Without the rule, `QueryParser` never gets created — the verb stays a
floating free function, the noun that should own it never appears, and
the model develops gaps. Free functions absorb behavior that should
belong to typed entities, and the type system can't track affordances
it doesn't know about. Programs that "look fine" end up missing whole
structural types they ought to have.

The rule of thumb: **every reusable verb belongs to a noun**. If you
can't name the noun, you haven't found the right model yet — keep
looking until you can.

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

**Anonymous tuples are not used at type boundaries** — not as return
types, not as parameter types, not as struct fields, not in type
aliases. The exception is **tuple newtypes**: `struct Md5([u8; 16])`,
`struct NodeName(String)`. They use tuple syntax to wrap a single
thing, but the wrapper itself is a named type — that's how typed
newtypes are spelled and is the whole point. Local destructuring like
`let (a, b) = pair;` against a tuple-newtype's inner is fine; the rule
is about type-level appearances of unnamed tuples.

The verb is the method name; the noun is the type. Don't smuggle the
verb into the type name (`DownloadRequest` + `download_url(req)`) —
make it a method on the input (`Request::download`).

```rust
// Wrong — multi-primitive args at the boundary
fn download_url(&self, md5: &str, path_index: Option<u32>,
                domain_index: Option<u32>) -> Result<Download, Error> { … }

// Wrong — free function with tuple return
fn parse_results(html: &str) -> Result<(Vec<SearchResult>, bool), Error> { … }

// Right — input is a Request; the verb is a method on it
struct Request { md5: Md5, path_index: Option<u32>, domain_index: Option<u32> }

impl Request {
    pub fn download(&self) -> Result<Download, Error> { … }
}

// Right — input is a SearchPage; parse is a method on it
struct SearchPage { html: String, page: u32 }

impl SearchPage {
    pub fn parse(&self) -> Result<SearchResponse, Error> { … }
}

// Right — one explicit object alongside self (relational operation)
impl Tree {
    pub fn merge(&self, other: Tree) -> Result<Tree, Error> { … }
}
```

`self` is implicit; the rule counts explicit arguments only. A method
takes zero or one typed object alongside `self`.

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

## Naming — full words by default

Identifiers are read far more than they are written. Cryptic abbreviations
optimize for the writer (a few keystrokes saved) at the reader's expense
(one mental lookup per occurrence). The empirical literature is unanimous;
the cultural inertia toward `ctx` / `tok` / `de` / `pf` is fossil from
6-char FORTRAN, 80-column cards, and 10-cps teletypes — none of which
still apply. See [naming-research.md](../programming/naming-research.md)
for the full research.

**Default: spell every identifier as full English words.**

```rust
// Wrong — cryptic in-group dialect
fn parse(input: &str) -> Result<Token, Error> {
    let mut lex = Lexer::new(input);
    let tok = lex.next_tok()?;
    let kd = tok.kind();
    let ctx = ParseCtx::new(&kd);
    let de = Deser::with_ctx(ctx);
    de.deser_op(&tok)
}

// Right — every name reads as English
fn parse(input: &str) -> Result<Token, Error> {
    let mut lexer = Lexer::new(input);
    let token = lexer.next_token()?;
    let kind = token.kind();
    let context = ParseContext::new(&kind);
    let deserializer = Deserializer::with_context(context);
    deserializer.deserialize_operation(&token)
}
```

Common offenders to spell out:

| bad | good |
|---|---|
| `lex` | `lexer` |
| `tok` | `token` |
| `ident` | `identifier` |
| `op` | `operation` (or specific: `assert_op`) |
| `de` | `deserializer` |
| `kd` | `kind_decl` (or `KindDecl`) |
| `pf` | `pattern_field` |
| `ctx` | `context` (or specific: `parse_context`) |
| `cfg` | `config` (or `configuration`) |
| `addr` | `address` |
| `buf` | `buffer` |
| `tmp` | `temporary` (or — better — name what it holds) |
| `arr` | `array` (or — better — what it contains) |
| `obj` | (name what it actually is) |
| `params` | `parameters` |
| `args` | `arguments` |
| `vars` | `variables` |
| `proc` | `procedure` or `process` |
| `calc` | `calculate` |
| `init` | `initialize` |
| `repr` | `representation` |
| `gen` | `generate` or `generator` |
| `ser` / `deser` | `serialize` / `deserialize` |
| `fn` (in identifier) | `function` (the `fn` *keyword* is fine) |
| `impl` (in identifier) | `implementation` (the `impl` *keyword* is fine) |

### Permitted exceptions — tight, named, no others

1. **Loop counters in tight scopes (<10 lines).** `for i in 0..n` is fine.
   Beyond ~10 lines or nested, use descriptive names.
2. **Mathematical contexts** where the math itself uses the symbol.
   `x`, `y`, `z`, `theta`, `phi`, `lambda`, `n` for sample size,
   `p` for probability — only when the surrounding code or comment
   establishes the math context.
3. **Generic type parameters.** `T`, `U`, `V`, `K`, `E`. Use a descriptive
   name when the parameter has non-trivial semantic content.
4. **Acronyms in general English.** `id`, `url`, `http`, `json`, `uuid`,
   `db`, `os`, `cpu`, `ram`, `io`, `ui`, `tcp`, `udp`, `dns`. Spell them
   out when ambiguous in context.
5. **Names inherited from `std` or well-known crates.** `Vec`, `HashMap`,
   `Arc`, `Rc`, `Box`, `Cell`, `RefCell`, `Mutex`, `mpsc`, `regex`. Don't
   rename these; do *not* extend the abbreviation pattern to your own
   types.
6. **Domain-standard short names already documented in an
   `ARCHITECTURE.md`.** `slot`, `opus`, `node`, `frame` are full words
   and need no exception. If a true short form is load-bearing in the
   schema, name it in `ARCHITECTURE.md` so the exception is explicit;
   otherwise spell it out.

### Rule of thumb (Martin / Linus, combined)

**Name length proportional to scope.** A 3-line loop counter can be `i`.
A module-level type that appears across the codebase must spell itself
out. A function parameter that lives for 50 lines must read as English.

### Interaction with the Rust convention `&self`

`self` is the implicit receiver and is universal across Rust — leave it.
This rule is about *naming the things you create*, not renaming the
language's primitives.

### How to apply when generating code

Spell identifiers as full English words by default. When the surrounding
code uses cryptic identifiers, do not propagate them into new code:
either rename (if rename is in scope) or use the full form for new
identifiers and flag the inconsistency as a follow-up. Pattern-matching
the local dialect is exactly the failure mode this rule exists to break.

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

## Actors: logical units with ractor

When the daemon grows enough concurrent state to need an actor
framework, the reason is **logical cohesion, coherence, and
consistency** — not performance. An actor is the unit you reach for
when you want to model a coherent component: it owns its state,
exposes a typed message protocol, and has a defined lifecycle. The
framework is [`ractor`](https://crates.io/crates/ractor).

- **Messages are typed.** Each actor's message type is its own enum,
  one variant per request kind. No untyped channels.
- **State is owned, not shared.** The actor's state lives inside the
  actor and is mutated only by its message handlers. `Arc<Mutex<T>>`
  shared between actors is a smell — send a message to whoever owns
  the state.
- **Supervision is recursive.** An actor that spawns sub-actors
  supervises them. Failures escalate; the parent decides restart vs
  shutdown. No detached tasks.
- **Use actors for components, not for chores.** A function that
  awaits an HTTP call is a method, not an actor. An actor exists
  because the *concept* it models warrants its own state and
  protocol.

**Ractor is the default** for any component with state and a
message protocol. The per-actor overhead is negligible on modern
hardware, and the discipline (typed messages, owned state,
supervision trees) pays back immediately — you never end up
retrofitting concurrency later. Ractor pulls tokio in; that's
acceptable everywhere, which is why "`tokio` only where async
I/O actually matters" above is a historical constraint being
relaxed: for daemons and structured services, tokio via ractor
is just the runtime.

For the *how* — the per-file four-piece template, perfect-
specificity messages, supervision, self-cast loops, and the
sync-façade pattern — see [`ractor.md`](ractor.md), grounded
in the [criome](https://github.com/LiGoldragon/criome) example.

Plain sync code is fine for one-shot CLIs, build tools, and
library crates with no concurrent state.

## One Rust crate per repo

Rust crates live in their own dedicated repos and are consumed via flake
inputs. Don't inline a Rust crate inside a non-Rust repo (e.g. under a
NixOS-platform repo's `packages/`).

```
# Right
github:LiGoldragon/clavifaber       — its own repo
github:LiGoldragon/brightness-ctl   — its own repo
github:LiGoldragon/horizon-rs       — its own workspace repo

# Wrong
CriomOS/packages/brightness-ctl/    — Rust crate inlined in a Nix repo
```

Why: a Rust crate has its own toolchain pin, its own Cargo lockfile, its
own test surface, its own release cadence, and its own style obligations.
Inlining one inside a heterogeneous repo couples those concerns to the
host repo's churn for no gain. Consume via flake input instead.

A workspace of related Rust crates (e.g. lib + cli) belongs in **one** repo
together. The split is per *project*, not per crate.

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

### Split traits into their own files when they accumulate

When a single file grows past ~300 lines because traits have piled up
on a type, split each trait impl into its own file. The file for a
type holds the type definition + its inherent impls; each separate
file holds one trait impl for that type, named for the trait.

```
src/cert/
├── mod.rs              # type definition + inherent impls (Cert::new, fields)
├── from_str.rs         # impl FromStr for Cert
├── display.rs          # impl Display for Cert
├── try_from_pem.rs     # impl TryFrom<Pem> for Cert
└── serde_impls.rs      # impl Serialize + Deserialize for Cert (paired traits)
```

This is the deliberate trade-off **explicit code is fine; long files
are not**. Splitting trait impls into separate files keeps any single
file readable, makes the type's surface discoverable from the
directory listing, and prevents impl blocks from growing into a wall
of unrelated behavior.

Use this pattern when traits accumulate. Don't pre-split a type with
two trait impls — that's premature ceremony. Split when a file is
becoming hard to navigate.

## Tests live in separate files

Unit tests do **not** go in a `#[cfg(test)] mod tests` block at the bottom
of the source file. They live in a sibling file under `tests/` at the
crate root, named for the module they exercise.

```
src/
├── cert.rs
├── tree.rs
└── error.rs
tests/
├── cert.rs      # integration tests for Cert
└── tree.rs      # integration tests for Tree
```

This keeps the source file focused on behavior, lets the test file grow
without bloating the source file, and forces tests to exercise the
public API (integration tests can't reach private items — which is the
right pressure: if something is hard to test from outside, the API
needs work, not the test). Private-helper tests are rare and can go in
a small `tests_internal` module with a clear boundary; if you find
yourself reaching for many, that's a signal the helper wants to be its
own type with a public constructor.

One test file per source file. Don't collect tests from multiple
modules into a single `tests/common.rs` unless the shared fixtures
genuinely apply to more than one module.

## Nix-based tests

Every Rust crate exposes its test suite as `checks.default` in its
`flake.nix`. `nix flake check` builds the crate and runs `cargo test`
inside a pure nix sandbox, which:

- Pins the toolchain to the flake's `fenix` component — no
  host-rustc drift.
- Resolves dependencies from a committed `Cargo.lock` — no
  "works on my machine" gaps.
- Makes the test invocation self-documenting: any Nix checkout
  reproduces the exact suite.

Always use `nix flake check` as the canonical pre-commit test
runner. `cargo test` alone skips the reproducibility guarantees.

**Canonical flake layout** for new crates (crane + fenix, with
layered cargo-deps caching so source-only changes recompile in
seconds): see [nix-packaging.md](nix-packaging.md). The same file
covers `rust-toolchain.toml`, git-URL deps, workspace handling, and
`checks.default` wiring.

**Commit `Cargo.lock`.** The flake reads it to vendor dependencies.
Without it, `nix flake check` fails.

If a bug is found while writing tests, fix the bug *and* keep the
test. The test documents the invariant; the fix keeps the invariant
true. A test marked `#[ignore]` with a bd-issue link in the reason is
acceptable when the fix is out of scope for the current work, but
don't `#[ignore]` without filing.

## Cargo.toml

- `edition = "2024"`.
- **One crate per repo. No Cargo workspaces.** A workspace is a
  deployment concern, not a source-layout concern. Each crate that
  compiles to an artifact (library or binary) lives in its own
  git repo with its own `Cargo.toml`, `flake.nix`, `rust-toolchain.toml`,
  `.gitignore`, and `LICENSE.md`. Upstream at
  `github.com/LiGoldragon/<name>`.
- Serialization: `rkyv` for binary contracts between our Rust
  components (storage, zero-copy reads); `serde` only at external
  boundaries that demand it (e.g., JSON for legacy interop).
  Internal text formats use [`nota-codec`](https://github.com/LiGoldragon/nota-codec)'s
  typed `Decoder` + `Encoder`, not serde.
- `tokio` comes in automatically via ractor for any service with
  concurrent state. Plain sync is fine for one-shot CLIs and
  library crates.
- Standard for errors: `thiserror`. Forbidden: `anyhow`, `eyre` —
  they erase error types at boundaries.

## Cross-crate dependencies

Because each crate lives in its own repo, cross-crate references
happen at two layers:

- **Local development:** flake inputs. Each repo's `flake.nix`
  declares its sibling dependencies as inputs; `cargo` resolves them
  through `path = "..."` pointers populated by the flake's
  `postUnpack` (or by symlinking during a dev shell).
- **Published crates:** crates.io version pins. Once a crate is
  published, consumers can pin to semver ranges instead of tracking
  the flake.

Do NOT use `path = "../sibling-crate"` directly in a `Cargo.toml` —
that assumes a layout that a fresh clone won't reproduce. Let the
flake populate the paths.

**Git-URL deps + `cargoLock.outputHashes` pattern.** When the sibling
crate isn't published to crates.io yet but is pushed to GitHub,
consume it via git URL:

```toml
# dependent crate's Cargo.toml
[dependencies]
sibling-crate = { git = "https://github.com/LiGoldragon/sibling-crate.git" }
```

Cargo.lock pins the resolved commit. For `nix flake check` to
fetch the git source inside its sandbox, add an `outputHashes`
entry in the flake's `cargoLock`:

```nix
packages.default = rustPlatform.buildRustPackage {
  ...
  cargoLock = {
    lockFile = ./Cargo.lock;
    outputHashes = {
      # Bump the hash when the locked rev changes. First run fails;
      # nix prints the expected sha256 in the "hash mismatch" error.
      "sibling-crate-0.1.0" = "sha256-...";
    };
  };
};
```

When the dep is later published to crates.io, drop the git URL
and remove the outputHashes entry; pin by semver range instead.

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
