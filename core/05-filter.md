
Filter
======

One might notice in the [Handler](03-handler.md) and [Handler Builder](04-handler-builder.md) chapters that handlers and handler builders are just plain functions without class. Therefore it is not possible to extend a handler through classical inheritance. Instead, in Quiver.js we introduce _filter_ to allow flexible handler extension through functional composition.

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

## Filter Chain

When a chain of filters is applied to a handler, the order of the filter chain matters and can affect the end result. For example consider the following filter application:

```javascript
var filteredHandlerBuilder = function(config, callback) {
  handlerBuilder(config, function(err, handler))
    if(err) return callback(err)

    filter1(config, handler, function(err, filteredHandler) {
      if(err) return callback(err)

      filter2(config, filteredHandler, callback)
    })
  })
}

filteredHandlerBuilder({}, function(err, handler) {
  ...
})
```

For simplicity, we can annotate such filter composition as `filter2(filter1(handler))`. `filter2` captures the filtered handler produced by `filter1`. Therefore the input transformation of `filter2` comes _before_ `filter1`, and result transformation of `filter2` comes _after `filter1`. As a result depending on whether a filter perform input or result or both transformation, some thinking is required to make sure the order of filter composition is correct.

## HTTP Filter

The above examples are mostly for writing _stream filters_, which extend functionality of stream handlers. It is of course also possible to write http filters_ that extend functionality of http handlers:

```javascript
var friendlyErrorRedirectionHttpFilter = function(config, httpHandler, callback) {
  var filteredHttpHandler = function(requestHead, requestStreamable, callback) {
    httpHandler(requestHead, requestStreamable, function(err, responseHead, responseStreamable) {
      if(!err) return callback(null, responseHead, responseStreamable)

      var newResponseHead = {
        statusCode: 500,
        headers: {
          'Content-Type': 'text/html'
        }
      }

      var newResponseStreamable = streamConvert.textToStreamable(
        'Oops, something has gone wrong. Please <a href="contact"> contact us for any enquiry.')

      callback(null, newResponseHead, newResponseStreamable)
    })
  }

  callback(null, filteredHttpHandler)
}
```

The above http filter intercepts a generic error generated by the inner http handler and turn it into a HTTP 500 response with friendly error page.

Quiver http filter is simple but powerful construct that can be used to build the same intermediary chains exist in other powerful web servers such as Apache. The major difference is that HTTP filters are plain Javascript functions that can easily be manipulated. Filters can also be easily enabled, disabled, or rearranged by simply manipulating the filter composition chain.

Nevertheless, it is recommended to write http filter only when the functionality is related to HTTP. Otherwise stream filter serve as much simpler alternative for implementing application-specific concerns.

We will refer to stream handler and http handler collectively as simply _handler_. The same as all higher order handler constructs such as handler builder and filter.

## Next: [Middleware](06-middleware.md)