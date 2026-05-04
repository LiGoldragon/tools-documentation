---
source: our own — derived from /tmp/gascity/{docs,examples,internal} and field operation 2026-05-01
fetched: 2026-05-01
scope: how to operate a Gas City — start/stop, attach to agents, talk to them, recover from stuck states. Not architecture (see workspace/reports/126).
---

# gas-city basic usage

## The shape

A city is a directory (e.g. `~/Criopolis`). When it's running:

- **One tmux server per city**, with socket name = city name. So
  `tmux -L Criopolis ls` shows every agent's tmux session.
- **One Unix supervisor** (`gascity-supervisor.service`) running all
  cities you've started. Per-user systemd service.
- **One dolt sql-server** per city, holding the beads database.
- **One controller-dispatcher** session per city — gas-city's internal
  bookkeeping pane.
- **Agent sessions** — mayor (always-on) plus on-demand workers that
  spawn when slung to.

## Default cwd assumption

Every command below assumes one of:

```bash
cd <city-dir>                      # easiest in ad-hoc shells
export GC_CITY=<city-dir>          # set once per shell
alias pc='gc --city <city-dir>'    # least typing
```

All examples below use `gc ...`; substitute your preferred form.

## Start, stop, status

```bash
gc start                # bring city up (idempotent; no-op if already up)
gc stop                 # bring city down (preserves beads)
gc status               # one-screen overview of agents + sessions
gc reload               # reload pack/city config without full restart
```

## Talking to agents

The mental model: each agent runs in its own tmux session inside the
city's tmux server. You're driving an LLM through a tmux pane.

```bash
gc session list                # see which agents have live sessions
gc session attach <name>       # take over a tmux pane (Ctrl-b d to detach)
gc session peek <name>         # last 80 lines of scrollback (no attach)
gc session logs <name> -f      # follow conversation transcript
gc session nudge <name> "text" # type into the pane without attaching
gc session kill <name>         # drop the session; controller respawns if always-on
```

## Switching between agents inside tmux

When you're attached to one agent (e.g. mayor) and want to look at
another without leaving the tmux server:

| Inside tmux | What it does |
|---|---|
| `Ctrl-b s` | list all sessions — arrow keys + Enter to switch |
| `Ctrl-b (` | previous session |
| `Ctrl-b )` | next session |
| `Ctrl-b w` | windows-and-sessions tree |
| `Ctrl-b d` | detach back to your shell |

The same effect from a separate terminal:

```bash
gc session attach <name> --city <city-dir>
```

You can have one terminal attached to mayor and another to aesthete
simultaneously — they're independent tmux clients on the same tmux
server. **Detach is `Ctrl-b d` always**; never `exit` (which kills
the LLM process).

## Sending work without attaching

```bash
# Quick one-shot to the mayor:
gc sling mayor "Should we use static typing in this codebase?"

# The full debate workflow (frame → 4 seats → synthesis):
gc order run daily-debate --var topic="..."

# Async message that survives session restart:
gc mail send <agent> -s "subject" -m "body"
gc mail inbox <agent>
```

`sling` creates a bead routed to the named agent. The agent finds it
on its next hook check and acts on it. `mail` is conversational
context. `nudge` types directly into the pane (use sparingly — it
bypasses the bead/work flow).

## Beads (work + memory)

```bash
bd ready                         # work eligible to start (the agent's view)
bd list --status=open            # everything open
bd show <id>                     # details
bd close <id>                    # mark done
bd remember "note worth keeping" # persistent memory
bd memories <keyword>            # search memories
```

## Watching what's happening

```bash
gc status
gc session list
gc session logs mayor -f
bd list --status=open
gc supervisor logs -n 50         # what the systemd service has been doing
gc trace                         # reconciler decisions (debug; verbose)
```

## Recovery: dolt lock stuck

Symptom: `gc start` reports *"could not acquire dolt start lock"* and
never recovers. Caused by stale state files when a prior start
crashed without cleanup.

```bash
gc stop                                     # ignore errors

# Kill leftovers
pgrep -af "gc-beads.*<city-name>|dolt sql-server.*<city-name>" \
  | awk '{print $1}' | xargs -r kill
sleep 1

# Clear stale state
rm -f <city-dir>/.beads/dolt-server.{pid,port,lock,log}
rm -f <city-dir>/.beads/dolt/.dolt/noms/LOCK
rm -f <city-dir>/.gc/runtime/packs/dolt/dolt.lock

# Hand-start dolt; supervisor will adopt it
cd <city-dir>
GC_CITY_PATH=$PWD bash .gc/system/packs/bd/assets/scripts/gc-beads-bd.sh start

# Now start the city normally
gc start
```

This is an upstream bug in gas-city's bd-start script
(`wait_for_concurrent_start_ready` doesn't `kill -0` the stored pid
before waiting). Bounded: it bites on recovery paths, not steady
state.

## Editing a live city

| What you changed | What to run |
|---|---|
| Prompt template (`agents/<name>/prompt.template.md`) | `gc reload` |
| `pack.toml` agent fields (effort, provider, prompt path) | `gc reload` |
| `city.toml` daemon settings | `gc reload` |
| Runtime provider for an agent (rare) | `gc stop && gc start` |

If `gc reload` returns "controller busy", wait 30s and retry. Stays
busy more than a couple minutes → cycle the city.

## Persistence across logout

`systemctl --user` services stop at logout by default. To keep cities
running across logout/reboot:

```bash
sudo loginctl enable-linger <user>
```

Tradeoff: gc/dolt run 24/7 even when idle (free in tokens; cheap in
RAM; mildly wasteful on a laptop on battery).

## When confused

```bash
gc <subcommand> --help     # every subcommand has --help
gc doctor                  # health check across deps and state
```
