# Agent instructions — workspace-wide canonical contract

This file is the **canonical workspace contract**. Every per-repo
`AGENTS.md` is a thin shim pointing here; the rules below apply
to every repo under `~/git/`. Repo-specific carve-outs live in
that repo's own `AGENTS.md`.

---

## 🚨 Required reading, in order 🚨

1. **`INTENTION.md`** — what the project is for. Priorities, the
   deepest value, the intent that shapes every decision. Upstream
   of every other doc; if a downstream doc conflicts with
   INTENTION, the downstream doc loses.
2. **criome's `ARCHITECTURE.md`** (in the criome repo,
   `github:ligoldragon/criome`) — what the engine IS.
   Project-wide invariants, the request flow, the daemon shape.
   Required before touching any sema-ecosystem repo.
3. **This file** (`lore/AGENTS.md`) — how agents work in the
   workspace.
4. **The repo-specific `AGENTS.md`** of whatever repo you're
   editing — repo-role + carve-outs only.

A new agent entering the workspace reads (1) and (2) before
making any decisions; they are upstream of every rule below.

---

## AGENTS.md / CLAUDE.md pattern

Across all canonical repos: **`AGENTS.md` holds the real content**;
`CLAUDE.md` is a one-line shim reading
**"You MUST read AGENTS.md."**
This way Codex (which reads `AGENTS.md`) and Claude Code (which
reads `CLAUDE.md`) converge on a single source of truth. When
creating or restructuring a repo, keep this pattern.

Per-repo `AGENTS.md` itself opens with:
**"You MUST read [lore/AGENTS.md](path) — the workspace-wide
contract."** plus the repo's role and carve-outs.

---

## Per-repo `ARCHITECTURE.md` at root

Every canonical repo carries an `ARCHITECTURE.md` at its root
(matklad convention). The file is short — typically 50-150
lines — and answers: *what does this repo do, where do things
live in it, and how does it fit into the wider sema-ecosystem.*
Standard sections: role, owned-and-not-owned boundaries, code
map, invariants, status, cross-cutting context (prose pointer
to criome's `ARCHITECTURE.md` and any relevant report).

Per-repo `ARCHITECTURE.md` describes only that repo's niche.
Project-wide invariants live once, in criome's `ARCHITECTURE.md`;
per-repo files point at criome by prose. When a repo's role
changes, edit that repo's `ARCHITECTURE.md` and (if the change
is system-level) criome's `ARCHITECTURE.md`.

When creating a new canonical repo: write `ARCHITECTURE.md` at
root before the first commit.

---

## Inclusion scope — HARD

Edit only repos listed as CANON or TRANSITIONAL in
`workspace/docs/workspace-manifest.md`. Repos outside that list
are out of scope; their files stay untouched. To bring a new
repo into scope: add it to the manifest and `workspace/devshell.nix`,
write a report, commit.

---

## Documentation layers — strict separation

| Where | What |
|---|---|
| `criome/ARCHITECTURE.md` | **Project-wide canonical.** Prose + diagrams only. High-level shape, invariants, relationships of the engine being built. |
| `workspace/ARCHITECTURE.md` | **Dev environment.** Workspace conventions, role, layout. Points at criome for the project itself. |
| `<repo>/ARCHITECTURE.md` | **Per-repo bird's-eye view.** Repo's role, owned-and-not-owned boundaries, code map, status. Points at criome by prose for cross-cutting context. |
| `workspace/reports/NNN-*.md` | **Decision records + design syntheses.** Prose + visuals. |
| Each repo's source | **Implementation.** Rust code, tests, flakes, `Cargo.toml`. Type sketches as compiler-checked skeletons live here. |

When a layer rule is violated, rewrite: move type sketches out
of architecture docs into a report or skeleton code; move runnable
code out of reports into the appropriate repo. Architecture stays
slim so it remains readable in one pass.

Cross-references flow *into* architecture from reports. Reading
lists, decision histories, type-spec details live in reports or
in `workspace/docs/workspace-manifest.md`; criome's architecture
stays free of them.

---

## Build the durable shape — INTENTION applied

`INTENTION.md` commits the project to the **right
shape now**. The right shape now is worth more than a wrong
shape sooner; unbuilding a wrong shape costs more than the speed
it bought.

When work that depends on a durable shape would land before that
shape is ready, the work waits. The waiting is a feature of the
system — it preserves the chance to land the right shape on the
first try.

**The recognition heuristic.** When two paths appear — one
"small / hand-written / quick" and one "the durable shape" — the
durable shape is the answer; the work continues there. If the
durable shape is not yet ready, the dependent work parks until
the durable shape lands.

When deadline pressure pulls toward the small/quick path,
INTENTION wins (per INTENTION §"What we are not"). When this
rule meets any other rule, INTENTION wins (per INTENTION §"The
relationship with rules").

---

## Cross-references to workspace files

Reference workspace files by **prose**, not by deep github URL.

Cross-repo file URLs of the form
`https://github.com/LiGoldragon/<repo>/blob/main/<path>` are
brittle: when files move, get renamed, or are deleted, the URL
silently breaks and the reader follows it into a 404.

Use one of these instead:
- **Prose** — "criome's `ARCHITECTURE.md`" or "lore's `programming/abstractions.md`."
  The reader is in the workspace and can navigate.
- **Repo-level pointer** — `github:ligoldragon/<repo>` (nix-flake
  notation) for "this repo exists at that URL." It points at the
  repo, not a specific file inside it; renames within don't break it.

Inside a single repo, use a markdown link with a relative path —
those resolve in editors and on github regardless of the repo's
location.

---

## Positive framing only

State what IS. Architecture docs and agent rules describe the
system's commitments — what kinds exist, what owns what, what
flow happens, what the agent does. The current shape lives in
the doc; the path that led there lives in git/jj history.

When a direction turns out to be wrong, the doc is rewritten to
state the new direction. The previous direction disappears from
the doc; the commit history preserves it for anyone who needs to
recover the path.

When an option is excluded — by constraint, preference, or
decision — the criterion that excludes it is stated as a positive
requirement. "Must be Rust" replaces "Go is excluded." The
candidates that satisfy the criteria appear in the doc; the
others stay silent.

Each architectural commitment lives once, positively, in the
appropriate canonical doc — criome's `ARCHITECTURE.md` for
project-wide invariants, per-repo `ARCHITECTURE.md` for that
repo's role, this file for cross-project agent rules.

---

## Components, not monoliths

The workspace is composed of **micro-components** — one capability
per crate, per repo, per protocol. Each lives in its own repo
with its own `Cargo.toml`, `flake.nix`, and tests; each fits in a
single LLM context window; each speaks to its neighbors only
through typed protocols.

**Adding a feature defaults to a new crate, not editing an existing
one.** The burden of proof is on the contributor (human or agent)
who wants to grow a crate. They must justify why the new behavior
is part of the *same capability* — not a new one. The default
answer is "new crate."

**criome communicates.** Effect-bearing work — spawning
subprocesses, writing files outside sema, invoking external
tools, code emission — is dispatched as typed verbs to dedicated
components (forge, arca-daemon, prism). Each communicator
component stays a communicator; each executor component stays an
executor.

Full case (historical canon, the catastrophic record of monolith
collapse, the modern LLM-context argument):
`programming/micro-components.md`.

---

## Beauty is the criterion

Read `programming/beauty.md`. Beauty is
the test of correctness — **ugly code is evidence that the
underlying problem is unsolved**.

The aesthetic discomfort *is* the diagnostic reading. When
something feels ugly, slow down and find the structure that makes
it beautiful — that structure is the one that was missing.

---

## Verb belongs to noun (thinking discipline)

Read `programming/abstractions.md`
before writing free functions. The discipline applies to any
language with method dispatch: behavior lives on types, not as
floating verbs. The rule's purpose is to force the question
"what type owns this verb?" — when the answer isn't obvious, the
model of the problem isn't fully formed yet, and slowing down to
find the noun is the load-bearing cognitive event.

The rule has a sharper version: the verb belongs to **the right**
noun, not just any nearby noun. Adjacency of *types* is not the
same as adjacency of *concerns* — see
`programming/abstractions.md`
§"The wrong-noun trap."

---

## Producers push, consumers subscribe

Read `programming/push-not-pull.md`.
Producers push state; consumers subscribe to a stream. The
subscription primitive is the contract. When the primitive
doesn't yet exist, the dependent real-time feature waits for it
to land.

---

## Naming — full English words

Read `programming/naming.md`. Identifiers
are read far more than they are written. Cryptic abbreviations
optimize for the writer (a few keystrokes saved) at the reader's
expense (one mental lookup per occurrence).

**Default: spell every identifier as full English words.**

The canonical rule + the full table of common offenders + the
six permitted exception classes + the "feels too verbose"
anti-pattern all live in `programming/naming.md`.

---

## Binary naming — `-daemon` suffix, full English

Long-running daemon binaries carry the `-daemon` suffix:
`nexus-daemon`, `criome-daemon`, `forge-daemon`. The library half
of the same crate keeps the bare name (`nexus`, `criome`, `forge`)
— `[lib] name = "nexus"` and `[[bin]] name = "nexus-daemon"` in
the same `Cargo.toml`. CLI binaries that the user invokes
interactively keep bare names (`nexus`, `criome`).

**Why:** `nexusd` / `criomed` are unix-folklore abbreviations
carrying no information beyond the suffix. `nexus-daemon` reads
as English; `nexusd` requires expansion. It also disambiguates
the daemon binary from the CLI at PATH level. Per-process stderr
log tags also use the suffix (`nexus-daemon: ready`).

---

## One-shot binaries — `<crate>-<verb>`

Several crates ship thin one-shot binaries that wrap a single
verb of the library code as a stdin/stdout filter — useful for
test pipelines, agent harnesses, debug scripts.

Naming convention: `<crate>-<verb>`, ~30 LoC each.

| Binary | What it does |
|---|---|
| `nexus-parse` | nexus text from stdin → length-prefixed signal Frames to stdout |
| `nexus-render` | length-prefixed Frames from stdin → rendered text to stdout |
| `criome-handle-frame` | reads a Frame, dispatches via `Daemon::handle_frame` against `$SEMA_PATH`, writes reply Frame |

These expose useful affordances of the library code as standalone
tools — same shape as `gcc -E` shipping alongside `gcc`. They
live as additional `[[bin]]` entries in the existing crate, not
in a separate test-tools crate.

When to add a new one: when a verb of the library code is
genuinely useful as a filter primitive. Each new one earns its
keep through a concrete pipeline use-case.

---

## Docs describe the present

Vision, design, and architecture docs describe what the system IS
and what it's heading toward. They live in the present tense.

The lineage is preserved by git/jj history. When a piece of
context truly matters (e.g. "the validator pipeline is
Datomic-inspired"), state the fact and cite the inspiration as
present-tense influence.

How to apply:
- When rewriting a stale doc, extract the durable ideas and
  recast in current terms. Drop meta-commentary about the rewrite
  itself.
- Commit messages of substance describe the change in present
  tense — what the new state is.
- Project framing inside docs is "criome" — version markers
  (`criomev3`, `v1`, `v2`) are bookkeeping for agents, kept out
  of published docs.

---

## Design reports — visuals, not code

Reports in `workspace/reports/` explain, propose, analyse, or
summarise. Their medium is **prose + visuals** — ASCII diagrams,
swimlanes, flowcharts, tables, dependency graphs. Implementation
code (Rust `impl` blocks, function bodies, struct definitions
with methods, full Nix derivations) belongs in skeleton code in
the relevant repo, not in a report.

**Why:** code in a design doc goes stale the moment it lands and
the real type drifts; readers can't tell whether the report's
snippet or the repo's actual type is authoritative; visual shapes
carry the same information without that freshness trap.

**Test:** if the report has more than a couple of lines that look
like Rust / Python / Nix implementation, refactor those into a
visual.

**The narrow allowance:** a few-line *sample* of the surface the
design talks about — a snippet of a config file showing its
shape, a one-line CLI invocation, a single field declaration to
anchor a name — is fine. The rule is about implementation blocks,
not about showing the shape of the thing the design is about.

---

## Session-response style — substance goes in reports

If the agent's final-session response would be more than very
minimal (a few lines), write the substance as a report (in
`workspace/reports/`) and keep the chat reply minimal — a one-line
pointer at the report. Two reasons: (1) the Claude Code UI is a
poor reading interface; files are easier; (2) the author reviews
responses asynchronously while the agent moves to next work, so
the substance must be in a stable, scrollable, file-backed place.

Small reports are fine — the report doesn't have to be large.
Acknowledgements, tool-result summaries, "done; pushed"
confirmations don't need reports. Anything that explains,
proposes, analyses, or summarises does.

**Use relative paths in reports.** When a report references files
in sibling repos, link via `../repos/<name>/...` (the workspace
symlinks). The relative path resolves in editors and stays valid
across repo renames.

---

## Verify each parallel-tool result

When batching tool calls (parallel `Write`, `Edit`, `Bash`), scan
each result block for errors **before** any follow-up step that
depends on the results. The bundle returning is not the same as
the bundle succeeding.

Failure mode: failed `Write` calls (typically the "must Read
first" guard) don't show up in subsequent state until something
else reads the file. By then the failure has cascaded — a wrong
commit, a misled subagent, a published doc that doesn't match
what the code reflects.

How to apply:
- After any parallel batch, look at every `<result>` block. If
  any says "File has not been read yet" or "Exit code N" or
  `<tool_use_error>`, fix that *before* moving on.
- Double-check by reading the file or running `git status` / `ls`
  if a write was meant to land.
- Especially load-bearing when the next step is committing,
  pushing, or spawning an agent that consumes the written file.
- When a `Write` fails on the must-Read-first guard, do the Read
  then redo the Write before continuing.

---

## Version control: always push

After every change in any repo under `~/git/`, push immediately.
**Blanket authorization** — proceed without asking for confirmation.

```
jj commit -m '<msg>' && jj bookmark set main -r @- && jj git push --bookmark main
```

Push per logical commit. Unpushed work is invisible to nix
flake-input consumers and to other machines.

Use `jj` for all VCS interactions.

---

## Implicit authorization for missing publish steps

When the user points out that an expected external side effect is
missing, and that side effect is the obvious intended completion of
the work just done, treat that statement as authorization to perform
the missing publish step immediately.

Canonical cases:

- "I don't see the repo on my GitHub" → create/push the repo if the
  work was "make a new repo".
- "The PR isn't open" → open the PR if the work was "publish this
  branch".
- "I don't see the branch / commit / issue on the remote" → push the
  missing VCS or bd remote state.

The rule is about **obvious completion of already-requested work**,
not about inventing unrelated side effects. Apply it when the missing
step is the direct publish/export/remote-materialization leg of the
task already in progress.

---

## Commit message style

Single line. Short. A short verb + scope, plus an optional short
clause naming the change. The repo is implicit (the commit is in
the repo). Detail lives in the diff and the report.

Examples:

- `Slot<T> migration`
- `report add 119`
- `reader for typed slots`
- `AGENTS commit-style shortened`

If a single change touches multiple repos, each repo gets its
own short commit.

---

## Tooling — bd

`bd` (beads) tracks short items (issues, tasks, workflow). Designs
and reports go in files. See
[bd vs files — when each is the right home](bd/basic-usage.md#bd-vs-files--when-each-is-the-right-home).

`bd prime` auto-runs at session start and gives current state.

---

## Tool basics

Curated daily-use docs for the tools Li works with. Read these
before assuming how a tool works — they're trimmed to what
actually matters for our workflow, based on verified upstream
sources.

**VCS / build:**
- `jj/basic-usage.md` — jj (Jujutsu) daily
  loop: `@` model, commit/describe distinction, push, undo,
  bookmarks, remotes.
- `nix/basic-usage.md` — Nix flakes +
  home-manager: blueprint layout, lock-side pinning, devshells,
  `nix log`, home-manager primitives.

**Programming discipline (cross-language):**
- `programming/beauty.md` — Beauty as the
  criterion of correctness.
- `programming/abstractions.md` —
  Where behavior lives. Every reusable verb belongs to a noun.
- `programming/push-not-pull.md` —
  Producers push, consumers subscribe.
- `programming/micro-components.md`
  — One capability, one crate, one repo.
- `programming/naming.md` — Full English
  words.

**Language style:**
- `rust/style.md` — Rust object style: methods on
  types, typed newtypes, single-object I/O at boundaries, typed
  `Error` enum per crate.
- `rust/nix-packaging.md` — Canonical crane
  + fenix flake layout for Rust crates.
- `rust/ractor.md` — Ractor 0.15 in CANON daemons:
  per-file four-piece template, per-verb typed messages.
- `rust/rkyv.md` — Rkyv 0.8 portable feature set,
  derive-alias pattern, encode/decode API, schema-fragility limit.

**Data / issues:**
- `bd/basic-usage.md` — bd (beads) daily loop.
- `dolt/basic-usage.md` — Dolt SQL daily loop.

**CLI tools** (on PATH via the home profile):
- `annas/basic-usage.md` — Anna's Archive
  CLI.
- `linkup/basic-usage.md` — Linkup
  web-search CLI.
- `substack/basic-usage.md` — Publish
  Markdown to Substack.

---

## Style docs describe patterns

Docs under `programming/` and language-style docs under
`<language>/` describe **patterns**. The pattern is what's
load-bearing; concrete examples (actor counts, file paths,
function names) are illustrative-now, kept to one-line "see for
current shape" pointers. The pattern stays valid as the example
repo evolves.

A style doc stands on its own principle. Cross-references to a
downstream `ARCHITECTURE.md` for backing read as appeals to
authority that rot when the cited doc restructures; the principle
is what carries the doc.

---

## Adding new docs

When you research something non-obvious about these tools, write
it as a new topic file in the appropriate subdir. Keep files
small (~100 lines), one topic each, with the frontmatter from
README.md (`source:`, `fetched:`, `trimmed:`). Prefer verbatim
excerpts from upstream over paraphrase.
