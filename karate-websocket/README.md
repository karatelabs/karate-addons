# Karate-Websocket

Karate-Websocket adds first-class support for testing communication over the [Websocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API). The challenge of sending and consuming messages in async fashion is solved via an elegant API.

## Highlights
* Unified syntax similar to HTTP but focused on Websocket
* Mix HTTP and Websocket calls within the same test flow
* Set up multiple async connections if needed
* Support for parallel execution
* Support for performance testing
* Express data as JSON and leverage Karate's powerful assertions
* Easy to define custom message types and sequence handling
* Automatic conversion from JSON to your custom wire-format
* Support for SSL/TLS and using certificates for secure auth

## License
### Runtime License:
To run Karate tests using this library, you need a license from Karate labs. You can email info@karatelabs.io and request a license.

### Developer License
To develop and run feature files from the IDE you need to upgrade to the paid versions of Karate Labs official plugins for [IntelliJ](https://github.com/karatelabs/intellij-plugin) or [VS Code](https://github.com/karatelabs/vscode-extension).

## Setup
You need a Maven or Gradle project. Please use the latest available version. The dependency info can be found here: https://central.sonatype.com/artifact/io.karatelabs/karate-websocket

The `karate.lic` file you receive should be placed in a `.karate` folder in your project root. You can also change the default path where the license is expected - by setting a `KARATE_LICENSE_PATH` environment property.

## Sample Project
You can find a sample project here: [Karate Websocket Example](https://github.com/karatelabs/karate-examples/blob/main/websocket/README.md)
 
## Syntax

The syntax is designed to give you full control over sending and receiving async messages.

Websockets is somewhat "low level" and most real-world usage layers a high-level protocol such as [STOMP](https://stomp.github.io/) or a custom JSON schema. Karate's enterprise Websocket support makes it easy to handle message conversions and hanshake flows using a simple and flexible [adapter](#adapter) approach.

Note that [variables](https://github.com/karatelabs/karate#native-data-types) and JSON [embedded expressions](https://github.com/karatelabs/karate#embedded-expressions) will work just like you expect in Karate.


Async handling requires a little more complexity than simple API tests, but `karate-websocket` still keeps it simple. Here is an example:

```cucumber
    * def session = karate.channel('websocket')
    * session.url = 'wss://ws.postman-echo.com/raw'
    * session.start()

    * session.send('hello')

    * def response = session.collect()
    * match response == ['hello']    
```

Note how the syntax is future-proof, and support for other async protocols such as [`grpc`](../karate-grpc/README.md) and [`kafka`](../karate-kafka/README.md) is very similar.

### `karate.channel()`

Typically you name the returned variable from `karate.channel()` as `session`. Now you can set properties before calling [`session.start()`](#sessionstart).

Behind the scenes a new Websocket client is created when `session.start()` is called.

### `session.url`

This is to set the URL that would start with `ws://` or `wss://`.

### `session.headers`

You can set custom headers at any time before or during a "flow" of messages.

```cucumber
    * def session = karate.channel('websocket')
    * session.url = 'wss://ws.postman-echo.com/raw'
    * session.headers = { myHeaderName: 'myHeaderValue' }
```

### `session.start()`
You have to call this to start the listener process. To complete the test flow, you have to call [`session.collect()`](#sessioncollect)

### `session.send()`
This is how to send messages from the client. If you need conversion of what you pass as the argument, refer to how to write an [adapter](#adapter).

### `session.collect()`
Since Websocket and async listeners can span or "collect" multiple messages, this is always an array of messages. The format of each individual message depends on the wire-format or any [adapter](#adapter) you have configured. Also see [`session.pop()`](#sessionpop)

### `session.pop()`
For the very common case where you know only one message would be collected or if you care only about the first message received, this is a convenience that will call [`session.collect()`](#sessioncollect) behind the scenes.

### `session.stop()`
Will close the Websocket client.

## Adapter

By default messages are treated as plain-text. So a test can be as simple as [`echo.feature`](https://github.com/karatelabs/karate-examples/blob/main/websocket/src/test/java/karate/echo.feature).

For full control over the messages and what happens immediately after a connection is established, you can implement the `io.karatelabs.websocket.WebsocketAdapter` Java interface.

```java
public interface WebsocketAdapter<T, U> {

    default void onStart(WebsocketConsumer consumer) {
    }

    default void onStop(WebsocketConsumer consumer) {
    }

    void onMessage(WebsocketConsumer consumer, T message);

    Object toWire(T message);

    T fromWire(U raw);

}
```

A simple implementation `io.karatelabs.websocket.JsonAdapter` is available that will auto convert each message from or to JSON. So a test can look like this: [`json.feature`](https://github.com/karatelabs/karate-examples/blob/main/websocket/src/test/java/karate/json.feature):

```cucumber
    * def session = karate.channel('websocket')
    * session.url = 'wss://ws.postman-echo.com/raw'
    * def Adapter = Java.type('io.karatelabs.websocket.JsonAdapter')
    * session.adapter = new Adapter()
    * session.start()

    * session.send({ message: 'hello' })

    * def response = session.collect()
    * match response == [{ message: 'hello' }]
```

It is worth looking at how simple the `JsonAdapter` is, which will help you come up with your own custom implementation:

```java
public class JsonAdapter implements WebsocketAdapter<Map<String, Object>, String> {

    @Override
    public void onMessage(WebsocketConsumer client, Map<String, Object> message) {
        client.receive(message);
    }

    @Override
    public String toWire(Map<String, Object> map) {
        return Json.of(map).toString();
    }

    @Override
    public Map<String, Object> fromWire(String raw) {
        return Json.of(raw).asMap();
    }

}
```

Note how `client.receive()` is called to pass the final message to the "Karate side".

Implementation of the [STOMP](https://stomp.github.io) protocol is available in the example project: [`StompAdapter.java`](https://github.com/karatelabs/karate-examples/blob/main/websocket/src/test/java/karate/StompAdapter.java).

With a proper Adapter in place, a test becomes very simple and readable: [`stomp.feature`](https://github.com/karatelabs/karate-examples/blob/main/websocket/src/test/java/karate/stomp.feature).

Observe how a re-usable JavaScript function called `result()` is used to abstract out the request-response flow into a "one-liner, so the *real* business flow ends up being the following two lines:

```cucumber
    * match result('foo') == 'Hello, foo!'
    * match result('bar') == 'Hello, bar!'
```

Details on how to run the full example (and the websocket server to test) are available here: [Karate Websocket example](https://github.com/karatelabs/karate-examples/blob/main/websocket/README.md).