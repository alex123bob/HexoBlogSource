---
title: Function as Child Components
date: 2018-12-29 17:44:28
tags:
---

#### **<font color="brown">WHAT</font> is FaCC?**

A Function as Child Component (or FaCC) is a pattern that lets you pass a render function to a component as the `chidren` prop.
It exploits the fact that you can change what you can pass as `children` to a component.
By default, `children` is of type `ReactNodeList` (think of this as an array of JSX elements).
It's what we're used to when placing JSX within other tags.

When you use a FaCC, instead of passing JSX markup, you can assign `children` as a function.

Let's check it out:

````Javascript
<Foo>
  {(name) => <div>`hello from ${name}`</div>}
</Foo>
````

````Javascript
const Foo = ({ children }) => {
  return children('foo');
};
````

**Check out a FaCC example with <font color="red">flesh</font> and <font color="brown">blood</font>**
````Javascript
class WindowWidth extends React.Component {
  constructor(props) {
    super(props);
    this.state = { width: window.innerWidth };
    this.onResize = this.onResize.bind(this);
  }

  componentDidMount() {
    window.addEventListener('resize', this.onResize);
  }

  componentWillUnmount() {
    window.removeEventListener('resize', this.onResize);
  }

  onResize({ target }) {
    this.setState({ width: target.innerWidth });
  }
  
  render() {
    const { width } = this.state;
    return this.props.children(width);
  }
}
````

````Javascript
<WindowWidth>
  {width => <div>window is {width}</div>}
</WindowWidth>
````
<font color="#663300" style="text-decoration: underline;">**The real power of render callbacks can be seen in this example. `WindowWidth` will do all of the heavy lifting, while exactly what is rendered can change, depending on the render function that is passed.**</font>


#### **<font color="brown">WHY</font> do we use FaCC? (AKA pros)**

1. The developer composing the components owns how these properties are passed around and used.
2. The author of the Function as Child Component doesn’t enforce how its values are leveraged allowing for very flexible use.
3. Consumers don’t need to create another component to decide how to apply properties passed in from a “Higher Order Component”. Higher Order Components typically enforce property names on the components they are composed with. To work around this many providers of “Higher Order Components” provide a selector function which allows consumers to choose your property names (think react-redux select function). This isn’t a problem with Function as Child Components.
4. Higher Order Components create a layer of indirection in your development tools and components themselves, for example setting constants on a Higher Order Component will be unaccessible once wrapped in a Higher Order Component. For example:

> MyComponent.SomeContant = ‘SCUBA’;

Then wrapped by a Higher Order Component,

> export default connect(...., MyComponent);

RIP your constant. It is no longer accessible without the Higher Order Component providing a function to access the underlying component class. **Sad :(**.


#### **I have mentioned <font color="brown">Render Callback</font>, right?**


<p style="text-decoration: underline;">Long Story short: **Not all rectangles are squares**</p>

It’s important not confuse Function as Child Components with render callbacks. A FaCC is simply one implementation of a render callback. There are other ways to implement render callbacks. In other words, a FaCC is a type of render callback, but not all render callbacks are FaCCs.


#### **FaCCs are an <font color="brown">anti-pattern</font>. Someone said that.**

What?! How can that be? FaCCs are loved and embraced by many in the React community. How can it be an anti-pattern?

> “There are only two hard things in Computer Science: cache invalidation and naming things.”
– Popularly attributed to Phil Karlton


Let’s say that you had a function that took two numbers, added them together and returned the result. What would you call that function? You might call it add, or maybe sum.

````Javascript
const add = (a, b) => {
  return a + b;
};

add(2, 2);
````

Clean coding practices state that you use descriptive names for properties, variables, functions, etc. I hope that we can all agree with this concept. You would never name your add function above badger (although that would be cool) or tapioca or even children, would you?

````Javascript
const children = (a, b) => {
  return a + b;
};

children(2, 2);
````

Of course you wouldn’t. That would be silly, confusing, and not very self-documenting. Then why in the world would you ever use a prop named children to pass a render callback function? I can only imagine seeing this used for the first time in a pull request. Not in my organization, fella. Rejected!

Using a pattern that goes against what we all consider to be a best practice is called an anti-pattern. 
> By definition, then, FaCCs are an <font color="red">**anti-pattern**</font>.


#### **Since Anti-Pattern, any alternative to improve it?**

> Render Props

````Javascript
const Foo = ({ render }) => {
  return render('foo');
};
````

**Example 1**
````Javascript
<Foo render={(name) => <div>`hello from ${name}`</div>} />
````

**Example 2**
````Javascript
const hello = (name) => {
  return <div>`hello from ${name}`</div>;
};
<Foo render={hello} />
````

Notice how this is much more descriptive? The code is self-documenting. You can say to yourself, “Foo calls a function to do the rendering,” instead of “Foo calls something; I don’t remember why.”

> Component Injection - A better solution

````Javascript
const Hello = ({ name }) => {
  return <div>`hello from ${name}`</div>;
};
<Foo render={Hello} />
````

Things look pretty much the same as in the Render Props example above, except the prop we pass is capitalized because components must be capitalized in React. We also pass in props, an object, instead of a single string parameter, but those are the only differences. This should all look very familiar to you.

And our `Foo` component looks a lot more like a traditional React component.

````Javascript
const Foo = ({ render: View }) => {
  return <View name="foo" />;
};
````

One thing to notice is that we rename `render` to `View` using ES6 destructuring assignment syntax (i.e., the `render: View` part), again because components must be capitalized in React.

But the cool part is that instead of calling a function to render, it uses standard JSX composition. Refreshing, huh?

The only thing that might be different from what you might be used to is that instead of `import`ing `Hello`, we inject it (i.e., pass it as a prop).

Injecting your dependencies is a powerful side effect of this technique that allows for easier run-time altering of components and increased ease of mocking tests.

Need more examples? Here’s the more advanced solution shown above, altered for Component Injection. Everything with `WindowWidth` remains the same except for the `render` method:

````Javascript
render() {
  const { width } = this.state;
  const { render: View } = this.props;
  return <View width={width} />;
}
````

…as well as how you use it.

````Javascript
<WindowWidth render={DisplayWindowWidthText} />
````

````Javascript
const DisplayWindowWidthText = ({ width }) => {
  return <div>window is {width}</div>;
};
````

As you can see, the DisplayWindowWidthText component is “injected” into WindowWidth as a prop named render.

We could even pass a different component and get a completely different rendered output – again, thanks to the power of render callback.

````Javascript
<WindowWidth render={DisplayDevice} />
````

````Javascript
const DisplayDevice = ({ width }) => {
  let device = null;
  if (width <= 480) {
    device = 'mobile';
  } else if (width <= 768) {
    device = 'tablet';
  } else {
    device = 'desktop';
  }
  return <div>you are using a {device}</div>;
};
````

#### **The Ugly of FaCC (AKA Cons)**

When you want it to work fast, FaCC might not be performance friendly, coz changes on render.
In many cases it doesn’t matter though. react-motion uses this pattern and works fine.