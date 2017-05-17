---
layout: post
title:  "Server-Side Rendering and Code Splitting with React-Router 4 and Webpack 2"
categories: web react javascript
---

At my day job at [tyl.io](https://tyl.io), I often encounter interesting challenges. We've had a universal, code-split app for a while now, and as part of our latest performance release (out soon!), we've decided to upgrade to the new React-Router 4.

While studying the new docs to familiarize myself with the new API, I encountered this paragraph:

> # Code-splitting + server rendering
> We’ve tried and failed a couple of times. What we learned:
>
> 1. You need synchronous module resolution on the server so you can get those bundles in the initial render.
> 1. You need to load all the bundles in the client that were involved in the server render before rendering so that the client render is the same as the server render. (The trickiest part, I think its possible but this is where I gave up.)
> 1. You need asynchronous resolution for the rest of the client app’s life.
>
> We determined that google was indexing our sites well enough for our needs without server rendering, so we dropped it in favor of code-splitting + service worker caching. Godspeed those who attempt the server-rendered, code-split apps.

Challenge accepted! Let's go through these, one at a time.

# Synchronous module resolution on the server.

As the new React-Router uses plain components now, you can't use the async `match` function from version 3 anymore. No more `getComponent` on routes either, everything is synchonous now. I'll spare you the different things I tried, let's get straight to the solution.

I already had a particular Webpack configuration: I actually have *two* configs, one with a few entry points for the client, and another with one entry point for the server. I did this previously because I wanted to properly leverage the pretty substantial ecosystem of Webpack loaders (for styles, images, fonts, etc.) while still server-rendering my app. That allowed me to get rid of a few hacks and babel-loader on the server which is nice, but that's not the point.

I basically did this:

- Moved all my async `import`s to one file (easy, I was pretty much set up that way already)
- Created a synchronous version of that file, with the same exports.
- Used Webpack's `NormalModuleReplacementPlugin` to replace the synchronous version on the server with the async version for the client, or vice-versa

Enough rambling, let's look at some code!

```javascript
// AsyncBundles.js

import asyncComponent from './lib/asyncComponent';

export const Landing = asyncComponent(() => import('./routes/landing'));
export const Dashboard = asyncComponent(() => import('./routes/dashboard'));
/* ... other route components */
```

```javascript
// Bundles.js

import syncComponent from './lib/syncComponent';

export const Landing = syncComponent(require('./routes/landing'));
export const Dashboard = syncComponent(require('./routes/dashboard'));
/* ... other route components */
```

So that's pretty straightforward. Let's use them!

```javascript
// routes.js

import {Landing, Dashboard} from './Bundles';

export const routes = [{
  component: Landing,
  path: '/',
  exact: true
}, {
  component: Dashboard,
  path: '/dashboard'

  /* ... other routes */
}]
```

You'll notice here that I use [static routes](https://reacttraining.com/react-router/web/guides/static-routes). This helps a lot with server-side rendering, especially when you need to load data to render (as most apps do). This works with [React Router Config](https://github.com/reacttraining/react-router/tree/master/packages/react-router-config), but we won't go in depth about that here. I'll keep it for another blog post!

```javascript
// webpack.config.js

const webpack = require('webpack');

const clientConfig = {
  // Client build configuration
  plugins: [
    // ...
    new webpack.NormalModuleReplacementPlugin(
      /\/Bundles.jsx/,
      './AsyncBundles.js'
    ),
  ]
};

const serverConfig = {
  // Server build configuration
};

module.exports = [clientConfig, serverConfig];
```

You'll notice that in the code, I used `Bundle.js`, and I replace it with `AsyncBundles.js` using Webpack for the client. I could very well have done it the other way around, I simply picked that way.

You might also have noticed that I have referenced two files that I still haven't shown. We'll look at them in detail in the next section, but for now let's just say that `./lib/syncComponent` is a simple higher-order-component (HOC) that renders its given component, and `./lib/asyncComponent` is another HOC heavily inspired from [this](https://gist.github.com/acdlite/a68433004f9d6b4cbc83b5cc3990c194).

By now, we should have a server-rendered, code-split app that pretty much works, but there is an annoying flash of empty content once React does the initial render on the client-side, while it downloads additional chunks. There is also this annoying warning from React about a checksum mismatch. Let's fix that!

# Loading all necessary chunks _before_ the initial client-side render

The solution I found for that is surprisingly simple:

- On the server, have our `syncComponent` HOC store which components are rendered somewhere we can find it after rendering (React-Router's `StaticRouter` [passes a `staticContext` prop to routes](https://reacttraining.com/react-router/web/guides/server-rendering) so I just used that, but a Redux store, or even plain React context are good choices too).
- Have that data sent to the client somehow (again, a Redux store or similar is a good choice for that).
- Load the relevant chunks client-side
- Wait until all chunks have loaded before rendering.

Something that helps a lot here is to name your chunks. With that, and [assets-webpack-plugin](https://www.npmjs.com/package/assets-webpack-plugin), we can proactively insert script tags in our initial server response to kick-start the loading of the relevant chunks.

First, we'll need to modify our Bundle files a bit:

```javascript
// AsyncBundles.js

// ...
export const Landing = asyncComponent('Landing', () => import(/* webpackChunkName: "Landing" */'./routes/landing'));
// ...
```

It's a bit annoying that Webpack doesn't give us a way to access the chunk's name in code, so we have to specify it twice. Although it's on the same line, so it shouldn't be too hard to maintain.

```javascript
// Bundles.js

// ...
export const Landing = syncComponent('Landing', require('./routes/landing'));
// ...
```

Now, let's take a look at `syncComponent`:

```javascript
// ./lib/syncComponent.js

function syncComponent(chunkName, mod) {
  const Component = mod.default ? mod.default : mod; // es6 module compat

  function SyncComponent(props) {
    if(props.staticContext.splitPoints) {
      props.staticContext.splitPoints.push(chunkName);
    }

    return (<Component {...props} />);
  }

  SyncComponent.propTypes = {
    staticContext: React.PropTypes.object,
  };

  return SyncComponent;
}
```

As you can see, very little magic here. By the way, [as they mention in the docs](https://reacttraining.com/react-router/web/guides/server-rendering/adding-app-specific-context-information), that `staticContext` prop is a wonderful way to have your route components pass useful information for the server response up the chain.

On the server, we can basically do this:

```javascript
// server.js

import React from 'react';
import ReactDOMServer from 'react-dom/server';
import {StaticRouter} from 'react-router';
import {renderRoutes} from 'react-router-config';

import {routes} from './routes';

// Let's assume this is an Express app
app.use((req, res) => {
  const context = {
    splitPoints: [], // Create an empty array
  };
  const markup = ReactDOMServer.renderToString(
    <StaticRouter
      location={req.url}
      context={context}
    >
      {renderRoutes(routes)}
    </StaticRouter>
  )
  // now context.splitPoints contains the names of the chunks we used during rendering

  const html = `
    <!doctype html>
    <html>
      <head>
        <title>My App</title>
      </head>
      <body>
        <div id="app">${markup}</div>
        <script>
          window.splitPoints=${JSON.stringify(context.splitPoints)}; // Send it down to the client
        </script>
      </body>
    </html>
  `;
  res.send(html);
});
```

With that, the client will receive a server-rendered response, and it will also know which chunks were used to render it. Now is also the time to look up your chunk names in the JSON generated [assets-webpack-plugin](https://www.npmjs.com/package/assets-webpack-plugin) and write the appropriate script tags in your response; it'll speed things up a little.

On the client, it's pretty simple, too:

```javascript
// client.js

import React from 'react';
import ReactDOM from 'react-dom';
import {BrowserRouter} from 'react-router-dom';
import {renderRoutes} from 'react-router-config';

import * as Bundles from './Bundles'; // Replaced by AsyncBundles by webpack at build-time
import {routes} from './routes.js';

const splitPoints = window.splitPoints || [];
Promise.all(splitPoints.map(chunk => Bundles[chunk].loadComponent())) // This is the important part
  .then(() => {
      const mountNode = document.getElementById('app');
      ReactDOM.render(
        <BrowserRouter>
          {renderRoutes(routes)}
        </BrowserRouter>,
        mountNode
      );
    });
```

As you can see, it's a pretty standard React entry point, but with a single twist: we wait for all promises to resolve before rendering. These promises are returned by the `loadComponent` function on `asyncComponent`.

Now, let's take a look at the last piece of our puzzle:

```javascript
// lib/asyncComponent.js

import React from 'react';

function asyncComponent(chunkName, getComponent) {
  return class AsyncComponent extends React.Component {
    static Component = null;

    static loadComponent() { // The function we call before rendering
      return getComponent().then(m => m.default).then(Component => {
        AsyncComponent.Component = Component;
        return Component;
      });
    };

    mounted = false;

    state = {
      Component: AsyncComponent.Component
    };

    componentWillMount() {
      if(this.state.Component === null) {
        AsyncComponent.loadComponent()
        .then(Component => {
          if(this.mounted) {
            this.setState({Component});
          }
        });
      }
    }

    componentDidMount() {
      this.mounted = true;
    }

    componentWillUnmount() {
      this.mounted = false;
    }

    render() {
      const {Component} = this.state;

      if(Component !== null) {
        return (<Component {...this.props} />);
      }
      return null; // or <div /> with a loading spinner, etc..
    }
  };
}

export default asyncComponent;
```

As you can see, it's indeed heavily inspired from [this](https://gist.github.com/acdlite/a68433004f9d6b4cbc83b5cc3990c194), but we exposed the `getComponent` function through the static `loadComponent` method. The little manipulation after loading in that method is to ensure that subsequent calls to that component render synchronously.

# Aynchronous resolution for the rest of the app's life

This last item on our checklist was pretty much already working, and our `asyncComponent` ensures that it remains that way.

## Putting it all together

So there you have it! A server-rendered, code-split app with React Router 4. If you want to run the code to play with it, I've published a sample app here: [https://github.com/emilecantin/ssr_code-splitting_sample](https://github.com/emilecantin/ssr_code-splitting_sample).

If you have further questions, don't hesitate to leave a comment!

