# Karate-gRPC

> [!NOTE] Not Yet Released

Karate-gRPC adds first-class support for testing [gRPC](https://grpc.io/). The challenge of sending and consuming messages in async fashion is solved via an elegant API.

## Highlights
* Unified syntax similar to HTTP but focused on gRPC
* Mix HTTP and gRPC calls within the same test flow
* Set up multiple async connections if needed
* Support for parallel execution
* Support for performance testing
* Express data as JSON and leverage Karate's powerful assertions
* Use your `*.proto` files directly, no code-generation required
* Automatic Protobuf to JSON conversion
* Support for SSL/TLS and using certificates for secure auth

## License
### Runtime License:
To run Karate tests using this library, you need a license from Karate labs. You can email info@karatelabs.io and request a license.

### Developer License
To develop and run feature files from the IDE you need to upgrade to the paid versions of Karate Labs official plugins for [IntelliJ](https://github.com/karatelabs/intellij-plugin) or [VS Code](https://github.com/karatelabs/vscode-extension).

## Setup
You need a Maven or Gradle project. Please use the latest available version. The dependency info can be found here: https://central.sonatype.com/artifact/io.karatelabs/karate-grpc

The `karate.lic` file you receive should be placed in a `.karate` folder in your project root. You can also change the default path where the license is expected - by setting a `KARATE_LICENSE_PATH` environment property.
 