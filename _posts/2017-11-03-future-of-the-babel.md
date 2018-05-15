---
layout: post
title: Future of the Babel.js
keywords: javascript, babel, react, JSX, seva zaikov, bloomca, ES2015, ES6, modern javascript, transpilation, future of javascript, ECMAScript, TC39
excerpt: Babel.js is here for a long time and won't go away – partially because of JSX, partially because of old browsers, and partially because of sweet new features.
---

JavaScript was originally added to make possible simple animations and effects. Since the beginning, browsers added a lot of APIs (the biggest breakthrough was introducing of [AJAX](https://developer.mozilla.org/en-US/docs/AJAX)), and nowadays we can make pretty powerful applications, which will be cross-platform out of the box. The latter advantage is so huge, that nowadays almost everybody targets web, since standartisation worked out pretty well, and websites look and behave exactly the same, no matter which platform you use.

The problem, however, is that JavaScript was not designed for such applications, and there is no modules system, some features are missing and so on. There were several attempts to replace JS (Google wanted to ship its own VM, [Dartium](https://webdev.dartlang.org/tools/dartium) instead of JS -- now Dart just compiles into JS), but all of them failed, and different tools written primarily in Node.js allowed JS to flourish and to serve as the main language of the web. So, JS remained (though [WebAssembly](http://webassembly.org/) may change the picture in the future), but it was obvious that we need improvements, and [ES2015 standard](https://www.ecma-international.org/ecma-262/6.0/) was accepted in 2015. You can read more about the whole process [in this article by Dr. Axel Rauschmayer](http://2ality.com/2015/11/tc39-process.html).

One of the goals of JavaScript is backward compatibility, so it does not break existing code. In practice it means that we can introduce new constructions and use new special words from the list of reserved words (like it was done with `class`, `async` and `await`), and your website will not work in older browsers, but we can not change existing behaviour, because it will break some code out there -- and there are so many different websites, that we can be sure that every feature of JavaScript is used nowadays somewhere in the web. It also is worth to note that other approaches exist as well -- for example, we could just use some attribute on `<script>` tag, like `<script version="6">`, but it will require to ship several versions of VMs, and it adds a lot of other complexity.

Considering that fact, and also due to omnipresence of JavaScript nowadays (there are incredible amount of browsers out there with absolutely different capabilities), one, if they want to build a working and usable web application, should write ES5 code, which is a standard de-facto amongst running JS VMs. A lot of modern browsers support many new features, and some are almost fully support ES2015, but again, a lot of mobile browsers have very limited support (or no support at all), and your fully functional website in the chrome for some reason does not really work in someone's application webview.

Nowadays, it is considered perfectly valid to transpile your code to older version of JavaScript, and you won't really see a lot of modern codebases without transpilation via Babel (or TypeScript). But what is interesting, even creators of babel recommend to use [babel-preset-env](https://github.com/babel/babel/tree/master/experimental/babel-preset-env), which will figure out which features to transpile, and which to leave alone. Why do we need to transpile, in case we support only modern browsers (let's say no IE, only green Chrome, Edge, Safari and Mozilla), they all have pretty good support?

## Tutorials

All tutorials out there either use webpack and babel, or don't use any of those. It creates a strong connection that if you want to develop professionally, you _have to_ use Babel, and transpile your code. Also, it is pretty common nowadays to not really dig into what exactly do we support (usually it is IE10+), and transpile everything into ES5, which works for sure everywhere.
Maybe in the future we will see something like "youmightnotneedbabel" (by analogy with [youmightnotneedjquery](http://youmightnotneedjquery.com/) or [youmightnotneedlodash](https://youmightnotneed.com/lodash/)), which actually could be run even now -- webpack@2 supports `import` and `export` statements, and in case you don't use JSX you might find that you can switch to just supported features.

## Babel Plugins

One of the biggest reasons is babel plugins. Basically, we can not really transpile a lot of features, but is it's plugins to somehow transform other things. The best example here, probably, is React -- not only babel adds transpilation of JSX, but also there are special plugins for production builds, which transform static components to constants, so they don't have to be recreated all the time. Also babel can strip away [flow annotations](https://babeljs.io/docs/plugins/transform-flow-strip-types/), add [angular annotations](https://www.npmjs.com/package/babel-plugin-angularjs-annotate), and do a lot of other cool stuff.
Also, you can write your own babel plugin, which will remove some pieces of code (let's say debug information, or some unnecessary logging), or which will do the opposite -- add some code. Not everybody needs this feature, but it is very good to know that this extensibility is possible in your application, and in case you need you'll just add a small piece of configuration.

## Using New Crucial Features

Another reason is actual new features. If something is already in stage-3 (so it definitely will end up in the specification), why not to use it already? And even if it is not there, but makes a lot of sense to your team, why not to adapt and then add a library for it later, in case it was rejected by the TC-39?
Also, as I mentioned, many people already have babel in their building pipeline just for some plugins, like JSX, so adding these new features is usually 1 new dependency and a line in `.babelrc` file.
It also does not make a lot of sense to workaround things which will be solved soon on the language level. For instance, [object destructuring](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment) is a pretty popular example, as less verbose [Object.assign](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign) functionality, and omitting some properties:

{% highlight js linenos=table %}
// with object desctructuring we have nice and clean solution
// to separate properties we want to use and to pass further
const { title, size, ...otherProps } = props;

// without this feature, we need to use something like lodash:
const otherProps = _.omit(props, ['title', 'size']);

// ==========================

// pretty popular pattern in something like Redux store
const newState = {
  ...state,
  [key]: {
    ...state[key],
    [id]: data
  }
};

// without object destructuring, we need to use Object.assign:
const newState = Object.assign({}, state, {
  [key]: Object.assign({}, state[key], {
    [id]: data
  })
});

// in this example it took even less space, but the code is less
// readable and cluttered with Object.assign
{% endhighlight %}

## Using New Appealing Features

Sometime features are so exciting, that even stage-0/stage-1 does not scare people, and they start to use immediately, as soon as stable plugin is developed. Based on their experience, comittee decides about API, changes, and whether they should proceed with it or reject the feautre. This workflow puts babel and JavaScript in a very unique position -- it allows you to optionally include some features, which might end up in the spec, and give immediate feedback about it. I don't know any other language, which has so short feedback loop, and which can afford to propose something, try it in the wild, and after that decide on its future.

## Node.js

Interesting enough, Node.js was not affected by Babel that much, and while projects like [babel-node](https://babeljs.io/docs/usage/cli/#babel-node) exist, and webpack supports compilation target [node](https://webpack.js.org/configuration/target/), it is not that popular there. There are couple of reasons here:

- we run Node.js on our own servers, so we are sure which version it has, and we can just choose set of allowed features
- after merging back with io.js, Node.js is pretty much up-to-date with V8, and nowadays it has almost 100% of ES2015 support
- bundling does not make a lot of sense -- we don't want to include images or styles, and we don't want to bundle a single file

The single group of people who are interested in using Babel in Node.js, is the one who writes small scripts, which are only for support/building/etc. E.g. you can find gulp scripts written in the most modern JS, some building scripts, sync stuff and others non-crucial for production running things.

## Conclusion

Babel.js is a standard thing in the modern applications, we don't even think about it and question this choice anymore, when we see the build process of a new application. So, it is already there, and often we depend on some functionality it provides -- JSX, some cool features, or just for trying out new stuff (especially there are some very appealing to lovers of functional programming proposals, like [pipe operator](https://github.com/tc39/proposal-pipeline-operator)).
What is really interesting, is that Babel has its own place in the feedback loop to the proposals -- it helps people to understand what is flawed in the design, and how it can be better; but this scheme works only if Babel will be there forever. JSX is very popular now, and speed of changes is very high, so seems that Babel won't go away for a long time -- seems we just get used to the ability of using new cool features almost immediately, and that we need to set up transpilation process first in order to start development.
