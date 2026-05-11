---
source: (our own) - distilled from kameo 0.20 source, docs.rs, and the local kameo-testing repos
fetched: 2026-05-11
---

# Kameo - actor framework

`kameo` 0.20 is the current Rust actor runtime for workspace
daemons. The discipline lives in the active workspace's
`skills/actor-systems.md` and `skills/kameo.md`; this file is the
tool-reference note: dependency shape, imports, API traps, and the
parts that are easy to misremember.

Kameo is pre-1.0 and actively moving. Pin it intentionally and read
the local skill before changing actor code.

## Cargo.toml

```toml
kameo = "0.20"
```

Kameo 0.20 requires Rust `1.88.0`. Crates adopting it must move their
`rust-toolchain.toml` and Nix fenix pin together.

Default features include `macros` and `tracing`. Leave `remote` off for
local Persona components unless the component architecture explicitly
chooses cross-process Kameo remoting. With `remote` enabled, the local
registry surface changes shape.

## Core shape

`Self` is the actor. Do not create a public ZST behavior marker plus a
separate state type. The data-bearing type implements `Actor`, and its
message handlers are `impl Message<T> for ThatType`.

```rust
use kameo::Actor;
use kameo::actor::{ActorRef, Spawn};
use kameo::error::Infallible;
use kameo::message::{Context, Message};

pub struct Counter {
    count: i64,
}

impl Actor for Counter {
    type Args = Self;
    type Error = Infallible;

    async fn on_start(args: Self, _actor_ref: ActorRef<Self>) -> Result<Self, Self::Error> {
        Ok(args)
    }
}

pub struct Increment {
    pub amount: i64,
}

impl Message<Increment> for Counter {
    type Reply = i64;

    async fn handle(
        &mut self,
        message: Increment,
        _context: &mut Context<Self, Self::Reply>,
    ) -> Self::Reply {
        self.count += message.amount;
        self.count
    }
}

let counter = Counter::spawn(Counter { count: 0 });
let value = counter.ask(Increment { amount: 1 }).await?;
```

Name the actor by what it is or does: `Counter`, `Router`,
`ClaimNormalizer`, `StoreSupervisor`. Do not add `Actor` to the type
name. The trait impl already says it is an actor.

## Module map

| Symbol | Path |
|---|---|
| `Actor`, `Spawn`, `ActorRef`, `WeakActorRef`, `ActorId`, `PreparedActor`, `Recipient`, `ReplyRecipient`, `RemoteActorRef` | `kameo::actor::*` |
| `Message`, `Context`, `StreamMessage`, `BoxMessage`, `BoxReply`, `DynMessage` | `kameo::message::*` |
| `Reply`, `ReplyError`, `ReplySender`, `DelegatedReply`, `ForwardedReply`, `BoxReplySender` | `kameo::reply::*` |
| `ActorStopReason`, `PanicError`, `PanicReason`, `SendError`, `HookError`, `Infallible`, `set_actor_error_hook` | `kameo::error::*` |
| `bounded(n)`, `unbounded()`, `MailboxSender`, `MailboxReceiver`, `Signal` | `kameo::mailbox::*` |
| `RestartPolicy`, `SupervisionStrategy`, `SupervisedActorBuilder` | `kameo::supervision::*` |
| `ACTOR_REGISTRY` | `kameo::registry::*` when `remote` is off |
| remote actor refs and network registry | `kameo::remote::*` when `remote` is on |

`bounded(n)` and `unbounded()` are module-level free functions that
return mailbox sender/receiver pairs. There is no `Mailbox::bounded`
type method.

## Messages and replies

Each message kind is its own named type. The actor implements
`Message<ThatMessage>` once per accepted message. Prefer full domain
names like `Increment`, `PromptPatternRegistration`, or
`ChannelGrantRequest` over abbreviations or `*Message` suffixes.

`actor_ref.ask(message).await` returns the reply result. For a handler
with `type Reply = Result<T, E>`, the caller sees:

- `Ok(T)` when the handler returns `Ok(T)`.
- `Err(SendError::HandlerError(E))` when the handler returns `Err(E)`.

`actor_ref.tell(message).await` is fire-and-forget. Do not `tell` a
fallible handler whose reply is `Result<_, _>` unless the actor
explicitly overrides panic handling for that case. A fallible handler
returning `Err(_)` to a `tell` is treated as an actor panic under
Kameo's default `on_panic`.

Use `DelegatedReply<R>` when a handler must start slow async work and
return the mailbox to service immediately. The spawned task owns the
reply sender.

## Spawning and mailboxes

| Form | Shape |
|---|---|
| `MyActor::spawn(args)` | synchronous; returns `ActorRef<MyActor>`; default mailbox capacity is 64 |
| `MyActor::spawn_with_mailbox(args, mailbox::bounded(n))` | synchronous; custom bounded mailbox |
| `MyActor::spawn_with_mailbox(args, mailbox::unbounded())` | synchronous; no backpressure |
| `MyActor::spawn_in_thread(args)` | synchronous; dedicated OS thread; panics on a current-thread Tokio runtime |
| `MyActor::spawn_link(&peer, args).await` | async; links before the run loop starts |
| `MyActor::supervise(&parent, args).restart_policy(...).spawn().await` | async; supervised child |
| `MyActor::prepare()` | obtains an `ActorRef` before the actor is run |

Defaults worth remembering:

| Concern | Default |
|---|---|
| Mailbox capacity | 64 |
| `RestartPolicy` | `Permanent` |
| `SupervisionStrategy` | `OneForOne` |
| Restart limit | 5 restarts per 5 seconds |
| `on_panic` | stop the actor |
| `on_stop` | no-op `Ok(())` |

`supervise` requires cloneable/sync args; use `supervise_with(factory)`
when actor construction needs a factory instead of a clone.

## Lifecycle traps

- `on_start` returning `Err` does not call `on_stop`.
- `on_stop` panics are not caught in Kameo 0.20.
- `on_stop` returned errors are stored for
  `wait_for_shutdown_result()`, despite stale upstream docs that imply
  a task panic.
- Self-`ask` from inside a handler deadlocks.
- `kill()` drops queued messages; an in-flight handler is aborted at
  its next `.await`.
- Supervised restarts reuse the same mailbox, so queued messages can
  survive into the new actor instance.

## Source-tested drift

The local kameo-testing repos found several upstream doc/source gaps:

- The documented `#[actor(mailbox = bounded(64))]` derive option is
  not implemented in 0.20; use `spawn_with_mailbox`.
- Default mailbox capacity is 64, not 1000.
- `RpcReply` is not a Kameo type. Use `DelegatedReply`,
  `ForwardedReply`, or `ReplySender`.
- `remote` replaces local registry behavior; do not turn it on just
  because a local actor needs a name.

## See also

- this workspace's `skills/kameo.md` - implementation discipline and
  examples.
- this workspace's `skills/actor-systems.md` - actor design rules.
- `testing.md` - Rust testing patterns.
- `style.md` - Rust toolchain reference.
