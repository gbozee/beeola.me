---
title: React.lazy! Friend or Foe
date: "2019-03-27"
spoiler: Coping with context switching as a full stack web developer
draft: false
keywords: frontend-development, loading, suspense, project complexity, webpack, files structure
---

Recent versions of [ReactJS](https://reactjs.org/docs/code-splitting.html#reactlazy), i.e **from 16.6.0 to date**, introduced `React.lazy()`, a new api for handling dynamic loading of components.

While this helps in ensuring small bundles, the complexity of the project grows. 

## Complexity case 1
Compare
```jsx
import React from 'react';
import { Route, BrowserRouter as Router } from "react-router-dom";
import LoadableComponent from './my-component';

const App = () => {
  return (
    <Router>
      <Route
        path="/holla"
        component={LoadableComponent} />
      ...
    </Router>
  );
};
```

to

```jsx
import React from 'react';
import Loading from "./my-loading-component";
import { Route, BrowserRouter as Router } from "react-router-dom";

const LoadableComponent = React.lazy(()=>import("./my-component"));

const App = () => {
  return (
    <Router>
      <Route
        path="/holla"
        render={props => (
          <React.Suspense fallback={<Loading />}>
            <LoadableComponent {...props} />
          </React.Suspense> )}
        />
      ...
    </Router>
  );
};
```

>This isn't a critique to `React.lazy`, it is just something to be aware of as a developer especially when working with in a team with varying grasp/understanding of how ReactJS works.

[Error boundaries](https://reactjs.org/docs/error-boundaries.html) would also need to be introduced since there is the possibility of modules failing to load as a result of network outage.

The implementation looks as follows
```jsx
import React from 'react';
import Loading from "./my-loading-component";
import { Route, BrowserRouter as Router } from "react-router-dom";

const LoadableComponent = React.lazy(()=>import("./my-component"));

class ErrorBoundary extends React.Component{
    state = {
        hasError:false
    }
    static getDerivedStateFromError(error) {
        // Update state so the next render will show the fallback UI.
        return { hasError: true };
    }
    componentDidCatch(){
        logErrorToMyService(error, info);
    }
    render() {
    if (this.state.hasError) {
      // You can render any custom fallback UI
      return <h1>Something went wrong.</h1>;
    }

    return this.props.children; 
  }
}

const App = () => {
  return (
      <ErrorBoundary>
        <Router>
          <Route
            path="/holla"
            render={props => (
              <React.Suspense fallback={<Loading />}>
                <LoadableComponent {...props} />
              </React.Suspense> )}
            />
        ...
        </Router>
      </ErrorBoundary>
  );
};
```
[https://reactjs.org/docs/error-boundaries.html](https://reactjs.org/docs/error-boundaries.html)

## Complexity case 2
This complexity is further compounded when dealing with **Named Export**

Compare this
```jsx
import React from 'react';
import { Route, BrowserRouter as Router } from "react-router-dom";
import {LoadableComponent} from './my-component';

const App = () => {
  return (
    <Router>
      <Route
        path="/holla"
        component={LoadableComponent} />
      ...
    </Router>
  );
};
```

to 

```jsx
// l-my-components.js
export { LoadableComponent as default } from "./my-component";
```
```jsx
import React from 'react';
import Loading from "./my-loading-components";
import { Route, BrowserRouter as Router } from "react-router-dom";

const LoadableComponent = React.lazy(()=>import("./l-my-component"));
...
const App = () => {
  return (
      <ErrorBoundary>
        <Router>
          <Route
            path="/holla"
            render={props => (
              <React.Suspense fallback={<Loading />}>
                <LoadableComponent {...props} />
              </React.Suspense> )}
            />
        ...
        </Router>
      </ErrorBoundary>
  );
};
```
[https://reactjs.org/docs/code-splitting.html#named-exports](https://reactjs.org/docs/code-splitting.html#named-exports)

Because `React.lazy` only supports **default exports**, a temporary file is required to make the **named export** a **default export**, as well as ensuring that tree shaking occurs at the webpack layer.

> Like I mentioned earlier, this is a price you end up paying for better loading performance.

## Complexity case 3
The cherry at the top, when it comes to complexity, is when control as to when the modules are fetched, is required. Since [Webpack](https://webpack.js.org/guides/code-splitting/#prefetchingpreloading-modules), which [Create React App]() is built on top of, supports both prefetching and preloading  of JavaScript modules, We would need to annotate our code with special comments, in dynamic `import` functions, that lets Webpack know how it should generate the bundles and also determine what point the bundles are to be loaded, i.e either at idle time (**prefetch**) or during current navigation (**preload**)

```jsx
import React from 'react';
import Loading from "./my-loading-components";
import { Route, BrowserRouter as Router } from "react-router-dom";

const LoadableComponent = React.lazy(()=>import(/*webpackPrefetch: true */"./l-my-component"));
...
const App = () => {
  return (
      <ErrorBoundary>
        <Router>
          <Route
            path="/holla"
            render={props => (
              <React.Suspense fallback={<Loading />}>
                <LoadableComponent {...props} />
              </React.Suspense> )}
            />
        ...
        </Router>
      </ErrorBoundary>
  );
};

```

In a large codebase, where performance is important, the codebase would be filled with such annotation comments. Knowing which loading strategy to use, is a deciding factor as regards to loading performance.


## In summary

Thinking about loading performance is almost as hard as building features of the application. It appears impractical to obsess over loading performance when the application isn't functional yet as it requires fundamental understanding of the following

1. **Default Exports** and **Named Exports**: this dictates how one structures code in a JavaScript file
2. **Webpack**: Knowing how the module should be loaded i.e *(prefetch, preload)*, naming the modules *(named chunks)* etc

`React.lazy` appears to be a friend when it comes to improving loading performance of our applications. But it appears to be an enemy, when it comes to keeping the codebase simple.

Does this mean it is impossible for a *Performant ReactJS Application* to be simple? 