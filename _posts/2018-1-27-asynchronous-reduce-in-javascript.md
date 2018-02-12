---
layout: post
title: Asynchronous Reduce in JavaScript
keywords: javascript, reduce, fold, asynchronous, seva zaikov, bloomca, promises, iterators, generators, map, filter
---

Reduce is a very powerful concept, coming from the functional programming (also known as `fold`), which allows to build any other iteration function – `sum`, `product`, `map`, `filter` and so on. However, how can we achieve asynchronous reduce, so requests are executed consecutively, so we can, for example, use previous results in the future calls?

> In our example, I won't use previous result, but rely on the fact that we need to execute these requests in this specific order

Let's start with a naïve implementation, using just normal iteration:

> I use [async/await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) here, which allows us to wait inside `for ... of`, or regular `for` loop as it was a synchronous call!

```js
async function createLinks(links) {
  const results = [];
  for (link of links) {
    const res = await createLink(link);
    results.push(res);
  }
  
  return results;
}

const links = [url1, url2, url3, url4, url5];
createLinks(links);
```

This small code inside is, basically, a reducer, but with asynchronous flow! Let's generalize it, so we'll pass handler there:

```js
async function asyncReduce(array, handler, startingValue) {
  let result = startingValue;

  for (value of array) {
    // `await` will transform result of the function to the promise,
    // even it is a synchronous call
    result = await handler(result, value);
  }

  return result;
}

function createLinks(links) {
  return asyncReduce(
    links,
    async (resolvedLinks, link) => {
      const newResolvedLink = await createLink(link);
      return resolvedLinks.concat(newResolvedLink);
    },
    []
  );
}

const links = [url1, url2, url3, url4, url5];
createLinks(links);
```

Now we have fully generalized reducer, but as you can see, the amount of code in our `createLinks` function stayed almost the same in size – so, in case you use once or twice, it might be not that beneficial to extract to a general `asyncReduce` function.

## No async/await

Okay, but not everybody can have fancy async/await – some projects have requirements, and async/await is not possible in the near future. Well, another new feature of modern JS [is generators](http://blog.bloomca.me/2017/12/19/how-to-use-generators.html), and you can use them to essentially repeat the same behaviour (and almost the syntax!) as we showed with async/await. The only problem is the following:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Have you ever used iterators/generators in JS?</p>&mdash; Asen Bozhilov (@abozhilov) <a href="https://twitter.com/abozhilov/status/940322101772374016?ref_src=twsrc%5Etfw">December 11, 2017</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Apparently, not so many projects/people dive into generators, due to their complicated nature and alternatives, and because of that, I'll separate our `asyncReduce` immediately, so you can hide implementation details:

```js
import co from 'co';

function asyncReduce(array, handler, startingValue) {
  return co(function* () {
    let result = startingValue;

    for (value of array) {
      // however, `co` does not wrap simple values into Promise
      // automatically, so we need to do so
      result = yield Promise.resolve(handler(result, value));
    }

    return result;
  });
}

function createLinks(links) {
  return asyncReduce(
    links,
    async (resolvedLinks, link) => {
      const newResolvedLink = await createLink(link);
      return resolvedLinks.concat(newResolvedLink);
    },
    []
  );
}

const links = [url1, url2, url3, url4, url5];
createLinks(links);
```

You can see that our interface remained the same, but the inside changed to utilize [co](https://github.com/tj/co) library – while it is not that complicated, it might be pretty frustrating to understand what do you need to do, if we ask all users of this function to wrap their calls in `co` manually. You also will need to import `co` or to write your own generator runner – which is not very complicated, but one more layer of complexity.

## ES5

Okay, but what about good old ES5? Maybe you don't use [babel](https://babeljs.io/), and need to support some old JS engines, or don't want to use generators. Well, it is still good – all you need is available implementation of [promises](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Promise) (which are [hard to cancel](http://blog.bloomca.me/2017/12/04/how-to-cancel-your-promise.html)) – either native or any polyfill, like [Bluebird](http://bluebirdjs.com/docs/getting-started.html).

```js
function asyncReduce(array, handler, startingValue) {
  // we are using normal reduce, but instead of immediate execution
  // of handlers, we postpone it until promise will be resolved
  array.reduce(
    function (promise, value) {
      return promise.then((acc) => {
        return Promise.resolve(handler(acc, value));
      });
    },
    // we started with a resolved promise, so the first request
    // will be executed immediately
    // also, we use resolved value as our acc from async reducer
    // we will resolve actual async result in promises
    Promise.resolve(startingValue)
  );
}
```

While the amount of code is not bigger (it might be even smaller), it is less readable and has to wrap your head around it – however, it works exactly the same.
