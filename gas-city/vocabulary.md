---
source: our own — derived from /tmp/gascity/{docs,examples,internal,AGENTS.md} and field operation 2026-05-01..02
fetched: 2026-05-02
scope: every term gas-city uses, organized by layer. Definitions only — for *how to operate*, see basic-usage.md.
---

# gas-city vocabulary

Use this as a glossary. Layers go from biggest (city) to smallest
(bead), then provider/runtime, then philosophy.

---

## Spatial structure

| Term | What it is |
|---|---|
| **city** | A directory on disk. Contains `city.toml` + `pack.toml` + `agents/` + `.gc/` runtime state. The unit you `gc start` and `gc stop`. |
| **pack** | A reusable, portable bundle of agents/formulas/orders/scripts. Lives at the city root or under `packs/<name>/`. Cities import packs via `[imports.<binding>]`. Same pack can be used by many cities. |
| **rig** | An external project (repo) registered with the city via `gc rig add <path>`. Each rig gets its own beads database and ID prefix. A city can have zero rigs (e.g. philosophy-city). |
| **workspace** | The `name` field of the city, used for routing and display. |
| **builtin pack** | Packs that ship inside the `gc` binary and are materialized into `.gc/system/packs/<name>/` at start. The current set: `bd` (beads exec provider) and `dolt` (managed dolt server). Not user-editable. |

## Configuration files

| File | Role |
|---|---|
| `pack.toml` | The portable layer — agent definitions, formulas, named sessions. Tracked in git, travels with the pack. |
| `city.toml` | The deployment layer — provider preset, daemon settings, rigs, patches, beads backend. Local to this machine. |
| `.gc/site.toml` | Machine-local rig path bindings. |
| `.gc/settings.json` | Provider-side hooks (Claude Code, Codex, etc.) that run on each agent turn. |
| `.gc/events.jsonl` | Append-only event-bus log. |
| `.gc/runtime/` | Runtime helper data (controller socket, dolt config, lock files). |
| `.gc/system/packs/` | Materialized builtin packs. Re-written on each `gc start`. |
| `.beads/` | Per-city or per-rig beads database (dolt-backed). |

## Process model

| Term | What it is |
|---|---|
| **controller** | The per-city reconciliation loop. Reads `city.toml` + beads, compares to live process state, spawns / kills / restarts sessions. Owned by the supervisor. |
| **supervisor** | A machine-wide `systemctl --user` service (`gascity-supervisor.service`) that runs zero or more cities. One supervisor per user. |
| **session** | A live process — typically a tmux pane running an agent CLI (`claude`, `codex`, `gemini`, …). The unit the controller tracks. |
| **session ID** | A short pc-XXX identifier. Also lives in beads as a `type=session` bead so the controller can persist session metadata. |

## Agents

| Term | What it is |
|---|---|
| **agent** | A configured entity: name + prompt template + provider + scaling rules. Pure config; no Go code defines roles. |
| **named session** | A `[[named_session]]` declaration that pins an agent template to a long-lived (or on-demand) session. `mode = "always"` keeps it alive; `mode = "on_demand"` spawns when slung to. |
| **pool** | An agent with `max_active_sessions > 1`. The reconciler scales sessions up/down based on routed work. |
| **provider preset** | A `[providers.<name>]` block plus the SDK's built-in defaults. Eight providers ship in the binary: `claude, codex, gemini, opencode, copilot, cursor, pi, omp`. Per-agent `provider = "..."` selects one. |
| **runtime provider** | The transport that spawns sessions: `tmux` (default), `subprocess`, `exec`, `acp`, `k8s`, `hybrid`, `auto`, `fake`. Set via `[session]` in `city.toml`. |
| **option_defaults** | Per-agent overrides for the provider's flag-mapped options. Common keys: `effort`, `model`, `permission_mode`. |
| **prompt template** | A Markdown file at `agents/<name>/prompt.template.md`, optionally Go `text/template`. Delivered to the agent CLI on session start via the provider's prompt-mode (arg / flag / none). |
| **fragment** | A `{{ define "name" }} … {{ end }}` block in `template-fragments/<file>.template.md`. Pulled into prompts via `inject_fragments` / `append_fragments` / `global_fragments`. |
| **global_fragments** | Workspace-wide fragments appended to every agent's prompt. Common ones: `command-glossary`, `operational-awareness`. |

## Work — the bead graph

| Term | What it is |
|---|---|
| **bead** | The universal unit. Every primitive — task, message, session, molecule, wisp, convoy, memory — is a bead with a `type` field. |
| **task bead** | The default — a unit of work (`type=task`). |
| **message bead** | `type=message`. What `gc mail send` creates. Hooks inject unread mail into the agent's next turn. |
| **session bead** | `type=session`. Persists session metadata (pid, port, lifecycle markers). The controller writes these. |
| **memory bead** | What `bd remember "…"` creates. Persistent, searchable across sessions via `bd memories <keyword>`. |
| **molecule** | A root bead (`type=molecule`) plus child step beads via parent-id linkage. Persistent workflow. Created by `gc formula cook`. |
| **wisp** | An ephemeral molecule. Garbage-collected by the `wisp-compact` order when closed. Created by `gc sling … --formula`. |
| **convoy** | A bead grouping — siblings under a common parent. `gc sling` auto-creates a convoy per slung-to agent. `gc convoy create` does it manually. |
| **formula** | A TOML-defined workflow at `formulas/<name>.toml`. Steps with `needs = […]` form a DAG. Schema v2 is the canonical form (DAG); v1 (tree-only) exists for legacy. |
| **order** | A trigger config at `orders/<name>.toml`. Five trigger kinds: `cooldown`, `cron`, `condition`, `event`, `manual`. Body is either `formula = "..."` (LLM-driven) or `exec = "..."` (shell-only). |
| **work query** | The shell command an agent runs on each turn to find work. Default 3-tier: in-progress assigned → ready assigned → ready routed-to-me. Override via `[[agent]] work_query`. |
| **routed bead** | A bead with `gc.routed_to=<agent-name>` metadata. The default sling sets this. The agent's work_query filters on it. |
| **claim** | Atomic ownership transfer: `bd update <id> --claim` flips the bead to in-progress and assigns it to the running session. |

## Verbs you'll use

| Verb | What it does |
|---|---|
| **sling** | `gc sling <agent> "task description"` — create a bead routed to that agent, auto-convoy it, optionally wrap in a formula. The default dispatch primitive. |
| **mail** | `gc mail send / inbox / read / reply / archive` — create / consume message beads. Async; survives session restart. |
| **nudge** | `gc session nudge <name> "text"` — type literal characters into a live tmux pane. Bypasses beads. Use only for live debugging of a session you're watching. |
| **prime** | `gc prime [agent]` — print the effective prompt for an agent. The output is what got delivered to the CLI on its session start. Used by Claude Code's startup hook. |
| **handoff** | `gc handoff "summary" "context"` — send mail to yourself + restart your session so the next incarnation has the context. |
| **cook** | `gc formula cook <name>` — materialize a formula as a persistent molecule (all steps as beads). |
| **dispatch** / **route** | Synonyms for sling, used in docs and error messages. |
| **adopt** | What the supervisor does when it finds an existing-but-unowned process and starts treating it as one of its sessions. |

## Lifecycle states

| State | Meaning |
|---|---|
| **reserved-unmaterialized** | Configured but no session yet. Typical for `mode = "on_demand"` agents. |
| **creating** | Session being spawned (tmux setup, hook install, prompt delivery). |
| **active** | Session is running and the controller has confirmed it. |
| **asleep** | Session is alive but suspended (idle / waiting for work). Different from `suspended` (config-level). |
| **awake** | Session is actively processing. `awake (always)` for `mode=always` agents. |
| **running** / **stopped** | Pool member states for scale-managed agents. |
| **suspended** (config) | `[[agent]] suspended = true` — reconciler ignores this agent. Slings to it warn but never spawn a session. |
| **orphaned** | Session marker exists but the live process is gone. Cleaned up by the orphan-sweep order. |

## Built-in roles (Gastown pack — not the SDK)

These are role names defined in `examples/gastown/packs/gastown/`. They're prompt-template + agent-config patterns, not SDK primitives.

| Role | What its prompt does |
|---|---|
| **mayor** | Coordinator. Files beads, dispatches to polecats, synthesizes. `mode = "always"`. |
| **polecat** | Worker. Pulls a bead from a pool, sets up a worktree, codes, commits, hands off to refinery. `wake_mode = "fresh"`. |
| **refinery** | Merger. Takes polecat output, rebases, runs tests, merges or rejects. |
| **witness** | Watchdog. Detects orphaned beads, stuck polecats, unresponsive sessions. Recovers work. |
| **deacon** | Patrol orchestrator. Triggers periodic gate-sweep, dispatches digest generation. |
| **boot** | Crash watchdog. Restarts dead infrastructure. |
| **crew** | Persistent peer (used in the `swarm` example). Self-coordinates with other crew via mail. |
| **dog** | Utility worker. Runs short mechanical tasks routed to it; usually paired with `mol-dog-*` formulas. |
| **committer** | Git-only worker that takes finished work and pushes (used in `swarm`). |

The SDK ships *zero* hardcoded references to any of these names. If you find `if role == "mayor"` in Go, that's a bug.

## Hooks

| Term | What it is |
|---|---|
| **hook** | A shell command provider-side that runs on each agent turn. Installed by `gc init` based on `install_agent_hooks`. For Claude Code: `.gc/settings.json` with hook entries. |
| **mail hook** | Injects unread message beads as a system reminder before the model's next turn. |
| **work hook** | Runs the agent's `work_query` and surfaces ready work as a system reminder. |
| **prime hook** | Re-delivers the agent's prompt after `/clear` or compaction in Claude Code (the `gc prime` mechanism). |

## Provider concepts (the eight built-ins)

Set per-agent with `provider = "..."`. Each provider wraps a CLI; gas-city assembles the actual command line.

| Provider | CLI it wraps | Notes |
|---|---|---|
| **claude** | `claude` (Claude Code) | Default. Uses `--dangerously-skip-permissions` and `--effort max` by default. |
| **codex** | `codex-raw` (ChatGPT Codex CLI) | Default `--dangerously-bypass-approvals-and-sandbox`, `model_reasoning_effort=xhigh`. |
| **gemini** | `gemini` (Google Gemini CLI) | Permissive default. |
| **opencode** | `opencode` | OpenCode CLI. |
| **copilot** | `copilot` | GitHub Copilot CLI. |
| **cursor** | `cursor` | Cursor CLI. |
| **pi** | `pi` (badlogic/pi-mono) | Coding-agent CLI built on tool-loop. |
| **omp** | (proprietary) | Internal/private. |

Provider option keys exposed via `option_defaults`: `effort`, `model`, `permission_mode`. The valid values per provider live in `internal/worker/builtin/profiles.go`.

## Philosophy abbreviations

| Term | Meaning |
|---|---|
| **ZFC** | *Zero Framework Cognition*. No judgment calls in Go. If a Go line decides "should I retry this?" it's a violation — move it to the prompt. |
| **GUPP** | *the Universal Propulsion Principle*. "If you find work on your hook, YOU RUN IT." No confirmation, no waiting. The presence of work IS the assignment. Encoded in agent prompts via fragments, never enforced by Go. |
| **NDI** | *Nondeterministic Idempotence*. Persistent beads + hooks + redundant observers + idempotent steps = reliability. Sessions die; the store is forever. |
| **Bitter Lesson** | (Sutton) Every primitive must become more useful as models improve. Don't build heuristics or decision trees — bake general computation, not human-encoded rules. |
| **MEOW stack** | *Molecular Expression of Work*. The bead → molecule → formula tower that makes work itself the orchestration primitive. |

## CLI command surface (compact)

The verbs you'll reach for, grouped:

**City lifecycle**
```
gc init / start / stop / reload / status / doctor / supervisor / trace
```

**Agents & sessions**
```
gc agent add / suspend / resume
gc session list / peek / attach / nudge / logs / kill / new
gc prime
```

**Beads & memory**
```
bd ready / show / close / list / update / dep / label
bd remember / memories / forget
bd dolt pull / push
gc bd  (rig-aware shorthand)
```

**Dispatch**
```
gc sling / mail / handoff
gc formula list / show / cook / compile
gc order list / show / check / run / history
gc convoy create / status / add
```

**Config inspection**
```
gc config show / explain / validate
gc rig add / list / status
```

## File/path quick reference

```
~/<city>/
├── city.toml                    # deployment config
├── pack.toml                    # portable definition
├── agents/<name>/
│   ├── prompt.template.md       # the model's behavior
│   └── agent.toml               # optional per-agent overrides
├── formulas/<name>.toml         # workflow DAGs
├── orders/<name>.toml           # triggered automation
├── commands/                    # custom CLI command scaffolds
├── doctor/                      # health-check scripts
├── overlays/                    # files copied additively into agent work_dir
├── template-fragments/<>.template.md
├── assets/                      # scripts, prompts, shared assets
└── .gc/                         # runtime state — git-ignored
    ├── site.toml                # rig bindings
    ├── settings.json            # provider hooks
    ├── events.jsonl             # event log
    ├── runtime/
    │   ├── controller.lock
    │   ├── controller.sock
    │   └── packs/dolt/dolt.lock
    ├── system/packs/{bd,dolt}/  # materialized builtins
    └── worktrees/<rig>/         # worktree-isolated agent sandboxes
```

## Common metadata keys

The `metadata` field on a bead is a free-form map. Conventions in use:

| Key | Set by | Meaning |
|---|---|---|
| `gc.routed_to` | sling | Target agent name (e.g. `aesthete`, `<rig>/polecat`). |
| `parent` | formula cooking | Parent bead id for molecule children. |
| `work_dir` | polecat (gastown) | Worktree path the agent operates in. |
| `branch` | polecat | Source feature branch. |
| `pr_url` | refinery | Canonical PR URL after handoff. |
| `rejection_reason` | refinery | Why a merge was rejected; bead returned to pool. |

Custom keys are fine — `bd update <id> --set-metadata key=value`.

## Honourable mentions

- **`gc.routed_to`** — the convention that makes sling work. If a bead has it, the named agent's default work_query will pick it up.
- **`mol-`** prefix on formulas — gas-city convention for molecule-producing formulas (`mol-do-work`, `mol-polecat-base`, `mol-dog-doctor`).
- **`hq-`** prefix — Gastown-style convention for city-level (no-rig) beads. The philosophy-city uses the workspace's auto-prefix `pc-`.
- **`gc trace`** — verbose reconciler decision log. For debugging "why didn't my session restart". Don't reach for it until you need it.
- **`gc doctor`** — health check across deps, config, supervisor, hooks. Run after any environmental change.
