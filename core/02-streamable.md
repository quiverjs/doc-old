
Streamable
==========

[Streams](01-stream.md) are great at asynchronously transferring large amount of data in chunks for efficient data processing. However most of the time data transferred across streams usually have small size and are derived from plain Javascript objects. Consider the follow greet function:

```javascript
var streamConvert = require('quiver-stream-convert')

var greet = function(readStream, callback) {
  streamConvert.streamToText(readStream, function(err, text) {
    if(err) return callback(err)

    try {
      var json = JSON.parse(text)
    } catch(err) {
      return callback(err)
    }

    var name = json.name
    var greeting = 'hello, ' + name

    callback(null, streamConvert.textToStream(greeting))
  })
}

var myName = 'john'

// Why convert to stream
var readStream = streamConvert.textToStream(myName)
greet(readStream, function(err, resultStream) { ... })

// instead of passing the string itself?
// greet(myName, function(err, greeting) { ... })
```

The `greet()` function works great if it is processing streams consist of raw bytes. But in the case where there is already a plain Javascript object reside in the program, the function would work less efficiently because the Javascript object has to be serialized into raw bytes only to be parsed again inside the function.

To solve this problem, we need to write functions that accept object that is convertible to both stream and plain Javascript objects. In Quiver.js such object is called a _streamable_. A streamable is a plain Javascript object that has a compulsory `toStream` method and arbitrary number of optional `toXXX()` methods where `XXXX` is the Javascript object representation of the streamable, such as `toText()` and `toJson()`.

The converted objects or streams are independent from the streamable object and its other representations. For instance if a json object returned from `streamable.toJson()` is modified, it cannot affect the stream representation returned from `streamable.toStream()` nor value returned from subsequent calls to `streamable.toJson()`.

A new version of greet function can accept a streamable and check whether it has a `toJson()` method. If so then the function can retrive the json object immediately and avoid parsing the json from raw bytes.

```javascript
var greet = function(streamable, callback) {
  if(streamable.toJson) {
    streamable.toJson(function(err, json) {
      if(err) return callback(err)

      var name = json.name
      var greeting = 'hello, ' + name

      callback(null, streamConvert.textToStream(greeting))
    })
  } else {
    // parse raw stream content
    ...
  }
}
```

One would notice that the above code is still not efficient. The code for converting from streamable to json should be refactored to a separate function. `quiver-stream-convert` provide such utility functions.

```javascript
var streamConvert = require('quiver-stream-convert')

var greet = function(streamable, callback) {
  streamConvert.streamableToJson(streamable, function(err, json) {
    if(err) return callback(err)

    var name = json.name
    var greeting = 'hello, ' + name

    callback(null, streamConvert.textToStream(greeting))
  })
}
```

Internally, `streamableToJson()` would check for existing `toJson()` method in the streamable. If the method is not found the raw byte stream is parsed, and finally the function would attach a new `toJson()` method to the same streamable if it is reusable.


```javascript
var copyObject = require('quiver-copy').copyObject

streamConvert.streamableToJson = function(streamable, callback) {
  if(streamable.toJson) return streamable.toJson(callback)

  streamableToText(streamable, function(err, text) {
    if(err) return callback(err)

    try {
      var json = JSON.parse(text)
    } catch(err) {
      return callback(err)
    }

    if(streamable.reusable) {
      streamable.toJson = function(callback) {
        // make a copy every time to make sure
        // the cached content cannot be modified
        callback(null, copyObject(json))
      }
    }

    callback(null, copyObject(json))
  })
}
```

Streamable objects have an optional reusable flag, which indicates whether the `streamable.toStream()` can be called multiple times. By default streamables made of raw streams are not reusable to avoid retaining the entire stream content in memory for too long. Having reusable streamable means that multiple quiver read streams can be created to be consumed by multiple readers. A reusable streamable is most useful to implement caches in quiver applications.

## API Specification

```javascript
streamable.toStream = function(function (err, readStream) { }) { }
```

Returns a quiver read stream representation of the data inside the streamable. If the streamable is not reusable and `toStream()` has been called before, an error will be returned.

```javascript
streamable.toText = function(function (err, text) { }) { }
streamable.toJson = function(function (err, json) { }) { }
streamable.toXXXX = function(function (err, convertedObject) { }) { }
```

If a function with prefix “to” exist on the streamable, it can be used to convert the streamable to the respective equivalent Javascript objects and allow one to skip parsing it from raw byte stream.


### quiver-stream-convert

For full documentation visit the [quiver-stream-convert](https://github.com/quiverjs/stream-convert) repository.

```javascript
var streamConvert = require('quiver-stream-convert')

var textStreamable = streamConvert.textToStreamable(text)
var jsonStreamable = streamConvert.jsonToStreamable(json)
```

Convert a string or plain Javascript object into streamable that contain optimized `toText()`/`toJson()` methods.


```javascript
streamConvert.streamableToText = function(streamable, function (err, text) { }) { }
streamConvert.streamableToJson = function(streamable, function (err, json) { }) { }
```

Convert streamable into string or plain Javascript object while making use of optimized `toText()`/`toJson()` methods in the streamable if exist.

## Next: [Handler](03-handler.md)