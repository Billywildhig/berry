---
category: features
slug: /features/pnp
title: "Plug'n'Play"
description: An overview of Plug'n'Play, a powerful and innovative installation strategy for Node.
---

## What is Yarn Plug'n'Play?

Yarn Plug'n'Play (generally referred to as Yarn PnP) is the default installation strategy in modern releases of Yarn. It can be swapped out for more traditional approaches (including `node_modules` installs, or pnpm-style symlink-based approaches), but we recommend it when creating new projects due to its numerous improvements.

## How does it work?

If you look into the files in your project, you may notice the absence of a `node_modules` folder. This is unusual! We regularly get asked on Discord where the folder is, by people thinking `yarn install` silently failed.

The thing is, this is actually expected! The way Yarn PnP works, it tells Yarn to generate a single Node.js loader file in place of the typical `node_modules` folder. This loader file, named `.pnp.cjs`, contains all information about your project's dependency tree, informing your tools as to the location of the packages on the disk and letting them know how to resolve require and import calls.

## What are the advantages?

Yarn PnP addresses various problems. It would be possible to address _some_ of them via smarter `node_modules` layout algorithms (that's for example what the pnpm-style symlink-based install strategy attempts to do), but PnP is the only strategy that can address them all:

### Minimal install footprint

A Yarn PnP install typically does one thing: generate the Node.js loader file (`.pnp.cjs`). In other package managers, a significant portion of the time is spent performing I/O operations to copy files from one location to another, be it on disk like npm, or via symlinks / hardlinks like pnpm.

### Shared installs across disks

Related to the previous point, Yarn PnP allows to reuse the same package artifacts across all projects on the disk. Unlike pnpm, which uses a content-addressable store where each file from each package needs to be hardlinked into its final destination, the PnP loader directly references packages via their cache path, removing a lot of complexity.

### Perfect and correct hoisting

Typical `node_modules` installs attempt to optimize the resulting `node_modules` size by hoisting packages, at the cost of higher risks of ghost dependencies. Unfortunately, even these optimizations have limits! Some dependency patterns prevent safe hoisting, leading to package duplication and multiple instantiations.

### Ghost dependencies protection

Because Yarn keeps a list of all packages and their dependencies, it can prevent accesses to dependencies unaccounted for during resolution, giving you the ability to quickly identify and fix those problems before they get deep into your codebase and jeopardize the stability of your application at deploy time.

:::info
This is sometimes mentioned as a challenge to adopting Yarn PnP. It means errors may be reported when other package managers would seem to work out of the box - that is, until strange breakages start happening as you add, upgrade, or remove unrelated dependencies.

While it does add a bit of friction, it's a critical part of what makes Yarn a **very** stable package manager. An application that works today won't suddenly break in the future, and your colleagues won't face seemingly random issues long after your PRs got merged.
:::

### Semantic erroring

You may never have noticed it, but when a Node.js import or require call is invalid, you only get a generic error in return, that doesn't really tell you what's the problem or how to address it:

```
Uncaught Error: Cannot find module 'not-found'
```

Yarn PnP not only tells you exactly *what* the problem is, but also which packages are involved. For example, the two following error messages may be emitted depending on the circumstances:

```
Error: Your application tried to access not-found, but it isn't declared in your dependencies; this makes the require call ambiguous and unsound.

Required package: not-found
Required by: /path/to/my-project/
```

```
Error: awesome-plugin tried to access awesome-core (a peer dependency) but it isn't provided by its ancestors; this makes the require call ambiguous and unsound.

Required package: awesome-plugin
Required by: awesome-core

Ancestor breaking the chain: awesome-template
```

Semantic erroring goes a long way into letting you understand and address problems caused by your dependencies.

## Is it difficult to use?

### When creating a new project

If you're creating a project from scratch, your project itself should work almost "out of the box". You may have to use `packageExtensions` from time to time to fix an occasional ghost dependency, but that remains uncommon, and this process is otherwise straightforward. Most tools in the ecosystem are designed and tested to work well in Yarn PnP environments, so problems are infrequent.

:::warn
A notable exception is React Native / Expo, which require using typical `node_modules` installs.
:::

Really, the main problem you will face will be around IDE integrations. All IDEs have some level of support for Yarn PnP, but in general you should expect having to follow one of the procedures from this guide to make sure all your imports are properly resolved.

### When migrating an existing project

:::info
Running `yarn install` in a project which used to be installed by Yarn Classic will cause Yarn PnP to be automatically disabled, to make the migration smoother. You'll still benefit from the enhanced stability and other features implemented in modern releases, and can decide whether to spend the time to migrate to PnP or not at a later time.
:::

Existing projects can be tougher to migrate to Yarn PnP for a couple of reasons:

- You already start with a lot of dependencies, so there'll be a proportionally higher amount of packages that may list ghost dependencies
- They may be locked on old versions of their respective packages, and thus have a higher chance to contain ghost dependencies
- Your own scripts may inadvertently rely on some implementation details or ghost dependencies, sometimes even without you realizing it.

None of these are blockers, but they mean it can take a couple of days to migrate an existing project to Yarn PnP. We however provide tools to simplify some of this process, and taking a look at the footguns below will help you identify quicker what way cause something to break, so it's not impossible.

Remember that migrating to Yarn PnP is optional: you can revert to `node_modules` installs at any time by setting the `nodeLinker: node-modules` setting in your project's `.yarnrc.yml` file.

## Footguns

### Peer dependencies

Peer dependencies are powerful, but are very difficult to implement - even more so for non-PnP projects, which have to work within the limits of what the filesystem hierarchy allows.

Yarn PnP, on the other hand, doesn't have this limitation, and will accurately represent the peer dependencies of every project in your dependency tree - even workspaces. If a workspace has a peer dependency, and if this dependency is fulfilled by different versions depending on its grandparent, then the workspace will instantiated twice, once for each unique "dependency set".

This is the correct behaviour, but it may cause an accidental explosion of the number of instantiated workspaces if your project heavily uses peer dependencies without ensuring they are always fulfilled by the exact same versions.

### Shared binaries

Yarn prevents ghost dependencies in the packages your project depends on, but also in your own code itself - this is to decrease the chances that a package would work on your development machine but break once published.

It however has a side effect when it comes to bins. If you have `typescript` listed at the root of your project, the `tsc` binary will be available in the root package _but only in the root project_. In other words, any workspace using the `tsc` binary in its scripts will need to declare it in its dependencies.

A good recommendation to avoid this kind of issue is to have a "tooling" workspace to contain your infrastructure tools and scripts, and have all other workspaces depend on it.

## Frequently asked questions

### Compatibility with npm / pnpm

Yarn PnP was designed to use the exact same "public interfaces" as other package managers, with differences being kept to what already were implementation details. If a project works with Yarn PnP, it should work everywhere!

One caveat though: the opposite isn't always true. Since other package managers don't / can't enforce proper listing of dependencies, they are more vulnerable to shipping ghost dependencies by accident to their consumers. In that way, using Yarn PnP can be seen as a good practice for the health of the ecosystem! 🙂

### How can I fix ghost dependencies?
