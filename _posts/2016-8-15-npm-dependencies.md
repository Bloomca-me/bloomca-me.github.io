---
layout: post
title: Npm dependencies explained
keywords: npm, dependencies, devDependencies, peerDependencies, difference, package.json, javascript
excerpt: "NPM has many types of dependencies: normal, dev, optional, peer. Let's look what is the difference between all of them."
---

## Hidden complexity of npm's dependencies

Recently (already famous) [yarn](https://github.com/yarnpkg/yarn) was published, which promises to solve all your problems related to the dependencies management, but while some oldfags don't use it in production yet (what a shame!), I would like to describe how npm covers different type of dependencies.

Let's start with [npm documentation](https://docs.npmjs.com/files/package.json#devdependencies):

- dependencies

  > Dependencies are specified in a simple object that maps a package name to a version range.

- devDependencies

  > If someone is planning on downloading and using your module in their program, then they probably don't want or need to download and build the external test or documentation framework that you use

- peerDependencies

  > In some cases, you want to express the compatibility of your package with a host tool or library, while not necessarily doing a require of this host. This is usually referred to as a plugin

Also, npm adds two more variables, which adds confusion:

- bundledDependencies
- optionalDependencies

But we have enough with first three options – basically, we'll see that it is already too much. The problem is that despite starting as a good idea, as we all know, "The road to Hell is paved with good intentions" – the attempt to separate concerns in dependencies just has added more confusion – [1](http://stackoverflow.com/questions/26737819/why-use-peer-dependencies-in-npm-for-plugins), [2](http://stackoverflow.com/questions/35207380/how-to-install-npm-peer-dependencies-automatically/35207983), [3](http://stackoverflow.com/questions/18875674/whats-the-difference-between-dependencies-devdependencies-and-peerdependencies), etc, without even mentioning numerous attempts to explain how to deal with different types of them.

So, the easiest case is when develop our project, and it is our package itself – then we need all our dependencies, and we just install everything. Running `npm i` inside our project will install both `dependencies` and `devDependencies`, and for `peerDependencies` the behaviour is different for `npm` versions – `3+` will ignore them (but will warn you that your package requires it, but it wasn't install), and previous versions will install them. So, here everything is clean, and we are moving to the most interesting case – what if we have the package as the dependency.

We have two prospectives here, and the first one is as a project developers. Usually `peerDependencies` are libraries which we need to develop and debug our code – good example is `react` package for some react component – you can't use it without react itself, so the argument that there is no reason to bring it with the package makes total sense. So, as a good intended people we move it to the `peerDependencies`, but then we have to install it manually (because in JS ecosystem if you don't use latest node, you're a loser) or add it to some other dependencies. And here goes the main difference between them – `devDependencies` are installed only we invoke installation from the package itself, so if it is a dependency itself – it won't install it. And then we'll end up with a manual installation in our own project. If a dependency lies in regular `dependencies`, when the size of your bundle will increase (unless you already require the same library number, thanks to default npm dedupe), but using any modern bundler it will be noticed and excluded. There is a possibility that the size of your final bundle will bloat, but it is really hard to trace this issue.

The `peerDependencies` tight version requirement can even [break your code](http://stackoverflow.com/questions/36880563/how-to-handle-npm3-peer-dependency-conflict), so you with new version of interesting library you might be needed to wait until author will change it's `peerDependency` to support your version. So, it means that `peerDependency` is needed only for showing absolutely incompatible APIs (like `"react": ">0.13"`, when it was _very_ different) – so, don't be shy to put very broad requirements. Omitting it completely can bait you (or someone else) in the future, therefore it is very nice to warn people about non-working versions. So, being said that final bundle has the same size, it seems that there is no reason not to put your `plugin` dependencies to all common `dependencies` as well. There is no standartized answer to this question – some people put everything to `dependencies`, some like to separate concerns nicely – but you never know, what will [happen after](https://nodejs.org/en/blog/npm/peer-dependencies/).
