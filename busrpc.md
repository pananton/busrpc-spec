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
  * [Exceptions](#exceptions)
  * [Endpoint](#endpoint)
  * [Type visibility](#type-visibility)
* [Protocol](#protocol)
  * [Directory layout](#directory-layout)
  * [Protobuf package names](#protobuf-package-names)
  * [Namespace description file](#namespace-description-file)
  * [Class description file](#class-description-file)
    * [`ObjectId`](#objectid)
  * [Method description file](#method-description-file)
    * [`Params` and `Retval`](#params-and-retval)
      * [Observable parameters](#observable-parameters)
    * [`Static`](#static)
  * [Service description file](#service-description-file)
    * [`Config`](#config)
    * [`Implements`](#implements)
    * [`Invokes`](#invokes)
  * [Built-in types](#built-in-types)
    * [`Errc`](#errc) 
    * [`Exception`](#exception)
    * [`CallMessage`](#callmessage)
    * [`ResultMessage`](#resultmessage)
  * [Default field values](#default-field-values)
  * [Endpoint encoding](#endpoint-encoding)
    * [Type encoding](#type-encoding)
      * [Boolean encoding](#boolean-encoding)
      * [Integer encoding](#integer-encoding)
      * [String encoding](#string-encoding)
      * [Byte sequence encoding](#byte-sequence-encoding)
      * [Structure encoding](#structure-encoding)
    * [Algorithm](#algorithm)
    * [Examples](#examples)
* [Documenting API](#documenting-api)
  * [Basic rules](#basic-rules)
  * [Documentation commands](#documentation-commands)
    * [Reference](#documentation-command-reference) 
* [Specializations](#specializations)

# Introduction

As appears from it's name, busrpc framework stands on two pillars.

First of all, it is a form of a remote procedure call (RPC) technology (like [grpc](https://grpc.io/), [json-rpc](https://www.jsonrpc.org/) and similar) used for inter-service communication.

Secondly, it relies on a message bus/queue/broker component as a transport layer. Usual RPC implementations operate in a peer-to-peer manner which means that communicating parties need to connect directly to each other. This is probably ok for the systems with small number (1-5) of services, but does not suite well for microservice backends where number of services can easily surpass one hundred. Instead, microservice backends utilize a dedicated component (called message bus/queue/broker) as a central point of communication which is responsible for inter-service message delivery. This greatly simplifies system configuration (only message bus address is required to access any part of the system API), service deployment (because all services are loosely-coupled in this scheme) and administration.

Busrpc framework global goal is to offer a turnkey solution for backend developers who apply microservice architecture for their projects. To achieve this, busrpc framework is designed with the following ideas in mind:
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

Busrpc API design is based on the concepts from object-oriented programming. This allows busrpc to re-use well-known OOP terminology and stay familiar for newcomers (note however that same terms from busrpc API design and object-oriented design may differ in some aspects and should not be treated as exactly equivalent). Moreover, we believe that many well-established and time-tested object-oriented design principles and decomposition strategies can also be applied for a good microservice backend API, which means that developers' OOP experience might come in handy in the context of the busrpc framework.

Busrpc uses Google's [protocol buffers](https://developers.google.com/protocol-buffers) as a format of network messages, but the details will be left for a separate [Protocol](#protocol) section. Here we focus on the concepts and terminology of the busrpc framework.

## Class

Busrpc **class** is a model of a similarly arranged entities from the API business domain. By similar arrangement we mean that all entities modelled by a class have the same format of internal state and expose the same set of supported operations, which are called **methods** as it is accepted in OOP. All together, class methods form an **interface** of a class.

**Object** of a class represents a concrete entity from the set of modelled entities. It is characterized by concrete value of internal state and an immutable **object identifier**, which uniquelly identifies object throughout the system.

Methods may be bound to a concrete object or to a class as a whole. The latter are called **static methods**. Busrpc specification also has a notion of a **static class**, which is a class without objects. Because of this, static class does not define an object identifier and every method of it's interface must be static. Static classes are mainly used to group related system-wide "utility" methods.

**Method call** is represented as a network message containing method parameters and, optionally, identifier of the object for which method is called (not needed for static method calls) sent to the [call endpoint](#endpoint). Some method parameters can additionally be defined as **observable**, which means that their values not only sent as payload of the message but also added to the call endpoint providing implementors with an ability to cherry-pick a subset of calls having a concrete values of the observable parameters.

**Method result** is represented as a network message containing either method return value or an [exception](#exceptions) sent to the [result endpoint](#endpoint). Method may be defined as a **one-way** - in that case it does not have any result or associated result endpoint. One-way methods are mostly used as event sinks.

## Service

**Service** is an application implementing and/or invoking busrpc class methods. This specification does not impose any limits on methods used by the service. For example, it is totally fine for a service to implement only a subset of class methods, or to implement/invoke methods from distinct classes. 

## Structure

Busrpc **structure** is an alternative term for a protobuf `message` introduced for consistency with OOP terminology.

**Predefined structures** are structures which have special meaning determined by this specification. In particular, **descriptors** are predefined structures which provide busrpc client libraries with type information about busrpc entity (namespace, class, method or service). Descriptors are usually never sent on the network - in fact, they even do not have any fields, only nested type definitions.
 
Busrpc specification defines a concept of an **encodable type** - a protobuf type which can be encoded as specified in the [Type encoding](#type-encoding) section below. Encodable type can't be `repeated` and should be one the following:
* [scalar](https://developers.google.com/protocol-buffers/docs/proto3#scalar) type except for floating-point types `float` and `double`
* [enumeration](https://developers.google.com/protocol-buffers/docs/proto3#enum) type
* structure type, which contains only fields of encodable scalar/enumeration types (structure without fields is also encodable)

Some examples of encodable and not encodable structure types:

```
enum MyEnum {
  MYENUM_VALUE_0 = 0;
  MYENUM_VALUE_1 = 1;
}

// empty structure is encodable
message Encodable1 { }

// all fields have encodable scalar/enumeration type, so the structure is encodable
message Encodable2 {
  optional int32 f1 = 1;
  string f2 = 2;
  bytes f3 = 3;
  MyEnum f4 = 4;
}

// not encodable (every field is not encodable scalar/enumeration type)
message NotEncodable {
  float f1 = 1;
  repeated int32 f2 = 2;

  oneof Variant {
    int32 f3 = 3;
    string v4 = 4;
  }

  map<int32, int32> f5 = 5;
  Encodable1 f6 = 6;
}
```

## Enumeration

Busrpc **enumeration** corresponds directly to a protobuf `enum`. **Predefined enumerations** are enumerations which have special meaning determined by this specification.

## Namespace

**Namespace** is a group of related busrpc classes, structures and enumerations. Typically namespace contains part of the API that corresponds to a single application subdomain.

## Exceptions

Busrpc method result is either a method-specified return value or an **exception**, which indicates abnormal method completion.

Busrpc exceptions are instances of a predefined structure `Exception`, which at least contains an error code expressed as a value of predefined enumeration `Errc`. Third-party APIs may extend both `Exception` and `Errc` types, however, should not modify their names.

By **throwing** an exception busrpc specification means sending an instance of `Exception` structure as the method result. By **catching** exception we mean handling (for example, using some fallback mechanism) the situation when caller receives `Exception` instance instead of the method return value.

Whenever caller receives an exception which he does not know how to handle, he must forward exception to the upstream caller. Consider situation, when method `A` calls method `B`, method `B` calls method `C` and method `C` throws an exception. If method `B` does not know, how to handle occurred exception, then it should send it as it's own result to the method `A`. This mechanism is called **exception propagation** and can be found in many programming languages with OOP support.

Busrpc method exceptions may be converted to a programming language exceptions by the client libraries. This may cause performance degrade and log/alert flooding if exceptions are used incorrectly. The main purpose of an exception is to report **exceptional** situations, which usually occur rarely and have no special handling (except for trivial logging) on the caller side. For example, network connection failure may be considered an exceptional situation by some APIs, while invalid user input almost never should be treated as exceptional situation.

## Endpoint

**Call endpoint** is a message bus topic to which method calls are published by a caller. Format of the call endpoint is `<namespace>.<class>.<method>.<object-id>[.<observable-params>].<eof>`, where:
* `<namespace>`, `<class>` and `<method>` are names of the namespace, class and method correspondingly
* `<object-id>` is identifier of an object for which method is called, or a reserved word `<null>` for static methods
* `<observable-params>` is a sequence of words representing values of the observable parameters (can be 0 or many)
* `<eof>` is a reserved word which designates the end of endpoint

**Result endpoint** is a message bus topic assigned to the `replyTo` parameter when method call is published. Format of the result endpoint is `<result-endpoint-prefix>.<call-endpoint>`, where:
* `<result-endpoint-prefix>` is a sequence of words, which provide information, necessary to demultiplex responses and bind them to the original requests (for example, in NATS specialization `<result-endpoint-prefix>` is defined as `_INBOX.<connection-id>.<request-id>`, where pair of connection and request identifiers uniquelly identifies request in the system)
* `<call-endpoint>` is an exact copy of the call endpoint used for the request; it's components are used by the framework [test clients](README.md#clients) to effectively catch only necessary responses when observing the system message flow

Note, that result endpoint is not defined for one-way methods.

Some message bus topic formats, commonly used for subscribing, are also mapped to a named endpoints to establish common terminology. This mapping is provided in the following table.

| Topic                                                                                          | Endpoint  | Description                                       |
| ---------------------------------------------------------------------------------------------- | --------- | ------------------------------------------------- |
| `<namespace>.<topic-wildcard-anyN>`                                                            | namespace | calls of any method of any class from a namespace |
| `<namespace>.<class>.<topic-wildcard-anyN>`                                                    | class     | calls of any method of a class                    |
| `<namespace>.<class>.<method>.<topic-wildcard-anyN>`                                           | method    | calls of a method                                 |
| `<namespace>.<class>.<topic-wildcard-any1>.<object-id>.<topic-wildcard-anyN>`                  | object    | calls bound to a specific object                  |
| `<namespace>.<class>.<method>.<topic-wildcard-any1>.<observable-params>.<topic-wildcard-anyN>` | value     | calls of a method with a specific value(s) of an                                                                                                                        observable parameter(s)                           |

Message bus may prohibit the use of some characters in it's topics. That means endpoint should be encoded to meet message bus requirements. Also care should be taken to avoid violation of a message bus topic length restriction. Finally, the encoding algorithm itself must be unambigous to allow various busrpc clients to communicate. All this issues are covered [below](#endpoint-encoding).

## Type visibility

**Scope** is a part of a busrpc API where name of a structure or enumeration can be used to refer to the corresponding type. In that case we also say that busrpc structure/enumeration is **visible** in this scope.

The scope of a type is determined by the place in the directory hierarchy where the protobuf file containing it's definition is located. Additionally, scopes form a hierarchy in which types visible in the parent scope are also visible in all child scopes. The following scopes are defined by the busrpc specification:
1. a single **global scope**
2. a single **API scope** for all API types
3. a **namespace scope** for each namespace
4. a **class scope** for each class
5. a **method scope** for each method
6. a single **implementation scope** for types used internally by the services
7. a **service scope** for each service

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
<project-dir>/
├── busrpc.proto
├── api/
│   ├── <namespace-dir>/
|       ├── namespace.proto 
│       ├── <class-dir>/
│           ├── class.proto
│           ├── <method-dir>/
│               ├── method.proto
├── implementation/
│   ├── <service-dir>/
|       ├── service.proto
```
 
Components of this tree are:
* project directory *\<project-dir>/*, which contains framework-provided *busrpc.proto* file with definitions of the [built-in](#built-in-types) busrpc types 
* API root directory *api/*
* namespace directory *\<namespace-dir>/*, which contains a separate subdirectory for each namespace class and a [namespace description file](#namespace-description-file) *namespace.proto*
* class directory *\<class-dir>/*, which contains a separate subdirectory for each class method and a [class description file](#class-description-file) *class.proto*
* method directory *\<method-dir>/*, which contains definition of a class method in the form of [method description file](#method-description-file) *method.proto*
* implementation root directory *implementation/*
* service directory *\<service-dir>/*, which contains definition of a service in the form of [service description file](#service-description-file) *service.proto*

Additionally, every directory in a tree can contain arbitrary number of protobuf files with definitions of general busrpc structures and enumerations. The place in the busrpc directory tree where protobuf file is located determines the [scope](#type-visibility) of all types defined in this file: types defined in the parent directory are visible in it's child directories, while types from the child directory are not visible in it's parent.

## Protobuf package names

Busrpc directory layout determines the hierarchy of the protobuf [packages](https://developers.google.com/protocol-buffers/docs/proto3#packages):
* top-level package name should be `busrpc`
* other package names follow directory hierarchy, for example, content of the *api/file.proto* file (as well as any other file in the API root directory) should be placed into `busrpc.api` package

## Namespace description file

Namespace description file *namespace.proto* must always contain definition of the namespace descriptor `NamespaceDesc` - a special protobuf `message` (or predefined structure in terms of this specification), which provides information about the namespace by means of a predefined nested types. Current version of the specification does not define any nested type with a special meaning, still empty descriptor still must be provided:

```
// file api/chat/namespace.proto

message NamespaceDesc { }
```

Note, that it is not recommended to define any types inside `NamespaceDesc` because this may cause conflicts in the future versions of the busrpc specification.

## Class description file

Class description file *class.proto* must always contain definition of the class descriptor `ClassDesc` - a special protobuf `message` (or predefined structure in terms of this specification), which provides information about the class by means of a predefined nested types. Busrpc specification currently recognizes only `ObjectId` structure, which describes class object identifier. Definitions of other types may also be nested inside `ClassDesc`, however this may cause conflicts in the future versions of this specification and thus not recommended.

### `ObjectId`

`ObjectId` is a predefined [encodable structure](#structure), which contains arbitrary number of fields that together form a unique identifier of the class object.

The following class represents a user of a Chat application. Every user is indentified by a unique username.

```
// file api/chat/user/class.proto

message ClassDesc {
  message ObjectId {
    string username = 1;
  }
}
```

If `ClassDesc` does not contain a nested `ObjectId` type, then corresponding class is considered static. Static classes are commonly used to group "utility" methods. For example, static class `translator` provides methods for translating UI controls to a user language.

```
// file api/chat/translator/class.proto

message ClassDesc { }
```

Note, that empty `ObjectId` is not equivalent to a missing `ObjectId` - the first one does not define a static class. In fact, class with empty `ObjectId` may be treated as singleton - a class, for which only one object exists.

## Method description file

Method description file *method.proto* must always contain definition of the method descriptor `MethodDesc` - a predefined busrpc structure, which provides information about the method by means of a predefined nested types. All predefined nested types are described in the subsections below. Definitions of other types may also be nested inside `MethodDesc`, however this may cause conflicts in the future versions of this specification and thus not recommended.

### `Params` and `Retval`

`Params` and `Retval` are predefined structures, which describe method parameters and return value. Both of this structures may be omitted in `MethodDesc`.

`MethodDesc` without nested `Params` describes a method without parameters. Note, that if structure `Params` exists but has no fields, corresponding method **is not** considered a method without parameters by this specification.

`MethodDesc` without nested `Retval` describes a [one-way method](#class), which does not involve any reply when it gets called. This means the caller can't determine when and whether one-way method call is processed. `MethodDesc` with empty `Retval` describes a regular method, for which reply, albeit empty, is sent when the call is processed.

Consider two methods from a `user` class described in the previous section:
* method `sign_in` verifies user credentials (`username` obtained from the object identifier and `password` passed in method parameters) and signs in user to the application if verification is passed; in pseudocode method signature can be seen as `Result user::sign_in(string password)`
* one-way method without parameters `on_signed_in` is invoked for signed in user to allow other services implement arbitrary actions for signed in user (for example, notify user contacts that he is online); in pseudocode method signature can be seen as `void user::on_signed_in()`

```
// file api/chat/user/sign_in/method.proto

enum Result {
  RESULT_SUCCESS = 0;
  RESULT_INVALID_PASSWORD = 1;
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
// file api/chat/user/on_signed_in/method.proto

message MethodDesc { }
```

Note, that `Result` enumeration has method scope and can't be used outside the method `user::sign_in` directory.

#### Observable parameters

[Observable](#class) method parameter is created using custom protobuf field option `observable`, defined by this specification. This option can be applied only to those `Params` fields, which have [encodable type](#structure). Remember, that observable parameters (as described [earlier](#endpoint)) are also added to the call endpoint and provide implementors with an ability to cherry-pick calls with a desired value(s) of the observable parameter(s).

Consider method `user::send_message` from the Chat application API.

```
// file api/chat/user/send_message/method.proto

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
// file api/chat/user/sign_up/method.proto

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

Service description file *service.proto* must always contain definition of the service descriptor `ServiceDesc` - a predefined busrpc structure, which provides information about the service by means of a predefined nested types. All predefined nested types are described in the subsections below. Definitions of other types may also be nested inside `ServiceDesc`, however this may cause conflicts in the future versions of this specification and thus not recommended.

### `Config`

`Config` is a predefined structure describing service configuration settings. Note, that protobuf supports JSON serialization for it's `message` types, which means that service configuration can be easily read/written from/to the text file.

```
message ServiceDesc {
  message Config {
    string bus_ip = 1;
    uint32 bus_port = 2;
  }
}
```

### `Implements`

`Implements` is a predefined structure which is used to provide information about methods implemented by the service. For each implemented method, `Implements` should contain a field with arbitrary name and type set to method descriptor `MethodDesc` of the method.

---

**NOTE**

Due to `Implements` nature, busrpc specification **allows** it to reference types which are formally outside of it's scope.

---

Consider a service that sends welcome message to any user who signed in to the Chat application for the first time. Such service needs to know when user signs in to check whether welcome message should be sent to him, so it implements method `user::on_signed_in`. This fact is expressed in the following way:

```
// file implementation/greeter/service.proto

message ServiceDesc {
  ...
  message Implements {
    busrpc.api.chat.user.on_signed_in.MethodDesc method1 = 1;
  }
}
```

### `Invokes`

`Invokes` is a predefined structure which is used to provide information about methods invoked by the service. For each invoked method, `Invokes` should contain a field with arbitrary name and type set to method descriptor `MethodDesc` of the method.

---

**NOTE**

Due to `Invokes` nature, busrpc specification **allows** it to reference types which are formally outside of it's scope.

---

Consider a service that sends welcome message to any user who signed in to the Chat application for the first time. When service decides, that welcome message should be sent for the signed in user (see previous section), it invokes `user::send_message` on behalf of some system account to deliver the message. This fact is expressed in the following way:

```
// file implementation/greeter/service.proto

message ServiceDesc {
  ...
  message Invokes {
    busrpc.api.chat.user.send_message.MethodDesc method1 = 1;
  }
}
```

## Built-in types

File *busrpc.proto* must provide definitions of the busrpc built-in types, which must have exactly the same name and format in all compliant third-party implementations. Apart from the built-in types, *busrpc.proto* also contains definitions of a [custom](https://developers.google.com/protocol-buffers/docs/proto3#customoptions) protobuf options introduced by the busrpc framework. Ready-to-use *busrpc.proto* file can be found [here](proto/busrpc.proto).

### `Errc`

`Errc` is a predefined enumeration, which contains system-wide error codes describing the reason of a busrpc [exception](#exceptions). Third-party implementations are allowed to add custom error codes to the `Errc` enumeration, however, they should not modify the name of the type. For example, `Errc` defined for some API may look like this:

```
// file busrpc.proto

enum Errc {
  // Unexpected error.
  ERRC_UNEXPECTED = 0;

  // Method failed because it called another method and no service accepted it.
  ERRC_NOT_AVAILABLE = 1;

  // Method failed because it called another method and the call timed out.
  ERRC_TIMED_OUT = 2;

  // Database query failed.
  ERRC_DB_QUERY_FAILED = 5;
}
```

### `Exception`

`Exception` is a predefined structure representing busrpc [exception](#exceptions). Third-party implementations are allowed to add custom fields to the `Exception` structure, however, they should not modify the name of the type. For example, `Exception` for some API may look like this:

```
// file busrpc.proto

message Exception {
  // Error code.
  Errc code = 1;

  // Error description.
  optional string description = 2;

  // Name of the service which encountered an exceptional situation.
  optional string service_name = 3;

  // Namespace name.
  optional string namespace_name = 4;

  // Class name.
  optional string class_name = 5;

  // Method name.
  optional string method_name = 6;
}
```

### `CallMessage`

`CallMessage` is a predefined structure, which determines format of the network packet used to transfer method call data. This structure should not be modified in any way by a third-party implementation, or some busrpc tools may stop working.

```
// file busrpc.proto

message CallMessage {
  optional bytes object_id = 1;
  optional bytes params = 2;
}
```

Field `object_id` contains protobuf-serialized `ObjectId` structure from the class descriptor and determines the object, for which method is called. Sender should not set this field when calling a static method, however, receiver should accept `CallMessage` with initialized `object_id` even if it is received for a static method and simply discard `object_id` (i.e., busrpc specification follows "be conservative in what you do, be liberal in what you accept from others" principle).

Field `params` contains protobuf-serialized `Params` structure from the method descriptor. Sender should not set this field when calling a method, for which `Params` is not defined. Receiver should accept `CallMessage` with initialized `params` even if it is sent for a method without `Params` and simply discard `params`.

### `ResultMessage`

`ResultMessage` is a predefined structure, which determines format of the network packet used to transfer method result. This structure should not be modified in any way by a third-party implementation, or some busrpc tools may stop working.

```
// file busrpc.proto

message ResultMessage {
  oneof Result {
    bytes retval = 1;
    busrpc.Exception exception = 2;
  }
}
```

Field `retval` contains protobuf-serialized `Retval` structure from the method descriptor and is set only if method did not throw an exception. Otherwise, thrown exception is transferred in the `exception` field.

## Default field values

File [*busrpc.proto*](proto/busrpc.proto) contains definition of a `default_value` option of a `string` type, which allows to specify default value for a scalar structure field. Of course, protobuf compiler does not understand semantics of this option, however, [client libraries](README.md#libraries) are expected to respect it. Note, that client libraries also need to perform conversion from the string value to a field type value.

This option is especially useful for describing method parameters and service configuration settings.

```
// file implementation/greeter/service.proto

message ServiceDesc {
  message Config {
    busrpc.implementation.BusConfig bus = 1;
    string welcome_text = 2 [(default_value) = "Thank you for trying Chat!"];
  }
}
```

## Endpoint encoding

Remember from the [Endpoint](#endpoint) section that endpoint is a character string, which is formatted like `[<result-endpoint-prefix>.]<namespace>.<class>.<method>.<object-id>[.<observable-params>].<eof>`. In this section we describe how to convert protocol data to the endpoint components. Note, that format of the `<result-endpoint-prefix>` part (meaningful for result endpoints only) is defined by a message bus and is out of scope of this specification.

Busrpc endpoint is inherently a message bus topic. Message bus may impose some limitations on it's topics, which must be respected by the endpoints:
1. some characters may be prohibited or treated specially
2. word or topic length may be limited

The first problem is solved by careful encoding of the prohibited/special characters. Busrpc specification assumes, that alphanumericals (`a-z`, `A-Z`, `0-9`), underscore `_` and hyphen `-` never require encoding. Other characters may potentially require encoding depending on the underlying message bus. Information about prohibited/special characters can be found from the message bus [specialization](#specializations).

The second problem requires developers to carefully design their APIs and avoid situations, when some endpoints may occasionally exceed the message bus limit. Busrpc specification helps to deal with this problem in the following way: it describes how fixed-size hash of the endpoint component's data can be used in the endpoint instead of the data itself.

As a hash function busrpc framework uses SHA-224, which is chosen for the following reasons:
* it is a cryptographic hash function, which means that it is practically impossible to find two distinct inputs that are hashed to the same value
* it's performance does not degrade on small inputs (tens of bytes)
* modern CPUs provide high-performance instructions for SHA hash calculation
* it's output is shorter than SHA-256, which is important because we try to make endpoint as short as possible

### Type encoding

To describe endpoint encoding algorithm we first define a function `string EncodeValue(value, flags)`, which unambiguously converts a value of [encodable](#structure) protobuf type to a string. Here, `value` is a value to be encoded and `flags` are flags, controlling the encoding process.

Specification currently defines only one flag APPLY_HASH, which tells algorithm to additionally apply SHA-224 hash to the `value`. This flag is frequently used together with `string`, `bytes` and structure types, because their values may have arbitrary length. However, it can also be applied to other types, though it usually makes no sense, because the result will occupy more space in the endpoint than the value, encoded without specifying this flag.

Next sections describe how `EncodeValue(value, flags)` is executed for all possible combinations of `value` and `flags`.

#### Boolean encoding

1. Set `result` to "1" if `value` is `true` or "0" otherwise.
2. If APPLY_HASH flag is not set, return `result`.
3. Otherwise, calculate SHA-224 hash of `result` and return `EncodeValue(hash, 0)`.

#### Integer encoding

1. Convert `value` to string and set it as `result`. Use single `-` sign for negative values. Do not use leading zeros, unless `value` is zero, in which case it is converted to "0".
2. If APPLY_HASH flag is not set, return `result`.
3. Otherwise, calculate SHA-224 hash of `result` and return `EncodeValue(hash, 0)`.

#### Enumeration encoding

1. Return `EncodeValue(ToInteger(value), flags)`.

#### String encoding

1. If `value` is empty, return `<empty>` reserved word.
2. If APPLY_HASH flag is not set, replace all prohibited/reserved characters (as defined by the message bus [specialization](#specializations)) in `value` with a triplets `<esc><hex><hex>` and return it as result. Here, `<esc>` is a bus-specific escape character and `<hex><hex>` is a hexadecimal representation of the prohibited/reserved character. Use **only lowercase** `a-f` digits in `<hex>`.
3. Otherwise, calculate SHA-224 hash of `value` and return `EncodeValue(hash, 0)`.

#### Byte sequence encoding

1. If `value` is empty, return `<empty>` reserved word.
2. If APPLY_HASH flag is not set, convert every byte of a sequence to it's hexadecimal representation `<hex><hex>` and return the result of conversion. Use **only lowercase** `a-f` digits in `<hex>`.
3. Otherwise, calculate SHA-224 hash of `value` and return `EncodeValue(hash, 0)`.

#### Structure encoding

1. If structure does not have fields, return `<empty>` reserved word.
2. Create an empty `tmp` byte sequence to hold intermediate result.
3. For each structure field in the ascending order of [field numbers](https://developers.google.com/protocol-buffers/docs/proto3#assigning_field_numbers) do:
  * APPLY_HASH flag is **not set**:
    1. If field is `optional` and is not set, append `<null>` reserved word to `tmp`.
    2. Otherwise, append `EncodeValue(field_value, 0)` to `tmp`.
    3. Append bus-specific field separator character `<field-sep>` to `tmp`.
  * APPLY_HASH flag is **set**:
    1. If field is `optional` and is not set, append `<null>` reserved word to `tmp`.
    2. Otherwise, if field type is not `string` or `bytes`, append `EncodeValue(field_value, 0)` to `tmp`.
    3. Otherwise, append `string`/`bytes` field value to `tmp` (note, that field value is appended as-is, without additional encoding).
4. If APPLY_HASH flag is not set, return `tmp` as a string
5. Otherwise, calculate SHA-224 hash of `tmp` and return `EncodeValue(hash, 0)`.

### Algorithm

Busrpc specification defines custom protobuf options that control when APPLY_HASH flag is passed to the `EncodeValue` function:
* structure-level option `hashed_struct`
* field-level option `hashed`

[Call endpoint](#endpoint) is created using the following algorithm (note, that creating result endpoint from the call endpoint is trivial):
1. Fill `<namespace>`, `<class>` and `<method>` components with the namespace, class and method names correspondingly. Note, that this components may contain only alphanumeric symbols and underscores, thus do not require additional encoding.  
2. If class is `static`, append `<null>` reserved word to the endpoint.
3. Otherwise (let `object_id` be an instance of `ObjectId` structure:
    1. If `hashed_struct` is not set of `false` for `ObjectId`, append `EncodeValue(object_id, 0)` to the endpoint.
    2. Otherwise, append `EncodeValue(object_id, APPLY_HASH)` to the endpoint.
4. For each [observable parameter](#observable-parameters) in the ascending order of their [field numbers](https://developers.google.com/protocol-buffers/docs/proto3#assigning_field_numbers):
    1. If `hashed` option is not set or `false` for parameter, append `EncodeValue(param_value, 0)` to the endpoint.
    2. Otherwise, append `EncodeValue(param_value, APPLY_HASH)` to the endpoint.
5. Append `<eof>` reserved word.

### Examples

To complete description of the `EncodeValue` function and endpoint encoding algorithm this section contains examples of their usage.

In this section we assume that underlying message bus prohibits/reserves the following characters in it's topics:
* all non-printable characters (0x00 - 0x31)
* `space`, `$`, `.`

This allows us to use the following characters and reserved words for the purpose of encoding algorithm:

| Token         | Value    |
| ------------- | -------- | 
| `<esc>`       | `%`      |
| `<field-sep>` | `:`      |
| `<null>`      | `%null`  |
| `<empty>`     | `%empty` |
| `<eof>`       | `%eof`   |

Note, that characters reserved for `<esc>` and `<field-sep>` also need to be encoded, so the complete list of prohibited/reserved characters is: 0x00-0x1F, `space`, `$`, `%`, `:`.

First consider how various structures are converted to string that can be subsequently used as an endpoint component.

```
enum MyEnum {
  MYENUM_0 = 0;
  MYENUM_1 = 7;
}

message S1 { }

message S2 {
  bool f1 = 7;
  int32 f2 = 6;
  int32 f3 = 5;
  int32 f4 = 4;
  MyEnum f5 = 3;
  string f6 = 2;
  bytes f7 = 1;
}

message S3 {
  optional string f1 = 1;
}
```

| Value                                                                | Flags      | Result                                                      |
| -------------------------------------------------------------------- | ---------- | ----------------------------------------------------------- | 
| s1 = ()                                                              | 0          | "%empty"                                                    |
| s1 = ()                                                              | APPLY_HASH | "%empty", see (1)                                           |
| s2 = (true, 10, 0, -10, MYENUM_1, "$aaa. bbb%:", <0x10, 0xaf, 0xb5>) | 0          | "10afb5:%24aaa%2e%20bbb%25%3a:7:-10:0:10:1:", see (2)       |
| s2 = (true, 10, 0, -10, MYENUM_1, "$aaa. bbb%:", <0x10, 0xaf, 0xb5>) | APPLY_HASH | see (3)                                                     |
| s3 = ()                                                              | 0          | "%null:"                                                    |
| s3 = ()                                                              | APPLY_HASH | sha224("%null:")                                            |
| s3 = ( "" )                                                          | 0          | "%empty:"                                                   |
| s3 = ( "" )                                                          | APPLY_HASH | sha224("%empty:")                                           |
| s3 = ( "$aaa. bbb%:" )                                               | 0          | "%24aaa%2e%20bbb%25%3a"                                     |
| s3 = ( "$aaa. bbb%:" )                                               | APPLY_HASH | sha224("$aaa. bbb%:")                                       |

Footnotes:
1. Algorithm encodes null or empty values to the same string ("%null" or "%empty") whether APPLY_HASH is specified or not.
2. Fields are encoded in the ascending order of their numbers, not in the declaration order! Also note, that field separator `:` must always be added, even for the last field. This allows to distinguish null/empty structure values from the values of structures that have one `string`/`bytes` field, whose value is null/empty.
3. Hash is calculated not for the string from the previous row, but for the following binary data (`string`/`bytes` value are added as-is, without encoding):
```
0x10 0xaf 0xb5     0x24 0x61 0x61 0x61 0x2e 0x20 0x62 0x62 0x62 0x25 0x3a 0x37 0x2d 0x31 0x30 0x30 0x31 0x30 0x31
<0x10, 0xaf, 0xb5> "$aaa. bbb%:"                                          "7"  "-10"          "0"  "10"      "1"
```

Finally, consider an example of obtaining endpoint for a method call. We use sligthly modified class `user` and it's method `send_message` to demonstrate encoding algorithm. Aforementioned modifications are introduced to remove limitation on a username size.

```
// file api/chat/user/class.proto

message ClassDesc {
  message ObjectId {
    option (hashed_struct) = true;
    string username = 1;
  }
}
```

```
// file api/chat/user/send_message/method.proto

message MethodDesc {
  message Params {
    string receiver = 1 [(observable) = true, (hashed) = true];
    string text = 2;
  }

  message Retval { }
}
```

Now if user "Alice" sends message to user "Bob", the endpoint will look like this: `chat.user.send_message.<sha224("Alice")>.<sha224("Bob")>.%eof`. Or, if we expand the hash: `chat.user.send_message.6874ecdbdb214ee888e37c8c983e2f1c9c0ed16907b519704db42bb6.279f0aba2b90ee54755e3772e7f4bd5599e46400617a7c080b955b9c.%eof`.

# Documenting API

Useful and consistent documentation is a must for any API. The cornerstone of documentation of a busrpc-compliant API is "code as documentation" principle. By simply inspecting busrpc project directory developers can obtain information about:
* API main entities (namespaces, classes, methods and structures)
* endpoints, where API is available
* system infrastructure:
  * services
  * their responsibilities (i.e., which methods are implemented or invoked by each service)
  * their configuration parameters

Besides this, busrpc specification defines a mechanism for documenting individual API entities using protobuf comments and busrpc-specific [documentation commands](#documentation-commands). Later busrpc [development tool](https://github.com/pananton/busrpc-dev) can be used to parse protocol files and build an API documentation from them. Those who used [Doxygen](https://doxygen.nl/) project will found this approach very familiar.

## Basic rules

**Block comment** is a sequence of 1 or many protobuf comments without any gaps between them. Which format is used for comments does not matter (can be any combination of `//` and `/* ... */` comments).

The following example contains 2 block comments: first consists of lines 1-3, second consists of lines 6-7

```
// block 1, line 1
// block 1, line 2
// block 1, line 3

/* block 2, line 1
   block 2, line 2 */
```

To bind block comment to some protobuf entity, it should be placed right **before** the entity, i.e. no empty lines allowed between the block comment and the bound entity. The first line of a bound block comment is treated as entity's **brief** description. First line together with all other block lines (except those containing documentation command, see next section) represent **full** entity description (or simply description).

```
// Brief description of MyEnum.
// Additional information about MyEnum.
enum MyEnum {
  // Brief description of MYENUM_VALUE_0.
  // Additional information about MYENUM_VALUE_0.
  MYENUM_VALUE_0 = 0;

  // This comment is not bound!
  // MYENUM_VALUE_1 is not documented.

  MYENUM_VALUE_1 = 1;
}
```

Block comment bound to the descriptor is considered as corresponding entity description. For example, block comment bound to `MethodDesc` provides method description.

It is important to note that when busrpc [development tool](https://github.com/pananton/busrpc-dev) generates documentation for the busrpc project, it **preserves** all whitespaces in the comment lines, which are included in the documentation. This is done to allow third-parties to use various documentation formats inside block comments, including those using indentation as a control sequences (for example, markdown).

## Documentation commands

Documentation commands allow to specify important information related to busrpc entities (methods, services, etc.). Each command is specified as a single comment line `\name [value]` placed somewhere in the entity's block comment. Note, that commands may interleave with general description lines, however, better to place them at the end of the block comment.

Every command may be specified more than once with the same or disinct values. In that case command is called *multivalued*. How multiple values are interpreted depends on the command semantics. For example, they may be considered an error and only one (first or last) value will be taken, or may be merged in some way to represent complex information.

Block comment in the next example represents the following documentation:
* brief description is " Brief description" (note, that leading space is preserved)
* full description is an array, consisting of lines " Brief description", " Line 1" and " Line 2"
* command "cmd1" has value "value1"
* command "cmd2" has 3 values "" (empty string), "value2" and "hello world"

```
// \cmd2
// Brief description
// \cmd1 value1
// Line 1
// \cmd2 value2
// Line 2
// \cmd2 hello world
```

### Documentation command reference

Table below contains alphabetically sorted list of supported documentation commands along with information about their applicability (i.e., protobuf entities they can be applied to) and value semantics.

| Name   | Applicability       | Value                                                                                             |
| ------ | ------------------- | ------------------------------------------------------------------------------------------------- | 
| accept | `Implements` fields | Pair `paramName acceptedValueDescription`. Indicates that service accepts only calls with specific values of observable parameter(s). If `paramName` is `@object_id`, then command relates to object identifier. Can be multivalued. |
| author | `ServiceDesc`       | Service author (a person or a team).                                                               |
| email  | `ServiceDesc`       | Service author contact email.                                                                      |
| post   | `MethodDesc`        | String describing in some way postcondition of a method call.                                      |
| pre    | `MethodDesc`        | String describing in some way precondition of a method call.                                       |
| url    | `ServiceDesc`       | Service sources URL.                                                                               |

# Specializations

Specializations complete this specification by defining various parameters of the framework for a specific message bus implementation. At the very least, every specialization should define concrete values for special tokens used throughout the specification and a complete list of reserved characters that should be hex-encoded by the endpoint encoding algorithm.

| Token                      | Description                                                                                                    |
| ---------------------------|--------------------------------------------------------------------------------------------------------------- |
| `<topic-word-sep>`         | Character used to separate words in a message bus topic.                                                       |
| `<topic-wildcard-any1>`    | Wildcard character representing a single arbitrary word in a message bus topic.                                |
| `<topic-wildcard-anyN>`    | Wildcard character representing a 1 to many or 0 to many arbitrary words in a message bus topic.               |
| `<result-endpoint-prefix>` | Common prefix for endpoints to which messages representing method result are sent.                             | 
| `<eof>`                    | Reserved word indicating last word in the endpoint.                                                            |
| `<empty>`                  | Reserved word indicating empty value of the endpoint word.                                                     |
| `<null>`                   | Reserved word indicating absent value for the endpoint word.                                                   |
| `<esc>`                    | Escape character used by the endpoint encoding algorithm.                                                      |
| `<field-sep>`              | Separator between structure fields when structure is converted to the endpoint word by the encoding algorithm. |

Additionally, specialization may provide other useful information regarding it's message bus:
* known issues
* best practicies
* etc.

Available specializations can be found in [*specializations/*](specializations) directory.
