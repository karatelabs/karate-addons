# Karate-Webhook

> [!NOTE]
> Not Yet Released

Karate-Webhook adds first-class support for integrating a [Webhook](https://en.wikipedia.org/wiki/Webhook) flow into your tests. The challenge of listening for incoming HTTP requests is solved via an elegant API.

## Highlights
* Unified syntax similar to HTTP but focused on Webhook
* Seamlessly integrate Webhook callbacks into API tests
* Set up multiple async listeners if needed
* Support for parallel execution and performance testing
* Simple Docker based solution that integrates into your CI/CD or cloud
* Local-first, no data leaves your firewall or cloud boundary

## License
### Runtime License:
To run Karate tests using this library, you need a license from Karate labs. You can email info@karatelabs.io and request a license.

### Developer License
To develop and run feature files from the IDE you need to upgrade to the paid versions of Karate Labs official plugins for [IntelliJ](https://github.com/karatelabs/intellij-plugin) or [VS Code](https://github.com/karatelabs/vscode-extension).

## Setup
You need a Maven or Gradle project. Please use the latest available version. The dependency info can be found here: https://central.sonatype.com/artifact/io.karatelabs/karate-webhook

The `karate.lic` file you receive should be placed in a `.karate` folder in your project root. You can also change the default path where the license is expected - by setting a `KARATE_LICENSE_PATH` environment property.
 