---
layout: post
title: "Asynchronous Javascript Patterns: Exclusive Task"
keywords: javascript, javascript async patterns
tags: javascript async_javascript
---

> I have more articles about asynchronous javascript!
> * [How to Cancel Your Promise](https://blog.bloomca.me/2017/12/04/how-to-cancel-your-promise.html)
> * [Loading patters](https://blog.bloomca.me/2018/03/24/async-patterns-js.html)
> * [Asynchronous Reduce in JavaScript](https://blog.bloomca.me/2018/01/27/asynchronous-reduce-in-javascript.html)
> * [How to use Generators](https://blog.bloomca.me/2017/12/19/how-to-use-generators.html)

Sometimes we want to execute some asynchronous code, and use results of it later, but in many places – something like a token: we might use it for every other request.

Now, let's imagine this token can be invalidated and we need to refresh it afterwards, but it does not require any action from the user – and we don't want to interrupt any active data fetching. It means in case of failed request, we need to check whether token became invalid, and in that case fetch new one, and repeat the request with the same parameters. Moreover, there might several concurrent requests, and we don't want all of them to refresh the token – we just want all of them waiting for one.

Let's write some code, which assumes token is available and always valid, and if something is wrong, it just redirects to the login page:

```js
function checkStatus(response) {
  if (response.status >= 200 && response.status < 300) {
    return response;
  } else if (response.status === 403) {
    window.location.href = '/href';
    return;
  } else {
    var error = new Error(response.statusText);
    error.response = response;
    throw error;
  }
}

function parseJSON(data) {
  return data.json();
}

function get(url, params) {
  const paramsWithToken = {
    ...params,
    accessToken: getToken()
  };
  const query = stringify(params);
  const urlWithParams = `${url}${query}`;
  return fetch(urlWithParams)
    .then(checkStatus)
    .then(parseJSON);
}
```

Let's handle it now in naïve way – we assume that we receive custom `Token is invalid` message in status text, and our function `refreshToken()` will handle redirect to the login page if it is impossible without user input.
The first step is to add one more condition into our `checkStatus`:

```js
// we need to mark somehow that we want to request it again
// we use [Symbol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol)
// to avoid any possible ambiguity and 
const HAS_TO_CALL_REQUEST_AGAIN = Symbol();

async function checkStatus(response) {
  // ...
  } else if (response.status === 403 && response.statusText === "Token is invalid") {
    await refreshToken();
    return HAS_TO_CALL_REQUEST_AGAIN;
  } // ...
}
```

Now we need to modify our `get` function a little bit, so that we react to our Symbol value, which tells us we need to call this function again.

```js
function get(url, params) {
  const paramsWithToken = {
    ...params,
    accessToken: getToken()
  };
  const query = stringify(params);
  const urlWithParams = `${url}${query}`;
  return fetch(urlWithParams)
    .then(checkStatus)
    .then(response => {
      if (response === HAS_TO_CALL_REQUEST_AGAIN) {
        // we need to check status, because we are already in the promise chain
        return get(url, params).then(checkStatus);
      }

      return response;
    })
    .then(parseJSON);
}
```

You can see an interesting effect of using a promise chain – inside our added code, after checking status, we can't just call the function again, because we'll fall through anyway, so we have to call already executed functions by ourselves (in our case, it is only `checkStatus`). We can workaround it using [async/await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function):

```js
async function get(url, params) {
  const paramsWithToken = {
    ...params,
    accessToken: getToken()
  };
  const query = stringify(params);
  const urlWithParams = `${url}${query}`;
  const response = await fetch(urlWithParams);
  const statusCheck = await checkStatus(response);

  if (statusCheck === HAS_TO_CALL_REQUEST_AGAIN) {
    // there is no promise chain, so we just return completely new promise
    return get(url, params);
  }

  return parseJSON(response);
}
```

They both work the same, though, but `async/await` version is more flexible.
Now we solved one part of the problem – in case of invalid token we'll try to refresh it first, and if it was successfull, we will call the same endpoint once again, so user won't notice anything.

However, now we can end up in a situation, where several simultaneous requests need to refresh the token, and we don't really want to do that. We want to hide this complexity from the `checkStatus` function, though, and tweak only `refreshToken()` function.

Let's start with a default implementation, which does not take care of concurrency:

```js
let token = null;
async function refreshToken() {
  token = await fetchToken();

  return token;
}

function getToken() {
  return token;
}
```

Now let's refresh token only once, and if it is in progress, just return this promise. This will solve our problem of several concurrent requests:

```js
let token = null;
let tokenPromise = null;

async function refreshToken() {
  if (tokenPromise) {
    return tokenPromise;
  }

  tokenPromise = fetchToken();

  const token = await tokenPromise;
  tokenPromise = null;

  return token;
}
```

We are good at this point – the initial problem is solved, but what if we have other places where we can use the same strategy? Let's abstract this pattern:

> This is an abstraction, which makes sense _only_ if you have a lot of places to use this pattern. As you saw, overhead is not that big, so if you have 1–2 places to use, feel free and just inline solution above

```js
function createExclusiveTask(fn) {
  let promise = null;

  return async function(...args) {
    if (promise) {
      return promise;
    }

    promise = fn(...args);

    const result = await promise;
    promise = null;

    return result;
  }
}

function _refreshToken() {
  token = await fetchToken();
  return token;
}

// now all calls to the refresh token will automatically return a promise
// if it is in the progress
const refreshToken = createExclusiveTask(refreshToken);
```

This abstraction will allow you to write declarative code without thinking too much about network overhead – you can just request new token, user object, etc, and be sure that in case of several concurrent requests it won't make additional calls.

There is one more possibility to extend this function – we can add a way to accept parameters and distinguish promises based on them (e.g. if we can fetch several users, they will be separated by id), but it is highly dependant on your task and your data.

As a conclusion, this pattern might help you, in case you have it throughout the whole codebase, and it will allow you to remove some mental overhead – this is why we have patterns on the first place. But again, as I said – if you have just couple ocurrences, please inline these and don't overengineer from the beginning (in our example we improved already working solution, which was not very effective).
