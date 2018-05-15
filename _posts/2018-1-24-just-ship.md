---
layout: post
title: Just Ship
keywords: blog, imposter syndrome, programming, seva zaikov, bloomca, jekyll, hugo, github pages, netlify, build product
excerpt: "\"If a tree falls in a forest and no one is around to hear it, does it make a sound?\""
---

Recently I attended a very interesting meetup about [static site generators](https://www.netlify.com/blog/2017/05/25/top-ten-static-site-generators-of-2017/) – tools, which are usually used to generate static websites; the classic example is blogging, though they are capable of much more: docs, portfolios, etc.
One of the talks was about personal blog tech journey - nothing special, just couple of posts per year. The speaker was talking about trying usual setup of [Jekyll + Github](https://help.github.com/articles/using-jekyll-as-a-static-site-generator-with-github-pages/), then moving to Linode VPS (so you can add webhooks and build some js), then getting rid of server and switching to [Hugo](https://gohugo.io/) and back to github pages, and finally, moving to Netlify, so they can serve it with benefits of CDN, executing building process on the server, and with proper CI/CD.

There is only one question. Why?

Let's be honest, if you want to have a blog, literally serving html files is fine. My own blog exists for about 1.5 years, and in that time I managed to write about 15 articles, so we can roughly map one post to one month. I also don't really edit them after publishing, unless something need to be clarified – so, let's assume I don't touch them after publishing. So, would it hurt to just switch to plain html files, and add one link to the list on the main page? No, it would not. Even if I take the weirdest path possible, and write articles in markdown, and then use some online tool to convert it to HTML, it is still a perfectly valid solution – it would take around 5 minutes to copy everything around. 5 minutes for a month – decent price for alternative playing with your setup for (possibly) days.

There is nothing wrong to play with technology, explore new possibilities and validating some tech solution. I was in the same situation, and before moving to this tech stack – [Jekyll + Github Pages](https://github.com/Bloomca-me/bloomca-me.github.io), I was playing with Hugo, wanted to write my own solution – and I never had time for it, so it took months to research these not so complicated things. After that I started this blog in just a day, using boring proven stack.

The only difference is that blog is a product, and the value is articles, knowledge you share with the community, your personal brand and so on. In case of tech exploration the value is your experience, which you can, again, share with the community, but they are independent, and exploration should not prevent you from shipping today, even using the boring technologies.

Just ship.