---
layout: post
title: The Most Clever Line of JavaScript
keywords: javascript, javascript quirks, seva zaikov, bloomca, javascript snippet, javascript prototypes, javascript, javascript call, javascript this
excerpt: addressParts.map(Function.prototype.call, String.prototype.trim) – how does it work?
---

<blockquote class="highlight-paragraph pull-in">
  <p>address.map(Function.prototype.call, String.prototype.trim);</p>
</blockquote>

Recently [my friend](https://twitter.com/vedroarbuzov) sent me a very interesting JS snippet, which he found in one [open-source library](https://github.com/pelias/openstreetmap/blob/313f208ea323232919e42bf88871d8e19ddacec3/stream/address_extractor.js#L54):

{% highlight js linenos=table %}
addressParts.map(Function.prototype.call, String.prototype.trim);
{% endhighlight %}

At first, I laughed and thought "nice try". Second thought was that `map` accepts only one argument, and I went [to MDN docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map), where I realized that you can pass context as a second argument. At this point I was really puzzled, and after running my confusion increased to the limit -- it worked as expected!

I've spent at least half an hour playing with it, and it was an interesting example how magical JavaScript can be, even after years spent writing it. Feel free to figure it out by yourself, and if you want to check out my understanding, keep reading!

So, how does it work? Let's start with super naive implementation (which is actually shorter and easier to read :)):

{% highlight js linenos=table %}
addressParts.map(str => str.trim());
{% endhighlight %}

But as it is clear, we strive here to avoid creating new functions, so let's move on. [Function.prototype.call](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/call) is a function in prototype of all javascript functions, and it invokes function, using first argument as `this` argument, passing rest arguments as parameters of function invokation. To illustrate:

{% highlight js linenos=table %}
// this function has `Function` in prototype chain
// so `call` is available
function multiply(x, y) {
  return x * y;
}

multiply.call(null, 3, 5); // 15
multiply(3, 5); // same, 15
{% endhighlight %}

Typical usage of the second argument can be the following -- imagine you have a class-based react component, and you want to render list of buttons:

{% highlight js linenos=table %}
class ExampleComponent extends Component {
  renderButton({ title, name }) {
    // without proper `this` it will fail
    const { isActive } = this.props;
    return (
      <Button key={title} title={title}>
        {name}
      </Button>
    );
  }

  render() {
    const { buttons } = this.props;

    // without second param our function won't be able
    // to access `this` inside
    const buttonsMarkup = buttons.map(this.renderButton, this);
  }
}
{% endhighlight %}

However, from my experience, it is not that common to use this second argument, usually class properties or decorators are used to avoid binding all the time.

There is one similar method -- [Function.prototype.apply](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply), which works the same, except that the second argument should be an array, which will be transformed to a normal list of arguments, separated by comma. So, let's see how we can use it to calculate maximum:

{% highlight js linenos=table %}
Math.max(1, 2, 3); // if we know all numbers upfront
// we can call it like that

Math.max([1, 2, 3]); // won't work!

Math.max.apply(null, [1, 2, 3]); // will work!

// however, ES2015 array destructuring works as well:
Math.max(...[1, 2, 3]);
{% endhighlight %}

Now, let's try to recreate a call which will solve our problem. We want to trim the string, and it is a method in `String.prototype`, so we call it using `.` notation (however, strings are primitives, but when we call methods, they are converted to objects internally). Let's go back to our console:

{% highlight js linenos=table %}
// let's try to imagine how trim method is implemented
// on String.prototype
String.prototype.trim = function() {
  // string itself is contained inside `this`!
  const str = this;
  // this is a very naive implementation
  return str.replace(/(^\s+)|(\s+$)/g, '');
};

// let's try to use `.call` method to invoke `trim`
" aa ".trim.call(thisArg);

// but `this` is our string itself!
// so, next two calls are equivalent:
" aa ".trim.call(" aa ");
String.prototype.trim.call(" aa ");
{% endhighlight %}

We are now one step closer, but still not yet there to understand our initial snippet:

{% highlight js linenos=table %}
addressParts.map(Function.prototype.call, String.prototype.trim);
{% endhighlight %}

Let's try to implement `Function.prototype.call` itself:

{% highlight js linenos=table %}
Function.prototype.call = function(thisArg, ...args) {
  // `this` in our case is actually our function!
  const fn = this;

  // also, pretty naive implementation
  return fn.bind(thisArg)(...args);
};
{% endhighlight %}

So, now we can put all pieces together. When we declare function inside the `.map`, we pass `Function.prototype.call` with bound `String.prototype.trim` as `this` context, and then we invoke this function on each element in the collection, passing each string as `thisArg` to the `call`.
It means that `String.prototype.trim` will be invoked using string as `this` context! We already found that it will work, using our example:

{% highlight js linenos=table %}
String.prototype.trim.call(" aa "); // "aa"
{% endhighlight %}

Problem solved! However, I think it is not the best example how one should write such a code in JavaScript, and instead just pass a simple anonymous function:

{% highlight js linenos=table %}
addressParts.map(str => str.trim()); // same effect
{% endhighlight %}