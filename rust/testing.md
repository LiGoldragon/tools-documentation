---
source: (our own) — distilled from working tests and the active Kameo actor discipline
fetched: 2026-05-11
---

# Rust testing — patterns

How we test Rust code in this workspace. The discipline lives
in `~/primary/skills/rust-discipline.md`; this file is the
toolchain reference for the patterns: where tests live, what
they exercise, how they run, what's worth mocking and what
isn't.

## The principle - direct domain methods on the actor

The cleanest unit for most actor tests is still the synchronous
domain method, but Kameo changes where that method lives:
`Self` is the actor. Put the real verb on the data-bearing actor
type, and let the async message handler be the thin shell that
calls it. **Tests call the actor's domain method directly - no
actor harness, no tokio runtime, no mocking - unless the test is
about mailbox, lifecycle, or supervision behavior.**

```rust
// production code - the data-bearing actor
pub struct Reconciler {
    phase: Phase,
    target: TargetPath,
}

impl Reconciler {
    pub fn apply(&mut self) -> Result<(), Error> {
        let result = self.apply_inner();
        self.phase = match &result {
            Ok(()) => Phase::Settled,
            Err(_) => Phase::Failed,
        };
        result
    }
}

pub struct RunReconciliation;

impl Message<RunReconciliation> for Reconciler {
    type Reply = Result<(), Error>;

    async fn handle(
        &mut self,
        _message: RunReconciliation,
        _context: &mut Context<Self, Self::Reply>,
    ) -> Self::Reply {
        self.apply()
    }
}
```

```rust
// tests/reconciler.rs - direct, no runtime
struct Fixture {
    target: TargetPath,
}

impl Fixture {
    fn apply(&self) -> Result<(), Error> {
        let mut reconciler = Reconciler::new(self.target.clone());
        reconciler.apply()
    }
}

#[test]
fn applies_a_proposal_to_a_target_file() {
    let fixture = Fixture { target: … };
    fixture.apply().unwrap();
    // assert the side effect on disk
}
```

This pattern works because the data-bearing actor is the noun and
`apply()` is the verb on it. The actor harness is only needed when
testing the message-passing, mailbox, lifecycle, or supervision
behavior itself. For everything else, test the domain method.

## Tests live in separate files

Unit tests do **not** go in a `#[cfg(test)] mod tests` block at
the bottom of the source file. They live in a sibling file
under `tests/` at the crate root, named for the module they
exercise.

```
src/
├── cert.rs
├── tree.rs
└── error.rs
tests/
├── cert.rs      # integration tests for Cert
└── tree.rs      # integration tests for Tree
```

This keeps the source file focused on behavior, lets the test
file grow without bloating the source, and forces tests to
exercise the public API (integration tests can't reach private
items — which is the right pressure: if something is hard to
test from outside, the API needs work, not the test).
Private-helper tests are rare and can go in a small
`tests_internal` module with a clear boundary; if you find
yourself reaching for many, that's a signal the helper wants
to be its own type with a public constructor.

**One test file per source file.** Don't collect tests from
multiple modules into a single `tests/common.rs` unless the
shared fixtures genuinely apply to more than one module.

## `nix flake check` is the canonical runner

Every Rust crate exposes its test suite as `checks.default` in
its `flake.nix`. `nix flake check` builds the crate and runs
`cargo test` inside a pure Nix sandbox, which:

- Pins the toolchain to the flake's `fenix` component — no
  host-rustc drift.
- Resolves dependencies from a committed `Cargo.lock` — no
  "works on my machine" gaps.
- Makes the test invocation self-documenting: any Nix checkout
  reproduces the exact suite.

Always use `nix flake check` as the canonical pre-commit test
runner. `cargo test` alone skips the reproducibility
guarantees.

For the canonical flake layout (crane + fenix, with layered
cargo-deps caching so source-only changes recompile in
seconds), see `nix-packaging.md`. The same file covers
`rust-toolchain.toml`, git-URL deps, workspace handling, and
`checks.default` wiring.

**Commit `Cargo.lock`.** The flake reads it to vendor
dependencies. Without it, `nix flake check` fails.

## Tempfile pattern for tests with disk state

When a test needs a writable directory (a database, a config
store, a log file), use `tempfile::tempdir()` so each test
gets an isolated fresh directory that's cleaned up
automatically.

```rust
#[test]
fn store_filters_messages_by_recipient() {
    let directory = tempfile::tempdir().expect("temporary directory");
    let store = MessageStore::from_path(StorePath::from_path(directory.path()));

    let operator_message = Message::from_nota(…)?;
    store.append(&operator_message).expect("operator append");

    let inbox = store.inbox(&ActorId::new("designer"))?;
    assert_eq!(inbox, vec![operator_message]);
}
```

`tempfile = "3"` lives in `[dev-dependencies]`. Tempdirs Drop
clean themselves; no manual teardown.

## Two-process integration tests

For tests that need to verify on-disk format crosses a process
boundary (a CLI binary writes; a different invocation reads),
use `Command::new(env!("CARGO_BIN_EXE_<binary-name>"))`. Cargo
populates `CARGO_BIN_EXE_*` env vars at compile time pointing
at the binary in the test's target directory.

```rust
#[test]
fn two_processes_exchange_a_message() {
    let dir = tempfile::tempdir().unwrap();

    // process A
    let send = std::process::Command::new(env!("CARGO_BIN_EXE_message"))
        .env("PERSONA_MESSAGE_STORE", dir.path())
        .arg(r#"(Send designer "hello")"#)
        .output().unwrap();
    assert!(send.status.success());

    // process B
    let read = std::process::Command::new(env!("CARGO_BIN_EXE_message"))
        .env("PERSONA_MESSAGE_STORE", dir.path())
        .arg("(Inbox designer)")
        .output().unwrap();
    let stdout = String::from_utf8(read.stdout).unwrap();
    assert!(stdout.contains("hello"));
}
```

This is the right shape for testing CLIs that pass state
through the filesystem. The two-process variant proves the
on-disk format works across a process boundary; the
in-process variant only proves the in-memory behavior.

## When to `#[ignore]`

`#[ignore]` marks a test as known-failing or known-skipped.
Acceptable when:

- The fix is out of scope for the current work and a bd issue
  tracks it: `#[ignore = "bd-link-id: <reason>"]`.
- The test depends on an external resource (network, paid API,
  a specific physical device). These should be **opt-in**, not
  default-skipped: gate behind a feature flag or env var
  instead of `#[ignore]` when possible.

Don't `#[ignore]` without filing. An ignored test is dead
weight that future agents will trust as if it ran.

If a bug is found while writing tests, **fix the bug and keep
the test.** The test documents the invariant; the fix keeps
the invariant true.

## Testing actors directly (when you must)

The direct-domain-method pattern handles most cases. The
remaining cases - testing supervision behavior, mailbox order,
fallible `tell`, delegated replies, links, registry, or lifecycle -
require a real actor runtime. For those:

- Use `tokio::test` to set up the runtime.
- Spawn the actor or full topology with Kameo's real spawn surface.
- Send messages through `ActorRef::ask(...)` or `ActorRef::tell(...)`.
- Assert observable side effects: file system state, returned replies,
  `SendError` variants, trace records, or shutdown results.
- Tear down through the component's public lifecycle method, an
  explicit stop message, or Kameo shutdown waiting when the test owns
  the actor directly.

Don't mock supervisors or fake mailboxes. The selected actor runtime
is fast enough to run for real; that is cheaper than building a
parallel mock harness.

## See also

- `style.md` — Rust toolchain reference (Cargo.toml shape,
  cross-crate deps).
- the active workspace's actor-system skill — actor framework usage,
  supervision, and public consumer surfaces.
- `kameo.md` — current Rust actor runtime tool reference.
- `nix-packaging.md` — canonical crane + fenix flake; the
  `checks.default` wiring this file builds on.
- `~/primary/skills/rust-discipline.md` — the discipline these
  patterns implement.
