---
layout: post
title: "First-class sums and products"
tags: [haskell, thunk, lazy, strict, library]
comments: true

---

Today I'm announcing [the "compound-types" library](https://github.com/nikita-volkov/compound-types). This library provides first-class multi-arity product- and sum-types and neat type-level utilities for their composition. The solution is quite simple and doesn't require the advanced proficiency in the language to be applied in practice.

<section id="table-of-contents" class="toc">
  <header>
    <h3>Contents</h3>
  </header>
  <div id="drawer" markdown="1"> 
  *  Auto generated table of contents
  {:toc}
  </div>
</section><!-- /#table-of-contents -->

## What are the product- and sum-types?

To express it with Algebraic Data-Types, for example, this is a product of `Int` and `Char`:

{% highlight haskell %}
data MyProduct =
  MyProduct Int Char
{% endhighlight %}

This is a sum of `Int` and `Char`:

{% highlight haskell %}
data MySum =
  MySum_Int Int | MySum_Char Char
{% endhighlight %}

However ADTs are not only about products and sums, they are also about type identity. I.e., each ADT declaration creates a new type.

The "compound-types" library reapproaches this by isolating the concepts of products and sums.

## Examples and benefits of "compound-types"

### First-class citizenship

Product- and sum-types are first-class, which means that they can be used in any context, where a type can be used. More specifically,

#### You can easily declare types with sums and products intertwined

{% highlight haskell %}
type SumAndProductMixture =
  Int + Char * (Int + Char + Bool)
{% endhighlight %}

where an alternative definition using ADTs would look like this:

{% highlight haskell %}
data SumAndProductMixture =
  SumAndProductMixture_1 Int |
  SumAndProductMixture_2 Char IntOrCharOrBool

data IntOrCharOrBool =
  IntOrCharOrBool_1 Int |
  IntOrCharOrBool_2 Char |
  IntOrCharOrBool_3 Bool
{% endhighlight %}

#### You can use those types directly in function signatures without having to predeclare them

{% highlight haskell %}
either' :: (a1 -> b) -> (a2 -> b) -> a1 + a2 -> b
{% endhighlight %}

### Composability

Products and sums are composable and the `+` and `*` operators are associative. I.e., the types `a + b + c`, `(a + b) + c` and `a + (b + c)` are all equal.

In practice this means that you can now have a code like this:

{% highlight haskell %}
type LaunchError =
  LaunchError1 + LaunchError2

type LandingError =
  LandingError1 + LandingError2

launchAndLandRockets :: ExceptT (LaunchError + LandingError) IO ()
{% endhighlight %}

where the function `launchAndLandRockets` ends up using the composed type `LaunchError1 + LaunchError2 + LandingError1 + LandingError2` for its error.

## What are those + and * operators?

Those are closed type-families, which resolve the specific sum- and product-types of the according arity. For instance the definition of the `SumAndProductMixture` type from the examples above actually gets resolved to the following:

{% highlight haskell %}
type SumAndProductMixture =
  Sum2 Int (Product2 Char (Sum3 Int Char Bool))
{% endhighlight %}

_Notice that because the closed type families were only introduced in GHC 7.8, it's the oldest compiler version supported by the library. Also you will need to enable the `TypeFamilies` extension._

## Aren't there conflicts with the standard functions + and *?

Nope, there aren't. As per the compiler's opinion, they live in separate universes: type- and value-level.

## What are those Sum3, Product2 and etc.?

Those are polymorphic types predefined by the library. E.g., here's the definition of one of them:

{% highlight haskell %}
data Sum3 v1 v2 v3 =
  Sum3_1 v1 | Sum3_2 v2 | Sum3_3 v3
{% endhighlight %}

## What about laziness and strictness?

All types are provided in two variations: with all fields lazy and with all fields strict. E.g., a strict variation of `Sum3` is defined like this:

{% highlight haskell %}
data Sum3 v1 v2 v3 =
  Sum3_1 !v1 | Sum3_2 !v2 | Sum3_3 !v3
{% endhighlight %}

## Type identity

Since the types are first-class, you can use them with aliases or directly in function signatures. However there are cases when you might need to introduce an identity to the type. E.g., when there is a recursion or when you plan to make the type abstract. In such cases you can just use the `newtype` construct. There's no overhead.

E.g., here's how you could define a linked-list:

{% highlight haskell %}
newtype List a =
  List (() + a * List a)
{% endhighlight %}

Neat, right? Looks like a formula and makes total sense!

## Sum-types vs. Either

Let's say you need a sum-type with 4 cases: `Int`, `Char`, `Bool` and `Text`. One could argue that it would be possible to declare it first-class using `Either`, e.g.:

{% highlight haskell %}
type MySumType =
  Either Int (Either Char (Either Bool Text))
{% endhighlight %}

However several problems are already evident here:

1. The choice of where to branch out is ambiguous. I.e., should it be the definition above or `Either (Either Int (Either Char Bool)) Text`, or any other possible combination? More so, this ambiguity grows exponentially in relation to the amount of branches you introduce.

1. There's only a lazy implementation of `Either`. The "compound-types" library provides both the lazy and strict variants of all of its types.

1. The parameter-types get boxed in `Either` three times. This introduces a memory consumption overhead. The more cases you'll have, the more memory you will lose compared to the multi-arity sum-types, which only do boxing once.

1. Which one is easier on the eye: the above or the following?

    {% highlight haskell %}
    type MySumType =
      Int + Char + Bool + Text
    {% endhighlight %}

## Product-types vs. tuples

Virtually there's no difference for the lazy Product, however for the strict Product there's no tuple alternative. Also the `*` operator syntax makes it consistent with the `+` of Sum.

The `*` symbol is also used for declaration of product-types in some other languages of the ML family.

## Enums

You can simulate enums using type-level literals and the `Proxy` type. E.g.,

{% highlight haskell %}
type Gender =
  Proxy "male" + Proxy "female"
{% endhighlight %}

But I agree, this is not the pretty bit. It introduces a memory and syntactic overhead.

It's worth mentioning though, that this example still proves that all the functionality of ADTs is achievable in a consistent way, although this time it is at a cost.


## Memory footprint

Generally there is no overhead. In cases such as the following the memory consumption is identical:

{% highlight haskell %}
data MyProduct =
  MyProduct Int Char

newtype MyProduct' =
  MyProduct' (Int * Char)
{% endhighlight %}

and

{% highlight haskell %}
data MySum =
  MySum_Int Int | MySum_Char Char

newtype MySum' =
  MySum' (Int + Char)
{% endhighlight %}

However in the following case "compound-types" will introduce an extra box:

{% highlight haskell %}
data MySumAndProduct =
  MySumAndProduct_1 Int Char |
  MySumAndProduct_2 Bool

newtype MySumAndProduct' =
  MySumAndProduct' (Int * Char + Bool)
{% endhighlight %}

And, of course, there are the already mentioned enums.

## Plans

At the current stage the library proves the concept, but it only supports types with arity of up to 7. Extending it to more types or maintaining such a codebase is tough. So it's planned to reimplement the library by generating the according code using Template Haskell. Once it's done, the libary will get the types with insane arities.

The value-level composition using type-classes similar to the `+` and `*` on types is an interesting subject to explore.

## Final words

In the beginning of this post I've mentioned Haskell's ADTs mixing three concepts together: type sum, type product and type identity. Frankly, I've always found this a bit awkward, because it clearly violates the separation of concerns. I find this an important reason why it's tough for the newcomers to get an intuition into them.

Atop of the problems this library solves, it also serves as a proof that ADTs can be decomposed into separate concepts, which are simpler and more flexible. 
Combined with the features that are already present in Haskell, now we get all those concepts in isolation: the sum-types, the product-types and the Haskell's `newtype` construct for zero-cost type identity.

This solution does introduce some overhead in case of enums though. But I love the conceptual clarity, consistency and simplicity behind it.
