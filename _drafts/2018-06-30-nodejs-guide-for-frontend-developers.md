---
layout: post
title: Node.js Guide for Frontend Developers
keywords: javascript, node.js, nodejs, nodejs guide, tutorial, seva zaikov, bloomca, ES2015, ES6, async/await, modern javascript
---

We deal with Node.js pretty often nowadays, even if you are just a frontend developer – npm scripts, webpack configuration, gulp tasks, programmatic run of bundlers or test runners. Even though you don't really need deep understanding for that sort of tasks, it might be confusing sometimes, and cause you write something in a very weird way just because of missing some key concept of Node.js. Familiarity with Node can also allow you to automate some things you do manually, feeling more confident looking into server-side code and write more complicated scripts.

> This guide is focused on frontend developers – people who know JavaScript, but are not very proficient with Node yet. I won't focus on the language here – Node.js uses V8, so it is the same interpreter as in Google Chrome, which you probably know about (however, it is possible to run on different VM, see [node-chakracore](https://github.com/nodejs/node-chakracore)).

## Node Version

The most schocking difference after client-side code is the fact that you decide on your runtime, and you can be absolutely sure in supported features – you choose which version you are going to use, depending on your requirements and available servers.

Node.js has a public [release schedule](https://github.com/nodejs/Release#release-schedule), which tells us that odd versions don't have long-term support,

Node.js is used extensively in modern frontend toolchain – it is hard to imagine modern project which does not include any processing using node tools, so you might be already familiar with [nvm](https://github.com/creationix/nvm) (node version manager), which allows you to have several node versions at the project simultaneously. The reason for such a tool is that different project very often are written using different Node version, and you don't want to constantly keep them in sync, you just want to preserve environment in which they were written and tested.
Such tools exist for many other languages, like [virtualenv](https://virtualenv.pypa.io/en/stable/) for Python, [rbenv](https://github.com/rbenv/rbenv) for Ruby and so on.

## No Babel

Because you are free to choose any Node.js version, there is a big chance that you can use LTS (Long term supported) version, which is [8.11.2](https://nodejs.org/en/) at the moment of writing, which supports [almost everything](https://node.green/) – all 2015 ECMAScript spec, except for [tail recursion](https://en.wikipedia.org/wiki/Tail_call).

It means that there is no need for Babel, unless you are stuck with old version of Node.js, or can't live without some bleeding edge transformation. In practice, it is not that crucial, so your running code is the same as you write it, without any transpiling – gift we already forgot on the client-side.

There is also no need for webpack or browserify, and therefore we don't have a tool to reload our code – in case you develop something like a web server, you can use [nodemon](https://github.com/remy/nodemon) to reload your application after filechanges.

Also, because we don't ship this code anywhere, there is no need to minify it – one step less: you just use your code as is! Feels really weird.

## Callback Style

Historically, asynchronous functions in Node.js accepted callbacks with a signature `(err, data)`, where first argument represented an error – if it was `null`, all is good, otherwise you have to handle the error.
For example, let's read a file:

{% highlight js linenos=table %}
const fs = require('fs');

fs.readFile('myFile.js', (err, file) => {
  if (err) {
    console.error('There was an error reading file :(');
    // process is a global object in Node
    process.exit(1);
  }

  // do something with file content
});
{% endhighlight %}

It was discovered soon that this style makes it extremely hard to write readable and maintable code, and even created a notion [callback hell](http://callbackhell.com/). At the moment, new asynchronous primitive was introduced, [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) – it was standartized at ECMAScript 2015 (it is a global object both for browser and Node.js runtime). Later, in ECMAScript 2017 [async/await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) was standartized, and it became available in Node.js 7.6+.

With promises we avoid "callback hell", but now we get the problem that old code, a lot of built-in modules still use these technique. However, it is not very hard to convert them to promises – to illustrate, let's convert [fs.readFile](https://nodejs.org/api/fs.html#fs_fs_readfile_path_options_callback) to promises:

{% highlight js linenos=table %}
const fs = require('fs');

function readFile(...arguments) {
  return new Promise((resolve, reject) => {
    fs.readFile(...arguments, (err, data) => {
      if (err) {
        reject(err);
      } else {
        resolve(data);
      }
    });
  });
}
{% endhighlight %}

This pattern can be easily extended to any function, and there is a special function in built-in `utils` module – [utils.promisify](https://nodejs.org/api/util.html#util_util_promisify_original). Example from official documentation:

{% highlight js linenos=table %}
const util = require('util');
const fs = require('fs');

const stat = util.promisify(fs.stat);
stat('.').then((stats) => {
  // Do something with `stats`
}).catch((error) => {
  // Handle the error.
});
{% endhighlight %}

Node.js core team understands that we need to move from old style, so they are trying to introduce promisified version of built-in modules – there is [promisified file system module](https://nodejs.org/api/fs.html#fs_fs_promises_api) already, although it is experimental at the moment of writing.

You still can encounter a lot of oldschool Node.js code with callbacks, and it is recommended to wrap them using `utils.promisify` for the sake of consistency.

## Event Loop

Event loop is almost the same as in the browser, but a little bit extended. However, since this topic is a little bit more advanced, I will cover it fully, not only the differences (I will highlight them though, so you know which part is Node.js-specific).

<p class="centred-image full-image">
  <img class="image" src="/assets/img/event_loop.jpg" />
  <em>Event loop in Node.js</em>
</p>

JavaScript is built with asynchronous behaviour in mind, and therefore very often we don't execute everything right there. Things which can be executed not in direct order:
- [microtasks](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)

For example, immediately resolved promises, like [Promise.resolve](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/resolve). It means that this code will execute in the _same_ event loop iteration, but after all synchronous code.

- [process.nextTick](https://nodejs.org/api/process.html#process_process_nexttick_callback_args)

This is Node.js specific thing, it does not exist in any browser. It behaves like a microtask, but with a priority, which means that it will be executed right after all synchronous code, even if other microtasks were introduced before – this is dangerous. Naming is unfortunate, since it is not really correct.

- [setImmediate](https://nodejs.org/api/timers.html#timers_setimmediate_callback_args)

While it does exist in [some browsers](https://developer.mozilla.org/en-US/docs/Web/API/Window/setImmediate#Browser_compatibility), it did not reach consistent behaviour across all of them, so you should be very careful using it in the browser. It is similar to `setTimeout(0)` code, but sometimes will take precedence over it.

- [setTimeout](https://nodejs.org/api/timers.html#timers_settimeout_callback_delay_args)/[setInterval](https://nodejs.org/api/timers.html#timers_setinterval_callback_delay_args)

Timers behave the same way both in Node and browser. One important thing about timers is that delay we put there is not a guaranteed time after which our callback will be executed. What it means – Node.js will execute this callback after this time _as soon_ as main execution cycle finished all operations (including microtasks) _and_ there are no other timers with higher priority.

Let's take a look at the example with all mentioned things:

> I'll put a correct output from script's execution further, but if you want to, try to go through it on yourself (be a "JavaScript interpreter"):

{% highlight js linenos=table %}
const fs = require('fs');
console.log('beginning of the program');

const promise = new Promise(resolve => {
  // function, passed to the Promise constructor
  // is executed synchronously!
  console.log('I am in the promise function!');
  resolve('resolved message');
});

promise.then(() => {
  console.log('I am in the first resolved promise');
}).then(() => {
  console.log('I am in the second resolved promise');
});

process.nextTick(() => {
  console.log('I am in the process next tick now');
});

fs.readFile('index.html', () => {
  console.log('==================');
  setTimeout(() => {
    console.log('I am in the callback from setTimeout with 0ms delay');
  }, 0);

  setImmediate(() => {
    console.log('I am from setImmediate callback');
  });
});

setTimeout(() => {
  console.log('I am in the callback from setTimeout with 0ms delay');
}, 0);

setImmediate(() => {
  console.log('I am from setImmediate callback');
});
{% endhighlight %}

The correct execution order is the following:

{% highlight sh linenos=table %}
> node event-loop.js
beginning of the program
I am in the promise function!
I am in the process next tick now
I am in the first resolved promise
I am in the second resolved promise
I am in the callback from setTimeout with 0ms delay
I am from setImmediate callback
==================
I am from setImmediate callback
I am in the callback from setTimeout with 0ms delay
{% endhighlight %}

You can read more about event loop and process.nextTick in [official Node.js docs](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#event-loop-explained).

## Event Emitters

A lot of core modules in Node.js emit or receive different events. Node.js has an implementation of [EventEmitter](https://nodejs.org/api/events.html#events_class_eventemitter), which is a [publish-subscribe pattern](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern). This is a very similar to browser DOM events with a little bit different syntax, and the easiest thing we can do to understand it fully is to actually implement it on our own:

{% highlight js linenos=table %}
class EventEmitter {
  constructor() {
    this.events = {};
  }
  checkExistence(event) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
  }
  on(event, cb) {
    this.checkExistence(event);

    this.events[event].push(cb);
  }
  off(event, cb) {
    this.checkExistence(event);

    this.events[event] = this.events[event].filter(
      registeredCallback => registeredCallback !== cb
    );
  }
  emit(event, ...args) {
    this.checkExistence(event);

    this.events[event].forEach(cb => cb(...args));
  }
}
{% endhighlight %}

This is all the basic code we need! It allows you to subscribe to events, unsubscribe later, and emit different events. For example, [response object](), [request object](), [stream]() – they all actually extend EventEmitter!

## Streams

> Streams are the most underappreciated feature of Node.js (Domenic Tarr)

Streams allow you to process data in chunks, not just as a whole object. To understand why do we need them, let me show you a simple example – let's say we want to

## Module System

Node.js uses [commonjs modules](https://nodejs.org/docs/latest/api/modules.html). Probably you already used it – every time you use `require` some module inside your webpack configuration, you actually use commonjs modules; every time you declare `module.exports` you use it as well – however, you might also seen something like `exports.some = {}`, without `module`, and in this section we'll see why is it so.

First of all, I'll talk about commonjs modules, with regular `.js` extensions, not about `.esm` (ECMAScript modules), which allow you to use `import/export` syntax. Also, what is important to understand, webpack and browserify (and other bundling tools) use their own require, so please don't be confused – we won't touch them here, just be aware that it is slightly different.

So, from where do we get these "global" objects like `module`, `require` and `exports`? Actually, Node.js runtime adds them – instead of just executing given javascript file, it actually wraps it in the function with all these variables:

{% highlight js linenos=table %}
function (require, module, exports) {
  // your code
}
{% endhighlight %}

## Environment Variables

As it is stated in [the twelve-factor app](https://12factor.net/), it is a good practice to store configuration in [environment variables](https://12factor.net/config). You can set up variables for the shell session:

{% highlight sh linenos=table %}
export MY_VARIABLE="some variable value"
{% endhighlight %}

You can add this line to your [bash/zsh profile](https://top.quora.com/What-is-bash_profile-and-what-is-its-use) so it is set up in any new terminal session.
However, you usually just run your application providing this variables specifically for this instance:

{% highlight sh linenos=table %}
APP_DB_URI="....." SECRET_KEY="secret key value" node server.js
{% endhighlight %}

You can access these variables in your Node.js app using [process.env](https://nodejs.org/api/process.html#process_process_env) object:

{% highlight js linenos=table %}
const CONFIG = {
  db: process.env.APP_DB_URI,
  secret: process.env.SECRET_KEY
}
{% endhighlight %}

## Putting Everything Together

In this example we will create a simple http server, which will return a file named as provided url string after `/`. In case file does not exist, we will return `404 Not Found` error, and in case user tries to cheat and use relative or nested path, we send him `403` error.

{% highlight js linenos=table %}
const { createServer} = require('http');
const fs = require('fs');

const server = createServer((req, res) => {
  if (req.pathname.startsWith('.')) {
    res.status = 403;
    res.send('Relative paths are not allowed');
  } else if (req.pathname.includes('/')) {
    res.status = 403;
    res.send('Nested paths are not allowed');
  } else {
    const fileStream = fs.createReadStream(req.pathname);

    res.pipe(fileStream);

    res.on('error', (e) => {
      if ()
    });
  }
});

server.listen(8080, () => {
  console.log('application is listening at port 8080');
});
{% endhighlight %}

## Conclusion

In this guide we covered a lot of fundamental Node.js principles. We did not dive into specific APIs, and we definitely missed some parts, but this guide should be a good starting point to feel confident while editing existing or creating new scripts – you are now able to understand errors, which interfaces built-in modules use, and what to expect from typical Node.js objects.

Next time we'll cover web servers using Node.js in depth, how to write a CLI application, and how to use Node.js for small scripts. Stay tuned!
