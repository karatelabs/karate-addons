# Karate-gRPC

Karate-gRPC adds first-class support for testing [gRPC](https://grpc.io/). The challenge of sending and consuming messages in async fashion is solved via an elegant API.

## Highlights
* Unified syntax similar to HTTP but focused on gRPC
* Mix HTTP and gRPC calls within the same test flow
* Support for all gRPC modes: Unary, Server Streaming, Client Streaming and Bidirectional
* Set up multiple async connections if needed
* Support for parallel execution
* Support for performance testing
* Express data as JSON and leverage Karate's powerful assertions
* Use your `*.proto` files directly, no code-generation required
* Automatic Protobuf to JSON conversion
* Support for SSL/TLS and using certificates for secure auth

<img src="https://github.com/karatelabs/karate-addons/assets/915480/b3ffac34-05a4-4711-a51c-db0745c65e99" height="550px"/>

## License
### Runtime License:
To run Karate tests using this library, you need a license from Karate labs. You can email info@karatelabs.io and request a license.

### Developer License
To develop and run feature files from the IDE you need to upgrade to the paid versions of Karate Labs official plugins for [IntelliJ](https://github.com/karatelabs/intellij-plugin) or [VS Code](https://github.com/karatelabs/vscode-extension).

## Setup
You need a Maven or Gradle project. Please use the latest available version. The dependency info can be found here: https://central.sonatype.com/artifact/io.karatelabs/karate-grpc

The `karate.lic` file you receive should be placed in a `.karate` folder in your project root. You can also change the default path where the license is expected - by setting a `KARATE_LICENSE_PATH` environment property.
 
## Sample Project
You can find a sample project here: [Karate gRPC Example](https://github.com/karatelabs/karate-examples/blob/main/grpc/README.md).

## Syntax

### `karate.channel()`

Async handling requires a little more complexity than simple API tests, but `karate-grpc` still keeps it simple. Here is an example:

```cucumber
    * def session = karate.channel('grpc')

    * session.host = 'localhost'
    * session.port = 9555
    * session.proto = 'classpath:karate/hello.proto'
    * session.service = 'HelloService'
    * session.method = 'Hello'

    * session.send({ name: 'John' })
    * match session.pop() == { message: 'hello John' }
```

Note how the syntax is future-proof, and support for other async protocols such as [`kafka`](../karate-kafka/README.md) and [`websocket`](../karate-websocket/README.md) is very similar.

Typically you name the returned variable from `karate.channel()` as session. Now you can set properties before calling [`session.start()`](#sessionstart) or [`session.send()`](#sessionsend).

Note how all the connection and service / method parameters can be set on the session before starting to send and receive messages.

## Variables

Note that [variables](https://github.com/karatelabs/karate#native-data-types) defined using `def` or in [`karate-config.js`](https://github.com/karatelabs/karate#karate-configjs) will work just like you expect in Karate. The syntax leans more towards "pure JS" than Karate's HTTP keywords.

> This means that within JSON, variables will work directly without needing to use the `'#(variableName)'` syntax for [embedded expressions](https://github.com/karatelabs/karate#embedded-expressions) that you may be used to.

## Config

The properties that you can set on the object returned by [`karate.channel()`](#karatechannel) are as follows:

| Name      | Description       |
| --------- | ----------------- |
| host      | Host name         |
| port      | Port number (string variable references also work for convenience) |
| proto     | Relative / Prefixed path to `*.proto` file |
| protoRoots | (optional) array of relative / prefixed paths which are the search "roots" for dependencies or proto files which are "imports"
| service   | gRPC service name |
| method    | gRPC method name  |
| count     | (defaults to 1) the number of response messages to wait for before the [`collect()`](#sessioncollect) or [`pop()`](#sessionpop) method returns, see [example](#sessioncount) |
| filter    | (optional) A JS function, see [example](#sessionfilter) |
| stream    | (defaults to `false`) Enable client-side streaming, so you have to call [`flush()`](#sessionflush) after [send()](#sessionsend) |
| trustCert | (optional) Trusted authority certificate, see [TLS](#tls) |
| clientCert | (optional) Client certificate for "mutual auth", see [TLS](#tls) |
| clientKey | (optional) Client key for "mutual auth", see [TLS](#tls) |

### `session.config`
For convenience, it is possible to set multiple properties on the session in one line.

So instead of:

```cucumber
    * def session = karate.channel('grpc')

    * session.host = 'localhost'
    * session.port = 9555
    * session.proto = 'classpath:karate/hello.proto'
```

You can do:

```cucumber
    * def session = karate.channel('grpc')

    * session.config = { host: 'localhost', port: 9555, proto: 'classpath:karate/hello.proto' }
```

And `session.config` can be called multiple times within a test flow, and you can over-write existing properties. This can be convenient for long flows.

## Methods

For actions such as sending messages or "collecting" async responses, you call methods on the session object. Since these are JS method invocations, they use round brackets and may take method arguments.

### `session.send()`

Once you have a session and have set connection parameters, you can start sending messages.

```cucumber
    * session.send({ name: 'John' })
    * match session.collect() == [{ message: 'hello John' }]
```

Although there is technically a `session.start()` method behind the scenes, it is automatically called behind the scenes for the first message sent, for convenience.

Refer to the [example](https://github.com/karatelabs/karate-examples/blob/main/grpc/src/test/java/karate/hello.feature) for a working demo.

Conversations are easy, you can continue to `send()` and `collect()` as long as needed:

```cucumber
    * session.send({ name: 'John' })
    * match session.pop() == { message: 'hello John' }
    * session.send({ name: 'Smith' })
    * match session.pop() == { message: 'hello Smith' }
```

### `session.collect()`

As seen above, `collect()` is designed to return an array of messages, since we are dealing with async channels. But since in many cases, you would be running with `session.count = 1` there is a convenient [`pop()`](#sessionpop) method that returns the first message in the "buffer".

### `session.pop()`

The example above can be re-written as follows since `session.count = 1`:

```cucumber
    * session.send({ name: 'John' })
    * match session.pop() == { message: 'hello John' }
```

### `session.count`

Since this defaults to `1` the first message received will stop the "wait mode" and "un-block" any call to [`collect()`](#sessioncollect) or [`pop()`](#sessionpop) after a [`send()`](#sessionsend).

For example, here is how to tell the test to wait for 3 messages:

```cucumber
    * session.count = 3
    * session.send({ name: 'John' })
    * def result = session.collect()
    * match result ==
      """
      [
        { message: 'hello John 1' },
        { message: 'hello John 2' },
        { message: 'hello John 3' }
      ]
      """

```

## Client Side Streaming

This is set up by doing `session.stream = true` as shown below.

### `session.stream`

The default is `false`, but when you want to simulate multiple messages sent by the client asynchonously - you can set `session.stream = true`.

This also means that you should call [`flush()`](#sessionflush) if you have to signal that the client is "done" or "complete". This is needed only in cases where the server waits for the client to "complete" before responding.

### `session.flush()`

Here is an example of client-side streaming, as explained in the sections above.


```cucumber
    * session.stream = true
    * session.send({ name: 'John' })
    * session.send({ name: 'Smith' })
    * session.send({ name: 'Jane' })
    * session.flush()
    * match session.pop() == { message: 'hello [John, Smith, Jane]' }
```

Here above, the server happened to be programmed to send a single message when the client has signalled that it is "complete".

Refer to the [example](https://github.com/karatelabs/karate-examples/blob/main/grpc/src/test/java/karate/hello.feature) for a working demo.

## Bi-Di

Bi-directional streaming is simple, combining all the concepts introduced above, mainly [`session.stream`](#sessionstream) and [`session.count`](#sessioncount).

Here we ask the test to wait until 3 messages were received from the server. Meanwhile we can send multiple messages.

```cucumber
    * session.stream = true
    * session.count = 3
    * session.send({ name: 'John' })
    * session.send({ name: 'Smith' })
    * session.send({ name: 'Jane' })
    * match session.collect() ==
      """
      [
        { message: 'hello [John]' },
        { message: 'hello [John, Smith]' },
        { message: 'hello [John, Smith, Jane]' }
      ]
      """
```

Refer to the [example](https://github.com/karatelabs/karate-examples/blob/main/grpc/src/test/java/karate/hello.feature) for a working demo.

### `session.filter`

Optional way to filter for only some kinds of messages to [collect](#sessioncollect).

You can use JS functions and be very dynamic. For example:

```cucumber
    * session.filter = x => x.message != 'zero'
```

## TLS

If the server only accepts encrypted connections, you can configure a trusted certificate store (CA) as follows:

```cucumber
    * session.trustCert = 'classpath:ca.crt'
```

You can also present a client certificate and key as follows. Here we are also demonstrating the use of [`session.config`](#sessionconfig) for reducing the lines of code.

```cucumber
    * session.config = { trustCert: 'classpath:ca.crt', clientCert: 'classpath:client.crt', clientKey: 'classpath:client.pem' }    
```

The [Karate gRPC example](https://github.com/karatelabs/karate-examples/blob/main/grpc/README.md) includes working samples of [TLS](https://github.com/karatelabs/karate-examples/blob/main/grpc/src/test/java/karate/hello-tls.feature) and [TLS with "mutual auth"](https://github.com/karatelabs/karate-examples/blob/main/grpc/src/test/java/karate/hello-tls-mutual.feature).

The steps taken to create the (self-signed) certificates for the CA, server and client for the demo are explained in this file for your reference: [crypto.txt](https://github.com/karatelabs/karate-examples/blob/main/grpc/src/test/java/crypto.txt).

