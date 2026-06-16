---
layout: post
title: 'pqi: Making libpq a Choice, Not a Requirement'
description: The story of how I stopped trying to beat "libpq" and made it optional instead - a pluggable interface that lets each user choose their PostgreSQL transport, with a pure-Haskell adapter ported from C by an LLM and verified against "libpq" itself.
tags: [pqi, hasql, postgresql, haskell, libpq, llm]
comments: false

---

For years, every serious Haskell PostgreSQL driver has been chained to the same C library - `libpq` - with no clean way out. I tried to cut it loose twice, and both times I failed at the finish line. They say the third time's the charm. I hope it will be - not because I've tried harder, but because I've changed the goal.

I'm Nikita Volkov, author of `hasql`. This is the story of that reframe, and of [`pqi`](https://github.com/nikita-volkov/pqi), the library it produced.

### 2014: hasql, tied to C

`hasql` came out in 2014. It was fast, type-safe, SQL-first - and, like everything else in the ecosystem, it stood on [`postgresql-libpq`](https://hackage.haskell.org/package/postgresql-libpq), a binding to the C `libpq` library.

That dependency is invisible right up until it isn't. You feel it when `libpq` isn't on the build machine. You feel it in CI. You feel it in a minimal production container, when you try to cross-compile, or when you just want a static binary and the linker has other ideas.

### Two attempts that stalled

I tried. Twice.

The first attempt was a pure-Haskell wire protocol written directly inside `hasql`. I got it working, but I couldn't make it beat `libpq` on performance. The one place it clearly won was pipelining, because `libpq` had none at the time - and pipelining was the reason I'd started the chase in the first place. "Wins on one axis, loses on the rest" wasn't enough to justify forcing every `hasql` user onto it, so I shelved it.

The second attempt, around 2021, went the other way: I led with optimisation. I wrote supporting C to bundle alongside the Haskell and pushed until I was outperforming `libpq` across a range of benchmarks. I was almost there.

Then two things happened. `libpq` shipped its own pipelining in version 14 - defeating my original motivation entirely. And I switched my own machine to ARM, where my recent performance gains against `libpq` started to become less pronounced. Once more, not satisfied enough to ship. Once more, shelved.

Both times I was chasing the same target: be faster than `libpq`. Both times the target moved, and I lost the race at the finish line. (The full story of those two attempts probably deserves a post of its own.)

### The demand never went away

Shelving a project twice doesn't shelve the need behind it. Questions about the fate of native Hasql persisted with a stable frequency over the years.

Eventually I realised I'd been answering the wrong question the whole time.

### The reframe: libpq can be a choice

I had been treating this as a duel - my implementation versus `libpq`, winner takes the ecosystem. That framing is what kept killing the project. The moment you set out to replace `libpq`, you've signed up to beat it on every axis, forever, on every architecture.

But a driver doesn't need to declare a winner. It can let the user declare one.

If swapping the C-backed implementation for a pure-Haskell one is a one-line change in your dependencies - and nothing in your driver code changes - then the performance question stops being existential. Need the battle-tested C path? Use it. Need a static binary with no `libpq` in sight? Swap the adapter at no cost to your codebase. The driver author writes against one interface and supports both, the user picks.

That's `pqi`: a driver-agnostic interface to the `libpq` API, plus interchangeable adapters behind it.

### How it works

`pqi` is an ecosystem of libraries:

- **[`pqi`](https://github.com/nikita-volkov/pqi)** - the interface. It reproduces the `postgresql-libpq` API surface, but reifies the connection (and its results) as a type class rather than a concrete type. Code written against it runs unchanged on any adapter.
- **[`pqi-ffi`](https://github.com/nikita-volkov/pqi-ffi)** - a thin adapter over `postgresql-libpq`. Battle-tested, production-safe, the default.
- **[`pqi-native`](https://github.com/nikita-volkov/pqi-native)** - a pure-Haskell adapter that speaks the PostgreSQL frontend/backend wire protocol directly. No C. Experimental.

You choose the adapter by choosing the connection type. Everything else is written against the `IsConnection` constraint, so it doesn't change:

```haskell
import qualified Pqi
import qualified Pqi.Ffi
import qualified Pqi.Native

-- C-backed (battle-tested, requires libpq)
connection <- Pqi.connectdb conninfo :: IO Pqi.Ffi.Connection

-- Pure Haskell (experimental, no C dependency)
connection <- Pqi.connectdb conninfo :: IO Pqi.Native.Connection
```

Same code, same API, different transport.

### How LLMs changed the equation

Reframing the goal solved the strategy, not the work. A correct PostgreSQL wire-protocol implementation takes a lot of sweat - exactly the kind of thing I'd burned out on twice. A complete implementation for the libpq API surface is even scarier.

My final realisation happened after getting used to LLMs and studying their patterns. You see, LLMs operate the better the clearer you define the context of the task and constrain it. Tasks like porting an existing solution from one language to another typically satisfy that condition perfectly. By the way, that is why you see the burst of everything getting ported to Rust in the wild these days.

So I married the two ideas. If the interface mirrors `postgresql-libpq` as closely as possible, then two things follow:

- The FFI adapter becomes a near-mechanical delegation - each method maps to the matching `Database.PostgreSQL.LibPQ` function. That's trivial to implement and review.
- The native adapter becomes a dumb port. No inventions needed. I can feed the original C `libpq` code to an LLM and ask it to reproduce the same behaviour against the same interface in Haskell.

But generation alone is worthless if you can't trust the output, and you cannot eyeball a wire protocol into correctness. So the real work here isn't the implementation - it's testing.

**[`pqi-conformance`](https://github.com/nikita-volkov/pqi-conformance)** is the fourth package, and it's what makes "LLM-generated wire protocol" something other than a liability. It's a differential test harness: it runs the same operation sequence on a candidate adapter and on `postgresql-libpq`, against the same live PostgreSQL, and asserts the observations are identical - result statuses, cell values, field metadata, command tags, and structured error fields, down to `libpq`'s exact error-message text and `LINE N:` position markers. If the outputs diverge anywhere, the adapter fails. `postgresql-libpq` has been correct for years, which makes it the perfect reference. The generator can be fuzzy because the gate is exact.

It's the same principle behind [pGenie](https://nikita-volkov.github.io/pgenie-in-production-part-1/): the thing you end up trusting is **the proof system that checks the output, not the LLM** that produced it. Let the machine handle the open-ended part and anchor it to something you can already verify.

### Where it stands

`pqi-native` implements the full API surface - every operation `pqi` defines - and the conformance suite holds it to identical output against `postgresql-libpq` across connectivity, simple and extended queries, escaping, async commands, large objects, and trust / MD5 / SCRAM-SHA-256 authentication. Every operation of the API surface is covered.

However most of the code is also LLM-generated and not battle-tested. Conformance-verified is not the same as proven at scale. Real workloads surface edge cases that a test suite - however strict - doesn't know to look for yet. That's exactly why the default stays on `libpq` and the native path is opt-in.

This is the intended process for how it matures:

1. Early adopters who can accept the risk start using it.
2. They report back what works and what breaks.
3. Every failure becomes a new case in the conformance suite.
4. The LLM fixes the adapter until it satisfies the suite again.

Each reported issue makes the conformance suite stricter and the adapter a little closer to `libpq` - permanently, and for everyone. The suite only grows.

### What's next

I'm currently integrating `hasql` with `pqi`. For users, this will be about as small as an API change gets: connection acquisition will accept one additional argument that selects which adapter to use. Pass the FFI adapter to keep the C-backed path, pass the native adapter to go pure-Haskell. Everything downstream - sessions, statements, codecs - stays exactly as it was.

So the door is open. Driver authors - `hasql`, the `postgresql-simple` family, anything new - can build against `pqi` and give their choice for free. And if you don't like my native adapter, the interface is the only contract - write a better one and it works everywhere. We don't need one perfect pure-Haskell driver to win a benchmark war. We just need to agree on an interface. What interface can be better than the one that everyone already uses?

---

- **Learn** - start with the [`pqi` README](https://github.com/nikita-volkov/pqi) or follow on to the [Hackage docs](https://hackage.haskell.org/package/pqi)
- **Discuss** - open a thread in [`pqi` Discussions](https://github.com/nikita-volkov/pqi/discussions)
- **Support** - if it's useful, give it a [star](https://github.com/nikita-volkov/pqi)

If your team builds on Haskell and PostgreSQL and wants a hand with architecture or development, that's what I do - [codemine.io](https://codemine.io).
