---
title: Using Github as my package registry
date: "2019-02-27T16:05:56.281Z"
draft: true
---

More often than not, I find myself working with libraries whose authors aren't actively maintaining anymore. And you can't blame them. They might have moved on to other things and the project doesn't interest them anymore. 

In these scenarios, I end up forking the project and applying the required fix for the project at hand. The question that then comes up is how this dependency would be pulled when deploying to a production/staging environment.

Fortunately, most package managers that I am familiar with, specifically npm and pip, support pulling dependencies from git repositories.

Take pip for example, installing a git repository from the command line is shown below

```bash
$ pip install -e 
```