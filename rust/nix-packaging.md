---
source: (our own)
fetched: 2026-04-25
---

# Rust Nix packaging — crane + fenix

The canonical way to package a Rust crate as a Nix flake in this
ecosystem. Replaces the older `rustPlatform.buildRustPackage` pattern
that used to live in `style.md`.

## Why crane + fenix

**crane** layers cargo dependencies as a separate Nix derivation from
the source build:

- `cargoArtifacts` builds all deps once (a content-addressed cache)
- `buildPackage` builds your source against `cargoArtifacts`
- Source-only changes recompile in **seconds**, not minutes

`rustPlatform.buildRustPackage` re-fetches and recompiles every dep on
every source change. That's wasted work the moment a crate has more
than a handful of deps and gets iterated on regularly.

**fenix** gives fine-grained, reproducible toolchain selection:

- Respects `rust-toolchain.toml` directly (`fenix.fromToolchainFile`)
- Lets you pick exact components (`rust-src`, `rust-analyzer`,
  `clippy`, `rustfmt`)
- One toolchain definition shared between `devShells.default` and
  `packages.default` — no dev/release drift

`rust-overlay` is also fine but doesn't read `rust-toolchain.toml` and
doesn't compose components as cleanly.

## Canonical flake.nix (single-crate repo, blueprint-based)

Drop-in for any new Rust repo. Adjust `pname`, `description`.

```nix
{
  description = "<crate description>";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs?ref=nixos-unstable";

    blueprint.url = "github:numtide/blueprint";
    blueprint.inputs.nixpkgs.follows = "nixpkgs";

    fenix.url = "github:nix-community/fenix";
    fenix.inputs.nixpkgs.follows = "nixpkgs";

    crane.url = "github:ipetkov/crane";
  };

  outputs = inputs: inputs.blueprint { inherit inputs; };
}
```

## Canonical packages/default.nix

```nix
{ pkgs, inputs, system, flake, ... }:

let
  toolchain = inputs.fenix.packages.${system}.fromToolchainFile {
    file = flake + "/rust-toolchain.toml";
    sha256 = ""; # first run prints the expected hash; paste it in
  };

  craneLib = (inputs.crane.mkLib pkgs).overrideToolchain toolchain;

  src = craneLib.cleanCargoSource flake;

  commonArgs = {
    inherit src;
    strictDeps = true;
    # nativeBuildInputs = [ pkgs.pkg-config ];
    # buildInputs = [ pkgs.openssl ];
  };

  cargoArtifacts = craneLib.buildDepsOnly commonArgs;
in
craneLib.buildPackage (commonArgs // {
  inherit cargoArtifacts;
})
```

`cargoArtifacts` is the layered cache. The next source-only rebuild
reuses it; only `buildPackage` re-runs.

## Canonical devshell.nix

Same toolchain as the build, plus extra dev tools.

```nix
{ pkgs, inputs, system, flake, ... }:

let
  toolchain = inputs.fenix.packages.${system}.fromToolchainFile {
    file = flake + "/rust-toolchain.toml";
    sha256 = ""; # match the hash from packages/default.nix
  };
in
pkgs.mkShell {
  packages = [
    toolchain
    pkgs.nixfmt-rfc-style
    pkgs.jq
  ];
}
```

## Canonical rust-toolchain.toml

```toml
[toolchain]
channel = "stable"
components = ["cargo", "rustc", "rustfmt", "clippy", "rust-analyzer", "rust-src"]
profile = "default"
```

`channel = "stable"` floats with upstream stable releases. Pin to an
explicit version (`channel = "1.85"`) at release time when bit-for-bit
reproducibility matters.

## Workspace fenix lockstep

When the workspace contains many Rust crates, every crate's
`flake.lock` independently pins its own fenix rev. Without
coordination, those locks drift across update windows: same Rust
version, different fenix overlays, different rustc store paths,
one ~200 MB rustc redownload per divergent rev. On a slim
connection, the cost compounds quickly.

The workspace convention: **one canonical Rust repo holds the
fenix lock; every other Rust repo copies it via
`nix flake lock --inputs-from`.**

In the LiGoldragon workspace today, the canonical fenix-lock
source is **`signal-core`**. All other Rust crates inherit from
its `flake.lock`. To bump fenix workspace-wide:

```sh
cd /git/github.com/LiGoldragon/signal-core
nix flake update fenix
jj commit -m 'bump fenix' && jj bookmark set main -r @- && jj git push --bookmark main
~/primary/tools/sync-rust-fenix
```

`tools/sync-rust-fenix` walks every Rust crate that has a
`fenix` input in its `flake.nix` and runs:

```sh
nix flake lock --inputs-from path:/git/github.com/LiGoldragon/signal-core
```

`--inputs-from` resolves any matching input (only `fenix` here,
since other inputs differ legitimately across repos) using the
canonical flake's locked entry. After the cascade, every Rust
repo references the same fenix rev → one rustc store path →
one download per machine.

When creating a **new Rust repo** that follows this template,
run the sync once after the first `flake.lock` lands so the new
repo joins the lockstep. The reasoning + ownership for the
script lives in
`~/primary/reports/designer/99-shared-rust-toolchain-pin-proposal.md`.

The mechanism (`--inputs-from`) is documented in
`~/primary/skills/nix-discipline.md` §"Lock-side pinning"; this
section names the workspace-specific application of it.

## Git-URL deps with crane

Crane needs an explicit `cargoVendorDir` (it doesn't accept the
`cargoLock = { outputHashes = ...; }` shape that
`rustPlatform.buildRustPackage` does). Build it via
`craneLib.vendorCargoDeps`, keyed by the **full git URL + revision**
(not crate-name-version):

```nix
let
  cargoVendorDir = craneLib.vendorCargoDeps {
    inherit src;
    outputHashes = {
      "git+https://github.com/<org>/<sibling-crate>#<full-rev>" =
        "sha256-<expected-hash-from-first-build-error>";
    };
  };
  commonArgs = {
    inherit src cargoVendorDir;
    strictDeps = true;
  };
in
…
```

Compute hashes ahead of time with `nix-prefetch-git --quiet --url URL
--rev REV`, then convert with `nix hash convert --to sri --hash-algo
sha256 <hash>`. (Or skip them entirely — crane builds fine without,
but emits a `No output hash provided` warning.)

When you bump `Cargo.lock` to a newer rev for a git dep, **the URL
key in `outputHashes` must include the new rev** — crane keys by
`URL#REV`, not `crate-name-version`.

## Workspace handling

For a workspace (`Cargo.toml` with `[workspace] members = [...]`),
crane builds all members at once by default. Filter to one binary with
`cargoExtraArgs = "--bin <name>"` on `commonArgs`.

```nix
craneLib.buildPackage (commonArgs // {
  inherit cargoArtifacts;
  cargoExtraArgs = "--bin <bin-name>";
  pname = "<bin-name>";
})
```

## checks.default

Wire `nix flake check` to run `cargo test` against the same toolchain:

```nix
# checks/default.nix
{ pkgs, inputs, system, flake, ... }:

let
  toolchain = inputs.fenix.packages.${system}.fromToolchainFile {
    file = flake + "/rust-toolchain.toml";
    sha256 = "";
  };
  craneLib = (inputs.crane.mkLib pkgs).overrideToolchain toolchain;
  src = craneLib.cleanCargoSource flake;
  commonArgs = { inherit src; strictDeps = true; };
  cargoArtifacts = craneLib.buildDepsOnly commonArgs;
in
craneLib.cargoTest (commonArgs // { inherit cargoArtifacts; })
```

Or share the `cargoArtifacts` derivation with `packages/default.nix`
through a `lib/`-level helper if you don't want to recompute deps for
checks.

## Pitfalls

- **First-run hash error**: `fenix.fromToolchainFile` and
  `cargoLockOutputHashes` both fail on the first build with a hash
  mismatch. Read the expected hash out of the error and paste it in.
  Subsequent builds are silent.
- **`strictDeps = true`** is required by crane to enforce that
  `nativeBuildInputs` and `buildInputs` don't leak into each other.
  Don't drop it.
- **`craneLib.cleanCargoSource`** strips non-cargo files (markdown,
  nix files, etc.) from the source. If your build needs extra files
  (e.g. embedded assets), use `lib.fileset` instead — see crane docs.
- **rust-overlay vs fenix**: don't mix. Pick one per repo. fenix is
  the recommended default for this ecosystem because of
  `fromToolchainFile`.
