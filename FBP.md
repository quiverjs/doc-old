
Quiver.js vs Flow Based Programming
===================================

I have been asked a few times by people comparing Quiver with Flow Based Programming (FBP) implementations, particularly [NoFlo](http://noflojs.org/). I have looked at Flow Based Programming for some time. Prior to Quiver I was particularly inspired by [Kamaelia](http://www.kamaelia.org/Home.html), a Python implementation of Flow Based Programming. 

I did not go into much details of using FBP, so I can't comment much on the advantages and disadvantages of FBP. But I can explain here why I decided not to use FBP and designed Quiver:


### Server-side Application

Quiver currently focus exclusively on server-side web applications. The design of quiver components follow the same single-request/single-response model of HTTP. It also follow RESTful principles and the Unix Philosophy way of creating scalable and modular components.


### Visual Programming

Quiver currently do _not_ rely on any visual programming elements to make it easier for programming. Rather Quiver focus on the classical way of creating many design patterns that developers can apply when writing their components.


### Graph Configuration

Quiver component graph is configured _once_ during setup time. The same component graph is then reused for multiple concurrent stream processing. Each processing starts with an input stream, which data arrive in chunks asynchronously.


### Dependency Management

Quiver component graph is composed at [_constructor level_](core/04-handler-builder.md). Quiver constructors are plain functions with standard signatures to accept at least a `config` parameter, which is used to inject dependencies to the component. Therefore it is very easy to reconfigure components simply through basic programming.


### Input/Output Count

Quiver components have only one input and one output. Nevertheless it is possible to create component graph of multiple input/output using the [quiver-split-stream](https://github.com/quiverjs/split-stream) library. It is just that graphs with extra input/output are hidden from external view, or one have to design non-standard function signatures to enable such composition.


### Polymorphic Input/Output

Quiver allows components to accept/return polymorphic input/output with the [streamable](core/02-streamable.md) object, which can act as either stream or plain JS objects like string or json.


### Intermediary Composition

Quiver components like [filters](core/05-filter.md) can wrap around other components and intercept their input, output, and error. The pattern is similar to Aspect-Oriented Programming and allow separation of concerns in component implementation.


### Stream Forwarding

Quiver components forward the whole stream as an object to another component. It avoids the overhead of having to forward individual data packets across too many layers.


### Manual Composition

The Quiver component system is completely optional to use. It is possible to manually combine components together without help from the system, albeit with a lot boilerplate code.