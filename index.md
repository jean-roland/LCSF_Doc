<p hidden># LCSF Project</p>

## Presentation

The LCSF project is a set of tools and documentation based around the LCSF (Light Command Set Format). The goal is to simplify and accelerate the development and deployment of custom sets of commands so that you can focus on your actual project instead of fiddling with low-level protocols.

It was mainly conceived with IoT/M2M applications in mind where communication channels are heavily restricted in volume and speed.

## When to use

Whenever you're developing an application where distant systems with limited resources need to automatically exchange data (send order, retrieve sensor measurements, status, reports...). You will often do so using a custom command set and responses.

At it's simplest, you can get away with sending ASCII characters, but when you start introducing data payloads that vary in length, when you start to have multiple versions of your commands coexisting and conflicting, it gets annoying to maintain.

On the other end of the "power spectrum" you will probably use gRPC or RESTful API, which can be resource-intensive, and you may want to consider a more lightweight option like LCSF.

The LCSF project provides you the tools to simplify the tedious process of developing, deploying and maintaining your custom command set. In exchange for the additional complexity, you get:
* Ability to represent virtually all protocols with infinitely branching attributes.
* Possibility to have different coexisting command sets.
* Automatically decode/encode messages and validate data payload.
* GUI to safely generate/edit command sets and documentation, allowing you to iterate easily.
* Built-in error handling protocol.

## Project components

The main components of the LCSF Project are:

* LCSF itself, which is two things:
    1. A specification to describe command sets that can fit most applications, even complex ones.
    2. A lightweight format to represent those command sets.
* [LCSF C Stack](https://github.com/jean-roland/LCSF_C_Stack): An embedded-friendly LCSF implementation written in C, add it to your project to easily encode and decode LCSF messages.
* [LCSF Stack Rust](https://github.com/jean-roland/LCSF_Stack_Rust): A memory-safe LCSF implementation written in Rust.
* [LCSF Generator](https://github.com/jean-roland/LCSF_Generator): A C++/Qt graphic tool used to create, edit and deploy LCSF protocols. It generates code for the LCSF stacks and documentation (wiki and markdown format).

For more information on the other components, check their respective documentation.

## Real-world examples

The following are two applications where LCSF was used:
* A protocol to update FPGA firmware binaries to a dozen distant micro-controllers, with integrity control.
* A protocol to process the UI of a product (screen + buttons) controlled by a micro-controller from a distant Linux computer. The intelligence of the UI was deported on the Linux because of architectural constraints.

## LCSF Documentation

This is the documentation of the LCSF specification.

### Protocol

A command set in LCSF is the top hierarchical object, and is called a protocol.
A protocol is composed of a number of commands.

### Command Type

There are two types of commands:
* Simple commands, that are enough by themselves (e.g. ping, acknowledge, reset, abort...).
* Complex commands, that contain a data payload (e.g. a jump command that contains the jump address in its payload).

### Command Payload

The command payload is a list of attributes (e.g. a send file command will contain, at least, a file size attribute and a file data attribute).
Since attributes are here to structure the command payload, all attributes must have a data payload, otherwise they don't have a reason to exist.

### Command Direction

Your protocol might have a notion of master and slave or in more generic terms, a protocol asymmetry where there are two different point of view.

The way it is handled in LCSF is that you simply give each command a "direction":
* (A -> B): A can only send the command, B can only receive it.
* (B -> A): B can only send the command, A can only receive it.
* (A <-> B) / Bidirectional: Both A and B can receive and send the command.

### Attribute Type

There are two types of attributes:
* Simple attributes, that have data payload (e.g. a jump address attribute that contains the address itself).
* Complex attributes, that have a list of sub-attributes payload to describe more complex objects (e.g. a color space attribute that will contain as sub-attributes its type (RGB, YUV, HSL...) and its three components).

### Sub-attribute

Sub-attributes are attributes in their own right. This means that if you have a data type `Address` in your protocol, you can use it as either an attribute or a sub-attribute.

Also, sub-attributes can have their own sub-attributes and there is no limit to the amount of attribute branching/nesting you can do. This is one of the key feature of LCSF and gives it the flexibility to describe most, if not all, command sets that you may need to create for your applications.

### Optional Attribute

Attributes have a mandatory payload but there are cases where an attribute won't be always there (e.g. a wait command may have a waiting time attribute or a default value if no time is given).

To avoid sending useless data, attributes can be set as optional, otherwise they're mandatory.

### Attribute Data Type

Simple attributes are given a data type to their payload. The different data types are:
* `(u)int8`
* `(u)int16`
* `(u)int32`
* `String`
* `Byte Array`

All the number types have a maximum size of respectively (1, 2 and 4 bytes). Basic variable length encoding is used to remove unnecessary null value bytes during transmission.
Strings must be null-terminated, therefore their sizes are implicit.

### Command Sequence

Protocols don't have an intrinsic notion of sequences. They must be handled at the application level, using a state machine or something similar.

### Good Practice

Creating your protocol and defining the different commands and attributes is not trivial and there is many ways to do one thing. This is why we propose the following good practices:

* To respect strict self-containment of protocol layers, if you have an attribute type of undetermined size like byte array (e.g. a file data attribute), you must accompany it with a size attribute (e.g. a `uint32` file size attribute).
One exception of this would be if you have fixed length array in your protocol (e.g. a SHA1 digest will always be 20 bytes) as the attribute itself will imply the size of the array.
* An attribute can only appear once in a command or as a sub-attribute. Therefore, if you need to send multiple similar pieces of data, you should consider using an array attribute to regroup them.
* Complex attributes are costly, they take a few bytes to represent and a recursive call to process. You should only use them if the hierarchical data they carry has value and/or you don't know in advance the data you want to send. For example, if you want to *stream* any file system with LCSF, both conditions are met.

### Wrapping-up

The following diagram sums up how a command set is structured:

![LCSF structure](./img/Struct.png)

## LCSF Message

As to how LCSF protocols are represented we need to distinguish two things:
* The protocol description, that represents all the commands and attributes that can be exchanged.
* A protocol message, that is the unit of data that will be transferred between a sender and a receiver. It will contains only one command and its attributes, if the command has any.

### Protocol Description

The description format is language-dependent and will vary, but it will be based around arrays of structures, detailing the commands and their attributes.

### Format Endianness

The format is little endian.

### Identifier Space

Protocols, commands and attributes have separate identifiers spaces as they are considered different objects. Actually, every commands and attributes have their own (sub) attribute identifier space.

There is no problem with attributes of different commands having the same identifiers.

### Protocol Message

The message structure is defined as:
* Protocol id: The user-defined protocol identifier. One id value (usually `~0`) is reserved for the built-in lcsf error protocol.
* Command id: The user-defined command identifier of the command being sent.
* Attribute number: The command's number of attributes.
* 1st Attribute id: The user-defined identifier of one of the command attributes.
* Complexity flag: Flag value is 0 if payload is data, 1 if sub-attributes.
* Payload size: Either the data size or the number of sub attributes, depending on complexity flag.
* Attribute payload.
* 2nd Attribute id.
* ...

Simple Attribute structure:
* Attribute id
* Complexity flag
* Data size
* Data payload

Complex Attribute structure:
* Attribute id
* Complexity flag
* Sub-attributes number
  * 1st sub-attribute id
  * Complexity flag
  * Payload size
  * Sub-attribute payload
  * 2nd sub-attribute id
  * ...

### Wrapping-up

The following diagram sums up how a message is formatted:

![LCSF structure](./img/Trame.png)

### Standard representation

Below is the standard representation for the different LCSF message components:
* Protocol id: `16 bits`, `65535` possible values - `0xFFFF` reserved for lcsf error protocol.
* Command id: `16 bits`, `65536` possible values.
* Attribute number: `16 bits`, up to `65535` attributes per command.
* Attribute id: `15 bits`, `32768` possible values - MSB used by complexity flag.
* Complexity flag. `1 bit`, `2` values - `0` indicates a simple attribute and `1` a complex attribute.
* Payload size: `16 bits`, up to `65535` sub-attributes or bytes of data.
* Data payload: User defined, in the limit of the payload size capabilities.

This means that small message sizes are:
* Simple command: `6 bytes`.
* Command with a simple attribute: `10 bytes + payload size`.

### Smaller representation

This smaller representation aims to reduce message overhead/size at the cost of representation space:
* Protocol id: `8 bits`, `255` possible values - `0xFF` reserved for lcsf error protocol.
* Command id: `8 bits`, `256` possible values.
* Attribute number: `8 bits`, up to `255` attributes per command.
* Attribute id: `7 bits`, `128` possible values - MSB used by complexity flag.
* Complexity flag. `1 bit`, `2` values - `0` indicates a simple attribute and `1` a complex attribute.
* Payload size: `8 bits`, encodes up to `255` sub-attributes or bytes of data.
* Data payload: User defined, in the limit of the payload size capabilities.

In this representation, small message sizes are:
* Simple command: `3 bytes`.
* Command with a simple attribute: `5 bytes + payload size`.

### Custom representations

If those representations don't correspond to your application, you can create a custom one.
In the future, other representations may be supported by the LCSF environment.

### Protocol Versioning

You might run in a case where different systems will use different versions of the same protocol. If the different versions have the same identifier, it might lead to errors as one of the system might use a newer command that the other system doesn't understand.

A simple way to avoid this issue is to change the protocol identifier each time the protocol is changed after initial deployment (e.g. using one byte for the version number).

## LCSF Error Protocol

LCSF has a built-in error protocol to report problems encountered while processing incoming LCSF messages back to the sender.

To report errors at a protocol level (eg: valid command received at wrong step of a sequence), you should either have a dedicated error command in your protocol or a dedicated error protocol.

The protocol id is `0xFFFF` (standard representation). It has only one command "Error" with two mandatory attributes "Error location" and "Error type".
* Error location indicates if the message has bad formatting or if it doesn't correspond to its protocol description
* Error type gives more information on the error nature

The tables below describe the protocol in more details.

Command description table:

| Command Name | Command Id  | Direction | Attributes (mandatory)       | Attributes (optional) | Description  |
|:-------------|:------------|:----------|:-----------------------------|:----------------------|:-------------|
| `Error`      |  `0x00`     | `A <-> B` | `Error_Location, Error_Type` | `/`                   | Indicates an error occurred when decoding a LCSF message |

Attribute description table:

| Attribute        | Attribute Id | Data Type | Description |
|:-----------------|:-------------|:----------|:------------|
| `Error_Location` | `0x00`       | `uint8`   | Enum describing the error location (bad formatting or protocol issue) |
| `Error_Type`     | `0x01`       | `uint8`   | Enum describing the type of error |

Error_Location enum table:

| Enum name          | Enum Value | Description |
|:-------------------|:-----------|:------------|
| `DECODE_ERROR`     | `0x00`     | Error happened while decoding (formatting error) |
| `VALIDATION_ERROR` | `0x01`     | Error happened while validating (protocol error) |

Error_Type enum table if decode error:

| Enum name        | Enum Value | Description |
|:-----------------|:-----------|:------------|
| `FORMAT_ERROR`   | `0x00`     | Message format error, missing or leftover bytes |
| `OVERFLOW_ERROR` | `0x01`     | Message is too big to be processed |
| `UNKNOWN_ERROR`  | `0xFF`     | Unspecified error |

Error_Type enum table if validation error:

| Enum name              | Enum Value | Description |
|:-----------------------|:-----------|:------------|
| `UNKNOWN_PROTOCOL_ID`  | `0x00`     | Unrecognized protocol id |
| `UNKNOWN_COMMAND_ID`   | `0x01`     | Unrecognized command id |
| `UNKNOWN_ATTRIBUTE_ID` | `0x02`     | Unrecognized attribute id |
| `TOO_MANY_ATTRIBUTES`  | `0x03`     | More attributes received than expected |
| `MISSING_NON_OPTIONAL` | `0x04`     | A non optional attribute is missing |
| `WRONG_DATA_TYPE`      | `0x05`     | Attribute data is of wrong type/size |
| `UNKNOWN_ERROR`        | `0xFF`     | Unspecified error |
