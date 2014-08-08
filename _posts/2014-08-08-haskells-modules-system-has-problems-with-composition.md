---
layout: post
title: Haskell's modules system has problems with composition
tags: [haskell, modules]
description: 
comments: true

---


## Composition and Decoupling

Proficient programming is all about decomposition of complex problems into smaller pieces. Most of us know that. When the smaller pieces are completely decoupled from each other you achieve the zen of being able to work on each of them in complete isolation. This means that whatever the entities you define in those smaller abstraction pieces, you can be sure that they will not affect the "sister" pieces of the same abstraction level. They all will have only a single dependant - a piece of a higher abstraction level, which is essentially a composition of them. 

Here's what I mean by a schematic example:

            Connection           <-- Abstraction layer 1.
           /    |     \              
          /     |      \             Essentially is a composition of the
         /      |       \            entities of abstraction layer 2.
        |       |        |
        v       v        v
    Protocol  Network  Client    <-- Abstraction layer 2.
             Transport  State        
                                     Multiple decoupled entities,
                                     each dealing with its own set of problems.
                                     Them being decoupled implies that, e.g.,
                                     "Protocol" does not refer to 
                                     "Client State" in its implementation.

                                     These entities might well themselves be
                                     decomposed into more abstraction layers.

Imagine a LEGO constructor. You can think of the entities of the second abstraction layer as of kinds of blocks and of the sole entity of the first abstraction layer as of the house you build from them. Such a thing as some form of a dependency of one block on another block simply doesn't make sense, however the house you're building evidently depends on what blocks you use and how you stick them together, i.e., how you **compose** them.

This is essentially all that the composition is about. By following this principle you ensure that your program's structure is consistent and predictable, and, hence, clear. Transitively this results in a code that is easy to reason about and maintain.

## The problem with the Haskell's module system

The sole purpose of module systems is in helping us deal with decomposition. However the current Haskell's module system sometimes gives us the business and facilitates spaghetti code instead.

Suppose you design a library and you have two decoupled modules, meaning they are supposed to not know anything of each other:

**Module 1**:

{% highlight haskell %}
module Lib.Connection where 
  data Failure =
    Failure1 String
{% endhighlight %}

**Module 2**:

{% highlight haskell %}
module Lib.Result where 
  data Failure =
    Failure1 String
{% endhighlight %}

As you can see, both modules export data types and constructors with the same names.

Now suppose, you have a third module, which is a **composition** of the above two and is actually the only module that your library is supposed to export.

**Module 3**:

{% highlight haskell %}
module Lib where
  import qualified Lib.Connection as Connection  
  import qualified Lib.Result as Result

  data Failure =
    ConnectionFailure Connection.Failure |
    ResultFailure Result.Failure
{% endhighlight %}

This is a classical composition. I doubt there is a way to do it more correctly. But do you spot a problem? The problem is that `Lib`, our sole exported module, does not export the `Failure` symbols of the submodules. And in fact it can't, because there are name conflicts. 

We are able to resolve the ambiguity of the `Failure` type by declaring type aliases in `Lib` and deal with the ambiguity of function definitions in similar fashion. While this definitely will require some annoying boilerplate, this will still be a solution, well, at least sorta. However there is no way for us to alias the `Failure1` data constructors (definitely a shortcoming of Haskell, but I'm going for another one). Not exporting the constructors is not an option, because this will render our users unable to pattern match failures.

So, what to do? Well, turns out, Haskell doesn't give you much options: either shoot yourself in the foot or cut it off. Specifically, you need to treat `Lib.Connection` and `Lib.Result` as a single shared namespace, removing the conflicts, and thus introducing a dependency between those modules. Otherwise you can introduce a wrapper API in the `Lib` module, which will have it's own constructors and will do a bunch of repacking:

{% highlight haskell %}
module Lib where
  import qualified Lib.Connection as Connection  
  import qualified Lib.Result as Result

  data Failure =
    ConnectionFailure1 String |
    ResultFailure1 String

  connectionFailureToFailure :: Connection.Failure -> Failure
  -- ...
  -- and so on

{% endhighlight %}

So, either a decoupling ruined with an absolutely unjustified cross-module dependency of abstractions otherwise having nothing to do with each other, or a massive boilerplate with performance penalty, but you get to keep your decoupling. There is a third option, of course: don't do composition. Wait. What?!

So, yeah, it's official: Haskell's module system facilitates spaghetti code in some cases. Don't get me wrong, I still believe that it's by far the best practical language out there, but there are important issues that need to be dealt with. Concerning the issue described here I am actually coming up with a proposal, which will be the subject of my next post, which I plan to post in the coming days.
