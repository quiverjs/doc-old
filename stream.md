
Stream
======

Quiver stream is designed to be a simple alternative of node stream in order to allow the construction of streams from plain Javascript objects. Consider the following hello function written using conventional node stream:

    var hello = function(nodeWriteStream) {
      nodeWriteStream.write('hello world')
      nodeWriteStream.end()
    }

The function above is hardly composable as the result is written to the write stream directly without any chance for modification. There is no way to intercept the stream result and do operation such as toUpperCase, and there is no way to recover from error produced by the hello function. A better way to write such function is to make it return a _read stream_ instead of accepting a write stream and writing to it. The same function written in using quiver stream would be as follow:

    var streamChannel = require('quiver-stream-channel')

    var hello = function(callback) {
      var channel = streamChannel.createStreamChannel()
      channel.writeStream.write('hello world')
      channel.writeStream.closeWrite()

      callback(null, channel.readStream)
    }

The `quiver-stream-channel` library allow creation of a pair of connected quiver read/write streams. The stream channel acts like a simple single-producer/single-consumer pipeline. Anything that is written to the writeStream side of the channel are readable from the readStream at the other end. The read/write stream pair are also made of distinct Javascript objects, i.e. it is not possible to cast a quiver read stream to its write stream counterpart or vice versa.

Instead of creating stream channel manually in each function, the `quiver-stream-convert` library is provided to make it easy to convert simple Javascript objects such as string and json to quiver read streams:

    var streamConvert = require('quiver-stream-convert')

    var hello = function(callback) {
      callback(null, streamConvert.textToStream('hello world'))
    }

The hello function written using quiver stream provide the flexibility of intercepting the results and change it to something else. For instance one could write a upper case stream transform function as follow:

    var toUppercase = function(readStream, callback) {
      var channel = streamChannel.createStreamChannel()
      var writeStream = channel.writeStream

      var doPipe = function() {
        readStream.read(function(streamClosed, data) {
          if(streamClosed) return writeStream.closeWrite(streamClosed.err)

          // Assume string in ASCII encoding
          writeStream.write(data.toUppercase())
          doPipe()
        })
      }

      doPipe()
      callback(null, channel.readStream)
    }

The transform function follows the same quiver style of writing result through a read stream result in async callback, instead of writing the result to a write stream directly. The uppercase function also accept an incoming readStream for it to perform transformation to the original stream. The quiver style of transforming data is by defining an async-recursive doPipe() function to continuously read data from a read stream and writing the transformed result to the write stream of newly created stream channel.

Unlike node streams, quiver streams have very few methods that focus only on the core function of transferring data between a producer and consumer. The main method for quiver `readStream` is the `read()` method that return either a streamClosed object or data buffer as read result. The `streamClosed` object can be any thruthy Javascript object used to indicate that all data have been read from the readStream. `streamClosed` also have a special attribute `err`, which indicates that the readStream was prematurely closed with error if the `err` attribute is truthy.

Having both hello and toUpppercase function allow us to easily compose the two functions together as follow:

    var uppercaseHello = function(callback) {
      hello(function(err, readStream) {
        if(err) return callback(err)

        toUppercase(readStream, callback)
      })
    }

Alternatively we can make use of higher order functions and write a combine function that combine a transform function with a source function:

    var combine = function(sourceFunc, transformFunc) {
      var combinedFunc = function(callback) {
        sourceFunc(function(err, readStream) {
          if(err) return callback(err)
          
          transformFunc(readStream, callback)  
        })
      }
    }

    var uppercaseHello = combine(hello, toUppercase)