---
title: Git submodules to the rescue (A deployment case study)
date: "2019-03-19"
spoiler: Getting the best of a mono repo architecture with Git submodules
draft: false
keywords: reactjs, next-js, git submodues, create react app, netlify, gitlab pipelines, docker, npm
---

# 

## The backstory
Imagine a frontend codebase as a single repository consisting of 3 sub applications, 
1. the marketing pages application built with [NextJS](https://nextjs.org/), 
2. the UI library, which houses the design system used by all apps 
3. the product app built with [Creact React App](https://facebook.github.io/create-react-app/).
4. An internal shared library used accross applications.

[Yarn workspace](https://yarnpkg.com/lang/en/docs/workspaces/) was used in wiring up these applications so that a single `yarn install` pulls all the necessary dependencies required by all the different applications and libraries. Each application is a [node]() package and has its own script commands for deployment.

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
```
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


## The Problem:
The above project setup makes it very easy to setup a development environment in a matter of minutes. The only command required is `yarn install` and all dependencies are available.

But when thinking about deploying to production, The following problems occured.
1. Inconsistency in build during development and on production platform [Netlify](https://www.netlify.com/). The solution was to use [Gitlab Piplines]() and build the applications in a Docker container. The resulting build, grouped by folders, was then commited to a deploy repo which is then picked up by [Netlify](https://www.netlify.com/)

2. The time to build all the apps took over 17 minutes. This made deploying small changes very painful and increased the time required to resolve bug fixes

The first problem was somewhat tolerable but the second was unacceptable. The time to build needed to come down considerably.

## The solution.
Irrespective of what solution was proposed, one thing was certain, it shouldn't affect the development setup process. Cloning the base repository and running `yarn install` should still pull all the dependencies of the corresponding applications.

The proposed solution was to get all sub applications as [Git submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules). This meant each sub application had its own git repository and could be deployed in isolation. As problems occurred in a particular application, it was immediately addressed and deployed. No more intermediate Docker build step on [Gitlab Pipelines].

**But then both the marketing pages app and the product application failed to build in isolation**

The problem was obvious. In the case of the [NextJS]() marketing application, the shared ui library wasn't linked anymore in the `node_modules`. This was what [Yarn Workspaces]() solved by automatically running `yarn link` on all the **packages/sub_applications**. 

The UI Libary needed to be deployed in its own git repository and since [it was possible to install a node package directly from any git repository](), the `package.json` file for the [NextJS]() codebase was updated to look like this

**repo/packages/marketing-app
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

By ensuring that a specific git tag was used instead of a particular branch, deployment needed to be an explicit action. To get the latest changes from the `ui-lib` codebase, the git tag had to be bumped to a new version. This ensures more experimentation without having to worry about regressions or breaking changes as a result of new commits in either the `ui-lib` or the `shared-lib` codebase.

These changes were made directly on the `master` branch of the `marketing app` codebase so that the `develop` branch remains similar to what it was and the **mono repo** codebase housing all the submodules references commit from the `develop` branches of all the attached sub_modules

An intermediate branch `staging` was created, to get the changes on the `develop` branch to the `master` branch, while also ensuring that the `dependencies` in the `master` branch never crept over to the `develop` branch and vice versa.

## The Result
The following were benefits gotten as a result of the solution above.
1. The time required to get a new feature or a bug-fix over to production came down from 17 minutes to 3 minutes.
2. Guaranteed versioning of dependencies just like those hosted on [npm](https://www.npmjs.com/)
3. Simpler build script as provided by the framework of choice used, instead of complicated setup using Docker containers.

You don't have to sacrifice the Developer Experience provided by most mono repository setup up when using [Git submodules](). You also don't have to battle with the stress and effort involved in mono repo deployments. 
