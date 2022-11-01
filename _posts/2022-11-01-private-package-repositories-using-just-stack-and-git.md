---
layout: post
title: Private package repositories using just Stack and Git
tags: [haskell, stack, git, package-repository, packages, repositories]
comments: false

---

If you're working in an organization that maintains a private Haskell codebase chances are you've stumbled upon the problem of distribution of packages. The typical approaches to this are either to avoid that altogether by maintaining the whole codebase of the org as a monorepo or to manage a private Hackage server. Both come with their own sets of problems. The first tends to entangle the codebase and makes it hard to introduce teams and isolate the areas of responsibility. The second introduces an adminstrative burden and complicates the development setup. Today I'm gonna share a different technique which I've found to be both simpler and correct.

<section id="table-of-contents" class="toc">
  <header>
    <h3>Contents</h3>
  </header>
  <div id="drawer" markdown="1"> 
  *  Auto generated table of contents
  {:toc}
  </div>
</section><!-- /#table-of-contents -->

# Concepts

**Stack** has the concept of **Snapshots**. The snapshot specifies a set of packages at specific versions and the information on where to get them from. The location can be a Hackage package, a tarball or a git-repository at a specific commit. Standard snapshots get distributed on [stackage.org](https://stackage.org), but we also can specify our own which can extend the standard ones. We can refer to them by URLs or by local paths in the `resolver` field of the `stack.yaml` file.

**Git** has a **Submodules** feature. It lets you bundle one repo into another and control which commit it's at. The contents of the submodule repository get pulled as if they were just a directory in the main repo. Another important thing is that Git also handles the access to the repos, so both can be private and once you have the SSH keys configured, the experience will be smooth.

This is all we need.

# The technique

The technique is about maintaining a shared Stack snapshot in a dedicated repo and bringing it into the Haskell codebase repos as a submodule and referring to the snapshot file from `stack.yaml`.

Whatever you want to specify in the `extra-deps` field of `stack.yaml` now moves into the `packages` field of the snapshot file. That includes your private packages.

# Example

Let's imagine having the following repositories in our org:

- "api" - web-server application
- "db" - database client SDK
- "logic" - business-logic
- "utils" - shared utils

"api" depends on "db" and "logic" and all of them depend on "utils".

## 1. Create the "stack-snapshot" repository

The repository will contain just one file: `snapshot.yaml`. Let's assume that we don't yet have any other repositories, so the contents of it are pretty simple:

```yaml
name: our-org-snapshot
resolver: nightly-2022-10-25
```

The `resolver` field mentions a standard Stackage snapshot.

Push the repo. Let's say to https://github.com/our-org/stack-snapshot.

## 2. Create the "utils" repository

Create a Cabal package with your great extensions over the basic libraries that you plan to use all over the org, your custom prelude and etc.

Use Git to add https://github.com/our-org/stack-snapshot as a submodule and point it to the `.stack-snapshot` directory.

Add a `stack.yaml` file with the following contents:

```yaml
resolver: .stack-snapshot/snapshot.yaml
```

Push the changes and copy the commit hash.

## 3. Update the "stack-snapshot" repository

Add the "utils" repository to the snapshot pointing to the commit hash that you've just copied.

```yaml
resolver: .stack-snapshot/snapshot.yaml
packages:
  - git: https://github.com/our-org/utils
    commit: <commit-hash>
```

Push the changes.

## 4. Create the "db" and "logic" repositories

By repeating the previous two steps for both. You'll end up with the following stack-snapshot:

```yaml
resolver: .stack-snapshot/snapshot.yaml
packages:
  - git: https://github.com/our-org/utils
    commit: <commit-hash>
  - git: https://github.com/our-org/db
    commit: <commit-hash>
  - git: https://github.com/our-org/logic
    commit: <commit-hash>
```

## 5. Create the "api" repository

The process is still the same: repeat the step 2 and 3. Notice that all the dependencies of "api" are already listed in the snapshot, so the build will go smoothly.

## 6. Introduce a change to the "utils" library

This is to showcase how the changes get introduced in this system.

Update the "utils" lib, push and copy the commit hash. Update the hash in the snapshot repo and push. Go to the "api" repo and do a `git pull` from the `.stack-snapshot` subdirectory. Now the "api" repo has the reference to an updated snapshot staged. You can commit that. And voila, your "api" repo is set to use the latest versions of its deps.

# Helpful tools

## Push and copy commit hash script

To do this in a single action I use the following script on Mac:

```bash
#!/bin/bash
git log -1 --format="%H" | tr -d '\n' | pbcopy && git push
```

## Automation

The manual actions that need to be taken to publish a repo can actually be automated using CI. E.g., you can set up a process which builds and tests the package that you want to publish and when it succeeds it will update the "stack-snapshot" repo. Thus you'll both automate the repetitive actions and ensure that only valid packages get propagated.

E.g., this can be done using GitHub Actions. Although we don't yet have such an action yet, it shouldn't be hard to implement.

# Testimony

I've successfully applied this approach in my private projects for a couple of years now. E.g., [pGenie](https://pgenie.io) relies on it heavily.
