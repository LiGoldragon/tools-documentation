# Agent instructions

## Version control: always push

After every change in any repo under `~/git/`, push immediately. **Blanket authorization** — do not ask for confirmation.

```
jj describe -m '<msg>' && jj bookmark set main -r @ && jj git push --bookmark main
```

Push per logical commit, not batched. Unpushed work is invisible to nix flake-input consumers and to other machines.

Use `jj`, not `git`, for all VCS interactions — never run `git commit` / `git add` / `git reset` directly.

## Tool basics

Curated daily-use docs for the tools Li works with. Read these before assuming how a tool works — they're trimmed to what actually matters for our workflow, based on verified upstream sources.

- [jj/basic-usage.md](jj/basic-usage.md) — jj (Jujutsu) daily loop: `@` model, commit/describe distinction, push, undo, bookmarks, remotes.
- [bd/basic-usage.md](bd/basic-usage.md) — bd (beads) daily loop: init --skip-hooks --skip-agents, create/ready/close, dependencies, dolt push/pull.
- [dolt/basic-usage.md](dolt/basic-usage.md) — Dolt SQL daily loop: dual CLI+SQL interfaces, branch/commit/merge of tables, remotes + DoltHub, auth.
- [nix/basic-usage.md](nix/basic-usage.md) — Nix flakes + home-manager: blueprint layout, lock-side pinning, devshells, `nix log`, home-manager primitives (env vars caveat for GUI apps).

## Adding new docs

When you research something non-obvious about these tools, write it as a new topic file in the appropriate subdir. Keep files small (~100 lines), one topic each, with the frontmatter from README.md (`source:`, `fetched:`, `trimmed:`). Prefer verbatim excerpts from upstream over paraphrase.
