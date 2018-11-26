---
layout: post
title: "Internal convention is a mistake"
tags: [haskell, internals, modules, decomposition]
comments: true

---

In this post I'm gonna highlight the issues of the "Internal" modularisation convention and provide a proper solution to the same set of problems.

<section id="table-of-contents" class="toc">
  <header>
    <h3>Contents</h3>
  </header>
  <div id="drawer" markdown="1"> 
  *  Auto generated table of contents
  {:toc}
  </div>
</section><!-- /#table-of-contents -->

## About the Internal convention

If you've been doing Haskell for any time at all you've probably already stumbled upon this convention. The reason is that it's so ubiquitous that not even the most centric packages of the ecosystem are immune to it. Here's a short list to give you a glance: "text", "bytestring", "containers", "vector".

So what does that convention solve? It approaches the issue of when a package contains some internal APIs, which were originally only intended for private usage inside of the package, and then at some point in time of the evolution of the package the authors discover that these internal APIs also need to be publicly available. Often is the case of the reason that there are some hardcore use-cases which are best approached using those internal APIs, like highly optimized extension APIs.

## The issues of the convention

[The documentation on an internal module of the "containers" package](http://hackage.haskell.org/package/containers-0.6.0.1/docs/Utils-Containers-Internal-BitQueue.html) would probably give a nice wrapping of the issues of the convention:

>WARNING
>This module is considered **internal**.
>
>The Package Versioning Policy **does not apply**.
>
>This contents of this module may change **in any way whatsoever** and **without any warning** between minor versions of this package.
>
>Authors importing this module are expected to track development closely.

So, basically they approach the problem by violating the rules of the versioning policy. The next logical questions are then, what is the versioning policy good for and are its rules worth following?

Before answering I'm gonna raise one extra question: is the package named "containers" a place you would go searching for [a strict implementation of pair](http://hackage.haskell.org/package/containers-0.6.0.1/docs/Utils-Containers-Internal-StrictPair.html), which it also exposes as an internal module?

## The versioning policy

None of the creations of the real world are ideal, and packages make no exception. Hence all the packages no matter how essential or ubiquitous they are have to evolve in time. When changes come they might as well break the existing API, thus rendering some of the dependent packages incompatible. However some changes do not break the API, like optimisations or bugfixes. This means that for the latter the authors of the dependent packages don't have to change anything about their packages to remain compatible.

Now, versioning policy is the solution to the problem of distinguishing between the two cases for the authors of the dependent packages. More precisely, it provides a way for authors to state that their package is gonna be compatible with a whole range of versions, even with the ones that will only be released in the future, as long as the authors of the depended package stick to the rule of not breaking the API.

IOW, versioning policy lets our ecosystem be more compatible by relieving the authors from having to maintain the dependencies on specific releases of packages. It exponentially reduces the probability of situations when users of a package have to wait for an update release, because they need to use a newer version of another depended package. Sounds like an important thing.

## The mess

Something obviously is wrong in this situation. We establish rules for a viable reason and then we establish a culture of breaking them. Why? Is there really no other way to approach the problem?

The situation is even worse. The newborn authors take an example from the centric packages and export internals without hesitation. We have hundreds of packages like that. The authors no longer problematize themselves with a question of whether they need to review the design of their API, when an itch to export the internals reveals.

The effects of this negligence seem to be accumulating and we seem to be on a fast pace to a state when somebody will suggest to make a sacrifice of the versioning policy and complicate it by introducing an ad-hoc rule, to make it conform with the state of the ecosystem. The existance of Stackage would make the crowd be more inclined to agree with that. To me such a prospect seems terrible, but fortunately we still have a way of saving the situation.

Imagine another situation: what's gonna happen if an internal module of the "bytestring" library changes? Most streaming, builder, serialization, encoding libraries are gonna break. We've only never noticed things like that before because we were lucky enough for "bytestring" authors to be cautious about introducing changes to the internal modules, because those are so heavily depended on. So here's another problem: the authors of internals-exposing libraries feel constrained of introducing changes to internal modules. IOW we get a redundant and absolutely lawless constraint on the evolution of the ecosystem.

## The solution

As much as with any complex problem, the solution is about extraction of smaller pieces, i.e., decomposition. For that all we need is to enable our pattern-detection skills!

Let's take a look at a few examples of the Internal modules:

- [`Data.Text.Internal`](http://hackage.haskell.org/package/text-1.2.3.1/docs/Data-Text-Internal.html)
- [`Data.IntSet.Internal`](http://hackage.haskell.org/package/containers-0.6.0.1/docs/Data-IntSet-Internal.html)
- [`Utils.Containers.Internal.BitQueue`](http://hackage.haskell.org/package/containers-0.6.0.1/docs/Utils-Containers-Internal-BitQueue.html)

The `Data.Text.Internal` module exports the data structure of the `Text` type and lower-level functions. The `Data.IntSet.Internal` module does the same for the `IntSet` type. The `Utils.Containers.Internal.BitQueue` module exports a data-type and an API, which is not at all available in the public modules.

What's the pattern here? All these modules export lower-level APIs, intended for usages, which are not covered by the public APIs of the libraries. Different usage means different audience. The `BitQueue` type could not only be used for the purposes of the "containers" library, but also the things that have nothing to do with containers.

Do you see what I'm reaching towards? All those APIs should have been released as separate libraries!

The "bit-queue" library would have it's own audience, which would likely be way smaller than that of "containers", it would have it's own test-suite, and it's own independent **versioning**. The "containers" package would depend on "bit-queue", and no matter how the "bit-queue" package would iterate on its releases and how frequently it's gonna bump its major version, the "containers" package would only have to bump its minor version, because it's not gonna reflect on the API.

The "text" package could extract a package like "text-core" or "text-internals", where it would expose the structure of its types and low-level functions. Again the audience for that package would be very different: the "text" package itself and hardcore packages, which need precise control over its internals. Clearly such audience is gonna be way smaller and very different from that of the "text" package.

So what would we gain with that approach?

## The benefits

Let's walk through the warnings from the "containers" package again:

>This module is considered **internal**.

No more modules of any special kind.

>The Package Versioning Policy **does not apply**.

Absolutely applies to every package with no exceptions.

>This contents of this module may change **in any way whatsoever** and **without any warning** between minor versions of this package.

No unexpected changes in any of the APIs.

>Authors importing this module are expected to track development closely.

No special expectations from the authors. Just follow the versioning policy!

Another benefit, which is not immediately visible, is that with this approach the true nature of the itch to expose the internal modules becomes evident. It is no more than a symptom of poor code isolation and as such is just another signal to decompose and get a better codebase in the end!
