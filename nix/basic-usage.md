---
source: (our own — synthesized from nix.dev manual, home-manager manual, numtide/blueprint README)
fetched: 2026-04-22
trimmed: skipped NixOS module system internals, overlays, channels, writing derivations, nixpkgs contribution workflow; nix-language code blocks (this file is the CLI reference)
---

# Nix + flakes + home-manager basic usage

## Mental model

Nix builds things deterministically from expressions. Given the same inputs, you get the same output — bit-for-bit. A **flake** is a self-describing unit with:

- `inputs` — dependencies (other flakes, nixpkgs), pinned exactly in `flake.lock`.
- `outputs` — what the flake provides (packages, devshells, NixOS / home-manager modules, apps).

Think "`Cargo.toml` + lockfile, for everything, not just Rust". The lock is machine-generated; you commit it.

`flake.lock` is generated; never hand-edit.

## Blueprint layout (what we use)

`numtide/blueprint` auto-maps folders to outputs so `flake.nix` stays tiny:

| Folder        | Flake output                                       |
|---------------|----------------------------------------------------|
| `packages/`   | `packages.<system>.*`                              |
| `devshells/`  | `devShells.<system>.*`                             |
| `modules/`    | `nixosModules.*` / `darwinModules.*`               |
| `hosts/`      | `nixosConfigurations.*` / `darwinConfigurations.*` |

## Day-to-day commands

```
nix flake show                           # list outputs this flake exposes
nix flake check                          # evaluate + run flake's tests/checks
nix flake update                         # refresh all inputs in flake.lock
nix flake update nixpkgs blueprint       # refresh named inputs only (positional)
nix build .#<name>                       # build an output; result → ./result
nix build .#packages.x86_64-linux.<name> # fully-qualified form
nix develop                              # enter the default devshell
nix develop .#<name>                     # enter a named devshell
nix run .#<app>                          # build + run an app output
nix log <drv-or-store-path>              # retrieve logs from a past build
```

`nix flake update` takes **positional** input names. The old `--update-input <name>` flag is deprecated.

## Pinning inputs

Prefer lock-side pinning — keep `flake.nix` generic, record the exact rev in `flake.lock`:

```
nix flake lock --override-input nixpkgs github:criome/nixpkgs/<rev>
```

This rewrites the `nixpkgs` entry in `flake.lock` while `flake.nix` still says `github:NixOS/nixpkgs?ref=nixos-unstable`. Our CriomOS / mentci-* flakes do exactly this: nixpkgs is pinned to `criome/nixpkgs` at a known rev in `flake.lock`, and any machine can re-pin by running `nix flake lock --override-input` again. Verify it took by inspecting `flake.lock` (look for the `locked` block under the input).

To reuse the rev another flake already pins — without ever typing a hash — use `--inputs-from`:

```
nix flake lock --inputs-from path:/home/you/git/sibling-flake
```

This resolves any inputs that match names in the sibling flake using the sibling's locked entries. Handy when you hoist a transitive input to the top level and want it to stay in sync with a consumer flake's lock.

## Dev shells

`nix develop` drops you into the default devshell (blueprint: `devshells/default.nix`). `nix develop .#<name>` for a named one. With direnv: drop `use flake` into `.envrc` and `direnv allow` — shell auto-loads on `cd`.

## Home-manager

```
home-manager switch --flake .#<name>       # apply config
home-manager generations                    # list past generations
home-manager expire-generations "-7 days"   # cleanup old generations
home-manager news                           # upstream news since last build
```

**Caveat: `home.sessionVariables` only reaches shells** — written to `hm-session-vars.sh`, sourced by shell init. GUI apps launched from your DE don't see it. Route GUI-reachable env through `xdg.configFile."environment.d/<name>.conf"` instead — systemd reads `~/.config/environment.d/*.conf` when starting the user session, so GUI apps inherit it.

## Debugging builds

- **Past build failed, want the log without rebuilding?** The error message gave you a `.drv` path — run `nix log /nix/store/...-foo.drv`. Works on store output paths too (`nix log /nix/store/...-foo`).
- **Want logs inline while building?** `nix build -L` (long form: `--print-build-logs`) streams full build output to stderr instead of the default compact UI.
- **Pinning not taking effect?** Re-run `nix flake lock --override-input <name> <url>` and read the relevant `nodes.<name>.locked` block in `flake.lock` to confirm the `rev`.
- **Flake won't evaluate?** `nix flake check` surfaces eval errors across all outputs; `nix flake show` is lighter (just lists outputs, fails on the first broken one).

## Want more

- Nix manual: https://nix.dev/manual/nix/latest/
- nix.dev tutorials: https://nix.dev/
- Home-manager manual: https://nix-community.github.io/home-manager/
- Blueprint: https://github.com/numtide/blueprint
