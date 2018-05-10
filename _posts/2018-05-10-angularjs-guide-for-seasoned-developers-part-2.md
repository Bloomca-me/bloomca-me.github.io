---
layout: post
title: Angular.js Guide for Seasoned Developers – part 2
keywords: javascript, angular, angular.js, angular 1, angularjs, angular.js tutorial, angular.js for react developers, angular directive, digest cycle
tags: javascript angular
---

This is the second part of the angular.js tutorial for seasoned developers. [In the first part](https://blog.bloomca.me/2018/04/15/angularjs-guide-for-seasoned-developers-part-1.html) we talked about basics, controllers and how digest cycle works in general – this time we'll focus on more basic building blocks – directives, factories and services, diving into more details.

## Directives

This is the most important part of Angular.js, because right now it is a recommended way to build your applications (actually, [components](https://docs.angularjs.org/guide/component) are, but they are just [wrappers around directives](https://docs.angularjs.org/guide/component)). It allows you to create isolated directives, which receive only selected properties (both values and functions).

What is a directive? Directive is a special angular entity, described by object, which allows you to add custom tags and attributes to your templates (as well as classes and comments, but they are a little bit confusing – mostly it is either a tag or an attribute).
We learned in basics about parsing cycle, but we did not dive into custom tags and attributes, except for recursive part – let's fill this gap now. As soon as angular parser encounters a custom tag (or an attribute) in the template, it searches for a registered directive using angular's dependency injection system (there is one trick, though – [it normalizes the tag's name](https://docs.angularjs.org/guide/directive#normalization), so directive name and tag look differently: `myDirective` for the DI system, and `<my-directive>` for a template).

After it found a corresponding directive, it [compiles](https://docs.angularjs.org/api/ng/service/$compile#-compile-) it first. It allows you to transform template DOM, before angular adds any behaviour to it – it allows you to manipulate the template, in case you want to transform just once, to avoid performance overhead. Usually you don't need it, so it is pretty hard to see this method in the wild.
You can return a function or an object from the `compile` method – a function is `post-link`, and an object contains both `pre` and `post` links. We'll return to them later, after looking into `controller`, but in a nutshell – `link` function allows you to manipulate and read DOM – add event listeners, add some non-angular behaviour, etc.

In order to understand the difference between `compile` and `link` method, let's look into the following example – we'll use [ng-repeat directive](https://docs.angularjs.org/api/ng/directive/ngRepeat) to illustrate that. We create two similar directives which simply add a new attribute to the element, but one will do it in `compile`, and other in `link` methods:

<p data-height="500" data-theme-id="0" data-slug-hash="yjKOYV" data-default-tab="js" data-user="bloomca" data-embed-version="2" data-pen-title="Compile and link in Angular.js" class="codepen">See the Pen <a href="https://codepen.io/bloomca/pen/yjKOYV/">Compile and link in Angular.js</a> by Seva Zaikov (<a href="https://codepen.io/bloomca">@bloomca</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

If you open a dev console now, you should see that `compile` method was called only once, but `link` was 4 times – but as always with optimizations, be careful and don't try to do it prematurely.

The next thing is to create a new scope for your directive – and we have several choices here; by default it will inherit current scope, like it works with controllers, but we can choose to create a completely isolated scope, or pass only selected properties. It is configured using `scope` property with the following values:

```js
{
  scope: true, // default option, inherits parent's scope
  scope: {}, // no inheritance at all, our $scope won't contain any parent's properties
  scope: { // pick some properties, preferred way
    name: '=name', // two-way data-binding with name property
    choose: '&select', // functions – and prop name in HTML will be `select`
    isVisible: '<' // one-way data binding, property's name should also be `isVisible`
  }
}
```

After executing `compile`, if present, and creating a new scope with chosen inheritance, angular looks at the `controller`. It works exactly as you expect – function works the same way as described [in part one](https://blog.bloomca.me/2018/04/15/angularjs-guide-for-seasoned-developers-part-1.html#controllers): you can use `$scope` and assign properties directly on it, or you can use property `controllerAs` and assing properties on `this`. In order to use `controllerAs`, you need to provide this property with a string indicating how you want to refer to controller object.

Next step is linking – we touched it briefly in `compile` function, mentioning that we can return a post-link function or an object with both pre-link and post-link functions. So, what are they and what is linking? Linking itself is a term to indicate connection between template and scope, and prelink will be executed right before linking, and postlink right after. Postlink is a default link function, so if you return a single function from `compile`, or declare a property named `link`, it is exactly a postlinking function – and you also likely to need only it. In this function you can safely add event listeners – but remember, that if you declare them using regular DOM API, and you want to update some data in your scope, you have to `$apply` these changes, otherwise it won't re-render.

All of that is possible to do manually, and angularjs provides [an example in docs](https://docs.angularjs.org/api/ng/service/$compile#example).


<p data-height="550" data-theme-id="0" data-slug-hash="OveZGM" data-default-tab="js" data-user="bloomca" data-embed-version="2" data-pen-title="angular.js directive example" class="codepen">See the Pen <a href="https://codepen.io/bloomca/pen/OveZGM/">angular.js directive example</a> by Seva Zaikov (<a href="https://codepen.io/bloomca">@bloomca</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

You can a lot in this example, but the main point is that we can choose what is passed to the directive, so we don't accidentally change a value from a parent's scope. This concept of allowing certain properties is very similar to properties in any component-based frameworks, like React or Vue!

Another exciting (yet confusing for some people) feature is transclusion (directive `ng-transclude`) – passing another template to our directive. Template string per se are just HTML, but this feature allows you to pass dynamic templates! Template which we are passing, is executed in the context where it was defined; it means you can create different wrappers around your content, without any coupling! You might think about it as `this.props.children` in React world – it is a very similar concept.

- [Official docs on directives](https://docs.angularjs.org/guide/directive)
- [Difference between controller, compile and link in directives](https://stackoverflow.com/questions/15676614/link-vs-compile-vs-controller)
- [SO question about the difference between directive and controller](https://stackoverflow.com/questions/18757679/angularjs-directives-vs-controllers)
- [SO question about directive's lifecycle](https://stackoverflow.com/questions/24615103/angular-directives-when-and-how-to-use-compile-controller-pre-link-and-post)

## Services and Factories

I will tell you immediately, that effectively there is no difference between them at all. Moreover, `angular.service` is defined in terms of `factory` inside, and in pseudocode we can write something like this:

```js
angular.service = function(name, Constructor) {
  return angular.factory(name, function(..args) {
    return new Constructor(...args);
  });
}
```

As you can see, the only difference is that function for `service` is a constructor one, and for the `factory` just a plain function. That is it about them, and since `factory` is more flexible, I can recommend to use only it; but if your team preferres services, it is perfectly fine, just stick with one, since having more entities doing exactly the same is a little bit confusing (big problem for angular in general).

You should treat services as a tool how different components can execute same actions. E.g. if we have user entity in our application, it makes sense to put all actions related to user – update personal data, upgrade subscription plan, add device, etc, to a unique service. Also, we can require all other services inside, so don't be afraid to separate network layer to another service, as well as some specific logic, like authentication, for example.

Let's write several services/factories:

<p data-height="500" data-theme-id="0" data-slug-hash="qYoZww" data-default-tab="js,result" data-user="bloomca" data-embed-version="2" data-pen-title="Factories in Angular.js" class="codepen">See the Pen <a href="https://codepen.io/bloomca/pen/qYoZww/">Factories in Angular.js</a> by Seva Zaikov (<a href="https://codepen.io/bloomca">@bloomca</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

In our example all factories are just singletones – but it is not correct universally. Instead of keeping state, we can just return a factory function, which will allow us to instantiate different 

> Directives are also nothing else but factories, which return object of a special type, recognized by angular

- [SO question about difference between service, factory and provider](https://stackoverflow.com/questions/15666048/angularjs-service-vs-provider-vs-factory)

## Minification

Angular has one limitation, due to it's desire to parse everything. Remember all these weird arrays, where we put all function's parameters in string version first, and only after declare a function with them? If you want, you can set parameters for your controller/factory functions other services/factories, as long their names are the same as they were registered. Angular parses this list, and decides what to inject; it works perfectly during the development, but all of the sudden fails after minification. The reason for that is that minifier has no idea about this behaviour, and just renames them to something like `a`, `b` (shorter the better – less bytes to transfer!), and parsing stops to work. In order to solve this problem, array notation is used – when we first list all dependencies names, and only then declare our function. Names should be 100% the same as we declare them (first argument in `directive`, `controller`, `factory`, etc).

<p data-height="420" data-theme-id="0" data-slug-hash="VxXaXO" data-default-tab="js" data-user="bloomca" data-embed-version="2" data-pen-title="Minification in Angular.js" class="codepen">See the Pen <a href="https://codepen.io/bloomca/pen/VxXaXO/">Minification in Angular.js</a> by Seva Zaikov (<a href="https://codepen.io/bloomca">@bloomca</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

- [How to minify angular.js code on SO](https://stackoverflow.com/questions/18782324/angularjs-minify-best-practice)
- [Minification in official docs](https://docs.angularjs.org/tutorial/step_07#a-note-on-minification)


## Conclusion 

There is still a lot to cover. Next time we'll talk about how Angular instantiates an application, and will render our own directive manually, without Angular magic – it will show you how to use Angular from other libraries and frameworks, which is probably the most helpful thing nowadays: [AngularJS has last 3 years of support](https://blog.angular.io/stable-angularjs-and-long-term-support-7e077635ee9c), and I think nobody will start a new project using AngularJS, so we'll touch extensively how to migrate from it gradually, without major rewrite.
