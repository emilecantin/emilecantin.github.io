---
layout: post
title:  "Universal React with Fluxible.io and React-Router"
categories: web react javascript
---

## Universal React with Fluxible.io and React-Router

I've been working with React for a while now, so I figured I'd blog about it a little, and how I've used it in a major project.

Back when I started, the [Flux](http://facebook.github.io/flux/) architecture was already a popular way of doing React. There were (and still are) many implementations of it, but it wasn't yet clear which one would end up being the most popular. I ended up choosing [Fluxible](http://fluxible.io/) because it's apparently well-suited to server-side rendering, which has been an obsession of mine for several years (I even wrote an [ORM for Backbone](https://github.com/emilecantin/backbone-relational-mapper) so I could use it on the server).

Of course, to build any serious app, you also need a router, and [React-Router](https://github.com/reactjs/react-router) looked like the most powerful solution around, so I used it.

It didn't take very long with both of these to see that merely following the tutorials wasn't going to work. You see, they both tell you that they want to wrap your app at its root, but it isn't clear which one should wrap which. Moreso, the examples that Fluxible provides with React-Router either don't work with server-side rendering, or are very hard to follow.

What I ended up doing was pretty simple once it clicked, but it took a while. There a few moving parts here. First, you need to follow React-Router's advice, and drop one level below their `<Router />` API. The process is pretty well documented [here](https://github.com/reactjs/react-router/blob/master/docs/guides/ServerRendering.md), but I'll reproduce the basic setup here:

```javascript
// server/render.js
import { renderToString } from 'react-dom/server'
import { match, RouterContext } from 'react-router'
import routes from '../client/routes'

// let's imagine this is an express app
app.use((req, res, next) => {
  // Note that req.url here should be the full URL path from
  // the original request, including the query string.
  match({ routes, location: req.url }, (error, redirectLocation, renderProps) => {
    if (error) {
      res.status(500).send(error.message);
    } else if (redirectLocation) {
      res.redirect(302, redirectLocation.pathname + redirectLocation.search);
    } else if (renderProps) {
      res.status(200).send(renderToString(<RouterContext {...renderProps} />));
    } else {
      next(); // no route was matched
    }
  })
});
```

To add Fluxible to the mix, we need a few things. First, we need to create our Fluxible _app_, as instructed in [Fluxible's documentation](http://fluxible.io/api/fluxible.html). The only twist is that we'll use our routes as our app's component:

```javascript
// client/app.jsx
import Fluxible from 'fluxible';

import routes from './routes'; // Our routes definitions, basically just this:
/*
  const routes = (
    <Route blabla>
      <Route blablabla />
      <Route blablibla />
    </Route>
  );
*/

import MyFirstStore from './stores/MyFirstStore';
import MySecondStore from './stores/MySecondStore';

const app = new Fluxible({
  component: routes,
  stores: [
    MyFirstStore,
    MySecondStore
  ]
});
export default app;
```

Then we need to render our app's component instead of the routes directly:

```javascript
// server/render.js

import app from '../client/app';

// ...
app.use((req, res, next) => {
  const routes = app.getComponent(); // <==========
  match({ routes, location: req.url }, (error, redirectLocation, renderProps) => {
// ...
```

And to create a Fluxible _context_ once we've done the routing:

```javascript
// server/render.js

// ...
    } else if (renderProps) {
      const context = app.createContext(); // <==========
      res.status(200).send(renderToString(<RouterContext {...renderProps} />));
// ...
```

Of course we still need to actually provide our context to our components. Luckily, Fluxible provides a handy little [`provideContext`](http://fluxible.io/addons/provideContext.html) function exactly for this purpose:

```javascript
// server/render.js

import provideContext from 'fluxible-addons-react/provideContext';

// ...
    } else if (renderProps) {
      const context = app.createContext();
      res.status(200).send(
        renderToString(
          React.createElement( // Use the non-JSX syntax
            provideContext(RouterContext), // Wrap RouterContext with provideContext
            {context: context.getComponentContext()} // Pass our Fluxible context as the 'context' prop
          )
        )
      );
// ...
```

This should work pretty well, but we still need one last item in this puzzle: getting the state to the client.

```javascript
import { renderToString } from 'react-dom/server'
import { match, RouterContext } from 'react-router'
import routes from './routes'

// let's imagine this is an express app
app.use((req, res, next) => {
  // Note that req.url here should be the full URL path from
  // the original request, including the query string.
  match({ routes, location: req.url }, (error, redirectLocation, renderProps) => {
    if (error) {
      res.status(500).send(error.message);
    } else if (redirectLocation) {
      res.redirect(302, redirectLocation.pathname + redirectLocation.search);
    } else if (renderProps) {
      res.status(200).send(renderToString(<RouterContext {...renderProps} />));
    } else {
      next(); // no route was matched
    }
  })
});
```
