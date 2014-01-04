
Handleable
==========

There are currently two types of handlers in Quiver.js: stream handler and http handler. Handler types are differentiated based on their function signature, and there is no class hierarchy or any common feature shared among handlers. There are times when one might want to create a handler that is usable as both stream and http handler, or write filters or middlewares that can be applied to both types of handlers. 

We might also want to have way to attach meta information about a handler, such as serializing it to URL or filesystem path. How about the problem of ensuring filters or middlewares are only applicable to the right handler type? The answer of all these is that Quiver.js adopt the same pattern of creating [streamable](02-streamable.md) and use it to create _handleable_

Like streamable, a handleable is a plain Javascript object that has `toXXX()` methods to convert it to the right handler type. But unlike streamable, handleable convertion function return in synchronous style.

```javascript
var streamHandler = function(args, inputStreamable, callback) { ... }

var handleable = {
  toStreamHandler: function() {
    return streamHandler
  }
}
```

Handleable makes it possible to create "handlers" that are both stream handler and http handler.

```javascript
var streamHandler = function(args, inputStreamable, callback) { ... }
var httpHandler = function(requestHead, requestStreamable, callback) { ... }

var handleable = {
  toStreamHandler: function() {
    return streamHandler
  },
  toHttpHandler: function() {
    return httpHandler
  }
}
```

## Handleable Builder

Since handleable is just simple wrapper around handler, it is simple to convert a handler builder into _handleable builder_

```javascript
var streamHandlerBuilder = function(config, callback) { ... }

var handleableBuilder = function(config, callback) {
  streamHandlerBuilder(config, function(err, streamHandler) {
    if(err) return callback(err)

    var handleable = {
      toStreamHandler: function() {
        return streamHandler
      }
    }

    return handleable
  })
}
```

## Handleable Middleware

To ensure that filters and middlewares are applied on the right handler type, Quiver.js provides tools to convert filter and middleware into _handleable middleware_:

```javascript
var myStreamFilter = function(config, handler, callback) { ... }

var myStreamFilterMiddleware = function(config, handleableBuilder, callback) {
  handleableBuilder(config, function(err, handleable) {
    if(err) return callback(err)

    if(!handleable.toStreamHandler) return callback(error(500, 'mismatch handler type'))

    myStreamFilter(config, handleable.toStreamHandler(), function(err, filteredHandler) {
      if(err) return callback(err)

      var filteredHandleable = streamHandlerToHandleable(filteredHandler)
      callback(null, filteredHandleable)
    })
  })
}
```

## Handleable Extension

The design of handleable enable possibility for one to "serialize" a handler by sending its location to some client. For example a file handler may have a `toFilePath()` method on its handleable so that local clients can make optimization in accessing the handler content.

```javascript
var fileHandleableBuilder = function(config, callback) {
  var filePath = config.filePath

  var handler = function(args, inputStreamable, callback) {
    ...
  }

  var handleable = {
    toStreamHandler: function() {
      return handler
    },
    toFilePath: function() {
      return filePath
    }
  }
}
```

The handleable extension is currently still in experimental design. The protocol for how extension methods should behave is still in development and not available for general usage.

## Next: [Component](08-component.md)