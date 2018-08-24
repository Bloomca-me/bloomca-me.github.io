---
layout: post
title: "Javascript Fundamentals: `this` keyword"
keyword: javascript, javascript tutorial, how this works in javascript, tutorial on this in javascript, this keyword in javascript, this in javascript, method chaining javascript, arrow functions in javascript, seva zaikov, bloomca, ES2015, ES6
---

One of the quirkiest parts of JavaScript is `this` keyword. Unlike other quirky parts of JavaScript, it is extremely hard to avoid – it's used in prototypal inheritance, which is an important part of the language, and with introducing [class keyword](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes) in ES2015 (which are, under the hood, almost the same as constructor functions with prototypes) they are used even more.
There is nothing bad about using `this`, but you need to be extra careful while using it and, especially, passing around a reference to functions which use `this` keyword.

## Table of Contents

- [Global context](#global-context)
- [Common usage](#common-usage)
- [Tricky examples](#tricky-examples)
- [Bind, Apply and Call](#bind-apply-and-call)
- [Arrow functions](#arrow-functions)
- [Method Chaining](#method-chaining)

## Global context

In a nutshell, `this` is a current context. It refers to an environment, in which our code is executed right now – for example, if you open a browser console window and type `this`, you'll get a reference to the `window` object. Also, check `this === window` will return `true`. In a node.js environment, the global object is called [global](https://nodejs.org/api/globals.html#globals_global), and check `this === global` also returns `true`.

However, it isn't recommended to use it in the global scope, and for good reasons: it is very confusing, and it is discouraged to use global objects in general (since every library can modify it, it's pretty easy to get into trouble – before the bundlers era it was a common problem). You will probably never encounter this type of usage in your career, so we won't talk about it it anymore.

A more interesting and common example is the usage of `this` inside functions. Let's look at the following example: try to tell, what is the value of `this`?

{% highlight js linenos=table %}
function someFunction() {
  console.log(this); // what is it?

  return this;
}
{% endhighlight %}

You might know or guess that `this` would be equal to a global object as well (or `undefined`, in case we are in strict mode). Even though it probably will be, this answer is not fully correct – `this` exists only during execution of the code, not during evaluation. But in general, functions in variables (not methods of objects) have a global object as `this` (or `undefined` in strict mode, but we won't focus on it):

{% highlight js linenos=table %}
function someFunction() {
  return this;
}

console.log(someFunction() === window); // true
{% endhighlight %}

> Each call of this function will get its own `this`, and while regular calls will have a global object as context, we will show later how we can actually call it with a different `this`.

## Common usage

Now, let's see an example where `this` has a meaningful value. For this, we'll create an object with a property and a method, which accesses the object's property internally.

{% highlight js linenos=table %}
const object = {
  value: 42,
  getAnswer(question) {
    if (!question.endsWith('?')) {
      throw new Error('This is not a question!');
    }

    return this.value;
  }
};

const answer = object.getAnswer('what is the meaning of life?');
console.log(answer); // 42
{% endhighlight %}

We can see that the method of an object has this object in context, and it makes sense! This is the most common-use case, but the devil is in the details, as usual. This detail is called `reference` and it starts to matter when you try to pass your methods around (remember the famous event handlers in React?). So, as I mentioned before, `this` is "assigned" at the moment of execution, and in the simplest form, it will be an object before the dot (no matter how many levels deep, it will be the closest one on the left) – if there isn't one, imagine there is a global one (like `window.someFunction()`). In case you assign a method to another variable (essentially, this is exactly what happens when you pass it somewhere, like in React), it loses the connection to its object. Moreover, you can assign it as a property to another object and, after calling it, you will have the new object as `this`!

{% highlight js linenos=table %}
const object = {
  value: 42,
  getAnswer(question) {
    if (!question.endsWith('?')) {
      throw new Error('This is not a question!');
    }

    return this.value;
  }
}

const newObject = {
  value: 3301
};

newObject.getAnswer = object.getAnswer;

const answer = newObject.getAnswer('what is the most mysterious secret out there?');
console.log(answer); // 3301
{% endhighlight %}

Let's also illustrate a several-levels-deep object, and the fact that `this` refers to the "closest" object, which has this function as a method:

{% highlight js linenos=table %}
const a = {
  value: 42,
  b: {
    value: 3301,
    getContext: function() {
      console.log(this.value);
      return this;
    }
  }
};

console.log(a.b.getContext() === a.b); // true
// it will print 3301 as well
{% endhighlight %}


Now we can get back to the original example with a globally defined function and a question about `this`. With our new knowledge we will notice that nothing prevents us from assigning our function to some object's property, and in case of a normal call, it will have this object as `this`:

{% highlight js linenos=table %}
function someFunction() {
  return this;
}

const a = {
  b: 42,
  c: someFunction
};

console.log(a.c() === a); // true
{% endhighlight %}

## Tricky examples

Let's look into some examples where we actually lose the context. Let's start with a node.js example using [utils.promisify](https://nodejs.org/api/util.html#util_util_promisify_original), which allows to convert [callback-styled](https://blog.bloomca.me/2018/06/21/nodejs-guide-for-frontend-developers.html#callback-style) functions to more modern promise-based ones. This is a fairly simple change, as all these functions follow the same interface of `function (err, data) {}`, but what is interesting for us is the fact that we pass a function as a first argument, so we pass a _reference_. It is important to understand that it is just a reference, and it has no connection to the original object whose method it is (however, we can add it, as explained in the [bind method](#bind-apply-and-call) section).

In the provided by documentation example [fs.stat](https://nodejs.org/api/fs.html#fs_fs_stat_path_options_callback) is used, but the FileSystem module does not use `this`, so it does not cause any troubles. However, something like [new AWS.DynamoDB().createTable](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html#createTable-property) relies on it, and just passing it inside will return a non-working function. In order to avoid that, use [bind method](#bind-apply-and-call).

<br />

-----

<br />

Let's look into more obscure examples. There are plenty of JS quizzes which will make you want to pull out your hair, but with our new knowledge, we can explain some weird questions where we need to carefully apply our knowledge about references and execution context:

{% highlight js linenos=table %}
const a = {
  b: 42,
  c: function() {
    return this.b;
  }
};

(a.c || [])(); // 1
(a.c)(); // 2
(1, a.c)(); // 3
{% endhighlight %}

Let's go over these expressions one by one. First of all, we need to understand parentheses here – if we wrap something in parenthesis, it will just return an expression inside (it is useful to create self-invoking functions, for example – `(function() { ... })()`). So, in our first example we first execute an expression inside, and then return it, and execute it without any arguments. What is important here, is that we actually have a result of `OR` operator, and it is a method of an object, but not an object itself! We can rewrite it to something like this:

{% highlight js linenos=table %}
const method = a.c || [];
(method)();
{% endhighlight %}

As we already know, assigning a method to a variable and then calling it without anything on the left before a dot, is equal to calling with `window` as `this` context – so, unless you have something in `window.b`, it will be `undefined`.

Now to the second expression. As I said, parentheses don't do anything – they just return what is inside, but what is important to understand is that it does not evaluate this expression, rather just pops up:

{% highlight js linenos=table %}
a.c(); // this is what it looks after removing parenthesis
{% endhighlight %}

So we end up with a regular call, in which context is an object on the left of the dot – this works how we want!

The last example uses [comma operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Comma_Operator), and the trick of this example is that it also evaluates it and returns the last expression, so we end with "reassigning" again, and we lose context once again, so it will be global.

> You can actually encounter such a question in a job interview. If you do, don't be upset, just ask why do they think such details of JavaScript interpreter are important for this job.

## Bind, Apply and Call

We learned that the context is the object to which our functions belong at the moment of execution. Using this knowledge, we can create our own `bind` function, which will create a new function based on the existing one, which will pass the right context at the moment of exection:

{% highlight js linenos=table %}
function bind(fn, context) {
  return function(...args) {
    const key = 'some_secret_key';
    Object.defineProperty(context, key, {
      configurable: true,
      enumerable: false,
      value: fn
    });
    const result = context[key](...args);

    delete context[key];

    return result;
  };
}
{% endhighlight %}

This is quite a hacky approach (especially assigning it to a "secret" property and then deleting it), but it should work: we make the passed function a non-enumerable property of the context, and remove it after calling. Because `context` is on the left side of the called function now, it will be in the `this` variable, which is exactly what we need.

However, you don't need to write such helpers by yourself, they already exist in the standard library. Every function has [Function.prototype.bind methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_objects/Function/bind), which works the same as our hacky function: it creates a new function with a provided context, which is attached to this new function no matter what. The last part is important: you can bind it again, or assign it to an object, or (as we'll learn in a minute) call it using `call` or `apply` – `this` will always be from the original binding call.

Also, you can pass any number of arguments, and a new function will automatically use them in calls, so it will require less arguments than the original function. This is similar to [partial application](https://en.wikipedia.org/wiki/Partial_application), and to illustrate, let's produce a couple of additional functions using just `bind`, without writing any new code:

{% highlight js linenos=table %}
function multiply(x, y) {
  return x + y;
}

const multiplyBy2 = multiply.bind(null, 2);
const multiplyBy5 = multiply.bind(null, 5);
{% endhighlight %}

There are other ways to use different context, [Function.prototype.call](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call) and [Function.prototype.apply](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply). They both work almost the same, but the difference is that the `call` method accepts arguments listed as a list, and `apply` requires an array. Before introducing [spread syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax) when you had a dynamic number of arguments you _had_ to use `apply`, now you can just use `.call(ctx, ...this)`.

The most important thing about them is that they don't create a new function. Rather, they switch the context during the execution time to one of the provided ones. Because it is an actual execution, you must provide all of the arguments after the context (in a slightly different way, described above). Why would you want to do that? Well, the first usage is when you actually pass context around as an argument, and execute functions with it. Another one was using `apply` when you were not sure about the number of arguments – [Math.max](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/max) is a perfect example of that (spread syntax basically eliminated this usage):

{% highlight js linenos=table %}
// context might be anything, Math.max does not use it
const values = [2, 3, 12, 5, 15];
const max = Math.max.apply(null, values); // 15

// nowadays just use this syntax
const easierMax = Math.max(...values);
{% endhighlight %}

The last usage is quite tricky – we use methods with a different `this` intentionally. Why would we want this?
A canonical example is [NodeList](https://developer.mozilla.org/en-US/docs/Web/API/NodeList), which is an object with items at numerical indexes (0, 1, 2, 3, etc) and has a property length, which is enough for array methods to work on it! However, it does not have an [Array.prototype](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/prototype) object in its prototype, so these methods do not exist. You can call them directly, though, passing NodeList as a context:

{% highlight js linenos=table %}
const images = document.querySelectorAll('img');

Array.prototype.forEach.call(images, img => {
  console.log(img.getAttribute('src'));
});
{% endhighlight %}

> All these methods are not that common to see – arrow functions, spread syntax, [for...of](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of) eliminated a lot of cases where you need to use it. But some things, like binding, are still used – sometimes to bind a context, sometimes to use it as a partial application.

## Arrow functions

In EcmaScript 2015 we got a new feature called [arrow functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions). They are great for brevity, but for us the most important thing is that it does not have its own `this` and simply grabs one from the outside context. Essentially, it means that it will have the context of the function in which it was _created_ (not necessarily executed). This is a working solution to the React event handlers problem, or to any event handlers; for simplicity in the following example I'll use [setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/setTimeout) callbacks:

{% highlight js linenos=table %}
const a = {
  b: 42,
  c: function(cb) {
    setTimeout(function() {
      return cb(this.b);
    }, 500);
  }
};

a.c(function (value) {
  console.log(value);
});
{% endhighlight %}

The output will be `undefined`, not `42`, as we want. We know the answer already – we pass a reference to a function without a bound context, so it will have global context in `this`, and unless you assigned `b` to it, it will be `undefined`.

What can we do to solve this problem? For example, we can assign `this` to a variable (usually called `self` or `that`), and then use it from the closure to get the value:

{% highlight js linenos=table %}
const a = {
  b: 42,
  c: function(cb) {
    const self = this;
    setTimeout(function() {
      return cb(self.b);
    }, 500);
  }
};

a.c(function (value) {
  console.log(value); // 42
});
{% endhighlight %}

There are other solutions, like to bind the anonymous function to the current context, which will create a new function, passed as a callback to `setTimeout`; or to refer to this object as `a` (we have this variable in closure). However, it all looks a little bit weird, does it not? The right context is right there, and even at the moment of creation we had the right `this`! We know the reason – when we create a function, it doesn't know anything about its context, it will be defined during execution.

Arrow functions address this issue – they don't have their own context, which is defined during execution, rather they take the current one during _creation_ time. So they work exactly as you might think functions should work, when you pass them into `setTimeout`. The following code will work properly:

{% highlight js linenos=table %}
const a = {
  b: 42,
  c: function(cb) {
    setTimeout(() => cb(this.b), 500);
  }
};

a.c(function (value) {
  console.log(value); // 42
});
{% endhighlight %}

> Be careful with arrow functions. Sometimes you actually want functions to get context during execution, and also they don't have their [arguments](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/arguments), they also take it from the outside.

## Method Chaining

Slightly unrelated, but helpful to know is [chaining method pattern](https://en.wikipedia.org/wiki/Method_chaining), which is often enabled by using `this`. It is used to refer to several calls combined by `.`, so each method returns an object which has the next method. Usually it is used in object-oriented code, and usually all methods are performed on some object, and in return we receive the same object, so we can call the same methods several times.

In JavaScript the best example of this is probably the chaining of iterating over an array:

{% highlight js linenos=table %}
const array = [{ value: 42 }, { value: 33 }, { value: 12 }];

const report = array
  .map(({ value }) => value)
  .filter(value => value > 30)
  .map(value => `Value is: ${value}`)
  .join('\n');
{% endhighlight %}

How does it work? We don't have the power to look into navide code, but we can easily guess how it works. First of all, all methods exist in an array prototype, and `this` is an actual array for them. All these methods return another array, so even though it is a different array (not the original one) it has all the same methods (since they share the prototype). But keep in mind, that in the end we got a string, so only [String.prototype methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/prototype#Methods) are available.

To illustrate, let's write our own versions `map` and `filter` for an array prototype:

{% highlight js linenos=table %}
Array.prototype.map2 = function(fn) {
  const newArray = [];
  for (let i = 0; i < this.length; i++) {
    const currentValue = this[i];
    newArray.push(fn(currentValue));
  }

  return newArray;
};

Array.prototype.filter2 = function(fn) {
  const newArray = [];
  for (let i = 0; i < this.length; i++) {
    const currentValue = this[i];
    if (fn(currentValue)) {
      newArray.push(currentValue);
    }
  }

  return newArray;
};

// our previous code should work the same with the new methods:
const array = [{ value: 42 }, { value: 33 }, { value: 12 }];

const report = array
  .map2(({ value }) => value)
  .filter2(value => value > 30)
  .map2(value => `Value is: ${value}`)
  .join('\n');
{% endhighlight %}

> We don't return `this` in our functions, because `map` and `filter` both create a new array, but if they mutated it, in the response we'd simply write `return this`.


<br />

----

<br />

To summarize, most of the time `this` works as expected. Sometimes it is confusing, and sometimes it is extremely tricky to use the right context. Whatever style you prefer, try to avoid confusing usage; use arrow functions if you can; pass it along; and remember: practice makes perfect!

Go get 'em Tiger!