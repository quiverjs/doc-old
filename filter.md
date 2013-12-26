
Filter
======

One might notice in the [Handler](handler.md) and [Handler Builder](handler-builder.md) chapters that handlers and handler builders are just plain functions without class. Therefore it is not possible to extend a handler through classical inheritance. Instead, in Quiver.js we introduce _filter_ to allow flexible handler extension through functional composition.

```javascript
var filter = function(config, handler, function(err, filteredHandler) { })
```

A filter have similar signature as handler builder, except that it has an additional parameter that accepts the original handler that it extends. Based on the supplied handler and the filter's configuration dependencies, the filter create an extended _filteredHandler_ closure to replace the original handler.

For example, the following filter converts a handler result stream to uppercase:

```javascript
var uppercaseResultFilter = function(config, handler, callback) {
  var filteredHandler = function(args, inputStreamable, callback) {
    handler(args, inputStreamable, function(err, resultStreamable) {
      if(err) return callback(err)

      streamConvert.streamableToText(resultStreamable, function(err, resultText))
        if(err) return callback(err)

        callback(null, streamConvert.textToStreamable(resultText.toUpperCase()))
      })
    })
  }

  callback(null, filteredHandler)
}
```

The filter accepts its original handler as a black box but has full power to access all its input and output. In other words, it can make changes to `args` and `inputStreamable` before passing it to the original handler, and it can intercept `err` and `resultStreamable` produced by the original handler before returning to its caller. Moreover, a filter can decide to skipe the original handler entirely and produce result on behalf of it.

Filter solves the problem of separation of concern by allowing developers to implement cross-cutting concerns as reusable filters. One can implement any of the `before()`, `after()`, or `around()` paradigms in Aspect Oriented programming all in one unified function interface.

For instance, one could implement a permission filter easily as follow:

```javascript
var permissionFilter = function(config, handler, callback) {
  var allowedUser = config.allowedUser

  var filteredHandler = function(args, inputStreamable, callback) {
    if(args.user != allowedUser) return callback(error(403, 'Forbidden'))

    handler(args, inputStreamable, callback)
  }

  callback(null, filteredHandler)
}
```

Or one could implement a friendly error logger filter:

```javascript
var errorFilter = function(config, handler, callback) {
  var logger = config.logger

  var filteredHandler = function(args, inputStreamable, callback) {
    handler(args, inputStreamable, function(err, resultStreamable) {
      if(!err) return callback(null, resultStreamable)

      logger.logError(err)
      callback(null, streamConvert.textToStreamable('Oops something went wrong!'))
    })
  }

  callback(null, filteredHandler)
}
```