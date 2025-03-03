# Karate-Kafka

Karate-Kafka adds first-class support for testing Kafka for both producing and consuming sides. A carefully designed syntax makes it easy to produce Kafka messages the same way you are used to making HTTP requests. The challenge of consuming messages in async fashion is solved via an elegant API.

## Highlights
* Unified syntax similar to HTTP but focused on Kafka
* Flexibility to set up multiple async producers or consumers
* Mix HTTP and Kafka calls within the same test flow
* Support for parallel execution
* Support for performance testing 
* Express data as JSON and leverage Karate's powerful assertions
* Avro, Protobuf or plain JSON serialization support
* Use Avro or Protobuf schemas directly, no code-generation required
* Kafka schema registry is optional, use schemas directly from files
* Support for SSL/TLS and using certificates for secure auth

## Launch Webinar
* Includes an explanation and demo, you can watch it [on YouTube](https://youtu.be/xapqNmZoolE?si=UGKB3RnYnzoi9g1H).

## License
### Runtime License:
To run Karate tests using this library, you need a license from Karate labs. You can email info@karatelabs.io and request a license.

### Developer License
To develop and run feature files from the IDE you need to upgrade to the paid versions of Karate Labs official plugins for [IntelliJ](https://github.com/karatelabs/intellij-plugin) or [VS Code](https://github.com/karatelabs/vscode-extension).

## Setup
You need a Maven or Gradle project. Please use the latest available version. The dependency info can be found here: https://central.sonatype.com/artifact/io.karatelabs/karate-kafka

The `karate.lic` file you receive should be placed in a `.karate` folder in your project root. You can also change the default path where the license is expected - by setting a `KARATE_LICENSE_PATH` environment property.
 
## Sample Project
You can find a sample project here: [Karate Kafka Example](https://github.com/karatelabs/karate-examples/blob/main/kafka/README.md)

## Syntax

Note that [variables](https://github.com/karatelabs/karate#native-data-types) and JSON [embedded expressions](https://github.com/karatelabs/karate#embedded-expressions) will work just like you expect in Karate.

### `configure kafka`
Any valid Kafka configuration can be set this way. For example:

```cucumber
* configure kafka =
"""
{ 
  'bootstrap.servers': 'localhost:29092',
}
"""
```

For an example of configuring MTLS / SSL, refer: [kafka-mtls-example](https://github.com/karatelabs/karate-examples/blob/main/kafka-mtls/README.md).

### `karate.channel()`
This is the unified way to initialize Karate's support for any async protocol. For Kafka you do this:

```cucumber
* def channel = karate.channel('kafka')
```

After you have a reference to a `channel` object, you can call methods on it to [register](#channelregister) a message schema or initialize a [consumer](#channelconsumer) or [producer](#channelproducer).

### `channel.register()`
Set up mappings from JSON to Avro or Protobuf if needed. For example:

```cucumber
* channel.register({ name: 'hello', path: 'classpath:karate/hello.avsc' })
```

The name you give here can be referenced later in [`consumer.schema`](#schema) or [`producer.schema`](#schema).

For Protobuf, you need to also specify the message name as a `*.proto` file can have multiple definitions.

```cucumber
* channel.register({ name: 'hello-proto', path: 'classpath:karate/hello.proto', message: 'Hello' })
```

If your protobuf files import other files, you can specify "search roots" as follows:

```cucumber
* channel.register({ name: 'hello-proto', path: 'classpath:karate/hello.proto', message: 'Hello', roots: ['classpath:karate'] })
```

### `channel.consumer()`
Async handling requires a little more complexity than simple API tests, but `karate-kafka` keeps it simple. Here is an example:

```cucumber
* def consumer = channel.consumer()
* consumer.topic = 'test-topic'
* consumer.start()
```

Note how the syntax is future-proof, and support for other async protocols such as [`grpc`](../karate-grpc/README.md) and [`websocket`](../karate-websocket/README.md) is very similar.

Typically you name the returned variable from `channel.consume()` as `consumer`. Now you can set properties before calling [`consumer.start()`](#consumerstart).

Behind the scenes a new Kafka consumer with a fresh group-id is created. Please provide feedback if you need a different model for your environment.

### `consumer.topic`
Set the topic.

### `consumer.count`
Defaults to 1. This is how you tell Karate how many messages to wait for when consuming.

### `consumer.schema`
When consuming, refer to a previously [registered](#channelregister) schema.

### `consumer.filter()`
Optional way to filter for only some kinds of messages to [collect](#consumercollect).

You can use JS functions and be very dynamic. For example:

```cucumber
* consumer.filter = x => x.key != 'zero'
```

### `consumer.timeout`
You can set a timeout (in milliseconds) so that you can stop the test if messages do not appear within a reasonable time.

```cucumber
* consumer.count = 1
* consumer.topic = 'test-topic'
* consumer.timeout = 5000
* consumer.start()
```

### `consumer.start()`
You have to call this to start the listener process. To complete the test flow, you have to call [`consumer.collect()`](#consumercollect)

### `consumer.collect()`
Since Kafka and async listeners can span or "collect" multiple messages, this is always an array. Within each object you can unpack the `key`, `offset`, `headers` and `value`. Everything is JSON just like you expect in Karate.

```cucumber
* def response = consumer.collect()
* match response[0].key == 'first'
* match response[0].headers == { foo: 'bar1', baz: 'ban1' }
```

### `consumer.pop()`
Since in many cases you need only one message, this does a [`collect()`](#consumercollect) and gets the first result message in one-shot. It means you can avoid using an array-index to refer to the collected message.

For example:

```cucumber
* def response = consumer.pop()
* match response.key == 'first'
* match response.value == { message: 'hello', info: { first: 1, second: true } }
```

### `channel.producer()`
This API is the counterpart of [`channel.consumer()`](#channelconsumer) and is focused on the business of sending messages. Just like the consumer, you can set the topic and schema. You also have options to set the headers, key and value of the message being sent. To finally send a message, call [`producer.send()`](#producersend)

### `producer.topic`
Declare the topic to which a message should be sent.

### `producer.schema`
Declare that a previously [registered](#channelregister) schema will be used for the message being sent.

### `producer.headers`
When producing, set all Kafka headers in one shot.

For example:
```cucumber
* producer.headers = { foo: 'bar1', baz: 'ban1' }
```

### `producer.key`
When producing, set the Kafka message key

### `producer.value`
When producing, set the Kafka message value. If [`producer.schema`](#producerschema) was set, the JSON will be converted to Avro automatically.

Example:

```cucumber
* producer.value = { message: 'hello', info: { first: 1, second: true } }
```

If you have a multi-line JSON message value, you have to do it in two steps:

> This will be improved in a future version of Karate.

```cucumber
  * def value =
  """
  {
      "meta": {
          "metaId": "123",
          "metaType": "AAA",
          "metaChildren": [{ "name": "foo", "status": "ONE" }]
      },
      "payload": {
          "payloadId": "456",
          "payloadType": null,
          "payloadEnum": "FIRST",
          "payloadChild": {"field1": "foo", "field2": "bar"}
      }
  }
  producer.value = value
```

## Example

Here is a simple example that sends plain JSON (serialized to bytes) and listens on the same topic.

```cucumber
Feature: karate-kafka demo

Background:
* configure kafka =
"""
{ 
  'bootstrap.servers': '127.0.0.1:29092'
}
"""

Scenario:
* def channel = karate.channel('kafka')
* def consumer = channel.consumer()
* consumer.topic = 'test-topic'
* consumer.count = 1
* consumer.start()

* def producer = channel.producer()
* producer.topic = 'test-topic'
* producer.key = 'first'
* producer.value = { message: 'hello', info: { first: 1, second: true } }
* producer.send()

* def response = consumer.pop()
* match response.key == 'first'
* match response.value == { message: 'hello', info: { first: 1, second: true } }
```
