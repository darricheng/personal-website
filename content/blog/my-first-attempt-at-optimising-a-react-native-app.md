+++
title = "My First Attempt at Optimising a React Native App"
date = 2023-10-28T16:55:13+08:00
draft = false
description = ""
slug = "my-first-attempt-at-optimising-a-react-native-app"
languages = ["javascript", "react"]
topic = ["performance optimisation"]
+++

A while back, I was tasked with improving the performance of our mobile app. Performance optimisation is not a new thing to me, as I've seen YouTubers doing various kinds of performance benchmark comparisons of different tools. However, it was my first time trying to optimise a real production app, and I found it fun! I detail some of my investigations and discoveries during the process, and share what I learnt from this experience.

For context, we ship React Native applications that enable our staff to complete work more effectively. However, the mobile devices that run the app are low-powered devices compared to the Apple iPhones and Samsung Galaxies that we have gotten used to. What seems fast on high-end consumer devices can take seconds on our devices. When working, they go through certain pages hundreds of times a day, so it can be very frustrating to have to wait for seconds every time they use those pages' functions. So I kicked off an investigation into our application to figure out what made our app so slow. There were two main points I discovered that improved the performance of our app by up to 50% on some pages.

## Component Structure

How components are grouped into parent-child relationships and where state is managed. `useState` is a hook that triggers the component to render again when the respective `setState` function is called. All the child components nested within that same component will also render again. Such re-renders allow React to update the DOM with the new state values.

However, these renders can be computationally expensive on slower devices, especially when large portions of the DOM are rendered at once. Using the Profiler in React DevTools, I discovered that there was a RootNavigator component rendering every time our staff was performing their duties with our app. Notably, it was typically the same two states that were triggering the render. The RootNavigator is where we define the routes and pages for the entire app, so rendering it is an expensive operation as multiple large child components are also rendered by the state update.

Why was this happening? The first step was to figure out what state was being updated. For the RootNavigator, these were the `user` and `ui` states, for managing information related to the user and user interface respectively. We'll start with the `ui` state.

The `ui` state managed the rendering of components related to managing user interactions with the app. Notably, we had a loading screen overlay that would be triggered whenever there are long operations taking place, such as getting data from a server or posting data mutations to a database. The loading screen covers the entire app and prevents any form of user interaction while the operation in the background completes its execution.

Typically, rendering one overlay component like this isn't an expensive operation. However, when the loading screen state is managed by the RootNavigator, the expense of this operation goes through the roof. Because the `useState` call for the `ui` state was made in RootNavigator, React will re-render the entire RootNavigator component instead of just the overlay. For a simple operation that happens so often in the app, we were doing way too much unnecessary computation work.

```jsx
function RootNavigator() {
  const [ui, setUi] = useState({
    showOverlay: false,
  });
  // lots of code...
  return (
    <>
      {/* Lots of heavy components 
			that will be rendered when the ui state is changed */}
      {ui.showOverlay && <LoadingOverlay />}
    </>
  );
}
```

Simply extracting the loading screen component into a container that manages the `ui` state preserves the overlay functionality and removes all the computational overhead from unnecessary renders of the heavy RootNavigator component.

```jsx
function RootNavigator() {
  // lots of code...
  return (
    <>
      {/* Lots of heavy components 
			that are no longer related to the ui state */}
      <LoadingOverlayContainer />
    </>
  );
}

function LoadingOverlayContainer() {
  const [ui, showUi] = useState({
    showOverlay: false,
  });
  // The ui state only triggers renders of
  // the component it is responsible for
  return <>{ui.showOverlay && <LoadingOverlay />}</>;
}
```

## State Updates

With the above code refactor, changes to the `ui` state no longer caused unnecessary renders of heavy components. However, the RootNavigator was still being updated because of changes to the `user` state.

Or at least that's what it looked like from the React DevTools profiler data. Upon further investigation, the user state wasn't being updated at all. There were calls to `setUser()`, but the object in the state had no changes. Yet, the profiler showed that the RootNavigator was being updated due to changes in the `user` state.

The following code snippet renders SomeComponent with a dummy object as its state. We use `useRef` to count the number of times SomeComponent renders.

```jsx
const { useState, useRef } = React;

const SomeComponent = () => {
  // The next two lines are for counting the number of renders
  const renderCounter = useRef(0);
  renderCounter.current = renderCounter.current + 1;

  const [someState, setSomeState] = useState({ someKey: 0 });
  const clickHandler = () => {
    console.log(someState);
    const obj = structuredClone(someState);
    setSomeState(obj);
  };
  return (
    <div>
      <h1>Renders: {renderCounter.current}</h1>
      <button onClick={clickHandler}>Click me</button>
    </div>
  );
};

ReactDOM.render(<SomeComponent />, document.getElementById("root"));
```

If you run the code snippet, you'll notice that clicking on the button will trigger a re-render of the component (the renderCounter should increment with each click), even though we did not change any value in the state at all! We simply do a deep clone of the existing state, then call `setSomeState` with the clone as its only argument. What's going on here?

Behind the hood, React makes use of [`Object.is()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is) to determine whether the old and new states are the same, triggering a re-render if they are different. There are [great](https://www.valentinog.com/blog/react-object-is/) [articles](https://blog.bitsrc.io/understanding-referential-equality-in-react-a8fb3769be0) out there that go into more detail on this. The main point is that objects are compared by reference, not value. That means a new object, even with exactly the same values, are different to React because they point to different locations in memory! So React triggers a re-render of the component.

How do we resolve this? Simple. Whenever we call `setState` with an unmodified state, we pass the previous state as an argument instead of a copy. Since the previous state points to the same reference in memory, React determines that the state is the same, thus doesn't trigger any renders of the component.

As an aside, our state was actually managed with Redux. I chose to use React's built-in `useState` hook to demonstrate the example in a simpler manner, but the underlying concepts are the same. When no state change happens, the reducer should return the initial state so that [no renders are triggered](https://redux.js.org/faq/react-redux#why-is-my-component-re-rendering-too-often).

While it might sound ridiculous to call `setState` when the state is unchanged, this was exactly what our application was doing. As we are using Redux, the reducer function was being called with an action type that did not modify the state. Pinpointing exactly where these calls were being triggered is no simple task. It would also necessitate too large of a change to try and reduce the number of reducer calls that don't change any state. Instead, it made more sense to simply alter how we manage the cases where no state change happens, limiting the blast radius should any bug be introduced by the change.

## What I learnt

In today's age of devices, we get used to handheld devices that have blazingly fast chips, such that sometimes we forgo performance in the name of development speed. Today's smartphones have more transistors, and thus more processing power than supercomputers of decades past.

When I look at tutorials and guides on the internet, I seldom see serious mention about performance. I think this is understandable given the computing power that we have immediate access to at our fingertips. However, I don't think that means we should completely forgo performance or take for granted that the devices will make our apps fast thanks to sheer computing power.

No matter the amount of computing power available, it is limited. Even though most of our devices are able to power through inefficient code, I think it is important to still keep performance in mind. There are people out there that do not have access to the latest devices, and will get bad experiences from the inefficient apps that we create. Performance might not be our top priority, but I'll start taking note of it in all of the code that I write moving forward.
