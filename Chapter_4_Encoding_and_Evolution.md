# Chapter 4 - Encoding and Evolution

*07 Dec 2019*

These are my notes from the fourth chapter of Martin Kleppmann's: Designing Data Intensive Applications.

**Table of Contents**

- [Formats for Encoding Data](#formats-for-encoding-data)
    - [Language-Specific Formats](#language-specific-formats)
    - [JSON, XML, and Binary Variants](#json-xml-and-binary-variants)
        - [Binary Encoding](#binary-encoding)
        - [Modes of Dataflow](#modes-of-dataflow)
            - [Dataflow Through Databases](#dataflow-through-databases)
            - [Dataflow Through Services: REST and RPC](#dataflow-through-services-rest-and-rpc)
            - [Message-Passing Dataflow](#message-passing-dataflow)
                - [Advantages of a message broker](#advantages-of-a-message-broker)
                - [Message brokers](#message-brokers)
                - [Distributed actor frameworks](#distributed-actor-frameworks)

* * *

## Introduction

We should aim to build systems that make it easy to adapt to change: **Evolvability.**

**Rolling upgrade:** Deploying a new version to a few nodes at a time, checking whether the new version is running smoothly, and gradually working your way through all the nodes.

With rolling upgrades, new and old versions of the code, and old and new data formats may potentially all coexist in the system at the same time. For a system to run smoothly, compatibility needs to be in both directions.

**Backward compatibility:** Newer code can read data that was written by older code. Simpler to achieve.

**Forward compatibility:** Older code can read data written by newer code. This is trickier because it requires older code to ignore additions made by a newer version of the code.

----

Programs work with data that have at least 2 different representations:

- In memory data structures: optimized for efficient access and CPU manipulation
- Sequence of bytes (e.g. JSON) for transmitting over the network.

---- 

We need some kind of translation between the two representations:

**Encoding/Serialization/Marshalling** \- Translation from in-memory representation to a byte sequence.

**Decoding/ Deserialization** \- Byte sequence to in-memory representation.

There are a number of different libraries and encoding formats to choose from which we'll discuss next.

## Language-Specific Formats [#](#language-specific-formats)

Different programming languages have their built-in support for encoding in-memory objects into byte sequences. Java has Serializable, Ruby has Marshal and so on. However, these language-specific encodings have their own problems:

- The encoding is tied to a particular programming language, and reading the data in another language is difficult.
- Versioning data is often an afterthought. They often neglect the problems of backward and forward compatibility since they're intended for quick and easy use.
- Efficiency is also an afterthought. Java's serialization is notorious for bad performance and bloated encoding.

## JSON, XML, and Binary Variants [#](#json%2C-xml%2C-and-binary-variants)

JSON and XML are the obvious contenders for standard encodings. CSV is another option. These formats are widely known, widely supported, and almost as widely disliked.

XML is often criticized for its verbose syntax. Apart from superficial syntactic issues, they also have subtle problems:

- *There's ambiguity around how numbers are encoded.* In XML and CSV, there's no distinction between a number and a string that happens to have digits. JSON distinguishes both, but does not distinguish integers and floating-point numbers, and does not specify a precision.
- No support for binary strings (sequences of bytes without a character encoding)
- Optional schema support for XML and JSON

### Binary Encoding [#](#binary-encoding)

The choice of data format can have a big impact especially when the dataset is in the order of terabytes.

JSON is less verbose than XML, but both still use a lot of space compared to binary formats. This has led to a number of binary encodings for JSON (BSON, BJSON etc.) and XML (WBXML etc.). BSON is used as the primary data representation in MongoDB for example.

**Thrift and Protocol Buffers:** These are binary encoding libraries. Protocol Buffers was developed at Google, while Thrift was developed at Facebook.

**Avro:** Another binary encoding format different from the two above. This started out as a sub project of Hadoop.

These encoding libraries have some interesting encoding rules which I skipped: [http://martin.kleppmann.com/2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html](http://martin.kleppmann.com/2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html)

### Modes of Dataflow [#](#modes-of-dataflow)

Recall that it was stated earlier that to send data from one process to another with which you don't share memory, it needs to be encoded as a sequence of bytes.

We also said that forward and backward compatibility are important.

Here, we'll explore how data flows between processes:

#### Dataflow Through Databases [#](#dataflow-through-databases)

The process that writes to a database encodes the data, while the process the reads from it decodes it. It could be the same process doing both, or different processes.

*Forward compatibility* is required in databases: If different processes are accessing the database, and one of the processes is from a newer version of the application ( say during a rolling upgrade), the newer code might write a value to the database. Forward compatibility is the ability of the process running the *old* code to be able to read the data written by the new code.

We also need *backward compatibility* so that code from a newer version of the app can read data written by an older version.

Data outlives code and often times, there's a new to migrate data to a new schema. Avro has sophisticated schema evolution rules that can allow a database to appear as if was encoded with a single schema, even though the underlying storage may contain records encoded with previous schema versions.

#### Dataflow Through Services: REST and RPC [#](#dataflow-through-services%3A-rest-and-rpc)

When there's communication between processes over a network, a common arrangement is to have two roles: *clients* (e.g. web browser) and *servers.*

The server typically exposes an API over the network for the client to make requests. This API is known as a *service.*

A server can also be a client to another service. E.g. a web app server is usually a client to a database.

A difference between a web app service and a database service is that there's usually tighter restriction on the former.

----

*Service-oriented architecture (SOA)*: Decomposing a large application into smaller components by area of functionality.

**Web Services**: If a service is communicated with using HTTP as the underlying protocol, it is called a *web service.* Two approaches to web services are *REST* and *SOAP* (Simple Object Access Protocol).

**RPC:** The RPC model tries to make a request to a remote network look the same as calling a function or method, within the same process ( *location transparency -* In computer networks, location transparency is the use of names to identify network resources, rather than their actual location).

----

There are certain problems with this approach though, which can be summarized under the fundamental fact that network calls are different from function calls. E.g.:

- A local function call is predictable and succeeds or fails depending on parameters under my control . A network call is unpredictable - the request or response packets can get lost, the remote machine may be slow etc.
- A local function call either returns a result, or throws an exception, or never returns (infinite loop). A network request has another possible outcome, it may return without a result, due to a timeout. There's no way of knowing whether the request got through or not.
- Retrying a failed network request could cause the action to be performed multiple times if the request actually got through, but the response was lost. Building a system for idempotence could prevent this though.
- The client and service may be implemented in different languages, so the RPC library would need to translate datatypes from one language to another. This is tricky because not all languages have the same types. This process does not exist in a single process written in a single language.

Despite these problems, RPC isn't going away. The new generation of RPC frameworks are explicit about the difference between a remote request and a local function call such as Finagle, [Rest.li](http://rest.li/) and GRPC.

----

The main focus of RPC frameworks is on requests between services owned by the same organization, typically within the same datacenter.

*Q - How exactly do RPCs differ from REST? Is it just the way the endpoints look?*

#### Message-Passing Dataflow [#](#message-passing-dataflow)

Asynchronous message-passing systems are somewhere between RPC and databases.

- Similar to RPCs because a client's request is delivered to another process with low latency (*q: What makes RPCs delivered with low latency? Do RPCs have low latency compared to REST calls? Or is this just when compared with databases? If it's the former, what makes it so given that they're both network calls? If it's the latter, I wonder what makes HTTP requests faster than say a JDBC request )*
- Similar to databases in that message is not sent via a direct network connection, but via an intermediary called a *message broker* or *message queue.*

##### Advantages of a message broker [#](#advantages-of-a-message-broker)

- It can act as a buffer if the recipient is down or unable to receive messages due to overloading.
- The sender can act without knowing the IP address and port number of the recipient (which can change quite often - especially in a cloud deployment where VMs come and go)
- A message can be delivered to multiple recipients.
- It can retry message delivering to a crashed process and prevent lost messages.
- It decouples the sender from the recipient. The sender does not need to know anything about the recipient.

The communication pattern here is usually asynchronous - the sender does not wait for the message to be delivered, but simply sends it and forgets about it.

##### Message brokers [#](#message-brokers)

The configuration settings of message brokers typically vary, but in general they're used like:

- A process sends a message to a named *queue* or *topic*
- The broker ensures that the message is delivered to one or more *consumers* or *subscribers* to that queue or topic.

A topic can have many producers and many consumers.

##### Distributed actor frameworks [#](#distributed-actor-frameworks)

The *actor* model is a programming model for concurrency in a single process. Each part of the system is represented as an actor. An actor is usually a client or an entity which communicates with other actors by sending and receiving asynchronous messages.

In distributed actor frameworks, this model is especially useful for scaling an application across multiple nodes as the same message-passing mechanism is used, regardless of whether the sender and recipient are on the same or different nodes.

This framework integrates the actor programming model and the message broker into a single framework. 3 popular distributed actor frameworks are:

- Akka
- Orleans
- Erland OTP