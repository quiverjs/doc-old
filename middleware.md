
Middleware
==========

[Filters](filter.md) are great at extending handlers from the outside, but the handlers they receive are blackboxes that already been instantiated. As a result, a filter has no way to know how the handler was instantiated, or how many other filters have been applied to a handler before it arrive. Filter is designed to be easily composable, but how is it possible to ensure a filter is only applied once in a composition chain?

The answer to that is that we need to intercept at _handler construction_ time and alter the way handler is instantiated. _Middleware_ is the superset of filter that does the job:

```javascript
var middleware = function(config, handlerBuilder, function(err, handler) { })
```

Unlike filter, middleware function accepts a _handler builder_ in place of handler in its second argument. In other words, it has full power to alter the how a handler builder receive its _configuration_, as well as extending the produced handler result.

All filters are just simplified middleware that only intercept on the result produced by a handler builder. They can easily be converted to middleware with the following helper function:

```javascript
var createMiddlewareFromFilter = function(filter) {
  var middleware = function(config, handlerBuilder, callback) {
    handlerBuilder(config, function(err, handler) {
      if(err) return callback(err)

      filter(config, handler, callback)
    })
  }

  return middleware
}
```

Since middleware are capable of altering the `config` parameter before forwarding it to the handler builder, it is also useful for dependency management by instantitating services the handler builder depend on and putting them into the config.

```javascript
var databaseMiddleware = function(config, handlerBuilder, callback) {
  config.database = createDatabase(config.dbUrl, config.dbUsername, config.dbPassword)

  handlerBuilder(config, callback)
}
```

Internally, Quiver.js make extensive use of middleware to preconfigure a user-written handler builder or middleware. For instance it allow filters to depend on other filters while making sure that each filter are only applied once at its outermost composition chain. Middleware also enable Quiver.js to provide dependency injection service by instantiating other handlers that a handler depend on and injecting it to config.

Middleware are also easily composable with handler builder to create new handler builder:

```javascript
var combineMiddlewareWithHandlerBuilder = function(middleware, handlerBuilder) {
  var combinedHandlerBuilder = function(config, callback) {
    middleware(config, handlerBuilder, callback)
  }

  return combinedHandlerBuilder
}
```

In Quiver.js handler builder and middleware are the cornerstone building block for all quiver components. Using the composition technique we can reduce a complex handler network graph all the way down to one encapsulated handler builder, all without knowing configuration ahead of time. The [quiver-middleware](https://github.com/quiverjs/middleware) library provide example middlewares that Quiver.js use internally to build the [quiver component](component.md) architecture, which we will discuss next.