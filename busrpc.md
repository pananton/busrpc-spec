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
  * [Exception](#exception)
  * [Endpoint](#endpoint)
  * [Type visibility](#type-visibility)
* [Protocol](#protocol)
  * [Directory layout](#directory-layout)
  * [Protobuf package names](#protobuf-package-names)
  * [Class description file](#class-description-file)
    * [`ObjectId`](#objectid)
  * [Method description file](#method-description-file)
    * [`Params` and `Retval`](#params-and-retval)
      * [Observable parameters](#observable-parameters)
    * [`Static`](#static)
  * [Service description file](#service-description-file)
    * [`Config`](#config)
  * [Default field values](#default-field-values)
  * [Network messages](#network-messages)
    * [`CallMessage`](#callmessage)
    * [`ResultMessage`](#resultmessage)
  * [Endpoint components encoding](#endpoint-components-encoding)
    * [Character encoding](#character-encoding)
    * [Structure encoding](#structure-encoding)
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

Methods may be bound to a concrete object or to a class as a whole. The latter are called **static methods**. Busrpc specification also has a notion of a **static class**, which is a class without objects. Because of this, static class does not define an object identifier and every method of it's interface must be static. Static classes are mainly used to group related system-wide "utility" methods.

**Method call** is represented as a network message containing method parameters and, optionally, identifier of the object for which method is called (not needed for static method calls) sent to the [call endpoint](#endpoint). Some method parameters can additionally be defined as **observable**, which means that their values not only sent as payload of the message but also added to the call endpoint providing implementors with an ability to cherry-pick a subset of calls having a concrete values of the observable parameters.

**Method result** is represented as a network message containing either method return value or an [exception](#exception) sent to the [result endpoint](#endpoint). Method may be defined as a **one-way** - in that case it does not have any result or associated result endpoint. One-way methods are mostly used as event sinks, i.e. invoked to signal some events in the system.

## Service

**Service** is an application implementing and/or invoking busrpc class methods. This specification does not impose any limits on methods used by the service. For example, it is totally fine for a service to implement only a subset of class methods, or to implement/invoke methods from distinct classes. 

## Structure

Busrpc **structure** is an alternative term for a protobuf `message` introduced for consistency with OOP terminology. Structures are busrpc wire types, i.e. every busrpc network message is represented by some structure.

**Predefined structures** are structures which have special meaning determined by this specification. In particular, **descriptors** are predefined structures which provide busrpc client libraries with type information about busrpc entity (service, class, or method). Descriptors are usually never sent over the network - in fact, they even do not have any fields, only nested type definitions.

**Encodable structure** is a structure, that can be encoded as specified in the [Structure encoding](#structure-encoding) section. To be encodable, structure must contain only non-`repeated`, non-`oneof` fields of the following types: 
* [scalar](https://developers.google.com/protocol-buffers/docs/proto3#scalar) types, except for floating-point types `float` and `double`
* [enumeration](https://developers.google.com/protocol-buffers/docs/proto3#enum) types

```
enum MyEnum {
  MYENUM_VALUE_0 = 0;
  MYENUM_VALUE_1 = 1;
}

// empty structure is encodable
message Encodable1 { }

// encodable, because all fields have valid types
message Encodable2 {
  optional int32 f1 = 1;
  string f2 = 2;
  bytes f3 = 3;
  MyEnum f4 = 4;
}

// not encodable, each field violates one of the encodable structure rules
message NotEncodable {
  // floating-point types are prohibited
  float f1 = 1;

  // repeated fields are prohibited
  repeated int32 f2 = 2;

  // oneof is prohibited
  oneof Variant {
    int32 f3 = 3;
    string v4 = 4;
  }

  // maps are prohibited
  map<int32, int32> f5 = 5;

  // fields of a user-defined type are prohibited
  Encodable1 f6 = 6;
}
```

## Enumeration

Busrpc **enumeration** corresponds directly to a protobuf `enum`.

## Namespace

**Namespace** is a group of related busrpc classes, structures and enumerations. Typically namespace contains part of the API that corresponds to a single application subdomain.

## Exception

Busrpc method result is either a method-specified return value or an **exception**, which is an instance of a global `Exception` structure. Exception indicates abnormal method completion.

Basic minimal `Exception` structure is defined in the [*busrpc.proto*](proto/busrpc.proto) file. It contains only error code information, represented as `Errc` enumeration value. Third-party APIs may add more fields to the `Exception` structure and/or define new values for the `Errc` enumeration, however, type names (`Exception` and `Errc`) must not be changed.

```
// Exception error code.
enum Errc {
  // Unexpected error.
  ERRC_UNEXPECTED = 0;
}

// Global exception type for busrpc methods.
message Exception {
  // Error code.
  Errc code = 1;
}
```

By **throwing** an exception busrpc specification means sending an instance of `Exception` structure as the method result. By **catching** exception we mean handling (for example, using some fallback mechanism) the situation when caller receives `Exception` instance instead of the method return value.

Whenever caller receives an exception which he does not know how to handle, he must forward exception to the upstream caller. Consider situation, when method `A` calls method `B`, method `B` calls method `C` and method `C` throws an exception. If method `B` does not know, how to handle occurred exception, then it should send it as it's own result to the method `A`. This mechanism is called **exception propagation** and can be found in many programming languages with OOP support.

## Endpoint

**Call endpoint** is a message bus topic to which method calls are published by a caller. Format of the call endpoint is `<namespace>.<class>.<method>.<object-id>[.<observable-params>].<eof>`, where:
* `<namespace>`, `<class>` and `<method>` are names of the namespace, class and method correspondingly
* `<object-id>` is identifier of an object for which method is called, or a reserved word representing null value for static methods
* `<observable-params>` is a sequence of words representing values of the observable parameters (can be 0 or many)
* `<eof>` is a reserved word which designates the end of endpoint

**Result endpoint** is a message bus topic assigned to the `replyTo` parameter when method call is published. Format of the result endpoint is `<result-endpoint-prefix>.<call-endpoint>`, where:
* `<result-endpoint-prefix>` is a sequence of words, which provide information, necessary to demultiplex responses and bind them to the original requests (for example, in NATS specialization `<result-endpoint-prefix>` is defined as `_INBOX.<connection-id>.<request-id>`, where pair of connection and request identifiers uniquelly identifies request in the system)
* `<call-endpoint>` is an exact copy of the call endpoint used for the request; it's components are used by the framework [test clients](README.md#clients) to effectively catch only necessary responses when observing the system message flow

Note, that result endpoint is not defined for one-way methods.

Some message bus topic formats, commonly used for subscribing for method calls, are also mapped to a named endpoints to establish common terminology. This mapping is provided in the following table.

| Topic                                                                                          | Endpoint  | Description                                       |
| ---------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------- |
| `<namespace>.<topic-wildcard-anyN>`                                                            | namespace | calls of any method of any class from a namespace |
| `<namespace>.<class>.<topic-wildcard-anyN>`                                                    | class     | calls of any method of a class                    |
| `<namespace>.<class>.<method>.<topic-wildcard-anyN>`                                           | method    | calls of a method                                 |
| `<namespace>.<class>.<topic-wildcard-any1>.<object-id>.<topic-wildcard-anyN>`                  | object    | calls bound to a specific object                  |
| `<namespace>.<class>.<method>.<topic-wildcard-any1>.<observable-params>.<topic-wildcard-anyN>` | value     | calls of a method with a specific value(s) of an                                                                                                                        observable parameter(s)                           |

Message bus may prohibit the use of some characters in it's topics. That means some components of the endpoint should be encoded to meet message bus requirement. Additionally, care should be taken to avoid violation of a message bus topic length restriction. All this issues are covered [below](#endpoint-components-encoding).

## Type visibility

**Scope** is a part of a busrcp API where name of a structure or enumeration can be used to refer to the corresponding type. In that case we also say that busrpc structure/enumeration is **visible** in this scope.

The scope of a type is determined by the place in the directory hierarchy where the protobuf file containing it's definition is located. Additionally, scopes form a hierarchy in which types visible in the parent scope are also visible in all child scopes. The following scopes are defined by the busrpc specification (in the order from parent to child):
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

---

**NOTE**

Most examples in this section are based on the busrpc API of a simple fictional IM application called Chat (see [*example/*](example) directory).

---

## Directory layout

All busrpc protobuf files should be organized in the tree represented below. Names in angle brackets are placeholders which are assigned real names by specific API implementation. For simplicity, only a single namespace, class, method and service is presented. Of course, real API may contain arbitrary number of this entities structured in a similar way.

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
 
Components of this tree are:
* root directory *\<busrpc-root-dir>/*, which contains two predefined directories: API root directory *api/* and services root directory *services/*
* file [*busrpc.proto*](proto/busrpc.proto), which is provided by the framework; it contains important busrpc-specific definitions and should be included to any busrpc-compliant API
* namespace directory *\<namespace-dir>/*, which contains definitions of all namespace classes
* class directory *\<class-dir>/*, which contains definitions of the class methods (collectively referred to as class interface) and a [class description file](#class-description-file) *class.proto*
* method directory *\<method-dir>/*, which contains definition of a class method and a [method description file](#method-description-file) *method.proto*
* service directory *\<service-dir>/*, which contains a [service description file](#service-description-file) *service.proto*

Busrpc [scopes](#type-visibility) and their hierarchy matches busrpc API directory layout:
* globally-scoped types are types defined in files in the API root directory
* namespace-scoped types are types defined in files in the namespace directory
* class-scoped types are types defined in files in the class directory
* method-scoped types are types defined in files in the method directory

Note, that type visibility rules can be expressed in terms of files and directories in the following way:
* file from the API root directory can be imported by any file
* file from the namespace directory can be imported by any file in the same namespace directory or nested class/method directory
* file from the class directory can be imported by any file in the same class directory or nested method directory
* file from the method directory can be imported by any file in the same method directory

Note, that visibility constraints are applied only inside API root directory. For example, service description file can (and, in fact, is required to) import necessary method description files despite the fact that service description file itself is not placed to the method directory.

## Protobuf package names

Busrpc directory layout determines the hierarchy of the protobuf [packages](https://developers.google.com/protocol-buffers/docs/proto3#packages):
* top-level package name is `busrpc`
* other package names follow directory hierarchy, for example, content of the *api/busrpc.proto* file (as well as any other file in the API root directory) should be placed into `busrpc.api` package

## Class description file

Class description file *class.proto* must always contain definition of a class descriptor `ClassDesc` - a special protobuf `message` (or predefined structure in terms of this specification), which provides information about the class by means of a nested types. Busrpc specification currently recognizes only `ObjectId` structure, which describes class object identifier. Definitions of other types may also be nested inside `ClassDesc`, however this may cause conflicts in the future versions of this specification and thus not recommended.

### `ObjectId`

`ObjectId` is a predefined [encodable structure](#structure), which contains arbitrary number of fields that together form a unique identifier of the class object.

The following class represents a user of a Chat application. Every user is indentified by a unique username.

```
// file ./api/chat/user/class.proto
// ...

message ClassDesc {
  message ObjectId {
    string username = 1;
  }
}
```

If `ClassDesc` does not contain a nested `ObjectId` type, then corresponding class is considered static. Static classes are commonly used to group "utility" methods. For example, static class `translator` provides methods for translating UI controls to a user language.

```
// file ./api/chat/translator/class.proto
// ...

message ClassDesc { }
```

Note, that empty `ObjectId` is not equivalent to a missing `ObjectId` - the first one does not make a class static. In fact, class with empty `ObjectId` may be treated as singleton - a class, for which only one object exists.

## Method description file

Method description file *method.proto* must always contain definition of a method descriptor `MethodDesc` - a predefined busrpc structure, which provides information about the method by means of a nested types. All this nested types are described in the subsections below. Definitions of other types may also be nested inside `MethodDesc`, however this may cause conflicts in the future versions of this specification and thus not recommended.

### `Params` and `Retval`

`Params` and `Retval` are predefined structures, which describe method parameters and return value. Both of this structures may be omitted in `MethodDesc`.

Missing or empty `Params` structure means that method does not have any parameters.

`MethodDesc` without nested `Retval` describes a [one-way method](#class), which does not involve any reply when it gets called. This means the caller can't determine when and whether one-way method call is processed. `MethodDesc` with empty `Retval` describes a regular method, for which reply, albeit empty, is sent when the call is processed.

Consider two methods from a `user` class described in the previous section:
* method `sign_in` verifies user credentials (`username` obtained from the object identifier and `password` passed in method parameters) and signs in user to the application if verification is passed; in pseudocode method signature can be seen as `Result user::sign_in(string password)`
* one-way method `on_signed_in` is invoked for signed in user to allow other services implement arbitrary actions for signed in user (for example, notify user contacts that he is online); in pseudocode method signature can be seen as `void user::on_signed_in()`

```
// file ./api/chat/user/sign_in/method.proto
// ...

enum Result {
  RESULT_SUCCESS = 0;
  RESULT_UNKNOWN_USERNAME = 1;
  RESULT_INVALID_PASSWORD = 2;
}

message MethodDesc {
  message Params {
    string password = 1;
  }

  message Retval {
    Result result = 1;
  }
}
```

```
// file ./api/chat/user/on_signed_in/method.proto
// ...

message MethodDesc { }
```

Note, that `Result` enumeration has method scope and can't be used outside the method `user::sign_in` directory.

#### Observable parameters

Observable method parameter is created using custom protobuf field option `observable`, defined in the *busrpc.proto* file. This option can be applied only if parameter meets the following requirement: it's type is either one of the types allowed for [encodable structure](#structure), or an encodable structure itself. Remember, that observable parameters (as described [earlier](#endpoint)) are also added to the call endpoint, which means that implementors may filter calls they want to handle to only those having specific value(s) of the observable parameter(s).

Consider method `user::send_message` from the Chat application API.

```
// file ./api/chat/user/send_message/method.proto
// ...

message MethodDesc {
  message Params {
    string receiver = 1 [(observable) = true];
    string text = 2;
  }

  message Retval { }
}
```

This method may be used in the following way:
* user "A" signs in to application and subscribes to a method's `user::send_message` value endpoint with the `receiver` parameter set to "A" (i.e., to `chat.user.send_message.<topic-wildcard-any1>.A.<topic-wildcard-anyN>`) 
* user "B" calls `user::send_message` with `receiver` set to "A"
* endpoint `chat.user.send_message.B.A.<eof>` of this call matches user "A" value endpoint, so user "A" receives the message and replies to it
* user "B" receives a reply indicating that message is delivered

### `Static`

`Static` is an empty predefined structure, which designates method as static. The following method is part of an Chat application API and is used to create new users. The method is defined as static to emphasize the fact that user (and corresponding object) does not exist at the moment the method is called.

```
// ./api/chat/user/sign_up/method.proto
// ...

enum Result {
  RESULT_SUCCESS = 0;
  RESULT_USERNAME_ALREADY_TAKEN = 1;
  RESULT_PASSWORD_TOO_WEAK = 2;
}

message MethodDesc {
  message Params {
    string username = 1;
    string password = 2;
  }

  message Retval {
    Result result = 1;
  }

  message Static { }
}
```

Note, that all methods of a static class must be defined as static.

## Service description file

Service description file *service.proto* must always contain definition of a service descriptor `ServiceDesc` - a predefined busrpc structure, which provides information about the service by means of a nested types. Busrpc specification currently recognizes only `Config` structure, which describes service configuration parameters. Definitions of other types may also be nested inside `ServiceDesc`, however this may cause conflicts in the future versions of this specification and thus not recommended.

Additionally, service description file must import description files of all methods that service implements or invokes.

Consider a service that sends welcome message to any user who signed in to the Chat application for the first time. Such service is pretty easy to implement using existing API: it needs to implement method `user::on_signed_in` to check whether user signs in for the first time, and if he is, call `user::send_message`. The fact that service uses two API methods is expressed in the service description file by importing corresponding *method.proto* files.

```
// file ./services/greeter/service.proto
// ...

import "api/chat/user/on_signed_in/method.proto";
import "api/chat/user/send_message/method.proto";

// ...
```

### `Config`

`Config` is a predefined strucuture describing service configration parameters. Note, that protobuf supports JSON serialization for it's `message` types, which means that service configuration can be easily read/written from/to the text file.

```
// file ./services/greeter/service.proto
// ...

message ServiceDesc {
  message Config {
    busrpc.services.ConfigBase general = 1;
    string welcome_text = 2;
  }
}
```

## Default field values

File *busrpc.proto* contains definitions of several options that allow to specify default values for structure fields (other than [those](https://developers.google.com/protocol-buffers/docs/proto3#default) defined by protobuf itself). Of course, protobuf compiler does not understand semantics of this options, however, [client libraries](README.md#libraries) are expected to respect them. This options are:
* `default_bool` - default value for protobuf `bool` type
* `default_int` - default value for protobuf integer types (`int32`, `uint32`, `int64`, `uint64`, `sint32`, `sint64`, `fixed32`, `fixed64`, `sfixed32`, `sfixed64`)
* `default_double` - default value for protobuf floating-point types (`float`, `double`)
* `default_string` - default value for protobuf `string` type

This options are especially useful for describing method parameters (`MethodDesc::Params` fields) and service configuration (`ServiceDesc::Config` fields).

```
// file ./services/greeter/service.proto
// ...

message ServiceDesc {
  message Config {
    services.ConfigBase general = 1;
    string welcome_text = 2 [(default_string) = "Thank you for trying Chat!"];
  }
}
```

## Network messages

Busrpc specification defines two protobuf `message` types which are directly used as a network format:
1. `CallMessage` for transferring method call data (object identifier and parameters)
2. `ResultMessage` for transferring method result (return value or exception)

Both types can be found in the *busrpc.proto* file and must not be modified in any way by busrpc-compliant APIs.

### `CallMessage`

`CallMessage` is defined as follows:

```
message CallMessage {
  optional bytes object_id = 1;
  optional bytes params = 2;
}
```

Field `object_id` contains protobuf-serialized `ObjectId` structure from the class descriptor and determines the object, for which method is called. Sender should not set this field when calling a static method, however, receiver should accept `CallMessage` with initialized `object_id` even if it is received for a static method and simply discard `object_id` (i.e., busrpc specification follows "be conservative in what you do, be liberal in what you accept from others" principle).

Field `params` contains protobuf-serialized `Params` structure from the method descriptor. Sender should not set this field when calling a method, for which `Params` is not defined. Receiver should accept `CallMessage` with initialized `params` even if it is sent for a method without `Params` and simply discard `params`.

### `ResultMessage`

`ResultMessage` is defined as follows:

```
message ResultMessage {
  oneof Result {
    bytes retval = 1;
    busrpc.Exception exception = 2;
  }
}
```

Field `retval` contains protobuf-serialized `Retval` structure from the method descriptor and is set only if method did not throw an exception.

Field `exception` contains global predefined [`Exception`](#exception) structure and is set if method threw an exception.

## Endpoint components encoding

Remember from the [Endpoint](#endpoint) section that endpoint is a character string, which is formatted like `[<result-endpoint-prefix>.]<namespace>.<class>.<method>.<object-id>[.<observable-params>].<eof>`. In this section we describe how to convert protocol data to the endpoint components. Note, that format of the `<result-endpoint-prefix>` part (meaningful for result endpoints only) is defined by a message bus and is out of scope of this specification.

Busrpc endpoint is inherently a message bus topic. Message bus may impose some limitations on it's topics, which must be respected by the endpoints:
1. some characters may be prohibited or treated specially
2. word or topic length may be limited

The first problem is solved using the busrpc [character encoding](#character-encoding).

The second problem requires developers to carefully design their APIs and avoid situations, when some endpoints may occasionally exceed the message bus limit. Busrpc specification helps to deal with this problem in the following way: it describes how fixed-size hash of the component's data can be used in the endpoint instead of the data itself.

As a hash function busrpc framework uses SHA-224, which is chosen for the following reasons:
* it is a cryptographic hash function, which means that it is practically impossible to find two distinct inputs that are hashed to the same value
* it's performance does not degrade on small inputs (tens of bytes)
* modern CPUs provide high-performance instructions for SHA hash calculation
* it's output is shorter than SHA-256

### Character encoding
 
Busrpc specification requires, that the fol, that the following characters can be used as-is in a message bus topic:
* alphanumericals (a-z, A-Z and 0-9)
* underscore `_` (to avoid encoding of the namespace/class/method names)
* hyphen `-` (to avoid encoding of the negative numbers)

Additionally, busrpc specification itself introduces several characters with a special meaning. This characters are chosen separetely for each message bus. In this document they are reffered to by the following tokens:
* `<esc>` - escape character
* `<field-sep>` - separator for structure fields when structure is encoded as endpoint component (see [below](#structure-encoding))

If endpoint component contains reserved characters, all of them should be encoded as `<esc><hex><hex>`. Here `<esc>` is an escape character and `<hex><hex>` is a hexadecimal representation of the reserved character. Busrpc specification recommends to use lowercase "a-f" for the hexadecimal digits, however, it is not required.

### Structure encoding

# Documentation commands

# Specializations

Some aspects of the busrpc API design were intentionally left unspecified in this document to avoid dependency on a particular message bus/queue/broker implementation. This unspecified aspects are defined in a seperate bus-dependent documents called *specializations*.

Currently the following specializations exist (more to be added):
* NATS [specialization](./docs/specializations/nats-busrpc.md)
