---
source: (our own)
fetched: 2026-04-23
---

# Rust toolchain reference

> The Rust **discipline** lives in this workspace's
> `skills/rust-discipline.md`. That skill is what to read
> before writing Rust: methods on types, no ZST holders,
> domain newtypes, one-object in/out, error enums, actor
> topology, tests in separate files, naming, module layout,
> documentation.
>
> This file is the toolchain-side reference: `Cargo.toml`
> shape, cross-crate dependency mechanics, pin strategy.
>
> Neighboring lore files: `nix-packaging.md` for crane +
> fenix; `rkyv.md` for the binary contract format;
> `testing.md` for testing patterns. Actor runtime choice lives
> in the active workspace's actor-system skill; Kameo tool notes
> live in `kameo.md`.

## Cargo.toml

- `edition = "2024"`.
- **One crate per repo. No Cargo workspaces.** A workspace is
  a deployment concern, not a source-layout concern. Each
  crate that compiles to an artifact (library or binary)
  lives in its own git repo with its own `Cargo.toml`,
  `flake.nix`, `rust-toolchain.toml`, `.gitignore`, and
  `LICENSE.md`.
- Serialization: `rkyv` for binary contracts between Rust
  components (storage, zero-copy reads); `serde` only at
  external boundaries that demand it (e.g., JSON for legacy
  interop). Internal text formats use a single chosen typed
  text codec (one project-wide decoder/encoder pair), not
  serde.
- `tokio` comes in through the selected async runtime for any
  service with concurrent state. Plain sync is fine for one-shot
  CLIs and library crates.
- Standard for errors: `thiserror`. Forbidden: `anyhow`,
  `eyre` — they erase error types at boundaries.

## Cross-crate dependencies

Because each crate lives in its own repo, cross-crate
references happen at two layers:

- **Local development:** flake inputs. Each repo's
  `flake.nix` declares its sibling dependencies as inputs;
  `cargo` resolves them through `path = "..."` pointers
  populated by the flake's `postUnpack` (or by symlinking
  during a dev shell).
- **Published crates:** crates.io version pins. Once a crate
  is published, consumers can pin to semver ranges instead
  of tracking the flake.

Do NOT use `path = "../sibling-crate"` directly in a
`Cargo.toml` — that assumes a layout that a fresh clone
won't reproduce. Let the flake populate the paths.

**Git-URL deps + `cargoLock.outputHashes` pattern.** When the
sibling crate isn't published to crates.io yet but is pushed
to GitHub, consume it via git URL:

```toml
# dependent crate's Cargo.toml
[dependencies]
sibling-crate = { git = "https://github.com/<org>/sibling-crate.git" }
```

Cargo.lock pins the resolved commit. For `nix flake check`
to fetch the git source inside its sandbox, add an
`outputHashes` entry in the flake's `cargoLock`:

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

When the dep is later published to crates.io, drop the git
URL and remove the outputHashes entry; pin by semver range
instead.

**Pin strategy — `branch = "main"` during fast development.**
Prefer `branch = "main"` over `rev = "<sha>"` for sibling
deps while development is moving fast across the workspace.
The lockfile still pins by sha (deterministic builds); the
Cargo.toml stays stable as upstream evolves.
`cargo update -p <dep>` bumps the lock when you want the
latest. When a sibling stabilises and you want frozen
upstream, switch that one to `rev = "<sha>"` — but the
default for fast-moving cross-crate deps is `branch = "main"`.

```toml
# Default for fast-moving sibling deps:
sibling-crate = { git = "https://github.com/<org>/sibling-crate.git", branch = "main" }
```

For Nix-flake input choice (a separate concern from
Cargo-dep mechanics), see this workspace's
`skills/nix-discipline.md` — specifically the rule against
`git+file://` flake inputs.
