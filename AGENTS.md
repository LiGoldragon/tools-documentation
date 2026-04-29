# Agent instructions

## Version control: always push

After every change in any repo under `~/git/`, push immediately. **Blanket authorization** — do not ask for confirmation.

```
jj commit -m '<msg>' && jj bookmark set main -r @- && jj git push --bookmark main
```

Push per logical commit, not batched. Unpushed work is invisible to nix flake-input consumers and to other machines.

Use `jj`, not `git`, for all VCS interactions — never run `git commit` / `git add` / `git reset` directly.

## Tool basics

Curated daily-use docs for the tools Li works with. Read these before assuming how a tool works — they're trimmed to what actually matters for our workflow, based on verified upstream sources.

**VCS / build:**
- [jj/basic-usage.md](jj/basic-usage.md) — jj (Jujutsu) daily loop: `@` model, commit/describe distinction, push, undo, bookmarks, remotes.
- [nix/basic-usage.md](nix/basic-usage.md) — Nix flakes + home-manager: blueprint layout, lock-side pinning, devshells, `nix log`, home-manager primitives (env vars caveat for GUI apps).

**Programming discipline (cross-language):**
- [programming/beauty.md](programming/beauty.md) — **Beauty is the criterion.** Ugly code is evidence that the underlying problem is unsolved. Read this before pushing back on any rule as "verbose" or "ceremonial." Philosophical foundation, the practitioners' explicit defense, the catastrophic record of what happens when ugly engineering ships.
- [programming/abstractions.md](programming/abstractions.md) — Where behavior lives. **Every reusable verb belongs to a noun.** Affordance vs operation framing, the noun-creation forcing function, why this matters more for LLM agents than humans, principled exceptions. Read this before writing free functions.
- [programming/push-not-pull.md](programming/push-not-pull.md) — **Polling is wrong. Always.** Producers push, consumers subscribe. The subscription primitive is the contract; if it doesn't exist yet, defer the real-time feature rather than poll "for now." Two narrow look-alikes that aren't polling (reachability probes; backpressure-aware pacing).
- [programming/micro-components.md](programming/micro-components.md) — **One capability, one crate, one repo.** Every functional capability lives in its own repo with its own Cargo.toml, flake.nix, and tests; components communicate through typed protocols; each fits in a single LLM context window. Default to new crate, not editing existing. The criome workspace is the antithesis of monoliths — the failure mode this rule closes is agents bundling features into existing crates until the result is unmodifiable.

**Language style:**
- [rust/style.md](rust/style.md) — Rust object style: methods on types not free fns, typed newtypes for domain values, single-object I/O at boundaries, manual `Error` enums (no thiserror/anyhow), trait-domain rule, doc protocol.
- [rust/nix-packaging.md](rust/nix-packaging.md) — Canonical crane + fenix flake layout for any Rust crate: layered cargo-deps cache, toolchain pinned via `rust-toolchain.toml`, git-URL deps, workspace handling, `checks.default`.
- [rust/ractor.md](rust/ractor.md) — Ractor 0.15 in CANON daemons: per-file four-piece template (Actor / State / Arguments / Message), per-verb typed `RpcReplyPort<T>` messages, supervision, self-cast loops, sync façade. Grounded in the criome example.
- [rust/rkyv.md](rust/rkyv.md) — Rkyv 0.8 portable feature set across the workspace, derive-alias pattern when paired with serde, encode/decode API, type-adaptation gotchas (`PathBuf` etc.), schema-fragility limit + version-skew-guard discipline.

**Data / issues:**
- [bd/basic-usage.md](bd/basic-usage.md) — bd (beads) daily loop: init --skip-hooks --skip-agents, create/ready/close, dependencies, dolt push/pull.
- [dolt/basic-usage.md](dolt/basic-usage.md) — Dolt SQL daily loop: dual CLI+SQL interfaces, branch/commit/merge of tables, remotes + DoltHub, auth.

**CLI bundle tools** (all bundled via `mentci-tools.packages.<sys>.cli`, on PATH in the med-tier home profile):
- [annas/basic-usage.md](annas/basic-usage.md) — Anna's Archive CLI: `book-search`, `book-download <hash>`, `article-search`, `article-download <doi>`, `mcp` mode. Secret key via gopass.
- [linkup/basic-usage.md](linkup/basic-usage.md) — Linkup web-search CLI: `linkup search "<q>" [--depth standard|deep|fast]`, grounded answers with citations. API key via gopass.
- [substack/basic-usage.md](substack/basic-usage.md) — Publish Markdown posts to Substack: create/update/list/get/delete, image uploads, publication settings. Auth via `substack.sid` cookie env.

## Style docs are patterns, not snapshots

Docs under [`programming/`](programming/) and language-style docs under [`<language>/`](rust/) describe **patterns**. They must not embed specifics from any *current* downstream repo — no exact actor counts, no file-path listings, no concrete function or message-variant names that will rot as the example evolves. Reference an example repo as a one-line "see for current shape" pointer, not as authoritative structure the doc seems to claim is true forever.

The same applies to cross-citations: a style doc should not invoke a downstream `ARCHITECTURE.md` to backstop its claims. The principle stands on its own; appeals-to-authority across repo boundaries rot when the cited doc restructures.

Failure mode this closes: an agent reads "the canonical example has five actors / `src/foo.rs` / `handle_assert()`" or "[per X-repo's ARCHITECTURE §2]," treats those specifics as load-bearing, and propagates them into new code — even after the example has evolved or the cross-link has rotted. The pattern is what's load-bearing; the example is illustrative-now.

## Adding new docs

When you research something non-obvious about these tools, write it as a new topic file in the appropriate subdir. Keep files small (~100 lines), one topic each, with the frontmatter from README.md (`source:`, `fetched:`, `trimmed:`). Prefer verbatim excerpts from upstream over paraphrase.
