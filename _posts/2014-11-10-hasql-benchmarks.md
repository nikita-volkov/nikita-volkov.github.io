---
layout: post
title: Hasql benchmarks
tags: [haskell, postgresql, postgres, database, sql, benchmarks, performance, hasql]
description: 
comments: true

---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Contents</h3>
  </header>
  <div id="drawer" markdown="1"> 
  *  Auto generated table of contents
  {:toc}
  </div>
</section><!-- /#table-of-contents -->

This post is all about the performance of [the "hasql" library](http://hackage.haskell.org/package/hasql) and particularly [its PostgreSQL backend](http://hackage.haskell.org/package/hasql-postgres) in comparison to its popular direct competitors: "HDBC" and "postgresql-simple".

## Benchmarks

The source of the PostgreSQL backend comes with [a suite of 3 benchmarks](https://github.com/nikita-volkov/hasql-postgres/blob/57b79ba6e28bb6b3b2965a1802e237a4bbc190ed/competition/Main.hs). I'll analyze them one by one.

### Results parsing

In this benchmark a simple `SELECT` query without any parameters is performed. This eliminates the overhead related to encoding of parameters and leaves us with execution of the query and decoding of its results. Decoding of results is actually what we're trying to compare here.

And here are the results:

[![Results parsing](/assets{{page.id}}/results-parsing.png)](/assets{{page.id}}/results-parsing.png) 
(<a href="/assets{{page.id}}/results-parsing.html" target="_blank">HTML version</a>)

We see that Hasql is 2.5 times faster than "postgresql-simple" and 8 times faster than "HDBC".

### Templates and rendering

In this example a `SELECT` query is performed as well. Only this time there are parameters to encode and no results get decoded. So, what we measure is how well the subject deals with rendering of parameters and queries.

[![Templates and rendering](/assets{{page.id}}/templates-and-rendering.png)](/assets{{page.id}}/templates-and-rendering.png) 
(<a href="/assets{{page.id}}/templates-and-rendering.html" target="_blank">HTML version</a>)

We see that Hasql is 1.5 times faster than "postgresql-simple" and 3 times faster than "HDBC".

### Writing transaction

This is the most general benchmark. It includes both encoding and decoding on top of transaction mechanics. It simulates a bunch of transfers from one account to another.

[![Writing transaction](/assets{{page.id}}/writing-transaction.png)](/assets{{page.id}}/writing-transaction.png) 
(<a href="/assets{{page.id}}/writing-transaction.html" target="_blank">HTML version</a>)

Here the difference is more suddle: Hasql is 1.18 times faster than "postgresql-simple" and 1.58 times faster than "HDBC".

## What makes Hasql so fast?

So we've seen that Hasql dominates in every case. There are fundamentally different design decisions from its competitors that cause this:

* Unlike in "HDBC", the font-end API brings no intermediate types, which eradicates unneccessary conversions.

* Unlike "postgresql-simple", the library utilizes parametric queries instead of recompiling them every time they are used.

* Unlike "postgresql-simple", the library utilizes prepared statements, which requires the database to parse the SQL statements only once per connection.

* Unlike both "HDBC" and "postgresql-simple", the PostgreSQL backend of Hasql uses a binary format for communication, which eradicates a bunch of overhead related to parsing, rendering and transfering of values on both ends: the library and the database itself.


