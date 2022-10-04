# Busrpc specification

This document contains general information for developers of busrpc-compliant microservices and tools: busrpc terminology, network model, protocol, API documentation, bus-dependent specializations of this specification, etc.

* [Introduction](#introduction)
* [Goals](#Goals)
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

# Introduction

As appears from it's name, busrpc framework stands on two pillars.

First of all, it is a form of a remote procedure call (RPC) technology (like [grpc](https://grpc.io/), [json-rpc](https://www.jsonrpc.org/) and similar) used for inter-service communication.

Secondly, it relies on a message bus/queue/broker component as a transport layer. Usual RPC implementations mostly operate in a peer-to-peer manner which means that communicating parties need to connect directly to each other. This is probably ok for the systems with small number (1-5) of services, but does not suite well for microservice backends where number of services can easily surpass 100. Instead, microservice backends utilize a dedicated component (called message bus/queue/broker) as a central point of communication responsible for inter-service message delivery and routing. This greatly simplifies system configuration (only message bus address is required to access any part of the system API), service deployment (because all services are loosely-coupled in this scheme) and administration.

# Goals

[Microservice architecture](https://en.wikipedia.org/wiki/Microservices) (MA) is a common standard nowadays for developing large systems backends. Basic MA principles and patterns are described in a numerous amount of technical articles, publications and books. However, low-level details are frequently left underspecified. Busrpc framework main goal is to offer a turnkey solution for backend developers who apply MA for their projects.

Because MA implies that all development is distributed among some (probably - large) number of small (from 1 to 5 developers) loosely-coupled teams, the following sub-goals were also specified:
* Busrpc framework should establish a common intuitive terminology to facilitate communication between development teams. 
* 
* To make learning curve for busrpc newcomers shorted Introduction of specific New concepts and definitions should be avoided and terminology should build on a well-known Framework should avoid introduction of new concepts and definitions and instead build on a well-known Instead of providing new concepts and definitions, busrcp terminology should build on a well-known concept 
* Busrpc framework should support (at least - *potentially*) a variery of message bus/queue/broker implementations and programming languages.
* Busrpc fram





Primary goal of the busrpc project is to provide a microservice development framework (consisting of technical documentation, tools and client libraries) suitable for a variety of platforms and programming languages.
* Because in 



 



lot of useful information regarding it can be found around the net.
Busrpc API specification is intended for developers following microservice architecture (MA) principles 

Busrpc API design is motivated by the following points:

Motivation behind busrpc is 




# Documentation commands

# Specializations

Some aspects of the busrpc API design were intentionally left unspecified in this document to avoid dependency on a particular message bus/queue/broker implementation. This unspecified aspects are defined in a seperate bus-dependent documents called *specializations*.

Currently the following specializations exist (more to be added):
* NATS [specialization](./docs/specializations/nats-busrpc.md)
