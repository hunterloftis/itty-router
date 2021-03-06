![Logo][logo-image]

[![npm package][npm-image]][npm-url]
[![minified + gzipped size][gzip-image]][gzip-url]
[![Build Status][travis-image]][travis-url]
[![Coverage Status][coveralls-image]][coveralls-url]
[![Open Issues][issues-image]][issues-url]

It's an itty bitty router. Like... super tiny, with zero dependencies. For reals.

Did we mention it supports route/query params like in Express.js and was built specifically for use in Workers (e.g. [Cloudflare Workers](https://developers.cloudflare.com/workers/))?

## Installation

```
yarn add itty-router
```

or if you've been transported back to 2017...
```
npm install itty-router
```

## Features
- [x] tiny (~430 bytes)
- [x] zero dependencies!
- [x] dead-simple usage
- [x] route params, with optionals (e.g. `/api/:foo/:id?`)
- [x] bonus query parsing (e.g. `?page=3`)
- [x] adds params & query to request: `{ params: { foo: 'bar' }, query: { page: '3' }}`
- [x] multiple (sync or async) [middleware handlers](#multiple-route-handlers-as-middleware) per route for passthrough logic, auth, errors, etc
- [x] handler functions "stop" at the first handler to return anything at all
- [x] supports [nested routers](#nested-routers)
- [x] supports [base path](#base-path) option to prefix all routes
- [x] chainable route declarations (why not?)
- [ ] have pretty code (yeah right...)

# Examples
### Kitchen Sink
```js
import { Router } from 'itty-router'

// create a Router
const router = Router()

// basic GET route (with optional ? flag)... delivers route params and query params as objects to handler
router.get('/todos/:id?', ({ params, query }) =>
  console.log('matches /todos or /todos/13', `id=${params.id}`, `limit=${query.limit}`)
)

// first match always wins, so be careful with order of registering routes
router
  .get('/todos/oops', () => console.log('you will never see this, thanks to upstream /todos/:id?'))
  .get('/features/chainable-route-declaration', () => console.log('yep!'))
  .get('/features/:optionals?', () => console.log('check!')) // will match /features and /features/14 both

// works with POST, DELETE, PATCH, etc
router.post('/todos', () => console.log('posted a todo'))

// ...or any other method we haven't yet thought of (thanks to @mvasigh implementation of Proxy <3)
router.future('/todos', () => console.log(`this caught using the FUTURE method!`))

// then handle a request!
router.handle({ method: 'GET', url: 'https://foo.com/todos/13?foo=bar' })

// ...and viola! the following payload/context is passed to the matching route handler:
// {
//   params: { id: '13' },
//   query: { foo: 'bar' },
//   ...whatever else was in the original request object/class (e.g. method, url, etc)
// }
```

# Usage
### 1. Create a Router
```js
import { Router } from 'itty-router'

const router = Router() // no "new", as this is not a real ES6 class/constructor!
```

### 2. Register Route(s)
##### `.{methodName}(route:string, handler1:function, handler2:function, ...)`
The "instantiated" router translates any attribute (e.g. `.get`, `.post`, `.patch`, `.whatever`) as a function that binds a "route" (string) to route handlers (functions) on that method type (e.g. `router.get --> GET`, `router.post --> POST`).  When the url fed to `.handle({ url })` matches the route and method, the handlers are fired sequentially.  Each is given the original request/context, with any parsed route/query params injected as well.  The first handler that returns (anything) will end the chain, allowing early exists from errors, inauthenticated requests, etc.  This mechanism allows ANY method to be handled, including completely custom methods (we're very curious how creative individuals will abuse this flexibility!).  The only "method" currently off-limits is `handle`, as that's used for route handling (see below).
```js
// register a route on the "GET" method
router.get('/todos/:user/:item?', (req) => {
  let { params, query, url } = req
  let { user, item } = params

  console.log('GET TODOS from', url, { user, item })
})
```

### 3. Handle Incoming Request(s)
##### `.handle(request = { method:string = 'GET', url:string })`
The only requirement for the `.handle(request)` method is an object with a valid **full** url (e.g. `https://example.com/foo`).  The `method` property is optional and defaults to `GET` (which maps to routes registered with `router.get()`).  This method will return the first route handler that actually returns something.  For async/middleware examples, please see below.
```js
router.handle({
  method: 'GET',                              // optional, default = 'GET'
  url: 'https://example.com/todos/jane/13',   // required
})

// matched handler from step #2 (above) will execute, with the following output:
// GET TODOS from https://example.com/todos/jane/13 { user: 'jane', item: '13' }
```

# Examples
### Within a Cloudflare Function
```js
import { Router } from 'itty-router'

// create a router
const router = Router() // note the intentional lack of "new"

// register some routes
router
  .get('/foo', () => new Response('Foo Index!'))
  .get('/foo/:id', ({ params }) => new Response(`Details for item ${params.id}.`))
  .get('*', () => new Response('Not Found.', { status: 404 }))

// attach the router handle to the event handler
addEventListener('fetch', event => event.respondWith(router.handle(event.request)))
```

### Multiple Route Handlers as Middleware
###### Note: Any of these handlers may be awaitable async functions!
```js
// withUser modifies original request, then continues without returning
const withUser = (req) => {
  req.user = { name: 'Mittens', age: 3 }
}

// requireUser optionally returns (early) if user not found on request
const requireUser = (req) => {
  if (!req.user) return new Response('Not Authenticated', { status: 401 })
}

// showUser returns a response with the user, as it is assumed to exist at this point
const showUser = (req) => new Response(JSON.stringify(req.user))

router.get('/pass/user', withUser, requireUser, showUser) // withUser injects user, allowing requireUser to not return/continue
router.get('/fail/user', requireUser, showUser) // requireUser returns early because req.user doesn't exist

router.handle({ url: 'https://example.com/pass/user' }) // --> STATUS 200: { name: 'Mittens', age: 3 }
router.handle({ url: 'https://example.com/fail/user' }) // --> STATUS 401: Not Authenticated
```

### Multi-route (Upstream) Middleware
```js
// withUser modifies original request, then continues without returning
const withUser = (req) => {
  req.user = { name: 'Mittens', age: 3 }
}

router.get('*', withUser) // embeds user before all other matching routes
router.get('/user', (req) => new Response(JSON.stringify(req.user))) // user embedded already!

router.handle({ url: 'https://example.com/user' }) // --> STATUS 200: { name: 'Mittens', age: 3 }
```

### Nested Routers
```js
  const parentRouter = Router()
  const todosRouter = Router()

  todosRouter.get('/todos/:id?', ({ params }) => console.log({ id: params.id }))

  parentRouter.get('/todos/*', todosRouter.handle) // all /todos/* routes will route through the todosRouter
```

### Base Path
```js
  const router = Router({ base: '/api/v0' })

  router.get('/todos', indexHandler)
  router.get('/todos/:id', itemHandler)

  router.handle({ url: 'https://example.com/api/v0/todos' }) // --> fires indexHandler
```


## Testing & Contributing
1. fork repo
2. add code
3. run tests (add your own if needed) `yarn dev`
4. verify tests run once minified `yarn verify`
5. commit files (do not manually modify version numbers)
6. submit PR
7. we'll add you to the credits :)

## The Entire Code (for more legibility, [see src on GitHub](https://github.com/kwhitley/itty-router/blob/v1.x/src/itty-router.js))
```js
const Router = (o = {}) =>
  new Proxy(o, {
    get: (t, k, c) => k === 'handle'
      ? async q => {
          for ([p, hs] of t[(q.method || 'GET').toLowerCase()] || []) {
            if (m = (u = new URL(q.url)).pathname.match(p)) {
              q.params = m.groups
              q.query = Object.fromEntries(u.searchParams.entries())

              for (h of hs) {
                if ((s = await h(q)) !== undefined) return s
              }
            }
          }
        }
      : (p, ...hs) =>
          (t[k] = t[k] || []).push([
            `^${(o.base || '')+p
              .replace(/(\/?)\*/g, '($1.*)?')
              .replace(/(\/:([^\/\?]+)(\?)?)/g, '/$3(?<$2>[^/]+)$3')
            }$`,
            hs
          ]) && c
  })
```

## Special Thanks
This repo goes out to my past and present colleagues at Arundo - who have brought me such inspiration, fun,
and drive over the last couple years.  In particular, the absurd brevity of this code is thanks to a
clever [abuse] of `Proxy`, courtesy of the brilliant [@mvasigh](https://github.com/mvasigh).
This trick allows methods (e.g. "get", "post") to by defined dynamically by the router as they are requested,
**drastically** reducing boilerplate.

[twitter-image]:https://img.shields.io/twitter/url?style=social&url=https%3A%2F%2Fwww.npmjs.com%2Fpackage%2Fitty-router
[logo-image]:https://user-images.githubusercontent.com/865416/79531114-fa0d8200-8036-11ea-824d-70d84164b00a.png
[gzip-image]:https://img.shields.io/bundlephobia/minzip/itty-router
[gzip-url]:https://bundlephobia.com/result?p=itty-router
[issues-image]:https://img.shields.io/github/issues/kwhitley/itty-router
[issues-url]:https://github.com/kwhitley/itty-router/issues
[npm-image]:https://img.shields.io/npm/v/itty-router.svg
[npm-url]:http://npmjs.org/package/itty-router
[travis-image]:https://travis-ci.org/kwhitley/itty-router.svg?branch=v1.x
[travis-url]:https://travis-ci.org/kwhitley/itty-router
[david-image]:https://david-dm.org/kwhitley/itty-router/status.svg
[david-url]:https://david-dm.org/kwhitley/itty-router
[coveralls-image]:https://coveralls.io/repos/github/kwhitley/itty-router/badge.svg?branch=v1.x
[coveralls-url]:https://coveralls.io/github/kwhitley/itty-router?branch=v1.x

# Contributors
These folks are the real heroes, making open source the powerhouse that it is!  Help us out and add your name to this list!

#### Core, Concepts, and Codebase
- [@mvasigh](https://github.com/mvasigh) - proxy hacks courtesy of this chap
- [@roojay520](https://github.com/roojay520) - TS interface fixes
#### Documentation Fixes
- [@arunsathiya](https://github.com/arunsathiya)
