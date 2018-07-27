---
layout: post
title: "If Haskell were strict, what would the laziness be like?"
tags: [haskell, thunk, lazy, strict, library]
comments: true

---

Recently [a question by Chris Done on Reddit](http://www.reddit.com/r/haskell/comments/36s0ii/how_do_we_all_feel_about_laziness/) has spawned yet another debate on the subject of whether Haskell's laziness is actually a good thing. 

With this post I'm not going to state my take on the matter, instead I'll speculate on what Haskell could be like were it a strict language and how it would approach the standard problems. 

<section id="table-of-contents" class="toc">
  <header>
    <h3>Contents</h3>
  </header>
  <div id="drawer" markdown="1"> 
  *  Auto generated table of contents
  {:toc}
  </div>
</section><!-- /#table-of-contents -->

Before we go on, please note that all the following code will go with an implication that Haskell is strict.

## Conditions

Let's take a look at the `bool` function, which essentially implements exactly the same thing as the "if-then-else" block:

{% highlight haskell %}
bool :: a -> a -> Bool -> a
bool f _ False = f
bool _ t True  = t
{% endhighlight %}

Of course the first question coming to mind is about dealing with the evaluation of alternative branches, because we don't want to waste resources on evaluation of a failed branch.

But look at this:

{% highlight haskell %}
bool' :: (() -> a) -> (() -> a) -> Bool -> a
bool' a b c = bool a b c ()
{% endhighlight %}

In other words, all we need to do is just to defer the evaluation of each branch using a function on a unit.

## Deferred

Already the type `() -> a` seems like a concept of its own, so we can as well give it a name:

{% highlight haskell %}
type Deferred a = () -> a
{% endhighlight %}

Next comes a thought "It suspiciously looks like something that could be a Monad". Yes it does and yes it is! I would have "newtyped" the thing and declared the instances, but there's actually no need, since there already are instances of `Monad`, `Applicative` and `Functor` for a type `((->) r)`, of which `Deferred` is just a special case.

So, with `Deferred` the intent of our condition function becomes clearer:

{% highlight haskell %}
bool' :: Deferred a -> Deferred a -> Bool -> a
{% endhighlight %}

Feels like we're coming closer to the laziness. But we're yet lacking the preservation of already computed results. I mean, we're in a pure language after all, so a specific function of type `() -> a` is guaranteed to always produce the same result, so wouldn't it be enough to compute it once and then just return a stored result? And that's exactly the thing that Haskell's Thunk is meant to solve.

## Thunk

_Please notice that I've improved the following solution and released it as [the "lazy" library](http://hackage.haskell.org/package/lazy). Still it's worth reading for understanding._

Thunk is a basic mechanism that drives Haskell's laziness. The internet is filled with excellent tutorials about it, if you don't know about it yet. Following is how we could implement it explicitly in a library if Haskell were strict:

{% highlight haskell %}
newtype Thunk a = Thunk (IORef (Either (Deferred a) a))

thunk :: Deferred a -> Thunk a
thunk f =
  Thunk $ unsafePerformIO $ newIORef (Left f)

unthunk :: Thunk a -> a
unthunk (Thunk ref) =
  -- Synchronisation is not an issue here,
  -- since there's nothing dangerous in two threads
  -- occasionally computing the same result.
  -- That would be the price of not having 
  -- to pay for locks-maintenance overhead.
  unsafePerformIO $ readIORef ref >>= \case
    Left f -> do
      let a = f ()
      writeIORef ref (Right a)
      return a
    Right evaluated -> return evaluated
{% endhighlight %}

This datatype evidently forms a `Monad`, `Applicative` and `Functor`, but I'll let your imagination figure the instances out.

Now, having thunks at hand we can easily implement any lazy data-structure. For instance, here is the lazy stream, which we call "list":

{% highlight haskell %}
data Stream a =
  Cons a (Thunk (Stream a)) |
  Nil
{% endhighlight %}

Looking at it we can easily extract a general piece, turning it into a more general List Monad Transformer:
{% highlight haskell %}
data ListT m a =
  Cons a (m (ListT m a)) |
  Nil
{% endhighlight %}

Which after refactoring becomes just

{% highlight haskell %}
newtype ListT m a =
  ListT (Maybe (a, m (ListT m a)))
{% endhighlight %}

And, since `Thunk` forms a monad, we get a lazy pure stream for free:

{% highlight haskell %}
type Stream a =
  ListT Thunk a
{% endhighlight %}

As well as the strict linked list:

{% highlight haskell %}
type List a =
  ListT Identity a
{% endhighlight %}

## Thoughts on compiler extensions

There's also an extra thing to consider: a compiler could implement automatic function memoization, or make a special case for functions of type `() -> a`, turning them into implicit thunks, such as the ones Haskell has already. Then there'd be no need for the explicit `Thunk` type facade. Whether that would be a good thing though is a subject for another debate.

## Conclusion

Looking at the above I see a code which has clear semantics about when and whether things get evaluated - it's all there in the type signatures. I also see the language's magical constructs implemented as libraries. Say what you will, but I would love to see that in an actual language.

