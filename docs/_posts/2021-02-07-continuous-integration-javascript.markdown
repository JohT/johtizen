---
layout: post
title:  "Continuous Integration for JavaScript with npm"
date:   2021-02-21 08:00:00 +0100
categories: build
tags: continuous integration javascript github npm parcel bundler jasmine eslint jsdoc
author: JohT
discussions-id: 36
---
Whereas there are numerous guidelines on how to setup specific tools like npm, 
combining all components to a build assembly line can be a tedious task. This article is intended to be a starting point, that gets you up and running with selected tools and references for vanilla JavaScript development. You can take it from there, exchange parts, add some more and dig into their configurations.

**Addition 2021-09:** [Setting up an angular project][angular setup] shows that there are frameworks,
that provide an easy way to set everything up. For them, only a few steps (4, 9 and 10) might be helpful.

## Prerequisites
- Existing GIT Repository
- Bash shell

## 10 Steps to continuous integration
These 10 steps cover right enough to get started. 
They don't cover everything and are for sure not the right choice for every project. 
Hopefully, the following steps help to save some time. 
This [living example][living example] was the original trigger and can be taken as reference too. 

### 1. Package with npm
[npm][about npm] manages dependencies and provides a registry to publish packaged code. 
After [installing npm][install npm], use `cd` to change into the directory of your repository 
and use the following command to create the `package.json` file:

```
npm init
```
Initializing npm within an already existing GIT repository [automatically fills in][npm init best practice] all repository related package fields.

### 2. Bundle with Parcel
[Parcel][about parcel] is an easy to use bundler, that copies multiple files into one, minifies them, transforms them,...
As described [here][parcel getting started], parcel is added using the following command:

```
npm install parcel-bundler --save-dev
```
This adds parcel-bundler as development dependency inside your `package.json` file.
If it hadn't already been there, the directory `nodes_modules` will show up, which should be added to the `.gitignore` file:

```
# Dependency directories
node_modules/
# Optional npm cache directory
.npm
```

To run parcel for development and production, these two script commands need to be added in the `package.json` file: 

```json
{
  "scripts": {
    "dev": "parcel --out-dir devdist ./src/js/*.js",
    "build": "parcel build ./src/js/*.js",
  }
}
```
You can execute them by their name:

```
npm run dev
npm run build
```

Parcel puts all build results into the `dist` folder. Building for development leads to different files,
that shouldn't get published or shouldn't even appear inside the repository. With `--out-dir devdist`
all development build files get written into their own folder (here `devdist`) that can be ignored using `.gitignore`.

To assure that the `dist` folder gets cleaned up right before the build,  
add the following script [as described here][prebuild]. `prebuild` will automatically run when `build` is started.

```json
{
  "scripts": {
    "prebuild": "rm -rf dist",
  }
}
```

If you have one entry point, like `index.html` or `index.js`, exchange `./src/js/*.js` with it.
`./src/js/*.js` can be a good match for libraries, that publish all their javascript sources in different files
or different variations (e.g. with/without IE compatibility).

If you accounter "command not found" problems while building in the pipeline, 
check if the first command in the chain is `npm ci`.  If it is still an issue, try to prefix the commands with `$(npm bin)/`, e.g. `$(npm bin)/parcel...`.

### 3. Unit Tests with Jasmine

[Jasmine][Jasmine about] is a unit test framework for JavaScript 
that strongly encourages [Behavior Driven Development (BDD)][Behavior Driven Development].
These two commands [add Jasmine][Jasmine getting started] as a development dependency
and initialize it by creating the file `spec/support/jasmine.json`:

```
npm install jasmine --save-dev
npx jasmine init
```

The script command inside `package.json` might look like this:

```json
{
  "scripts": {
    "test": "jasmine --config=./test/js/jasmine.json",
  }
}
```

If you prefer having all tests and their configuration inside the folder `test/js/`, 
you can move `jasmine.json` there and refer to it by specifying `--config=./test/js/jasmine.json`
in the command, as in the example above.

You can run the tests with:

```
npm test
```

The following example shows a test configuration in `jasmine.json` 
that reads all files ending with `Test` instead of `Spec` inside the folder `test/js` 
instead of `spec`. This sacrifices a bit of the [Behavior Driven Development (BDD)][Behavior Driven Development]
philosophy in favour of a more commonly known and used structure:

```json
{
  "spec_dir": "test/js",
  "spec_files": [
    "**/*Test.js"
  ],
  "helpers": [
    "polyfills/**/*.js",
    "**/*Data.js",
  ],
  "stopSpecOnExpectationFailure": false,
  "random": true
}
```

### 4. Unit Test coverage measurement with nyc
[nyc][nyc test coverage] measures test code coverage and [extends the functionality of the formerly known Istanbul][nyc history]. Like all other development dependencies it is installed by the following command: 

```
npm install nyc --save-dev
```

The script command inside `package.json` looks like this:

```json
{
  "scripts": {
    "coverage": "nyc npm run test",
  }
}
```

Use the following command to run the tests including code coverage measurement:

```
npm run coverage
```

To assure that nyc fails when code coverage doesn't meet the expectations, 
put a `.nycrc` configuration file into your project root. 
Here is an example including, among others, file name settings:

```json
{
    "all": true,
    "include": [
        "src/**/*.js"
    ],
    "exclude": [
        "src/**/*-ie.js"
    ],
    "reporter": [
        "html",
        "text",
        "json-summary"
    ],
    "check-coverage": true,
    "branches": 90,
    "lines": 80,
    "functions": 80,
    "statements": 80
}
```

### 5. Include Code Coverage Badge in README.md
[istanbul-badges-readme][nyc code coverage badge] provides an easy way to dynamically add the test code coverage
inside `README.md`.

Install it with:

```
npm install istanbul-badges-readme --save-dev
```

Add it as script command:

```json
{
  "scripts": {
    "coverage-badge": "istanbul-badges-readme",
  }
}
```

Assure that nyc outputs the report additionally as json by adding `"json-summary"` as reporter inside 
the configuration file `.nycrc`:

```json
"reporter": ["json-summary"]
```

Add the following line to your `README.md` to show the current branch coverage:

```markdown
![Branches](https://img.shields.io/badge/Coverage-91.45%25-brightgreen.svg)
```

More examples can be found [here][nyc code coverage badge].
Finally run the following command after measuring the test coverage to update your `README.md`:

```
npm run coverage-badge
```

### 6. Static code analysis with ESLint
[ESLint][eslint about] is a static code analyzer (aka "linter") that detects typos and bugs inside the code.
Use the following command to [install it][eslint getting started] and initialize its configuration file `.eslintrc.json`:

```
npm install eslint --save-dev
npx eslint --init
```

Add the following script command inside `package.json`:

```json
{
  "scripts": {
    "lint": "eslint \"./src/js/**\""
  }
}
```
Adapt the command if your JavaScript files are located somewhere else.
Use the following command to run static code analysis:

```
npm run lint
```

### 7. Documentation generation with JSDoc
[JSDoc][jsdoc] is a code documentation generator based on code comments.
Use the following command to [install it][install jsdoc]:

```
npm install jsdoc --save-dev
```

Add the following script command inside `package.json`:

```json
{
  "scripts": {
    "doc": "jsdoc -d doc --configure ./docs/jsdoc.json --readme ./README.md ./src/js/*.js"
  }
}
```

#### Command line parameters explained
- `-d doc` uses the directory `doc` for the generated files
- `--configure ./docs/jsdoc.json` points to the configuration file
- `--readme ./README.md` uses the `README.md` as landing page
- `./src/js/*.js` looks at all files ending with `.js` inside the directory `src/js/`
   
To use markdown inside comments, the configuration file `jsdoc.json` will look like this:

```json
{
    "plugins": [
        "plugins/markdown"
    ]
}
```

Use the following command to run documentation generation:

```
npm run doc
```

### 8. Run the whole chain within a single command
Running all previously mentioned commands in the right order for every change would be a tedious task.
This script command shows how to put them into a chain and run them in sequence:

```json
{
  "scripts": {
    "package": "npm run lint && npm run coverage && npm run coverage-badge && npm run doc && npm run build"
  }
}
```

The above example focuses on the raw sources and uses the build as finishing step. If lint fails, the chain fails early.
`coverage` generates results for `coverage-badge` which changes the `README.md` that is included in `doc` generation.

Using `&` instead of `&&` can be used to run tasks in parallel. Parentheses could be used to organize task groups.
The chain depends on which tools are used and if the build needs to run first. Don't hesitate to rearrange it as you like.

The whole chain can be started using the following command:

```
npm run package
```

### 9. Continuous Integration with GitHub Actions
[GitHub Actions][about github actions] are triggered by events like `git push`
and run predefined jobs like continuous integration pipelines. 
Create the directory `.github/workflows/` and a `.yaml` file inside it,
for example `.github/workflows/action.yaml`:

```yaml
name: Node CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: actions/setup-node@v2
      with:
        node-version: '12'
    - name: Install nodes packages
      run: npm ci
    - name: Run linter
      run: npm run lint
    - name: Run tests
      run: npm test
    - name: Measure test coverage
      run: npm run coverage
    - name: Generate documentation
      run: npm run doc
    - name: Build
      run: npm run build
```

This action will run the previously introduced chain of commands in separate, traceable named steps.
`npm ci` needs to be the first command. It loads all dependencies that are described inside `package.json`.
Without it you may encounter "command not found" error messages.

`coverage-badge` is missing by intention, since it updates `README.md`. This is not practical, since
the result would need another commit and push.

It is much easier to run `npm run package` before a git commit and push. You can even skip `npm run doc`
and `npm run build` in the GitHub Actions job, because their results should already be checked in. 
On the other hand it can be helpful to see (e.g. on pull requests) if all steps succeeded.

### 10. Publishing to npm
Even if publishing new versions could be automated, it is often preferred to do it manually by intention,
especially if it doesn't happen that often and could imply to communicate breaking changes.

["How to publish packages to npm"][publish to npm] covers this topic very well.

<br>

----

## Summary

Here is a list of commands, configurations and files in a nutshell.
This [living example][living example] can also be taken as reference.

### Setup Commands
- `npm init` initializes node package manager and creates `package.json`
- `npm install parcel-bundler --save-dev` installs parcel-bundler
- `npm install jasmine --save-dev` installs jasmine unit test framework 
- `npx jasmine init` initializes jasmine and creates `spec/support/jasmine.json`
- `npm install nyc --save-dev` installs nyc test code coverage measurement
- `npm install istanbul-badges-readme --save-dev` installs test code coverage badge
- `npm install jsdoc --save-dev` installs JavaScript Documentation (JSDoc)
- `npm install eslint --save-dev` installs ESLint static code analyzer
- `npx eslint --init` initializes ESLint and creates `.eslintrc.json` file
- `npm audit fix` fixes vulnerabilities


### Script Commands
- `npm install` installs all dependencies and creates the folder `/node_modules`, that is needed for all following commands
- `npm ci` is the same as `npm install` but [preferable][npm ci vs install] in pipeline build 
- `npm run package` runs all steps incl. test, coverage, doc generation and build
- `npm run coverage` runs all unit tests **with** coverage report
- `npm test` runs all unit tests **without** coverage report
- `npm run coverage-badge` updates code coverage badge inside `README.md`
- `npm run doc` generates JSDoc documentation in folder `/doc`
- `npm run build` builds the application for production including minification,...
- `npm run dev` builds the application for development without minification and starts the live server


### Script Configuration in package.json

```json
{
  "scripts": {
    "prebuild": "rm -rf dist",
    "lint": "eslint \"./src/js/**\"",
    "test": "jasmine --config=./test/js/jasmine.json",
    "coverage": "nyc npm run test",
    "coverage-badge": "istanbul-badges-readme",
    "doc": "jsdoc -d doc --configure ./docs/jsdoc.json --readme ./README.md ./src/js/*.js",
    "dev": "parcel --out-dir devdist ./src/js/*.js",
    "build": "parcel build ./src/js/*.js",
    "package": "npm run lint && npm run coverage && npm run coverage-badge && npm run doc && npm run build"
  }
}

```

### Files and Directories
- `./package.json` main configuration for npm and build scripts
- `./package-lock.json` npm managed file, shouldn't be ignored, used by `npm ci`
- `./.gitignore` GIT configuration for file(pattern)s to ignore
- `./nycrc` nyc test code coverage configuration and limits
- `./eslintrc.json` ESLint static code analysis configuration
- `/.cache` belongs to parcel, can be ignored and should not be included in the repository
- `./github/workflows/action.yaml` GitHub Actions workflow definition for the continuous integration pipeline
- `/coverage` test code coverage results and reports
- `/devdist` parcel development build output if configured like described above
- `/dist` default parcel production build output
- `/docs` JSDoc documentation generation output if configured like described above
- `/node-modules` directory for all npm dependencies, created by `npm install` or `npm ci`


<br>

----

## Updates

- 2021-03-13: Prefer [npm ci over npm install][npm ci vs install] in pipeline


## References

- [Living example of a JavaScript repository with Continuous Integration][living example]
- [Setting up an angular project][angular setup]
- [About npm][about npm]
- [How to install npm][install npm]
- [The best time to npm init][npm init best practice]
- [npm ci vs. npm install][npm ci vs install]
- [How to get the path of the npm binaries ][npm bin]
- [How to publish packages to npm][publish to npm]
- [About Parcel][about parcel]
- [YouTube: The Parcel Bundler - A SUPER Easy JavaScript Bundler for your Projects][YouTube Parcel Bundler]
- [Getting started with Parcel][parcel getting started]
- [Clean dist directory on repeated builds][prebuild]
- [About Jasmine unit testing][Jasmine about]
- [Getting started with Jasmine unit testing][Jasmine getting started]
- [Behavior Driven Development (BDD)][Behavior Driven Development]
- [nyc/istanbul for test coverage measurement][nyc test coverage]
- [Why nyc is named that way][nyc history]
- [Add nyc/istanbul code coverage badge to README.md][nyc code coverage badge]
- [ESLint static code analyzer][eslint about]
- [Getting started with ESLint][eslint getting started]
- [JSDoc documentation generator reference][jsdoc]
- [Getting started with JSDoc][install jsdoc]
- [GitHub Actions][about github actions]


[living example]: https://github.com/JohT/data-restructor-js

[about npm]: https://docs.npmjs.com/about-npm
[install npm]: https://www.npmjs.com/get-npm
[npm init best practice]: https://zellwk.com/blog/best-time-to-npm-init/
[npm bin]: https://docs.npmjs.com/cli/v6/commands/npm-bin
[npm scripts]: https://krishankantsinghal.medium.com/scripting-inside-package-json-4b06bea74c0e
[npm ci vs install]: https://betterprogramming.pub/npm-ci-vs-npm-install-which-should-you-use-in-your-node-js-projects-51e07cb71e26
[publish to npm]: https://zellwk.com/blog/publish-to-npm

[about parcel]: https://parceljs.org
[parcel getting started]: https://parceljs.org/getting_started.html
[YouTube Parcel Bundler]: https://youtu.be/OK6akGZCC88
[prebuild]: https://github.com/parcel-bundler/parcel/issues/1234

[Jasmine about]: https://jasmine.github.io/index.html
[Jasmine getting started]: https://jasmine.github.io/pages/getting_started.html
[Behavior Driven Development]: https://dannorth.net/introducing-bdd/

[nyc test coverage]: https://github.com/istanbuljs/nyc#nyc
[nyc history]: https://github.com/istanbuljs/nyc/issues/208#issuecomment-450505144
[nyc code coverage badge]: https://github.com/olavoparno/istanbul-badges-readme#istanbul-badges-readme

[eslint about]: https://eslint.org
[eslint getting started]: https://eslint.org/docs/user-guide/getting-started

[jsdoc]: https://jsdoc.app
[install jsdoc]: https://github.com/jsdoc/jsdoc#installation-and-usage

[about github actions]: https://docs.github.com/en/actions

[angular setup]: https://angular.io/guide/setup-local
