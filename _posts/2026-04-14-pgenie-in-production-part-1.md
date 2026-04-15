---
layout: post
title: My 14-Year Journey Away from ORMs - How I Built pGenie, the SQL-First Postgres Code Generator
description: Story about how I went from shipping a popular ORM in 2012... to throwing it all away... to realizing that the database itself should be the single source of truth.
tags: [pgenie, postgresql, codegen, haskell, rust, java, sql-first, db-first]
comments: false

---

Hi, I'm Nikita Volkov - architect, consultant, and the author of `hasql`, one of the two main PostgreSQL drivers in Haskell used in major production projects like PostgREST and IHP. After 25 years in IT and more late-night schema-drift fires than I care to count, I open-sourced something I wish had existed a decade ago: **[pGenie](https://pgenie.io?utm_source=blog&utm_campaign=series-part1)**.

Following is the story of how I went from shipping a popular ORM in 2012... to throwing it all away... to realizing that the database itself should be the single source of truth.

### 2012: I Built an ORM

Back then I was a typical web developer of the time. CRUD was all we needed. Performance problems were for "later". Deployments were `ssh` + `git pull`. Migrations were non-existent.

So I wrote **SORM** - an ORM framework for Scala - and open-sourced it. Instead of treating objects as mutable views of the database, it exchanged them as immutable values with the db. It felt novel. It gained traction. For a while it felt like success.

Then reality hit.

- Every non-trivial query exposed the limits of the DSL.  
- Keeping the code model in sync with the real database schema became an endless source of headaches.  
- I was essentially inventing a worse version of SQL just to make the integration "safer".

In 2014 I stopped development. In 2016 I killed support. Lesson learned: **abstracting over SQL is usually a mistake**.

### Insight #1: Instead of replacing SQL - we should make integration with it better

Instead of building another layer on top, I decided to make the integration with raw SQL as safe and convenient as possible.

That led to **hasql** in 2014 - a driver that embraced raw SQL and gave composable abstractions supporting it. It's one of the two main Postgres drivers in the Haskell ecosystem today and the most efficient one.

With this library I found the perfect integration point.

### Insight #2: The Query is the Optimal Integration Point

Not the business logic model. Not the table row structure. **The query itself.**

The query determines the structure of parameters it needs and what it returns. The types that represent those parameters and results are just that - parameters and results **of that particular query**. They are not your table rows, they are not your business models. However similar they may look, they are not the same thing and they **will** evolve independently and become different given enough time.

**Similar ≠ same, looks ≠ identity.**

Reusing row types across queries creates hidden dependencies and plants future failures: change one query and another silently fails. Attaching domain models to database entities escalates this problem to other application layers. Coupling to API types is just a recipe to get either an API violation or a database integration bug.

That perspective made a solid foundation, but something was still missing.

The user had to accompany every query with hand-rolled codecs and the SQL string was lacking any validation. So in 2019 I shipped **hasql-th** - a compile-time SQL syntax checking library based on a port of the PostgreSQL parser. It made things simpler and safer, but still not enough. We needed the queries validated against the actual schema.

I saw projects pop up over the years that tried to solve this problem by introducing a compile-time connection to the database. But that always felt like a hack that made builds unreproducible. I was looking for a solution that could be contained in code in a repository without infrastructure dependencies, and eventually it came in the form of the following insight.

### Insight #3: Migrations + Queries = Database API

Here's the mental model that changed everything for me:

If you treat your Postgres server like a microservice, then by the analogy of REST APIs:

- Migrations = the contract (like OpenAPI schemas)
- Queries = the operations (like OpenAPI endpoints)

Your application is just a client of that API. The database is primary. The application code is secondary.

**This is the essence of the DB-First (or SQL-First) approach.**

Suddenly everything clicked. The schema lives in SQL migrations. The queries live in `.sql` files. All of that gets checked into a repository and becomes a subject to development, testing and review. Everything else - types, encoders, decoders, index recommendations - can be automatically derived and safely integrated into codebases with the help of type-checking compilers.

### 2022–2026: From Closed SaaS Experiment to Open Source

I first released pGenie as SaaS in 2022. Companies didn't like sending their schemas and queries to a third party. Fair enough.

So in 2026 I open-sourced it completely and switched to the OSS+Consulting business model:  

**[github.com/pgenie-io/pgenie](https://github.com/pgenie-io/pgenie)**

It's a simple CLI. You point it at your `migrations/` and `queries/` folders. It spins up a real PostgreSQL container, recreates the schema by running migrations, prepares queries, analyses the information schema, and generates a type-safe client code for various targets via plugins. Currently Haskell, Rust and Java are fully supported as such plugins.

### What pGenie Actually Gives You

- Zero boilerplate type-safe SDKs
- Fidelity to advanced Postgres features (JSONB, arrays, composites, multiranges, extensions, etc.) and queries of any complexity
- Decentralized code generators written in safe, reproducible Dhall
- Signature files that catch schema drift at build time
- Automatic index recommendations and sequential-scan warnings  

The full side-by-side of source SQL and the generated client code for Haskell, Rust and Java is in [the demo repo](https://github.com/pgenie-io/demo).

### How this fits in the AI Era

LLMs are great at writing SQL. They are not that great at ensuring it still works after your next migration.

pGenie closes that gap: AI writes the query, pGenie deterministically verifies it against the real schema, you get a predictably structured type-safe SDK in seconds.

SQL-First + AI = maximum velocity with zero surprises.

---

**Sounds interesting?**

- Learn pGenie in Y minutes: [pgenie.io/docs/tutorials/learn-pgenie-in-y-minutes](https://pgenie.io/docs/tutorials/learn-pgenie-in-y-minutes?utm_source=blog&utm_campaign=series-part1)  
- Ask questions and share ideas: [github.com/pgenie-io/pgenie/discussions](https://github.com/pgenie-io/pgenie/discussions)
- Give it a ⭐ on GitHub: [github.com/pgenie-io/pgenie](https://github.com/pgenie-io/pgenie)
