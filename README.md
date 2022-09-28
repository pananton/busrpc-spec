# Overview

**Busrpc** is an RPC framework which relies on a generalized message bus/queue/broker (for example, [NATS](https://nats.io/) or [RabbitMQ](https://rabbitmq.com/)) as a transport layer and [protocol buffers](https://developers.google.com/protocol-buffers) as message format.

The project consists of the following components:
* API [specification](./docs/busrpc.md) defining general rules to be followed by busrpc backends
* API [specializations](Specializations) defining bus-dependent rules which can not be placed to common specification
* Command-line [tool](./tools/busrpc-tool.md) providing useful commands for busrpc backends developers (checking protocol for conformance, generating documentation, etc.)
* Bus-dependent [clients](Clients) for testing/tracing running busrpc backends
* Bus-dependent [libraries](Libraries) for busrpc backends development

# Specializations

Busrpc [specification](./busrpc.md) tries to stay as general as possible and leaves some aspects of API design unspecified, because otherwise specification may depend on specific message bus/queue/broker technology. This unspecified aspects are defined in a seperate bus-dependent documents called *specializations*.

Currently the project provides the following specializations:
* NATS [specialization](./docs/specializations/nats-busrpc.md)

# Clients

# Libraries

Currently there are no ready libraries providing easy-to-use APIs for accessing busprc backends. Note, that such libraries (when created) will differ in 2 dimensions: language (C++, Go, Rust, ...) and targeted message bus/queue/broker.  
