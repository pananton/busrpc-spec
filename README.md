# Overview

**Busrpc** is an RPC framework which relies on a generalized message bus/queue/broker (for example, [NATS](https://nats.io/) or [RabbitMQ](https://rabbitmq.com/)) as a transport layer and [protocol buffers](https://developers.google.com/protocol-buffers) as message format.

The project consists of the following components:
* API [specification](./docs/busrpc.md) defining general rules to be followed by busrpc backends
* API [specializations](#specializations) defining bus-dependent rules which can not be placed to common specification
* Busrpc protobuf [style guide](./docs/style.md)
* Command-line [tool](./tool/README.md) providing useful commands for busrpc backends developers (checking protocol for conformance, generating documentation, etc.)
* Bus-dependent [clients](#clients) for testing/tracing running busrpc backends
* Bus-dependent [libraries](#libraries) for busrpc backends development (to be done)
* [Example](https://github.com/pananton/busrpc/tree/main/example) of a busrpc API for a fictional restaurtant

# Specializations

Busrpc specification tries to stay as general as possible and leaves some aspects of API design unspecified to avoid dependency on a specific message bus/queue/broker implementation. This unspecified aspects are defined in a seperate bus-dependent documents called *specializations*.

Currently the project provides the following specializations:
* NATS [specialization](./docs/specializations/nats-busrpc.md)

# Clients

Clients allow developers to test and trace running busprc backends and usually provide at least the following commands:
* `call` - to call a busrpc method
* `impl` - to implement a busrpc method
* `trace` - to trace calls of busrpc methods

Currently the project provides the following cliens:
* NATS [busrpc client](https://github.com/pananton/nats-busrpc-cli)

# Libraries

Currently no client libraries designed specifically for busrpc-based APIs exist, which means that you will probably implement one for your platform (combination of programming language and particular message bus/queue/broker). It's usually not a big deal, because you will probably use some client library from message bus/queue/broker developer (for example, see [here](https://nats.io/download/#nats-clients) for NATS client libraries or [here](https://www.rabbitmq.com/devtools.html) for RabbitMQ client libraries) and implement busrpc-specific wrappers over it. Contributions of such busrpc client libraries are highly appreciated!

# Contributing

Any contributions are highly appreciated:
* suggestions to specification
* specializations for different message bus/queue/broker techonologies
* new commands for busrpc development tool
* clients for testing/tracing busrpc backends
* busrpc client libraries

If you want to discuss/suggest some feature, create an issue.

If you want to contribute code to busprc development tool, fork this repository, make your changes and create a pull request.

If you have implemented new busrpc client or library, create an issue with a link to it and I will add your client/library to the list.
