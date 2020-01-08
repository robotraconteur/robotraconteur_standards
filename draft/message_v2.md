<p align="center"><img src="../images/RRheader2.jpg"></p>

# Robot Raconteur Message Version 2 Serialization Format

http://robotraconteur.com

http://github.com/robotraconteur

Copyright &copy; 2020 Wason Technology, LLC

*Robot Raconteur is a communication framework designed for use with robotics, automation, building control, and internet of things applications.*

## Abstract

Robot Raconteur communicates between nodes using messages, as described in [Robot Raconteur Framework Architecture Standard](framework_standard.md), [Robot Raconteur Object Protocol Standard](object_protocol.md), and [Robot Raconteur Value Types Standard](value_types.md). Messages are defined in these documents as data structures independent of their representation. The Message Version 2 Serialization Format (Message2) specifies a binary serialization format that can be used to serialize the messages to binary form for use on a data stream or storage device. This format is the canonical binary message form, with all implementations expected to support it.

## Introduction

Robot Raconteur uses "messages" to communicate between nodes. These messages have standard structures, as described in [Robot Raconteur Framework Architecture Standard](framework_standard.md). Messages consist of the "Message", the highest level container with routing information, "Message Entries", which contain a single request, response, or packet, and "Message Elements", which contain values. The Message Version 2 Serialization Format specifies how the message structures are serialized to binary form.

## Endianness

**Message 2 is always little endian.**

(Except for UUID which are always big endian.)

## String Format

Strings are always encoded as utf-8

## MetaData

MetaData is a string available for future use. Additional fields in the MetaData is stored one field per line, with either a single name, or a name with a value separated by a colon.

Examples fields:

    a_single_name
    a_name_with: a value of any format

## UUID

UUIDs are 16 byte unique identifies. UUID are always stored in "big endian" binary format. (Not Microsoft GUID format)

## Common Message Header

Messages of all versions must start with the common header:

| Field Name           | Type   | Bytes  | Description |
| ---                  | ---    | ---    | ---         |
| "RRAC"               | utf-8  | 4      | Magic at start of message. Must be "RRAC" { 0x52, 0x52, 0x41, 0x43 } |
| MessageSize          | uint32 | 4      | Total bytes in message including header |
| MessageVersion       | uint16 | 2      | The version of the serialized message that follows |

## Message

The message structure is first, before message elements and message entries. The MessageSize field contains the total size of the message in bytes.

The message header is defined as:

| Field Name           | Type   | Bytes  | Description |
| ---                  | ---    | ---    | ---         |
| "RRAC"               | utf-8  | 4      | Magic at start of message. Must be "RRAC" { 0x52, 0x52, 0x41, 0x43 } |
| MessageSize          | uint32 | 4      | Total bytes in message including header |
| MessageVersion       | uint16 | 2      | Version of message serialization format. Always 2 for Message2 |
| HeaderSize           | uint16 | 2      | Size of the message header in bytes before entries begin |
| SenderNodeID         | UUID   | 16     | The NodeID of the sender as a UUID |
| Receiver NodeID      | UUID   | 16     | The NodeID of the intended receiver as a UUID |
| SenderEndpoint       | uint32 | 4      | The sender endpoint number |
| ReceiverEndpoint     | uint32 | 4      | The receiver endpoint number |
| SenderNodeName_len   | uint16 | 2      | The length of the NodeName of the sender in bytes after utf-8 encoding |
| SenderNodeName       | utf-8  | varies | The NodeName of the sender encoded as utf-8 string |
| ReceiverNodeName_len | uint16 | 2      | The length of the NodeName of the intended receiver in bytes after utf-8 encoding |
| ReceiverNodeName     | utf-8  | varies | The NodeName of the receiver encoded as utf-8 string |
| MetaData_len         | uint16 | 2      | The length of MetaData in bytes after utf-8 encoding |
| MetaData             | utf-8  | varies | MetaData encoded as utf-8 string |
| EntryCount           | uint16 | 2      | The number of message entries following header |
| MessageID            | uint16 | 2      | *Unused* |
| MessageResID         | int16  | 2      | *Unused* |

## MessageEntry

A message entry contains a request, response, or packet. The standard usage of entries is discussed in [Robot Raconteur Object Protocol Standard](object_protocol.md).

| Field Name           | Type   | Bytes  | Description |
| ---                  | ---    | ---    | ---         |
| EntrySize            | uint32 | 4      | The total size of the entry in bytes, including message elements |
| EntryType            | uint16 | 2      | The type of the entry (request, response, or packet type) |
| *Reserved*           | uint16 | 2      |  |
| ServicePath_len      | uint16 | 2      | The length of ServicePath in bytes after utf-8 encoding |
| ServicePath          | utf-8  | varies | The ServicePath encoded as utf-8 string |
| MemberName_len       | uint16 | 2      | The length of MemberName in bytes after utf-8 encoding |
| MemberName           | utf-8  | varies | The MemberName encoded as utf-8 string |
| RequestID            | uint32 | 4      | The RequestID to match up request and response messages |
| Error                | uint16 | 2      | The error code, or zero for no error |
| MetaData_len         | uint16 | 2      | The length of MetaData in bytes after utf-8 encoding |
| MetaData             | utf-8  | varies | MetaData encoded as utf-8 string |
| ElementCount         | uint16 | 2      | The number of *top level* elements that follow |

*Note that ElementCount only refers to the top-level elements that follow. Nested elements are not counted in the ElementCount field.*

## MessageElement

Message Elements contain values. The standard usage of message entries is discussed in [Robot Raconteur Value Types Standard](value_types.md).

| Field Name           | Type   | Bytes  | Description |
| ---                  | ---    | ---    | ---         |
| ElementSize          | uint32 | 4      | The total size of the entry in bytes, including array data or nested message elements |
| ElementName_len      | uint16 | 2      | The length of ElementName in bytes after utf-8 encoding |
| ElementName          | utf-8  | varies | The ElementName encoded as utf-8 string |
| ElementType          | uint16 | 2      | Data type code for the data |
| ElementTypeName_len  | uint16 | 2      | The length of ElementTypeName_len in bytes after utf-8 encoding |
| ElementTypeName      | utf-8  | varies | The ElementTypeName encoded as utf-8 string |
| MetaData_len         | uint16 | 2      | The length of MetaData in bytes after utf-8 encoding |
| MetaData             | utf-8  | varies | MetaData encoded as utf-8 string |
| DataCount            | uint16 | 2      | The number of array elements or *top level* elements that follow |

## Data

Data follows message elements. The data can either be an array, or a list of nested message elements. Multiple levels of nesting of message elements are allowed, creating a tree structure.

The following array types are supported:

| Type | Bytes/Element | Description | Data Type Code | Notes |
| --- | --- | --- | --- | --- |
| void | 0 | void/null | 0 | Always zero count |
| double | 8 | 64-bit floating point | 1 | IEEE 754 Double-Precision Floating Point |
| single | 4 | 32-bit floating point | 2 | IEEE 754 Single-Precision Floating Point |
| int8 | 1 | Signed 8-bit integer | 3 | |
| uint8 | 1 | Unsigned 8-bit integer | 4 | |
| int16 | 2 | Signed 16-bit integer | 5 | |
| uint16 | 2 | Unsigned 16-bit integer | 6 | |
| int32 | 4 | Signed 32-bit integer | 7 | |
| uint32 | 4 | Unsigned 32-bit integer | 8 | |
| int64 | 8 | Signed 64-bit integer | 9 | |
| uint64 | 8 | Unsigned 64-bit integer | 10 | |
| string | 1 | utf-8 encoded string | 11 | |
| cdouble | 16 | 128-bit complex floating point | 12 | IEEE 754 Double-Precision Floating Point, interleaved, real first, then imaginary |
| csingle | 8 | 64-bit complex floating point | 13 | IEEE 754 Single-Precision Floating Point, interleaved, real first, then imaginary |
| bool | 1 | Logical boolean | 14 | 0 for False, 1 for True. One byte per element. |

Data is always stored as an array or list of elements. Scalars are stored as arrays with one element.

Nested elements are detected by the ElementType value. There is no universal flag to specify nested elements.

**Array data is always little endian.**
