---
source: (our own) — distilled from /home/li/git/criome/src/ and ractor 0.15.6 upstream
fetched: 2026-04-28
---

# Ractor in CANON projects

[`ractor`](https://crates.io/crates/ractor) is the actor framework for any
CANON daemon with state and a message protocol. See [`style.md` §Actors](style.md#actors-logical-units-with-ractor)
for *when* to reach for one; this doc is *how*. For a current working example,
read the actor files under [criome](https://github.com/LiGoldragon/criome)'s
`src/` — every pattern below appears there.

## Cargo.toml

```toml
ractor = { version = "0.15", features = ["async-trait"] }
```

The `async-trait` feature gates `#[ractor::async_trait]`, which every `impl
Actor` block needs. Without the feature the macro path doesn't resolve.

## Four pieces per file, bare-named

One actor per file. The bare noun is the actor; the module path is the
qualifier. A file `engine.rs` exports `Engine`, not `EngineActor` —
`engine::Engine` already reads correctly. Every actor file exports exactly
four pieces; `State` fields stay private (mutate via `Message`, not by
reaching in):

```rust
pub struct Engine;            // ZST behaviour marker
pub struct State { ... }      // owned state, private fields
pub struct Arguments { ... }  // pre_start input (only struct with pub fields)
pub enum Message { ... }      // per-verb typed messages
```

## Messages — perfect specificity

One variant per verb. Each RPC variant carries its own typed `RpcReplyPort<T>`.
No god `HandleFrame` envelope — the actor boundary is a perfect-specificity
seam. Use **named-field** variants — illustrative shape:

```rust
pub enum Message {
    SomeVerb        { payload: Payload,
                      reply_port: RpcReplyPort<Outcome> },
    SomeOtherVerb   { argument: ArgumentType,
                      reply_port: RpcReplyPort<OtherReply> },
}
```

`ractor::call!` only accepts positional variants; use `actor_ref.call(...)`
directly. `let _ = reply_port.send(...)` is the right send pattern — the port
consumes itself, and the only error means the caller already dropped its
future.

## `CallResult` flattening — free function, not trait method

`ActorRef::call` returns `Result<CallResult<T>, MessagingErr<M>>` — outer is
mailbox-dead-at-send-time, inner is `Success(T) | Timeout | SenderError`.
Flatten via one free helper:

```rust
fn call_into<T>(result: ractor::rpc::CallResult<T>, label: &'static str) -> Result<T> {
    match result {
        ractor::rpc::CallResult::Success(value) => Ok(value),
        ractor::rpc::CallResult::Timeout =>
            Err(Error::ActorCall(format!("{label}: call timed out"))),
        ractor::rpc::CallResult::SenderError =>
            Err(Error::ActorCall(format!("{label}: sender dropped before reply"))),
    }
}
```

A trait extension method on `CallResult<T>` shadows method resolution and
produces opaque closure-type errors. The free helper is the working shape.

## Implementing `Actor`

```rust
#[ractor::async_trait]
impl Actor for Engine {
    type Msg = Message;
    type State = State;
    type Arguments = Arguments;

    async fn pre_start(&self, _myself: ActorRef<Self::Msg>, arguments: Arguments)
        -> std::result::Result<Self::State, ActorProcessingErr>
    {
        Ok(State::new(arguments.dependency))
    }

    async fn handle(&self, _myself: ActorRef<Self::Msg>, message: Message,
                    state: &mut State)
        -> std::result::Result<(), ActorProcessingErr>
    {
        match message {
            Message::SomeVerb { payload, reply_port } => {
                let _ = reply_port.send(state.handle_some_verb(payload));
            }
            // ...
        }
        Ok(())
    }
}
```

Spell `std::result::Result<…, ActorProcessingErr>` fully — the crate's
`Result<T>` alias is single-parameter and shadows the bare name with a
confusing error.

## Self-cast loops — accept and read

A long-running loop is a `Message` cast from `pre_start`, then re-cast from
`handle`. The mailbox stays the serialisation point.

```rust
async fn pre_start(&self, myself: ActorRef<Self::Msg>, args: Arguments)
    -> std::result::Result<Self::State, ActorProcessingErr>
{
    let listener = UnixListener::bind(&args.socket_path)?;
    ractor::cast!(myself, Message::Accept)?;
    Ok(State { listener, ... })
}
// handle re-arms by casting Message::Accept again after each accept
```

The same shape covers connection read loops (`Message::ReadNext`).

## Supervision

A child uses `Actor::spawn_linked(name, behaviour, args, parent.get_cell())`.
The root uses bare `Actor::spawn(...)` — only one unrooted spawn per process,
at the top.

The default `handle_supervisor_evt` **dies on any child failure**. Correct for
the daemon root and the reader-pool parent (a dead reader is fatal). Wrong for
the per-connection-spawning Listener — override to log and continue:

```rust
async fn handle_supervisor_evt(&self, _myself: ActorRef<Self::Msg>,
    event: SupervisionEvent, _state: &mut State)
    -> std::result::Result<(), ActorProcessingErr>
{
    if let SupervisionEvent::ActorFailed(actor, reason) = event {
        eprintln!("daemon: connection {actor:?} failed: {reason}");
    }
    Ok(())
}
```

`pre_start` panics are **not** supervised — they bubble out of `Actor::spawn`
as `SpawnErr`. Use `?` on fallible startup, never `unwrap`. `post_stop` is
skipped on `Signal::Kill` and on panic in `handle`; don't put must-run cleanup
there.

**Mailbox semantics.** Each actor's mailbox is processed in priority order:
Signals > Stop > SupervisionEvents > user messages. Priority only kicks in
**between** `handle` invocations — an `await` inside `handle` blocks the
mailbox entirely; even a `Stop` signal can't preempt, the current handle has
to return first. Two consequences worth knowing:

- An actor calling `ActorRef::call` on **itself** deadlocks (it's waiting for
  itself to process the message it just sent). The same goes for any
  bidirectional `call` cycle in the supervision graph; keep `call` arrows
  strictly downward.
- `myself.stop(...)` is async-fire-and-forget — it queues a Stop signal,
  current handle finishes, then the loop exits. Use `stop_and_wait` if you
  need to synchronize on the actor having actually exited (rare; usually
  fire-and-forget is correct).

**State doesn't drop when `handle` returns.** Ractor wraps the exiting actor's
State in a `BoxedState` and queues it to the supervisor as part of the
`ActorTerminated` event. The State only drops when the supervisor's mailbox
processes that event — which can be much later if the supervisor is inside an
active `await` (per *Mailbox semantics* above).

If the State holds a resource that something *external* is waiting on — a
UnixStream the client is reading from, a TcpStream a peer is waiting on EOF
from — close it **explicitly** inside `handle` before `myself.stop()`. Relying
on `Drop` is a deadlock when the supervisor is busy:

```rust
async fn handle(&self, myself: ActorRef<Self::Msg>, _message: Message,
                state: &mut State)
    -> std::result::Result<(), ActorProcessingErr>
{
    state.do_work().await?;
    let _ = state.client.shutdown().await;  // close the write half eagerly
    myself.stop(None);
    Ok(())
}
```

## Worker pools — `Vec<ActorRef<…>>` plus shared cursor

A worker pool is N spawned-linked siblings plus an `Arc<AtomicUsize>` cursor
cloned into every dispatcher. Lock-free; no factory needed:

```rust
let index = self.cursor.fetch_add(1, Ordering::Relaxed) % self.workers.len();
self.workers.get(index)
```

`Arc<Mutex<T>>` between actors is the smell; an uncontended atomic counter is
a coordination atom, not shared state in the dangerous sense.

## Sync façade on `State`

When the crate ships a one-shot CLI binary or wants tests without a tokio
runtime, expose the dispatch as inherent methods on `State`. The async
`handle` and the sync façade share the same per-verb method implementations
— no duplication.

Add a façade only when there's a non-actor consumer (binary, tests). Don't
manufacture one for ceremony.

## Daemon entry point

`Daemon::start` is the only place you call bare `Actor::spawn`. `main` awaits
its `JoinHandle`:

```rust
impl Daemon {
    pub async fn start(arguments: Arguments)
        -> Result<(ActorRef<Message>, tokio::task::JoinHandle<()>)>
    {
        Actor::spawn(Some("daemon".into()), Daemon, arguments)
            .await
            .map_err(|error| Error::ActorSpawn(error.to_string()))
    }
}
```

## See also

- [`style.md` §Actors](style.md#actors-logical-units-with-ractor) — when to reach for one
- [`programming/abstractions.md`](../programming/abstractions.md) — methods-on-types
- [criome](https://github.com/LiGoldragon/criome)'s `src/` — current working example of every pattern above
- [ractor 0.15 docs](https://docs.rs/ractor/0.15) · [supervisor example](https://github.com/slawlor/ractor/blob/v0.15.6/ractor/examples/supervisor.rs)
