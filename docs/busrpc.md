# Busrpc specification

This document contains general information for developers of busrpc microservices and tools: terminology, network model, API design, etc.

* [Introduction](#introduction)
* [Message bus model](#message-bus-model)
* [Design](#design)
  * [Class](#class)
  * [Service](#service)
  * [Structure](#structure)
  * [Enumeration](#enumeration)
  * [Namespace](#namespace)
  * [Endpoint](#endpoint)
  * [Type visibility](#type-visibility)
* [Protocol](#protocol)
  * [Directory layout](#directory-layout)
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

**Topic** is essentially a sequence of characters used by the message bus to route messages from publisher to subscriber. On a closer look, it is a list of words separated by a special character `<topic-word-sep>`. Most (if not all) message bus techonologies use dot (`.`) as separator, however described model does not require this (still dot will be used in the examples thorought this document for simplicity).

Words constituiting a topic usually form a hierarchy, for example: `time.us`, `time.us.east`, `time.us.east.atlanta`. Abstract bus model supports the following wildcard characters which can be used in the `SUBSCRIBE` operation:
* `<topic-wildcard-any1>` - matches a single word, for example `time.<topic-wildcard-any1>.east` matches `time.us.east` and `time.eu.east`
* `<topic-wildcard-anyN>` - matches 1 to many or 0 to many words at the end of a topic, for example `time.us.<topic-wildcard-anyN>` matches `time.us.east`, `time.us.east.atlanta` and may or may not match `time.us` (makes no difference for busrpc specification)

Note that core publish/subscribe mechanism implies one-way message flow (from publisher to subscriber). To enable request/response two-way message flow, `PUBLISH` operation provides optional `replyTo` parameter containing topic on which publisher expects to receive a response for his request. In that case subscriber will call `PUBLISH(replyTo, response)` to send the response.

# Design

Busrpc API design is based on the concepts from object-oriented programming. This allows busrpc to re-use well-known OOP terminology and stay familiar for newcomers (note hovewer that same terms from busrpc API design and object-oriented design may differ in some aspects and should not be treated as exactly equivalent). Moreover, we believe that many well-established and time-tested object-oriented design principles and decomposition strategies can also be applied for a good microservice backend API, which means that developers' OOP experience might come in handy in the context of the busrpc framework.

This section introduces the concepts of busrpc API, implementation details are covered in the [Protocol](#protocol) section.

## Class

Busrpc **class** is a model of a similarly arranged entities from the API business domain. By similar arrangement we mean that all entities modelled by a class have the same format of internal state and expose the same set of supported operations, which are called **methods** as it is accepted in OOP. All together, class methods form an **interface** of a class.

**Object** of a class represents a concrete entity from the set of modelled entities. It is characterized by concrete value of internal state and an immutable **object identifier**, which uniquelly identifies object throughout the system.

Methods may be bound to a concrete object or to a class as a whole. The latter are called **static methods**. Busrpc specification also has a notion of a **static class**, which is a class without objects. Because of this, static class does not define an object identifier and every method of it's interface is treated as static. Static classes are mainly used to group related system-wide "utility" methods.

**Method call** is a network request containing method parameters and, optionally, identifier of the object for which method is called (not needed for static method calls). Some method parameters can additionally be defined as **observable**, which means that their values not only sent as payload of a call but also provide implementors with an ability to cherry-pick a subset of calls having a concrete values of the observable parameters (see [Endpoint](#endpoint) section for more information).

**Method result** is a network response containing either method return value or an exception signalling abnormal method completion.

## Service

**Service** is an application implementing and/or invoking busrpc class methods. This specification does not impose any limits on methods used by the service. For example, it is totally fine for a service to implement only a subset of class methods, or to implement/invoke methods from distinct classes. 

## Structure

Busrpc **structure** is an alternative term for a protobuf `message` introduced for consistency with OOP terminology. Structures are busrpc wire types, i.e. every busrpc network message is represented by some structure.

**Predefined structures** are structures which have special meaning determined by this specification. In particular, **descriptors** are predefined structures which provide busrpc client libraries with type information about busrpc entity (service, class, or method). Descriptors are usually never sent over the network - in fact, they even do not have any fields, only nested type definitions.

## Enumeration

Busrpc **enumeration** corresponds directly to a protobuf `enum`.

## Namespace

**Namespace** is a group of related busrpc classes, structures and enumerations. Typically namespace contains part of the API that corresponds to a single application subdomain.

## Endpoint

**Call endpoint** (or simply, endpoint) is a message bus topic to which method calls are published by a caller. Format of the call endpoint is `<namespace>.<class>.<method>.<object-id>[.<observable-params>].<eof>`, where:
* `<namespace>`, `<class>` and `<method>` are names of the namespace, class and method correspondingly
* `<object-id>` is identifier of an object for which method is called, or a reserved word representing null value for static methods
* optional `<observable-params>` is a list of topic words each representing a value of a single observable parameter
* `<eof>` is a reserved word which designates the end of endpoint

Some message bus topic formats, commonly used for subscribing for method calls, are also mapped to a named endpoints to establish common terminology. This mapping is provided in the following table.

| Topic                                                                                          | Endpoint  | Description                                       |
| ---------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------- |
| `<namespace>.<topic-wildcard-anyN>`                                                            | namespace | calls of any method of any class from a namespace |
| `<namespace>.<class>.<topic-wildcard-anyN>`                                                    | class     | calls of any method of a class                    |
| `<namespace>.<class>.<method>.<topic-wildcard-anyN>`                                           | method    | calls of a method                                 |
| `<namespace>.<class>.<topic-wildcard-any1>.<object-id>.<topic-wildcard-anyN>`                  | object    | calls bound to a specific object                  |
| `<namespace>.<class>.<method>.<topic-wildcard-any1>.<observable-params>.<topic-wildcard-anyN>` | value     | calls of a method with a specific value(s) of an                                                                                                                        observable parameter(s)                           |

## Type visibility

**Scope** is a part of a busrcp API where name of a structure or enumeration can be used to refer to the corresponding type. In that case we also say that busrpc structure/enumeration is **visible** in this scope.

The scope of a type is determined by the place in the directory hierarchy where the protobuf file containing it's definition is located. Additionally, scopes form a hierarchy in which types visible in the parent scope are also visible in all it's child scopes. The following scopes are defined by the busrpc specification (in the order from parent to child):
1. a single **global scope**
2. a **namespace scope** for each namespace
3. a **class scope** for each class
4. a **method scope** for each method

Limiting the scope of busrpc structures and enumerations developers can easily control which parts of their API can be affected when some type is updated. For example, if some structure defined in the class `C` scope needs to be updated, only class `C` methods should be checked for compatability.

# Protocol

Busrpc framework uses Google's [protocol buffers](https://developers.google.com/protocol-buffers) as a format of network messages. This mechanism was considered as most suitable for the busrpc framework needs because of the following features:
* support of all popular programming languages
* support for extension of protobuf file syntax with [custom options](https://developers.google.com/protocol-buffers/docs/proto3#customoptions)
* rich API for introspecting protobuf generated types, which facilitates custom tools and client libraries development
* easy conversion from binary data to text (JSON) and vice versa

## Directory layout

All busrpc protobuf files should be organized in the tree represented below. Names in angle brackets are placeholders which are assigned real names by specific API implementation. For simplicity, only a single namespace, class, method and service is represented. Of course, real API may contain arbitrary number of this entities structured in a similar way.

```
<busrpc-root-dir>/
├── api/
|   ├── busrpc.proto
│   ├── <namespace-dir>/
│       ├── <class-dir>/
│           ├── class.proto
│           ├── <method-dir>/
│               ├── method.proto
├── services/
│   ├── <service-dir>/
|       ├── service.proto
```
 
Components of the busrpc directory tree are:
* `<busrpc-root-dir>` is a busrpc **root directory**, which must contain two predefined directories: **API root directory** (`api`) and **services root directory** (`services`)
* `api/busrpc.proto` is a framework-provided file containing definitions of some predefined structures
* `api/<namespace-dir>` is called **namespace directory**; it contains definitions of all namespace classes
* `api/<namespace-dir>/<class-dir>` is called **class directory**; it contains definition of the class interface
* `api/<namespace-dir>/<class-dir>/class.proto` is called **class description file**; it must contain [class descriptor](#class-description-file) definition
* `api/<namespace-dir>/<class-dir>/<method-dir>` is called **method directory**; it contains definition of the class method
* `api/<namespace-dir>/<class-dir>/<method-dir>/method.proto` is called **method description file**; it must contain [method descriptor](#method-description-file) definition
* `services/<service-dir>` is called **service directory**
* `service/<service-dir>/service.proto` is called **service description file**; it must contain [service descriptor](#service-descriptor) definition

Busrpc [scopes](#type-visibility) and their hierarchy matches busrpc API directory layout:
* globally-scoped types should be defined in files placed to the API root directory
* namespace-scoped types should be defined in files placed to the namespace directory
* class-scoped types should be defined in files placed to the class directory
* method-scoped types should be defined in files placed to the method directory

Note, that type visibility rules can be expressed in terms of files and directories in the following way:
* file from the API root directory can be imported by any file
* file from the namespace directory can be imported by any file in the same namespace directory or nested class/method directory
* file from the class directory can be imported by any file in the same class directory or nested method directory
* file from the method directory can be imported by any file in the same method directory

Note, that visibility constraints are applied only inside API root directory. For example, service description file can (and, in fact, is required to) import necessary method description files despite the fact that service description file is not itself placed to the method directory.

# Documentation commands

# Specializations

Some aspects of the busrpc API design were intentionally left unspecified in this document to avoid dependency on a particular message bus/queue/broker implementation. This unspecified aspects are defined in a seperate bus-dependent documents called *specializations*.

Currently the following specializations exist (more to be added):
* NATS [specialization](./docs/specializations/nats-busrpc.md)
