---
layout: post
title: Node.js REPL in Depth
keywords: node.js, node, REPL, node.js REPL, node.js REPL tutorial, node.js REPL in depth, javascript, seva zaikov, bloomca
---

[REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop) stands for read-eval-print-loop, or just an interactive session (usually in your terminal), where you can enter some expression and immediately evaluate it, seeing the result. After evaluating, the whole flow repeats, and it works until you exit the process. So, `R` stands for reading your command, `E` stands for evaluating it, `P` stands for printing the result of the execution, and `L` means to run the whole process again, "in the loop".

In order to run Node.js REPL, you need to run the `node` command without any arguments:

{% highlight sh linenos=table %}
> node
{% endhighlight %}

REPL session is similar to the console window in the browser – you can create and store variables, declare functions, require modules and installed dependencies (check [module object](https://blog.bloomca.me/2018/06/21/nodejs-guide-for-frontend-developers.html#module-system during your REPL session)). However, one important difference between executing a file and working in a REPL session is that a REPL session is not wrapped into the function, so you don't have [__dirname](https://nodejs.org/docs/latest/api/modules.html#modules_dirname) and [__filename](https://nodejs.org/docs/latest/api/modules.html#modules_filename) variables. So, you can require any files as you would do in any existing commonjs file. To illustrate, let's create a new file where we export some string, and require it from the REPL:

{% highlight js linenos=table %}
> touch example.js
> echo "exports.some = 'some text';" >> example.js
> node
> require('./example');
{ some: 'some text' }
> _.some
'some text'
{% endhighlight %}

I did not assign the result of the `require` expression, but I was able to access it anyway! I got the last expression's result by using [underscore variable](https://nodejs.org/api/repl.html#repl_assignment_of_the_underscore_variable) – Node.js REPL automatically assigns the last expression's result to `_` variable. It is interesting, that despite being a popular choice for [lodash](https://lodash.com/), you can't assign to this variable. It will save the value, but after the next expression it will be different:

{% highlight js linenos=table %}
> node
> _ = 'something'
Expression assignment to _ now disabled.
'something'
> _
'something'
> _ = 'something else'
'something else'
> _
'something else'
{% endhighlight %}

## Special commands

Node.js REPL has special commands, even though I feel they are not so widely known. First, let's look at the multiline command. By default, every line is evaluated after you type it and hit "Enter": there is no way in JavaScript to tell when we need to continue or can evaluate our code (like it is done in [Python REPL](https://docs.python.org/3/tutorial/interpreter.html#interactive-mode), for example). Because of that it is challenging to write functions, since it is very unreadable and very error-prone to type all in one line. However, there is an [.editor](https://nodejs.org/api/repl.html#repl_commands_and_special_keys) command, which allows you to type as many lines as you need:

{% highlight js linenos=table %}
> node
> .editor
// Entering editor mode (^D to finish, ^C to cancel)
function sum(a, b) {
  return a + b;
}

sum(5, 4);

// I type ^D at this point

9

> sum(10, 6)
16
{% endhighlight %}

Other special commands (from the [official docs](https://nodejs.org/api/repl.html#repl_commands_and_special_keys)):

- `.break` - When in the process of inputting a multi-line expression, entering the .break command (or pressing the `<ctrl>-C` key combination) will abort further input or processing of that expression.
- `.clear` - Resets the REPL context to an empty object and clears any multi-line expression currently being input.
- `.exit` - Close the I/O stream, causing the REPL to exit.
- `.help` - Show this list of special commands.
- `.save` - Save the current REPL session to a file: > `.save ./file/to/save.js`
- `.load` - Load a file into the current REPL session. > `.load ./file/to/load.js`
- `.editor` - Enter editor mode (`<ctrl>-D` to finish, `<ctrl>-C` to cancel).

## Automatically imported modules

If you hit the TAB key twice, it will give you all possible autocompletions (also, if you already typed something, and there is only one option, it will do it automatically). We can use it to check what is available globally (keep in mind that everything you declare in your session, is also available there).

{% highlight sh linenos=table %}
> node
// I tap TAB twice
Array                         Boolean                       Date                          Error                         EvalError                     Function                      Infinity
JSON                          Math                          NaN                           Number                        Object                        RangeError                    ReferenceError
RegExp                        String                        SyntaxError                   TypeError                     URIError                      decodeURI                     decodeURIComponent
encodeURI                     encodeURIComponent            eval                          isFinite                      isNaN                         parseFloat                    parseInt
undefined

ArrayBuffer                   Atomics                       Buffer                        DTRACE_HTTP_CLIENT_REQUEST    DTRACE_HTTP_CLIENT_RESPONSE   DTRACE_HTTP_SERVER_REQUEST    DTRACE_HTTP_SERVER_RESPONSE
DTRACE_NET_SERVER_CONNECTION  DTRACE_NET_STREAM_END         DataView                      Float32Array                  Float64Array                  GLOBAL                        Int16Array
Int32Array                    Int8Array                     Intl                          Map                           Promise                       Proxy                         Reflect
Set                           SharedArrayBuffer             Symbol                        Uint16Array                   Uint32Array                   Uint8Array                    Uint8ClampedArray
WeakMap                       WeakSet                       WebAssembly                   _                             assert                        async_hooks                   buffer
child_process                 clearImmediate                clearInterval                 clearTimeout                  cluster                       console                       crypto
dgram                         dns                           domain                        escape                        events                        fs                            global
http                          http2                         https                         module                        net                           os                            path
perf_hooks                    process                       punycode                      querystring                   readline                      repl                          require
root                          setImmediate                  setInterval                   setTimeout                    stream                        string_decoder                tls
tty                           unescape                      url                           util                          v8                            vm                            zlib

__defineGetter__              __defineSetter__              __lookupGetter__              __lookupSetter__              __proto__                     constructor                   hasOwnProperty
isPrototypeOf                 propertyIsEnumerable          toLocaleString                toString                      valueOf
{% endhighlight %}

What is important here, aside from the expected global objects and methods, is that you automatically have access to [core Node.js modules](https://nodejs.org/api/repl.html#repl_accessing_core_node_js_modules), like [fs](https://nodejs.org/dist/latest-v10.x/docs/api/fs.html), [http](https://nodejs.org/dist/latest-v10.x/docs/api/http.html), [os](https://nodejs.org/dist/latest-v10.x/docs/api/os.html), [path](https://nodejs.org/dist/latest-v10.x/docs/api/path.html) and so on. While it is not a big deal, it might be a very convienent thing, especially when you are trying out some ideas in your REPL.

You can use these imported modules in a special way of running Node.js: if you provide a `-e` argument with a following string, containing JS code, it will be evaluated immediately:

{% highlight sh linenos=table %}
> node -e "console.log(os.platform())"
darwin
{% endhighlight %}

Keep in mind, that in this mode node does not print anything in the console by default, and you would have to wrap your code in `console.log` if you want to see some result. This execution might be helpful, if you don't want to enter REPL, and don't want to create a separate file for it.

## Using repl module

You can import the [repl](https://nodejs.org/api/repl.html#repl_repl) module directly and use it within your Node.js application. All Node.js does by default can be done using the [repl.start](https://nodejs.org/api/repl.html#repl_repl_start_options) method. It will use all the usual defaults, standard input and output streams. To run an interactive shell with our own prompt symbol, we have to create a small javascript file:

{% highlight js linenos=table %}
const repl = require('repl');

// Our own prompt
repl.start('Code:: ');
{% endhighlight %}

Now we can run this file, and it will work as before, but with the custom prompt:

{% highlight sh linenos=table %}
> node repl.js
Code:: const sum = (...args) => args.reduce((a, b) => a + b, 0)
undefined
Code:: sum(1, 2, 3)
6
Code:: .exit
{% endhighlight %}

As you can see, all commands work as expected. You can cusomize plently of things, so feel free to play around with your own REPL session! For example, you can add your own variables to the scope of REPL, so you don't need to require them:

{% highlight js linenos=table %}
const repl = require('repl');

const r = repl.start('Code:: ');
r.context.enrich = (obj) => {
  return Object.assign(obj, { prop: 'something' });
}
{% endhighlight %}

Now we will have this function in our scope available immediately:

{% highlight sh linenos=table %}
> node repl.js
Code:: enrich({ num: 123 })
{ num: 123, prop: 'something' }
{% endhighlight %}

- [Full REPL documentation](https://nodejs.org/api/repl.html).
