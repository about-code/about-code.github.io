---
layout: post
title:  "Angular-CLI v6: Why do I need to build the library everytime I make changes?"
date:   2018-05-22
categories: web-development javascript angular
image: 2018-04-28-large-scale-angular/article.png
related: 2018-04-28-large-scale-angular.md
tags: large-scale, angular
---

In the [first part](./large-scale-angular.html) of this series we showed how to set up and build an Angular project from scratch without using the CLI. The article series is intentionally not using the CLI for educational purposes. We do not aim for dissuading from using the CLI. Our article is purely intended to teach alternative options if the CLI can't be used for whatever reason.

Since the first article Angular-CLI v6 has been released as expected and eventually incorporates *library support*. This means it's getting a lot easier to build package-oriented Angular apps with the CLI. But there remain some differences, of course.

## Do I need to build the library everytime I make changes?

In a CLI-v6-project packages [have to be built prior to building the app in the repo](https://github.com/angular/angular-cli/wiki/stories-create-library#why-do-i-need-to-build-the-library-everytime-i-make-changes) [[1][1]]. In [Large Scale Angular from Scratch Part 1](./large-scale-angular.html) we proposed a different configuration which uses a generic `paths` config in `tsconfig.json` [[2][2]]. This approach was basically referred to in [[1][1]] when writing:

> *"Some similar setups instead add the path to the source code directly inside tsconfig."* [[1][1]]

The outstanding property of both setups is to being able to write code which imports sources from packages in the same repo *by package name* rather than *relative paths*, for example:

`import {Foo} from "@foo/bar`;

rather than something like:

`import {Foo} from "../../packages/@foo/bar/src/Foo";`

This increases long-term maintainability of a codebase by an order of a magnitude. With Angular-CLI, though, the package `@foo/bar` has to be built into [Angular Package Format](https://docs.google.com/document/d/1CZC2rcpxffTDfRDs6p1cfbmKNLA6x5O-NtkJglDaBVs/preview#!) (APF) [[3][3]] after each change to it. When building a dependent library or application the name `@foo/bar` then resolves to the *flat-es-module* (FESM) output produced previously when building the imported library. While this comes closer to how a published package would be consumed by third parties  *[...] running `ng build my-lib` every time you change a file is bothersome and takes time.* [[1][1]]

In our setup [[2][2]] *webpack* and *TypeScript* are configured to resolve `@foo/bar` to a package's `index.ts` source file which in turn points onto `src/Foo` by means of an `export`. In short, webpack includes individual modules of a package like any other source file, transpiles it and compiles it into one or more chunks as part of building the app.
There's no need to build packages into APF. APF is a *package distribution format*. It isn't required to build an app.

> *"This makes seeing changes in your app faster. But doing that is risky. When you do that, the build system for your app is building the library as well. But your library is built using a different build system than your app."* [[1][1]]

While this is technically correct we see room for debate about how to handle the risks. Having to rebuild a library into APF format after every change increases build times. With our setup we would recommend to run tests against pre-built packages only at particular points in time, e.g. as part of a package release workflow.

## Finding the Balance between Quality and Efficiency

In finding the right balance between package quality and development efficiency the CLI team decided *"to err on the side of caution, and make the recommended usage the safe one"* [[1][1]] which is reasonable given the large and diverse user base of the CLI.

In our setup we priortize development efficiency over risk mitigation. If we use the term *efficiency* we exclude one-time efforts such as initial ramp-up time (also remember, we intentionally chose not to use CLI tooling). So if we talk about *efficiency* we actually mean build-times since we build things so often a day that it matters most to us.

## Primary and Secondary Goals

The CLI v6 approach is a *fail-fast* approach, meaning to find potential errors fast. Such an approach might be preferable indeed if the primary project goal is library development. If the goal is application development, though, a package-oriented development model often serves only secondary goals (code reuse, maintainability, etc). The primary goal is delivering customer value as soon as possible. As long as the application works it doesn't matter whether it was compiled from pre-built packages in APF format or package sources.

Risks to secondary goals such as sharing bug-free APF packages with others lose importance. When code reuse is only a secondary goal then anything related to a package-build is certainly to become a secondary goal as well. Therefore instead of pre-building packages after every change it is often acceptable to test an application against pre-built versions of the packages only from time to time. For example, when the app development team is ready to release new versions of its packages and when the project situation allows for fixing bugs in a package-build.

## Technical Risks vs. Management Risks

Even if it sounds selfish, bugs stemming from misconfigured package-builds may "only" affect third parties or other teams. In many organizations this is enough for a team-lead to vote for efficiency over trying to avoid risks only relevant to others. If there is pressure from other departments to fix broken packages then the situation even serves as an argument to split up a monorepo held by your team and contribute shared packages to a shared repository which is maintained collaboratively. This way a team can even benefit from others working at those part's of the app which matter to them as well. If they are reluctant to contribute, a bit of pain can even help because their team-lead is likely to be as willing to reach her or his goal as much as your team-lead is. Code sharing comes with shared responsibility. Let them contribute their fixes.

<!--
## Why we think different is good as well

Packages in our setup are plain Angular/TypeScript packages and should be consumable from their sources exactly the same way as they would be consumed after a package built, because in the end we consume ES2015 modules. The package-build produces an optimized flat ES2015 module or bundles the sources for some additional third-party consumption patterns like UMD modules. Be we do not yet see how this justifies to run a package-build continuously. We are not sure what other features [1] refers to, though. A package with very special build requirements could be developed in a separate repo alltogether. Alternatively it could be pre-built individually and TypeScript and webpack could be configured such that they resolve *this particular* package name to the path of the package-build output. But pre-building *all* packages is often not necessary (in app development) from our point of view.
-->
## Package releases are less frequent than App builds

We consider package releases to be a lot less frequent than code changes to packages, especially in their incubation phase within an application project. Testing the app against the results of a package-build too early can even be hindering. Making those tests a *quality gate*  prior to release should be well enough to mitigate risks from different build processes.

Furthermore we think a package-build wouldn't change constantly, so once we can verify that a package-build works there's no reason to assume that every change to a TypeScript file in a package is going to break the complete build. Functional errors can also be detected when compiling an app from package sources. There are only a few situations we can imagine where a package-build should be verified:

- when preparing a package release
- when adding new external dependencies
- when changing build features like adding SASS support when the package previously contained TypeScript files, only

## Outdated packages are a risk as well

> *"Running `ng build my-lib` every time you change a file is bothersome and takes time. [...] In the future we want to add watch support to building libraries so it is faster to see changes. We are also planning to add internal dependency support to Angular CLI. This means that Angular CLI would know your app depends on your library, and automatically rebuilds the library when the app needs it."* [[1][1]]

Until watch support is implemented it's easy to forget triggering a build manually even more when changing multiple packages at once. From a personal experience it can happen multiple times during a day. This also raises the risk to test against packages not containing the latest changes and is likey to cause additional friction in the development process. When changing multiple packages they have not just to be rebuilt - they have to be rebuilt *in order*. Doing this manually is error prone and requires to know exactly the package inter-dependencies. From the information given in [[1][1]] we can't see whether this is to be addressed by *dependency support*. Watch support is likely to work sufficiently well most of the time if a developer saves files one-by-one. But attention needs to be put on *save all files at once* scenarios. We guess full dependency resolution is required to handle the correct build order in those cases.

## Architecture and Team-Dynamics

When new packages are created initial changes are very frequent. From a socio-dynamic perspective there's an architectural risk if people try to avoid creating packages where they would actually make sense. They should not find themeselves in a situation of having to defend their (right) decision against accusations of slowing down productivity for the whole team. The *single responsibility principle* is an important architectural pillar - important enough to be the first letter in the SOLID principles. So creation of packages should be as easy as possible (which is the case with CLI v6) but builds should also remain as fast as possible (which might improve with CLI v6+).

## Releasing packages might not be a goal at all

Constructing an app from packages as a matter of good design provides a value on its own even if package distribution is not even a secondary goal. We can imagine projects where packages are only created for the sake of an inner structural quality of an app with no plans for distribution. In these cases there is also no need to constantly build a package into a distribution format like APF.

## Summary

In this post we tried to outline why we think the risks mentioned in the CLI wiki [[1][1]] regarding the way our project sample is being configured and built in [[2][2]] is managable. We tried to argue for comparing the risks mentioned in [[1][1]] to the project priorities. Further we think testing an app with pre-built packages only from time to time provides a similar level of safety than doing so constantly for every change. Nevertheless we cannot yet present a sample project with a package-release workflow supporting those tests.

## References

[1]: https://github.com/angular/angular-cli/wiki/stories-create-library/1cf783837c392f5fadc7286e1fb28220b9a1b507
\[1\] *Library support in Angular CLI 6*. Google Inc. 2018. Last updated on 2018-05-03. Last read on 2018-05-03 15:48 +0100. https://github.com/angular/angular-cli/wiki/stories-create-library#why-do-i-need-to-build-the-library-everytime-i-make-changes

[2]: https://about-code.github.io/web-development/javascript/angular/large-scale-angular.html
\[2\] Large Scale Angular from Scratch Part 1. Last updated on 2018-04-28. Last read on 2018-05-03 12:48 +0100. https://about-code.github.io/web-development/javascript/angular/large-scale-angular.html

[3]: https://docs.google.com/document/d/1CZC2rcpxffTDfRDs6p1cfbmKNLA6x5O-NtkJglDaBVs/preview#
\[3\] *Angular Package Format*. Google Inc. 2018. https://docs.google.com/document/d/1CZC2rcpxffTDfRDs6p1cfbmKNLA6x5O-NtkJglDaBVs/preview#

<!--
## Appendix

### A: Angular Package Format, Why and When?

A package-build creates a file structure and bundles up package source files to conform to [Angular Package Format](https://docs.google.com/document/d/1CZC2rcpxffTDfRDs6p1cfbmKNLA6x5O-NtkJglDaBVs/preview#!).

Angular Package Format

1. describes how to distribute npm packages from TypeScript or ECMAScript2015 sources to be compatible with legacy module patterns like UMD or CommonJS
1. defines special requirements on packages which provide NgComponents or NgModules in order to enable *Ahead-of-Time-Compilation* for applications consuming the package.

But APF is **only required when publishing  packages to a registry**. Published packages must be pre-compiled because

1. it is bad practice to publish TypeScript packages to a JavaScript eco system
1. TS `strict` mode isn't backward compatible. Older TS sources may not compile with newer TS versions in `strict` mode
1. without TS package sources available the *Angular-Ahead-of-Time-Compiler* of a package consumer requires additional metadata
-->
