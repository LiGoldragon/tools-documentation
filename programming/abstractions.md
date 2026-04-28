---
source: (our own — distilled from abstractions-research.md)
fetched: 2026-04-27
---

# Abstractions — where behavior lives

A short discipline for thinking about programs: **every reusable
verb belongs to a noun.** This applies to any language with method
dispatch (Rust, Python, Go, Java, C++, Smalltalk) and is enforced
by convention in languages without it (C's `_operations` vtables,
Haskell's typeclass-constrained free functions). The full research
backing this rule, with citations across cognitive science,
software-engineering practice, type theory, and LLM-codegen
empirical work, lives in
[abstractions-research.md](abstractions-research.md).

This file is the working document. Read this; consult the research
doc when you need the *why* in depth.

## The rule

Behavior that is reusable lives on a type. Free functions are for
things that genuinely belong nowhere else: a binary's `main`, a
small private helper inside one module, a pure mathematical
operation between values of equal status.

```rust
// Wrong — verb floating, type implicit
fn parse_query(text: &str) -> Result<QueryOp, Error> { … }

// Right — verb on the type that owns it
struct QueryParser<'input> { lexer: Lexer<'input> }

impl<'input> QueryParser<'input> {
    pub fn new(input: &'input str) -> Self { … }
    pub fn into_query(self) -> Result<QueryOp, Error> { … }
}
```

The rule is not aesthetic. It is a forcing function.

## Affordances vs operations

Methods encode **affordances** — what kinds of things a value of
this type *can do*. Free functions encode **operations** that
happen to take some arguments. The distinction is structural.

In the real world, fruits can be eaten and clouds cannot. Code
that models the world correctly says `fruit.eat()`, not
`eat(fruit)`. The method form binds the verb to the type that
owns it. The free-function form lets the verb float — and
`eat(cloud)` becomes thinkable, type-checked only if you happen
to have given `Cloud` an explicit "missing eat" marker.

The vocabulary comes from outside CS. James Gibson's 1979
*Ecological Approach to Visual Perception* defined an *affordance*
as "what [the environment] offers the animal, what it provides or
furnishes, either for good or ill." Donald Norman's 1988 *Design
of Everyday Things* applied it to artifacts: a door's handle
affords pulling; a flat panel affords pushing. The affordance is
a property of the relationship between the object and the agent.

A method-bearing type *advertises* its affordances at every call
site. A passive record next to a free-function library does not.
The type system knows which is which only when the operations are
attached to the things that own them.

## The forcing function

The deeper purpose of the rule is not what it makes you write;
it's what it makes you do *before* you write.

If you sit down to write a verb, the rule forces the question:
*what type owns this verb?* Sometimes the answer is obvious — a
method on an existing type. Sometimes the answer is "no type
exists yet for this," and the rule forces you to invent one.
That forced invention is the load-bearing cognitive event.

Without the rule, the verb gets written as a free function and
the noun never appears. The model develops gaps: verbs without
owning nouns, missing structural types, behavior smeared across
the call graph. Programs that "look fine" end up missing whole
structural types they ought to have.

The pattern is named in the refactoring catalogue. Martin Fowler:
**Feature Envy** is "a method that seems more interested in a
class other than the one it is in" — a verb in the wrong place.
**Data Class** is the same drift seen from the other side — a
type with no behavior because the verbs that should have lived
on it ended up elsewhere. **Anemic Domain Model** is the
codebase-scale form. The cure for all three is the same:
*Move Function* / *Extract Class* — find the type, attach the
verb.

The rule is: do this once, up front, instead of accumulating the
debt and refactoring later.

## Why this matters more for LLM agents

Humans procrastinate creating types because typing out
`struct QueryParser { … }` *feels heavier* than `fn
parse_query(…)`. There is tactile friction in declaring a noun,
naming its fields, deciding its constructor. That friction is a
feature: it makes humans ask "is this type pulling its weight?"
before paying the cost.

LLMs have no such friction. Generating `struct QueryParser` and
generating `fn parse_query` cost the same number of tokens, take
the same wall-clock time, and produce no felt sense of "this is
heavy." The result is predictable: LLMs default to whichever
shape is *shorter* — almost always the free function.

The rule reintroduces, by fiat in a style guide, the friction
the substrate has erased. It changes what the agent can think,
by changing what it is *required* to write.

The empirical work on LLM-generated code documents the symptoms
without naming the cause. Tambon et al. 2024 found LLM output is
"shorter yet more complicated" than canonical solutions, with
"misunderstanding and logic errors" as the largest bug category.
Spinellis et al. 2025 found 33.7% of LLM-generated JavaScript
contains "unused code segments" and 83.4% of Python shows
"invalid naming conventions." The underlying failure is **verbs
without owning nouns**: naming conventions go bad because there
is no type to anchor a name to; unused code accumulates because
nothing carries a clean responsibility.

## The Karlton bridge

Phil Karlton: "There are only two hard things in Computer
Science: cache invalidation and naming things."

When an LLM agent skips creating a type, **it skips the naming
step entirely.** The hard thing is not avoided; it is hidden.
The methods-on-types rule restores the hard step into the
workflow, where it belongs.

This is the cleanest one-line statement of the rule's purpose:
*the rule exists to make sure naming happens.*

## Principled exceptions

The rule has carve-outs, named directly. Use them honestly; they
are not a back door for skipping the noun-creation step.

### The local-helper carve-out

A small private helper inside one module is fine if it is
genuinely local — a three-line `fn hex(h: &Hash) -> String` next
to a single `Display` impl is not a missing noun, it is a
private fragment of one impl. The rule kicks in when the verb is
*reusable* — when more than one caller might want it, when it
would be discoverable from multiple sites, when its life as a
free function would let it spread.

### The relational-operation carve-out

Some operations are genuinely **relational** between two values
of equal status, with no state on either side. `add(a, b)` over
two numbers is the canonical case. William Cook's 2009 essay
*On Understanding Data Abstraction, Revisited* gives the formal
frame: ADTs (operations outside the data) and objects (operations
inside the data) are dual / complementary, neither wrong. Pure
mathematical operations fit the ADT axis.

In practice, in object-oriented or method-bearing languages, this
exception is usually expressed via operator overloading — `a + b`
desugars to `Add::add(a, b)`, which IS a method on a type, just
with operator-syntax sugar. The rule is preserved.

### The standard-library carve-out

Names inherited from well-known libraries get to keep their
shape. `serde_json::from_str` and `serde_json::to_string` are
free functions because the ecosystem convention demands them. A
serde-format crate that hides this convention behind methods
would surprise every user who has ever reached for `serde_json`.
The carve-out is **narrow**: the crate-root `from_str` /
`to_string` shape is preserved; everything inside the crate's
own implementation should still attach behavior to its owning
types.

The general principle: don't invent gratuitous deviations from
established conventions, but don't let "convention" be a sloppy
excuse for missing types.

### When the language doesn't have methods

The rule still applies. C codebases follow it via vtables —
`struct file_operations`, `struct inode_operations`, `struct
backlight_ops` in the Linux kernel. Behavior is attached to the
type; only the dispatch is manual. Haskell follows it via
typeclass-constrained free functions — `Eq a => a -> a -> Bool`
is conceptually a method on `a` even though the syntax is
top-level. Python follows it via `class … def …`. The discipline
is universal even when the syntax varies.

## What "find the noun" actually looks like

When the rule's question — "what type owns this verb?" — is
hard, that hardness is a signal. The signal is that the model
of the problem isn't fully formed yet. Three kinds of resolution:

1. **The noun already exists.** You missed it. Attach the verb
   as a method.
2. **The noun is implicit but unnamed.** A `parse_query` free
   function already has a `QueryParser` inside it: parser
   state, input cursor, error context. Name it. Make the
   implicit explicit.
3. **The verb is genuinely relational.** Two values of equal
   status, no state, no privileged owner. Use the relational-
   operation carve-out.

If none of these apply, you don't have a clean program model
yet. Slow down. Don't paper over the gap with a free function.

## Companion disciplines

This rule pairs with two others that push the same direction:

- **Wrapped field is private.** A newtype wraps a primitive to
  give it identity; if the wrapped field is `pub` (`Slot(pub
  u64)`), callers can construct unchecked values and read raw
  bytes back out, defeating every reason to wrap. Same
  discipline: the type owns its representation. (Rust enforcement
  in [rust/style.md §"Domain values are types, not primitives"](../rust/style.md).)

- **Perfect specificity.** Every typed boundary in the system
  names exactly what flows through it — no wrapper enums that
  mix concerns, no string-tagged dispatch, no generic-record
  fallback. Same discipline: the type system carries the
  meaning, not stringly-typed metadata.

All three rules say the same thing in different domains: **the
type system is the model**. Use it.

## The one-line summary

**Every reusable verb belongs to a noun. If you can't name the
noun, you haven't found the right model yet — keep looking until
you can.**
