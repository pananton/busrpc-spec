# Busrpc specification

This document contains general information for developers of busrpc microservices and tools: terminology, network model, API design, etc.

* [Introduction](#introduction)
* [Message bus model](#message-bus-model)
* [Design](#design)
  * [Terminology](#terminology)
  * [Signaling errors](#signaling-errors)
  * [Class description](#class-description)
  * [Method description](#method-description)
  * [Service description](#service-description)
  * [Directory layout](#directory-layout)
  * [Types visibility](#types-visibility)
* [Documentation commands](#documentation-commands)
* [Specializations](#specializations)

# Introduction

As appears from it's name, busrpc framework stands on two pillars.

First of all, it is a form of a remote procedure call (RPC) technology (like [grpc](https://grpc.io/), [json-rpc](https://www.jsonrpc.org/) and similar) used for inter-service communication.

Secondly, it relies on a message bus/queue/broker component as a transport layer. Usual RPC implementations operate in a peer-to-peer manner which means that communicating parties need to connect directly to each other. This is probably ok for the systems with small number (1-5) of services, but does not suite well for microservice backends where number of services can easily surpass one hundred. Instead, microservice backends utilize a dedicated component (called message bus/queue/broker) as a central point of communication which is responsible for inter-service message delivery. This greatly simplifies system configuration (only message bus address is required to access any part of the system API), service deployment (because all services are loosely-coupled in this scheme) and administration.

Busrpc framework global goal is to offer a turnkey solution for backend developers who apply microservice architecture for their projects. To acheive this, busrpc framework is designed with the following ideas in mind:
* framework should be applicable for a variety of message buses and programming languages
* framework should establish unambiguous and intuitive terminology to facilitate framework learning
* promoted API design should help to solve typical problems arising when designing custom backend API (how to extract API entities from the business domain, name and document them, etc.) 
* framework should provide tool for enforcing promoted API design (for example, as part of CI/CD pipeline) in third-party development process
* promoted API design should be strict and complete to facilitate development of tools which can be used for all conforming implementations
* underlying network protocol should be extensible and should support switching between binary (for higher throughoutput and lesser latency) and text (for human-readability) format

# Message bus model

One of the goals of the busrpc framework is to be applicable for a variety of message bus/queue/broker technologies. For this reason, busrpc specification relies on an abstract message bus model instead of any particular implementation like NATS or RabbitMQ. Note, that if message bus implementation violates this model, it probably can't be used along with busrpc framework.

The message bus model provides two basic operations:
1. `PUBLISH(topic, message, [replyTo])` - to send arbitrary `message` on a `topic`
2. `SUBSCRIBE(topic)` - to listen for messages published on a `topic`

*Topic* is essentially a sequence of characters used by the message bus to route messages from publisher to subscriber. On a closer look, it is a list of words separated by a special character `<sep>`. Most (if not all) message bus techonologies use dot (`.`) as separator, however described model does not require this (still dot will be used in the examples thorought this document for simplicity).

Words constituiting a topic usually form a hierarchy, for example: `time.us`, `time.us.east`, `time.us.east.atlanta`. Abstract bus model supports the following wildcards which can be used in the `SUBSCRIBE` operation:
* `<any1>` (usually `*`) - matches a single word, for example `time.<any1>.east` matches `time.us.east` and `time.eu.east`
* `<anyN>` - matches 1 to many or 0 to many words at the end of a topic, for example `time.us.<anyN>` matches `time.us.east`, `time.us.east.atlanta` and may or may not match `time.us` (makes no difference for busrpc specification)

Note that core publish/subscribe mechanism implies one-way message flow (from publisher to subscriber). To enable request/response two-way message flow, `PUBLISH` operation can also be called with a `replyTo` parameter containing topic on which publisher expects to receive a response for his request. In that case subscriber will call `PUBLISH(replyTo, response)` after original request is handled.

# Design

Busrpc API design is formulated in terms of object-oriented programming. This terms when applied for busrpc API have similar meaning as in OOP, however should not be treated as equivalent.

Every busrpc-compliant API is made up the following building blocks:
* **namespace** - a collections of somehow related *classes*
* **class**
* **namespace** is a collection of closely related *classes* representing entities 

Any busrpc API consists of *namespaces* representing distinct API subdomains. Inside That means it's building blocks are *classes*, which contain arbitrary number of *methods*. Each class is placed into some *namespace* representing an API subdomain. Each namespace, class and method has own *scope* which  





# Documentation commands

# Specializations

Some aspects of the busrpc API design were intentionally left unspecified in this document to avoid dependency on a particular message bus/queue/broker implementation. This unspecified aspects are defined in a seperate bus-dependent documents called *specializations*.

Currently the following specializations exist (more to be added):
* NATS [specialization](./docs/specializations/nats-busrpc.md)
