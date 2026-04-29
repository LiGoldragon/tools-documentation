---
source: (our own)
fetched: 2026-04-29
---

# Micro-components — one capability, one crate, one repo

Every functional capability — state engine, code emitter, executor,
store, parser, schema, transport — lives in its own independent
repository with its own `Cargo.toml`, `flake.nix`, and test suite.
Components communicate only through typed protocols, never shared
mutable state. Each component is sized so that the *entire
component, including tests*, fits comfortably in a single LLM
context window.

The discipline is **source-organization, not deployment**:
components may compile into one binary, many binaries, or talk
over a network. The workspace is the assemblage; no individual
component knows about more than its protocol-typed neighbors.

This doctrine is the only known antidote to the failure mode it
closes: **agents and humans bundling new features into existing
crates** until the result is a monolith no one — including the
language model assisting them — can hold in mind.

## The rule

1. **One capability, one crate, one repo.** If you can name the
   new functionality with a noun, it gets its own `Cargo.toml`
   and its own git history — never a new `mod` in an existing
   crate.

2. **A component must fit in a single LLM context window.**
   Roughly: a crate of ~3k–10k lines (~30k–80k tokens including
   tests) can be reasoned about end-to-end. Above that ceiling,
   split. This is not aesthetic — it is the operational gate
   for AI-assisted editing.

3. **Components communicate only through typed protocols.** No
   shared mutable state, no leaked internals via `pub use`, no
   cross-crate `unsafe`. The protocol *is* the contract; the
   type-checker enforces it.

4. **Every component is independently buildable, testable, and
   replaceable.** `cargo build` and `nix flake check` must
   succeed inside the component's own repo with no
   workspace-level helpers. If they don't, the boundary is a
   fiction.

5. **Depend on protocols, not implementations.** A consumer
   crate names the trait/schema crate, never the engine crate.
   This is what makes a component swappable without touching
   its callers.

6. **Adding a feature defaults to a new crate, not editing an
   existing one.** The burden of proof is on the contributor
   (human or agent) who wants to grow a crate. They must
   justify why the new behavior is part of the *same
   capability*, not a new one. The default answer is "new
   crate."

7. **No component owns more than one bounded context.** When
   the ubiquitous language inside a crate starts using two
   vocabularies — "session" meaning two different things, or
   "build" used for both the verb and the artifact — the crate
   must split along that seam.

## Why

The literature on monolith collapse (Parnas 1972 onward)
converges on five structural failures, each closed by
per-capability decomposition:

- **Cognitive load.** No developer holds the whole picture;
  changes are made on partial mental models. Per-capability
  components ensure no one needs to.
- **Change blast radius.** A fix in module A breaks module Z
  because a hidden coupling existed. (Parnas, *On the Criteria
  To Be Used in Decomposing Systems into Modules*, 1972 — the
  foundational paper. Information hiding is the only known
  antidote.)
- **Dependency knots.** Circular and transitive dependencies
  make build/test order brittle. Independent crates make
  cycles a compile error, not a runtime bug.
- **Deployment coupling.** In a monolith, one bug blocks all
  releases. Even when components compile into a single
  binary, the *source* boundary keeps each capability
  releasable on its own schedule.
- **Test fragility.** Integration tests dominate monoliths;
  unit tests become meaningless because units are not
  isolated. Per-capability components have meaningful unit
  tests because the unit is the boundary.

The historical record is unambiguous. Twitter's Ruby monolith
became un-deployable; the eventual rewrite into JVM services
took years. Facebook's PHP monolith was so large the response
was to invent a new compiler (HHVM) rather than decompose.
Healthcare.gov collapsed at launch because integration was
discovered at launch time. Bank and government COBOL systems
persist because they cannot be modified — institutional
knowledge of the whole evaporates with retirements.

## The LLM-context argument

This is the new structural reason in 2024–2026. Frontier model
context windows are 200k–1M tokens. A monolith of millions of
lines simply cannot be loaded; the agent operates on partial
views and produces changes that violate invariants it cannot
see.

The fix is *not* a larger context window — codebases grow
faster than windows. The fix is **components small enough that
the whole component fits**.

Empirically, a Rust crate of ~3k–10k lines including tests
fits in ~30k–80k tokens, can be loaded in full, and can be
reasoned about end-to-end. This is the operational definition
of LLM-context-sized.

The historical accident is fortunate: McIlroy's 1978
Unix-philosophy crate-size advice and a 2026 frontier-model
context window converge on the same number.

## How

When you reach for a new feature:

1. **Name the capability as a noun.** If you can't, you don't
   yet understand what you're adding.
2. **Look at the existing crates.** Does the new noun *already
   match* an existing crate's stated capability? Only then add
   to that crate.
3. **Default to a new repo.** Cost: a `Cargo.toml`, a
   `flake.nix`, a few minutes of plumbing. Benefit: a
   permanent boundary the build system enforces.
4. **Define the protocol crate first** if the new component
   will have multiple consumers. Implementation crates depend
   on the protocol crate, not on each other.
5. **Each component carries its own `ARCHITECTURE.md`,
   `AGENTS.md`, and tests at its repo root.**

The boundary is filesystem-enforced; nothing else holds.
Module-level boundaries inside one crate decay under deadline
pressure into shared internals — the "modular monolith"
failure mode (Brown / Grzybek note this directly).

## Distinctions

- **Microservices** (Newman, 2015) — runtime processes
  communicating over a network. Different layer.
  Micro-components is *source organization* and is
  deployment-agnostic; the same components may link into one
  binary, many binaries, or talk over a network.
- **Microkernel** — OS design (Mach, L4, seL4). Different
  domain.
- **Modular monolith** — one deployable unit with internal
  modules. Right intent, wrong enforcement: without
  filesystem boundaries, "explicit module boundaries" decay.

The axis micro-components occupies and the others miss:
**source-level filesystem-enforced decomposition,
deployment-agnostic.**

## When you're tempted to grow a crate

Stop. Ask:

- Can I name this new behavior with a noun *distinct from*
  the crate's current capability?
- Would a fresh reader of the resulting crate think "this
  crate does one thing"?
- Does the new behavior introduce vocabulary the current
  crate doesn't already use?

If any answer is "yes," start a new crate. The cost is a few
minutes of plumbing; the cost of bundling is months or years
of future-friction that no LLM and no team will resolve
cleanly.

The Unix advice (McIlroy, 1978) and the modern AI-assisted-
development reality both point at the same shape: *small
components that compose*. There is no third path that scales.

## See also

- [`programming/abstractions.md`](abstractions.md) — every
  reusable verb belongs to a noun (the same discipline at the
  type level).
- [`programming/beauty.md`](beauty.md) — when a crate's
  structure feels ugly, the right decomposition usually
  hasn't been found yet.
- [`programming/push-not-pull.md`](push-not-pull.md) —
  components communicate via subscription primitives, not by
  polling each other.
