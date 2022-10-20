# Busrpc specification

This document contains general information for developers of busrpc microservices and tools: terminology, network model, API design, etc.

* [Introduction](#introduction)
* [Message bus model](#message-bus-model)
* [Design](#design-and-terminology)
  * [Class](#class)
  * [Structure](#structure)
  * [Enumeration](#enumeration)
  * [Namespace](#namespace)
  * [Endpoint](#endpoint)
  * [Service](#service)
  * [Type visibility](#type-visibility)
* [Protocol](#protocol)
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

Message bus model provides two basic operations:
1. `PUBLISH(topic, message, [replyTo])` - to send arbitrary `message` on a `topic`
2. `SUBSCRIBE(topic)` - to listen for messages published on a `topic`

*Topic* is essentially a sequence of characters used by the message bus to route messages from publisher to subscriber. On a closer look, it is a list of words separated by a special character `<topic-word-sep>`. Most (if not all) message bus techonologies use dot (`.`) as separator, however described model does not require this (still dot will be used in the examples thorought this document for simplicity).

Words constituiting a topic usually form a hierarchy, for example: `time.us`, `time.us.east`, `time.us.east.atlanta`. Abstract bus model supports the following wildcard characters which can be used in the `SUBSCRIBE` operation:
* `<topic-wildcard-any1>` - matches a single word, for example `time.<topic-wildcard-any1>.east` matches `time.us.east` and `time.eu.east`
* `<topic-wildcard-anyN>` - matches 1 to many or 0 to many words at the end of a topic, for example `time.us.<topic-wildcard-anyN>` matches `time.us.east`, `time.us.east.atlanta` and may or may not match `time.us` (makes no difference for busrpc specification)

Note that core publish/subscribe mechanism implies one-way message flow (from publisher to subscriber). To enable request/response two-way message flow, `PUBLISH` operation provides optional `replyTo` parameter containing topic on which publisher expects to receive a response for his request. In that case subscriber will call `PUBLISH(replyTo, response)` to send the response.

# Design

Busrpc API design is based on the concepts from object-oriented programming. This allows busrpc to re-use well-known OOP terminology and stay familiar for newcomers (note hovewer that same terms from busrpc API design and object-oriented design may differ in some aspects and should not be treated as exactly equivalent). Moreover, we believe that many well-established and time-tested object-oriented design principles and decomposition strategies can also be applied for a good microservice backend API, which means that developers' OOP experience might come in handy in the context of the busrpc framework.

## Class

Busrpc **class** is a model of a similarly arranged entities from the API business domain. By similar arrangement we mean that all entities modelled by a class have the same format of internal state and expose the same set of supported operations, which are called **methods** as it is accepted in OOP. All together, class methods form an **interface** of a class.

**Object** of a class represents a concrete entity from the set of modelled entities. It is characterized by concrete value of internal state and an immutable **object identifier**, which uniquelly identifies object throughout the system.

Methods may be bound to a concrete object or to a class as a whole. The latter are called **static methods**. 

**Method call** is a network request containing method parameters and, optionally, identifier of the object for which method is called (not needed for static method calls). **Method result** is a network response containing either method return value or an exception signalling abnormal method completion. Exact format of this messages is described in the [Protocol](#protocol) section.

## Structure

Busrpc *structure* is an alternative term for a protobuf `message` introduced for consistency with OOP terminology. Structures are busrpc wire types, i.e. every busrpc network message is represented by some structure.

This specification introduces several *predefined* structures which are used to describe:
* object identifier
* method parameters
* method return value
* method exception
* service configuration
* type-erased request and response

Additionally, several structures were introduced to provide busrpc client libraries with useful information about busrpc entities (classes, methods, etc.). Such structures are called *descriptors*. Descriptors are usually never sent over the network - in fact, they even do not have any fields, only nested type definitions.

Description of the predefined structures and descriptors can be found in the [Protocol](#protocol) section.

## Enumeration

Busrpc *enumeration* corresponds directly to the protobuf `enum`.

## Namespace

*Namespace* is a group of somehow related busrpc classes, structures and enumerations. For example, namespace may contain types which are part of the same application subdomain.

## Endpoint

## Service

Busrpc *service* is an application implementing and/or invoking busrpc class methods. 

## Type visibility

# Protocol

# Documentation commands

# Specializations

Some aspects of the busrpc API design were intentionally left unspecified in this document to avoid dependency on a particular message bus/queue/broker implementation. This unspecified aspects are defined in a seperate bus-dependent documents called *specializations*.

Currently the following specializations exist (more to be added):
* NATS [specialization](./docs/specializations/nats-busrpc.md)
