
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

The most basic form of component definition look not much interesting, since one could as well define just a function and get the same result. The real benefit of component definition is to allow components to have complex dependencies among each others. The two common types of component dependencies are _middleware_ and _handleable_ dependencies.

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

Note that there is no need for the original handler to be defined as simple handler to be used by others as input simple handlers. `quiver-component` provides conversion from handleable to simple handler regardless of how the handler was originally encapsulated. Any simple handler type mismatch will be ignored by the component system and have the error triggered during run time.

Question may arise on why the need of specifying the handler type twice, once in the handler definition and once in the input handler definition. The reason is because component definition is completely independent of each others, and `quiver-component` may has no knowledge of the handler type at the time of component parsing. Another reason for this is to maximize the customizability, so that the specified input handler can be transparently replaced by another handler without the component aware of it.


## White-box Composition

The component definition is designed following the principle of _white-box composition_ which is the opposite of blackbox composition. You might wonder what is the need to define quiver components in that way rather than composing them directly using the [quiver-middleware](https://github.com/quiverjs/middleware) library:

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

With the component definition approach it become much clearer on how the components are composed. Internally `quiver-component` do make use of `quiver-middleware` to compose components in similar way, so you don't have to do the glueing yourself.

Since components relationship are defined by name, the reference be modified at handler-building time to change the behavior of handler. On top of the definition is written in plain Javascript, it allows custom libraries to easily written to interprete and manipulate the component graph without having to resort to techniques like reflection.


## Component Loading

Since quiver component definitions are defined in a `quiverComponent` array, component definitions coming from different sources can easily be combined using the Javascript `Array.concat()` method. Once all quiver components are loaded through any means, the components are then configured and instantiated. Together the created component instances are stored in a componentConfig object, which is then returned asynchronously by the [quiver-component](https://github.com/quiverjs/component) library.

The following example shows how the echo component is loaded:

```javascript
var echoLib = require('./component/echo')
var otherLib = require('./component/other')
var componentLib = require('quiver-component')

var quiverComponents = [].concat(echoLib.quiverComponents, otherLib.quiverComponents)

componentLib.installComponents(quiverComponents, function(err, componentConfig) {
  if(err) throw err

  var echoHandleableBuilder = componentConfig.quiverHandleableBuilders['demo echo handler']

  ...
})
```

## Component Config

`quiver-component` parses each component definition and store its results in a `componentConfig` object. Different types of components are stored in different fields in the component config. The two most common component configs are the `quiverHandleableBuilders` and `quiverMiddlewares`. They are respectively used to store _component-managed handleable builders_ constructed from handler components, and _component-managed handleable middlewares_ constructed from middleware or filter components.

Regardless of whether the handler types of the components are stream handler or http handler, the components are finally encapsulated into handleable builders or handleable middlewares by the component system. This allow the component system to handler different handler types all using the same code with possible extension in future. The configured components are _managed_, so the component dependency is resolved on instantiation time when config is passed to the handler builder or middleware. 

The following pseudocode shows the equivalent actions quiver-component to manually create a managed handleable builder from the given component definition:

```javascript
var userComponent = {
  name: 'user handler',
  type: 'stream handler',
  middlewares: [
    'user middleware'
  ],
  handlerBuilder: userHandlerBuilder
}

var userHandlerBuilder = function(config, callback) { ... }

var userHandleableBuilder = streamHandlerBuilderToHandleableBuilder(userHandlerBuilder)

var managedUserHandleableBuilder = function(config, callback) {
  var handleableMiddleware = config.quiverMiddlewares['user middleware']

  handleableMiddleware(config, userHandleableBuilder, callback)
}

var componentConfig = {
  quiverHandleableBuilders: {
    'user handler': managedUserHandleableBuilder
  }
}
```

The point is that the handler/middleware dependencies are resolved based on the config supplied to the managed handler builder/middleware. In other words the `componentConfig` returned from `quiver-component` is usually required to get merged with user-provided config and get passed together to the managed handler builder/middleware.

The rational for this is again to maximize the customizability of the quiver component system. One can for example manually inject/replace with custom handleable builder/middleware anywhere in user code without having to interact with the quiver component system:

```javascript
var fooHandlerBuilder = function(config, callback) {
  var inputHandler = config.quiverStreamHandlers['non-existent handler']

  ...
}

var barMiddleware = function(config, handlerBuilder, callback) {
  var myCustomBarHandler = function(args, inputStreamable, callback) {
    ...
  }

  config.quiverStreamHandlers['non-existent handler'] = myCustomBarHandler
  handlerBuilder(config, callback)
}

var quiverComponents = [
  {
    name: 'foo handler',
    type: 'stream handler',
    handleables: [
      {
        handler: 'non-existent handler',
        type: 'stream handler'
      }
    ],
    middlewares: [
      'bar middleware'
    ],
    handlerBuilder: fooHandlerBuilder
  },
  {
    name: 'bar middleware',
    type: 'handleable middleware',
    middleware: barMiddleware
  }
]
```

In the above example, fooHandler has dependency on a non-existent handler that is not defined anywhere as a component. However the handler also have a dependency to the barMiddleware, which modifies the config and inject a custom handler into `config.quiverStreamHandlers['non-existent handler']`. With that fooHandler would continue to work as expected even though looking at the component definition alone, the dependency does not seem to be resolvable.

## Namespace

Quiver component names are not separated by explicit namespace. Therefore it is common practice to have explicit naming convention to ensure there is no general naming conflict among software projects. The first word of a component name is typically reserved for namespace. For example, all "standard" quiver components developed by Quiver.js will have the word "quiver" prefixed in their component name.

Component names may currently contain any identifier-friendly characters, including 0-9, a-z, A-Z, "-", "_", and white space " ". The symbol characters are reserved for possible future DSL extensions.

Although quiver components have global namespace, the name reference are resolved on component initialization time through the passed-in `config` argument. Therefore it remain possible to have variable-shadowing-like effect of overriding a name reference to new component through manipulation by middlewares. A subcomponent system is also currently in development to allow subcomponents to be visible to some specific components.

## Component Types

Below is a list of component types currently available:

### Handler

Handler is the most basic type of quiver component. Although the component is called handler, by default it accepts a handler builder for constructing the handler. There are currently four types of handler components: stream handler, http handler, simple handler, and handleable. The handler builder signature for all four handler types are the same, but the result handler these handler builders return must have the same handler type as specified.


```javscript
{
  name: 'my http handler',
  type: 'http handler',
  handlerBuilder: function(config, callback) { ... }
}
```

Alternatively the handler component may accept the name of another handler in its `handler` field. Handler component of this type is called _extension handler_, because it extends the behavior of existing handler component. There is however no way of extending the component inside component definition. Instead they are typically used to specify extra attributes to the original component, such as adding middleware or configOverride options.

```javscript
{
  name: 'my extended http handler',
  type: 'http handler',
  middlewares: [
    'my http extension filter'
  ],
  handler: 'my http handler'
}
```

Simple handler component are very much the same as other handler component, except that it require two other compulsory parameters which are `inputType` and `outputType`. These are to specify and input and output type for the simple handler respectively. Simple handler is a subtype of stream handler, and they are implicitly converted and treated as stream handlers at component installation time.

```javscript
{
  name: 'my simple handler',
  type: 'simple handler',
  inputType: 'void',
  outputType: 'text',
  handlerBuilder: function(config, callback) { ... }
}
```

Regardless of the handler type, all handler components are converted into handleable builders and can be found in `componentConfig.quiverHandleableBuilders`.

### Filter

Filter components extend on instantiated quiver handlers of the same handler type. Since simple handler are really the same as stream handler, there are only three types of filters: stream filter, http filter, and handleable filter. There is no simple filter available, another reason being that it is not as easy to simplify stream filter as compared to stream handler.

Behind the scene filters are implicitly converted to handleable middleware at component installation time. The wrapped filter may be applied to any handleable builder, but the component system will unbox the handleable and make sure it is the right handler before passing to the filter function.

```javascript
{
  name: 'my stream filter',
  type: 'stream filter',
  filter: function(config, handler, callback) { ... }
}
```

### Middleware

The middleware component is very much similar to filter component. There are three types of middleware: stream middleware, http middleware, and handleable middleware. Typically handleable middlewares are defined for modifiying only the config and directly return result from its handler buider. That way the middleware can be applied to all types of handler builders.

```javascript
{
  name: 'my handleable middleware',
  type: 'handleable middleware',
  middleware: function(config, handleableBuilder, callback) { ... }
}
```

### Pipeline

Pipeline combines multiple stream handler components into one stream handler that process through a pipeline chain. In the pipeline handler the input stream is first processed by the first stream handler, and the result stream returned is fed as the input stream of the second handler, and so on until the last handler in the pipeline.

```javascript
{
  name: 'my pipeline handler',
  type: 'stream pipeline',
  pipeline: [
    'first handler',
    'second handler',
    'third handler'
  ]
}
```

The combined pipeline produce a new pipeline stream handler.

### Router

Router is a special stream handler component that routes incoming request to different stream handlers based on `args.path`. This is the main way of combining multiple handler components into one handler component.

There are currently three types of routes available. Static route require exact string match with `args.path`. regex route matches the provided regex with `path`, and adds the regex match results into `args` based on keys provided in `matchFields`. Dynamic route use provided matcher function to match a path, and add extracted parameters returned from matcher function into `args`.

Stream handler routing is consist of two route components. The route list component is used to specify part of the routes and the handlers routed to. The router component then accepts a list of route list names and combine all routes into one handler component.

```javascript
{
  name: 'my route list',
  type: 'route list',
  routeList: [
    {
      routeType: 'static',
      path: '/static/path',
      handler: 'my handler'
    },
    {
      routeType: 'regex',
      regex: /^\/prefix(\/\w+)$/,
      matchFields: ['path'],
      handler: 'my other handler'
    }
  ]
}

{
  name: 'my other route list',
  type: 'route list',
  routeList: [
    {
      routeType: 'dynamic',
      matcher: function(path) { ... },
      handler: 'my another handler'
    }
  ]
}

{
  name: 'my router',
  type: 'router',
  routeLists: [
    'my route list',
    'my other route list'
  ]
}
```

The reason to separate route list from router definition is so that complex routes can be defined in multiple source files. It also allow part of the routes to be reused to create different router handlers.

The router handler created from the router component is of type stream handler. Currently there is no http router available.

More information about how stream handler routing works can be found at [quiver-router](https://github.com/quiverjs/router).


## Component Options

For all components that produce handler builder or middleware, there are a number of component options that can be applied to these components to extend their functionalities. This is applicable to all the components described earlier, except the route list component.

### Middlewares

The `middlewares` option accept an array of middlewares to apply to the handler builder/middleware. Middlewares are by default applied only once at its outer most occurance. In the case of applying middlewares to middleware, it really means that the middleware require other middlewares to be applied before reaches itself.

```javascript
{
  name: 'my middleware',
  type: 'stream middleware',
  middlewares: [
    'my other middleware'
  ],
  middleware: function(config, handlerBuilder, callback) { ... }
}
```

The application of middleware is only activated at handler building time. Error will be returned at that time if there is a mismatch of middleware type. Prior to that the component system do not verify handler type compatibility of a middleware with a handler.


### Handleables

Handleables provide input handlers to a middleware or handler builder through config. The component system will instantiate the input handlers at handler building time and convert them to the right handler type. By default the input handler type is handleable, and require explicit specification for other handler types.

```
{
  name: 'my handler',
  type: 'stream handler',
  handleables: [
    'my helper handleable',
    {
      handler: 'my helper stream handler',
      type: 'stream handler'
    },
    {
      handler: 'my helper simple handler',
      type: 'simple handler',
      inputType: 'text',
      outputType: 'void'
    }
  ],
  handlerBuilder: function(config, callback) { ... }
}
```

### ConfigOverride

ConfigOverride overrides some `config` parameters just before it reaches the handler builder or middleware function. This is especially useful for extending a component from some generic component.

```javascript
{
  name: 'my generic handler',
  type: 'stream handler',
  handlerBuilder: function(config, callback) {
    var target = config.target

    // do something with target
    ...
  }
}

{
  name: 'my foo handler',
  type: 'stream handler',
  configOverride: {
    target: 'foo'
  },
  handler: 'my generic handler'
}

{
  name: 'my bar handler',
  type: 'stream handler',
  middlewares: [
    'bar filter'
  ],
  configOverride: {
    target: 'bar'
  },
  handler: 'my generic handler'
}
```

### ConfigParam

The config param is used to enforce type safety of config parameters through the [quiver-param](https://github.com/quiverjs/param) module.

```javascript
{
  name: 'my generic handler',
  type: 'stream handler',
  configParam: [
    {
      key: 'target',
      required: true,
      valueType: 'string'
    }
  ],
  handlerBuilder: function(config, callback) {
    var target = config.target

    ...
  }
}
```

## Next: [Module](09-module.md)