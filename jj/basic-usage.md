---
source: (our own — based on docs.jj-vcs.dev/latest/tutorial/ trimmed to essentials)
fetched: 2026-04-22
trimmed: dropped rebase/merge/cherry-pick flows and internals; kept only what's needed for daily solo work
---

# jj basic usage

## The one concept that matters

The working copy IS a commit, called `@`. jj auto-snapshots `@` on every jj command. There is no staging area, no `git add`, no "nothing to commit" state. When you run any jj command, jj notices changes to files in the working tree and absorbs them into `@` before doing its thing.

A normal session: edit files → run `jj commit -m "..."` → jj finalizes `@` with the description and starts a fresh empty `@` on top. Repeat.

## Status & log

```
jj st        # what's changed in the working copy
jj log       # recent history (ancestors of @ by default)
jj show <r>  # details + diff of a specific revision
jj diff      # diff of @ vs its parent (your uncommitted work)
```

## Make a commit

```
jj commit -m "message"
```

What you want 95% of the time. It: (a) gives `@` the description, (b) moves `@` to a new empty commit on top, so your next edits start clean.

`jj describe -m "msg"` just **names** `@` without advancing. Use this when you want to set the message of the current WIP without moving on.

## Push

```
jj bookmark set main -r @-      # point "main" at @- if @ is the new empty one (after jj commit)
jj bookmark set main -r @       # point "main" at @ if you used jj describe (no advance)
jj git push --bookmark main
```

Push after every change. Don't batch.

## Undo

```
jj undo        # undo the last jj operation (any — describe, commit, rebase, abandon)
jj abandon     # drop @ entirely (still recoverable via jj undo)
jj restore     # drop working-copy edits back to parent state (for paths or whole @)
```

`jj undo` is the superpower. git can't do this.

## Switch between commits

```
jj new <rev>     # new empty commit on top of <rev>; @ moves there
jj edit <rev>    # make <rev> the working-copy commit; you mutate it in place
```

`jj new` = "check out and start new work from here".
`jj edit` = "go back and tweak that old commit".

## Bookmarks (= branches)

```
jj bookmark list
jj bookmark create <name> -r <rev>
jj bookmark move <name> --to <rev>
jj bookmark delete <name>
```

jj calls branches "bookmarks" because there is no "current branch" — bookmarks are movable name refs; you're never "on" one.

## Remotes

```
jj git clone <url>                   # initial clone (jj-native)
jj git init --colocate               # convert an existing git repo to jj colocation
jj git fetch [--remote origin]
jj git push --bookmark <name> [--remote origin]
jj git remote add <name> <url>
```

Colocation means jj and git share the same `.git/`; both CLIs work in the repo.

## Daily loop

```
jj st                                        # see state
# ... edit files ...
jj commit -m '(<repo> <subject>) (<note>)'   # commit, Mentci-style message
jj bookmark set main -r @-                   # point main at what we just committed
jj git push --bookmark main
```

That's 95% of real usage.

## Want more

- Full docs: https://docs.jj-vcs.dev/latest/
- Git comparison table: https://docs.jj-vcs.dev/latest/git-command-table/
- Tutorial: https://docs.jj-vcs.dev/latest/tutorial/
