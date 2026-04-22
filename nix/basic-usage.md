---
source: (our own — synthesized from nix.dev manual, home-manager manual, numtide/blueprint README)
fetched: 2026-04-22
trimmed: skipped NixOS module system internals, overlays, channels, writing derivations, nixpkgs contribution workflow
---

# Nix + flakes + home-manager basic usage

## Mental model

Nix builds things deterministically from expressions. Given the same inputs, you get the same output — bit-for-bit. A **flake** is a self-describing unit with:

- `inputs` — dependencies (other flakes, nixpkgs), pinned exactly in `flake.lock`.
- `outputs` — what the flake provides (packages, devshells, NixOS / home-manager modules, apps).

Think "`Cargo.toml` + lockfile, for everything, not just Rust". The lock is machine-generated; you commit it.

## Flake file structure

`flake.nix`:

```nix
{
  description = "...";
  inputs.nixpkgs.url = "github:NixOS/nixpkgs?ref=nixos-unstable";
  outputs = { self, nixpkgs, ... }: { /* ... */ };
}
```

`flake.lock` — generated; never hand-edit.

### Blueprint layout (what we use)

`numtide/blueprint` auto-maps folders to outputs so `flake.nix` stays tiny:

| Folder        | Flake output                                       |
|---------------|----------------------------------------------------|
| `packages/`   | `packages.<system>.*`                              |
| `devshells/`  | `devShells.<system>.*`                             |
| `modules/`    | `nixosModules.*` / `darwinModules.*`               |
| `hosts/`      | `nixosConfigurations.*` / `darwinConfigurations.*` |

Minimal `flake.nix` with blueprint:

```nix
{
  inputs.blueprint.url = "github:numtide/blueprint";
  inputs.nixpkgs.url = "github:NixOS/nixpkgs?ref=nixos-unstable";
  outputs = inputs: inputs.blueprint { inherit inputs; };
}
```

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

## Dev shells

`devshells/default.nix` (with blueprint):

```nix
{ pkgs, ... }: pkgs.mkShell {
  packages = [ pkgs.ripgrep pkgs.jq ];
  shellHook = ''echo "hi"'';
}
```

`nix develop` drops you in. With direnv: drop `use flake` into `.envrc` and `direnv allow` — shell auto-loads on `cd`.

## Home-manager essentials

Modules receive the standard attrset:

```nix
{ lib, pkgs, config, ... }: {
  home.packages = [ pkgs.ripgrep pkgs.fd ];

  home.sessionVariables.EDITOR = "hx";   # shells only — see caveat

  home.file.".config/foo/bar".text = "...";      # literal path under $HOME
  xdg.configFile."foo/bar".text = "...";         # under $XDG_CONFIG_HOME

  systemd.user.services.syncthing = { /* Unit/Service/Install */ };
  systemd.user.timers.backup      = { /* OnCalendar etc */ };

  home.activation.linkStuff = lib.hm.dag.entryAfter ["writeBoundary"] ''
    # runs on every `home-manager switch`
  '';
}
```

**Caveat: `home.sessionVariables` only reaches shells** — it's written to `hm-session-vars.sh`, sourced by your shell init. GUI apps launched from your DE don't see it. For GUI-reachable env, use:

```nix
xdg.configFile."environment.d/10-editor.conf".text = ''
  EDITOR=hx
'';
```

systemd reads `~/.config/environment.d/*.conf` when starting the user session, so GUI apps inherit it.

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
