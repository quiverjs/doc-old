
Stream
======

Quiver stream is designed to be a simple alternative of node stream in order to allow the construction of streams from plain Javascript objects. Consider the following hello function written using conventional node stream:

```javascript
var hello = function(nodeWriteStream) {
  nodeWriteStream.write('hello world')
  nodeWriteStream.end()
}
```

The function above is hardly composable as the result is written to the write stream directly without any chance for modification. There is no way to intercept the stream result and do operation such as toUpperCase, and there is no way to recover from error produced by the hello function. A better way to write such function is to make it return a _read stream_ instead of accepting a write stream and writing to it. The same function written in using quiver stream would be as follow:

```javascript
var streamChannel = require('quiver-stream-channel')

var hello = function(callback) {
  var channel = streamChannel.createStreamChannel()
  channel.writeStream.write('hello world')
  channel.writeStream.closeWrite()

  callback(null, channel.readStream)
}
```

The `quiver-stream-channel` library allow creation of a pair of connected quiver read/write streams. The stream channel acts like a simple single-producer/single-consumer pipeline. Anything that is written to the writeStream side of the channel are readable from the readStream at the other end. The read/write stream pair are also made of distinct Javascript objects, i.e. it is not possible to cast a quiver read stream to its write stream counterpart or vice versa.

Instead of creating stream channel manually in each function, the `quiver-stream-convert` library is provided to make it easy to convert simple Javascript objects such as string and json to quiver read streams:

```javascript
var streamConvert = require('quiver-stream-convert')

var hello = function(callback) {
  callback(null, streamConvert.textToStream('hello world'))
}
```

The hello function written using quiver stream provide the flexibility of intercepting the results and change it to something else. For instance one could write a upper case stream transform function as follow:

```javascript
var toUppercase = function(readStream, callback) {
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
  callback(null, channel.readStream)
}
```

The transform function follows the same quiver style of writing result through a read stream result in async callback, instead of writing the result to a write stream directly. The uppercase function also accept an incoming readStream for it to perform transformation to the original stream. The quiver style of transforming data is by defining an async-recursive doPipe() function to continuously read data from a read stream and writing the transformed result to the write stream of newly created stream channel.

Unlike node streams, quiver streams have very few methods that focus only on the core function of transferring data between a producer and consumer. The main method for quiver `readStream` is the `read()` method that return either a streamClosed object or data buffer as read result. The `streamClosed` object can be any thruthy Javascript object used to indicate that all data have been read from the readStream. `streamClosed` also have a special attribute `err`, which indicates that the readStream was prematurely closed with error if the `err` attribute is truthy.

By providing result stream in asynchronous callback, it is also made possible to differentiate overall error in the processing function from stream errors. One can for instance write a wrapper that intercepts errors from the original function and return alternative result stream:

```javascript
var errorFriendly = function(callback) {
  hello(function(err, readStream) {
    if(err) return callback(null, 
      streamConvert.textToStreamable('Oops something went wrong!'))

    callback(null, readStream)
  })
}
```

Having both hello and toUpppercase function allow us to easily compose the two functions together as follow:

```javascript
var uppercaseHello = function(callback) {
  hello(function(err, readStream) {
    if(err) return callback(err)

    toUppercase(readStream, callback)
  })
}
```

Alternatively we can make use of higher order functions and write a combine function that combine a transform function with a source function:

```javascript
var combine = function(sourceFunc, transformFunc) {
  var combinedFunc = function(callback) {
    sourceFunc(function(err, readStream) {
      if(err) return callback(err)
      
      transformFunc(readStream, callback)  
    })
  }
}

var uppercaseHello = combine(hello, toUppercase)
```


## API Specification

### quiver-stream-channel

For full documentation visit the [quiver-stream-channel](https://github.com/quiverjs/stream-channel) repository.

```javascript
var streamChannel = require('quiver-stream-channel')

var channel = streamChannel.createStreamChannel()
var readStream = channel.readStream
var writeStream = channel.writeStream
```

Create a pair of connected read/write stream through the `quiver-stream-channel` library. The two end points are usually then passed to different functions that are responsible for the read or write operation. Content written into the write stream will be read from the read stream at the other end.


```javascript
readStream.read(readCallback)
```

Attempt to read data from a readStream, which the read result will be pass to readCallback asynchronously.


```javascript
readCallback(streamClosed, data)
```

`streamClosed` is thruthy if when the end of stream is reached. If error occured during read, the error value can be retrieved from `streamClosed.err`. Otherwise the data variable contain the read value, which is typically a single buffer object. `streamClosed` and data exist exclusively from each other, which mean if is streamClosed exists then data is guaranteed to be null.


```javascript
readStream.closeRead(err)
```

Close the read stream optionally with an error value.


```javascript
writeStream.write(data)
```

Write data to a write stream immediately regardless of whether the reader on the other side is ready to read it. Calling this without prepareWrite gives the risk of having too much data buffered within the stream without reader reading it quick enough.


```javascript
writeStream.prepareWrite(writeCallback)
```

Prepare to write to the write stream once the reader on the other side is ready, in which writeCallback will be called asynchronously. It is prohibited to call the write stream's `write()` method while `prepareWrite()` is waiting for its write callback. Such action do not have well defined semantics to the stream channel state and will cause an exception to be thrown.


```javascript
writeCallback(streamClosed)
```

Non-null streamClosed indicates that the stream has been prematurely closed and the writer should cancel all write operations. Otherwise it is now optimal for the writer to call `writeStream.write()`.


```javascript
writeStream.closeWrite(err)
```

Close the write stream with an optional error value.


### quiver-stream-convert

For full documentation visit the [quiver-stream-convert](https://github.com/quiverjs/stream-convert) repository.

```javascript
var streamConvert = require('quiver-stream-convert')

var textStream = streamConvert.textToStream(text)
var jsonStream = streamConvert.jsonToStream(json)

```

Create quiver read stream from Javascript string or plain json object.


## Stream Extension

Quiver streams are specially made to be _not_ extensible. The read/write streams created from `quiver-stream-channel` are made of plain Javascript objects. There is no base class for one to extend the stream functionality. Instead Quiver.js encourage the use of composition to add functionalities to a stream object. It is also easy to create new stream implementation from scratch, thanks to the minimal number of quiver stream API.

Here are some examples of how to extend stream functionalities through functional composition: 

  - [quiver-pipe-stream](https://github.com/quiverjs/split-stream) provide simple way of pipe data from read stream to write stream.
  - [quiver-split-stream](https://github.com/quiverjs/split-stream) duplicates a read stream to be read by multiple readers.
  - [quiver-stream-timeout](https://github.com/quiverjs/stream-timeout) shows how to set a timeout for non-responding streams.
  - [quiver-stream-throttle](https://github.com/quiverjs/stream-throttle) shows how to easily implement speed throttling to a stream. 

On implementing native stream sources such as file read stream, conventional techniques make use of inheritance to implement a conrete subclass of a `ReadStream` class. Such technique make sense from the library user's perspective, but makes it difficult for implementors to implement the subclass correctly. 

Quiver.js have more natural way to implement such native streams. Instead of writing subclasses library authors are encouraged to simply make use of the quiver streams created from `quiver-stream-channel`. The write stream part is retained by the library code to read data from native APIs and write to it, while the read stream part is returned to the library user to consume the data.

There is currently lack of example of implementing native streams, because Quiver.js make use of existing node streams instead of reinventing the wheel. However the following modules show how native node streams are wrapped to become quiver streams.

  - [quiver-node-stream](https://github.com/quiverjs/node-stream) library creates quiver stream equivalent from a node stream. 
  - [quiver-file-stream](https://github.com/quiverjs/file-stream) acts as a thin wrapper around node's `fs` module to create quiver streams from filesystem using `quiver-node-stream`.

## Next: [Streamable](02-streamable.md)