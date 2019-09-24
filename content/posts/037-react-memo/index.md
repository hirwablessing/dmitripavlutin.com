---
title: Use React.memo() wisely
description: "React.memo() increases the performance of functional components by preventing useless re-renders. But such performance tweaks must be applied wisely."
published: "2019-07-17T11:30:00Z"
modified: "2019-08-27T06:26Z"
thumbnail: "./images/instruments.jpg"
slug: use-react-memo-wisely
tags: ["react", "component", "memoization"]
recommended: ["7-architectural-attributes-of-a-reliable-react-component", "solve-react-render-props-callback-hell"]
type: post
---

Users enjoy fast and responsive user interfaces (UI). An UI response delay of less than 100 milliseconds feels instant to the user. A delay between 100 and 300 milliseconds is already perceptible.  

To improve user interface performance, React offers a higher-order component `React.memo()`. By memoizing the rendered output, React skips unnecessary re-rendering.  

This post helps you distinguish the situations when `React.memo()` improves the performance, and, not less important, understand when its usage is useless.  

Plus I'll describe some useful memoization tips you should be aware of.  

## 1. React.memo()

When deciding to update DOM, React first renders your component, then compares the result with the previous render. If the render results are different, React decides to update the DOM.  

Current vs previous render results comparison is fast. But you can *speed up* the process under some circumstances.  

When a component is wrapped in `React.memo()`, React renders the component and memoizes the result. Before the next render, if the props are the same, React reuses the memoized content.  

Let's look at a React memo example. The functional component `Movie` is wrapped in `React.memo()`:  

```jsx
export function Movie({ title, releaseDate }) {
  return (
    <div>
      <div>Movie title: {title}</div>
      <div>Release date: {releaseDate}</div>
    </div>
  );
}

export const MemoizedMovie = React.memo(Movie);
```

`React.memo(Movie)` returns a new memoized component `MemoizedMovie`. It will output the same content as the original `Movie` component, but with one difference. 

`MemoizedMovie` render output is memoized. The memoized content is reused as long as `title` or `releaseDate` props are the same during the next rendering rounds.  

```jsx{3-4,10-11}
// First render. React calls MemoizedMovie function.
<MemoizedMovie 
  movieTitle="Heat" 
  releaseDate="December 15, 1995" 
/>

// On next round React does not call MemoizedMovie function,
// preventing re-render
<MemoizedMovie
  movieTitle="Heat" 
  releaseDate="December 15, 1995" 
/>
```

This is where you gain performance benefit: by reusing the memoized result, React skips re-rendering the component and doesn't perform a virtual DOM difference check.  

The same functionality for class components is implemented by [PureComponent](https://reactjs.org/docs/react-api.html#reactpurecomponent).  

### 1.1 Custom equality check of props

By default `React.memo()` does a [shallow](https://github.com/facebook/react/blob/v16.8.6/packages/shared/shallowEqual.js) comparison of props and objects of props. 

You can use the second argument to indicate a custom equality check function:  

```javascript
React.memo(Component, [areEqual(prevProps, nextProps)]);
```

`areEqual(prevProps, nextProps)` function must return `true` if `prevProps` and `nextProps` are equal.

For example, let's manually calculate if `Movie` component props are equal:

```jsx
function moviePropsAreEqual(prevMovie, nextMovie) {
  return prevMovie.title === nextMovie.title
    && prevMovie.releaseDate === nextMovie.releaseDate;
}

const MemoizedMovie2 = React.memo(Movie, moviePropsAreEqual);
```

`moviePropsAreEqual()` function returns `true` if prev and next props are equal.  

## 2. When to use React.memo()

![Inforgraphic explaining when to use React.memo()](./images/when-to-use-react-memo-infographic.jpg)

### 2.1 Component renders often with the same props

`React.memo()` is applied on functional components that given the same props, render the same output.  

The best case of wrapping a component in `React.memo()` is when you expect the functional component to render often and usually with the same props.  

A common situation that makes a component re-render with the same props is being forced to re-render by a parent component.  

Let's reuse `Movie` component defined above. A new parent component `MovieViewsRealtime` displays the number of views of a movie, with realtime updates:  

```jsx{4}
function MovieViewsRealtime({ title, releaseDate, views }) {
  return (
    <div>
      <Movie title={title} releaseDate={releaseDate} />
      Movie views: {views}
    </div>
  );
}
```
The application regularly polls the server in background (every second), updating `views` property of `MovieViewsRealtime` component.  

```jsx{3,10,17}
// Initial render
<MovieViewsRealtime 
  views={0} 
  title="Forrest Gump" 
  releaseDate="June 23, 1994" 
/>

// After 1 second, views is 10
<MovieViewsRealtime 
  views={10} 
  title="Forrest Gump" 
  releaseDate="June 23, 1994" 
/>

// After 2 seconds, views is 25
<MovieViewsRealtime 
  views={25} 
  title="Forrest Gump" 
  releaseDate="June 23, 1994" 
/>

// etc
```

Every time `views` prop is updated with a new number, `MovieViewsRealtime` re-renders. This triggers `Movie` re-render too, regardless of `title` and `releaseDate` being the same.  

That's the right case to apply memoization on `Movie` component.  

Improve `MovieViewsRealtime` to use the memoized component `MemoizedMovie`:

```jsx{4}
function MovieViewsRealtime({ title, releaseDate, views }) {
  return (
    <div>
      <MemoizedMovie title={title} releaseDate={releaseDate} />
      Movie views: {views}
    </div>
  )
}
```

As long as `title` and `releaseDate` props are the same, React skips re-rendering of `MemoizedMovie`. This improves the performance of `MovieViewsRealtime` component.  

> The more often the component re-renders with the same props, the heavier and the more computationally expensive the output is, the more chances are that component needs to be wrapped in `React.memo()`  

Anyways, use [profiling](https://reactjs.org/docs/optimizing-performance.html#profiling-components-with-the-chrome-performance-tab) to measure the benefits of applying `React.memo()`.   
## 3. When to avoid React.memo()

If your component's rendering situation doesn't fit into the case described above, most likely you don't need `React.memo()`.  

Use the following rule of thumb: don't use memoization if you can't quantify the performance gains.  

> Performance-related changes applied incorrectly can even harm performance. Use `React.memo()` wisely.  

While technically possible, it doesn't make much sense to wrap class-based components in `React.memo()`. Just extend `PureComponent` class or define a custom implementation of `shouldComponentUpdate()` method if you need memoization for class-based components.  

### 3.1 Useless props comparison

Imagine a component typically renders with different props. In this case, memoization doesn't provide benefits. 

Even if you wrap such a volatile component in `React.memo()`, React will have to do 2 jobs on every re-render:

1. Invoke the comparison function to determine whether the previous and next props are equal
2. Because props comparison almost always returns `false`, React performs the diff of previous and current render output

The invocation of the comparison function is useless because it almost always returns `false`.  

## 4. React.memo() and callback functions

Function objects follow the same principles of comparison as "regular" objects. The function object equals only to itself.  

Let's compare some functions:
```javascript{8}
function sumFactory() {
  return (a, b) => a + b;
}

const sum1 = sumFactory();
const sum2 = sumFactory();

console.log(sum1 === sum2); // => false
console.log(sum1 === sum1); // => true
console.log(sum2 === sum2); // => true
```
`sumFactory()` is a factory function. It returns arrow functions that sum 2 numbers.  

The functions `sum1` and `sum2` are created by the factory. Both functions sum 2 numbers. However, `sum1` and `sum2` are different function objects.  

The implicit new functions creation might happen when a parent component defines a callback for its child. Let's study how this can break memoization, and how to fix it.   

The following component `Logout` accepts a callback prop `onLogout`:

```jsx
function Logout({ username, onLogout }) {
  return (
    <div onClick={onLogout}>
      Logout {username}
    </div>
  );
}

const MemoizedLogout = React.memo(Logout);
```

Because of function equality pitfall, a component that accepts a callback must be handled with care when applying memoization. There's a chance that the parent component provides different instances of the callback function on every re-render:  

```jsx{7}
function MyApp({ store, cookies }) {
  return (
    <div className="main">
      <header>
        <MemoizedLogout
          username={store.username}
          onLogout={() => cookies.clear()}
        />
      </header>
      {store.content}
    </div>
  );
}
```
Even if provided with the same `username` value, `MemoizedLogout` re-renders every time because it receives new instances of `onLogout` callback.  

Memoization is broken.  

To fix it, the same callback instance must be used to set `onLogout` prop. Let's apply [useCallback()](https://reactjs.org/docs/hooks-reference.html#usecallback) to preserve the callback instance between renderings:

```jsx{4,10}
const MemoizedLogout = React.memo(Logout);

function MyApp({ store, cookies }) {
  const onLogout = useCallback(() => cookies.clear(), [cookies]);
  return (
    <div className="main">
      <header>
        <MemoizedLogout
          username={store.username}
          onLogout={onLogout}
        />
      </header>
      {store.content}
    </div>
  );
}
```

`useCallback(() => cookies.clear(), [cookies])` always returns the same function instance as long as `cookies` is the same. Memoization of `MemoizedLogout` is fixed.  

## 5. React.memo() is a performance hint

Strictly, React uses memoization as a performance hint. 

While in most situations React avoids re-rendering a memoized component, you shouldn't count on that to prevent rendering.  

## 6. Conclusion

`React.memo()` is a great tool to gain the benefits of memoization for functional components. When applied correctly, it prevents component re-render when the next props equal previous.  

Take precaution when memoizing components that use props with callback functions. Make sure to provide the same callback function instance between renderings.  

Don't forget to use [profiling](https://reactjs.org/docs/optimizing-performance.html#profiling-components-with-the-chrome-performance-tab) to measure the performance gains of memoization.  

*Do you know interesting use cases of React.memo()? If so, please write a comment below!*