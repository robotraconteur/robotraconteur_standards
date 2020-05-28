<p align="center"><img src="../images/RRheader2.jpg"></p>

# Robot Raconteur Message Version 4 Serialization Format

http://robotraconteur.com

http://github.com/robotraconteur

Copyright &copy; 2020 Wason Technology, LLC

*Robot Raconteur is a communication framework designed for use with robotics, automation, building control, and internet of things applications.*

## Abstract

Robot Raconteur communicates between nodes using messages, as described in [Robot Raconteur Framework Architecture Standard](framework_standard.md), [Robot Raconteur Object Protocol Standard](object_protocol.md), and [Robot Raconteur Value Types Standard](value_types.md). Messages are defined in these documents as data structures independent of their representation. The [Message Version 2 Serialization Format](message_v2.md) is the canonical serialization format for these messages for transfer between nodes, however this message format has some limitations. These limitations include redundant header information, lack of extensibility, and inefficient representation of strings in headers. Message Version 4 Serialization Format described in this document is designed to improve on these limitations, while containing generally the same information as Message Version 2 Serialization Format.

## Introduction

Robot Raconteur uses "messages" to communicate between nodes. These messages have standard structures, as described in [Robot Raconteur Framework Architecture Standard](framework_standard.md). Messages consist of the "Message", the highest level container with routing information, "Message Entries", which contain a single request, response, or packet, and "Message Elements", which contain values. The Message Version 4 Serialization Format specifies an improved method to serialize messages to binary form, while containing generally the same information as Message Version 2 Serialization Format.

## Endianness

**Message 4 is always little endian.**

(Except for UUID which are always big endian.)

## String Format

Strings are always encoded as utf-8

## uint_x and uint_x2

An expanding unsigned integer is used for lengths to save space. This type can represent up to a 64-bit unsigned integer, although only up to 32-bit unsigned integers are used in most places of the Message 4 format. The `uint_x` type always contains an initial 8-bit unsigned integer. If the value is less than or equal to 252, the length is specified in the single byte. If the value is 253, a 16-bit unsigned integer follows. If the value is 254, a 32-bit unsigned integer follows. If the value is 255, a 64-bit unsigned integer follows. For most fields in the Message 4 format, only a value up to `UINT32_MAX` is used. These values are represented by `uint_x`. Values that can have a value up to `UINT64_MAX` are represented with the `uint_x2` type.

## int_x and int_x2

An expanding integer is used for lengths and codes to save space. This type can represent up to a 64-bit integer, although only up to 32-bit integers are used in most places of the Message 4 format. The `int_x` type always contains an initial 8-bit signed integer. If the value is less than or equal to 124, the length is specified in the single signed byte. If the value is 125, a 16-bit signed integer follows. If the value is 126, a 32-bit signed integer follows. If the value is 127, a 64-bit signed integer follows. For most fields in the Message 4 format, only a value between `INT32_MIN` and `INT32_MAX` is used. These values are represented by `int_x`. Values that can have a value between `INT64_MIN` and `INT64_MAX` are represented with the `int_x2` type.

## MetaData

MetaData is a string available for future use. Additional fields in the MetaData is stored one field per line, with either a single name, or a name with a value separated by a colon.

Examples fields:

    a_single_name
    a_name_with: a value of any format

## Extended

The Extended header fields are intended to be used to store extensions to the messages that are not specified in this document. These extensions should be enabled using capability codes exchanged during CreateConnection of a transport connection.

The Extended header field format consists of zero, one, or more extended entries. Each entry has the following format:

| Description          | Type   | Bytes  | Description |
| ---                  | ---    | ---    | ---         |
| ExtendedEntryLen     | uint_x  | 1-5   | The number of bytes in the extended entry, including the length code |
| ExtendedEntryType    | uint_x  | 1-5   | A code representing the type of the extended entry. |
| ExtendedEntryData    | uint8   | varies | The data as defined by the extensions specification. All data shall be little endian (except for UUID) |

ExtendedEntryType 252 is reserved for vendor-specific extensions. The first 16 bytes of ExtendedEntryData shall be a UUID representing the vendor-specific extension type.

ExtendedEntryTypes with bit 31 set (0x80000000) shall be reserved for experimental use.

## UUID

UUIDs are 16 byte unique identifies. UUID are always stored in "big endian" binary format. (Not Microsoft GUID format)

## Transport Capability Codes

The following codes are used to enable Message Version 4 serialization when exchanged during the CreateConnection handshake:

Page Codes:

| Page Code   | Name                | Notes                                |
| ---         | ---                 | ---                                  |
| 0x04000000  | MESSAGE4_BASIC_PAGE | Message Version 4 Basic Capabilities Page |

Page Flags: (or'd together with page and flags on page)

| Page Flag   | Name                           | Notes                                |
| ---         | ---                            | ---                                  |
| 0x00000001  | MESSAGE4_BASIC_ENABLE          | Enable Message Version 4 |
| 0x00000002  | MESSAGE4_BASIC_CONNECTCOMBINED | Enable use of EntryType ConnectClientCombined (121) with Message Version 4 |

## String Table Codes

Robot Raconteur uses strings in headers for flexbility. This has the downside of being inefficient with bandwidth due to the repeated use of strings instead of codes. To help improve the efficiency of version 4 messages, commons strings may be replaced with a code. This conversion between string and code is done at the transport connection stream level. Transports replace strings with codes right before serialization, and convert back to strings directly after serialization. The specific codes used for conversion are specific to each transport connection stream.

These codes are defined in "String Tables". These "String Tables" contain a table of strings, and a `uint32` code that represents the string. String tables can be predifined and loaded by communicating nodes, or can be local to the message. Predefined string tables are enabled using capability codes exchanged during the CreateConnection handshake (see below). Currently only the "default" table is available. This default table is defined later in this document.

Local string tables are used to avoid repeating the same string within the same message. This is to avoid a situation where an array repeats the same string thousands of times. The message header stores one copy of the string, with a message local code for the string.

The following bits are used as flags to define the scope of the code:

| Bit | Description |
| 0x1 | 1 for local message, 0 for transport stream table |
| 0x2 | Reserved, must be 0 |
| 0x80000000| Vendor specific |

The following codes are used to enable Message Version 4 string table features when exchanged during the CreateConnection handshake:

Page Codes:

| Page Code   | Name                | Notes                                |
| ---         | ---                 | ---                                  |
| 0x04100000  | MESSAGE4_STRINGTABLE_PAGE | Message Version 4 String Table Page |

Page Flags: (or'd together with page and flags on page)

| Page Flag   | Name                           | Notes                                |
| ---         | ---                            | ---                                  |
| 0x00000001  | MESSAGE4_STRINGTABLE_ENABLE          | Enable Message Version 4 String Table |
| 0x00000002  | MESSAGE4_STRINGTABLE_MESSAGE_LOCAL | Enable Message Version 4 local string table |
| 0x00000004  | MESSAGE4_STRINGTABLE_STANDARD_TABLE | Enable Message Version 4 standard predefined table |

## Header Flags

Flags are used to enable and disable different fields within a message. Since fields are often empty or contain redundent information, disabling fields can significantly reduce the amount of data sent between nodes. Flags are specified for each header in a table. The fields are specified in another table, with the matching flag specified, or no flag if the header field is always enabled. If the flag is not set, the field should be skipped and not encoded into the binary stream.

## Common Message Header

Messages of all versions must start with the common header:

| Field Name           | Type   | Bytes  | Description |
| ---                  | ---    | ---    | ---         |
| "RRAC"               | utf-8  | 4      | Magic at start of message. Must be "RRAC" { 0x52, 0x52, 0x41, 0x43 } |
| MessageSize          | uint32 | 4      | Total bytes in message including header |
| MessageVersion       | uint16 | 2      | The version of the serialized message that follows |

## Message

The message structure is first, before message elements and message entries. The MessageSize field contains the total size of the message in bytes.

The message header flags are defined as:

| Flag (uint8) | Name | Description |
| ---          | ---  | ---         |
| 0x01 | ROUTING_INFO | Include routing information including NodeIDs and NodeNames |
| 0x02 | ENDPOINT_INFO | Include endpoints information |
| 0x04 | PRIORITY | Include message priority |
| 0x08 | UNRELIABLE | Mark the message as unreliable, not requiring reliable delivery |
| 0x10 | META_INFO | Include MetaData and MessageID fields |
| 0x20 | STRING_TABLE | Include message local string table |
| 0x40 | MULTIPLE_ENTRIES | Message contains more than one entry. If unset, assume one entry follows |
| 0x80 | EXTENDED | Include Extended field |


The message header is defined as:

| Field Name           | Type   | Bytes  | Flag | Description |
| ---                  | ---    | ---    | ---  | ---         |
| "RRAC"               | utf-8  | 4      |      | Magic at start of message. Must be "RRAC" { 0x52, 0x52, 0x41, 0x43 } |
| MessageSize          | uint32 | 4      |      | Total bytes in message including header |
| MessageVersion       | uint16 | 2      |      | Version of message serialization format. Always 4 for Message4 |
| HeaderLen            | uint_x | 1-5    |      | Size of the message header in bytes before entries begin |
| MessageFlags         | uint8  | 1      |      | Flags for the message header |
| SenderNodeID         | UUID   | 16     | ROUTING_INFO     | The NodeID of the sender as a UUID |
| ReceiverNodeID       | UUID   | 16     | ROUTING_INFO     | The NodeID of the intended receiver as a UUID |
| SenderNodeName_len   | uint_x | 1-5    | ROUTING_INFO     | The length of the NodeName of the sender in bytes after utf-8 encoding |
| SenderNodeName       | utf-8  | varies | ROUTING_INFO     | The NodeName of the sender encoded as utf-8 string |
| ReceiverNodeName_len | uint_x | 1-5    | ROUTING_INFO     | The length of the NodeName of the intended receiver in bytes after utf-8 encoding |
| ReceiverNodeName     | utf-8  | varies | ROUTING_INFO     | The NodeName of the receiver encoded as utf-8 string |
| SenderEndpoint       | uint_x | 1-5    | ENDPOINT_INFO    | The sender endpoint number |
| ReceiverEndpoint     | uint_x | 1-5    | ENDPOINT_INFO    | The receiver endpoint number |
| Priority             | uint16 | 2      | PRIORITY      | The priority of the message |
| MetaData_len         | uint_x | 1-5    | META_INFO     | The length of MetaData in bytes after utf-8 encoding |
| MetaData             | utf-8  | varies | META_INFO     | MetaData encoded as utf-8 string |
| MessageID            | uint16 | 2      | META_INFO     | *Unused* |
| MessageResID         | int16  | 2      | META_INFO     | *Unused* |
| StringTableCount     | uint_x | 1-5    | STRING_TABLE | Number of string table entries |
| StringTableData      | varies | varies | STRING_TABLE |String table entries, uint_x code, followed by uint_x len, followed uy char8 data. Must be less than 1024 bytes. |
| EntryCount           | uint_x | 1-5    | MULTIPLE_ENTRIES  | The number of message entries following header. Assume 1 if not included |
| Extended_len         | uint_x | 1-5    | EXTENDED | Length of extended headers in bytes |
| Extended             | uint8  | varies | EXTENDED | Extended headers. Shall use little endian. |

## MessageEntry

A message entry contains a request, response, or packet. The standard usage of entries is discussed in [Robot Raconteur Object Protocol Standard](object_protocol.md).

The message entry flags are defined as:

| Flag (uint8) | Name | Description |
| ---          | ---  | ---         |
| 0x01 | SERVICE_PATH_STR | Service path is included as string |
| 0x02 | SERVICE_PATH_CODE | Service path is included as a code |
| 0x04 | MEMBER_NAME_STR | Member name is included as string |
| 0x08 | MEMBER_NAME_CODE | Member name is include as code |
| 0x10 | REQUEST_ID | RequestID is included in entry |
| 0x20 | ERROR | Error has occured, code included |
| 0x40 | META_INFO | Meta info is included |
| 0x80 | EXTENDED | Include Extended field |

The message entry header is defined as:

| Field Name           | Type   | Bytes  | Flag | Description |
| ---                  | ---    | ---    | ---  | ---         |
| EntrySize            | uint_x | 1-5    |      | The total size of the entry in bytes, including message elements |
| EntryFlags           | uint8  | 1      |      | Flags for message entry |
| EntryType            | uint16 | 2      |      | The type of the entry (request, response, or packet type) |
| ServicePath_len      | uint_x | 1-5    | SERVICE_PATH_STR | The length of ServicePath in bytes after utf-8 encoding |
| ServicePath          | utf-8  | varies | SERVICE_PATH_STR | The ServicePath encoded as utf-8 string |
| ServicePathCode      | uint_x | 1-5    | SERVICE_PATH_CODE | String table code of the service path |
| MemberName_len       | uint_x | 1-5    | MEMBER_NAME_STR   | The length of MemberName in bytes after utf-8 encoding |
| MemberName           | utf-8  | varies | MEMBER_NAME_STR   | The MemberName encoded as utf-8 string |
| MemberNameCode       | uint_x | 1-5    | MEMBER_NAME_CODE  | String table code of the member name |
| RequestID            | uint_x | 1-5    | REQUEST_ID | The RequestID to match up request and response messages |
| Error                | uint16 | 2      | ERROR | The error code, or zero for no error |
| MetaData_len         | uint_x | 1-5    | META_INFO | The length of MetaData in bytes after utf-8 encoding |
| MetaData             | utf-8  | varies | META_INFO | MetaData encoded as utf-8 string |
| Extended_len         | uint_x | 1-5    | EXTENDED | Length of extended headers in bytes |
| Extended             | uint8  | varies | EXTENDED | Extended headers. Shall use little endian. |
| ElementCount         | uint_x | 1-5    |      | The number of *top level* elements that follow |

*Note that ElementCount only refers to the top-level elements that follow. Nested elements are not counted in the ElementCount field.*

## MessageElement

Message Elements contain values. The standard usage of message entries is discussed in [Robot Raconteur Value Types Standard](value_types.md).

Message Version 4 introduces "ElementNumber". This is identical to "ElementName", but encoded as a number to save space when "ElementName" represents a number.

The message element flags are defined as:

| Flag (uint8) | Name | Description |
| ---          | ---  | ---         |
| 0x01 | ELEMENT_NAME_STR | Element name is included as string |
| 0x02 | ELEMENT_NAME_CODE | Element name is included as a code |
| 0x04 | ELEMENT_NUMBER | Element number is included |
| 0x08 | ELEMENT_TYPE_NAME_STR | Element type name is included as a string |
| 0x10 | ELEMENT_TYPE_NAME_CODE | ELement type name is included as a code |
| 0x20 | META_INFO | Meta info is included |
| 0x40 | RESERVED | Reserved |
| 0x80 | EXTENDED | Include Extended field |

The message element header is defined as:

| Field Name           | Type   | Bytes  | Flag | Description |
| ---                  | ---    | ---    | ---  | ---         |
| ElementSize          | uint_x | 1-5    |      | The total size of the entry in bytes, including array data or nested message elements |
| ElementFlags         | uint8  | 1      |      | Flags for the element |
| ElementName_len      | uint_x | 1-5    | ELEMENT_NAME_STR | The length of ElementName in bytes after utf-8 encoding |
| ElementName          | utf-8  | varies | ELEMENT_NAME_STR | The ElementName encoded as utf-8 string |
| ElementNameCode      | uint_x | 1-5    | ELEMENT_NAME_CODE | String table code for ElementName |
| ElementNumber        | uint_x | 1-5    | ELEMENT_NUMBER | The number of the element |
| ElementType          | uint16 | 2      |     | Data type code for the data |
| ElementTypeName_len  | uint_x | 1-5    | ELEMENT_TYPE_NAME_STR | The length of ElementTypeName_len in bytes after utf-8 encoding |
| ElementTypeName      | utf-8  | varies | ELEMENT_TYPE_NAME_STR | The ElementTypeName encoded as utf-8 string |
| ElementTypeNameCode  | uint_x | 1-5    | ELEMENT_TYPE_NAME_CODE | String table code for ElementTypeName |
| MetaData_len         | uint_x | 1-5    | META_INFO | The length of MetaData in bytes after utf-8 encoding |
| MetaData             | utf-8  | varies | META_INFO | MetaData encoded as utf-8 string |
| Extended_len         | uint_x | 1-5    | EXTENDED | Length of extended headers in bytes |
| Extended             | uint8  | varies | EXTENDED | Extended headers. Shall use little endian. |
| DataCount            | uint_x | 1-5    |    | The number of array elements or *top level* elements that follow |

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

## Default string table

| Code | String |
| ---  | ---    |
| 0 | (empty string) |
| 4 | array |
| 8 | attributes |
| 12 | AuthenticateUser |
| 16 | capabilities |
| 20 | capabilities2 |
| 24 | clientversion |
| 28 | confirmcodes |
| 32 | Continue |
| 36 | count |
| 40 | CreateConnection |
| 44 | credentials |
| 48 | data |
| 52 | DimCount |
| 56 | Dimensions |
| 60 | dims |
| 64 | errorname |
| 68 | errorparam |
| 72 | errorstring |
| 76 | errorsubname |
| 80 | extraimports |
| 84 | false |
| 88 | GetRemoteNodeID |
| 92 | index |
| 96 | Length |
| 100 | LogoutUser |
| 104 | MaxTransferSize |
| 108 | memorypos |
| 112 | messageversion |
| 116 | MonitorContinueEnter |
| 120 | MonitorEnter |
| 124 | MonitorExit |
| 128 | mutualauth |
| 132 | nanoseconds |
| 136 | nodeid |
| 140 | nodeid |
| 144 | nodename |
| 148 | nolock |
| 152 | nolockread |
| 156 | null |
| 160 | objectimplements |
| 164 | objecttype |
| 168 | OK |
| 172 | packet |
| 176 | packetnumber |
| 180 | packettime |
| 184 | parameter |
| 188 | password |
| 192 | pause |
| 196 | perclient |
| 200 | readonly |
| 204 | ReleaseClientObjectLock |
| 208 | ReleaseObjectLock |
| 212 | requestack |
| 216 | RequestClientObjectLock |
| 220 | RequestObjectLock |
| 224 | resume |
| 228 | return |
| 232 | returnservicedefs |
| 236 | robotraconteur |
| 240 | RobotRaconteur |
| 244 | RobotRaconteur.TimeSpec |
| 248 | seconds |
| 252 | seqno |
| 256 | service |
| 260 | servicedef |
| 264 | servicedefs |
| 268 | ServiceIndex |
| 272 | servicename |
| 276 | servicepath |
| 280 | ServiceType |
| 284 | stringtable |
| 288 | timeout |
| 292 | timespec |
| 296 | timestamp |
| 300 | true |
| 304 | unreliable |
| 308 | urgent |
| 512 | username |
| 516 | value |
| 520 | writeonly |
| 524 | Attributes |
| 528 | ConnectionURL |
| 532 | GetDetectedNodes |
| 536 | GetLocalNodeServices |
| 540 | GetRoutedNodes |
| 544 | LocalNodeServicesChanged |
| 548 | Name |
| 552 | NodeID |
| 556 | NodeInfo |
| 560 | NodeName |
| 564 | RobotRaconteurServiceIndex |
| 568 | RobotRaconteurServiceIndex.NodeInfo |
| 572 | RobotRaconteurServiceIndex.ServiceIndex |
| 576 | RobotRaconteurServiceIndex.ServiceInfo |
| 580 | RootObjectImplements |
| 584 | RootObjectType |
| 588 | ServiceIndex |
| 592 | ServiceIndexConnectionURL |
| 596 | ServiceInfo |
| 600 | node |
| 604 | level |
| 608 | component |
| 612 | componentname |
| 616 | componentobjectid |
| 620 | endpoint |
| 624 | member |
| 628 | message |
| 632 | time |
| 636 | sourcefile |
| 640 | sourceline |
| 644 | threadid |
| 648 | fiberid |

