# Busrpc specification

This document contains general information for developers of busrpc microservices and tools: terminology, network model, protocol design, etc.

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
* underlying network protocol should be extensible and should support conversion between binary (for higher throughoutput and lesser latency) and text (for human-readability) format

# Documentation commands

# Specializations

Some aspects of the busrpc API design were intentionally left unspecified in this document to avoid dependency on a particular message bus/queue/broker implementation. This unspecified aspects are defined in a seperate bus-dependent documents called *specializations*.

Currently the following specializations exist (more to be added):
* NATS [specialization](./docs/specializations/nats-busrpc.md)
