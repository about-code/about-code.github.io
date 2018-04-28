---
layout: post
title:  "Large Scale Angular from Scratch Part 1"
date:   2018-04-01 12:00:00 +0200
categories: web-development javascript angular
---

Results from the <a href="https://www.ng-conf.org/survey-results-angular-community/" target="_blank">2018 Angular survey</a> list *App Architecture/Layout* at the 8th position of the top 10 things people struggle with and *Getting started manually (w/o the CLI)* at the 11th position of what people think others don't understand. *Building Large Apps* is even the 6th most wanted topic people whish to see at [ng-conf 2018](ng-conf.org).

The reasons why there's considerable interest in these topics might be because there aren't many angular docs or articles out there focusing on requirements of large-scale enterprise app development. Things are likely to improve with Angular CLI v6 which comes with [library support](https://github.com/angular/angular-cli/issues/6510) but it took a long road to get there. Nevertheless, even with library support the CLI may still not be able to provide the flexibility required sometimes in enterprise settings. Thankfully, the CLI developers provide us with great [`@ngtools/webpack`](https://npmjs.com/package/@ngtools/webpack) which allows us to compile an Angular application within a custom project setting and with a custom webpack build.

In this article I am going to propose a project setup which puts an emphasis on building modular apps from *packages*. Since all those packages will be developed in a common source code repository the approach is sometimes referred to as a *monorepo* approach.

We'll be copying parts from a [sample project](https://github.com/about-code/ng-mono-sample). I recommend to [download the ZIP](https://github.com/about-code/ng-mono-sample/archive/master.zip) or clone the project locally with `git clone https://github.com/about-code/ng-mono-sample.git`. After downloading rename the folder to something different, e.g. `ng-mono-sample-files`, because we'll be recreating the same folder in the course of this tutorial.

**Table of Contents**<a name="toc"></a>

- [Architectural Considerations](#architectural_considerations)
- [Setting up the Project Package](#set_up_project)
- [Installing Project Dependencies and Defining the Toolchain](#install_deps)
- [Setting up Git](#set_up_git)
- [Setting up TypeScript](#set_up_typescript)
- [Writing the App](#writing_the_app)
  - Packages
  - Implementing the Core Application Package
  - Implementing the Application Shell
  - Adding a Theme Package
- [Building and Bundling the App](#building_the_app)
  - Requirements
  - Adding Config Files
  - Development and Production Config
  - Development Server Config
  - Common Config
  - Running the Build and Serving the App
- [If you're in trouble](#troubleshooting)
- [Customizing the Sample Project](#customizing_sample)

Part 2 (yet to be written)<a name="toc_part2"></a>
- Setting up Unit Tests with Karma and Jasmine
  - Configuring Karma
  - Writing a Unit Test
  - Running the Tests
- Setting up E2E-Tests with Protractor
- Publishing Packages
- Advanced Workflows

## Architectural Considerations<a name="architectural_considerations"></a>

It can't be emphasized enough how much *project structure affects program structure*. Writing a large piece of software should be considered a *process of developing reusable libraries*. Monorepo structures are particularily suited here. They scale from simple to very large and complex apps *because* they imply a concept of modularity via a `packages` folder. In that folder you develop npm packages! Its just a simple folder but it can be very important. I've seen experienced enterprise developers writing Angular apps based on an Angular-CLI structure or custom project structure never thinking of modularizing their apps beyond `NgModules` - so modularizing them down to the deployment layer which means the layer on which you operate whenever using `npm install`.

As soon as you think in terms of npm packages, though, you automatically think in terms of *public vs. private API, package boundaries, dependencies* etc. The Angular repositories follow the [monorepo approach made popular by the Babel project](https://github.com/babel/babel/blob/master/doc/design/monorepo.md) and thus are actually good examples of how to structure complex pieces of software into...

- **packages**,
- which are **separated by concerns**,
- which adhere to the **single responsibility principle**,
- which **hide private implementation details** behind a public API surface.

Unfortunately **few of these principles of good architecture are really being embraced by the CLI** as of v1.7 or lower. Instead it's easy to generate components, services or NgModules which import each other by **relative paths**. Relative paths create a very tight coupling between project structure and implementation. A (structurally) scalable and maintainable app should import things with relative paths only within a limited scope - e.g. within a package scope. When importing things *accross* package scopes, though, paths should be **absolute**. These absolute paths should refer to a *package name* such that it doesn't matter whether the package is a folder in the source tree of the monorepo or a package being installed from NPM. This flexibility allows to externalize individual packages from an *incubator project's* source tree into a separate repo if they should be shared (reused) and maintained independently. After externalization the incubator project could install the package like any normal npm package *without any changes to its source code*.

<!-- The bad thing aren't relative paths, though.  Its the fact that there is nothing in a CLI project which helps you find out when it *becomes* a bad thing. With package-oriented structure, though, it becomes much more obvious.

### import from *packages* not *modules*
-->

There are some other advantages of monorepos: they don't require a separate build for each package. A build configuration for a monorepo can be very generic such that it is easy to copy the project structure, delete any packages inside and create a new app with new packages -  with almost no changes to tool configs (see chapter [Advanced Workflows](#toc_part2)).

## Setting up the Project Package<a name="set_up_project"></a>

We begin with setting up a directory structure following best practices for *npm packages*. In fact the first package we create will become our *project package*. The project package is not meant to be published. It's a *private package* whose `package.json` manifest declares the dependencies required to build the project.

> Throughout the article we'll use the following package naming conventions to distinct between semantic flavours of "packages":
>
> - `-project` suffix for the project package or project directory
> - `-app` suffix for an application's core package (the one bootstrapping Angular, for example)
> - `-feature` suffix for packages implementing app-specific business features
> - arbitrary names which describe the purpose or intent of a package for library packages implementing app-independent generic concerns that can be shared and reused accross applications

Create the project folder and name it e.g. *ng-mono-sample-project* and `cd` into it. We'll refer to this as `${PROJECT_HOME}`.

```
npm init
  // package name: (ng-mono-sample-project) ng-mono-sample-project
  // version: (1.0.0)
  // description: Sample project
  // entry point: (index.js) index.ts
  // test command:
  // git repository:
  // keywords:
  // license: (ISC) SEE LICENSE IN LICENSE.txt
touch README.md LICENSE.txt
mkdir config packages scripts src
```
Now you should have a project folder which looks similar to this one:

```
${PROJECT_HOME}
    |- config/        // tool configurations
    |- packages/      // home of your packages and modules
    |- scripts/       // scripts called via `npm run ...` commands
    |- src/           // application shell (index.html and entry bundles)
    |- LICENSE.txt
    |- package.json   // project manifest (create with `npm init`)
    |- README.md      // explaining the project development setup
```
Open `package.json` and add `"private": true,` as the first line following the opening curly brace. This prevents the project-package from being accidentally published to a npm repo. Fill `README.md` or `LICENSE.txt` later. You'll find open-source license texts e.g. at [https://opensource.org/licenses](https://opensource.org/licenses).

## Installing Project Dependencies and Defining the Toolchain<a name="install_deps"></a>

Finding out what dependencies and tools it takes to build a production-ready Angular app from scratch became a bit of a research task. IMHO Angular docs were better at that in the past but have started to refer to *angular-cli* or the [angular quickstart](https://github.com/angular/quickstart) seed project. The seed project is a little outdated. So when building an angular project from scratch its probably better to generate a temporary project with the cli and look at the `dependencies` section of the generated `package.json`.

For the sake of this tutorial copy the `dependencies` and `devDependencies` section from our [sample project](https://github.com/about-code/ng-mono-sample) and type `npm install` within your `${PROJECT_HOME}`. If you are overwhelmed by the amount of dev dependencies have a closer look at them. There are only a few core tool dependencies while all the other dependencies are helper plug-ins for these tools (e.g. webpack loader plugins) or *typings* to provide optional type information when using JavaScript libraries with TypeScript. In chapter [Building and Bundling the App](#building_and_bundling) we will see which requirements they serve. For the moment you should just see that the actual toolchain is much simpler and consists of these tools and purposes:

- **TypeScript** for writing and transpiling typed JavaScript
- **Webpack** for building and bundling our app
- **Karma** for running unit tests, instrumenting sources and code-coverage reports
- **Jasmine** the testing API for writing unit tests
- **Protractor** for browser automation and ui testing
- **ng-packagr** for building sharable libraries from our *packages*
- **Lerna** for handling a package release process

Our actual application dependencies are common to every Angular project:

- **Angular Packages** (`@angular/*`)
- **RxJS** (Reactive Extensions)
- **Zone.js**
- **Core-JS** (Polyfills)

> **Npm fundamentals:** Additional development tools should be added/installed with `npm i -D <package-name>`. Packages listet in `dependencies` contribute source code which is going to be bundled and deployed with our application. These deps should be added with `npm i <package-name>`.

At the end of this step your project's top-level structure should look like this:

```
${PROJECT_HOME}
  |- config
  |- node_modules/        // New
  |- packages/
  |- scripts/
  |- src/
  |- LICENSE.txt
  |- package-lock.json    // New
  |- package.json
  |- README.md
```
> **Important:** Keep both `package.json` *and* `package-lock.json` under source control and reinstall `node_modules` whenever one of them changes. The lock file fixates exact versions which is important for *reproducable builds*. Without the lock file it may happen that due to version ranges in `package.json` a local build works but a build on the build server or your colleague's machine fails because `npm install` installed an updated version of a package.

> **Hint:**  If you're interested in what all the deps do head over to https://npmjs.com or have a look into the `node_modules` folder. Most packages come with a README.md by convention.


## Setting up Git<a name="set_up_git"></a>

We assume you have [Git](https://www.git-scm.com) installed. Let's create a local Git repository:

```
git init
touch .gitignore
```

This adds a new `.git` folder (may be hidden on Linux systems). Open `.gitignore` and add some file patterns we want to exclude from source code management:

*.gitignore*
```
*.gitignore.*   # Excludes any files with a .gitignore. extension
**/dist
node_modules
```

Adding and comitting files
```
git add -A
git status  // Optional: see what's being committed
git commit -m "Empty unconfigured Angular project"
```

## Setting up TypeScript<a name="set_up_typescript"></a>

When you installed the dependencies you also got TypeScript. Within `${PROJECT_HOME}` run

```
node ./node_modules/typescript/bin/tsc --init
```

to generate an initial `tsconfig.json`. Head over to our [sample project](https://github.com/about-code/ng-mono-sample) again and copy the contents of its `tsconfig.json` into your own. Unfortunately it would be too much to explain all the different TS options. They are well documented in the [TypeScript GitHub Wiki](https://github.com/microsoft/typescript/wiki).  These are the important bits and pieces in our setup:

#### `target`
With `"target": "es5"` we tell TypeScript to transpile to ECMAScript5 **syntax**.

> **Important**: The setting is only about **syntax**. ES2015 *APIs* such as `Map` or `Set` or `Promise` do not exist in an ES5 runtime and require **polyfills**. Further, transpilers can only transpile language features for which an equivalent representation in the target syntax exists. Some completely new ES2015+ features such as *Proxies* may require native support and can't be used with ES5 syntax and runtimes. A good overview of what's possible to use with a target of ES5 is http://babeljs.io/learn-es2015/

#### `module`
ES5 didn't have a native module syntax. There were proprietary systems like [CommonJS](https://www.commonjs.org) or [AMD](https://requirejs.org). ES2015 introduced a native module syntax (ESM) which is statically analysable and supports a form of dead-code elimination called **tree-shaking**: if we have imports which are never used in a file they can be dropped from a final bundle and with them all transitive dependencies used nowhere else. Tree-Shaking will not be carried out by the TypeScript transpiler but by bundlers *after* any transpilation. So we tell TS to transpile to ES5 but keep ES2015 `import` and `export` statements. They'll be then analysed and removed by the bundler. Eventually we will end up with fully compliant ES5 code.

#### `moduleResolution`, `baseUrl` and `paths`

By default the TypeScript module resolution will look for package imports like `import {Foo} from "@scope/my-foo-feature"` in the `node_modules` folder, only. To make the TypeScript compiler find package sources in the `packages` directory but also look into `node_modules` if it's not there, we have to configure it like so:

*baseUrl and generic path mapping*:

```
"baseUrl": "./packages",
"paths": {
    "*": ["*", "../node_modules/*"]
},
```

> Note that this configuration also affects VSCode's auto-import feature. By making `baseUrl` point to the `packages` folder VSCode absolute path suggestions will suggest import paths identical to package names.


#### `angularCompilerOptions`

The Angular compiler is a wrapper around TypeScript which is capable of statically analysing and transforming Angular specific artifacts and code concepts, like decorator meta data or HTML templates. A `tsconfig.json` for an Angular application project needs an `angulerCompilerOptions` section instructing the Angular Ahead-of-Time compiler. If you've copied the config from our [sample project](https://github.com/about-code/ng-mono-sample) it should look like this:

```
"angularCompilerOptions": {
  "entryModule": "./packages/@foo/ng-mono-sample/src/AppModule#AppModule",
  "skipMetadataEmit": true
}
```
You should note that we need to announce our main *AppModule* as `entryModule` to the Angular compiler.
We'll add this *AppModule* in the next step. At the end of this step your projects top-level structure should look like this:

```
${PROJECT_HOME}
  |- .git              // hidden on Linux
  |- config/
  |- node_modules/
  |- packages/
  |- scripts/
  |- src/
  |- .gitignore        // hidden on Linux
  |- LICENSE.txt
  |- package.json
  |- package-lock.json
  |- README.md
  |- tsconfig.json     // New
```

Commit our changes:

```
git add -A
git status  // Optional: see what's being committed
git commit -m "TypeScript and Angular AoT compiler config"
```

## Writing the App<a name="writing_the_app"></a>

### Packages

Our first package will be the *core* or *app* package. It will contain the root `NgModule` often called *AppModule*. The package's structure should be considered prototypal for any additional feature packages and follows NPM package standards. `cd` into `${PACKAGE_ROOT}`, if you're not there:

```
mkdir -p packages/@foo/ng-mono-sample-app/src
cd packages/@foo/ng-mono-sample-app
npm init
  // !! Don't accept every suggestion !!
  // name: "@foo/ng-mono-sample-app"
  // license: "SEE LICENSE IN LICENSE.txt"
touch README.md LICENSE.txt .gitignore .npmignore index.ts
```
At the end of this step you should have a project structure like this:

```
${PROJECT_HOME}
  |- ...
  |- packages/
  |   |- @foo/            // Optional: only required for scoped packages
  |     |- ng-mono-sample-app/
  |       |- src/         // required - the actual package source code
  |       |- .gitignore   // optional - may be required when moving to a separate repo
  |       |- .npmignore   // optional - may be required when publishing package
  |       |- index.ts     // required - public api surface (facade module)
  |       |- LICENSE.txt  // optional - required when publishing a package
  |       |- package.json // optional - required when publishing package
  |       |- README.md    // required - how to use the package api (see note below)
  |- ...
```

> Scoped packages are optional. Enterprises often use [scoped npm packages](https://docs.npmjs.com/getting-started/scoped-packages) to avoid collisions with public unscoped packages. If you don't declare scoped packages just skip the intermediary `@pkg-scope` folder level.

#### Package Best Practices

1. packages **must** be created inside the *packages* folder
1. packages **should** follow npm package best practices (e.g. contain a `README.md`)
1. packages **must** exhibit a *facade module* (`index.ts` in our case) which exports a package's *public* API contract
1. packages **must not** depend on another package's internal directory structure (no *deep imports*)

>- `import {MyClass} from` **`"@foo/sample-feature/src/path/to/MyClass";`** is **BAD** because it bypasses the facade module of package `@foo/sample-feature` with a deep import
>- `import {MyClass} from` **`"../../@foo/sample-feature/src/path/to/MyClass";`** is **BAD** because of a deep import and assumptions about the relative location of package `@foo/sample-feature`
>- `import {MyClass} from` **`"@foo/sample-feature";`** is **GOOD** because we only depend on the package name independent of its location

The fourth rule actually only suggests to import packages *within the source-tree of the monorepo* as if they were installed into *node_modules* like a normal npm package. This way it becomes easy to separate parts of an app's source code into its own distinct repository, once there's a need to share and maintain it as an independent package.

`cd` back into `${PROJECT_HOME}` if you're not yet there and commit your changes.

```
git add -A
git status  // Optional: see what's being committed
git commit -m "Adding an empty app package"
```

### Implementing the Core Application Package

Replace the `ng-mono-sample-app` package within our project with the one from the [sample project](https://github.com/about-code/ng-mono-sample). We'll explain the most interesting files below:

```
cd packages/@foo/ng-mono-sample-app
```

*main-aot.ts*

```ts
import { platformBrowser }    from "@angular/platform-browser";
import { AppModuleNgFactory } from "./src/AppModule.ngfactory";
platformBrowser().bootstrapModuleFactory(AppModuleNgFactory);
```

This is where our application package registers its *AppModule* with Angular and enters the Angular framework lifecycle.
The snippet is pretty standard Angular and you should find a similar one in the [angular.io](https://angular.io) docs.

> **Note:** `You might wonder where {NgModuleName}.ngfactory.ts` is in the source tree. `*.ngfactory*` files won't ever exist there but are generated dynamically during *Ahead-of-Time Compilation (AoT)* from the `{NgModuleName}.ts` counter-part in the source tree. You can safely ignore editor complaints about the missing file.

*src/AppModule.ts*

```ts
import {NgModule} from "@angular/core";
import {BrowserModule} from "@angular/platform-browser";
import {CommonModule} from "@angular/common";
import {RouterModule} from "@angular/router";
import {ReactiveFormsModule} from "@angular/forms";
import {FeatureModule} from "@foo/sample-feature"; // adding a feature module
import {RoutesModule} from "./RoutesModule";
import {AppComponent} from "./app/AppComponent";
import {HomeViewComponent} from "./home-view/HomeViewComponent";

@NgModule({
    imports: [
        BrowserModule
        , CommonModule
        , RouterModule
        , ReactiveFormsModule
        , RoutesModule
        , FeatureModule  // adding a feature module
    ],
    declarations: [
        AppComponent
        , HomeViewComponent
    ],
    providers: [],
    bootstrap: [ AppComponent ]
})
export class AppModule {}
```

Our copied `AppModule` depends on a feature package `@foo/sample-feature` which exports an `@NgModule` class `FeatureModule`. We don't have such a feature package yet, so put those two lines in comments or remove them for the moment. What you can already see from these two lines, though, is that it will be very easy to use new feature packages.

What you should note:

- `AppModule` must be referenced as `entryModule` under `angularCompilerOptions` in `tsconfig.json`.
- `BrowserModule, CommonModule, RouterModule` are Angular modules which browser applications typically need to import. See [angular.io](https://www.angular.io) for details.
- By Importing `ReactiveFormsModule` we've decided to write an application which uses the Reactive Forms approach. See [angular.io](https://www.angular.io) for other options and details.
- In a minimal example having an `AppModule` with an `AppComponent` and a corresponding component template would already be sufficient to display some contents under the default route `/`. But the sample helps you getting startet with *routing* . It declares an additional `HomeViewComponent` with a welcome message. `AppComponent` is a *parent component* with a `<router-outlet>`. Think of router outlets as kind of HTML-Frames. They tell Angular where to embed child components associated with a route. The associations are declared in `RoutesModule`. It configures the router to load a `HomeViewComponent` into the outlet upon routing to `/home`. Further it declares a *redirect* for the empty route `""` which causes the home view to be displayed initially when routing to `/`.

Commit the changes from within `${PROJECT_HOME}`:

```
git add -A
git status  // Optional: see what's being committed
git commit -m "Implementing the app package and bootstrap sequence"
```

### The Application Shell

The files for our application shell will be created in `${PROJECT_HOME}/src/` folder. Copy the `src` folder from the [sample project](https://github.com/about-code/ng-mono-sample).



> A shell is required when developing and bundling an Angular *application project*. In a *library project* developers *might* choose to have an application which acts as a test-bed for the library.

After this step you should have a file layout like the following:

```
${PROJECT_HOME}
    |- config/
    |- packages/
    |- scripts/
    |- src/
    |    |- app-aot.ts  // imports Angular-AoT main-aot.ts from app package
    |    |- app-jit.ts  // imports Angular-JiT main-jit.ts from app package
    |    |- bundle-polyfills.ts // imports polyfills (browser compatibility)
    |    |- bundle-theme.ts     // imports a theme package
    |    |- bundle-vendor.ts    // imports vendor libs
    |    |- index.html          // the true "main()" of a browser app
    |- .gitignore
    |- LICENSE.txt
    |- package.json
    |- README.md
    |- tsconfig.json
```

The TypeScript files are *entry bundles* for the webpack bundler. The bundles tell webpack where to start module resolution from. We declare entry bundles for

- our app (`app-aot.ts`, `app-jit.ts`),
- an app theme (`bundle-theme.ts`)
- external libraries (`bundle-vendor.ts`)
- polyfills (`bundle-polyfills.ts`)


The `app-jit.ts` entry bundle will be used by our webpack development build. JiT builds are faster and therefore better suited for incremental builds during development. Likewise the `app-aot.ts` is used by the production build. The files are very simple. They just import the Angular bootstrap modules `main-{aot|jit}.ts` from our `@foo/ng-mono-sample-app` package:

*app-aot.ts*

```ts
import "@foo/ng-mono-sample-app/main-aot.ts";
```

The actual bootstrapping of any Single Page App begins by requesting an `index.html` in a browser. You might miss the `<script>` tags loading the JavaScript application. They will be added during the build process. For now you should see that the `index.html` looks pretty similar to the examples on [angular.io](https://angular.io). There's nothing project-specific apart from the `<ng-mono-sample-app>` tag which is the component selector of our `AppComponent` in our app package `@foo/ng-mono-sample-app`.

> You could use a more generic `AppComponent` selector, e.g. `ng-app`. Then you need to change less when reusing the shell and project structure for another app project.

*index.html*

```html
<html>
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
    <meta http-equiv="x-ua-compatible" content="ie=edge">
    <title>My sample app</title>
  </head>

    <body>
        <base href="/">
        <ng-mono-sample-app>Loading...</ng-mono-sample-app>
    </body>
</html>
```

Commit the changes

```
git add -A
git status  // Optional: see what's being committed
git commit -m "Adding the application shell"
```

### Adding a Theme Package

If you are a selling software solutions to customers you probably have customers requiring a certain amount of customization. Many customers require a particular look and feel which integrates with their own corporate identity guidelines. External software solutions should be able to integrate into the customer's in-house software landscape or public product portfolio.

In many tutorials you'll see `NgComponents` referencing stylesheets. While this sometimes make sense I tend to say these styles should only describe minimal layout and positioning properties to make the component appear as a visual entity. But they should not prescribe colors or component margins etc. Otherwise they will be difficult to customize. Instead anything relevant to customization and *theming* an application should be grouped into a distinct application *theme-package*.

```
cd packages/@foo/
mkdir ng-mono-sample-app-theme-default
cd ng-mono-sample-app-theme-default
npm init
   // name: @foo/ng-mono-sample-app-theme-default
touch index.scss .gitignore LICENSE.txt README.md variables.scss
mkdir fonts icons images packages
```

After this step you should have a structure

```
${PROJECT_HOME}
  |- ...
  |- packages/
  |   |- @foo/
  |     |- ng-mono-sample-app/
  |     |- ng-mono-sample-app-theme-default/   // new
  |       |- .gitignore
  |       |- .npmignore
  |       |- index.scss
  |       |- LICENSE.txt
  |       |- package.json
  |       |- README.md
  |       |- variables.scss
  |- ...
```
You may want to have a look into `${PROJECT_HOME}/src/bundle-theme.ts`. You should see that we can import the `index.scss` like a normal JavaScript package. When building the application webpack's `sass-loader` will care for transpiling SASS to plain CSS. Other loaders will care for rewriting asset paths to match the eventual build output.
Switching to another theme later should be as easy as changing the import or replacing the entry bundle with another one.

> There's one issue with the `bundle-theme.ts` entry bundle, currently. Webpack (or our webpack config) produces an empty JavaScript file for that and injects a `<script>` tag for it into `index.html`. This causes an unnecessary http request. I've not yet found the cause. But if you want to prevent this as a workaround you can import `index.scss` just the same into `app-{aot|jit}.ts` files and remove the `bundle-theme` entry bundle from `config/common.webpack.js`.

```
git add -A
git status  // Optional: see what's being committed
git commit -m "Adding a theme package"
```

We won't go any further into theming or structuring the theme package. However if you add assets such as fonts or icons then we found it useful to add package sub-folders `fonts`, `icons`, `images` etc., and define a sass variable for each path for use in the theme's `.scss` files.

## Building and Bundling the App<a name="building_the_app"></a>

[Webpack](https://webpack.js.org) is the defacto standard for building and bundling Angular apps (and many others). Before we go into the nitty gritty of the build let's define some requirements first. In brackets we've listed the tools and npm `devDependencies` helping us implement them.

### Requirements

The build setup shall be able to...

1. clean previous builds `[clean-webpack-plugin]`
1. embed Angular HTML templates into Angular Components `[html-loader]*`
1. transpile TypeScript sources into JavaScript `[@ngtools/webpack, typescript]`
1. produce an Angular application which compiles templates Ahead-of-Time (production) or Just-in-Time (development) `[@ngtools/webpack]`
1. transpile [SASS](https://www.sass-lang.com)-CSS into plain CSS `[sass-loader][css-loader]`
1. keep track of paths while assets being moved/copied `[webpack], [file-loader]`
1. produce minified, hashed bundles and assets to support [caching and cache invalidation](https://webpack.js.org/guides/caching/) `[webpack]`
1. inject bundle and asset paths (JS/CSS) into `index.html` of our application shell `[html-webpack-plugin]`
1. serve and incrementally rebuild the app during development `[webpack-dev-server]`
1. support developer-private dev-server configurations which are not under version control `implemented on our own`

> \* Soon we might be able to drop `html-loader` in favour of an Angular v6 compiler option [`enableResourceInlining`](https://github.com/angular/angular/commit/40315be) in `tsconfig.json`.

### Adding Config Files

```
cd config
touch webpack.common.js webpack.dev.js webpack.prod.js webpack.server.js
```
After this step you should have a folder structure

```
${PROJECT_HOME}
    |- config/
    |    |- webpack.common.js  // Config used in every build scenario
    |    |- webpack.dev.js     // Config for development builds (JiT)
    |    |- webpack.prod.js    // Config for production builds (AoT)
    |    |- webpack.server.js  // Development server config
    |- packages/
    |- scripts/
    |- src/
    |- .gitignore
    |- LICENSE.txt
    |- package.json
    |- README.md
    |- tsconfig.json
```

### Development Server Config

The file `webpack.server.js` configures [`webpack-dev-server`](https://webpack.js.org/configuration/dev-server/) which will serve our single page app.

> Serving an app during development rather than just opening the shell's `index.html` with the local browser is necessary to make HTTP-Requests from within JavaScript programmes. HTTP-Requests aren't possible with files loaded from a `file:///`-URL.

*webpack.server.js*
```js
module.exports = {
    historyApiFallback: true
    // ,port: 8000
    // ,proxy: {
    //     // HTTP proxy middleware config:
    //     // https://www.npmjs.com/package/http-proxy-middleware
    //     "/api": {
    //         target: "http://localhost:8888/"
    //         ,headers: {
    //             "Authorization": "Basic ..."
    //         }
    //     }
    // }
}
```

Enabling `historyApiFallback` makes *webpack-dev-server* return the `index.html` for unknown URLs.

> **Don't break the web**:  Servers hosting a *Single Page App* must be configured to return the app's `index.html` instead of a `HTTP 404 NOT FOUND` when they receive requests to URLs which contain an SPA-internal route. Such configuration is **vital** to enable proper *hyperlinking* (*bookmarkability*)  and *page reloads* (`F5`) and thus for remaining compatible with *web principles*. Hyperlinking into an SPA is also sometimes referred to as *deep-linking*.


### Development and Production Config

*webpack.dev.js*
```js
var webpack = require('webpack');
var webpackMerge = require('webpack-merge');
var commonConfig = require('./webpack.common.js');
var ExtractTextPlugin = require('extract-text-webpack-plugin');

const ENV = process.env.NODE_ENV = process.env.ENV = 'development';
module.exports = webpackMerge(commonConfig, {
  mode: 'development',
  entry: {
    'app': './src/app-jit.ts'
  },
  plugins: [
    new webpack.DefinePlugin({
      'process.env': { 'ENV': JSON.stringify(ENV) }
    })
  ]
});
```
<!--
> Webpack 4 introduced *build modes*. With `mode: 'production'` for example, webpack will apply minification using its UglifyJSPlugin. Since you don't want to debug minified code during development it doesn't minify with `mode: 'development'`. Build modes use *sensible defaults* to reduce config boilerplate but you are always free to configure everything to your need up to drpping `mode` alltogether.
-->

You see, the config which is special to development builds is limited to

- setting the webpack build `mode`
- selecting the entry bundle which specifies the Angular build mode (AoT/JiT)
- setting a build process variable (optional)

`webpack.prod.js` looks exactly the same as `webpack.dev.js` except:

- any use of string `'development'` must be replaced with `'production'`
- `'./src/app-jit.ts'` must be replaced with `'./src/app-aot.ts'`

Anything else can be configured in `common.webpack.js`. We are using [`webpack-merge`](https://npmjs.com/package/webpack-merge) to merge common webpack settings and use-case specific settings together. This significantly increases maintainability and prevents us from repeating ourselves.

### Common Config

*webpack.common.js (1/3)*

```js
var path = require('path');
var webpack = require('webpack');
var webpackMerge = require('webpack-merge');
var AngularCompilerPlugin = require('@ngtools/webpack').AngularCompilerPlugin;
var ExtractTextPlugin = require('extract-text-webpack-plugin');
var CleanWebpackPlugin = require('clean-webpack-plugin');
var HtmlWebpackPlugin = require('html-webpack-plugin');

// =============== DEV-SERVER ===============
var serverConf;
try {
   // allow to overwrite a shared default config with a gitignored local config
  serverConf = require('./webpack.server.gitignore.js')
} catch (err) {
  serverConf = require('./webpack.server.js');
}
```

Remember requirement #10: we want to be able to have a separate `webpack.server.gitignore.js` which will be excluded from version control by means of the pattern `*.gitignore.*` in `.gitignore` file. This is to allow developers to tweak the server config locally (e.g. its port) without affecting their colleagues. Thus we attempt to load an unversioned server config first and fall back to the shared and versioned default if none is present.

*webpack.common.js (2/3)*

```js
// =============== SASS ===============
const outputPath = path.resolve(__dirname, '../', 'dist');
const extractSass = new ExtractTextPlugin({
    filename: "[name].[contenthash].css",
    disable: process.env.NODE_ENV === "development"
});
```

Here were are going to prepare a few lines required by the `sass-loader` (requirement #5). Append this to the previous section. You should note that apart from path adjustments we basically follow the documentation given for [`sass-loader`](https://www.npmjs.com/package/sass-loader), here.

*webpack.common.js (3/3)*

```js
// =============== WEBPACK ===============
module.exports = {
  context: path.resolve(__dirname, '../'),
  devServer: serverConf,
  entry: {
    'theme': './src/bundle-theme.ts',
    'polyfills': './src/bundle-polyfills.ts',
    'vendor': './src/bundle-vendor.ts'
  },
  resolve: {
    extensions: ['.ts', '.js'],
    modules: ['node_modules', './packages']
  },
  output: {
    path: outputPath,
    filename: '[name].[hash].js',
    chunkFilename: '[id].[hash].chunk.js',
    publicPath: '/'
  },
  module: {
    rules: [
      {
        test: /(?:\.ngfactory\.js|\.ngstyle\.js|\.ts)$/,
        loader: '@ngtools/webpack',
        //sourcemap: true
      }
      ,{
        test: /\.html$/,
        loader: 'html-loader'
      }
      ,{
        test: /\.(png|jpe?g|gif|svg|woff|woff2|ttf|eot|ico)$/,
        loader: 'file-loader?name=assets/[name].[hash].[ext]'
      }
      ,{
        test: /\.scss$/,
        use: extractSass.extract({
            use: [{
                loader: "css-loader"
            }, {
                loader: "sass-loader"
            }],
            // use style-loader in development
            fallback: "style-loader"
        })
      }
    ]
  },

  plugins: [
    extractSass
    ,new CleanWebpackPlugin([outputPath])
    ,new AngularCompilerPlugin({
      tsConfigPath: './tsconfig.json',
    })
    // Workaround for https://github.com/angular/angular/issues/11580
    // The (\\|\/) piece accounts for path separators in *nix and Windows
    ,new webpack.ContextReplacementPlugin(
      /angular(\\|\/)core(\\|\/)@angular/,
      path.resolve(__dirname, '.', 'src'),
      {}
    )
    ,new HtmlWebpackPlugin({
      template: './src/index.html'
    })
    ,new webpack.LoaderOptionsPlugin({
      htmlLoader: {
        minimize: false // workaround for ng2
      }
    })
  ]
};
```

That's quite a lot but it should make sense to you, if you compare it to our requirements. There are some workarounds in it. They do not satisfy a concrete requirement apart from dealing with some inconveniences.

> If you feel a bit depressed now asking yourself how you would ever be able to come up with such a config, just remember the JavaScriptster workflow we've followed on our own:
>
> - we have a problem / requirement
> - we read [webpack docs](https://webpack.js.org), search the [npm package repository](https://npmjs.com) or [the web](https://www.startpage.com)
> - we find a package (e.g. webpack loader or plug-in)
> - we follow the instructions given in the package's README.md
> - we apply trial and error
> - from time-to-time we stumble upon really nasty issues (see use of `ContextReplacementPlugin` or `LoaderOptionsPlugin`). Then we search the [GitHub Repos](https://github.com/webpack/webpack) or ask support questions on [Stackoverflow](https://www.stackoverflow.com).
>
> These tasks can be tedious but everybody can do them, so can you.

There is only a single instruction which is a bit more specific to our chosen project layout and requires some understanding of how webpack and TypeScript work:

```js
resolve: {
  modules: ['node_modules', './packages']
},
```

You might remember that we configured the TypeScript compiler's module resolution algorithm to look into `packages` using `baseUrl` and `paths` in `tsconfig.json`. Of course webpack (or bundlers in general) also need some module resolution algorithm to implement their powerful bundling magic. What we did here is the same as we did for TypeScript: we tell webpack to resolve absolute import paths like `import {Foo} from` **`"@foo/sample-feature"`** not just relative to `node_modules` but to `packages/` as well.

With those TS and webpack path settings we are able to import things *by package name* without having to link the packages into `node_modules` prior to building our app! It enables us to build an app from packages without requiring us to have a library build for each package unless absolutely needed. A library build is only needed if we plan to share and publish a package or we plan to move it from the monorepo into a separate repo. Until then with our project setup we are prepared for that but we are *not required* to think about this upfront if our main goal is to build an app. Writing a package-oriented app is limited to creating package folders and adhering to some package conventions.

> Extending the project with a build process for the packages might become a separate post. For the moment you can have a look at how it is solved in the [sample project](https://github.com/about-code/ng-mono-sample).

### Running the Build and Serving the App

Npm scripts provide convenient shortcuts to execute more complicated npm or shell commands. They need to be defined in the project manifest and can be started with `npm run <script>`. These will be the primary build scripts:

*${PROJECT_HOME}/package.json*
```json
"scripts": {
  "start": "webpack-dev-server --config ./config/webpack.dev.js --progress",
  "build": "webpack --config ./config/webpack.prod.js",
  "build-serve": "webpack-dev-server --config ./config/webpack.prod.js --watch --progress"
},
```

- Use `npm start` during everyday development. It starts a dev server and incrementally builds with our webpack development config
- Use `npm run build` locally or on your build server to write an optimized production build to `dist/`
- Use `npm run build-serve` to start a development server with the production build config

```
npm start
```

If this works open `http://localhost:8080/`. You should see a text *Welcome Home*
from the `HomeViewComponent` template. Also try the other npm scripts, now. Then
commit the results.

```
git add -A
git commit -m "Adding webpack build and bundling config"
```

## If you're in trouble<a name="troubleshooting"></a>

If you couldn't make the setup work, don't panic. Then go on with the [sample](https://github.com/about-code/ng-mono-sample) and tailor it manually.

## Customizing the Sample Project<a name="customizing_sample"></a>

To generate a tailored version of the sample for your own projects I've written [ng-mono](https://github.com/about-code/ng-mono), a scaffolding tool generating a project layout as described in this article. Its emphasis is on *scaffolding*. Don't consider it the next CLI.

> The tool currently has some *naive* implementation for assembling packages, NgModules or NgComponents using embedded code comments ("snippet comments"). You can drop any `// ::` or `/* ::` comments if you don't plan on using the tool after having generated your project.

If you want to tailor the sample project manually, then these may be the steps you want to get started with:

- copy the repo
- `npm install` dependencies
- rename the `-app` package
  - adjust `name` in its `package.json`
  - adjust the Angular `entryModule` path in your `tsconfig.json`
  - change `title` and (optionally) the `AppComponent` selector in `index.html`
  - change `app-{aot|jit}.ts` shell files to import from the new package
- rename the `-theme-default` package
  - adjust its `package.json`
  - adjust import path in `bundle-theme.ts`.
- `npm start`

<!--
## Advanced Workflows

### How to move a package from an incubator project into a separate repo

- fork/copy the the incubator repo
- delete any packages in `packages` apart from the package you want to keep
- Build the remaining package with `npm run build-pkg`
- publish the package to an internal or external npm registry with `npm run lerna publish`
  > if you want to try out publishing locally, first, set up a local npm registry e.g. with [`verdaccio`](https://npmjs.com/package/verdaccio).
- `npm install` the published package version into the incubator project again
  > **Note:** If you want to install external package sources `npm` supports installing from a git repo URL directly. This may be an option you have not yet a working library build and publish process.
- add it as a dependency to the `dependencies` section of dependent packages in the incubator project
- remember: **no** changes to the source code of the incubator project should be necessary
-->
