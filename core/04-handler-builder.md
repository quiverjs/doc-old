
Handler Builder
===============

The [Handler](03-handler.md) chapter introduced the quiver stream handler and HTTP handler for writing composable handler functions. While it is useful to pass handler functions around and compose them, one would find that handler implementations typically have dependency on application-wide configurations such as database credentials and file system paths. Before getting the handlers we need a way for the handlers to receive these configurations without exposing them on global scope - in other words we need a constructor.

_Handler builder_ provides a uniform way of configuring handlers:

```javascript
var handlerBuilder = function(config, function(err, handler) { })
```

A Handler builder accepts a plain Javascript object `config` and asynchronously return a handler instance. The `config` parameter should contain all parameters the handler needs, and the handler builder creates a handler function closure that captures the configurations. Following is an example of a greet handler builder with custom greeting word:

```javascript
var greetingHandlerBuilder = function(config, callback) {
  var greetingWord = config.greetingWord || 'hello'

  var handler = function(args, inputStreamable, callback) {
    streamConvert.streamableToText(inputStreamable, function(err, name) {
      if(err) return callback(err)

      var greeting = greetingWord + ', ' + name

      callback(null, streamConvert.textToStreamable(greeting))
    })
  }

  callback(null, handler)
}
```

Handler builder also solves the dependency injection problem. When a handler function depends on other functions which in turn have their own configuration dependencies, the handler should not require any knowledge of how to supply the configuration or instantiate its dependencies. Instead the handler expects instantiated dependencies in its config which should allow it to make use of its dependencies immediately. For example consider the following code:

```javascript
var handlerBuilder = function(config, callback) {
  var database = createDatabase(config.dbUrl, config.dbUsername, config.dbPassword)

  ...
}
```

The handler builder have hard dependency on the database constructor, as it needs to know the dependencies of the database constructor and know how to build the database instance correctly. Instead the handler builder should move the constructor dependency outside and make use of a database instance directly:


```javascript
var handlerBuilder = function(config, callback) {
  var database = config.database

  ...
}
```

Later chapters will show examples of handlers having dependency on other handlers. In such cases the quiver component system configure each handlers externally and pass instantiated handlers to to the handler builder that depend on them. We will also show that it is better practice to compose on handler builders instead of handler instances themselves. This makes it possible to combine many quiver components down to one handler builder with no configuration known ahead, such as during script startup time.

## Next: [Filter](05-filter.md)