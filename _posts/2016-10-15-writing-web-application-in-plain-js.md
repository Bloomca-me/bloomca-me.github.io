---
layout: post
title: Creating web application in plain javascript
keywords: vanilla.js, javascript, SPA, seva zaikov, bloomca, web application, frameworks, software development
excerpt: Modern JS frameworks provide all sorts of start help, along with really high-level concepts out of the box. Let's try to re-create them to understand what stands behind each app.
---

## JS fatigue

> any application that can be written in JavaScript, will eventually be written in JavaScript

This quote is not a joke, and JS community is growing in [the outstanding pace](http://alexandros.resin.io/npm-now-the-largest-module-repository/) (evil voices say that's because of publishing very interesting [repositories](https://www.npmjs.com/package/is-negative), but we all know that not only because of that), and people very often are frustrated. Requirements for modern applications are incredible, and for newbies it is too overwhelming. Also, there is quite vibrant and unstable tooling development process, which is very easy tracked by branches like `alpha-0.0.5rc` and so on. And react arise (and other frameworks happily joined soon) the whole problem of boilerplates – people create them trying to get the fanciest setup (the most popular react boilerplate has adapter for `graphql` server by default – which is complete nonsence, because 99.99% servers don't use it), and they still do, without even some understanding how the whole system works, and in case of problems they just start to copy & past snippets (so, we can say that `jQuery-plugin` approach moved to the configuration).

## Starting from scratch

But I don't really want to explain build systems – this is another concern, I want to warn everyone in a more deep way than just reconsider way how you start your project and build it around something – but think a bit, maybe your framework of choice is actually not a result of thorough decision and is a part of a bigger fatigue? I think it is really crucial to be able to recreate the basic application in plain javascript – otherwise, it is too easy to get a lot of things for granted, and to be actually locked to your framework, which you `know`. Framework, as any library (and language as well), is just a tool – and you should be able to pick a new one, more suitable; but it is impossible without understanding how such tools are created in general. Don't get me wrong – I won't try to explain how all libraries work (at least here), I will just show how to create fully functional application.
So, I'll build literally the _simplest_ application, which will fetch songs from [Itunes](https://affiliate.itunes.apple.com/resources/documentation/itunes-store-web-service-search-api/) and display them according to the search query. The application code is possible to look at [here](https://github.com/Bloomca/vanilla-web-app) and if you prefer to read source code, just start [here](https://github.com/Bloomca/vanilla-web-app/blob/master/src/index.js).

## API

We will start with API. You can call it data layer, models, state, anything else – it doesn't matter. Here we define domain level and _adapter_ for it – to get the data in the format which our application will know 100%. Sometimes adapter is not needed, that is true – when you have a stable API and 100% sure that requirements won't ask you to support some other option, like different API. With adapter we get independence from our exact API, because here, for instance, we can replace Itunes API with SoundCloud, and still be able to render them using the same code – we just have to add something like `parseSongFromSoundCloud`. So, never lock yourself into very specific and fragile things, don't be afraid to add additional processing step. Yeah, it can hurt performance, but only if you receive thousands of results and, also, never do premature optimizations – you'll have time for it later.
Also, fetching _methods_ and parsing data should be separated in 100% situations – such tangle will bait you later.

{% highlight javascript %}
function parseSong(song) {
  if (song) {
    return {
      title: song.trackName,
      artist: song.artistName,
      picture: song.artworkUrl100
    }
  }
}
{% endhighlight %}
> with such parsing we can easily add songs from different endpoints as well

I want to point out that there is no state management here. This is intentionally to reduce complexity, but if you want to, you can add a wrapper around any `fetch` call, and implement some sort of state between, with caching and other stuff, what do you need. We are not really restricted even from [Redux](https://github.com/reactjs/redux) here, or from [Baobab](https://github.com/Yomguithereal/baobab). Any caching, though, will add a lot of complexity, and it might be a true overhead, so use it only if you really need it. Redux is the de-facto standard now in react-based application, but sometimes you don't need it, and you can safely use internal react state management (I know, I know, what you'll say to me. The thing is that you do not _always_ need it – sometimes app is too small and it doesn't worth it to introduce big state management library).

## Views

After this we can create views for our application. Usually, in more mature solutions, it is some type of classes/factories, but here we will have only string primitives, because we are not very interested in a lifecycle.

{% highlight javascript %}
export default `
  <div class="loader">
    <div class="spinner"></div>
  </div>
`
{% endhighlight %}

Here we have only markup, but with more full solution we will have some kind of class/function, which will return events, dispose function along with markup. Views are close to classical objects – data with methods; they are different depending on the framework of choice, but in general they all allow you to create some markup with attached events, and to invoke methods during lifecycle. For views rendering I'll use the simpliest function ever:

{% highlight javascript %}
export function render({ markup, el }) {
  // this is syncronous rendering, so hooks would be:
  // beforeMount(component)
  el.innerHTML = markup
  // afterMount(component)

  // but some frameworks can do it asyncronously,
  // and invoke this stuff in different time
}
{% endhighlight %}

This is sufficient for us here – we just add possibility to render it into the real DOM. We can build a template system, which would be injected into the real HTML in place of these templates (for this task we would have to parse HTML and look what do we have to do – such approach is used by `Angular` or `Knockout`, for instance), or just keep references to the element, like in `Backbone`, and attach all events to container and listen to them through delegation. A lot of possibilities are here, but they are basically the same idea – render some stuff and add functions to work with this rendered stuff.

## App

The application itself is just a loop that invokes again and again, rendering new stuff according to reactions. A loop can be recursive (e.g. internal parts can wait some changes by themselves and re-render them) or just a top-level changing. The loop can be invoked by interval and check changes, or by some event listening. I think pub/sub here is a bit more elegant, but both versions have their right to exist. So, what we do – we render the initial layout of the application, and then we create this loop manually by listening to the search input:

{% highlight javascript %}
// we render our initial application
const layoutContainer = document.getElementById('app')
render({ markup: layout, el: layoutContainer })

const appContainer = document.getElementById('content') || layoutContainer

// and then we start to listen to changes
const searchInput = document.getElementById('search')
searchInput.addEventListener('keydown', (e) => {
  const newValue = e.target.value

  // render loader after new request
  render({ markup: loader, el: appContainer })

  // here we can get race conditions, but it is not very
  // important right now
  fetchSongs(newValue).then(songs => {
    // render results
    render({ markup: createSongs(songs), el: appContainer })
  })
})
{% endhighlight %}

In real-world application this loop will be internal – for instance, event will be attached during constructing of the layout `component`, and we will just render initial application. Sometimes, like in React, some [approaches](https://github.com/omniscientjs/omniscient) suggest to re-render everything, which is not so far from our approach – except that react provides intellectual granular updates. Of course, we don't have any intelligent update system here – but this is not the point, because the general idea is quite close.

## Routing

There is nothing special in routing at all, but this part is traditionally considered as _magical_. All that we need – somehow react to link clicks, and if it is internal redirect, we have to handle it and react as an application. So, you can think about router as a higher part of the application which has events of clicked links and which renders corresponding markup. There are two approaches to handle clicks – first one is just to add global listeners to all clicks and then to filter clicks on links with internal urls, or just provide link component by your library, which will have automatic listener. Both approaches has good and bad sides, as usual. Programmatic url changes are done through [history API](https://developer.mozilla.org/en-US/docs/Web/API/History_API), there by `pushState` method you can easily add new entries and history. This method usually wraps in other Router library's method and we emit needed event to react correspondingly. Paths have usual format in almost all libraries, and are just parsed by regular expressions to extract all needed params.

## Wrap-up

Basically, these components are enough to build the application – data layer, application loop, views and router. Naming can be different, but what you should look at here – only functionality, they are true building blocks of application. I don't really want anyone to build applications from scratch, without using anything; it is not a crusade against popular frameworks, I just think that it is important to understand underlying principles in a very high level overview, without distracting to details, how exactly `$.digest` is implemented in Angular, or `reconciliation` mechanism in React. They work, but their idea is just to support this loop and to update your application efficiently.
