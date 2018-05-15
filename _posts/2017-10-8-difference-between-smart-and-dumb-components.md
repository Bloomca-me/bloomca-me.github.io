---
layout: post
title: Difference between smart and dumb components in React
keywords: javascript, react, redux, seva zaikov, bloomca, components, smart components, dumb components
excerpt: React allows you to write components in different ways, and I'd like to describe my own approach to smart and dumb components – in a nutshell, don't be afraid to make too many of smart components.
---

In early days of single-page applications, we used to hardcode a lot of stuff, and very often we ended up with a lot of chunks of code, which did not make a lot of sense outside of their page. In other words, very often code was tighly coupled. For instance, in Angular 1, default behaviour of creating new scope with parent scope as a prototype encouraged code, there we rely on this feature, and this made later reuse very hard.
[React](https://reactjs.org/) popularized [component approach](https://reactjs.org/docs/components-and-props.html), which means that we incapsulate all logic into separate components, and then compose them in declarative way. So, using JSX, we will have something like:

{% highlight js linenos=table %}
const DescriptionSection = (
  <ShadowSection>
    <WelcomeMessage name={name} />
    <Code withCopy>
      {code}
    </Code>
  </ShadowSection>
);
{% endhighlight %}

The beauty of the solution above starts to appear when you have to move this component a lot, or remove some pieces from it (or move part of this component to another). Basically, all sorts of possible business changes, which we face all the time in our frontend applications -- you know, pace has never been so fast.

Also, because we need to somehow make these components flexible, we need to pass data to them, and here it becomes more complicated. Originally (and which is still true), React was standing only for "V" (view layer), so we had to deal with external state on our own. Also, there is a mechanism inside components called "state", which works, but does not really scale (you can do it, but general consensus is that there are easier ways). Soon after discovering all that issues, Facebook announced Flux architecture -- unidirection data flow, which was pretty different from usual two-way data binding at the moment in the industry.

Flux and it's successors (Alt, Reflux, Nuclear, others, and finally Redux, which is a de-facto standard nowadays) popularized and made it very prominent, that state should be contained outside of the views; and developer decided how and where do we connect this external state to our application, and I will show you 3 different approaches:

## Pass Everything Down

This is the most verbose way, but also the most explicit -- there is absolutely no magic, we just pass store object down the component tree, and each components render based on what data they need.

One of the problems here is performance. VDOM approach is pretty fast by default, so until your application is big, you won't really notice it, but at some point you'll need to dig into it. [React's PureComponent](https://reactjs.org/docs/react-api.html#reactpurecomponent) won't help here, because we don't pass just granular updates, rather the whole state, and it means we will need to write our own [shouldComponentUpdate](https://reactjs.org/docs/optimizing-performance.html#shouldcomponentupdate-in-action) functions. For example, let's take a look at the `<WelcomeMessage>` component. Let's think that we greet a user there, so we need to pass the user down there, and `DescriptionSection` JSX will look like:

{% highlight js linenos=table %}
const DescriptionSection = (
  <ShadowSection>
    <WelcomeMessage store={this.props.store} />
    <Code withCopy>
      {code}
    </Code>
  </ShadowSection>
);
{% endhighlight %}

And welcome message itself might look like the following (with performance optimisiations applied):

{% highlight js linenos=table %}
import React, { Component } from 'react';
import RPT from 'prop-types';

class WelcomeMessage extends Component {
  static propTypes = {
    store: RPT.shape({
      user: RPT.shape({
        name: RPT.string
      })
    }).isRequired
  }
  
  shouldComponentUpdate(nextProps) {
    return nextProps.store.user.name !== store.user.name;
  }
  
  render() {
    const { store: { user: { name } } } = this.props;
    return (
      <h2>
        {`Welcome, ${name}`}
      </h2>
    );
  }
}
{% endhighlight %}

This component is pretty simple, but with the growth of used props it will become more and more complicated to write these update functions. Also, passing store around is pretty tedious in the big projects, but it is definitely a viable solution for not so big applications.
The biggest advantage is the simplicity -- there is no magic at all, no [context](https://reactjs.org/docs/context.html) used to store some variables in the instance of React's VDOM.

## Build Presentation Components ("Containers")

This approach was popularized by [Dan Abramov's article](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0), where he says that some components are in charge of data, and some are just for the render. In the wild I can say it is interpreted in a way that only some components (usually router's top-level components) are allowed to get all needed data, fire requests for it, and then pass it down to all its child components.

Important thing here is that "dumb" components are not allowed to know anything about the store, so `WelcomeMessage` from the previous example should not receive the whole state, rather just a `name` string. So, the component will look like the following:

{% highlight js linenos=table %}
import React, { PureComponent } from 'react';
import RPT from 'prop-types';

export default class WelcomeMessage extends PureComponent {
  static propTypes = {
    name: RPT.string.isRequired;
  }
  
  render() {
    const { name } = this.props;
    return (
      <h2>
        {`Welcome, ${name}`}
      </h2>
    );
  }
}
{% endhighlight %}

It's true that the result component is more flexible -- instead of any knowledge about store implementation, we just pass `name` property explicitly, which helps us to render this component with arbitrary names.

Performance-wise, as I've shown above, it is much simpler -- we just have to be careful what do we pass down, and in case of using something like [reselect](https://github.com/reactjs/reselect), pureComponent will do its work most of the time. Just try to avoid using `.bind`, arrow functions, new objects\arrays when passing down all needed props.

The biggest problem in this case is the necessity to pass this parameter, so the "smart" component should explicitly know about need of each "dumb" component. It is fine in the beginning, but as soon as we reach several levels of nesting components, it becomes pretty tricky to track what exactly do we need to pass down, and, moreover, it is even trickier when we want to remove some part of the application, but _inside_ another dumb component. So we can easily keep passing unnecessary property, which we forget to remove from propTypes because of the nesting.

The answer to this concern is that we don't necessarily want as little smart components as possible -- if there is a big chunk of reusable UI (let's say, calculator), don't be afraid to make this one a smart component as well -- it will make removing calculator from one page and adding it to another much simpler.  

## (Almost) Everything is a Presentation Component

This is a very similar approach as it was shown in the previous section, but with applied advice from the end it to extreme. It means that as soon as we need some data from the global store, we have to access it by ourselves -- so, a lot of our components become presentational. Of course, basic UI elements, like buttons, loaders, tabs and so on will be "dumb" anyway -- it is very rare that we need to adjust their state to some global property; but all that need, should access the store by themselves.

For instance, `<WelcomeMessage>` component from our code example should do exactly this -- instead of receiving store or the name, we will add [HOC -- Higher-Order Components](https://reactjs.org/docs/higher-order-components.html) to our class, which will add this property to props. It simplifies adding this component to any place, because the single thing we need to do -- just to add this line `<WelcomeMessage />`, and that is it! If we don't need it there anymore, we can safely remove this line and be sure that nothing should be changed in the current component!
So, let's see how it will work in our `<WelcomeMessage>` component. I will assume that we use [Redux](https://github.com/reactjs/redux) and [React-redux](https://github.com/reactjs/react-redux) for accessing store.

{% highlight js linenos=table %}
import React, { PureComponent } from 'react';
import RPT from 'prop-types';
import { connect } from 'react-redux';

const mapStateToProps = state => ({
  name: state.user.name
});

class WelcomeMessage extends PureComponent {
  static propTypes = {
    name: RPT.string.isRequired;
  }
  
  render() {
    const { name } = this.props;
    return (
      <h2>
        {`Welcome, ${name}`}
      </h2>
    );
  }
}

export default connect(mapStateToProps)(WelcomeMessage);
{% endhighlight %}

I could use [reselect](https://github.com/reactjs/reselect) to make reusable selector functions, but for the sake of simplicity let's avoid it. Also, because I return a string primitive, PureComponent will work perfectly.

You might ask at this point, "but isn't it too inflexible to tie all these components to the store?", and it is a perfectly valid concern. What you do in this approach, if you see that you need to reuse some piece of functionality, you just create a "dumb" component and in your small "smart" component you render it with properties from the store.

So, let's rewrite our previous component to these two new. I'll keep them in the same file, again, for the sake of simplicity.

{% highlight js linenos=table %}
import React, { PureComponent } from 'react';
import RPT from 'prop-types';
import { connect } from 'react-redux';

const mapStateToProps = state => ({
  name: state.user.name
});

export class WelcomeMessage extends PureComponent {
  static propTypes = {
    name: RPT.string.isRequired;
  }
  
  render() {
    const { name } = this.props;
    return (
      <h2>
        {`Welcome, ${name}`}
      </h2>
    );
  }
}

class WelcomeUserMessage extends PureComponent {
  static propTypes = {
    name: RPT.string.isRequired;
  }
  
  render() {
    return <WelcomeMessage name={this.props.name}>;
  }
}

export default connect(mapStateToProps)(WelcomeUserMessage);
{% endhighlight %}

This might be a little bit more verbose, but we still have this advantage of adding and removing components in a very easy manner, which might be a huge advantage, if you need to restructure your views pretty often.

When to use this approach?

## Network Requests

I put it to a special section, because you might fire your requests from whatever components you want in any of those approaches, so it makes sense to discuss it separately. By network requests here I mean requests which receive data for all those components on our page, and here I see two main approaches.

The first one tells that we need to fetch all needed data at the top component (usually the top component of the given route). It does not really matter how we will retrieve this data later -- either in the top component as well, or inside the individual components. The main benefit of this approach is that because it happens in one place, we understand very well what we fetch, in which order, and so on. The downside is that, again, we need to understand precisely what are we going to render, which data is needed, and we need to track that this fetching function is up-to-date to what we render right now (because content of the route tends to change over time).

The other approach is to fetch everything you need inside the component. It means that as soon as component mounts, we fetch everything it needs, and in reality you might easily end up with several identical requests, which were fired from different components. In order to prevent it, you need to build some sort of caching, which will not fire a request again in case it is in the progress, or to be very attentive which components do you render, so they don't request the same info. It is a very hard task, and this drawback can easily outweight everything else, so if you don't want to deal with it and have more predictable network requests, fetch everything only in several well-known places.

But also, please keep in mind, that the same exact problem might arise in case you implement some sort of route guards -- for example, if we want to check status of user before allowing him to enter the route, and then, in the route itself we fetch user to get this info. Caching is a very [tricky subject], so don't try to be too clever in this situation. I've built a library on top of redux, [redux-tiles](https://github.com/Bloomca/redux-tiles), with first-class support of caching, but it is just my preference to fire requests everywhere and to be sure that it won't cause any unnecessary requests.

> Server-side rendering is a little bit different beast, so I shall not touch it here -- feel free to read [my article about it](http://blog.bloomca.me/2017/06/11/server-side-rendering-with-prefetch.html)

## Conclusion

So, which approach to choose? Unfortunately, there is no right answer -- otherwise, everyone would already use it. Choose one which makes sense for your situation and is the most appealing to you and your team -- if you are pretty sure about your design, you can go with just fetching all needing info in the top components and passing it down. It will also work even with changing design, but not so big application -- if your team is not very big, it is possible to keep all details in your head at once, and change everything during moving components. But if your application is going to be big (or already is), don't be afraid to rethink what "smart" components are, and how to avoid big cognitive pressure when making any changes -- at the end of the day, our goal is to make codebase more maintanable and easy to add new features or change something in existing ones.




