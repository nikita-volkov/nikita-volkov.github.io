---
layout: post
title: >
  Hasql v2: the Native Era Begins
description: |
  Hasql v2 makes "libpq" a choice: the same battle-tested FFI path by default, an opt-in pure-Haskell adapter, and one argument separating them.
tags: [hasql, pqi, postgresql, haskell, libpq]
comments: false

---

Hasql v2 is here! It can run natively in Haskell with no external dependencies or the same way it always has, using "libpq". It's the user's choice now. No breaking changes in the API except for the choice of the adapter in the connection settings.

With this release it enters a new era, where the native implementation matures incrementally without imposing risks on users and providing them the same old battle-tested path by default.

# The why and the prior

"libpq" is an external C library that is distributed along with PostgreSQL. It has been a centerpiece of Hasql since its inception as much as of any other popular PostgreSQL client library in the Haskell ecosystem.

External C dependencies are an obstacle in distribution. In some use cases it resolves trivially, in others it either limits the distribution opportunities or requires extra effort to work around. 

For this and other reasons I've already made several attempts to migrate Hasql to a pure Haskell implementation in the past but they all failed for reasons that boil down to one argument: Hasql is a production-grade library driving many businesses, so it cannot allow itself to risk downgrading in quality. Replacing a C dependency that is battle-tested across the industry with a custom implementation is a risky move. The amount of edge cases the implementation can reach is huge and open-ended.

With my two previous attempts I've consistently hit some conditions where "libpq" was better and had to postpone. The breakthrough came when I realized that it does not have to be a competition and can be resolved by isolating the competing part via the adapter pattern. So I've released "pqi", an interface over "libpq" that comes with a choice of the adapter between "libpq"-based one and native-one. See [the announcement post](https://nikita-volkov.github.io/pqi-making-libpq-a-choice/).

# Changes in API

The "hasql" v2.0 lib now depends on "pqi" (the interface). The only breaking change in the API is the following (in [`Hasql.Connection`](https://hackage-content.haskell.org/package/hasql-2.0.1.0/docs/Hasql-Connection.html#v:acquire)):

```diff
- acquire :: Settings -> IO (Either ConnectionError Connection)
+ acquire :: Adapter -> Settings -> IO (Either ConnectionError Connection)
```

The user supplies the adapter by providing either [`Pqi.Ffi.adapter`](https://hackage-content.haskell.org/package/pqi-ffi-1.0.1.0/docs/Pqi-Ffi.html#v:adapter) or [`Pqi.Native.adapter`](https://hackage-content.haskell.org/package/pqi-native-1.0.1.2/docs/Pqi-Native.html#v:adapter).

That change escalates to "hasql-pool" where its pool acquisition function changes similarly. The rest of the ecosystem is the same.

That's it.

# Evolution of "pqi"

Integrating the first version of "pqi" with "hasql" has quickly uncovered the problems in its initial API. It was typeclass-based and that complicated the migration a lot, by requiring the "hasql" codebase to introduce existentials and manually rewrite the instances.

So I've tried the record-of-functions approach as the alternative. That one has made "pqi" virtually a drop in replacement for "libpq". The process has essentially turned out to be a replacement of `Database.PostgreSQL.LibPQ` imports with `Pqi` with no changes to logic. It's there in [the migration diff](https://github.com/nikita-volkov/hasql/commit/08819036fd4458d21d62ffe159e2bce4df3138e8). Performance-wise there was no observable difference either.

With this I'm calling the attention of maintainers of libpq-dependant libs. Let's make "pqi" a shared standard that we can all evolve and benefit from.

# Quality of the native adapter

I'll have to be conservative in the claims here and keep its status as Alpha until it gets decent traction and successful usage reports. (Please do report about your experience in its [issue tracker](https://github.com/nikita-volkov/pqi-native)).

"pqi-native" is mostly LLM-generated under a strategy that I've described in [my previous post](https://nikita-volkov.github.io/pqi-making-libpq-a-choice/). The key here is that it is tested to reproduce the behavior of "libpq" with every operation covered via the ["pqi-conformance"](https://github.com/nikita-volkov/pqi-conformance) test suite. Though I cannot claim that the test-suite is exhaustive since every operation requires simulating a context, which is an open-ended process.

The test suites of "hasql", its "hasql-pool" and "hasql-transaction" extensions have now been updated to run against both adapters. That has initially uncovered a few bugs in the native implementation, which have since been reproduced in the "pqi-conformance" test suite and fixed in "pqi-native". That should give you a hint on the process of how "pqi-native" is supposed to incrementally mature.

Since then according to my usage the ride has been pretty smooth and I've even migrated [pGenie](https://pgenie.io) to the native adapter with no problems observed. That has turned out to be a major case for simplifying distribution!

# Performance

Following is a summary of the same benchmark suite executed against the previous-era "hasql" and the new one on both adapters.

| Benchmark | v1.10.3.7 | v2 FFI | v2 native | native/FFI |
|---|---|---|---|---|
| largeResultInVector | 1.922 ms | 1.855 ms | 3.455 ms | 1.86x slower |
| largeResultInList | 1.951 ms | 1.963 ms | 3.558 ms | 1.81x slower |
| manyLargeResults | 320.8 ms | 331.1 ms | 372.0 ms | 1.12x slower |
| manyLargeResultsViaPipeline | 302.1 ms | 306.1 ms | 297.0 ms | 1.03x faster |
| manySmallResults | 2.563 ms | 2.858 ms | 5.406 ms | 1.89x slower |
| manySmallResultsViaPipeline | 1.401 ms | 1.450 ms | 0.400 ms | 3.63x faster |

What can we derive from this?

1. **There is no performance degradation in the FFI adapter** against the previous generation of "hasql". So it's safe to migrate to.

2. The native adapter is slower than the FFI one in all benchmarks where the individual operation overhead dominates. That is expected for two reasons: we're up against C here while reproducing the same API and there's been no optimization effort in pqi-native yet, because correctness is the priority.

3. Where native is faster is pipelining. And it gets impressively faster there. So the "libpq" implementation of pipelining is quite suboptimal. It's not a surprise because pipelining has been bolted on rather than a part of the original design of "libpq".

# Vision

There are other problems with "libpq". E.g., in its standard API it preloads the result sets into memory prior to letting the caller process them. That's bad for large results. It does have an API for streamed processing of results but it is much slower than non-streaming one. That is exclusively imposed by the design of "libpq", because there is no distinction coming from the wire protocol. I've attempted to integrate with it before and rejected for that reason.

So there is room for improvement in the native implementation, which may be approached in the future.

Also the API of "libpq" imposes preconsumption of a byte array per cell, which could be avoided by immediately decoding the cell into the target type. That would save memory allocations and copies, especially noticeable in large result sets. Unfortunately that one is a constraint that "pqi" has to inherit because it stems from the "libpq" API. That may however be addressed in the future by repeating the same pattern on the higher level API determined by the needs of "hasql".

The primary mission of the v2 era is to make the native implementation production-grade. The "libpq"-based adapter is not going anywhere. It is battle-tested and now switching the adapter is a one-argument change.

In the meantime, the work on other aspects of the library continues. I'm moving in two directions: improving the UX and extracting the general pieces into the driver-agnostic space. You may have already noticed ["postgresql-types"](https://hackage.haskell.org/package/postgresql-types). Automatic statement preparation is in the works.

# Conclusion

If you're on v1, the upgrade is a one-argument change and the FFI adapter behaves exactly as it did before. If you can afford to be an early adopter, pass the native adapter instead and tell me your success story or report what breaks. Every report becomes a conformance test and makes the adapter permanently closer to "libpq", for everyone.

---

If your team builds on Haskell and PostgreSQL or wants a hand with architecture and development, that's what I do - [codemine.io](https://codemine.io).
