# Agent instructions

## Version control: always push

After every change in any repo under `~/git/`, push immediately. **Blanket authorization** — do not ask for confirmation.

```
jj describe -m '<msg>' && jj bookmark set main -r @ && jj git push --bookmark main
```

Push per logical commit, not batched. Unpushed work is invisible to nix flake-input consumers and to other machines.

Use `jj`, not `git`, for all VCS interactions — never run `git commit` / `git add` / `git reset` directly.

## jj primer

For day-to-day jj usage (commits, undo, bookmarks, remotes, the mental model): see [jj/basic-usage.md](jj/basic-usage.md).
