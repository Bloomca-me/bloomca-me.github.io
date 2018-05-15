---
layout: post
title: Single Page Application Is Not a Silver Bullet
keywords: javascript, SPA, application, react, angular, native application, server-side application, responsiveness
excerpt: SPA is a tool – use it wisely. Recent trend is to use it everywhere, but is it worth it to be able to write your landing page using hottest JS framework?
---

Single-page applications are everywhere. Even blogs, simple html pages (in the beginning something like [https://danluu.com/](https://danluu.com/)), have turned into big fat monsters – for example, [jlongster's blog](https://github.com/jlongster/blog) has 1206 stars at the moment of writing, and not because of the content (people usually subscribe to blogs rather than star the source): the only reason is that once he [implemented it using React & Redux](https://jlongster.com/The-Seasonal-Blog-Redux). What is the problem, though? He wants, he makes it, no questions here. The problem is that it is considered _normal_ to make blogs for reading so bloated – of course, there are some people who complain, but the general response is utterly positive. But who cares about blogs – the problem is that nowadays pretty often question is not "to SPA or not to SPA", rather "which client-side framework should we use for our SPA".

I am a frontend developer, who all his career was involved into creating quite complicated single-page applications (like [Lebara Play](https://play.lebara.com), or [app.contentful.com](https://app.contentful.com/) – you have to have an account to see the actual app, though) – and for a long time the question "should be a single-page application" did not exist for me: of course it should!
But some time ago, at my previous job, my manager came to me asking to investigate our application size – about two years ago we migrated from WordPress to React/Redux/Immutable.js stack (which is a pretty common choice, I guess), and it turned out that average load time increased twice over this time! Alright, maybe it was just our problem, and we are not that good – so I was looking into it for a couple of weeks, and after my research I re-thought a lot in front-end development.

## State of the Art

Nowadays a lot of startups, cool companies and personal websites are made using single-page applications – it is pretty common, and nobody is particularly surprised when they don't see a reaload after clicking on a link inside the application.

Let's summarize the experience, starting with the **pros**:

- no page reloads after clicking on the link
- with implemented caching subsequent pages open faster, because a lot of info can be already downloaded (like user info, movie details, etc)
- granular actions are super fast – like add to the favourites, subscribe to the news; of course, it is not rocket science, so you can sprinkle jQuery here and there, but it is much more complicated to maintain
- frontend is decoupled from backend, makes development of complicated features much easier (especially when we need to interact between screens)
- possible to create almost "native" (I think nobody ever felt it, though) experience, with fallbacks if no internet is there

**Cons**:

- broken "back" button (sometimes it works properly, but in general people don't trust it)
- broken "open in a new tab" behaviour – people like to handle links in `onClick` handler, and browser can't recognize it as a link (even if the majority of the links are valid, sooner or later you'll encounter a non-link "link")
- sometimes broken "refresh" button – after refreshing you end up in a different UI (usually slightly, but still different)
- increased TTI (time to interaction), more on this later
- bad performance on low-end devices (and I suspect higher battery consumption, but can't really proof this one) 

## Why is it slow?

Back to the beginning, where I said we found that WordPress was actually faster than our new shiny stack. Our application was not the most sofisticated, but had some interactive parts – it was a financial website, where you can apply for a loan, go to your personal accout, restructure your loan, and change some personal details. Because it is financial and fully online, we had to collect a lot of data, and it was several forms with live validation, on-the-fly validation and some intrinsic flows, so it made it a perfect case for frontend. However, one problem – it was slow; as I mentioned, load time increased twice. So, I was looking into the reasons why it is so big, and is turned out, the size was basically dictated by our react ecosystem – all libraries we needed created this big chunk of js (around 350KB gzipped). Another big chunk was our actual application code – it was another 350kb, and in total we ended up with ~2.6MB non-gzipped data, which is, of course, insane.
It means that no matter how optimized our code is, in order to start the application, a browser has to do several things:

1. Download this 650KB file
2. Ungzip it
3. Parse it (this task is extremely pricy for mobiles)
4. Execute it (so we'll have React runtime and our components become interactive)

You can read more on this process in [Addy's Osmani article](https://medium.com/dev-channel/the-cost-of-javascript-84009f51e99e).

At the end, my findings were that we owe this time increase to big bundle size. As I mentioned, though, our vendor chunk was half of the size, so it means that basic functionality (we had pretty "rich" homepage) would require a big chunk already, and can solve the problem only partially.

I have to say that we were modern enough, and in order to help SEO and mobile clients, we had [server-side rendering](http://blog.bloomca.me/2017/06/11/server-side-rendering-with-prefetch.html), implemented using Node.js + Express, where we fetched all needed data for the client. And it really helped – the user was able to see the markup (though it does not work on old mobile devices), but the problem is that before the javascript is downloaded, parsed and executed, so React can add event listeners and your page is finally interactive, a lot of time actually passes.

## How slow is it?

Let's take a look at actual sizes of the applications. Usually I am logged out, and I have my adblocker enabled.

## Airbnb ([individual place page](https://www.airbnb.com/rooms/9089815))

Bright engineering team, many articles about migration to React. Let's see how is it going, on the typical page of individual place:

<p class="centred-image full-image">
  <img class="image" src="/assets/img/airbnb.jpg" />
  <em>A lot of files. Microservices in action...</em>
</p>

There are tons of files, and in total it makes it 1.3MB gzipped. Again, 1.3MB to show you a place – pictures and description. Of course, there is a lot of user interaction, but at the end of they day, as a user, I want to look at the place – also I can be logged out. Is it user-friendly? I don't know, but I'd be happy with static content to just explore what are requirements, to read description, etc.
I am pretty sure that aggressive code-splitting allows them to ship features faster, but the price here is user's comfort and speed.

## Twitter (regular feed of logged in user)

<p class="centred-image full-image">
  <img class="image" src="/assets/img/twitter.jpg" />
</p>

Just 3 initial files, and 1 later (I guess lazy loading):
- init file, 161KB
- common file, 249KB
- home page file (page splitting!), 65KB
- "native bundle", 44.5KB (not sure what it is, but it was loaded afterwards)

In total 475KB + 44.5KB for some lazy-loaded file. Much better than AirBnB, but still, a lot of stuff, just to show you feed. There is also a [mobile version](https://mobile.twitter.com), which feels much lighter, but the size is actually similar.

## Medium ([Article about cost of JS](https://medium.com/dev-channel/the-cost-of-javascript-84009f51e99e))

Medium is a cool platform for blogs. So, essentially, it is an advanced place for texts, similar to one you are reading right now. Let's open an article, which I mentioned before, about cost of JS:

<p class="centred-image full-image">
  <img class="image" src="/assets/img/medium.jpg" />
</p>

Also 3 files:
- main bundle, 337KB
- common bundle, 183KB
- notes, 22.6 (maybe this amazing functionality to highlight commas)

In total 542KB, just to read an article. You can read it without JS at all, by the way, so it means that it is not that crucial for the main task.

## What can be done?

All these websites I've mentioned load on my latest 15" MBP with stable internet connection for 2–3 seconds, and become fully usable after another 2 seconds, which is not that bad nowadays. The problem is how _normal_ it has become over the last decade – [slack has 0.5s delay](https://twitter.com/freetonik/status/932965318816911361) between clicking and switching to the channel, and we don't even notice it anymore.

Don't take this rant as an attack on the SPAs itself – I like them (at the end of the day, I write one for my day job!), and they allow to create very powerful user experience, which works across all platforms. However, a lot of things I was working on as a SPA, in fact should not be one. For example, some time ago I've made a [portfolio website](https://jess.gallery/) to my wife, and of course I did using [cool SPA stack](https://github.com/JessArt/website) – but in fact, it is just a blog with a lot of images and couple of widgets; so I'm guilty of this trend more than many.

Also, feel free to take a look [at the first CERN website](http://info.cern.ch/). Just compare navigation speed when you are on this website, and the responsiveness, when you are leaving it to some external, modern one.
