---
layout: single
title: React Rendering Optimization
categories: React
tag: [React, Optimization]
toc: true
toc_sticky: true
---

## How the Browser renders a web page

In short, the browser parses HTML into DOM and CSS into CSSOM, and then it draws layout and get painted.

Then, what would be rendering in React?

`Rendering in React === calling functions`

## Code example

Here are some example codes to provide better understanding on how actually React rendering works.

### Parent

```jsx
function Parent() {
	const [value, setValue] = useState(null);

	const handleClick = () => {};

	useEffect(() => {
		setTimeout(() => {
			setValue('changedValue');
		}, 3000);
	}, []};

	return (
		<>
			<FirstChild value={value} />
			<SecondChild onClick={handleClick} />
		</>
	);
)
```

In 3 seconds, the Parent component will be re-rendered by change of state ‘value’.

### FirstChild

```jsx
function FirstChild({ value }) {
  return <div>{value}</div>;
}
```

The FirstChild component will be also re-rendered since it gets updated state value from the Parent component.

### SecondChild

```jsx
function SecondChild({ onClick }) {
	return (
		<div onClick={onClick}>
			{Array.from({ length: 1000 }).map((_, idx) => {
				<GrandChild key={idx + 1} order={idx} />
		</div>
	};
}
```

The SecondChild component isn’t related to the updated state, but it will be also re-rendered since its parent component is re-rendered. This is where the non-necessary rendering takes place.

## How to prevent non-necessary rendering

Before we dive into how to prevent non-necessary rendering, we need to be reminded with conditions where rendering takes place. There are two conditions triggering rendering.

- change on state
- change on props

From the code example, `FirstChild` was re-rendered since the props value has been changed. `SecondChild` only gets onClick function from `Parent`, which didn’t change. So, we might think it shouldn’t be re-rendered, but it does.

The reason why `SecondChild` is re-rendered is because every time when `Parent` is re-rendered, handleClick function is also re-generated. The newly generated handleClick function has different reference value from the previous one, since the function in JavaScript is reference type data.

Here you might think if the reference value does not change, can we prevent the non-necessary rendering?

In order to maintain the same reference value during re-rendering, `useCallback()` feature can be an option. `useCallback()` is a hook for function memoization that allows you to assign the same function to variables unless there is change on dependency. Memoization is the method to save the result from the previous operation and reuse it.

## How to apply useCallback

```jsx
const handleClick = useCallback(() => {}, []);
```

Simply wrap the function body with useCallback() function.

## After applying useCallback

But even after applying `useCallback()`, we can still see the `GrandChild` being re-rendered.

Let's see what exactly happened.

### Parent component(compiled by Babel)

```jsx
function Parent() {
	const [value, setValue] = useState(null);

	const handleClick = () => {};

	useEffect(() => {
		setTimeout(() => {
			setValue('changedValue');
		}, 3000);
	}, []};

	return React.createElement(
		React.Fragment,
		null,
		React.createElement(FirstChild, {
			value: value,
		}},
		React.createElement(SecondChild, {
			onClick: handleClick,
		}}
	);
}
```

When we compile the previous code by Babel, we can see there is React.createElement statement for each child component. So, the `SecondChild` couldn’t avoid re-rendering, even if we applied useCallback and transferred the same function to it.

Does this mean `useCallback()` didn’t affect at all?

To answer this question, we need to look into rendering process of React.

## Rendering in React

React rendering has the following two main phases.

- Render Phase
- Commit Phase

### Render Phase

In render phase, React calls components and return React elements, and generates the virtual DOM. If this isn’t the first rendering, React implements reconciliation of the virtual DOM and checks what to reflect on the real DOM. Reconciliation is a process of simply comparing the current virtual DOM with the previous one.

### Commit Phase

The commit phase is where we actually reflect the change we confirmed from the render phase.

In short, we make an update list in render phase and apply them in commit phase. In the previous example, we kept the same props by using `useCallback()` function, so when we compared DOMs, there wasn’t any change to apply, and we could skip the commit phase, which means `useCallback()` worked partially.

Then in this case, is there a way to prevent even the render phase?

Yes, we can use another React tool called `React.memo`.

## React.memo

```jsx
function Component({ content }) {
  return <p>{content}</p>;
}

export default React.memo(Component);
```

`React.memo` is a higher order component which prevent re-rendering and reuse the previous rendering result, in case of the current props and the previous props are considered the same. React implements the shallow copy when it compares the props.

The shallow copy compares values for the primitive type data and reference value for the reference type data.

We can also specify our own comparison function and pass that function as the second parameter of `React.memo`.

Let’s apply `React.memo` on `SecondChild` component.

```jsx
function SecondChild({ onClick }) {
	return (
		<div onClick={onClick}>
			{Array.from({ length: 1000 }).map((_, idx) => {
				<GrandChild key={idx + 1} order={idx} />
		</div>
	);
}

export default React.memo(SecondChild);
```

In above example, `React.memo` compares reference value of the onClick function with the previous one, before `SecondChild` gets into the render phase. If the reference value is the same, then it prevents the render phase.

As mentioned before, React compares the reference values for the reference type data. So, if `SecondChild` recieves object as a prop, which is being re-rendered with `Parent` component, the reference value of the object will continuously change and `React.memo` will not work.

## useMemo()

In that case, we can use `useMemo()` hook. As `useCallback()` is memoization hook for functions, useMemo() is memoization hook for values. Likewise, if the dependency stays the same, `useMemo()` returns the same value.

```jsx
function Parent() {
	const [value, setValue] = useState(null);

	const user = {
		id: 'John',
		pw: '1234',
	};

	const memoizedUser = useMemo(() => user, []);

	useEffect(() => {
		setTimeout(() => {
			setValue('changedValue');
		}, 3000);
	}, []};

	return (
		<>
			<FirstChild value={value} />
			<SecondChild user={memoizedUser} />
		</>
	);
)
```

If the value is the same, then `useMemo()` maintains and returns the same reference value, allowing React.memo to be applied and eventually preventing of both render and commit phase.

## Summary

Even if `useCallback(), useMemo(), React.memo` helps optimization, preventing unnecessary re-rendering, we must consider that those functions also take considerable resources. So, deciding optimal place to apply those features must be considered first.
