---
layout: post
title: Asynchronous Patterns in JavaScript
keywords: javascript, asynchronous, async, waiting once, loading scripts, dynamic loading, loading libraries, mvar, waiting, promises, seva zaikov, bloomca
---

Sometimes you need to load something once, like a script, and you need to execute your code only _after_ loading it. For example, you have some magical library, but it weights around 1 MB, so you don't want to just load it, until you actually use it. One possible solution would be to use [code splitting](https://webpack.js.org/guides/code-splitting/), but let's assume it is not a viable solution (also this approach is good not only for scripts).

> I've used all these patterns in production code, and while they all just "make sense", I thought it might be helpful to list all of them.

So we write something like the following:

```js
function loadLibrary() {
  return new Promise((resolve, reject) => {
    // create new script element, which will load our script
    // and execute it globally
    const script = document.createElement('script');
    // we want to hide the fact, that we add a global property
    // so we resolve this object in returned promise
    script.onload = () => resolve(window.OurGlobalLibrary);
    script.onError = reject;
    script.src = 'https://cdn.example.com/your_global_library.js';

    document.body.appendChild(script);
  });
}
```

This code will be problematic, though, if we invoke this code somewhere else, like in 2 separate components, which both want to use this heavy library – we will download and execute this library again (and some libraries are not very happy about it). But we explicitly want to do it only once, so let's fix it.

Fix is pretty small – we just need to use [once](https://lodash.com/docs/4.17.4#once):

```js
// my own implementation, just to show how it works internally
function once(fn) {
  return function(...args) {
    let res;
    let invoked = false;

    if (!invoked) {
      res = fn(...args);
      invoked = true;
    }

    return res;
  };
}

// if this function will be executed more than once, every time
// after first call, we will get result of the initial call
const loadLibraryOnce = once(loadLibrary);
```

## Disconnecting promise and loading

Let's complicate our requirements a little bit – let's say we want to execute some code after loading our library, but this code _should not_ initiate library's loading. In other words, we want to react to the library's presence in the app: we can use some sort of event emitter, broadcasting event like `ourLibrary:loaded`, but sometimes you don't want to introduce this pattern just for one message, or you don't want to rely on strings.

```js
let resolveLibraryPromise;
let rejectLibraryPromise;

const libraryPromise = new Promise((resolve, reject) => {
  // we just assign functions to variables in clojure
  // they will be called later after we load our lib
  resolveLibraryPromise = resolve;
  rejectLibraryPromise = reject;
});

function loadLibrary() {
  const script = document.createElement('script');
  script.onload = () => resolveLibraryPromise(window.OurGlobalLibrary);
  script.onError = rejectLibraryPromise;
  script.src = 'https://cdn.example.com/your_global_library.js';

  document.body.appendChild(script);

  return libraryPromise;
}

const loadLibraryOnce = once(loadLibrary);

// ==============
// Usage:

libraryPromise.then(() => {
  // e.g. render some data
  // but only after the library was loaded
  renderLibraryConsole();
});
```

Example of such a usage – let's say you want to render statistics of usage, but only after library was loaded. If library is not here, it means that there is nothing to show, but as soon as it was requested by any component, we'd like to display it, and such an approach will help us. Also, if we use code splitting, we'd have to create the same wrapper around it.

## Extending to mutliple

Our last improvement will be to drop a requirement to wait only once, and to allow replacement of resolved values (but we won't stream updated results, we will keep promise interface). Let's first write a naïve implementation, and later abstract it.
Waiting for library multiple times does not make a lot of sense, so let's switch to loading user's object.

```js
let user;
let userExists = false;
let promises = [];

function _createPromise() {
  let resolvePromise;
  let rejectPromise;
  promise = new Promise((resolve, reject) => {
    resolvePromise = resolve;
    rejectPromise = reject;
  });
  return { promise, resolvePromise, rejectPromise };
}

// instead of `libraryPromise`, we need to call a function
// to return current promise
function readUser() {
  // if we already have cached user's object, we can just
  // resolve it immediately
  if (userExists) {
    return Promise.resolve(user);
  }

  // otherwise, we create a new promise
  const promiseData = _createPromise();

  // we save `resolve` handler
  promises.push(promiseData.resolvePromise);

  return promiseData.promise;
}

// we put new user's object, resolving all 
function putUser(userObject) {
  user = userObject;
  userExists = true;

  // we resolve all waiting promises with user's object
  promises.forEach(resolve => resolve(user));
  promises = [];
}

// this function will remove user's object, and will cause
// calls `readUser()` to wait until we put new user's object
function removeUser() {
  userExists = false;
}
```

You can see that the amount of code increased significantly – now we have to keep track of all places which want to read the next user's object, and we can't return the same promise, because it is not a singletone anymore!
Let's see how we can use it:


```js
function checkPermissions() {
  const user = await readUser();

  // do some checks using latest user object
}

// somewhere in the app
function getApplication() {
  const application = await fetchApplication();

  // after this line permissions will be resolved (in case they were called)
  putUser(application.user);
}
```

What is important here is that what we want in our checking permissions function is only the latest user object, but we don't control it's loading at all – we are sure that some other code will take care about it. Also, we do it only once, so it means that there is no subscribing functionality, and if you need it, think about using some sort of [streams](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754).

## MVar

Now, the last part is about abstracting code from above. This pattern resembles concurrent pattern from Haskell [called MVar](https://hackage.haskell.org/package/base-4.10.1.0/docs/Control-Concurrent-MVar.html), which allows you to read value and to put new value.

Let's abstract the code from the previous chapter:

```js
class MVar {
  constructor() {
    this.promises = [];
    this.valueExists = false;
  }

  read() {
    if (this.valueExists) {
      return Promise.resolve(this.value);
    } else {
      return new Promise((resolve) => {
        this.promises.push(resolve);
      });
    }
  }

  put(value) {
    this.valueExists = true;
    this.value = value;

    this.promises.forEach(resolve => resolve(value));
    this.promises = [];
  }

  empty() {
    this.valueExists = false;
  }
}
```

Code is much cleaner, right? Now, using this abstraction we can create our user's object reader:

```js
const userMVar = new MVar();

function checkPermissions() {
  const user = await userMVar.read();

  // do some checks using latest user object
}

// somewhere in the app
function getApplication() {
  const application = await fetchApplication();

  // after this line permissions will be resolved (in case they were called)
  userMVar.put(application.user);
}
```

These patterns (especially in the end) are pretty hard to follow and not so intuitive, so I can recommend to use them only in case you really need these specific use-cases.