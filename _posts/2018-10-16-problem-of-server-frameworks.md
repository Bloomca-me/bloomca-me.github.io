---
layout: post
title: Problem of Server Frameworks
keywords: server views, server frameworks, server side views, server frameworks problem, mvc problem, server views problem, software development, spa, javascript, seva zaikov, bloomca
---

Doing a lot of frontend work using modern frameworks for single-page applications and some with backend in [Node.js](https://nodejs.org), [Python](https://www.python.org/) and [Go](https://golang.org/), I came to realization that traditional approaches (for example, [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller), or just rendering templates with provided data) are not very good for later refactoring or experiments with views. I have to specify that I am talking about rendering HTML here mostly and the way we organize it in our code on the backend.

## Traditional Rendering

By traditional model I mean approaches when you store your views (which will end up as HTML we serve to clients) in a separate language (like [Jade/pug](https://github.com/pugjs/pug#rename-from-jade), [haml](http://haml.info/), [erb](https://puppet.com/docs/puppet/5.0/lang_template_erb.html), etc) with some variables used inside. We populate them during _rendering_, when we call a function to translate our view templates into final HTML – this invocation might be explicit or implicit, depending on your framework of choice. Also, these variables usually represent some dynamic data (otherwise we'd be fine with static content), and we need to instantiate this data before. In MVC model, controller is responsible for that, and we need to know which data we'll need for the corresponding template.

When we write our initial version, all works just fine – we have separation of concerns, logic to retrieve data is described in models, and inside our controllers we just instantiate them, and we are good to refactor all the network\db layer later. However, views usually have very limited access to the controller, and using anything except regular properties or iterating is not considered as a good practice. Also, in some languages, like JavaScript, asynchronous nature of network calls makes it impossible (technically speaking, it depends on VM implementation) to execute them during rendering.

## Making Changes

At some point, though, you'll want to remove _that one_ section, and all of the sudden you don't need _this_ model anymore. Or do you? In the beginning it is pretty easy to decide, but over time it becomes more and more challenging to keep all the dependencies in your head. The problem grows in the other direction as well – why do you need this model here (or helpers, however you call it)? Why do you need to extend that controller? It is really hard to say where exactly we use these properties, since templates, being just a single file per view in the beginning, tend to be fragmented later. In other words, server views are not very composable – they depend on some data, and we _have to_ know about it in other places.

At first SPA came probably more as an attempt to decompose API into microservices, and client-side templating was a very cumbersome process in the beginning (remember [Backbone](http://backbonejs.org/#View-template)?), but now there is a huge advantage over server rendering – [components](https://reactjs.org/docs/components-and-props.html) everywhere! Everything is separated, and with lifecycle hooks you can make them independent, which allows you to just reshuffle your page (removing something or adding) in the markdown, and that's it – everything else will be taken care of, no need for "cleanup" or adding new functionality.
I am [not a big fan of SPAs everywhere](https://blog.bloomca.me/2018/02/04/spa-is-not-silver-bullet.html) by myself: they still have a lot of problems on its own – big bundle size, long loading time, possibly SEO problems, but from my point of view, this is a great advantage, which increases speed of development and allowes much greater flexibility.

**p.s.** As I said, I have a limited experience with server frameworks. So if I am wrong, please tell me which frameworks allow to do that and how you organize your components!
