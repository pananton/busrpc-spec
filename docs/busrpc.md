# Busrpc design

This document defines an busrpc framework design.

* [Introduction](#introduction)
* [Network model](#network-model)
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

# Goals

Primary goal of the busrpc project is to provide a microservice development framework (consisting of technical documentation, tools and client libraries) suitable for a variety of platforms and programming languages.
* Because in 


[Microservice architecture](https://en.wikipedia.org/wiki/Microservices)(MA) is a common standard nowadays for developing large systems backends. Among many other things, it determines how to decompose the system on microservices 

 Basic MA principles and patterns are described in a numerous amount of technical articles, publications and books. However, low-level details are frequently left underspecified.



lot of useful information regarding it can be found around the net.
Busrpc API specification is intended for developers following microservice architecture (MA) principles 

Busrpc API design is motivated by the following points:

Motivation behind busrpc is 


# Introduction

As appears from it's name, busrpc API stands on two pillars.

First of all, it is a form of RPC meaning that logical unit of busrpc API is a *method* (like in [grpc](https://grpc.io/), [json-rpc](https://www.jsonrpc.org/) and similar) grouped into *classes*. To *call* it, client sends a message containing values of method *parameters*, etc.) to a service. Service in it's turn performs necessary work (*implements* a method) and then optionally sends message containing method *result* back to the caller.

Secondly, it relies on a message bus/queue/broker component as a transport layer. Usual RPC implementations mostly operate in a peer-to-peer manner meaning that caller needs to connect directly to the service implementing the method. This is probably ok for the systems where API is implemented by a small number (1-5) of services. However, microservice architecture implies that API parts are scattered among large number of small services (can be more than 100) additionally duplicating each other for high-availability and load-balancing. For such architecture, peer-to-peer communication leads to a configuration burden and complex service interconnections which in turn greatly complicates system deployment and management. Instead, MA-based systems utilize a dedicated component (called message bus/queue/broker) responsible for inter-service message delivery and routing. Examples of this component are [NATS](https://nats.io/) and [RabbitMQ](https://rabbitmq.com/). Caller sends a message with call information to the message bus which in turn routes it to an appropriate service(s). Note that caller and service become loosely coupled in this scheme: both only need to connect to a message bus component at a well-known location.

# Documentation commands

# Specializations

Some aspects of the busrpc API design were intentionally left unspecified in this document to avoid dependency on a particular message bus/queue/broker implementation. This unspecified aspects are defined in a seperate bus-dependent documents called *specializations*.

Currently the following specializations exist (more to be added):
* NATS [specialization](./docs/specializations/nats-busrpc.md)
