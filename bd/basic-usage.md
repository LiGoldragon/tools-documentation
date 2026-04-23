---
source: (our own — verified against /home/li/git/beads@v1.0.2 source in cmd/bd/ and docs/CLI_REFERENCE.md)
fetched: 2026-04-22
trimmed: dropped federation/shared-server/--global, molecules/formulas/wisps, gas-town/polecat, hooks internals, integrations installers; kept only day-to-day solo flow
---

# bd basic usage

## The one concept that matters

bd stores issues in a Dolt database under `.beads/` (embedded mode: no separate server; bd opens the DB directly). The DB has git-like history — commits, branches, push/pull — so your issue log is versioned. bd commands are CLI for that DB: create, list, update, close, dep-graph, sync.

## bd vs files — when each is the right home

**Rule:** bd is for short tracked items (issues, tasks, workflow state); designs and long-form content live in files in the repo.

- One-paragraph fit → bd issue.
- Multi-section design / survey / report → file in `reports/`, `docs/`, or the crate's directory.
- bd entries summarize and point at the file. Don't paste prose into `--description` or `--notes`; that's the wrong storage for prose.
- Cross-reference: a bd issue's description links to the design file by path; the design file references bd issue IDs for the tracked work.

## Setup

In a fresh repo (this is what you want for a jj-colocated project):

```
bd init --non-interactive --skip-hooks --skip-agents
bd config set export.auto false
```

- `--skip-hooks` is important in jj repos: bd's default git pre-commit hook deadlocks with the git commit bd itself runs during init. Skip them.
- `--skip-agents` suppresses the generated AGENTS.md / Claude plugin settings; you manage those yourself.
- `--non-interactive` prevents prompts (agent/CI safe).
- `export.auto false` disables the automatic jsonl export on every write. The jsonl is a GitHub-legible projection of the Dolt DB; agents read the DB directly, so it's just churn.

Embedded-mode data lives at `.beads/embeddeddolt/<dolt_database>/.dolt/` (the path bd opens in-process). `<dolt_database>` is the `dolt_database` field in `.beads/metadata.json` (defaults to the repo name with hyphens → underscores). Directly queryable with `cd .beads/embeddeddolt/<dolt_database> && dolt sql -q "select count(*) from issues"`.

## Finding work

```
bd ready              # open issues with no active blockers (excludes in_progress/blocked/deferred)
bd list --status open # all open issues (including already-claimed ones)
bd show <id>          # details of one issue
bd blocked            # issues that are blocked by something
```

`bd ready` is not `bd list --ready` — ready applies blocker-aware semantics; `list` only filters by status.

## Creating

```
bd create "Title here" -t task -p 2 -d "Description text"
```

- `-t` type: `task | bug | feature | epic | chore` (default `task`).
- `-p` priority: **0-4 or P0-P4** (0 = critical, 4 = backlog). NOT "high/medium/low".
- `-d` or `--description` for the body; use `--body-file description.md` or stdin for long text.

## Claiming & closing

```
bd update <id> --claim                    # atomic: sets assignee + status=in_progress, fails if already claimed
bd close <id> --reason "what shipped"     # also accepts --resolution, -m, --comment as aliases
```

## Dependencies

```
bd dep add <dependent> <blocker>          # "<dependent> depends on <blocker>" — blocker must close first
bd dep add bd-42 --blocked-by bd-41       # same thing, clearer phrasing
bd dep tree <id>                          # show the graph from an issue
bd blocked                                # list issues that are blocked
```

Note the arg order: first arg is the issue that has the dependency; second arg is what it waits on. `bd dep add A B` = A is blocked by B.

Only `blocks` dependencies (the default for `dep add`) affect `bd ready`.

## Sync to DoltHub

```
bd dolt push          # commit pending + push to the Dolt remote
bd dolt pull          # pull + merge from the Dolt remote
```

`bd dolt push` is what makes your issue log visible to other clones / humans on DoltHub. Do it at session end.

## Editing issue fields

```
bd update <id> --title "new title"
bd update <id> --description "new body"
bd update <id> --notes "scratch notes"
bd update <id> --design "design notes"
bd update <id> --priority 1
```

Prefer these inline forms. `bd edit <id>` opens `$EDITOR` interactively — fine for humans, **do not use from agents** (it hangs waiting on the editor). The bd docs explicitly call this out and do not expose `bd edit` to the MCP server.

## Memory (per-project notes that survive sessions)

```
bd remember "always run tests with -race" --key test-race
bd memories                     # list all
bd memories dolt                # substring search across keys + values
bd recall test-race             # fetch one by key
```

In embedded mode, memories live in this repo's DB — they're project-scoped, not global. That's a feature: each project has its own working-set.

## Daily loop

```
bd ready                                          # what can I do?
bd update <id> --claim                            # claim it
# ... do the work ...
bd close <id> --reason "<repo subject>"           # close with an S-expr-style reason
bd dolt push                                      # sync to DoltHub
```

That's 95% of real usage.

## Want more

- Full CLI reference (verbose, all flags): `/home/li/git/beads/docs/CLI_REFERENCE.md`
- Config keys + import/orphan handling: `/home/li/git/beads/docs/CONFIG.md`
- Dependency model: `/home/li/git/beads/docs/DEPENDENCIES.md`
- Upstream site: https://steveyegge.github.io/beads/
- Source: https://github.com/steveyegge/beads
