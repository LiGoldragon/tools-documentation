# Agent instructions — canonical contract

This file is the **canonical workspace contract**. Every per-repo
`AGENTS.md` is a thin shim pointing here; the rules below apply
to every repo your workspace tracks. Repo-specific carve-outs
live in that repo's own `AGENTS.md`.

---

## 🚨 Required reading, in order 🚨

1. **The workspace's intent document** — typically named
   `ESSENCE.md` or `INTENTION.md` and living at the workspace's
   root (e.g. `~/primary/ESSENCE.md`). It states what the work
   is for: priorities, the deepest value, the intent that
   shapes every decision. **Upstream of every other doc**; if a
   downstream doc conflicts with the intent, the downstream doc
   loses.
2. **The workspace orchestration protocol** — typically
   `protocols/orchestration.md` at the workspace root. It states
   how autonomous agents claim files, check BEADS tasks, and
   avoid interfering with each other.
3. **This file** (`lore/AGENTS.md`) — how agents work in the
   workspace.
4. **The repo-specific `AGENTS.md`** of whatever repo you're
   editing — repo-role + carve-outs only.
5. **The repo's `ARCHITECTURE.md`** — what the repo is and how
   it fits in.
6. **The repo's `skills.md`** — what an agent needs to know to
   be effective in *this* repo (project-specific intent,
   invariants about how to work, pointers to neighboring
   skills). Increasingly canonical.

A new agent entering the workspace reads (1) before making any
decisions; it is upstream of every rule below.

---

## AGENTS.md / CLAUDE.md pattern

Across all canonical repos: **`AGENTS.md` holds the real content**;
`CLAUDE.md` is a one-line shim reading
**"You MUST read AGENTS.md."**
This way Codex (which reads `AGENTS.md`) and Claude Code (which
reads `CLAUDE.md`) converge on a single source of truth. When
creating or restructuring a repo, keep this pattern.

Per-repo `AGENTS.md` itself opens with:
**"You MUST read [lore/AGENTS.md](path) — the canonical agent
contract."** plus the repo's role and carve-outs.

---

## Per-repo `ARCHITECTURE.md` at root

Every canonical repo carries an `ARCHITECTURE.md` at its root
(matklad convention). The file is short — typically 50-150
lines — and answers: *what does this repo do, where do things
live in it, and how does it fit into the wider workspace.*
Standard sections: role, owned-and-not-owned boundaries, code
map, invariants, status, cross-cutting context.

Per-repo `ARCHITECTURE.md` describes only that repo's niche.
Project-wide invariants live once, in the root project's
canonical doc; per-repo files point at that doc by prose. When
a repo's role changes, edit that repo's `ARCHITECTURE.md` and
(if the change is system-level) the project-wide doc.

When creating a new canonical repo: write `ARCHITECTURE.md` at
root before the first commit.

---

## Documentation layers — strict separation

| Where | What |
|---|---|
| Workspace `ESSENCE.md` / `INTENTION.md` | **The intent.** What the work is for, the deepest value, the priorities. Upstream of everything else. |
| Workspace `protocols/<name>.md` | **Coordination protocols.** How autonomous agents share workspace state, claim scopes, check BEADS, and hand off blocked work. |
| Workspace `skills/<name>.md` | **Cross-cutting agent skills.** How to act on routine obstacles, how to edit skill files, etc. One capability per file. |
| `lore/AGENTS.md` (this file) | **Workspace agent contract.** How agents work across repos. |
| Project-wide canonical doc | **Project-wide invariants.** Prose + diagrams. The high-level shape, the relationships of the engine being built. |
| `<repo>/ARCHITECTURE.md` | **Per-repo bird's-eye view.** Repo's role, owned-and-not-owned boundaries, code map, status. Points at the project-wide doc by prose for cross-cutting context. |
| `<repo>/skills.md` | **What an agent needs to know to work in this repo.** Project-specific intent, invariants about how to work, pointers to neighboring skills. Cross-references to other repos' `skills.md` are by repo name (e.g. "criome's `skills.md`"), never by HTTPS URL. |
| `<repo>/reports/NNN-*.md` | **Decision records + design syntheses.** Prose + visuals. |
| Each repo's source | **Implementation.** Code, tests, build files. Type sketches as compiler-checked skeletons live here. |

When a layer rule is violated, rewrite: move type sketches out
of architecture docs into a report or skeleton code; move
runnable code out of reports into the appropriate repo.
Architecture stays slim so it remains readable in one pass.

Cross-references flow *into* architecture from reports. Reading
lists, decision histories, type-spec details live in reports;
the architecture doc stays free of them.

---

## Build the durable shape — intent applied

The intent doc commits the project to the **right shape now**.
The right shape now is worth more than a wrong shape sooner;
unbuilding a wrong shape costs more than the speed it bought.

When work that depends on a durable shape would land before
that shape is ready, the work waits. The waiting is a feature
of the system — it preserves the chance to land the right
shape on the first try.

**The recognition heuristic.** When two paths appear — one
"small / hand-written / quick" and one "the durable shape" —
the durable shape is the answer; the work continues there. If
the durable shape is not yet ready, the dependent work parks
until the durable shape lands.

When deadline pressure pulls toward the small/quick path, the
intent wins. When this rule meets any other rule, the intent
wins.

---

## Cross-references between repos

Reference cross-repo files by **prose**, not by deep github URL.

Cross-repo file URLs of the form
`https://github.com/<org>/<repo>/blob/main/<path>` are
brittle: when files move, get renamed, or are deleted, the URL
silently breaks and the reader follows it into a 404.

Use one of these instead:

- **Prose** — "the project-wide `ARCHITECTURE.md`" or "lore's
  `programming/abstractions.md`." The reader is in the
  workspace and can navigate.
- **Repo-level pointer** — `github:<org>/<repo>` (nix-flake
  notation) for "this repo exists at that URL." It points at
  the repo, not a specific file inside it; renames within stay
  valid.

Inside a single repo, use a markdown link with a relative path
— those resolve in editors and on github regardless of the
repo's location.

---

## Positive framing only

State what IS. Architecture docs and agent rules describe the
system's commitments — what kinds exist, what owns what, what
flow happens, what the agent does. The current shape lives in
the doc; the path that led there lives in version-control
history.

When a direction turns out to be wrong, the doc is rewritten
to state the new direction. The previous direction disappears
from the doc; the commit history preserves it for anyone who
needs to recover the path.

When an option is excluded — by constraint, preference, or
decision — the criterion that excludes it is stated as a
positive requirement. "Must be Rust" replaces "Go is excluded."
The candidates that satisfy the criteria appear in the doc;
the others stay silent.

Each architectural commitment lives once, positively, in the
appropriate canonical doc.

---

## Components, not monoliths

The workspace is composed of **micro-components** — one
capability per crate, per repo, per protocol. Each lives in
its own repo with its own build descriptor and tests; each
fits in a single LLM context window; each speaks to its
neighbors only through typed protocols.

**Adding a feature defaults to a new crate, not editing an
existing one.** The burden of proof is on the contributor
(human or agent) who wants to grow a crate. They must justify
why the new behavior is part of the *same capability* — not a
new one. The default answer is "new crate."

Effect-bearing work — spawning subprocesses, writing files
outside the canonical state store, invoking external tools,
code emission — is dispatched as typed verbs to dedicated
executor components. Communicator components stay
communicators; executor components stay executors.

Full case (the catastrophic record of monolith collapse, the
modern LLM-context argument): the workspace's
`skills/micro-components.md`.

---

## Beauty is the criterion

Read the workspace's `skills/beauty.md`. Beauty is the test of
correctness — **ugly code is evidence that the underlying
problem is unsolved**.

The aesthetic discomfort *is* the diagnostic reading. When
something feels ugly, slow down and find the structure that
makes it beautiful — that structure is the one that was
missing.

---

## Verb belongs to noun (thinking discipline)

Read the workspace's `skills/abstractions.md` before writing free
functions. The discipline applies to any language with method
dispatch: behavior lives on types, not as floating verbs. The
rule's purpose is to force the question "what type owns this
verb?" — when the answer isn't obvious, the model of the
problem isn't fully formed yet, and slowing down to find the
noun is the load-bearing cognitive event.

The rule has a sharper version: the verb belongs to **the
right** noun, not just any nearby noun. Adjacency of *types*
is not the same as adjacency of *concerns* — see the workspace's
`skills/abstractions.md` §"The wrong-noun trap."

---

## Producers push, consumers subscribe

Read the workspace's `skills/push-not-pull.md`. Producers push
state; consumers subscribe to a stream. The subscription
primitive is the contract. When the primitive doesn't yet
exist, the dependent real-time feature waits for it to land.

---

## Naming — full English words

Read the workspace's `skills/naming.md`. Identifiers are read
far more than they are written. Cryptic abbreviations optimize
for the writer (a few keystrokes saved) at the reader's expense
(one mental lookup per occurrence).

**Default: spell every identifier as full English words.**

The canonical rule + the full table of common offenders + the
six permitted exception classes + the "feels too verbose"
anti-pattern all live in the workspace's `skills/naming.md`.

---

## Binary naming — `-daemon` suffix, full English

Long-running daemon binaries carry the `-daemon` suffix. The
library half of the same crate keeps the bare name —
`[lib] name = "<crate>"` and `[[bin]] name = "<crate>-daemon"`
in the same `Cargo.toml`. CLI binaries that the user invokes
interactively keep bare names.

**Why:** unix-folklore abbreviations (`<crate>d`) carry no
information beyond the suffix. `<crate>-daemon` reads as
English; `<crate>d` requires expansion. It also disambiguates
the daemon binary from the CLI at PATH level. Per-process
stderr log tags also use the suffix
(`<crate>-daemon: ready`).

---

## One-shot binaries — `<crate>-<verb>`

Several crates ship thin one-shot binaries that wrap a single
verb of the library code as a stdin/stdout filter — useful for
test pipelines, agent harnesses, debug scripts.

Naming convention: `<crate>-<verb>`, ~30 LoC each.

These expose useful affordances of the library code as
standalone tools — same shape as `gcc -E` shipping alongside
`gcc`. They live as additional `[[bin]]` entries in the
existing crate, not in a separate test-tools crate.

When to add a new one: when a verb of the library code is
genuinely useful as a filter primitive. Each new one earns its
keep through a concrete pipeline use-case.

---

## Docs describe the present

Vision, design, and architecture docs describe what the system
IS and what it's heading toward. They live in the present
tense.

The lineage is preserved by version-control history. When a
piece of context truly matters (e.g. "the validator pipeline
is Datomic-inspired"), state the fact and cite the inspiration
as present-tense influence.

How to apply:

- When rewriting a stale doc, extract the durable ideas and
  recast in current terms. Drop meta-commentary about the
  rewrite itself.
- Commit messages of substance describe the change in present
  tense — what the new state is.
- Version markers (`v3`, `v1`, `v2`) are bookkeeping for
  agents, kept out of published docs.

---

## Reporting

The reporting discipline — when to write a report vs answer
in chat, where reports live, the prose-plus-visuals medium,
the always-name-paths rule for chat references, present
tense, relative paths in cross-references — lives in the
workspace's `skills/reporting.md`. Read it before writing
any output that explains, proposes, analyses, or
summarises.

---

## Verify each parallel-tool result

When batching tool calls (parallel `Write`, `Edit`, `Bash`),
scan each result block for errors **before** any follow-up
step that depends on the results. The bundle returning is not
the same as the bundle succeeding.

Failure mode: failed `Write` calls (typically the "must Read
first" guard) don't show up in subsequent state until
something else reads the file. By then the failure has
cascaded — a wrong commit, a misled subagent, a published doc
that doesn't match what the code reflects.

How to apply:

- After any parallel batch, look at every `<result>` block. If
  any says "File has not been read yet" or "Exit code N" or
  `<tool_use_error>`, fix that *before* moving on.
- Double-check by reading the file or running `git status` /
  `ls` if a write was meant to land.
- Especially load-bearing when the next step is committing,
  pushing, or spawning an agent that consumes the written
  file.
- When a `Write` fails on the must-Read-first guard, do the
  Read then redo the Write before continuing.

---

## Version control

Read the workspace's `skills/jj.md`. After every
change in any tracked repo, commit and push immediately —
blanket authorization. Push per logical commit. Use `jj` for
local history work; Git remains the remote/storage layer. If a
repo lacks `.jj/`, run `jj git init --colocate` after claiming
it through the orchestration protocol.

The skill carries the canonical one-liner, the logical-commit
grouping criteria, the commit-message style (short verb +
scope; one short clause), and the standard fixes for routine
obstacles (HTTPS push failure, divergence, uncommitted state,
missing `.jj/`).

---

## Implicit authorization for missing publish steps

When the user points out that an expected external side effect
is missing, and that side effect is the obvious intended
completion of the work just done, treat that statement as
authorization to perform the missing publish step immediately.

Canonical cases:

- "I don't see the repo on my GitHub" → create/push the repo
  if the work was "make a new repo".
- "The PR isn't open" → open the PR if the work was "publish
  this branch".
- "I don't see the branch / commit / issue on the remote" →
  push the missing VCS or issue-store remote state.

The rule is about **obvious completion of already-requested
work**, not about inventing unrelated side effects. Apply it
when the missing step is the direct
publish/export/remote-materialization leg of the task already
in progress.

---

## Tooling — short tracked items

Short tracked items (issues, tasks, workflow markers) live in
a git-versioned issue store with searchable memory. Designs
and reports go in files. The tracked item's body summarises
and points at the design file; long prose in `--description`
is the wrong storage. See
[bd vs files — when each is the right home](bd/basic-usage.md#bd-vs-files--when-each-is-the-right-home)
for the canonical formulation in the `bd` (beads) tool.

---

## Tool basics

Curated daily-use docs for the tools you reach for in this
workspace. Read these before assuming how a tool works —
they're trimmed to what actually matters for the workflow,
based on verified upstream sources.

**VCS / build:**
- `jj/basic-usage.md` — jj (Jujutsu) daily loop: `@` model,
  commit/describe distinction, push, undo, bookmarks, remotes.
- `nix/basic-usage.md` — Nix flakes + home-manager: blueprint
  layout, lock-side pinning, devshells, `nix log`,
  home-manager primitives.
- `nix/flakes.md` — inputs and locks.
- `nix/integration-tests.md` — chained-derivation patterns
  for daemon-stack tests.

**Programming discipline (cross-language) — now in the workspace
`skills/` directory:**
- `skills/beauty.md` — Beauty as the criterion of correctness.
- `skills/abstractions.md` — Where behavior lives. Every
  reusable verb belongs to a noun.
- `skills/push-not-pull.md` — Producers push, consumers
  subscribe.
- `skills/micro-components.md` — One capability, one crate,
  one repo.
- `skills/naming.md` — Full English words.
- `skills/rust-discipline.md` — Rust-specific application
  (methods on types, no ZST holders, domain newtypes,
  one-object in/out, error enums).
- `skills/nix-discipline.md` — Nix-specific discipline
  (flake input forms, lock-side pinning, store-path hygiene,
  `nix run` over `cargo install`, `nix flake check` as
  pre-commit).
- `skills/language-design.md` — Designing a text notation,
  request language, schema, or query surface.

**Language style:**
- `rust/style.md` — Rust object style: methods on types, typed
  newtypes, single-object I/O at boundaries, typed `Error`
  enum per crate.
- `rust/nix-packaging.md` — Canonical crane + fenix flake
  layout for Rust crates.
- the active workspace's actor-system skill — current runtime,
  mailbox, supervision, and handle discipline. `rust/kameo.md`
  is the current Rust actor runtime tool reference.
- `rust/rkyv.md` — Rkyv 0.8 portable feature set,
  derive-alias pattern, encode/decode API, schema-fragility
  limit.
- `rust/testing.md` — testing patterns: sync façade on State,
  tests in separate files, tempfile, two-process integration
  via `CARGO_BIN_EXE_*`, when to test actors directly.

**Data / issues:**
- `bd/basic-usage.md` — bd (beads) daily loop.
- `dolt/basic-usage.md` — Dolt SQL daily loop.

**CLI tools:**
- `annas/basic-usage.md` — Anna's Archive CLI.
- `linkup/basic-usage.md` — Linkup web-search CLI.
- `substack/basic-usage.md` — Publish Markdown to Substack.

---

## Style docs describe patterns

Docs under `programming/` and language-style docs under
`<language>/` describe **patterns**. The pattern is what's
load-bearing; concrete examples (actor counts, file paths,
function names) are illustrative-now, kept to one-line "see
for current shape" pointers. The pattern stays valid as the
example repo evolves.

A style doc stands on its own principle. Cross-references to
a downstream `ARCHITECTURE.md` for backing read as appeals to
authority that rot when the cited doc restructures; the
principle is what carries the doc.

---

## Adding new docs

When you research something non-obvious about these tools,
write it as a new topic file in the appropriate subdir. Keep
files small (~100 lines), one topic each, with the
frontmatter from `README.md` (`source:`, `fetched:`,
`trimmed:`). Prefer verbatim excerpts from upstream over
paraphrase.
