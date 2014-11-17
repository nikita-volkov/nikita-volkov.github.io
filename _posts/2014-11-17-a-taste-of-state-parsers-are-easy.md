---
layout: post
title: "A taste of State: parsers are easy"
tags: [haskell, state, parser, parsing, monad, transformer, postgres, postgresql, binary, format, c, decoder]
description: A story of how I translated a C decoder to Haskell
comments: true

---

Like many other Haskellers even after considering myself an expert in the language I still get that amazing "wow" moment, when I learn another elegant thing about its basic concepts, from time to time. 

Recently I was working on a project, which required a primitive parsing functionality, and I discovered that amazing results in terms of performance can be achieved using the good old _State_ monad. This post is about what I discovered and how.

## Introduction

So I was working on [a library of codecs for the _PostgreSQL_'s _Binary Format_](http://hackage.haskell.org/package/postgresql-binary) as part of the [_Hasql_](http://hackage.haskell.org/package/hasql)'s ecosystem. The first thing I discovered was that _Postgres_' authors are very bad at documentation. Google all day, and you won't find even a reminder of specs of the _Binary Format_. Nada. Though very frustrated by that fact I was still determined, so I found a solution.

After digging around I found [the _"libpqtypes"_ library](http://libpqtypes.esilo.com/), which implemented the format in _C_. Luckily it was open source, so I decided to use it as a sort of a spec. My discoveries concerning the _State_ monad hit me, when I was translating that _C_ code.

## About binary encodings

Prior to going further I have to say a little about binary encodings. An important thing about them is that unlike human-readable formats they are strictly positional. This means that contexts are always unambiguous and are always declared ahead. One typical example is the following:

    byte1, byte2, byte3, byte4, byte5, byte6, ..., byteN
              HEAD             |           BODY         

Where the first four bytes are the head, which encodes a 32-bit integer (8 * 4 = 32). This integer specifies the amount of the following bytes, which make up the body. While this isn't the only possible scenario, it proves that it is possible to encode a data of an arbitrary length without any delimiters. So no quotes, braces, commas, dashes or anything like that to distinguish contexts.

Here's another example:

    byte1, byte2, byte3, ..., byteN
     HEAD |         BODY

The single byte of the head is a _Word8_, which determines the alternative scenarios of how the body should be decoded. E.g., if it is `0`, then the body is a text, if it is `1`, then it is a list of ints and etc.

Turns out, mixing the aforementioned strategies together gives you all you need to encode pretty much anything. This means that a parser for this encoding will require no backtracking or alternatives, or any other complex strategies needed to parse human-readable formats, and this is what makes the _State_ monad sufficient for implementing a binary parser.

## The parser

Let's start with a code snippet from the _"libpqtypes"_ library. It is extracted from a function, which is supposed to decode a value of _Postgres_' _interval_ type from a binary format.

{% highlight c %}
pqt_swap8(tvalbuf, value, 0);
days = pqt_buf_getint4(value + 8);
mons = pqt_buf_getint4(value + 12);
{% endhighlight %}

So what's going on in this piece of code? It's easy to spot that in the first step they must be doing something with the first 8 bytes, the following step somehow associates the next 4 bytes with days and the next step associates another 4 bytes with months. I already know that the first 8-byte section actually encodes an amount of microseconds as an integer.

Let's start translating it by extracting the sections from a byte string:

{% highlight haskell %}
let 
  (micros, byteString') = ByteString.splitAt 8 byteString
  (days, months)        = ByteString.splitAt 4 byteString'
{% endhighlight %}

Now we have an input byte string split into 3 useful sections, which we can later process. When talking about such things as sections or parts a seasoned Haskeller might instinctively start thinking about composition, but hold that thought for now. 

In our code there's already an easily spottable pattern and an annoying and error-prone noise caused by the explicit updating of `byteString`. Imagine having more sections to split into and accidentally confusing `byteString'` with `byteString''` - that's a bug the compiler won't spot for you, since the types are the same. 

Turns out, this pattern is abstracted from long ago, and it's the place where the _State_ monad steps in. Let's refactor.

{% highlight haskell %}
do
  micros <- state $ ByteString.splitAt 8
  days   <- state $ ByteString.splitAt 4
  months <- get
{% endhighlight %}

So much better. Sure, it's one extra line of code, but it's so much easier to reason about. But wait, there's a new pattern right there, let's refactor again.

{% highlight haskell %}
do
  micros <- bsOfSize 8
  days   <- bsOfSize 4
  months <- get
  ...
where
  bsOfSize = state . ByteString.splitAt
{% endhighlight %}

And here comes the composition! A smaller independent piece is used to make up a bigger one of the same type.

Okay, so we have our three sections now, but it's still all binary data, which we need to decode. I already know that months and days are integers encoded in the _Big Endian_ format. A decoder for it is pretty simple:

{% highlight haskell %}
decodeInt :: (Bits a, Num a) => ByteString -> a
decodeInt = 
  ByteString.foldl' (\n h -> shiftL n 8 .|. fromIntegral h) 0
{% endhighlight %}

I won't dive into details of the implementation here, since it's not related to the subject. Just accept it as a given.

Using this function we can now decode each section into a number:

{% highlight haskell %}
do
  micros <- bsOfSize 8
  days   <- bsOfSize 4
  months <- get
  let
    microsInt = decodeInt micros
    daysInt   = decodeInt days
    monthsInt = decodeInt months
  ...
where
  bsOfSize = 
    state . ByteString.splitAt
  decodeInt = 
    ByteString.foldl' (\n h -> (n `shiftL` 8) .|. fromIntegral h) 0
{% endhighlight %}

Another pattern. Refactoring:

{% highlight haskell %}
do
  micros <- decodeInt <$> bsOfSize 8
  days   <- decodeInt <$> bsOfSize 4
  months <- decodeInt <$> get
  ...
where
  bsOfSize = 
    state . ByteString.splitAt
  decodeInt = 
    ByteString.foldl' (\n h -> (n `shiftL` 8) .|. fromIntegral h) 0
{% endhighlight %}

Still a pattern. Refactoring:

{% highlight haskell %}
do
  micros <- intOfSize 8
  days   <- intOfSize 4
  months <- intOfSize 4
  ...
where
  bsOfSize = 
    state . ByteString.splitAt
  decodeInt = 
    ByteString.foldl' (\n h -> (n `shiftL` 8) .|. fromIntegral h) 0
  intOfSize = 
    fmap decodeInt . bsOfSize
{% endhighlight %}

And now it seriously starts to look like a typical parser library in action. You might have spotted though that during our last refactoring we've introduced a little overhead by redundantly splitting the bytestring remainder in the _months_ parser, but it's a little price to pay for a general solution. 

Let's finish our parser:

{% highlight haskell %}
-- | Decode an interval as an amount of picoseconds.
interval :: ByteString -> Integer
interval =
  evalState $
    combineInterval <$> intOfSize 8 <*> intOfSize 4 <*> intOfSize 4
  where
    bsOfSize = 
      state . ByteString.splitAt
    decodeInt = 
      ByteString.foldl' (\n h -> (n `shiftL` 8) .|. fromIntegral h) 0
    intOfSize = 
      fmap decodeInt . bsOfSize
    combineInterval u d m =
      10 ^ 6 * (u + 10 ^ 6 * 60 * 60 * 24 * (d + 31 * m))
{% endhighlight %}

Look at the _where_ section. There's a whole library right there! Let's export it.

{% highlight haskell %}
type Parser = 
  State ByteString

run :: Parser a -> ByteString -> a
run = 
  evalState

bsOfSize :: Int -> Parser ByteString
bsOfSize = 
  state . ByteString.splitAt

intOfSize :: (Bits a, Num a) => Int -> Parser a
intOfSize = 
  fmap decodeInt . bsOfSize
  where
    decodeInt = 
      ByteString.foldl' (\n h -> (n `shiftL` 8) .|. fromIntegral h) 0

interval :: Parser Integer
interval =
  combine <$> intOfSize 8 <*> intOfSize 4 <*> intOfSize 4
  where
    combine u d m =
      10 ^ 6 * (u + 10 ^ 6 * 60 * 60 * 24 * (d + 31 * m))
{% endhighlight %}

Neat! Our own library of composable components. We could of course spend our day shifting indexes, constantly repeating ourselves and hoping we don't mess up, like they do in _C_, but that's not how we roll in _Haskell_! We make reusable libraries out of nowhere, without even intending to do so.

You might be wondering now about the parsers that can fail. Turns out, the following updates are all it takes to implement the support for that:

{% highlight diff %}
- type Parser = State ByteString
+ type Parser = StateT ByteString Either

- run = evalState
+ run = evalStateT
{% endhighlight %}

Want your parser to be a transformer? Use _EitherT_ instead.

Now, as I understand, the mysterious _Zepto_ parser from the _"attoparsec"_ library is essentially the same thing as the one we've just implemented. In the benchmarks I ran it was performing on par. However the performance of both _Zepto_ and our own parser for the described task is an order of magnitude better compared to _Attoparsec_, forget about _Parsec_.

## An extra taste

Aside from everything related to parsers I want to give another example I extracted during that project of how nicely the _State_ monad abstracts from other typical imperative patterns.

_C_ (from _libpqtypes_):

{% highlight c %}
int   putit = (d > 0);

d1 = dig / 1000;
dig -= d1 * 1000;
putit |= (d1 > 0);
if (putit)
  *cp++ = (char) (d1 + '0');
d1 = dig / 100;
dig -= d1 * 100;
putit |= (d1 > 0);
if (putit)
  *cp++ = (char) (d1 + '0');
d1 = dig / 10;
dig -= d1 * 10;
putit |= (d1 > 0);
if (putit)
  *cp++ = (char) (d1 + '0');
*cp++ = (char) (dig + '0');
{% endhighlight %}

_Haskell_:

{% highlight haskell %}
map (chr . (+) (ord '0')) $ dropWhile (== 0) $ evalState $ do
  a <- state $ flip divMod 1000
  b <- state $ flip divMod 100
  c <- state $ flip divMod 10
  d <- get
  return $ [a, b, c, d]
{% endhighlight %}

And another more practical function:

{% highlight haskell %}
microsToTimeOfDay :: Int64 -> TimeOfDay
microsToTimeOfDay =
  evalState $ do
    h <- state $ flip divMod $ 10 ^ 6 * 60 * 60
    m <- state $ flip divMod $ 10 ^ 6 * 60
    u <- get
    return $
      TimeOfDay (fromIntegral h) (fromIntegral m) (microsToPico u)
{% endhighlight %}

You might have noticed a certain similarity between the _splitAt_ and the _divMod_ functions - it's that they both result in a pair. So the next time you'll have to deal with a function returning a pair, you will know that the _State_ monad might well turn out to be not any less useful for its composition.

Cheers!
