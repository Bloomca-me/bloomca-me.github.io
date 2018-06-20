---
layout: post
title: Node.js Guide for Frontend Developers
keywords: javascript, node.js, nodejs, nodejs guide, tutorial, nodejs guide for frontend developers, nodejs introduction, node.js explanation, streams, event loop, commonjs tutorial, seva zaikov, bloomca, ES2015, ES6, async/await, modern javascript
---

> This guide is focused on frontend developers – people who know JavaScript, but are not very proficient with Node yet. I won't focus on the language here – Node.js uses V8, so it is the same interpreter as in Google Chrome, which you probably know about (however, it is possible to run on different VMs, see [node-chakracore](https://github.com/nodejs/node-chakracore)).

## Table of Contents

- [Node Version](#node-version)
- [No Babel is Needed](#no-babel-is-needed)
- [Callback Style](#callback-style)
- [Event Loop](#event-loop)
- [Event Emitters](#event-emitters)
- [Streams](#streams)
- [Module System](#module-system)
- [Environment Variables](#environment-variables)
- [Putting Everything Together](#putting-everything-together)

We deal with Node.js pretty often nowadays, even if you are a frontend developer – [npm scripts](https://docs.npmjs.com/misc/scripts), webpack configuration, gulp tasks, programmatic run of [bundlers](https://webpack.js.org/api/node/) or [test runners](https://karma-runner.github.io/2.0/dev/public-api.html). Even though you don't really need deep understanding for that sort of tasks, it might be confusing sometimes, and cause you to write something in a very weird way just because of missing some key concept of Node.js. Familiarity with Node can also allow you to automate some things you do manually, feeling more confident looking into server-side code and writing more complicated scripts.

## Node Version

The most schocking difference after client-side code is the fact that you decide on your runtime, and you can be absolutely sure in supported features – you choose which version you are going to use, depending on your requirements and available servers.

Node.js has a public [release schedule](https://github.com/nodejs/Release#release-schedule), which tells us that odd versions don't have long-term support, and that current LTS (long-term support) version will be active developed until April 2019, and then maintained with critical updates until December 31, 2019. New versions are being actively developed, and they bring a lot of new features, along with security updates and performance improvements, which might be a good reason to follow current active version. However, nobody really forces you, and if you don't want to do that – there is nothing wrong to use an old version, and to avoid updates until it is a good moment for you.

Node.js is used extensively in modern frontend toolchain – it is hard to imagine modern project which does not include any processing using node tools, so you might be already familiar with [nvm](https://github.com/creationix/nvm) (node version manager), which allows you to have several node versions at the project simultaneously. The reason for such a tool is that different project very often are written using different Node version, and you don't want to constantly keep them in sync, you just want to preserve environment in which they were written and tested.
Such tools exist for many other languages, like [virtualenv](https://virtualenv.pypa.io/en/stable/) for Python, [rbenv](https://github.com/rbenv/rbenv) for Ruby and so on.

## No Babel is Needed

Because you are free to choose any Node.js version, there is a big chance that you can use LTS (Long term supported) version, which is [8.11.3](https://nodejs.org/en/) at the moment of writing, which supports [almost everything](https://node.green/) – all 2015 ECMAScript spec, except for [tail recursion](https://en.wikipedia.org/wiki/Tail_call).

It means that there is no need for Babel, unless you are stuck with old version of Node.js, need [JSX transformation](https://babeljs.io/docs/en/babel-plugin-transform-react-jsx), or can't live without some [bleeding edge transformation](https://github.com/tc39/proposal-pipeline-operator). In practice, it is not that crucial, so your running code is the same as you write it, without any transpiling – gift we already forgot on the client-side.

There is also no need for webpack or browserify, and therefore we don't have a tool to reload our code – in case you develop something like a web server, you can use [nodemon](https://github.com/remy/nodemon) to reload your application after filechanges.

And because we don't ship this code anywhere, there is no need to minify it – one step less: you just use your code as is! Feels really weird.

## Callback Style

Historically, asynchronous functions in Node.js accepted callbacks with a signature `(err, data)`, where first argument represented an error – if it was `null`, all is good, otherwise you have to handle the error.
For example, let's read a file:

{% highlight js linenos=table %}
const fs = require('fs');

fs.readFile('myFile.js', (err, file) => {
  if (err) {
    console.error('There was an error reading file :(');
    // process is a global object in Node
    // https://nodejs.org/api/process.html#process_process_exit_code
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

This is a Node.js specific thing, it does not exist in any browser. It behaves like a microtask, but with a priority, which means that it will be executed right after all synchronous code, even if other microtasks were introduced before – this is dangerous. Naming is unfortunate, since it is not really correct, but because of [compatibility reasons](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#process-nexttick-vs-setimmediate) it probably will remain the same.

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
  once(event, cb) {
    this.checkExistence(event);

    const cbWithRemove = (...args) => {
      cb(...args);

      this.off(event, cbWithRemove);
    };

    this.events[event].push(cbWithRemove);
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

Because it is such a simple concept, it is implemented in a lot of npm packages: [1](https://www.npmjs.com/package/event-emitter), [2](https://www.npmjs.com/package/events), [3](https://www.npmjs.com/package/eventemitter3), and [many others](https://www.npmjs.com/search?q=event+emitter) – so, if you want to use the same event emitter in the browser, feel free to use them.

This is all the basic code we need! It allows you to subscribe to events, unsubscribe later, and emit different events. For example, [response object](https://nodejs.org/api/http.html#http_class_http_serverresponse), [request object](https://nodejs.org/api/http.html#http_class_http_incomingmessage), [stream](https://nodejs.org/api/stream.html) – they all actually extend or implement EventEmitter!

## Streams

<blockquote class="highlight-paragraph pull-in">
  <p>“Streams are Node's best and most misunderstood idea.”</p>
  <cite>
    Dominic Tarr
  </cite>
</blockquote>

Streams allow you to process data in chunks, not just as a whole object. To understand why do we need them, let me show you a simple example – let's say we want to return to a user a requested file of arbitrary size. Our code might look like this:

{% highlight js linenos=table %}
function (req, res) {
  const filename = req.query;
  fs.readFile(filename, (err, data) => {
    if (err) {
      res.status = 500;
      res.send('Something went wrong');
    }

    res.send(data);
  }); 
}
{% endhighlight %}

This code will work, especially locally, but it _might_ fail. See the issue?
We can have problems here in case file is too big – when we read a file, we put everything into memory, and if we don't have enough resources, this won't work. This also won't work in case we have a lot of concurrent requests – we have to keep `data` object in memory until we sent everything.

However, we don't really need this file at all – we just return it from file system, and we don't look inside the content by ourselves, so we can read some part, return it immediately, free our memory, and repeat the whole thing again until we are done. This is a description of streams in a nutshell – we have a mechanism to receive data in chunks, and _we_ decide what to do with this data. For example, we can do exactly the same:

{% highlight js linenos=table %}
function (req, res) {
  const filename = req.query;
  const filestream = fs.createReadStream(filename, { encoding: 'utf-8' });

  let result = '';

  filestream.on('data', chunk => {
    result += data;
  });

  readableStream.on('end', () => {
    res.send(result);
  });

  // if file does not exist, error callback will be called
  readableStream.on('error', () => {
    res.status = 500;
    res.send('Something went wrong');
  });
}
{% endhighlight %}

Here we create a stream to read from the file – this stream implements [EventEmitter](https://nodejs.org/api/events.html#events_class_eventemitter) class, and on `data` event we receive next chunk, and on `end` we get a signal that stream ended and full file was read. This implementation works the same as before – we wait for reading the whole file, and then return it in the response. Also, it has the same problem: we keep the whole file in our memory before sending it back. We can solve this problem if we know that response object implements a [writable stream](https://nodejs.org/api/http.html#http_class_http_serverresponse) itself, and we can write information into that stream without keeping it in our memory:

{% highlight js linenos=table %}
function (req, res) {
  const filename = req.query;
  const filestream = fs.createReadStream(filename, { encoding: 'utf-8' });

  filestream.on('data', chunk => {
    res.write(chunk);
  });

  readableStream.on('end', () => {
    res.end();
  });

  // if file does not exist, error callback will be called
  readableStream.on('error', () => {
    res.status = 500;
    res.send('Something went wrong');
  });
}
{% endhighlight %}

> Response object implements a [writable stream](https://nodejs.org/api/stream.html#stream_class_stream_writable), and [fs.createReadStream](https://nodejs.org/api/fs.html#fs_fs_createreadstream_path_options) creates a [readable stream](https://nodejs.org/api/stream.html#stream_readable_streams), and there are also [duplex and transform streams](https://nodejs.org/api/stream.html#stream_duplex_and_transform_streams). Difference between them and how they exactly work is out of the scope of this tutorial, but it is good to know about their existence

Now we don't need `result` variable anymore, we just write read chunks into response immediately, and we don't keep it in our memory! It means that we can read even big files, and we don't have to worry about a lot of parallel requests – since we don't keep files in our memory, we won't get out of it.
However, there is one problem – in our solution we read from one stream (filesystem reading from a file) and write it into another (network request), and these two things have [different latencies](https://gist.github.com/jboner/2841832). By different I mean _really_ different, and after some time our response stream will be overwhelmed, since it is much slower. This problem is a description of [backpressure](https://nodejs.org/en/docs/guides/backpressuring-in-streams/), and Node has a solution for it: each readable stream has a [pipe method](https://nodejs.org/api/stream.html#stream_readable_pipe_destination_options), which redirects all data into given stream respecting its load: if it is busy, it will [pause](https://nodejs.org/api/stream.html#stream_readable_pause) original stream, and [resume](https://nodejs.org/api/stream.html#stream_readable_resume) it. Using this method, we can simplify our code to:

{% highlight js linenos=table %}
function (req, res) {
  const filename = req.query;
  const filestream = fs.createReadStream(filename, { encoding: 'utf-8' });

  filestream.pipe(res);

  // if file does not exist, error callback will be called
  readableStream.on('error', () => {
    res.status = 500;
    res.send('Something went wrong');
  });
}
{% endhighlight %}

> Streams changed several times during Node's history, so be extra careful while reading old manuals, and try to compare with official documentation!

## Module System

Node.js uses [commonjs modules](https://nodejs.org/docs/latest/api/modules.html). Probably you already used it – every time you use `require` some module inside your webpack configuration, you actually use commonjs modules; every time you declare `module.exports` you use it as well – however, you might also seen something like `exports.some = {}`, without `module`, and in this section we'll see why is it so.

First of all, I'll talk about commonjs modules, with regular `.js` extensions, not about `.esm` (ECMAScript modules), which allow you to use `import/export` syntax. Also, what is important to understand, webpack and browserify (and other bundling tools) use their own require, so please don't be confused – we won't touch them here, just be aware that it is slightly different.

So, from where do we get these "global" objects like `module`, `require` and `exports`? Actually, Node.js runtime adds them – instead of just executing given javascript file, it [actually wraps](https://nodejs.org/api/modules.html#modules_the_module_wrapper) it in the function with all these variables:

{% highlight js linenos=table %}
function (exports, require, module, __filename, __dirname) {
  // your module
}
{% endhighlight %}

You can see this wrapper by executing the following snippet in your command-line:

{% highlight sh linenos=table %}
node -e "console.log(require('module').wrapper)"
{% endhighlight %}

These are variables injected into your module and available as "global" ones, even though they are not really global. I highly recommend you to look into them, especially into `module` variable – you can just call `console.log(module)` in a javascript file: try to compare results when you print from the "main" file, and then from a required one.

----

Next, let's look into `exports` object – we will have a small example showing some caveats related to it:

{% highlight js linenos=table %}
exports.name = 'our name'; // this works

exports = { name: 'our name' }; // this doesn't work

module.exports = { name: 'our name' }; // this works!
{% endhighlight %}

Example above might get you puzzled – why is it so? The answer is in the nature of `exports` object – it is just an argument passed to a function, so in case we assign a new object to it, we just rewrite this variable, and old reference is gone. It is not fully gone, though – `module.exports` is the same object, so they are actually the same reference to a single object:

{% highlight js linenos=table %}
module.exports === exports; // true
{% endhighlight %}

----

The last part is `require` – it is a function which takes a module name and returns the `exports` object of this module. How does it exactly resolve a module? There is a pretty straightforward rule:

- check core modules with provided name
- if path begins with `./` or `../`, try to resolve a file
- if there was no file found, try to find a directory with `index.js` file in it
- if path does not begin with `./` or `../`, go to the `node_modules/` and check folder/file there:
  - in the folder where we run script
  - one level above, until we reach `/node_modules`

There are other libraries, and you can also provide your path to look in by specifying variable [NODE_PATH](https://nodejs.org/api/modules.html#modules_loading_from_the_global_folders), which might be useful. If you want to see exact order in which `node_modules` are being resolved, just print `module` object in your script and look for `paths` variable – for me it prints the following:

{% highlight sh linenos=table %}
➜ tmp node test.js
Module {
  id: '.',
  exports: {},
  parent: null,
  filename: '/Users/seva.zaikov/tmp/test.js',
  loaded: false,
  children: [],
  paths:
   [ '/Users/seva.zaikov/tmp/node_modules',
     '/Users/seva.zaikov/node_modules',
     '/Users/node_modules',
     '/node_modules' ] }
{% endhighlight %}

Another interesting thing about `require` is that after first require module is cached, and won't be executed again, we'll just get back cached `exports` object – so it means that you can do some logic and be sure that it will be called only once after first `require` call (this is not exactly true – you can remove module id from [require.cache](https://nodejs.org/docs/latest/api/modules.html#modules_require_cache), and then module might be reloaded, if it is required again).

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

In this example we will create a simple [http](https://nodejs.org/api/http.html#http_http) server, which will return a file named as provided url string after `/`. In case file does not exist, we will return `404 Not Found` error, and in case user tries to cheat and use relative or nested path, we send him `403` error.

{% highlight js linenos=table %}
const { createServer} = require('http');
const fs = require('fs');

const server = createServer((req, res) => {
  if (req.pathname.startsWith('.')) {
    res.status = 403;
    res.send('Relative paths are not allowed');
    res.end();
  } else if (req.pathname.includes('/')) {
    res.status = 403;
    res.send('Nested paths are not allowed');
    res.end();
  } else {
    const fileStream = fs.createReadStream(req.pathname);

    res.pipe(fileStream);

    res.on('error', (e) => {
      // you can get all common codes from docs:
      // https://nodejs.org/api/errors.html#errors_common_system_errors
      if (e.code === 'ENOENT') {
        res.status = 404;
        res.send('This file does not exist.');
        res.end();
      }
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
