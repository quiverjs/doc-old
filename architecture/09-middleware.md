# Middleware

![Middleware](figures/middleware-1.png)

Filters are great at extending handlers from the outside, but the handlers they receive are blackboxes that already been instantiated. As a result, a filter has no way to know how the handler itself was instantiated, or how many other filters have been applied to a handler before it arrive. Filter is designed to be easily composable, but there is a problem of ensuring a filter is only applied once in a composition chain.

To solve the problem, interception need to be done at handler construction time and alter the way handler is instantiated. _Middleware_ is the superset of filter that does the extension both before and after handler construction. 

Unlike filter, a middleware accepts a _handler builder_ in place of handler in its second argument. In other words, it has full power to alter the how a handler builder receive its _configuration_, as well as extending the produced handler result.

With it, filter become a simplified middleware that only intercept on the result produced by a handler builder. It is trivial to convert a filter into middleware by making a converted middleware that manipulate handler result returned from handler builder.

## API Specification

```javascript
  api middleware = function(config, handlerBuilder, 
    callback(err, handler));
```

## Dependency management

Since middleware are capable of altering the `config` parameter before forwarding it to the handler builder, it is also useful for dependency management by instantitating services the handler builder depend on and putting them into the config.

```javascript
  var databaseMiddleware = function(config, handlerBuilder, callback) {
    config.database = createDatabase(config.dbUrl, 
      config.dbUsername, config.dbPassword)

    handlerBuilder(config, callback)
  }
```

The example code above is a database middleware which instantiates a database based on credentials specified in the config. It then injects the database instance into the config and pass it to the handler builder so that it can make use of the database immediately.

Internally, Quiver make extensive use of middleware to preconfigure a user-written handler builder or middleware. For instance it allow filters to depend on other filters while making sure that each filter are only applied once at its outermost composition chain. Middleware also enable Quiver to provide dependency injection service by instantiating other handlers that a handler depend on and injecting it into their `config`.

Middleware are also easily composable with handler builder to create new handler builder:

```javascript
  var combineMiddlewareWithHandlerBuilder = 
    function(middleware, handlerBuilder) {
      return function(config, callback) {
        middleware(config, handlerBuilder, callback)
      }
    }
```

## Handleable Middleware

In Quiver handler builder and middleware are the cornerstone building block for all quiver components. Using composition techniques it is possible to reduce a complex handler network graph all the way down to one encapsulated handler builder, all without knowing configuration ahead of time.

The quiver component system convert all components into handleable builder or handleable middleware. This greatly simplifies the component system as at the highest level it only have to handle two component supertype.

## Next: [Component](10-component.md)