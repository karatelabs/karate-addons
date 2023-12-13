# Karate Kafka

Karate Kafka adds first-class support for testing Kafka for both producing and consuming sides. Additional keywords make it easy to produce Kafka messages the same way you are used to making HTTP requests. The challenge of consuming messages in async fashion is solved via an elegant API.

## Highlights
* Unified syntax similar to HTTP but focused on Kafka
* Flexibility to set up multiple async listeners
* Support for parallel execution
* Support for performance testing 
* Express data and assertions as JSON
* Avro or plain JSON serialization support
* Use Avro schemas directly, no code-generation required

## Launch Webinar
* Includes an explanation and demo, you can watch it [on YouTube](https://youtu.be/xapqNmZoolE?si=UGKB3RnYnzoi9g1H).

## License
### Runtime License:
To run Karate tests using this library, you need a license from Karate labs. You can email info@karatelabs.io and request a license.

### Developer License
To develop and run feature files from the IDE you need to upgrade to the paid versions of Karate Labs official plugins for [IntelliJ](https://plugins.jetbrains.com/plugin/19232-karate) or [VS Code](https://marketplace.visualstudio.com/items?itemName=karatelabs.karate).

## Setup
You need a Maven or Gradle project. The dependency info can be found here: https://central.sonatype.com/artifact/io.karatelabs/karate-kafka

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

### `register`
Set up mappings from JSON to Avro if needed. For example:

```cucumber
* register { name: 'hello', path: 'classpath:karate/hello.avsc' }
```

The name you give here can be referenced later in [`session.schema`](#sessionschema) and the `schema` keyword.

### `schema`
When producing, refer to a previously [registered](#register) schema.

### `header`
When producing, set one Kafka header at a time

For example:

```cucumber
* header foo = 'bar'
```

### `headers`
When producing, set all Kafka headers in one shot.

For example:
```cucumber
* headers { foo: 'bar1', baz: 'ban1' }
```

### `key`
When producing, set the Kafka message key

Example:
```cucumber
* key 'first'
```

### `value`
When producing, set the Kafka message value. If you specify a [`schema`](#schema) the JSON will be converted to Avro automatically.

Example:

```cucumber
* value { message: 'hello', info: { first: 1, second: true } }
```

You can also use the multi-line "docstring" syntax.

```cucumber
    * value
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
```

### `karate.consume()`
Async handling requires a little more complexity than simple API tests, but karate-kafka still keeps it simple. Here is an example:

```cucumber
* def session = karate.consume('kafka')
* session.topic = 'test-topic'
* session.start()
```

Note how the syntax is future-proof, and support for `grpc` and `websockets` is coming soon.

Typically you name the returned variable from `karate.consume()` as session. Now you can set properties before calling [`session.start()`](#sessionstart).

Behind the scenes a new Kafka consumer with a fresh group-id is created. Please provide feedback if you need a different model for your environment.

### `session.topic`
Set the topic.

### `session.count`
Defaults to 1. This is how you tell Karate how many messages to wait for when consuming.

### `session.schema`
When consuming, refer to a previously [registered](#register) schema.

### `session.filter()`
Optional way to filter for only some kinds of messages to [collect](#sessioncollect).

You can use JS functions and be very dynamic. For example:

```cucumber
* session.filter = x => x.key != 'zero'
```

### `session.start()`
You have to call this to start the listener process. To complete the test flow, you have to call [`session.collect()`](#sessioncollect)

### `session.collect()`
Since Kafka and async listeners can span or "collect" multiple messages, this is always an array. Within each object you can unpack the `key`, `offset`, `headers` and `value`. Everything is JSON just like you expect in Karate.

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
* def session = karate.consume('kafka')
* session.topic = 'test-topic'
* session.count = 1
* session.start()

* topic 'test-topic'
* key 'first'
* value { message: 'hello', info: { first: 1, second: true } }
* produce kafka

* def response = session.collect()
* match response[0].key == 'first'
* match response[0].value == { message: 'hello', info: { first: 1, second: true } }
```
