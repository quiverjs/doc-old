
Component
=========

In the previous chapters we have learned the following:

  - [Stream](01-stream.md) enable functions to create and return read streams instead of writing result to write streams. 
  - [Streamable](02-streamable.md) is then built on top of stream to represent objects convertible to multiple format representations. 
  - [Stream handler and HTTP handler](03-handler.md) are then introduced to enforce uniform function interface across all programs for easy composition similar to Unix processes. 
  - [Handler builder](04-handler-builder.md) provides a uniform interface for constructing handlers from runtime configuration.
  - [Filter](05-filter.md) extends the functionality of handler through composition.
  - [Middleware](06-middleware.md) perform extension on handler builder level and are easily composable with handler builders.