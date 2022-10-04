# Overview

**Busrpc** framework is an RPC framework designed with [microservice architecture](https://en.wikipedia.org/wiki/Microservices) principles in mind. It relies on a generalized message bus/queue/broker (for example, [NATS](https://nats.io/) or [RabbitMQ](https://rabbitmq.com/)) as a transport layer and [protocol buffers](https://developers.google.com/protocol-buffers) as message format.

The project consists of the following components:
* Technical documentation, namely protocol [specification](./docs/busrpc.md) with accompanying bus-dependent [specializations](./docs/busrpc.md#specializations) and a [style guide](./docs/style.md) for protocol protobuf files
* Command-line development [tool](./devtool/README.md) for busrpc microservice developers
* Bus-dependent [clients](#clients) for testing/tracing running busrpc backends
* Bus-dependent [libraries](#libraries) for busrpc microservice development
* [Example](https://github.com/pananton/busrpc/tree/main/example) of a busrpc API for a fictional restaurtant "Weeping Willow" which is used throughout documentation

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
