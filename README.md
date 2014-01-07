
Quiver.js Documentation
=======================

## Introduction

Quiver.js is a server-side component system that allow web applications to be built consist of small and reusable components. It consist of a collection of small libraries written in Javascript running on Node.js.

Web applications using Quiver.js are easily reconfigurable by simply rewiring components in different ways with minimal coding required. Components in Quiver.js are written as pure Javascript functions that are combined using functional composition techniques. For example here is a simple hello world handler component:

```javascript
var helloHandlerBuilder = function(config, callback) {
  var handler = function(args, callback) {
    callback(null, 'Hello World!')
  }

  callback(null, handler)
}

var quiverComponents = [
  {
    name: 'hello handler',
    type: 'simple handler',
    inputType: 'void',
    outputType: 'text',
    handlerBuilder: helloHandlerBuilder
  }
]
```

A quiver application is made of many such quiver components, such as [handlers](core/03-handler.md) and [filters](core/05-filter.md). Each of these component types have well defined function signatures that are specifically designed to make it modular and scalable. The [component system](08-component.md) allow complex component dependency graph to be built by specifying it in a JSON-like DSL.

## Core Concepts

Quiver.js is made of the following core concepts, with each of them built on top of the previous ones.

  1. [Stream](core/01-stream.md)
  2. [Streamable](core/02-streamable.md)
  3. [Handler](core/03-handler.md)
  4. [Handler builder](core/04-handler-builder.md)
  5. [Filter](core/05-filter.md)
  6. [Middleware](core/06-middleware.md)
  7. [Handleable](core/07-handleable.md)
  8. [Component](core/08-component.md)
  9. [Module](core/09-module.md)

## Demo and Tutorial

Simple demos and tutorials are written for quick introduction to Quiver.js.

  1. [Hello User](https://github.com/quiverjs/demo/tree/master/01-hello-user)

## [FAQ](FAQ.md)

## Libraries

A list of available Quiver.js libraries can be found at the project GitHub [organization page](https://github.com/quiverjs).

## Coming Soon

Quiver.js is currently at its final stage of private development. The library will soon have its beta release by February 2014.

If you are interested to learn more about the project, please contact the author Soares Chen at soares.chen@gmail.com.