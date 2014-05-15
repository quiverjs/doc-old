# Handler Builder

![Handler Builder](figures/handler-builder-1.png)

Stream handler and HTTP handler makes it easy for writing composable handler functions. While it is useful to pass handler functions around and compose them directly, one would find that handler implementations typically have dependency on application-wide configurations such as database credentials and file system paths. Before getting the handlers there is a need for the handlers to receive these configurations without exposing them on global scope. With that the _handler builder_ is designed as a construction function to provide uniform way of configuring handlers.

## API Specification

```javascript
  api handlerBuilder = function(config, callback(err, handler));
```

A Handler builder accepts a plain JavaScript object `config` and asynchronously return a handler instance. The `config` parameter should contain all parameters the handler needs, and the handler builder creates a handler function closure that captures the configurations. 

## Dependency Management

Handler builder also solves the dependency management problem. Very often different parts of web application have dependencies on some global configuration, such as database credentials. The usual solution for many web frameworks is for users to define application-wide globals to deliver the dependencies to the code in framework-specific ways.

Quiver solves the dependency problem by operating at the handler builder level, which enable explicit passing of configuration through the constructor function. With that the handler builder function can return instantiated handler function that capture the dependencies inside closures.

```javascript
  var handlerBuilder = function(config, callback) {
    var database = createDatabase(config.dbUrl, 
      config.dbUsername, config.dbPassword)

    var handler = function(args, inputStreamable, callback) {
      database.query(...)
    }

    callback(null, handler)
  }
```

The example code above shows a handler builder function that instantiate a database instance based on the database credentials provided in config. It then create a handler function closure which capture the database variable and use it when handling requests.

One problem with the example shown is that  the handler builder have hard dependency to the database constructor, as it needs to know the dependencies of the database constructor and know how to build the database instance correctly. There is a better way to handle such dependency: the handler builder can leave the database construction as external dependency and instead require dependency on instantiated database instance:


```javascript
  var handlerBuilder = function(config, callback) {
    var database = config.database
    ...
  }
```

## Handleable Builder

The handler builder constructor pattern can be applied to any kind of handler or object in general. For easy reference the unqualified name "handler builder" is used to refer to handler constructor of any kind, including stream handler builder, http handler builder, and handleable builder. 

Of all the handleable builder is used in Quiver as the supertype of all handler builders. As explained in coming sections, the quiver component system convert all handler builders into handleable builders and make use of the handleable type system to verify the type of handlers created.

## Next: [Filter](08-filter.md)