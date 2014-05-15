# Summary

In this chapter eight types of constructs have been introduced to build the foundation of the Quiver architecture. The architecture may be applied to other programming languages and developers would still benefit from the architecture without the libraries from Quiver.

Following is a short summary for each of the constructs that have been introduced:

  - *Stream* is implemented as a single producer-consumer channel. The quiver stream is designed to be minimalistic and delegate extensibility to higher layers.

  - *Streamable* encapsulates stream into plain object that may contain conversion methods to other object types. Streamable enables local optimization by allowing local components to pass objects without the overhead of parsing raw streams.

  - *Stream Handler* is similar to Unix process that accepts single input/output streams. It accepts input in the form of streamable and produce result by returning the result as another streamable.

  - *HTTP Handler* accepts a request head along with request body as streamable, and return a response head along with response body as streamable. It is used when there is need to manipulate the HTTP headers.

  - *Handleable* acts as a supertype of stream handler and http handler. It is a plain object with optional conversion functions to convert to specific handler types.

  - *Handler Builder* is constructor function for handler. It solves the dependency management problem and provide uniform way to instantiate handlers.

  - *Filter* extends existing handlers through composition. A returned filtered handler can intercept all input/output of the original handler and manipulate them to produce augmented results.

  - *Middleware* is one more level ahead of filter by extending handler builders instead of handlers. It can affect the way handler is constructed as well as extending the handler result just as filter does.

  - *Component* acts as a glueing layer to connect together different types of handlers, filters and middlewares. It eventually convert all components into handleable builder or handleable middleware and compose them based on the component definition.
