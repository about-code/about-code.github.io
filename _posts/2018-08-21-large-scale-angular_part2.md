---
layout: post
title:  "Large Scale Angular from Scratch Part 2"
date:   2018-08-21 11:15:00 +0200
categories: web-development javascript angular
image: 2018-04-28-large-scale-angular/article.png
tags: large-scale, angular
---

In the second part of *Large Scale Angular from Scratch* we show you how to set
up unit tests and end-to-end tests using *Karma*, *Webpack* and *Protractor*
based on the monorepo project structure introduced in part 1.

**Table of Contents**<a name="toc"></a>

[Part 1](./large-scale-angular.html)
- [Architectural Considerations](./large-scale-angular.html#architectural_considerations)
- [Setting up the Project Package](./large-scale-angular.html#set_up_project)
- [Installing Project Dependencies and Defining the Toolchain](./large-scale-angular.html#install_deps)
- [Setting up a Local Git Repository](./large-scale-angular.html#set_up_git)
- [Setting up TypeScript](./large-scale-angular.html#set_up_typescript)
- [Writing the App](./large-scale-angular.html#writing_the_app)
  - [Packages](./large-scale-angular.html#packages)
  - [Implementing the Core Application Package](./large-scale-angular.html#implementing_core_package)
  - [Implementing the Application Shell](./large-scale-angular.html#application_shell)
  - [Adding a Theme Package](./large-scale-angular.html#adding_theme_package)
- [Building and Bundling the App](./large-scale-angular.html#building_the_app)
  - [Requirements](./large-scale-angular.html#requirements)
  - [Adding Config Files](./large-scale-angular.html#adding_config_files)
  - [Development and Production Config](./large-scale-angular.html#dev_and_prod_config)
  - [Development Server Config](./large-scale-angular.html#dev_server_config)
  - [Common Config](./large-scale-angular.html#common_config)
  - [Running the Build and Serving the App](./large-scale-angular.html#running_the_app)
- [If you're in trouble](./large-scale-angular.html#troubleshooting)
- [Customizing the Sample Project](./large-scale-angular.html#customizing_sample)

Part 2 <a name="toc_part2"></a>
- [Setting up Unit Tests with Karma and Jasmine](#karma_and_jasmine)
  - [Configuring Karma](#karma_config)
  - [Writing a Unit Test](#writing_unit_tests)
  - [Running the Tests](#running_the_tests)
  - [Debugging Unit Tests](#debugging_unit_tests)
- [Setting up E2E-Tests with Protractor](#protractor)

Part 3 (yet to be written)
- Publishing Packages
- Advanced Workflows

## Setting up Unit Tests with Karma and Jasmine<a name="karma_and_jasmine"></a>

If you followed [part 1](./large-scale-angular.html) of our tutorial and downloaded our [sample project](https://github.com/about-code/ng-mono-sample/archive/v1.2.0.zip) you should already be able to build an Angular application. Your top-level folder should look like this:

```
${PROJECT_HOME}
    |- .git           (may be hidden on Unix systems)
    |- config/
    |- node_modules/  (run `npm install` if missing)
    |- packages/
    |- scripts/
    |- src/
    |- .gitignore
    |- LICENSE.txt
    |- package.json
    |- README.md
    |- tsconfig.json
```
In the second part of this series we'll be focusing on testing. In your `package.json` you should already find these development dependencies (the versions may have changed since writing the article):

```
"devDependencies": {
    [...]
    "@types/jasmine": "^2.8.4",
    "@types/jasminewd2": "^2.0.2",
    "jasmine-core": "^3.1.0",
    "jasmine-spec-reporter": "^4.2.1",
    [...]
    "karma": "~2.0.0",
    "karma-chrome-launcher": "~2.2.0",
    "karma-cli": "~1.0.1",
    "karma-coverage-istanbul-reporter": "^2.0.1",
    "karma-jasmine": "~1.1.2",
    "karma-jasmine-html-reporter": "^1.2.0",
    "karma-webpack": "~3.0.0",
    [...]
    "angular2-template-loader": "^0.6.2",
    "istanbul-instrumenter-loader": "^3.0.1",
    "raw-loader": "^0.5.1",
    "ts-loader": "^4.4.2"
}
```
> We use version range specifiers like `^` or `~`. In the past version ranges did often break things when `npm install` pulled updated packages with bugs or from package authors who didn't consider semantic versioning. Unintentional updates should happen less often in the presence of a `package-lock.json` but if you want to keep the risk at a minimum you could even choose to omit version range specifiers alltogether and manually test for newer versions of packages with `npm outdated`.

**[Jasmine](https://jasmine.github.io)** provides us with a BDD (Behavior Driven Development) style assertion API. While Jasmine can be run as a standalone test runner on node we need a bit more than that. Our sample app is a browser app. So unit tests should run in a browser as well. Due to this we run tests with **[Karma](https://karma-runner.github.io/2.0/index.html)**. **[karma-jasmine](https://npmjs.com/package/karma-jasmine)** integrates Jasmine and Karma. Next we use **[karma-webpack](https://npmjs.com/package/karma-webpack)** to run *webpack* as a *preprocessor* which transpiles our TypeScript to JavaScript. We also want to collect code coverage metrics, so we need to *instrument* the sources. We use **[istanbul-instrumenter-loader](https://npmjs.com/package/istanbul-instrumenter-loader)** here, which is a webpack loader. So instrumentation happens within the preprocessing phase in the Karma lifecycle. Since it must be run *after* transpilation it's a postprocessing step within the webpack lifecycle, though. Eventually **[karma-chrome-launcher](https://npmjs.com/package/karma-chrome-launcher)** cares for running tests within Chrome (Chrome or Chromium Browser must be installed).

> **Note:** There are quite some people who experience issues with `karma-chrome-launcher` and [tests not starting](https://github.com/karma-runner/karma-chrome-launcher/issues/154). We could not reproduce the issue on our system. If you experience similar issues you might want to try other launchers, e.g. [karma-firefox-launcher](https://npmjs.com/package/karma-firefox-launcher) or search the web for solutions.

### Configuring Karma <a name="karma_config"></a>

Now that we know which roles the various dependencies play we can have a look at how to configure Karma. Copy `karma.conf.js` and `karma.shim.js` from the sample project into your `config` folder. After this step you should have a project structure like this one:

```
${PROJECT_HOME}
    |- .git
    |- config/
    |    |- karma.conf.js       // new
    |    |- karma.shim.js       // new
    |    |- webpack.common.js
    |    |- webpack.dev.js
    |    |- webpack.prod.js
    |    |- webpack.server.js
    |- node_modules/
    |- packages/
    |- scripts/
    |- src/
    |- .gitignore
    |- LICENSE.txt
    |- package.json
    |- README.md
    |- tsconfig.json
```

```
git add "*karma*"
git status  // Optional: see what's being committed
git commit -m "Configuring Karma Test Runner"
```

*karma.conf.js*
```js
const path = require('path');
const process = require('process');
const webpackMerge = require('webpack-merge');
const webpackTestConf = require('./webpack.test.js');

// ==== Command Line Arguments ====
const DEBUG = process.argv.filter(arg => arg === "debug").length > 0;

module.exports = function(config) {
  config.set({
    basePath: path.resolve(__dirname)
    ,files: ['./karma.shim.js'] // one webpack entry point requires all tests
    ,preprocessors: {
        './karma.shim.js': ['webpack']
    }
    ,plugins: [
        'karma-jasmine',
        ,'karma-webpack'
        ,'karma-chrome-launcher'
        ,'karma-coverage-istanbul-reporter'
    ]
    ,frameworks: ['jasmine']
    // see karma-chrome-launcher's README.md for how to run with flags
    ,browsers:   ['ChromeHeadless']
    // When debugging tests...
    // ...keep watching files and recompile on changes
    // ...keep running server (allow rerun tests with F5 and browser dev tools)
    // ...keep code readable and don't instrument it
    ,autoWatch: DEBUG == true
    ,singleRun: DEBUG == false
    ,reporters: DEBUG == true ? [] : ['coverage-istanbul']
    ,webpack:   DEBUG == true ? webpackTestConf : webpackMerge(webpackTestConf, {
        module: {
            rules: [{
                test: /\.tsx?$/
                ,loader: 'istanbul-instrumenter-loader'
                ,enforce: 'post'
                ,options: { esModules: true }
                ,exclude: [/\.(spec|e2e|mock)\.tsx?/, /node_modules/]
            }]
        }
    })
    ,coverageIstanbulReporter: {
        dir: path.resolve(__dirname, '../reports/coverage')
        ,reports: ['html', 'text-summary', 'lcovonly']
        ,fixWebpackSourcePaths: true
    }
    ,webpackMiddleware: { stats: 'errors-only' }
    ,logLevel: config.LOG_INFO
    ,colors: true
    // ,autoWatchBatchDelay: 300
    // ,captureTimeout: 6000
  });
};
```
If you're already familiar with the official [Karma Docs](https://karma-runner.github.io) you may recognize the typical parts of our Karma configration. Below we describe a few details specific to our setup.

#### `basePath`

We set **`basePath`** to the directory where our Karma config file is in. This way we can be sure paths within the config file work relative to the config file which makes it easier to understand.

> Many examples on the web put their karma and webpack configs in `${PROJECT_HOME}`. This enables some editors and IDEs to detect the tool usage and provide a better user experience for running these. We rely on `npm run` commands instead, which many editors can detect as well. This allows us to keep config files in a separate directory and maintain a (subjectively cleaner) repository structure. There can be exceptions to the rule like with `tsconfig.json`, for example.

#### `webpack`

We import our webpack test config from `webpack.test.js` and use it with the **`webpack`** option evaluated by `karma-webpack`.

> You may wonder why we configure the *istanbul-instrumenter-loader* in `karma.conf.js` rather than in `webpack.test.js`. This is merely to keep the instrumentation configuration together in one place. Otherwise we had parts of it in `karma.conf.js` and parts of it in `webpack.test.js`.

#### `files`

For the **`files`** option we follow a [sample given by `karma-webpack`](https://www.npmjs.com/package/karma-webpack#alternative-usage) and specify the actual spec file-pattern in a single *entry point* file named `karma.shim.js` (see below). As the `karma-webpack` authors point out this way we can instruct webpack to generate only a *single* `bundle` with all our test specs rather than a bundle for each individual spec. By looking into `karma.shim.js` you should also find that we are able to run all the tests of all the packages in the repo.

*karma.shim.js*
```js
// See https://github.com/preboot/angular-webpack/blob/51c72e481eecdaa461b8c3a1f021d2f2064e4b80/karma-shim.js#L1
Error.stackTraceLimit = Infinity;

require('core-js/client/shim');
require('reflect-metadata'); // see note below
require('zone.js/dist/zone');
require('zone.js/dist/long-stack-trace-zone');
require('zone.js/dist/proxy');
require('zone.js/dist/sync-test');
require('zone.js/dist/jasmine-patch');
require('zone.js/dist/async-test');
require('zone.js/dist/fake-async-test');

// https://npmjs.com/package/karma-webpack
// https://angular-2-training-book.rangle.io/handout/testing/intro-to-tdd/setup/karma-config.html
// https://github.com/webpack-contrib/karma-webpack
var testContext = require.context('../packages', true, /spec\.ts/);
testContext.keys().forEach(testContext);

// see https://github.com/AngularClass/angular2-webpack-starter/issues/124
var testing = require('@angular/core/testing');
var browser = require('@angular/platform-browser-dynamic/testing');

testing.TestBed.initTestEnvironment(
    browser.BrowserDynamicTestingModule,
    browser.platformBrowserDynamicTesting()
);
```

> **Note:** In `ng-mono-sample` we still import the `reflect-metadata` polyfill.
> Angular 5.x doesn't require the `reflect-metadata` polyfill anymore so you may
> choose to remove it.

> **Note:** Once again, neither our webpack configs nor our karma configs make any assumptions about particular packages inside the `packages` folder which makes it very easy to reuse the exact same configuration in a new application project with other packages. This is what we referred to in the first part when we said the monorepo setup allows for a very generic configuration.

### Writing a Unit Test <a name="writing_unit_tests"></a>

The sample project only includes a very simple demo to show you that the specs are picked up correctly when you run `npm run test`. Have a look at [Angular's Testing Guide](https://angular.io/guide/testing) and the [Jasmine API](https://jasmine.github.io) for further details.

*AppComponent.spec.ts*
```ts
/**
 * .spec.-Files will be found anywhere inside a package. If you want to have unit
 * tests in a separate folder feel free to move the file elsewhere.
 */
describe('AppComponent', () => {

    it('should succeed', () => {
        expect(true).toBe(true, 'Sample Unit Test');
    });
});
```
> **Where to put test specs?**
>
> You might want to decide for yourself whether you like to put your test specs as a direct sibling file of the *module under test* (MUT) or whether you want to put them in a distinct `test` folder of a package. We have seen a change in what the Angular authors favour. In the past they argued for making tests direct siblings of the MUT to immediately see when tests for a module are missing. With some bias we think this approach can have a positive effect on a team's test dicipline.
>
> Yet, since their initial recommendations, the Angular team itself has been restructuring their own package sources and moved all test specs in a package's `test` folder. In a large codebase this is likely to scale better because it is less distracting when browsing the sources while working on app code. You might want to start with the first approach and refactor later when a team shows sufficient discipline. You don't need to change any configuration for that.

### Running the Unit Tests <a name="running_unit_tests"></a>

```
npm run test
```

### Debugging Unit Tests <a name="debugging_unit_tests"></a>

You may see in `karma.conf.js` that we adjust some karma options depending on whether a `debug` command line argument has been provided. Run tests with `npm run test debug` when developing tests. It starts Karma and webpack in *watch-mode* and turns off code instrumentation to keep code readable. In debug mode you can debug tests by opening `http://localhost:9876/debug.html` in a browser and setting breakpoints in the browser's developer tools.

## Setting up E2E-Tests with Protractor <a name="protractor"></a>

It took quite some tools and configuration to be able to run UnitTests with Karma. End-to-End-(E2E)-Tests are external to the application thus don't have to run inside a browser. This simplifies the toolchain. We'll be using [`protractor`](https://protractortest.org) with [`ts-node`](https://npmjs.com/package/ts-node) to run E2E-Tests written in TypeScript near natively on node.

This is how our `protractor.conf.js` looks like:

*protractor.conf.js*

```js
let path = require('path');

// An example configuration file
exports.config = {
  // The address of a running selenium server.
  //seleniumAddress: 'http://localhost:4444/wd/hub',
  seleniumArgs: []
  ,jvmArgs: []
  // Capabilities to be passed to the webdriver instance.
  ,capabilities: {
    browserName: 'chrome'
  }
  // Spec patterns are relative to the configuration file location passed
  // to protractor (in this example conf.js).
  // They may include glob patterns.
  ,specs: [
      '../src/**/*-e2e.{ts,js}'
      ,'../packages/**/*-e2e.{ts,js}'
  ]
  // Options to be passed to Jasmine-node.
  ,jasmineNodeOpts: {
    showColors: true // Use colors in the command line report.
    ,defaultTimeoutInterval: 15000
  }
  ,beforeLaunch: function() {
    require('ts-node').register({
      project: path.resolve(__dirname, '../tsconfig.json')
      ,compiler: 'typescript'
      ,transpileOnly: true
      ,compilerOptions: {
        module: "commonjs"
      }
    });
  }
};
```

Run the tests with `npm run test-e2e`.

## Final words

In this part we have shown a testing toolchain and how to configure it.
The article basically summarizes what's been a typical toolchain before there
was an Angular-CLI. The CLI still uses a pretty similar toolchain internally
yet tries to simplify overall configuration at the cost of some flexibility
in terms of configuring the tools. In the third part we are going to focus on
advanced workflows in a monorepo project.
