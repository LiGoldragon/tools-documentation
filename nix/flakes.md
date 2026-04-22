---
source: (our own — 2026-04-22)
fetched: 2026-04-22
trimmed: none
---

# Flakes: inputs and locks

## Common commands

- `nix flake lock` — add missing entries; leave existing ones alone.
- `nix flake update` — re-resolve all inputs to latest.
- `nix flake update <input>` — re-resolve one input only.
- `nix flake show` — list outputs (packages, devShells, checks).
- `nix flake metadata --json` — machine-readable lock state.

## Transitive inputs

When flake A consumes flake B as an input and B declares its own inputs
(like `fenix`), A's lock inherits B's locked revs transitively. No need to
re-declare them in A. `nix flake lock` in A copies from B's lock; it does
not re-resolve.

## Pinning without typing hashes

`nix flake lock --inputs-from <flake-url>` resolves this flake's inputs
using another flake's locked entries. Useful when you hoist a transitive
input to the top level and want it to match the rev a sibling flake
already pins — no hash ever appears in your shell.

## Hash-free lock comparison

```bash
test "$(jq -r '.nodes.X.locked.rev' a/flake.lock)" \
   = "$(jq -r '.nodes.X.locked.rev' b/flake.lock)" \
   && echo MATCH
```
