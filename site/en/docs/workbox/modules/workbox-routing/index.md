---
layout: 'layouts/doc-post.njk'
title: workbox-routing
date: 2017-11-27
updated: 2021-03-22
description: >
  Routes requests in your service worker to specific caching strategies or callback functions.
---

A service worker can intercept network requests for a page. It may respond to
the browser with cached content, content from the network or content generated
in the service worker.

`workbox-routing` is a module which makes it easy to "route" these requests to
different functions that provide responses.

## How Routing is Performed

When a network request causes a service worker fetch event, `workbox-routing`
will attempt to respond to the request using the supplied routes and handlers.

{% Img src="image/jL3OLOhcWUQDnR4XjewLBx4e3PC3/ZEakAmYC60sNemFI7KTP.png", alt="Workbox Routing Diagram", width="800", height="845" %}

The main things to note from the above are:

- The method of a request is important. By default, Routes are registered for
  `GET` requests. If you wish to intercept other types of requests, you'll need
  to specify the method.

- The order of the Route registration is important. If multiple Routes are
  registered that could handle a request, the Route that is registered first
  will be used to respond to the request.

There are a few ways to register a route: you can use callbacks, regular
expressions or Route instances.

## Matching and Handling in Routes

A "route" in workbox is nothing more than two functions: a "matching" function
to determine if the route should match a request and a "handling" function,
which should handle the request and respond with a response.

Workbox comes with some helpers that'll perform the matching and handling for
you, but if you ever find yourself wanting different behavior, writing a
custom match and handler function is the best option.

A
[match callback function](/docs/workbox/reference/workbox-core/#method-RouteMatchCallback)
is passed a
[`ExtendableEvent`](https://developer.mozilla.org/docs/Web/API/ExtendableEvent),
[`Request`](https://developer.mozilla.org/docs/Web/API/Request), and a
[`URL` object](https://developer.mozilla.org/docs/Web/API/URL) you can
match by returning a truthy value. For a simple example, you could match against
a specific URL like so:

```js
const matchCb = ({url, request, event}) => {
  return url.pathname === '/special/url';
};
```

Most use cases can be covered by examining / testing either the `url` or the
`request`.

A
[handler callback function](/docs/workbox/reference/workbox-core/#type-RouteHandler)
will be given the same
[`ExtendableEvent`](https://developer.mozilla.org/docs/Web/API/ExtendableEvent),
[`Request`](https://developer.mozilla.org/docs/Web/API/Request), and
[`URL` object](https://developer.mozilla.org/docs/Web/API/URL) along with
a `params` value, which is the value returned by the "match" function.

```js
const handlerCb = async ({url, request, event, params}) => {
  const response = await fetch(request);
  const responseBody = await response.text();
  return new Response(`${responseBody} <!-- Look Ma. Added Content. -->`, {
    headers: response.headers,
  });
};
```

Your handler must return a promise that resolves to a `Response`. In this
example, we're using
[`async` and `await`](https://developers.google.com/web/fundamentals/primers/async-functions).
Under the hood, the return `Response` value will be wrapped in a promise.

You can register these callbacks like so:

```js
import {registerRoute} from 'workbox-routing';

registerRoute(matchCb, handlerCb);
```

The only limitation is that the "match" callback **must synchronously** return a truthy
value, you can't perform any asynchronous work. The reason for this is that
the `Router` must synchronously respond to the fetch event or allow falling
through to other fetch events.

Normally the "handler" callback would use one of the strategies provided
by [workbox-strategies](/docs/workbox/modules/workbox-strategies) like so:

```js
import {registerRoute} from 'workbox-routing';
import {StaleWhileRevalidate} from 'workbox-strategies';

registerRoute(matchCb, new StaleWhileRevalidate());
```

In this page, we'll focus on `workbox-routing` but you can
[learn more about these strategies on workbox-strategies](/docs/workbox/modules/workbox-strategies).

## How to Register a Regular Expression Route

A common practice is to use a regular expression instead of a "match" callback.
Workbox makes this easy to implement like so:

```js
import {registerRoute} from 'workbox-routing';

registerRoute(new RegExp('/styles/.*\\.css'), handlerCb);
```

For requests from the
[same origin](https://developer.mozilla.org/docs/Web/Security/Same-origin_policy),
this regular expression will match as long as the request's URL matches the
regular expression.

- https://example.com**/styles/main.css**
- https://example.com**/styles/nested/file.css**
- https://example.com/nested**/styles/directory.css**

However, for cross-origin requests, regular expressions
**must match the beginning of the URL**. The reason for this is that it's
unlikely that with a regular expression `new RegExp('/styles/.*\\.css')`
you intended to match third-party CSS files.

- https://cdn.third-party-site.com**/styles/main.css**
- https://cdn.third-party-site.com**/styles/nested/file.css**
- https://cdn.third-party-site.com/nested**/styles/directory.css**

If you _did_ want this behaviour, you just need to ensure that the regular
expression matches the beginning of the URL. If we wanted to match the
requests for `https://cdn.third-party-site.com` we could use the regular
expression `new RegExp('https://cdn\\.third-party-site\\.com.*/styles/.*\\.css')`.

- **https://cdn.third-party-site.com/styles/main.css**
- **https://cdn.third-party-site.com/styles/nested/file.css**
- **https://cdn.third-party-site.com/nested/styles/directory.css**

If you wanted to match both local and third parties you can use a wildcard
at the start of your regular expression, but this should be done with caution
to ensure it doesn't cause unexpected behaviors in your web app.

## How to Register a Navigation Route

If your site is a single page app, you can use a
[`NavigationRoute`](/docs/workbox/reference/workbox-routing/#type-NavigationRoute) to
return a specific response for all
[navigation requests](https://developers.google.com/web/fundamentals/primers/service-workers/high-performance-loading#first_what_are_navigation_requests).

```js
import {createHandlerBoundToURL} from 'workbox-precaching';
import {NavigationRoute, registerRoute} from 'workbox-routing';

// This assumes /app-shell.html has been precached.
const handler = createHandlerBoundToURL('/app-shell.html');
const navigationRoute = new NavigationRoute(handler);
registerRoute(navigationRoute);
```

Whenever a user goes to your site in the browser, the request for the page will
be a navigation request and it will be served the cached page `/app-shell.html`.
(Note: You should have the page cached via `workbox-precaching` or through your
own installation step.)

By default, this will respond to _all_ navigation requests. If you want to
restrict it to respond to a subset of URLs, you can use the `allowlist`
and `denylist` options to restrict which pages will match this route.

```js
import {createHandlerBoundToURL} from 'workbox-precaching';
import {NavigationRoute, registerRoute} from 'workbox-routing';

// This assumes /app-shell.html has been precached.
const handler = createHandlerBoundToURL('/app-shell.html');
const navigationRoute = new NavigationRoute(handler, {
  allowlist: [new RegExp('/blog/')],
  denylist: [new RegExp('/blog/restricted/')],
});
registerRoute(navigationRoute);
```

The only thing to note is that the `denylist` will win if a URL is in both
the `allowlist` and `denylist`.

## Set a Default Handler

If you want to supply a "handler" for requests that don't match a route, you
can set a default handler.

```js
import {setDefaultHandler} from 'workbox-routing';

setDefaultHandler(({url, event, params}) => {
  // ...
});
```

## Set a Catch Handler

In the case of any of your routes throwing an error, you can capture and
degrade gracefully by setting a catch handler.

```js
import {setCatchHandler} from 'workbox-routing';

setCatchHandler(({url, event, params}) => {
  ...
});
```

## Defining a Route for Non-GET Requests

All routes by default are assumed to be for `GET` requests.

If you would like to route other requests, like a `POST` request, you need
to define the method when registering the route, like so:

```js
import {registerRoute} from 'workbox-routing';

registerRoute(matchCb, handlerCb, 'POST');
registerRoute(new RegExp('/api/.*\\.json'), handlerCb, 'POST');
```

## Router Logging

You should be able to determine the flow of a request using the logs from
`workbox-routing` which will highlight which URLs are being processed
through Workbox.

{% Img src="image/jL3OLOhcWUQDnR4XjewLBx4e3PC3/IufA3Cp68hdk1HTPM6uC.png", alt="Routing Logs", width="800", height="192" %}

If you need more verbose information, you can set the log level to `debug` to
view logs on requests not handled by the Router. See our
[debugging guide](/docs/workbox/troubleshooting-and-logging/) for more info on
setting the log level.

{% Img src="image/jL3OLOhcWUQDnR4XjewLBx4e3PC3/lAJuL5UQuLYiTWTp5qH4.png", alt="Debug and Log Routing Messages", width="800", height="223" %}

## Advanced Usage

If you want to have more control over when the Workbox Router is given
requests, you can create your own
[`Router`](/docs/workbox/reference/workbox-routing/#type-Router) instance and call
it's [`handleRequest()`](/docs/workbox/reference/workbox-routing/#method-Router-handleRequest)
method whenever you want to use the router to respond to a request.

```js
import {Router} from 'workbox-routing';

const router = new Router();

self.addEventListener('fetch', event => {
  const {request} = event;
  const responsePromise = router.handleRequest({
    event,
    request,
  });
  if (responsePromise) {
    // Router found a route to handle the request.
    event.respondWith(responsePromise);
  } else {
    // No route was found to handle the request.
  }
});
```

When using the `Router` directly, you will also need to use the `Route` class,
or any of the extending classes to register routes.

```js
import {Route, RegExpRoute, NavigationRoute, Router} from 'workbox-routing';

const router = new Router();
router.registerRoute(new Route(matchCb, handlerCb));
router.registerRoute(new RegExpRoute(new RegExp(...), handlerCb));
router.registerRoute(new NavigationRoute(handlerCb));
```
