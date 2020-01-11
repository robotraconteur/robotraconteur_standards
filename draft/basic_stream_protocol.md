<p align="center"><img src="../images/RRheader2.jpg"></p>

# Robot Raconteur Basic Stream Protocol Standard

http://robotraconteur.com

http://github.com/robotraconteur

Copyright &copy; 2020 Wason Technology, LLC

*Robot Raconteur is a communication framework designed for use with robotics, automation, building control, and internet of things applications.*

## Abstract

Robot Raconteur communicates between nodes using messages. These messages can be serialized to binary form for transmission over a network, interprocess, intraprocess, or other communication method. The most common form of communication is a "stream", implemented by a socket or pipe. Streams are FIFO connections between two nodes, where bytes supplied to the stream arrive reliably at the peer node. All existing transports operate as streams, with some transports providing auxillary options for performance critical aspects of the protocol, for example wire packet transmission. A basic stream protocol is defined for use by all stream transports, with additional capabilities being defined for each transport type.

## Introduction

This specification defines a basic stream protocol that is used as the basis for all stream protocol implementation. In the core C++ library, this is implemented by the ASIOStreamBaseTransport class.

Streams are always unicast, with one stream in use for one pair of endpoints.

## Message Transmission

**The stream MUST be reliable for messages that have reliable QoS**

Messages are serialized and sent to the peer one after the other, in order, with no interleaving. By default Message Version 2 is used. See [Robot Raconteur Message Version 2 Serialization Format](message_v2.md). Other message versions can be negotiated, but Message Version 2 **must** be supported.

No specific timing or congestion control is specified. It is assumed the underlying stream will manage these considerations.

## StreamOp Requests

StreamOp requests are "Stream Operations" that allow the stream to communicate. Only one StreamOp per direction may be in flight at any given time. The routing information is ignored (except for GetNodeID). The RequestID field is ignored. Only one entry per message is allowed for StreamOp.

The following StreamOp are defined:

#### StreamOp(CreateConnection)

StreamOp(CreateConnection) establishes a stream connection. The client sends the request, and the service responds. The client should include the receiver routing information if available, with the sender routing information optionally blank. The service must respond with filled in routing information.

##### Request:

Direction: Client to Service

* EntryType: 1
* ServicePath: *empty*
* MemberName: "CreateConnection"
* RequestID: 0
* Request QoS: reliable
* Data: Message element list:
  * Element 1- (optional):
    * ElementName: "capabilities"
    * ElementType: 8 (uint32)
    * Data: Array of capabilities codes encoded as uint32 array
  * Element 2- (optional):
    * ElementName: "capabilities2"
    * ElementType: 10 (uint64)
    * Data: Array of capabilities2 codes encoded as uint64 array

##### Response:

Direction: Service to Client

* EntryType: 2
* ServicePath: *empty*
* MemberName: "CreateConnection"
* RequestID: 0
* Request QoS: reliable
* Data: Message element list:
  * Element 1- (optional):
    * ElementName: "capabilities"
    * ElementType: 8 (uint32)
    * Data: Array of capabilities codes encoded as uint32 array
  * Element 2- (optional):
    * ElementName: "capabilities2"
    * ElementType: 10 (uint64)
    * Data: Array of capabilities2 codes encoded as uint64 array

Desired capabilities codes are passed from the client to the service in the optional "capabilities" and "capabilities2" message elements. These fields are uint32 and uint64 data arrays, respectively. The capabilities are broken down into pages, flags, and integers. The usage of these fields is defined below, with other standards adding more pages, codes, and values to indicate support by the transport. The service then responds with which capabilities are enabled. If both the client and service have the same flag, that flag is enabled. If the capability is an integer, the lowest value is used.

Capabilities codes:

Page Code:     Bits 31 - 20
Page Contents: Bits 19 - 0

Page Codes:

| Page Code   | Name                | Notes                                |
| ---         | ---                 | ---                                  |
| 0x02000000  | MESSAGE2_BASIC_PAGE | Message Version 2 Basic Capabilities |

Page Flags: (or'd together with page and flags on page)

| Page Flag   | Name                           | Notes                                |
| ---         | ---                            | ---                                  |
| 0x00000001  | MESSAGE2_BASIC_ENABLE          | Enable Message Version 2. Ignored since Message Version 2 is always enabled |
| 0x00000002  | MESSAGE2_BASIC_CONNECTCOMBINED | Enable use of EntryType ConnectClientCombined (121) with Message Version 2 |

Capabilities2 codes:

Vendor Defined: Bit 63 (1 for vendor defined, 0 for standard)
Page Code:      Bits 62 - 48
Page Contents:  Bits 47 - 0

*No codes defined for capabilities2.*

## StreamCapabilities Requests

The StreamCapabilities request provides another method to determine capabilities between two stream peers. The MemberName field of the request contains the name of a capability, and the response contains a scalar integer indicating the support level for that capability. Zero means no support, while positive integers represent the level of support for that capability. Like the StreamOp request, the RequestID and ServicePath fields are unused. Only one capabilities request may be in flight at a time.

##### Request:

Direction: Client to Service

* EntryType: 3
* ServicePath: *empty*
* MemberName: The capability name
* RequestID: 0
* Request QoS: reliable
* Data: *none*

##### Response:

Direction: Service to Client

* EntryType: 4
* ServicePath: *empty*
* MemberName: Same as request
* RequestID: 0
* Request QoS: reliable
* Data: Message element list:
  * Element 1
    * ElementName: "return"
    * ElementType: 8 (uint32)
    * Data: Scalar uint32 array of the support level for the capability

Currently supported capabilities names:

| Name | Expected Capabilities Levels |
| ---  | ---                          |
| com.robotraconteur.v2 | Stream supports Message Version 2 Serialization Format  (always >1) |
| com.robotraconteur.v2.0 | Stream supports Message Version 2.0 Serialization Format (always >1) |
| com.robotraconteur.v2.minor | The minor revision number of Message Version 2.0 Serialization Format (currently zero) |

## Connection Tests and Timeouts

Streams must use a connection test heartbeat to detect if the stream connection has been lost. Because robotics is highly time sensitive, these heartbeats must be relatively often. If a message is not received within a certain timeout period, the connection must be closed.

By default, the heartbeat parameters are:

Heartbeat Period: 10 seconds
Connection Timeout: 15 seconds

The parameters for connection may vary depending on the transport technology.

Connection test messages, sometimes called ping-pong messages, are sent if no message has been received for the specified heartbeat period. The heartbeat should be sent by the client, and responded by the service. The connection test message entry must be the only entry in the message. Routing information is ignored.

##### Request:

Direction: Client to Service

* EntryType: 111
* ServicePath: *empty*
* MemberName: *empty*
* RequestID: 0
* Request QoS: reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 112
* ServicePath: *empty*
* MemberName: *empty*
* RequestID: 0
* Request QoS: reliable
* Data: *none*

## Connection Establishment

Streams are created based on service URLs. These URLs are discussed in [Robot Raconteur Framework Architecture Standard](framework_architecture.md). The exact contents of the URL will vary depending on the transport technology. Clients must always send the first message. Services sending the first message shall be treated as an error.

The first message sent shall be a StreamOp(CreateConnection) request message.

## Message Routing Information

Since the stream is unicast for the session in use, the routing information is not necessary after the StreamOp(CreateConnection) request is complete. For Message Version 2, the SenderNodeName and ReceiverNodeName should be left blank to save bandwidth. Other message versions have more aggressive routing information compression methods. These are discussed in the standard for each message version.

## Message size limits

By default, messages larger than 10 MB shall cause the stream to immediately terminate. This limit can be adjusted as needed by the user.

## Error Handling

If a protocol error occurs, the stream shall be immediately terminated.

## Stream Shutdown

Stream shutdown is implementation defined. In general, the reception of a DisconnectClient (109) or the transmission DisconnectClientRet (110) message entry shall be considered a signal that the stream should be cleanly terminated. No messages should be transmitted after the transmission of one of these messages.
