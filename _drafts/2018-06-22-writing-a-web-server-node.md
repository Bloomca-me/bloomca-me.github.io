---
layout: post
title: Web Server in Node.js from Scratch
keywords: node.js, node.js web server, node.js tutorial, web server without dependencies, web servers explained, bloomca, seva zaikov
---

Node.js is a pretty popular choice to build web servers, and has plenty of popular web frameworks, such as [express](https://expressjs.com/), [koa](https://koajs.com/), [hapijs](https://hapijs.com/) and so on. In this tutorial, though, we'll build a full-fledge server without any dependencies. While this might sound weird, it will actually help us to understand underlying technologies, and how they all are connected together, and why, for example, we need to connect certain middleware in our express applications.

We will implement a file server with support to download, upload and remove files. We'll list all uploaded files with links to download or remove, and a form to upload new files.

## Hello, world

First, let's start with the simplest thing possible – returning the famous "hello, world" response. In order to create a server in Node.js, we need to use [http](https://nodejs.org/api/http.html) built-in module, specifically [createServer](https://nodejs.org/api/http.html#http_http_createserver_options_requestlistener) function.

{% highlight js linenos=table %}
const { createServer } = require('http');

const PORT = process.env.PORT || 8080;

const server = createServer();

server.on('request', (req, res) => {
  res.end('Hello, world'!);
});

server.listen(PORT, () => {
  console.log(`starting server at port ${PORT}`);
});
{% endhighlight %}

Let's list everything we did in this short example:

1. Created a new server instance calling `createServer` function
2. Added an event listener on `request` event on our server
3. Ran our server using port specified in environment variables with fallback on `8080`

Our created server is an instance of [http.Server](https://nodejs.org/api/http.html#http_class_http_server) class, which inherits from [net.Server](https://nodejs.org/api/net.html#net_class_net_server), which is not so important for us, but it implements [EventEmitter](https://nodejs.org/api/events.html#events_class_eventemitter). The most important event for us is `request`, and it is so common that you can provide its listener when creating a server:

{% highlight js linenos=table %}
// this is equivalent to `server.on('request', fn);`
const server = createServer((req, res) => {
  res.end('Hello, world'!);
});
{% endhighlight %}

The last part is starting our server. We start it by calling [server.listen](https://nodejs.org/api/http.html#http_class_http_server) method, and you can specify port and which callback to call after server was started. This function is asynchronous, and after starting the server [listening](https://nodejs.org/api/net.html#net_event_listening) event is emitted and (if) specified callback is called.

Now, after we learned how to instantiate a new server, let's look how to actually reply to a user's request. We already implemented the simplest form, reply with canonic [Hello, world](https://en.wikipedia.org/wiki/%22Hello,_World!%22_program).

We have two parameters in handler function: [request]() and [response](). We'll omit request object for now, since we don't really want to react to user's data right now, and focus only on request. 

templates – just a function, we can use any (pug, etc), but for the sake of simplicity and given that we have only one page, we will just use strings in HTML.

2. routing mechanism

You might be familiar with express style routing, when we provide a string with url and a handler. In other frameworks/languages it might be decorators, annotations, or something else, but the idea is that we provide strings which will be matched against the requested URL. All of that is sugar, and in the beginning we have only full url, which you can access via property [req.url](). Frameworks add their functionality on top of that single string.

Let's add file downloading mechanism, with `/files/:file` routing. We don't have uploaded files yet, though, so let's create couple of files and put some content inside.

{% highlight sh linenos=table %}
touch 
echo "first file" >>> first.txt
{% endhighlight %}

3. POST requests (PUT/DELETE)

So far we have not really thought about [http methods]() – in fact, if we send a POST request using curl, we'll get the same response as before.

4. Headers (accept) – e.g. for authorization

Another helpful feature of [HTTP protocol]() is headers – additional information carried with requests, describing its nature. Headers are standartized, and we can control [caching](), [localization](), and ...
For example, it is common to separate versions of REST API using some headers.

Another use-case is to indicate which response a client is waiting for via [Accept header](). Let's build a 

5. Cookies

Cookies are also a feature of HTTP protocol, and they also, as headers, allow to carry some information with requests. The difference is that this information is going back and forth – browsers automatically attach it 

6. JSON in response

7. Multipart form

