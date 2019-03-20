---
title: Scaling Frontend codebase (without sacrificing DX).
date: "2019-03-19"
spoiler: Getting the best of a mono repo architecture with Git submodules
draft: false
keywords: reactjs, next-js, git submodues, create react app, netlify, gitlab pipelines, docker, npm, codebase, frontend, dx, developer experience
---


## The backstory
Imagine a frontend codebase as a single repository consisting of 3 sub applications, 
1. the marketing pages application built with [NextJS](https://nextjs.org/), 
2. the UI library, which houses the design system used by all apps 
3. the product/main app built with [Creact React App](https://facebook.github.io/create-react-app/).
4. An internal shared library used accross applications. We would call this `shared-lib`

[Yarn workspace](https://yarnpkg.com/lang/en/docs/workspaces/) was used in wiring up these applications, so that a single `yarn install` pulls all the necessary dependencies required by all the different applications and libraries. Each application is a [NodeJS]() package and has its own internal script commands for deployment.

These scripts are then aggregated in the root `package.json`

**repo/packages/ui-lib**
```json
{
    "name": "ui-lib",
    ...,
    "scripts":{
        "build": "build-storybook -s public"
    },
    ...
}
```
**repo/package.json**
```json
{
    "private": true,
	"workspaces": [
		"packages/*"
	],
    "scripts": {
		"build:ui_lib": "yarn workspace ui_lib build",
		...
    }
}
```

## The development build process

The UI library, built with [Storybook](https://storybook.js.org/), is supposed to be shared with both the `marketing-pages` app as well as the `main-app` application. All visual elements/pages used by sub applications are housed and developed in this package.

Getting this library to be used as a dependency, in both the `marketing-pages` app and the `main-app` application, wasn't straight forward because, the `main-app` was built with [Create React App]() and the `marketing app` was built with [NextJS](), both of which by default have a pre-defined way of finding dependencies.

> [Create React App]() is a zero-config approach to building [ReactJS]() applications. By default, the only way you can get a dependency into a react application, is by installing it from [npm](). This was what `main-app` was built with.

Getting `main-app` to detect the UI library as a dependency could be achieved, but requires a lot of overhead. At development time, changes are often made to both UI Library as well as `main-app`, so near instant feedback was very essential.

Using Yarn workspaces, all application packages i.e (`ui-lib`, `marketing-pages`, `main-app`) were automatically symlinked in the root `node_modules`. This means they can be referenced easily across packages. 

The issue was getting the [Create React App]() setup for `main-app`, to compile the [React]() components provided by `ui-lib`, without having to eject. [React App Rewired](https://github.com/timarney/react-app-rewired) made this possible. It provides an extension to the internal [Webpack]() config used by [Create React App]() so that it can be extended.

> [React App Rewired](https://github.com/timarney/react-app-rewired) allows for simple extensions while still defaulting to [Create React App]() for all the heavy lifting. While there are other options, it is the least invasive approach in my opinion.

[Create React App]() needs to know where to look, to find all the the directories  that needs to be compiled. *(The default is the `src` directory of the app)*

Since the folder structure of the root repository looks somewhat like this

```
├── node_modules
├── .git
├── packages
|   ├── ui-lib
|   |   └── node_modules
|   |   └── src
|   |   └── package.json
|   └── main-app
|   |   └── node_modules
|   |   └── public
|   |   └── src
|   |   └── config-overrides.js
|   |   └── package.json
|   └── marketing-app
|   |   └── node_modules
|   |   └── pages
|   |   └── next.config.js
|   |   └── package.json
|   └── shared-app
|   |   └── src
|   |   └── package.json
├── .gitignore
└── package.json
```

**main app `config-overrides.js` implementation.**

```js
const path = require('path');
...
function compileLinkNodeModules(config, env) {
    // this tells create react app to compile the  following directories 
        config.module.rules[2].oneOf[1].include = [
        path.resolve(__dirname, "src"),
        path.resolve(__dirname, "../shared-lib"), // the shared library which might contain jsx components
        path.resolve(__dirname, "../ui-lib") // the ui library which definitely contains jsx components.
    ];

  return config;
}

module.exports = function(config, env){
    ...
    config = compileLinkNodeModules(config, env);
    ...
    return config
}
```

It becomes obvious where the paths to `ui-lib` and `shared-app` are and so are accounted appropriately in the `config-overrides.js` file.

With this, running `yarn start` in the `main-app` directory works without any issue.
Also, components from in the `ui-lib` codebase can be accessed in `main-app` as follows

```js
import React from 'react';
import AccountingPage from 'ui-lib/src/pages/AccountingPage';
...
```

As for the `marketing-app`, [NextJS]() also had similar issues. This required using some of [NextJS]() plugins in resolving the issue

1. [Next Transpile modules](https://www.npmjs.com/package/next-transpile-modules)
2. [Next Fonts](https://www.npmjs.com/package/next-fonts)  *(Unlike [Create React App]() which automatically transpiles web fonts, [NextJS]() doesn't, so an extra configuration was needed)*


**repo/packages/marketing-pages/next.config.json**
```js
const withTM = require("next-plugin-transpile-modules");
const withFonts = require("next-fonts");
const path = require("path");

module.exports = withFonts(
  withTM({
    transpileModules: [
      "ui-lib", //the package name registered by yarn workspace
      "shared-lib",
      path.resolve(__dirname, "../ui-lib") // the directory of the ui lib just in case
    ],
    ...
    webpack(config, options) {
        //Additional webpack config to override.
      return config;
    }
  })
);

```


## The Production Build Problem:
The above project setup makes it very easy to setup a development environment in a matter of minutes. The only command required, for setup, is `yarn install`.

But when thinking about deploying to production, The following problems occured.

1. Inconsitency in build during development and on production platforms like [Netlify](https://www.netlify.com/). 

2. The time to build all the apps took over 17 minutes. This made deploying small changes very painful and increased the time required to resolve bug fixes

*An initial solution to the first problem was to use [Gitlab Piplines]() and build the applications in a Docker container. The resulting build, grouped by folders, was then commited to a deploy repo which is then picked up by [Netlify](https://www.netlify.com/).*

## The solution.
Irrespective of what solution was proposed, one thing was certain, it shouldn't affect the development setup process. The only command requred to setup after cloning the base repository was `yarn install`.

The proposed solution was to get all sub applications as [Git submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules). This meant, each sub application was a [git]() repository and could be deployed in isolation. As problems occured in a particular application, it was immediately addressed in isolation and deployed. 

**But then both `marketing pages` app and `main-app` failed to build in isolation**

The problem is interesting. In the case of `marketing-pages`, Both the `shared-lib` and `ui-lib` packages weren't linked anymore in the `node_modules`. This was what [Yarn Workspaces]() solved by default.

Both `ui-lib` and `shared-lib` also needed to be git sub modules. Since [git repository can be installed as dependencies](https://docs.npmjs.com/files/package.json#git-urls-as-dependencies), the `package.json` for `marketing-app`  was updated.

**repo/packages/marketing-app**
```json
{
    "name": "marketing-app",
    ...,
    "dependencies":{
        "ui-lib": "<the repository link of the ui lib>#<the last git tag>",
        "shared-lib": "<the repository link of the shared lib>#<the last git tag>"
    }
}
```

By ensuring using specific git tag instead of a particular branch, deployment becomes an explicit action. To get the latest changes from the `ui-lib` codebase, the git tag had to be bumped to a new version. This ensures experimenetation without having to worry about regressions or breaking changes  in both `marketing-pages` and `main-app` as a result of new commits introduced in either the `ui-lib` or the `shared-lib` codebases.

These changes were made directly on the `master` branch of both the `marketing app` as well as `main_app`. The `develop` branch for both `marketing-pages` and `main-app`, is what is then referenced by the root **mono repo** codebase housing all the submodules.

An intemeditate branch `staging` for both `marketing-pages` and `main-app` is required, to get changes from the `develop` branch to the `master` branch, and also ensuring that the `dependencies` in the `master` branch never creep over to the `develop` branch and vice versa.

Also since, in the `master` branch that houses the `main-app`, Lookups to the actual location of jsx components needs to be updated in the  `config-overrides.js`

**repo/packages/main-app(master)**
```js
const path = require('path');
...
function compileLinkNodeModules(config, env) {
    // this tells create react app to compile the  following directories 
        config.module.rules[2].oneOf[1].include = [
        path.resolve(__dirname, "src"),
        //path.resolve(__dirname, "../shared-lib"), 
        path.resolve(__dirname, "node_modules/shared-lib")
        // path.resolve(__dirname, "../ui-lib") 
        path.resolve(__dirname, "node_modules/ui-lib")
    ];

  return config;
}
...
```
**NB: The name should match the package names in the `packge.json` file.**

The `package.json` for the `main-app` codebase would take the following look for both the `develop` branch as well as the `master` branch

**repo/packages/main-app/package.json(develop)**
```json
{
    "name": "main-app",
    ...,
    "dependencies":{
        "ui-lib": "*",
        "shared-lib": "*"
    }
}
```
The reason for this is simple. In the `develop` branch, work would most likely happen in the root repository that houses the sub_modules. Since this repository has [Yarn Workspace]() handling all the dependencies, whatsoever was resolved is what is needed, hence the need to expicitly use  `*` for both the `ui-lib` as well as the `shared-lib`

**repo/packages/main-app/package.json(master)**
```js
{
    "name": "main-app",
    ...,
    "dependencies":{
        "ui-lib": "<the repository link of the ui lib>#<the last git tag>",
        "shared-lib": "<the repository link of the shared lib>#<the last git tag>"
    }
}
```

