
Quiver.js Documentation
=======================

## Introduction

Quiver.js is a server-side component system for creating large-scale Node.js applications. In Quiver, instead of organizing code as classes or functions, you write interconnected _Quiver Components_ to form a complex component graph.

Quiver components are very easy to use and can be built in three simple steps. First you define _plain functions_ with one of the three _standard component signatures_ in Quiver. Next you write component definitions in a JSON-like Quiver DSL that is made of _plain Javascript constructs_. Lastly simply export the components as an array from your Node modules. The Quiver Component System will then take care the rest for you and connect all your components together.

As an example here is a simple hello world handler component:

```javascript
var helloHandlerBuilder = function(config, callback) {
  var handler = function(args, callback) {
    callback(null, 'Hello World!')
  }

  callback(null, handler)
}

var quiverComponents = [
  {
    name: 'my hello handler',
    type: 'simple handler',
    inputType: 'void',
    outputType: 'text',
    middlewares: [
      'my permission filter',
      'my cache filter'
    ],
    handleables: [
      'my cloud services handler'
    ],
    handlerBuilder: helloHandlerBuilder
  }
]

module.exports = {
  quiverComponents: quiverComponents
}
```

## Demo and Tutorial

Simple demos and tutorials are written for quick introduction to Quiver.js.

  1. [Hello User](https://github.com/quiverjs/demo/tree/master/01-hello-user) demonstrates the way to write a set of quiver components to serve different users in different ways.

## Why Quiver?

You should use Quiver if 

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

## [FAQ](FAQ.md)

## Libraries

A list of available Quiver.js libraries can be found at the project GitHub [organization page](https://github.com/quiverjs).

## Status

Quiver.js is currently at its final stage of private development. The library will soon have its beta release by February 2014.

If you are interested to learn more about the project, please contact the author [Soares Chen](https://github.com/soareschen) at soares.chen@gmail.com.

## Acknowledgement

I'd like to thank my company DoReMIR Music Research AB for giving me chance to make use of Quiver.js in the company's backend development. Quiver.js currently runs on the production backend server of [ScoreCloud](http://scorecloud.com/). (p.s. and we are [hiring](http://scorecloud.com/jobs/#backenddev)!)