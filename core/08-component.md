
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

## Component Dependencies

### Middleware

### Input Handleables

## Handler Component

### Simple Handler

## Filter and Middleware Component

## Router Component

## Pipeline Component

## Next: [Module](09-module.md)