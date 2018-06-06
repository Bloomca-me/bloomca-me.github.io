---
layout: post
title: Node.js Guide for Frontend Developers
keywords: javascript, node.js, nodejs, nodejs guide, tutorial, seva zaikov, bloomca, ES2015, ES6, async/await, modern javascript
---

I won't focus on the language here – Node.js uses V8, so it is the same interpreter as in Google Chrome, which you probably know about (however, it is possible to run on different VM, see [node-chakracore](https://github.com/nodejs/node-chakracore)).

## Node version

The most schocking difference after client-side code is the fact that you decide on your runtime, and you can be absolutely sure in supported features – you choose which version you are going to use, depending on your requirements and available servers.

Node.js has a public [release schedule](https://github.com/nodejs/Release#release-schedule), which tells us that odd versions don't have long-term support, 

Node.js is used extensively in modern frontend toolchain – it is hard to imagine modern project which does not include any processing using node tools, so you might be already familiar with [nvm](https://github.com/creationix/nvm) (node version manager), which allows you to have several node versions at the project simultaneously. The reason for such a tool is that different project very often are written using different Node version, and you don't want to constantly keep them in sync, you just want to preserve environment in which they were written and tested.
Such tools exist for many other languages, like [virtualenv](https://virtualenv.pypa.io/en/stable/) for Python, [rbenv](https://github.com/rbenv/rbenv) for Ruby and so on.

## No Babel

Because you are free to choose any Node.js version, there is a big chance that you can use LTS (Long term supported) version, which is [8.11.2](https://nodejs.org/en/) at the moment of writing, which supports [almost everything](https://node.green/) – all 2015 ECMAScript spec, except for [tail recursion](https://en.wikipedia.org/wiki/Tail_call).

It means that there is no need for Babel, unless you are stuck with old version of Node.js, or can't live without some bleeding edge transformation. In practice, it is not that crucial, so your running code is the same as you write it, without any transpiling – gift we already forgot on the client-side.

There is also no need for webpack or browserify, and therefore we don't have a tool to reload our code – in case you develop something like a web server, you can use [nodemon](https://github.com/remy/nodemon) to reload your application after filechanges.

Also, because we don't ship this code anywhere, there is no need to minify it – one step less: you just use your code as is! Feels really weird.

## Callback style

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

Event loop is almost the same, but it is

process.nextTick()

setImmediate()

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

## Module System

Node.js uses [commonjs modules](https://nodejs.org/docs/latest/api/modules.html). Probably you already used it – for example, if you defined a webpack configuration file, you actually used 
