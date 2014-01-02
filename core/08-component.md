
Component
=========

In the previous chapters we have learned the following:

  - [Stream](01-stream.md) enable functions to create and return read streams instead of writing result to write streams. 
  - [Streamable](02-streamable.md) is then built on top of stream to represent objects convertible to multiple format representations. 
  - [Stream handler and HTTP handler](03-handler.md) are then introduced to enforce uniform function interface across all programs for easy composition similar to Unix processes. 
  - [Handler builder](04-handler-builder.md) provides a uniform interface for constructing handlers from runtime configuration.
  - [Filter](05-filter.md) extends the functionality of handler through composition.
  - [Middleware](06-middleware.md) perform extension on handler builder level and are easily composable with handler builders.

Together, the Quiver.js component system acts as a layer to glue all the different building blocks together to form a working web application. There are mainly three types of quiver components:

  - **Handler component** turns handler builders into managed handleable builders.
  - **Middleware component** turns middlewares or filters into managed handleable middlewares.
  - **Config component** adds entries into generated component config.

## Component Definition

Quiver components are defined in DSL-like fashion, except that they consist of plain Javascript constructs and look much like JSON. Here is an example handler component:

```javascript
{
  name: 'my handler',
  type: 'stream handler',
  handlerBuilder: function(config, callback) { ... }
}
```

Other than allowing reference to function, quiver components are pretty much defined the same way as simple JSON. There is no special quiver library or framework required to define a component.

## Component Export

Quiver components are typically placed in separate source files and have the components exported in the special `quiverComponents` field, which is an array of component definitions. Below is an example of a simple but complete component source file:

```javascript
var echoHandlerBuilder = function(config, callback) {
  var handler = function(args, inputStreamable, callback) {
    callback(null, inputStreamable)
  }

  callback(null, handler)
}

var quiverComponents = [
  {
    name: 'demo echo handler',
    type: 'stream handler',
    handlerBuilder: echoHandlerBuilder
  }
]

module.exports = {
  quiverComponents: quiverComponents
}
```

By convention, function bodies are typically defined as separate local variables in the module. The functions are then referenced inside the `quiverComponents` array, with each quiver component definition separated through comma. The `quiverComponents` variable is finally exported to `module.exports`, optionally together with other exported functions.

Notice that at its minimal, a component source file has no need to import any Quiver.js library. Although Quiver.js supply many packages to build up this component system, the component architecture is more of a programming convention to create scalable web applications through well-defined function interfaces. Therefore one will able to easily migrate their code to another library or framework that follow the Quiver.js architecture, with minimal dependency on any Quiver.js libraries.


## Component Dependencies

The most basic form of component definition look not much interesting, since one could as well define just a function and get the same result. The real benefit of component definition is to allow components to have complex dependencies among each others. The two most basic types of component dependencies are _middleware_ and _handleable_ dependencies.

### Middleware

Middleware dependency allow a handler or another middleware to depend on other middleware or filter components based on name. For example a hello handler with dependency on uppercase result filter can be defined as follow:

```javascript
var helloHandlerBuilder = function(config, callback) { ... }
var uppercaseResultFilter = function(config, handler, callback) { ... }

var quiverComponents = [
  {
    name: 'demo hello handler'.
    type: 'stream handler',
    middlewares: [
      'demo uppercase result filter'
    ],
    handlerBuilder: helloHandlerBuilder
  },
  {
    name: 'demo uppercase result filter',
    type: 'stream filter',
    filter: uppercaseResultFilter
  }
]
```

Notice that the component definitions can also be defined in different source files. There is no need for the source files to include the dependency source files because dependencies are referenced by name in the form of string. The Quiver component system will automatically manage the dependencies and apply dependency with the instantiated components.

Middleware dependencies can also be applied to middleware. That really mean that the given middleware require other specified middlewares to be applied before itself. This could lead to an issue which a middleware is repeated inside nested middleware dependencies. In this case `quiver-component` by default would apply the middleware only once when it is found the first time at the outermost layer.

```javascript
var quiverComponents = [
  {
    name: 'handler'.
    type: 'stream handler',
    middlewares: [
      'filter1',
      'filter2'
    ],
    handlerBuilder: handlerBuilder
  },
  {
    name: 'filter1',
    type: 'stream filter',
    middlewares: [
      'filter3'
    ],
    filter: filter1
  },
  {
    name: 'filter2',
    type: 'stream filter',
    middlewares: [
      'filter1'
      'filter3'
    ],
    filter: filter2
  },
  {
    name: 'filter3',
    type: 'stream filter',
    filter: filter3
  }
]
```

In the above example each filters will be applied only once in the order of `filter3(filter1(filter2(handler)))`. Note that cyclic dependency is however not allowed and error is generated when that is encountered.

### Input Handleables

Handler builders and middlewares may also have another dependencies, which are _input handlers_. Input handlers allow a handler builder or middleware to access another handler that has already been instantiated and component-configured. Consider the following example:

```javascript
var fooHandlerBuilder = function(config, callback) {
  var barHandleable = config.quiverHandleables['bar handler']

  ...
}

var barHandlerBuilder = function(config, callback) { ... }
var bazMiddleware = function(config, handlerBuilder, callback) { ... }

var quiverComponents = [
  {
    name: 'foo handler',
    type: 'stream handler',
    handleables: [
      'bar handler'
    ],
    handlerBuilder: fooHandlerBuilder
  },
  {
    name: 'bar handler',
    type: 'stream handler',
    middlewares: [
      'baz middleware'
    ],
    handlerBuilder: barHandlerBuilder
  },
  {
    name: 'baz middleware',
    type: 'stream middleware',
    middleware: bazMiddleware
  }
]
```

In the body of `fooHandlerBuilder` at the above example, `foo handler` would receive an instantiated `bar handler` that has been applied with `baz middleware`, possibly configured with the same config as `foo handler` was about to receive.

The most basic of the handleables specification instantiate the dependencies into handleables. This allows the components to have dependency on both stream handlers and http handlers in the same way. It might however be tedious to check whether the handleables are of the right handler type the component needs, since there is no type checking involved.

A better way is to specify the input handler type in the handleables specification:

```javascript
var fooHandlerBuilder = function(config, callback) {
  var barHandler = config.quiverStreamHandlers['bar handler']

  ...
}

var quiverComponents = [
  {
    name: 'foo handler',
    type: 'stream handler',
    handleables: [
      {
        handler: 'bar handler',
        type: 'stream handler'
      }
    ],
    handlerBuilder: fooHandlerBuilder
  },
```

In the above example the input handler type of bar handler is specified as stream handler, and the instantiated bar handler is retrieved from `config.quiverStreamHandlers` instead.

Stream handlers by default accept and return streamables, and it might be tedious to convert between trivial values to streamable. Fortunately it is possible to convert input handlers into simple handlers too:

```javascript
var fooHandlerBuilder = function(config, callback) {
  var barHandler = config.quiverSimpleHandlers['bar handler']

  barHandler({}, 'foo', function(err, json) {
    ...
  })
}

var quiverComponents = [
  {
    name: 'foo handler',
    type: 'stream handler',
    handleables: [
      {
        handler: 'bar handler',
        type: 'simple handler',
        inputType: 'text',
        outputType: 'json'
      }
    ],
    handlerBuilder: fooHandlerBuilder
  },
```

Note that there is no need for the original handler to be defined as simple handler to be used by others as input simple handlers. `quiver-component` provides conversion from handleable simple handler regardless of how the handler was originally encapsulated. Any simple handler type mismatch will be ignored by the component system and have the error triggered during run time.

Question may arise on why the need of specifying the handler type twice, once in the handler definition and once in the input handler definition. The reason is because component definition is completely independent of each others, and `quiver-component` may has no knowledge of the at the time of component parsing. Another reason for this is to maximize the customizability, so that the specified input handler can be transparently replaced by another handler without the component aware of it.


## White-box Composition

The component definition is designed following the principle o _white-box composition_ which is the opposite of blackbox composition. You might wonder what is the need to define quiver components in that way rather than composing them directly using the `quiver-middleware` library:

```javascript
var middlewareLib = require('quiver-middleware')

var helloHandlerBuilder = function(config, callback) { ... }
var uppercaseResultFilter = function(config, handler, callback) { ... }

var uppercaseResultMiddleware = middlewareLib.createMiddlewareFromFilter(
  uppercaseResultFilter)

helloHandlerBuilder = middlewareLib.createMiddlewareManagedHandlerBuilder(
  uppercaseResultMiddleware, helloHandlerBuilder)
```

There are several problems with the approach above:

  - The code contains a lot of boilerplate of stitching components together.
  - There is no way to inspect what components the result function is consist of, because the information is fully encapsulated inside the newly created function.
  - There is lack of dependency management especially in ensuring filters are only applied once at outermost layer.
  - It is hard to customize the dependencies, because the functions are directly referencing each others. As a result one may not for example replace a middleware with another mock middleware for unit testing purpose.

With the component definition approach it become much clearer on how the components are composed. Since components relationship are defined by name, the reference be modified at handler-building time to change the behavior of handler. On top of the definition is written in plain Javascript, it allows custom libraries to easily written to interprete and manipulate the component graph without having to resort to techniques like reflection.


## Component Loading

Since quiver component definitions are defined in a `quiverComponent` array, component definitions coming from different sources can easily be combined using the Javascript `Array.concat()` method. Once all quiver components are loaded through any means, the components are then configured and instantiated. Together the created component instances are stored in a componentConfig object, which is then returned asynchronously by the [quiver-component](https://github.com/quiverjs/component) library.

The following example shows how the echo component is loaded:

```javascript
var echoLib = require('./component/echo')
var componentLib = require('quiver-component')

componentLib.installComponents(echoLib.quiverComponents, function(err, componentConfig) {
  if(err) throw err

  var echoHandleableBuilder = componentConfig.quiverHandleableBuilders['demo echo handler']

  ...
})
```

## Component Config

`quiver-component` parses each component definition and store its results in a `componentConfig` object. Different types of components are stored in different fields in the component config. The two most common component configs are the `quiverHandleableBuilders` and `quiverMiddlewares`. They are respectively used to store _component-managed handleable builders_ constructed from handler components, and _component-managed handleable middlewares_ constructed from middleware or filter components.

The `componentConfig` also acts as reference to 

## Handler Component

### Simple Handler

## Filter and Middleware Component

## Router Component

## Pipeline Component

## Next: [Module](09-module.md)