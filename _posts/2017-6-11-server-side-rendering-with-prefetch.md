---
layout: post
title: Server Side Rendering with Prefetch
keywords: react.js, react, redux, server-side rendering, seva zaikov, bloomca, prefetch, javascript
excerpt: Detailed explanation of what server side rendering is about, how is it done and how can you prefetch all the data, so you send fully formed page to the client.
---

## What is a server-side rendering

Server-side rendering (I'll use SSR later for the sake of brevity) is a pretty recent term, it started its life just couple of years ago. Initially the main problem was lack of SEO for complex single-page applications, and projects like [prerenderer](https://prerender.io/) appeared. The main idea of it was pre-render of the application somewhere else (e.g. PhantomJS), with waiting of the execution of the javascript, and then just grabbing all the html, and serving it later for robots. With this approach, you updated your pages only sometimes (maybe in the background), and they were served only for search engines, so direct users were not impacted at all.

After react came out, with one of the principles that whatever is rendered inside it, should be able to be processed on the server side (or whatever side actually, later it allowed to write react-native and react-iot renderers), it became clear that it is possible to render all the content on the node.js, and then just pass directly to the user. Later, other frameworks picked this idea (for example, Vue.js has pretty similar concepts of rendering), so nowadays SSR is considered a good practice, rather than rocket science (as it kind of was 2 years ago).

## Why do we need it?

This is actually a pretty good question -- crawlers are [much better now](https://webmasters.googleblog.com/2014/05/understanding-web-pages-better.html), so your content should be parsed anyway, computers became much faster (as well as mobiles, so nobody really makes dedicated mobile websites nowadays, except for some rare and specific cases). Also, there still aren't any really good library solutions -- you would likely have to wire some things by yourself (or use some boilerplate, but it might bring you some other kinds of problems). Also, [official starter kit](https://github.com/facebookincubator/create-react-app#limitations) from Facebook does not contain server-side rendering. So, why all the fuss?

Well, crawlers are actually [not so good](https://www.stephanboyer.com/post/122/does-google-execute-javascript) – and it is google! Maybe other engines are even worse, so it is better to stay safe and pre-render all your content. Also, parsing javascript on mobiles [is pretty slow](https://aerotwist.com/blog/the-cost-of-frameworks/), so if delegate all the work of sending javascript, then parsing, then executing, and then, after several possible AJAX requests (which can fail over sloppy 3G network), we will render the content on mobile. Instead, we can do the following:
- make all requests on the server (which has perfect bandwidth)
- perform initial render on the server
- send plain html before javascript
- send javascript (so the full cycle, which we descibed before, will go)

So, the user will get html first (before any app javascript), and only after we will start to parse and execute javascript. The application will be unresponsive until that point, and this is an obvious downside (which can be worked around to some extent, remembering all the user interactions), but still, the ability to see the important content fast is priceless. For desktops it is not that important, because CPUs are much faster there usually, but it is a nice addition.

Do you need server-side rendering or not? If you need SEO, then it is definitely a big bonus. But if you need only SEO, then take a look at [webpack-prerenderer-plugin](https://github.com/chrisvfritz/prerender-spa-plugin). If performance and a critical rendering path is crucial, then I would recommend taking a look at the amount of users from mobile devices, and to try and analyze which networks your users use your application from, and if they are low-end devices with possible usage of mobile networks, then I would definitely recommend that you give it a try. Also, one of the best test scenarious is to actually get a real android phone and use your application for a while – you will discover some interesting things, which you never thought of before!

## Solutions 

As I mentioned before, this area is still uncharted, so there is no real best practice. Boilerplates, if they provide any server-side rendering, usually go as far as just rendering your application without any prefetching (and without offering any ways to do so). So, you are basically on your own after you step into the server code part, which invokes something like `renderToString`. Starting from here, I'll assume we are talking about React and [react-router](https://github.com/ReactTraining/react-router) – while it is possible to achieve in, let's say, [vue](https://vuejs.org/v2/guide/ssr.html), I am more familiar with React, and also, approaches will be the same (as you know, any good idea in the front-end world will be immediately copied, as what happened with virtual DOM).

## Attaching function to top-level components

So, at the moment of rendering, when we actually invoke `renderToString`, we know which component we are going to render, and this leads us to the first guess. If we know the component, why can't we just attach a function to resolve all the needed data? With the router, we know exactly which routes we are going to receive, so we can add the needed query functions to it. Moreover, this is an official recommendation on both [react-router](https://reacttraining.com/react-router/web/guigrdes/server-rendering/data-loading) and [vue](https://ssr.vuejs.org/en/data.html#server-data-fetching).

The sequence is as follows:
- we match all the suitable components (they will be top-level components, defined in our router)
- we map over them, filtering by whether or not they have this function to fetch data
- we invoke all these functions, waiting for their execution
- we stringify the state and send it to the client
- on the client, we restore the state, so the same requests won't be fired.

> Please note, that the same requests will _try_ to be invoked on the client, so without some caching mechanism it is pretty useless – sent html will be rendered with data, but then the client will re-fetch the data, so the user will see blinking content, the loader and then the same data again.

I don't really want to provide any code, because react-router has changed it's API quite a bit, but a lot of projects still use an older version ([for example](https://github.com/erikras/react-redux-universal-hot-example/blob/master/src/server.js#L84)), but the idea should be clear.

## Problems with this approach

The biggest downside is the coupling – your top-level component should know too much – how to get all the needed data for this route, and while it makes sense in the beginning, as your application grows, it will become more and more complicated. Also, it is possible that some component will stop being rendered inside another component, but on the top-level, you will continue to fetch this data – it is possible that the person who will change this nested component, won't be aware of this exact prefetch.

Also, you will essentially duplicate your code – you will need to prefetch it inside other components (but on the client, more likely, you will fetch them inside nested components), and because of this multi-file nature, it is really easy to make them out-of-sync (completely accidentally!).

It is possible to work around those, though – basically, we can resolve other components inside these functions, and it makes problem of fetching unnecessary data much easier. Another problem arises, though – now we have to pass the correct props to these nested components, but it is a trade-off we have to make; and also, it won't solve the code duplication problem. But lack of any common solution to this problem shows that it is not that easy to generalize, and people stick with writing custom functions for specific routes.

## Double rendering

When we render our react application on the server, one lifecycle hook is invoked [componentWillMount](https://facebook.github.io/react/docs/react-component.html#componentwillmount) (or we can use `constructor`, what matters is that it is invoked on both the client and server).

> Please note that because `componentWillMount` is invoked on the server, all the requests that you fire in it will be performed, so even if you don't wait for anything, they will still be executed. So, as a general rule, better put such requests in `componentDidMount`.

The idea is the following: if we render our application first, without saving the rendered output, all `componentWillMount` hooks will be executed, and if we somehow catch all of them and wait, after resolving, we can safely render the application again, but now it will be fullfilled with data. The approach is much simpler, because we just execute the application several times. It _almost_ solves both problems of the previous approach – there is no coupling at all, and also the same `componentWillMount` will be invoked on the client side – so our content 

The biggest drawback is because we don't specify all requests in a single place, we get them from these hooks, but only from components which were rendered, so if some components were not presented because of missing data, it means that they won't prefetch their data. In general, abstracting from any coupling, we don't know how many times we'd have to render our application before everything would be prefetched, but this is another trade-off we have to make (usually 1 dry render is enough, but it depends on your application).
Double rendering (or triple), of course, is a drawback too, but it is just more CPU, and not some conceptual problem in our code, so I don't count it as a big problem.

## Double rendering implementation

I am more experienced with this approach, and it can be generalized much better than the previous one. In my library, [redux-tiles](https://github.com/Bloomca/redux-tiles), I provide exactly this approach to do [prefetch for server-side rendering](https://github.com/Bloomca/redux-tiles#user-content-server-side-rendering). The main idea is that we would like to catch all asynchronous requests and then wait for all of them, and only after this render the final output. We can create some object to store requests, and then pass it to all the async actions, but it is pretty annoying, and also requires us to rewrite all the data layer part. In the react + redux application, we have two places where we can instantiate independent per-request objects: [context](https://facebook.github.io/react/docs/context.html) and [middleware](http://redux.js.org/docs/advanced/Middleware.html). The first approach will require us to pass this object to actions (and syntax to get context is not the most convenient), so we are ending up with just one possibility – middleware. Middleware in redux allows us to handle different dispatched action types – in [redux-tiles](https://github.com/Bloomca/redux-tiles#user-content-middleware) middleware handles returned functions. It passes object with promises, so asynchronous "tiles" (think about them as redux modules) keep their requests in it (and remove them after resolving), and it allows us to grab all the active promises and then wait for them.

## Conclusion

As you can see, there are more questions about than answers to this topic; so if you feel that server-side rendering with prefetching is critical for you, you'd have to actually try different approaches by yourself – maybe your route components can easily serve as a smart component which will know everything about the underlying page, maybe it will make more sense to use the latter approach; you will have to weigh the tradeoffs for yourself.
