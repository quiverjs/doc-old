
Handler
=======

This is the conventional way of writing a request handler in Node.js:

```javascript
var nodeHandler = function(request, response) { 
  var user = request.user
  ...
}
```

To provide extended functionalities to the handler, developers typically attach information from intermediary code to the request object to be made use by the handler. This way of extension complicate the request object by adding unrelated attribute to a stream object. This also makes it hard to differentiate plain data from attributes inherited from the ReadStream and Request class. A better way to enable extensibility to the handler is by separating plain data into separate function argument:

```javascript
var handler = function(args, request, response) {
  var user = args.user
  ...
}
```

One issue with the above handler is that it is given the responsibility and full power of HTTP. Since HTTP is a complicated protocol, it is difficult to implement a handler that fully conform to the protocol. Moreover the result produced by the handler can only be understood by large and well implemented applications. A better way is to write handler functions that process plain stream rather than HTTP request and respons.

```javascript
var handler = function(args, nodeReadStream, nodeWriteStream) { }
```

The above code looks ok until we discover the weakness of node stream. In the [Stream](01-stream.md) chapter we learn that supplying write stream directly to functions reduce composibility and make it hard to handle errors. To solve the problem we make use of quiver stream to create a more elegant handler:

```javascript
var handler = function(args, readStream, function(err, readStream) { }) { }
```

The handler making use of quiver read stream is much more elegant, but is not efficient in serializing/deserializing existing Javascript objects to the handler function. In the [Streamabe](02-streamable.md) chapter we also learn that streamable can be used to store multiple representation of a stream for fast access of stream content. With that we finally have the quiver _stream handler_:

```javascript
var streamHandler = function(args, inputStreamable, function(err, resultStreamable) { }) { }
```

## Stream Handler

Quiver stream handler is designed following the Unix Philosophy. Unix process is well known for their extensibility through the uniform stream interfaces `STDIN`, `STDOUT`, and `STDERR`. Similarly stream handler is designed to have uniform interface across all kinds of programs to make them as easily composable as Unix processes. All quiver stream handler accept an `args` as their first argument, which is the equivalent of command line arguments. The second argument inputStreamable is a streamable object corresponds to STDIN. Finally the handler asynchronously returns either an error (STDERR) or a resultStreamable (STDOUT).

Similar to Unix pipeline, it is easy to combine multiple stream handlers to form a pipeline to perform complex functionalities. An example of writing a pipeline uppercase hello handler would be written as follow:

```javascript
var streamConvert = require('quiver-stream-convert')
var streamChannel = require('quiver-stream-channel')

var greetHandler = function(args, inputStreamable, callback) {
  var greetWord = args.greetWord || 'hello'

  streamConvert.streamableToJson(inputStreamable, function(err, json) {
    if(err) return callback(err)

    // Access key case insensitively for demonstration purpose
    var name = json.name || json.NAME
    var greeting = greetWord + ', ' + name

    callback(null, streamConvert.textToStreamable(greeting))
  })
}

var toUppercaseHandler = function(args, inputStreamable, callback) {
  inputStreamable.toStream(function(err, readStream) {
    if(err) return callback(err)

    var channel = streamChannel.createStreamChannel()
    var writeStream = channel.writeStream

    var doPipe = function() {
      readStream.read(function(streamClosed, data) {
        if(streamClosed) return writeStream.closeWrite(streamClosed.err)

        // Assume string in ASCII encoding
        writeStream.write(data.toString().toUpperCase())
        doPipe()
      })
    }

    doPipe()

    var resultStreamable = streamConvert.streamToStreamable(channel.readStream)
    callback(null, resultStreamable)
  })
}

var combinePipeline = function(handler1, handler2) {
  var pipelineHandler = function(args, inputStreamable, callback) {
    handler1(args, inputStreamable, function(err, resultStreamable) {
      if(err) return callback(err)

      handler2(args, resultStreamable, callback)
    })
  }

  return pipelineHandler
}

var allUppercaseHandler = combinePipeline(greetHandler, toUppercaseHandler)
var nameUppercaseHandler = combinePipeline(toUppercaseHandler, greetHandler)

var printResult = function(args, inputStreamable, handler) {
  handler(args, inputStreamable, function(err, resultStreamable) {
    if(err) throw err

    streamConvert.streamableToText(resultStreamable, function(err, text) {
      if(err) throw err

      console.log(text)
    })
  })
}

var args = { greetWord: 'bonjour' }
var inputStreamable = streamConvert.jsonToStreamable({ name: 'john' })

// Will print "BONJOUR, JOHN"
printResult(args, inputStreamable, allUppercaseHandler)

// Will print "bonjour, JOHN"
printResult(args, inputStreamable, nameUppercaseHandler)
```

We write the uppercase handler above in streaming fashion to demonstrate that with uniform stream interface, we can mix functions that process plain objects with functions that process streams and they would work seamlessly together through streamable conversions.

## Simple Handler

The above stream handlers look rather complicated, due to the need to convert between streamable and their deserialized forms. Fortunately the example is written that way to allow deeper understanding on the way stream handler works. In pratice The [quiver-simple-handler](https://github.com/quiverjs/simple-handler) library simplifies the creation of handler functions by converting functions that accept and return Javascript object into stream handler with uniform function interface.

```javascript
var simpleHandler = require('quiver-simple-handler')

var greetHandler = function(args, json, callback) {
  var greetWord = args.greetWord || 'hello'

  var name = json.name || json.NAME
  var greeting = greetWord + ', ' + name

  callback(null, greeting)
}

var toUppercaseHandler = function(args, text, callback) {
  callback(null, text.toUpperCase())
}

var greetStreamHandler = simpleHandler.simpleHandlerToStreamHandler('json', 'text', greetHandler)
var toUppercaseStreamHandler = simpleHandler.simpleHandlerToStreamHandler('text', 'text', toUppercaseHandler)

```

Currently the data types available for simple handler are `void`, `text`, `json`, `stream`, and `streamable`. Having void type allow the simple handler to ignore the input/result stream and omit the argument in the input or result parameter.

## HTTP Handler

Stream handler is designed to free users from the complexity of HTTP and allow handler functions to be composed more easily. While the `args` argument can loosely related to HTTP request header, in the handler callback there only exist the resultStreamable which loosely corresponds to HTTP response body but there is no correspondance of HTTP response header. The lack of response header equivalent is intentional as HTTP request and response headers alter the semantics of the input and result stream and therefore make it hard to understand.

When there are time one needs the full power HTTP, Quiver.js provides the alternative _HTTP Handler_ function signature:

```javascript
var httpHandler = function(requestHead, requestStreamable, function(err, responseHead, responseStreamable) { })
```

Unlike the original Node http handler, quiver http handler separates the HTTP header part from the HTTP body. It is designed such that it is easy for intermediaries to strip/add HTTP headers that alter the content of the body, such as Content-Encoding and Transfer-Encoding, and supply a different stream body independent of the original header.

The `requestHead` and `responseHead` format is compatible with the existing node API for the `request` object, minus the node stream and event API. They are also made of plain Javascript objects and can be easily created using object literals.

Here is an example http handler that sets cookies and return HTTP 303 redirection if the request method is POST.

```javascript
var redirectHttpHandler = function(requestHead, requestStreamable, callback) {
  if(requestHead.method != 'POST') return callback(405, 'method not allowed')

  var responseHead = {
    statusCode: 303,
    headers: {
      'Set-Cookie': 'key=value',
      'Location': '/path/to/redirect'
    }
  }

  // empty streamable represents empty response body
  var responseStreamable = streamChannel.createEmptyStreamable()
  callback(null, responseHead, responseStreamable)
}
```

In the above example the HTTP 405 error is returned as a simple error object instead of as full HTTP response. In the handler callback the error object is retained so that HTTP handlers can make a quick return in the case of internal errors. Returning error is different from returning explicit status such as 500, as it indicates that the handler is explicitly _undecisive_ on how to render the error as HTTP response. Therefore this allows intermediaries to intercept the error and provide customized HTTP error response back to the client.

## Next: [Handler Builder](04-handler-builder.md)