# The Event Bus

The `event bus` is the **nervous system** of Vert.x.

There is a single event bus instance for every Vert.x instance and it is
obtained using the method `eventBus`.

The event bus allows different parts of your application to communicate
with each other, irrespective of what language they are written in, and
whether they’re in the same Vert.x instance, or in a different Vert.x
instance.

It can even be bridged to allow client-side JavaScript running in a
browser to communicate on the same event bus.

The event bus forms a distributed peer-to-peer messaging system spanning
multiple server nodes and multiple browsers.

The event bus supports publish/subscribe, point-to-point, and
request-response messaging.

The event bus API is very simple. It basically involves registering
handlers, unregistering handlers and sending and publishing messages.

First some theory:

## The Theory

### Addressing

Messages are sent on the event bus to an **address**.

Vert.x doesn’t bother with any fancy addressing schemes. In Vert.x an
address is simply a string. Any string is valid. However it is wise to
use some kind of scheme, *e.g.* using periods to demarcate a namespace.

Some examples of valid addresses are europe.news.feed1,
acme.games.pacman, sausages, and X.

### Handlers

Messages are received by handlers. You register a handler at an address.

Many different handlers can be registered at the same address.

A single handler can be registered at many different addresses.

### Publish / subscribe messaging

The event bus supports **publishing** messages.

Messages are published to an address. Publishing means delivering the
message to all handlers that are registered at that address.

This is the familiar **publish/subscribe** messaging pattern.

### Point-to-point and Request-Response messaging

The event bus also supports **point-to-point** messaging.

Messages are sent to an address. Vert.x will then route them to just one
of the handlers registered at that address.

If there is more than one handler registered at the address, one will be
chosen using a non-strict round-robin algorithm.

With point-to-point messaging, an optional reply handler can be
specified when sending the message.

When a message is received by a recipient, and has been handled, the
recipient can optionally decide to reply to the message. If they do so,
the reply handler will be called.

When the reply is received back by the sender, it too can be replied to.
This can be repeated *ad infinitum*, and allows a dialog to be set up
between two different verticles.

This is a common messaging pattern called the **request-response**
pattern.

### Best-effort delivery

Vert.x does its best to deliver messages and won’t consciously throw
them away. This is called **best-effort** delivery.

However, in case of failure of all or parts of the event bus, there is a
possibility messages might be lost.

If your application cares about lost messages, you should code your
handlers to be idempotent, and your senders to retry after recovery.

### Types of messages

Out of the box Vert.x allows any primitive/simple type, String, or
`buffers` to be sent as messages.

However it’s a convention and common practice in Vert.x to send messages
as [JSON](http://json.org/)

JSON is very easy to create, read and parse in all the languages that
Vert.x supports so it has become a kind of *lingua franca* for Vert.x.

However you are not forced to use JSON if you don’t want to.

The event bus is very flexible and also supports sending arbitrary
objects over the event bus. You can do this by defining a `codec` for
the objects you want to send.

## The Event Bus API

Let’s jump into the API.

### Getting the event bus

You get a reference to the event bus as follows:

``` js
let eb = vertx.eventBus();
```

There is a single instance of the event bus per Vert.x instance.

### Registering Handlers

This simplest way to register a handler is using `consumer`. Here’s an
example:

``` js
let eb = vertx.eventBus();

eb.consumer("news.uk.sport", (message) => {
  console.log("I have received a message: " + message.body());
});
```

When a message arrives for your handler, your handler will be called,
passing in the `message`.

The object returned from call to consumer() is an instance of
`MessageConsumer`.

This object can subsequently be used to unregister the handler, or use
the handler as a stream.

Alternatively you can use `consumer` to return a MessageConsumer with no
handler set, and then set the handler on that. For example:

``` js
let eb = vertx.eventBus();

let consumer = eb.consumer("news.uk.sport");
consumer.handler((message) => {
  console.log("I have received a message: " + message.body());
});
```

When registering a handler on a clustered event bus, it can take some
time for the registration to reach all nodes of the cluster.

If you want to be notified when this has completed, you can register a
`completion handler` on the MessageConsumer object.

``` js
consumer.completionHandler((res) => {
  if (res.succeeded()) {
    console.log("The handler registration has reached all nodes");
  } else {
    console.log("Registration failed!");
  }
});
```

### Un-registering Handlers

To unregister a handler, call `unregister`.

If you are on a clustered event bus, un-registering can take some time
to propagate across the nodes. If you want to be notified when this is
complete, use `unregister`.

``` js
consumer.unregister((res) => {
  if (res.succeeded()) {
    console.log("The handler un-registration has reached all nodes");
  } else {
    console.log("Un-registration failed!");
  }
});
```

### Publishing messages

Publishing a message is simple. Just use `publish` specifying the
address to publish it to.

``` js
eventBus.publish("news.uk.sport", "Yay! Someone kicked a ball");
```

That message will then be delivered to all handlers registered against
the address news.uk.sport.

### Sending messages

Sending a message will result in only one handler registered at the
address receiving the message. This is the point-to-point messaging
pattern. The handler is chosen in a non-strict round-robin fashion.

You can send a message with `send`.

``` js
eventBus.send("news.uk.sport", "Yay! Someone kicked a ball");
```

### Setting headers on messages

Messages sent over the event bus can also contain headers. This can be
specified by providing a {@link io.vertx.core.eventbus.DeliveryOptions}
when sending or publishing:

``` $lang
{@link docoverride.eventbus.Examples#headers(io.vertx.core.eventbus.EventBus)}
```

### Message ordering

Vert.x will deliver messages to any particular handler in the same order
they were sent from any particular sender.

### The Message object

The object you receive in a message handler is a `Message`.

The `body` of the message corresponds to the object that was sent or
published.

The headers of the message are available with `headers`.

### Acknowledging messages / sending replies

When using `send` the event bus attempts to deliver the message to a
`MessageConsumer` registered with the event bus.

In some cases it’s useful for the sender to know when the consumer has
received the message and "processed" it using **request-response**
pattern.

To acknowledge that the message has been processed, the consumer can
reply to the message by calling `reply`.

When this happens it causes a reply to be sent back to the sender and
the reply handler is invoked with the reply.

An example will make this clear:

The receiver:

``` js
let consumer = eventBus.consumer("news.uk.sport");
consumer.handler((message) => {
  console.log("I have received a message: " + message.body());
  message.reply("how interesting!");
});
```

The sender:

``` js
eventBus.request("news.uk.sport", "Yay! Someone kicked a ball across a patch of grass", (ar) => {
  if (ar.succeeded()) {
    console.log("Received reply: " + ar.result().body());
  }
});
```

The reply can contain a message body which can contain useful
information.

What the "processing" actually means is application-defined and depends
entirely on what the message consumer does and is not something that the
Vert.x event bus itself knows or cares about.

Some examples:

  - A simple message consumer which implements a service which returns
    the time of the day would acknowledge with a message containing the
    time of day in the reply body

  - A message consumer which implements a persistent queue, might
    acknowledge with `true` if the message was successfully persisted in
    storage, or `false` if not.

  - A message consumer which processes an order might acknowledge with
    `true` when the order has been successfully processed so it can be
    deleted from the database

### Sending with timeouts

When sending a message with a reply handler, you can specify a timeout
in the `DeliveryOptions`.

If a reply is not received within that time, the reply handler will be
called with a failure.

The default timeout is 30 seconds.

### Send Failures

Message sends can fail for other reasons, including:

  - There are no handlers available to send the message to

  - The recipient has explicitly failed the message using `fail`

In all cases, the reply handler will be called with the specific
failure.

### Message Codecs

You can send any object you like across the event bus if you define and
register a {@link io.vertx.core.eventbus.MessageCodec message codec} for
it.

Message codecs have a name and you specify that name in the {@link
io.vertx.core.eventbus.DeliveryOptions} when sending or publishing the
message:

``` java
{@link docoverride.eventbus.Examples#example10}
```

If you always want the same codec to be used for a particular type then
you can register a default codec for it, then you don’t have to specify
the codec on each send in the delivery options:

``` java
{@link docoverride.eventbus.Examples#example11}
```

You unregister a message codec with {@link
io.vertx.core.eventbus.EventBus\#unregisterCodec}.

Message codecs don’t always have to encode and decode as the same type.
For example you can write a codec that allows a MyPOJO class to be sent,
but when that message is sent to a handler it arrives as a MyOtherPOJO
class.

### Clustered Event Bus

The event bus doesn’t just exist in a single Vert.x instance. By
clustering different Vert.x instances together on your network they can
form a single, distributed event bus.

### Clustering programmatically

If you’re creating your Vert.x instance programmatically you get a
clustered event bus by configuring the Vert.x instance as clustered;

``` js
import { Vertx } from "@vertx/core"
let options = new VertxOptions();
Vertx.clusteredVertx(options, (res) => {
  if (res.succeeded()) {
    let vertx = res.result();
    let eventBus = vertx.eventBus();
    console.log("We now have a clustered event bus: " + eventBus);
  } else {
    console.log("Failed: " + res.cause());
  }
});
```

You should also make sure you have a `ClusterManager` implementation on
your classpath, for example the Hazelcast cluster manager.

### Clustering on the command line

You can run Vert.x clustered on the command line with

    vertx run my-verticle.js -cluster

## Automatic clean-up in verticles

If you’re registering event bus handlers from inside verticles, those
handlers will be automatically unregistered when the verticle is
undeployed.

# Configuring the event bus

The event bus can be configured. It is particularly useful when the
event bus is clustered. Under the hood the event bus uses TCP
connections to send and receive messages, so the {@link
io.vertx.core.eventbus.EventBusOptions} let you configure all aspects of
these TCP connections. As the event bus acts as a server and client, the
configuration is close to {@link io.vertx.core.net.NetClientOptions} and
{@link io.vertx.core.net.NetServerOptions}.

``` $lang
{@link examples.EventBusExamples#example13}
```

The previous snippet depicts how you can use SSL connections for the
event bus, instead of plain TCP connections.

**WARNING**: to enforce the security in clustered mode, you **must**
configure the cluster manager to use encryption or enforce security.
Refer to the documentation of the cluster manager for further details.

The event bus configuration needs to be consistent in all the cluster
nodes.

The {@link io.vertx.core.eventbus.EventBusOptions} also lets you specify
whether or not the event bus is clustered, the port and host.

When used in containers, you can also configure the public host and
port:

``` $lang
{@link examples.EventBusExamples#example14}
```
