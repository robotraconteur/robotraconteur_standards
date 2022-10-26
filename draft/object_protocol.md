<p align="center"><img src="../images/RRheader2.jpg"></p>

# Robot Raconteur Object Protocol Standard

http://robotraconteur.com

http://github.com/robotraconteur

Copyright &copy; 2020 Wason Technology, LLC

*Robot Raconteur is a communication framework designed for use with robotics, automation, building control, and internet of things applications.*

## Abstract

Robot Raconteur separates *value* from *reference* data types. See [Robot Raconteur Framework Architecture Standard](framework_architecture.md) for more details on the distinction between *value* and *reference* types. Objects are always reference types, meaning that they are passed between nodes *by reference*. Messages are sent and from objects by clients to interfacet with *object members*. These messages use the Robot Raconteur *object protocol*, which specifies the format of these messages. This document specifies the *object protocol* used to interact with *service objects* by *reference*.

## Introduction

Robot Raconteur is a Client-Service Remote Procedure Call (RPC) framework, intended for use with robotics and automation applications. See [Robot Raconteur Framework Architecture Standard](framework_architecture.md) for more information on the overall framework architecture. Simplicity is a primary objective of the design of the framework. For this reason, two design simplifications were adopted:

* Strict separation between *value* and *reference* (also called *object*) data types
* *Objects* are always owned by *services*, with *clients* interacting with *objects* by *reference*.

These design choices are in contrast to other RPC technologies such as Java RMI and .NET Remoting, where the distinction between pass-by-reference and pass-by-value types is fuzzier, and garbage collection attempts to cross the boundaries between client and service. The fuzzy distinction between pass-by-reference and pass-by-value significantly complicates other RPC software, and can easily cause memory management errors.

See [Robot Raconteur Value Types Standard](value_types.md) for more information on value types.

*Objects* owned by services are exposed using the Robot Raconteur *object protocol* using object members. Object members receive and send messages to clients. These messages contain the object *service path*, the *entry type*, (often called an *opcode*), and any parameters, stored as *message elements*. Communication between clients and services can either be a *request* or a *packet*. Requests are request-response, meaning that the initiator expects a response. Packets do not expect a response. Individual requests or packets are stored in *message entries*. Messages can carry multiple *message entries*, meaning that an individual message can contain multiple member requests/packets. Packets and requests can be either client-to-service, or service-to-client.

Robot Raconteur uses an Augmented Object-Oriented model with eight member types. See [Robot Raconteur Framework Architecture Standard](framework_architecture.md) for an overview of the member types.

## Service Objects

Objects exposed to clients Robot Raconteur are owned by the service, and are called *service objects*. Each service has a *root object*, which is the object returned to clients when the client connects. Other objects within the service are returned using *objref* members. The *objref* can optionally have an index, allowing it to return a different *service object* depending on the index specified. *Service objects* must form a tree structure, with the *root object* being the root of the tree, and each *objref* being a link.

*Service objects* are owned by the service. Once an object has been returned by an *objref*, it is owned by the service. A *service object* can be *released*, meaning that it is no longer owned by the service, and can no longer be accessed by a client. Clients *should* be notified when a *service object* has been released.

## Service Paths

Objects are uniquely identified within a node using a *ServicePath*. A *ServicePath* is essentially an address within a node. The *globally unique* identifier for an object is the combination of the *NodeID* and *ServicePath* for an object.

A *ServicePath* is a utf-8 encoded string consisting of hierarchical named levels separated by dots. These names must conform to the Robot Raconteur type naming rules as specified in [Robot Raconteur Service Definition Specification](service_definition.md). The first name is the name of the service, as set when the *root object* is registered. The remaining segments correspond to *objref* members. These remaining segments have a name, and optionally an *index* between square brackets after the name, if the objref is an array or map type. The *index* uses a form of URL encoding, where characters other than letters and numbers are replaced with the percent symbol (%) following by the two digit hexadecimal character code. The *ServicePath* must match the following PCRE regex:

    ^[a-zA-Z](?:\w*[a-zA-Z0-9])?(\.[a-zA-Z](?:\w*[a-zA-Z0-9])?(?:\[(?:[a-zA-Z0-9_]|\%[0-9a-fA-F]{2})+\])?)*$

## Message Entries

*Message entries* are used to store requests, responses, or packets for transmission between nodes. See [Robot Raconteur Framework Architecture Standard](framework_architecture.md) for a general overview of messages.

*Message entries* have, at minimum, the following contents:

* The EntryType, a numeric code for the type of entry (also referred to as an opcode)
* The ServicePath, a utf-8 string representing the *ServiceObject* that the entry is associated with
* The MemberName, a utf-8 string representing the member within the *ServiceObject* the entry is associated with
* The RequestID, a number used to match up a response *Message Entry* with the request
* The ErrorType, a numeric code representing an error condition, or zero.
* Requested QoS for the message (assumed reliable unless specified)
* Data and parameters in the form of a list of *Message Elements*

Different implementations or transports may have more fields to hold implementation specific metadata. These extra fields are not considered in this document.

### EntryType

The *EntryType* is a numeric code that identifies the type of the entry. The entry can either be a request, a response, or a packet. In general, odd codes represent requests or packets, while even codes represent responses.

The following *EntryType* codes are defined:

| EntryType | Code | Category | Direction | QoS | Description |
| --- | --- | --- | --- | --- | --- |
| StreamOp | 1 | Request | Either | Reliable |Used internally by transport connections |
| StreamOpRet | 2 | Response | Either | Reliable |StreamOp response |
| StreamCheckCapability | 3 | Request | Either | Reliable | Request the capability level for transport connection feature |
| StreamCheckCapabilityRet | 4 | Response | Either | Reliable | StreamCheckCapability Response |
| StringTableOp | 5 | Request | Either | Reliable | Run-time transport connection string table arbitration request |
| StringTableOpRet | 6 | Response | Either | Reliable | StringTableOp Response |
| GetServiceDesc | 101 | Request | Client to Service | Reliable | Request service definition file text |
| GetServiceDescRet | 102 | Response | Service to Client | Reliable | GetServiceDescRet return |
| ObjectTypeName | 103 | Request | Client to Service | Reliable | Request the type of the object at *ServicePath* |
| ObjectTypeNameRet | 104 | Response | Service to Client | Reliable | ObjectTypeName response |
| ServiceClosed | 105 | Packet | Service to Client | Reliable | Notification that service has been closed |
| ServiceClosedRet | 106 | *reserved* | | |
| ConnectClient | 107 | Request | Reliable | Client to Service | Create a client connection |
| ConnectClientRet | 108 | Response | Reliable | Service to Client | ConnectClient response |
| DisconnectClient | 109 | Request | Reliable | Client to Service | Close a client connection |
| DisconnectClientRet | 110 | Response | Reliable | Service to Client | DisconnectClient response |
| ConnectionTest | 111 | Request | Either | Reliable | Used by transports for heartbeat |
| ConnectionTestRet | 112 | Response | Either | Reliable | ConnectionTest response |
| GetNodeInfo | 113 | Request | Client to Service | Retrieve NodeID and NodeName of server node |
| GetNodeInfoRet | 114 | Response | Service to Client | GetNodeInfo response |
| NodeCheckCapability | 117 | Request | Client to Service | Reliable | Request the capability level for node feature |
| NodeCheckCapabilityRet | 118 | Response | Service to Client | Reliable | NodeCheckCapability response |
| GetServiceAttributes | 119 | Request | Client to Service | Reliable | Retrieve the attributes of a service |
| GetServiceAttributesRet | 120 | Response | Service to Client | Reliable | GetServiceAttributes response |
| ConnectClientCombined | 121 | Request | Client to Service | Reliable | Combine several steps into one to create a client connection |
| ConnectClientCombinedRet | 122 | Response | Service to Client | Reliable | ConnectClientCombined response |
| EndpointCheckCapability | 501 | Request | Client to Service | Reliable | Request the capability level for endpoint feature |
| EndpointCheckCapabilityRet | 502 | Response | Service to Client | Reliable | EndpointCheckCapability response |
| ServiceCheckCapabilityReq | 1101 | Request | Client to Service | Reliable | Request the capability level for service feature |
| NodeCheckCapabilityRet | 1102 | Response | Service to Client | Reliable | ServiceCheckCapability response |
| ClientKeepAliveReq | 1105 | Request | Client to Service | Reliable | Heartbeat packet for client connection to maintain session |
| ClientKeepAliveRet | 1106 | Response | Service to Client | Reliable | ClientKeepAliveReq response |
| ClientSessionOpReq | 1107 | Request | Client to Service | Reliable | Client session operation request |
| ClientSessionOpRet | 1108 | Response | Service to Client | Reliable | ClientSessionOpReq response |
| ServicePathReleasedReq | 1109 | Packet | Service to Client | Reliable | Notification that a service path object has been released |
| ServicePathReleasedRet | 1110 | *reserved* | | |
| PropertyGetReq | 1111 | Request | Client to Service | Reliable | Get the value of a property member |
| PropertyGetRes | 1112 | Response | Service to Client | Reliable | PropertyGetReq response |
| PropertySetReq | 1113 | Request | Client to Service | Reliable | Set the value of a property member |
| PropertySetRes | 1114 | Response | Service to Client | Reliable | PropertySetSeq response |
| FunctionCallReq | 1121 | Request | Client to Service | Reliable | Call a function member |
| FunctionCallRes | 1122 | Response | Service to Client | Reliable | FunctionCallReq response |
| GeneratorNextReq | 1123 | Request | Client to Service | Reliable | Call *Next* on a function generator |
| GeneratorNextRes | 1124 | Response | Service to Client | Reliable | GeneratorNextRet response |
| EventReq | 1131 | Packet | Service to Client | Reliable | Event packet sent from service to client |
| EventRes | 1132 | *reserved* | | |
| PipePacket | 1141 | Packet | Either | Reliable or Unreliable | Pipe packet |
| PipePacketRet | 1142 | *reserved* | | |
| PipeConnectReq | 1143 | Request | Client to Service | Reliable | Create indexed pipe connection |
| PipeConnectRet | 1144 | Response | Service to Client | Reliable | PipeConnectRet response |
| PipeDisconnectReq | 1145 | Request | Client to Service | Reliable | Close indexed pipe connection |
| PipeDisconnectRet | 1146 | Response | Service to Client | Reliable | PipeDisconnectReq response |
| PipeClosed | 1147 | Packet | Service to Client | Reliable | Notification that the service has closed a pipe connection |
| PipeClosedRet | 1148 | *reserved* | | |
| CallbackCallReq | 1151 | Request | Service to Client | Reliable | Call a callback member on a specific client |
| CallbackCallRet | 1152 | Response | Client to Service | Reliable | CallbackCallReq response |
| WirePacket | 1161 | Packet | Either | Unreliable | Wire value packet |
| WirePacketRet | 1162 | *reserved* | | |
| WireConnectReq | 1163 | Request | Client to Service | Reliable | Create wire connection |
| WireConnectRet | 1164 | Response | Service to Client | Reliable | WireConnectRet response |
| WireDisconnectReq | 1165 | Request | Client to Service | Reliable | Close wire connection |
| WireDisconnectRet | 1166 | Response | Service to Client | Reliable | WireDisconnectReq response |
| WireClosed | 1167 | Packet | Service to Client | Reliable | Notification that the service has closed a wire connection |
| WireClosedRet | 1168 | *reserved* | | |
| MemoryRead | 1171 | Request | Client to Service | Reliable | Read memory request |
| MemoryReadRet | 1172 | Response | Service to Client | Reliable | MemoryReadRet response |
| MemoryWrite | 1173 | Request | Client to Service | Reliable | Write memory request |
| MemoryWriteRet | 1174 | Response | Service to Client | Reliable | MemoryWriteRet response |
| MemoryGetParam | 1175 | Request | Client to Service | Reliable | Read memory parameter request |
| MemoryGetParamRet | 1176 | Response | Service to Client | Reliable | MemoryGetParam response |
| WirePeekInValueReq | 1181 | Request | Client to Service | Reliable | Get the in value of a wire member |
| WirePeekInValueRet | 1182 | Response | Service to Client | Reliable | WirePeekInValueReq response |
| WirePeekOutValueReq | 1183 | Request | Client to Service | Reliable | Get the out value of a wire member |
| WirePeekOutValueRet | 1184 | Response | Service to Client | Reliable | WirePeekOutValueReq response |
| WirePokeOutValueReq | 1185 | Request | Client to Service | Reliable | Set the out value of a wire |
| WirePokeOutValueRet | 1186 | Response | Service to Client | Reliable | WirePokeOutValueReq response |

Even EntryType codes following packet EntryType codes are always reserved.

### ServicePath

The *ServicePath* points to the *object* to which the *message entry* is addressed. See the [Service Paths](#Service-Paths) section for more information.

### MemberName

The *MemberName* points to the member to which entry is address, or names a command. The exact usage depends on the *EntryType*.

### RequestID

The *RequestID* is used to match up requests with responses. For example, the function call requestor will assign the request a *RequestID*, and the responder will use this *RequestID* in the response. The requestor can then use this ID to determine which incoming packet responds a specific request. The *RequestID* is arbitrarily assigned by the client, and has no universal meaning outside of the requestor.

### ErrorType

The *ErrorType* field contains a numeric code (normally uint16) specifying the type of error that has occurred. Zero means no error, while any non-zero code represents and error. See [Errors](#Errors) for a table of defined ErrorType codes, and how the message entry contents are modified for an error.

### Data

Data and/or parameters are stored as a list of *message elements*. The format of these *message elements* is discussed in [Robot Raconteur Value Types Specifications](value_types.md). Message elements that do not have a required order are marked with a dash next to their index, such as 1-.

## Transport Messages

Transport messages have *ElementType* codes between 1-100. They are strictly for internal use by transports, and never reach the nodes or services.

## Special Requests

*Special Requests* are used to initialize, teardown, and manage connections between nodes. They are also used to query information. Special requests are "special" because they do not require an endpoint on the service side; the response is immediately returned to the sender. This behavior is used since the requests operate before an endpoint connection is established. Special requests have *ElementType* codes between 101 and 500.

Special requests must always be the only entry in a message.

### GetServiceDesc

*GetServiceDesc* requests a the full text of service definitions from a node. Full text of the service definition is returned, so that clients can parse at run-time and generate proxies dynamically. *GetServiceDesc* can operate in two modes: return service definitions based on a *ServicePath*, or return based on a service definition name. *ServicePath* must point to a root object.

#### GetServiceDesc using ServicePath

Return the full text of a service definition selected by a *ServicePath*. Return the service definition of the type of the service. This form will also return the attributes of the service.

##### Request:

Direction: Client to Service

* EntryType: 101
* ServicePath: Name of service to query service definition
* MemberName: *empty*
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "clientversion"
    * ElementType: 11 (string)
    * Data: Robot Raconteur standard version as utf-8 string, with three decimal components separated by dots (ie "0.9.2")

##### Response:

Direction: Service to Client

* EntryType: 102
* ServicePath: Same as request
* MemberName: *empty*
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1-:
    * ElementName: "servicedef"
    * ElementType: 11 (string)
    * Data: Service definition full text as utf-8 string
  * Element 2-: (optional)
    * ElementName: "attributes"
    * ElementType: 103 (varvalue{string})
    * Data: Service attributes packed as varvalue{string}
  * Element 3-: (optional)
    * ElementName: "extraimports"
    * ElementType: 108 (string{list})
    * Data: List of extra service definition names as utf-8 strings that may be used with `varvalue` types.

**attributes must not use any named types.**

#### GetServiceDesc by Service Definition Name

Return the full text of a service definition by name. Return the service definition with the name specified in the *ServiceType* parameter.

##### Request:

Direction: Client to Service

* EntryType: 101
* ServicePath: *unspecified*
* MemberName: *empty*
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1-:
    * ElementName: "ServiceType"
    * ElementType: 11 (string)
    * Data: The name of the requested service definition as a utf-8 string
  * Element 2-:
    * ElementName: "clientversion"
    * ElementType: 11 (string)
    * Data: Robot Raconteur standard version as utf-8 string, with three decimal components separated by dots (ie "0.9.2")

##### Response:

Direction: Service to Client

* EntryType: 102
* ServicePath: Same as request
* MemberName: *empty*
* RequestID: Same as request
* Requested QoS: Reliable
  * Element 1:
    * ElementName: "servicedef"
    * ElementType: 11 (string)
    * Data: Service definition full text as string

### ObjectTypeName

Returns the fully qualified type of the object at a specified service path. The service shall compare the clientversion specified by the client, and compare it to the version of the service definition of the object type. If the service definition requires a newer version, the service shall search, in order, the implemented object types to see if it is of an equal or lesser version compared to the clientversion. The first matching implemented object type shall be returned. If no types are of a compatible version, a ServiceError shall be raised.

##### Request:

Direction: Client to Service

* EntryType: 103
* ServicePath: Service path to object of interest
* MemberName: *empty*
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "clientversion"
    * ElementType: 11 (string)
    * Data: Robot Raconteur standard version as utf-8 string, with three decimal components separated by dots (ie "0.9.2")

##### Response:

Direction: Service to Client

* EntryType: 104
* ServicePath: Same as request
* MemberName: *empty*
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1-:
    * ElementName: "objecttype"
    * ElementType: 11 (string)
    * Data: Fully qualified object type name as utf-8 string
  * Element 2-: (optional)
    * ElementName: "objectimplements"
    * ElementType: 108 (string{list})
    * Data: List of fully qualified type names implemented by object as utf-8 strings
  
### ServiceClosed

Notification from services to clients that a service is no longer available. This information should be sent to all clients connected to the service.

##### Packet:

Direction: Service to Client

* EntryType: 105
* ServicePath: Name of service that has been closed
* MemberName: *empty*
* RequestID: 0
* Requested QoS: Reliable
* Data: *empty*

### ConnectClient

Connects a client to a service. This operation creates a *ServerEndpoint* on the server node, and returns the new EndpointID as part of the return message header. After the client has been connected, the client can communicate with service members. Before the client is connected, only *special requests* may be used. All requests to the service must utilize the assigned EndpointID in the message header.

This request is executed *after* the transport connection has been created to the remote node.

##### Request:

Direction: Client to Service

* EntryType: 107
* ServicePath: Name of the service to connect
* MemberName: *empty*
* RequestID: Assigned by client
* Request QoS: Reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 108
* ServicePath: Same as request
* MemberName: *empty*
* RequestID: Same as request
* Request QoS: Reliable
* Data: *empty*

### DisconnectClient

Closes a client connection. This operation will normally result in the service closing the transport connection after sending the response.

##### Request:

Direction: Client to Service

* EntryType: 109
* ServicePath: Name of the service previously connected connected
* MemberName: *empty*
* RequestID: Assigned by client
* Request QoS: Reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 110
* ServicePath: Same as request
* MemberName: *empty*
* RequestID: Same as request
* Request QoS: Reliable
* Data: *empty*

### ConnectionTest

Used by transports as a heartbeat to keep connection alive and detect if a connection has failed. If no message or heartbeat is detected after a certain timeout, the connection is closed. Typical heartbeat periods are 5 seconds, with 15 second timeouts. Heartbeats may be initiated by either the client or server, but are typically initiated by the client.

##### Request:

Direction: Either

* EntryType: 111
* ServicePath: *empty*
* MemberName: *empty*
* RequestID: 0
* Request QoS: Rate limit
* Data: *empty*

##### Response:

Direction: Either

* EntryType: 111
* ServicePath: *empty*
* MemberName: *empty*
* RequestID: 0
* Request QoS: Rate limit
* Data: *empty*

### GetNodeInfo

Request the NodeID and NodeName of the server node. Information is returned in the message header fields.

##### Request:

Direction: Client to Service

* EntryType: 113
* ServicePath: *empty*
* MemberName: *empty*
* RequestID: Assigned by client
* Request QoS: Reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 114
* ServicePath: *empty*
* MemberName: *empty*
* RequestID: Same as request
* Request QoS: Reliable
* Data: *empty*

### NodeCheckCapability

Return the capability level of a node feature.

##### Request:

Direction: Client to Service

* EntryType: 117
* ServicePath: *empty*
* MemberName: The name of the capability
* RequestID: Assigned by client
* Request QoS: Reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 118
* ServicePath: *empty*
* MemberName: Same as request
* RequestID: Same as request
* Request QoS: Reliable
* Data: Message element list
  * Element 1:
    * ElementName: "return"
    * ElementType: 7 (int32)
    * Data: The capability level as a scalar int32

### GetServiceAttributes

Return the attributes of a service.

##### Request:

Direction: Client to Service

* EntryType: 119
* ServicePath: Name of service to retrieve attributes from
* MemberName: *empty*
* RequestID: Assigned by client
* Request QoS: Reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 120
* ServicePath: Same as request
* MemberName: *empty*
* RequestID: Same as request
* Request QoS: Reliable
* Data: Message element list
  * Element 1:
    * ElementName: "return"
    * ElementType: 103 (varvalue{string})
    * Data: Service attributes packed as varvalue{string}

**attributes must not use any named types.**

### ConnectClientCombined

ConnectClientCombined executes GetServiceDesc, ObjectTypeName, ConnectClient, and ClientSessionOp(AuthenticateUser) in one operation.

Since: version 0.9.0

##### Request:

Direction: Client to Service

* EntryType: 121
* ServicePath: Name of service to connect to
* MemberName: *empty*
* RequestID: Assigned by client
* Request QoS: Reliable
* Data: Message element list
  * Element 1-:
    * ElementName: "clientversion"
    * ElementType: 11 (string)
    * Data: Robot Raconteur standard version as utf-8 string, with three decimal components separated by dots (ie "0.9.2")
  * Element 2-:
    * ElementName: "returnservicedefs"
    * ElementType: 11 (string)
    * Data: "true" to return service definitions, otherwise "false"
  * Element 3- (optional):
    * ElementName: "username"
    * ElementType: 11 (string)
    * Data: The username for authentication as utf-8 string
  * Element 4- (optional):
    * ElementName: "credentials"
    * ElementType: 103 (varvalue{string})
    * Data: The credentials for authentication packed as a varvalue{string}

##### Response:

Direction: Service to Client

* EntryType: 122
* ServicePath: Same as request
* MemberName: *empty*
* RequestID: Same as request
* Request QoS: Reliable
* Data: Message element list
  * Element 1-:
    * ElementName: "objecttype"
    * ElementType: 11 (string)
    * Data: Fully qualified object type name of the root object as utf-8 string
  * Element 2-: (optional)
    * ElementName: "objectimplements"
    * ElementType: 108 (string{list})
    * Data: List of fully qualified type names implemented by object as utf-8 strings
  * Element 3- (optional, returned if "requestservicedefs" is "true"):
    * ElementName: "servicedefs"
    * ElementType: 108 (string{list})
    * Data: List of service definition full texts as utf-8 string.

## Endpoint Requests

Requests directed at endpoints, but not objects.

### EndpointCheckCapability

Check the capability level of an endpoint feature. Must be called after connection is created and endpoint is assigned.

##### Request:

Direction: Client to Service

* EntryType: 501
* ServicePath: *empty*
* MemberName: The name of the capability
* RequestID: Assigned by client
* Request QoS: Reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 502
* ServicePath: *empty*
* MemberName: Same as request
* RequestID: Same as request
* Request QoS: Reliable
* Data: Message element list
  * Element 1:
    * ElementName: "return"
    * ElementType: 7 (int32)
    * Data: The capability level as a scalar int32

## Client Connection Management Requests

The following requests are used to manage a connection after it has been created using a *special request*.

### ServiceCheckCapability

Check the capability level of an service feature. Must be called after client connection is established.

##### Request:

Direction: Client to Service

* EntryType: 1101
* ServicePath: The name of the service
* MemberName: The name of the capability
* RequestID: Assigned by client
* Request QoS: Reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 1102
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Request QoS: Reliable
* Data: Message element list
  * Element 1:
    * ElementName: "return"
    * ElementType: 7 (int32)
    * Data: The capability level as a scalar int32

### ClientKeepAlive

The ClientKeepAlive request is used as a heartbeat to keep the client connection session alive. It is also used to detect if the connection has been lost. This request is typically issued if no message has been sent from the client for one minute.

##### Request:

Direction: Client to Service

* EntryType: 1105
* ServicePath: The name of the service
* MemberName: *empty*
* RequestID: Assigned by client
* Request QoS: Reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 1106
* ServicePath: Same as request
* MemberName: *empty*
* RequestID: Same as request
* Request QoS: Reliable
* Data: *empty*

### ClientSessionOp

ClientSessionOp is used to request a session level operation. Currently this operation is used to handle authentication, object locking, and monitor locks.

#### Authentication

Authentication is implemented using a Security Policy that specifies the permissions for clients in the service. Clients authenticate using usernames and credentials, where credentials is a varvalue{string} type. Typically credentials contains a "password" entry with a string value, but it can also be used to carry other forms of security tokens. The permissions allowed to each client is dependent on the security policy.

#### ClientSessionOp(AuthenticateUser)

Authenticate the user. The *ServicePath* is the name of the service. The authentication is associated with the endpoint connected by the client.

##### Request:

Direction: Client to Service

* EntryType: 1107
* ServicePath: Name of service
* MemberName: "AuthenticateUser"
* RequestID: Assigned by client
* Request QoS: reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "username"
    * ElementType: 11 (string)
    * Data: Username as utf-8 string
  * Element 2:
    * ElementName: "credentials"
    * ElementType: 103 (varvalue{string})
    * Data: Credentials packed as varvalue{string}

##### Response:

Direction: Service to Client

* EntryType: 1108
* ServicePath: Same as request
* MemberName: "AuthenticateUser"
* RequestID: Same as request
* Request QoS: reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "return"
    * ElementType: 11 (string)
    * Data: "OK" as utf-8 string

#### ClientSessionOp(LogoutUser)

Logs out a user. Note that this does not destroy the client connection, it only cancels the privileges granted to the client. Clients are automatically logged out when a client connection is closed.

##### Request:

Direction: Client to Service

* EntryType: 1107
* ServicePath: Name of service
* MemberName: "LogoutUser"
* RequestID: Assigned by client
* Request QoS: reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "username"
    * ElementType: 11 (string)
    * Data: Username as utf-8 string

##### Response:

Direction: Service to Client

* EntryType: 1108
* ServicePath: Same as request
* MemberName: "LogoutUser"
* RequestID: Same as request
* Request QoS: reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "return"
    * ElementType: 11 (string)
    * Data: "OK" as utf-8 string

### Object Locking

Objects can be locked to prevent other users or other sessions from accessing an object's members. When an object is locked, the specified object and all child objects in the service object tree are locked. Locks can either be "user", meaning that only the current logged in user can access the object, from any active session by that user. A "user session" lock will lock the specified object and all child objects to the current session. This will prevent all other sessions from accessing the object's members, even with the same username. In both cases, anonymous users are blocked by object locks. If an attempt is made to access a locked object without the current user/session, an ObjectLockedError is raised.

There are situations where it is still necessary to allow other users to access members, even when object locked. For this purpose, the *nolock* and *nolockread* modifiers can be applied to object members. The *nolock* modifier allows full access to the member when object locked. The *nolockread* allows read only access to the member when object locked. How these modifiers are applied is specific to each member type.

#### ClientSessionOp(RequestObjectLock)

The ClientSessionOp(RequestObjectLock) request will lock the specified *ServicePath* and any child *service objects* to the current username. Attempts to send requests to members of these objects by a different username or an unauthenticated user will result in an *ObjectLockedError*. The member modifiers *nolock* or *nolockread* can be used to allow members to be accessed even while an object lock is in place.

The lock will be held as long as the username has a client connection session alive. If all sessions are closed or time out, the lock is released.

##### Request:

Direction: Client to Service

* EntryType: 1107
* ServicePath: Service path to object to lock
* MemberName: "RequestObjectLock"
* RequestID: Assigned by client
* Request QoS: reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 1108
* ServicePath: Same as request
* MemberName: "RequestObjectLock"
* RequestID: Same as request
* Request QoS: reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "return"
    * ElementType: 11 (string)
    * Data: "OK" as utf-8 string


#### ClientSessionOp(ReleaseObjectLock)

The ClientSessionOp(ReleaseObjectLock) request will release a previously requested lock on the specified *ServicePath* and any child *service objects* by the current username. Typically only the owning username can release the lock, or a username with superuser privileges.

##### Request:

Direction: Client to Service

* EntryType: 1107
* ServicePath: Service path to release object lock
* MemberName: "ReleaseObjectLock"
* RequestID: Assigned by client
* Request QoS: reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 1108
* ServicePath: Same as request
* MemberName: "ReleaseObjectLock"
* RequestID: Same as request
* Request QoS: reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "return"
    * ElementType: 11 (string)
    * Data: "OK" as utf-8 string

#### ClientSessionOp(RequestClientObjectLock)

The ClientSessionOp(RequestClientObjectLock) request is the same as ClientSessionOp(RequestObjectLock), but locks the service object and child service objects to the current session. Requests from any other sessions, even from the same username, will result in ObjectLockedError. The *nolock* and *noreadlock* modifiers are honored by this lock type.

The lock will be held as long as the client connection session alive. If the session is closed or times out, the lock is released.

##### Request:

Direction: Client to Service

* EntryType: 1107
* ServicePath: Service path to object to lock
* MemberName: "RequestClientObjectLock"
* RequestID: Assigned by client
* Request QoS: reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 1108
* ServicePath: Same as request
* MemberName: "RequestClientObjectLock"
* RequestID: Same as request
* Request QoS: reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "return"
    * ElementType: 11 (string)
    * Data: "OK" as utf-8 string

#### ClientSessionOp(ReleaseClientObjectLock)

The ClientSessionOp(ReleaseClientObjectLock) request is the same as ClientSessionOp(ReleaseObjectLock), but is used for locks created with ClientSessionOp(ReleaseObjectLock). 

##### Request:

Direction: Client to Service

* EntryType: 1107
* ServicePath: Service path to release object lock
* MemberName: "ReleaseClientObjectLock"
* RequestID: Assigned by client
* Request QoS: reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 1108
* ServicePath: Same as request
* MemberName: "ReleaseClientObjectLock"
* RequestID: Same as request
* Request QoS: reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "return"
    * ElementType: 11 (string)
    * Data: "OK" as utf-8 string

### Monitor Locks

Monitor locks are intended to mimick single thread fence locks to prevent data corruption from concurrent modification. They are intended to be short lived and quickly time out. The monitor lock must be constantly refreshed every few seconds to avoid timeout. Ideally, they should be held just long enough to complete quick data manipulation task, and avoided if possible.

#### ClientSessionOp(MonitorEnter)

Monitor are lock guards for a single client thread. They operate on a single object, and are short lived. They should be used when fencing is required to avoid data corruption. While the exact behavior depends on the implementation of the client and service, the lock should only allow the owning client thread to interact with the members.

Instead of raising an error, attempts to access the members are be queued, and processed once the lock is released.

Monitor locks will be held until the are released with ClientsessionOp(MonitorExit), the client connection session is closed, or the monitor times out. A typical timeout for a monitor is 15 seconds. The monitor can be renewed using the ClientSessionOp(MonitorContinueEnter) request.

Return element will contain "Continue" if the monitor enter attempt should be retried.

##### Request:

Direction: Client to Service

* EntryType: 1107
* ServicePath: Service path to request monitor lock
* MemberName: "MonitorEnter"
* RequestID: Assigned by client
* Request QoS: reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "timeout"
    * ElementType: 7 (int32)
    * Data: Timeout in milliseconds as int32 scalar array

##### Response:

Direction: Service to Client

* EntryType: 1108
* ServicePath: Same as request
* MemberName: "MonitorEnter"
* RequestID: Same as request
* Request QoS: reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "return"
    * ElementType: 11 (string)
    * Data: "OK" or "Continue" as utf-8 string

#### ClientSessionOp(MonitorContinueEnter)

Refresh an existing monitor lock created with ClientSessionOp(MonitorEnter) to prevent the lock timing out.

##### Request:

Direction: Client to Service

* EntryType: 1107
* ServicePath: Service path to renew monitor lock
* MemberName: "MonitorContinueEnter"
* RequestID: Assigned by client
* Request QoS: reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 1108
* ServicePath: Same as request
* MemberName: "MonitorContinueEnter"
* RequestID: Same as request
* Request QoS: reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "return"
    * ElementType: 11 (string)
    * Data: "OK" or "Continue" as utf-8 string

#### ClientSessionOp(MonitorExit)

Releases existing monitor lock created with ClientSessionOp(MonitorEnter). Next in request will be given the lock (if any).

##### Request:

Direction: Client to Service

* EntryType: 1107
* ServicePath: Service path to release monitor lock
* MemberName: "MonitorExit"
* RequestID: Assigned by client
* Request QoS: reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 1108
* ServicePath: Same as request
* MemberName: "MonitorExit"
* RequestID: Same as request
* Request QoS: reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "return"
    * ElementType: 11 (string)
    * Data: "OK" as utf-8 string

### Service Objected Release

Service objects can be released by the owning service. The object and all of its child objects can no longer be accessed once the object has been released. Clients connected to the service are notified when a service object has been released using a message packet.

#### ServicePathReleased

Notification that a service path has been released. All child objects of the specified service path have also been released.

##### Packet:

Direction: Service to Client

* EntryType: 1109
* ServicePath: ServicePath of the object that has been released
* MemberName: *empty*
* RequestID: 0
* Requested QoS: Reliable
* Data: *empty*

## Member Protocols

The RPC communication between object references (object proxies) and service objects is accomplished using messages. Each member type has a specific protocol, implemented using *message entries*. While the names and parameter data types varies, the overall protocols are always as specified. Member declaration syntax is specified in [Robot Raconteur Service Definition Standard](service_definition.md)

### Property

The "property" member type is used to read and write a value stored in a property. This is implemented using two message commands, PropertyGet and PropertySet.

Properties may use the following modifiers:

| Modifier | Effect |
| --- | --- |
| readonly | PropertySet results in ReadOnlyMember error |
| writeonly | PropertyGet results in WriteOnlyMember error |
| nolock | PropertyGet and PropertSet request will not be prevented by object or session locks |
| nolockread | PropertyGet request will not be prevented by object or session locks. PropertySet will still result in an ObjectLockedError |
| urgent | PropertyGet or PropertySet operation will be given priority by the transport |
| perclient | The value stored by the session object is associated with the client connection and not global to the service |

Properties have a user defined type. This type is specified by property member declarations in a service definitions.

#### PropertyGet

Retrieves the value of a property member.

##### Request:

Direction: Client to Service

* EntryType: 1111
* ServicePath: Service path to object
* MemberName: Name of member
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 1112
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "value"
    * ElementType: *user defined*
    * Data: *user defined*

#### PropertySet

Sets the value of a property member.

##### Request:

Direction: Client to Service

* EntryType: 1113
* ServicePath: Service path to object
* MemberName: Name of property
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "value"
    * ElementType: *user defined*
    * Data: *user defined*

##### Response:

Direction: Service to Client

* EntryType: 1114
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: *empty*

### Function

Function calls accept zero or more parameters, execute a command, and optionally return a value (or return a generator).

Functions may use the following modifiers:

| Modifier | Effect |
| --- | --- |
| nolock | FunctionCall request will not be prevented by object or session locks |
| urgent | FunctionCall operation will be given priority by the transport |

Functions have zero or more user defined parameter types and an optional return type. These types are specified by function member declarations in a service definitions.

#### FunctionCall (no generator)

Function calls without a generator take zero or more parameters, and optionally return a value. The parameters may be any valid Robot Raconteur type, and have user defined names. These user defined names are used as element names for the request message elements.

If the function specifies a void return type, the response include a zero int32 scalar return message element.

##### Request:

Direction: Client to Service

* EntryType: 1121
* ServicePath: Service path to object
* MemberName: Name of function
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list 0..i..N_params:
  * Element i-:
    * ElementName: *user defined*
    * ElementType: *user defined*
    * Data: *user defined*

##### Response:

Direction: Service to Client

* EntryType: 1122
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list length 0 or 1
  * Element 1:
    * ElementName: "return"
    * ElementType: *user defined* or 7 (int32)
    * Data: *user defined* or \[0\] if void

#### FunctionCall (with generator)

Function calls with a generator take zero or more parameters and returns a generator. A generator is a special form of iterator that can be called repeatedly until exhausted, or is closed. Generators are useful for implementing long running operations, transferring large amounts of data, implementing coroutines, or other iterator tasks that benefit from *yield* semantics. Robot Raconteur Generators are modeled after Python generators.

The generator function will return an "index" message element instead of "return". This index is used by GeneratorNext to access the correct generator. A generator index is returned if the last parameter is declared as a generator, even if the return type is void.

Generators are owned by the service, and must be destroyed (or will time out).

The last parameters in a function declaration in service definition may use the generator container type, ie `param_type param_name{generator}`. If the last parameter of the declaration is of this form, it is not passed as a message element with the FunctionCall request. It is instead used with GeneratorNext.

##### Request:

Direction: Client to Service

* EntryType: 1121
* ServicePath: Service path to object
* MemberName: Name of function
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list 1..i..N_params:
  * Element i-:
    * ElementName: *user defined*
    * ElementType: *user defined*
    * Data: *user defined*

##### Response:

Direction: Service to Client

* EntryType: 1122
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "index"
    * ElementType: 7 (int32)
    * Data: Generator index as scalar int32

#### GeneratorNext

When a function returns a generator, the return message element contains a generator index as an int32 scalar. This index is used to identify the generator for GeneratorNext calls. GeneratorNext is used to increment the iterator, optionally sending a parameter, and optionally receiving a return value. The existence and type of param and return is specified in the function declaration in the service definition. See [Robot Raconteur Service Definition Standard](service_definition.md) for more details.

If the function specifies a void a return type, the response to Next includes a zero int32 scalar return message element.

Next can be used to close or abort a generator by setting the error in the message entry to StopIteration or AbortOperation. Close signals to the generator that the client is finished with the generator, while abort signals the generator that the operation represented by the generator should be aborted. The generator will return the error StopIteration if there are no more values, AbortOperation if the generator aborted by request or due to failure, no error if Next was successful, or any other Robot Raconteur error.

Generators use errors in message entries differently than other members. The StopIteration error type is used for standard program control, and should not be considered an error. StopIteration and AbortOperation error types are used in requests. This is unusual, since error types are typically only used in responses, and ignored in all other request types (unless specified otherwise).

When a generator receives a request with error type StopIteration, it destroys the generator and returns scalar zero int32.

When a generator receives a request with error type AbortOperation, it aborts the underlying operation, destroys the generator, and returns a scalar zero int32.

The generator may return StopIteration at any time to notify the client that there are no more entries (or the operation is complete). It may return any other valid error (including OperationAborted) to notify the client an error has occurred. The generator is destroyed after Next returns any error type.

##### Request:

Direction: Client to Service

* EntryType: 1123
* ServicePath: Service path to object
* MemberName: Name of function
* RequestID: Assigned by client
* Error: 0 (Success), 109 (StopIteration), or 107 (AbortOperation)
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "index"
    * ElementType: 7 (int32)
    * Data: Generator index as scalar int32
  * Element 2- (optional, if no error):
    * ElementName: "parameter"
    * ElementType: *user defined*
    * Data: *user defined*
  
**Request may optionally contain *errorname* and *errorstring* elements if ErrorCode is nonzero as described in [Errors](#Errors), in addition to the generator index. The order of elements is not assigned.**

##### Response:

Direction: Service to Client

* EntryType: 1124
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Error: 0 (Success), 109 (StopIteration), any valid error
* Requested QoS: Reliable
* Data: Message element list length 0 or 1
  * Element 1 (optional):
    * ElementName: "return"
    * ElementType: *user defined* or 7 (int32)
    * Data: *user defined* or \[0\]

**Response will contain *errorname* and *errorstring* elements if ErrorCode is nonzero as described in [Errors](#Errors)**

### Event

Events notify all clients that an event has occurred, and includes zero or more parameters.

| Modifier | Effect |
| --- | --- |
| urgent | Event packet will be given priority by the transport |

Events optionally have user defined parameter types. These types are specified by event member declarations in a service definitions.

Events are broadcasted to all client connections, or to client connections that have been authenticated if authentication is required.

Because events do not have flow control, they should be used sparingly.

#### Event

##### Packet:

Direction: Service to Client

* EntryType: 1131
* ServicePath: Service of object
* MemberName: Name of event
* RequestID: 0
* Requested QoS: Reliable
* Data: Message element list:
  * Data: Message element list 0..i..N_params:
    * Element i-:
      * ElementName: *user defined*
      * ElementType: *user defined*
      * Data: *user defined*

### Object References (objref)

Object Reference (objref) members return the type of an object at a given service path. This service path is computed based on the name of the object, the name of the objref, and the (optional) index specified.

objref are implemented using the ObjectTypeName special request. See [ObjectTypeName](#ObjectTypeName). Note that because this is a special request, authentication may not be enforced by the implementation.

### Pipe

Pipes are used to transmit packets in order from client to service or service to client, implementing fifo pipes. Packets can be transmitted in either direction, with each direction having its own independent queue. Pipe members are used to create PipeEndpoints, which implement sending, queuing, and receiving packets. PipeEndpoints are indexed, meaning that a client can have multiple PipeConnections created for the same member. Pipes may be marked *unreliable*, meaning that packets do not arrive in order, and may be dropped. PipeEndpoints must be connected before data can be transmitted. Either the client or service can close an endpoint connection at any time.

| Modifier | Effect |
| --- | --- |
| readonly | Pipe packets may only be transmitted from service to client |
| writeonly | Pipe packets may only be transmitted from client to service |
| nolock |  Connect request will not be prevented by object or session locks |
| unreliable | Packets may be arrive out of order or be dropped |

#### PipeConnect

Connect a pipe endpoint with the specified index. If the specified index is -1, the service will select a random index.

The "unreliable" message element is used to enable unreliable operation. Both the request and response must contain this message element to enable unreliable operation.

##### Request:

Direction: Client to Service

* EntryType: 1143
* ServicePath: Service path to object
* MemberName: Name of pipe
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "index"
    * ElementType: 7 (int32)
    * Data: *user specified scalar*, or \[-1\] for any
  * Element 2 (optional):
    * ElementName: "unreliable"
    * ElementType: 7 (int32)
    * Data: Scalar int32 \[1\]

##### Response:

Direction: Service to Client

* EntryType: 1144
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "index"
    * ElementType: 7 (int32)
    * Data: Same as request, or random scalar int32 if request index == \[-1\]
  * Element 2 (optional):
    * ElementName: "unreliable"
    * ElementType: 7 (int32)
    * Data: Scalar int32 \[1\]

#### PipeDisconnect

Disconnect a previously connected pipe by index.

##### Request:

Direction: Client to Service

* EntryType: 1145
* ServicePath: Service path to object
* MemberName: Name of pipe
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "index"
    * ElementType: 7 (int32)
    * Data: Index of connected pipe endpoint

##### Response:

Direction: Service to Client

* EntryType: 1144
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: *empty*

#### PipeClosed

PipeClosed is an event packet used to notify the client that the service has closed the PipeEndpoint connection. This is an event packet, meaning that no response is expected from the client. Unlike event members, this event is only sent to the PipeEndpoint connection client.

##### Packet:

Direction: Service to Client

* EntryType: 1147
* ServicePath: Service path to object
* MemberName: Name of pipe
* RequestID: 0
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "index"
    * ElementType: 7 (int32)
    * Data: Index of closed pipe endpoint

#### PipePacket

Pipe packets are used to transmit pipe data. The packet includes the index, the packet sequence number, the packet data, and an optional flag requesting a PacketAck notification that the packet has been received. Multiple pipe packets can be stored in a single PipePacket message entry packet.

Packet numbers must begin at 0, and increment by 1 for each packet. Reliable PipeEndpoint must reorder packets if they arrive out of order. Unreliable packets deliver packets as soon as they arrive.

##### Packet:

Direction: Either

* EntryType: 1141
* ServicePath: Service path to object
* MemberName: Name of pipe
* RequestID: 0
* Requested QoS: Reliable or Unreliable
* Data: List of 1..i..N packets
  * Element i:
    * ElementName: Index of pipe as string
    * ElementType: 103 (varvalue{string})
    * Data: Message element list:
      * Element 1:
        * ElementName: "packetnumber"
        * ElementType: 7 (int32)
        * Data: Packet sequence number
      * Element 2:
        * ElementName "packet"
        * ElementType: *user defined*
        * Data: *user defined*
      * Element 3 (optional):
        * ElementName: "requestack"
        * ElementType: 7 (int32)
        * Data: \[1\]

#### PipePacketAck

The PipePacketAck is used to notify a sending PipeEndpoint that the packet has been received. This is useful for flow control.

##### Packet:

Direction: Either

* EntryType: 1142
* ServicePath: Service path to object
* MemberName: Name of pipe
* RequestID: 0
* Requested QoS: Reliable or Unreliable
* Data: List of 1..i..N packets
  * Element i:
    * ElementName: Index of pipe as string
    * ElementType: 7 (int32)
    * Data: Packet number as int32 scalar

### Callback

A callback is essentially a reverse function that is requested by a service and executed on a specific client. The client is typically identified by the server endpoint representing that client connection. Callback calls calls accept zero or more parameters, execute a command, and optionally return a value. Callbacks cannot return generators.

Callbacks may use the following modifiers:

| Modifier | Effect |
| --- | --- |
| urgent | CallbackCall operation will be given priority by the transport |

Callbacks have zero or more user defined parameter types and an optional return type. These types are specified by callback member declarations in a service definitions.

#### CallbackCall

Callback calls take zero or more parameters, and optionally return a value. The parameters may be any valid Robot Raconteur type, and have user defined names. These user defined names are used as element names for the request message elements.

If the callback specifies a void a return type, the response includes a zero int32 scalar return message element.

##### Request:

Direction: Service to Client

* EntryType: 1151
* ServicePath: Service path to object
* MemberName: Name of callback
* RequestID: Assigned by service
* Requested QoS: Reliable
* Data: Message element list 0..i..N_params:
  * Element i-:
    * ElementName: *user defined*
    * ElementType: *user defined*
    * Data: *user defined*

##### Response:

Direction: Client to Service

* EntryType: 1152
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list length 0 or 1
  * Element 1:
    * ElementName: "return"
    * ElementType: *user defined* or 7 (int32)
    * Data: *user defined* or \[0\]

### Wire

Wires are designed to transmit the *most recent* value of a variable. The value can be sent in either direction, with each direction being independent. WireConnections are established using the WireConnect request. A pair of WireConnections is created by the WireConnect request. WireConnections have an InValue, and an OutValue. When an OutValue is set by a connection, a packet is generated to transmit that value to the opposing WireConnection, and the value in the packet becomes the new InValue of the receiving WireConnection. The value packets are unreliable, and contain a timestamp. The unreliable nature is to ensure that the most recent value arrives as quickly as possible. If an packet arrives that is older than a previously received packet, it is discarded. It is assumed that wire value packets are transmitted rapidly enough that any losses during transmission will not significantly impact the performance of the software.

WireConnections must be connected before values can be transmitted. Either the client or service can close a connection connection at any time.

| Modifier | Effect |
| --- | --- |
| readonly | Wire value packets may only be transmitted from service to client |
| writeonly | Wire value packets may only be transmitted from client to service |
| nolock |  Connect request will not be prevented by object or session locks |

Wires also implement "peek" and "poke" requests. These requests operate without forming a WireConnection, and allow the client to "peek" the InValue, "peek" the OutValue, and "poke" the OutValue. In this usage, the InValue is the value that would be transmitted from the service to the client, and the OutValue is the value that would be transmitted from the service to the client. Using the "peek" and "poke" operations, the wire can used similar to a property for clients that do not need real-time updates on the value.

#### WireConnect

Connect a wire, creating a pair of WireConnections.

##### Request:

Direction: Client to Service

* EntryType: 1163
* ServicePath: Service path to object
* MemberName: Name of wire
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 1164
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: *empty*

#### WireDisconnect

Disconnect a previously connected WireConnection.

##### Request:

Direction: Client to Service

* EntryType: 1165
* ServicePath: Service path to object
* MemberName: Name of wire
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 1166
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: *empty*

#### WireClosed

WireClosed is an event packet used to notify the client that the service has closed the WireConnection. This is an event packet, meaning that no response is expected from the client. Unlike event members, this event is only sent to the WireConnection client.

##### Packet:

Direction: Service to Client

* EntryType: 1167
* ServicePath: Service path to object
* MemberName: Name of pipe
* RequestID: 0
* Requested QoS: Reliable
* Data: *empty*

#### WirePacket

Wire packets are used to transmit the current value. The packet contains the new value and a timestamp.

Packet numbers must begin at 0, and increment by 1 for each packet. Reliable PipeEndpoint must reorder packets if they arrive out of order. Unreliable packets deliver packets as soon as they arrive.

The epoch for *packettime* is Jan 1, 1970, 00:00 UTC. The "nanoseconds" element is always normalized to be positive and less than 1e9.

##### Packet:

Direction: Either

* EntryType: 1161
* ServicePath: Service path to object
* MemberName: Name of wire
* RequestID: 0
* Requested QoS: Unreliable
* Data: List of message elements
  * Element 1-:
    * ElementName: "packettime"
    * ElementType: 101 (structure)
    * ElementTypeName: "RobotRaconteur.TimeSpec"
    * Data: List of message elements
      * Element 1:
        * ElementName: "seconds"
        * ElementType: 9 (int64)
        * Data: Timespec seconds as int64 scalar
      * Element 2:
        * ElementName: "nanoseconds"
        * ElementType: 7 (int32)
        * Data: Timespec nanaseconds as int32 scalar
  * Element 2-:
    * ElementName: "packet"
    * ElementType: *user defined*
    * Data: *user defined*

#### WirePeekInValue

Retrieve the current InValue using a request without creating a WireConnection.

##### Request:

Direction: Client to Service

* EntryType: 1181
* ServicePath: Service path to object
* MemberName: Name of wire
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 1182
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list:  
  * Element 1-:
    * ElementName: "packettime"
    * ElementType: 101 (structure)
    * ElementTypeName: "RobotRaconteur.TimeSpec"
    * Data: List of message elements
      * Element 1:
        * ElementName: "seconds"
        * ElementType: 9 (int64)
        * Data: Timespec seconds as int64 scalar
      * Element 2:
        * ElementName: "nanoseconds"
        * ElementType: 7 (int32)
        * Data: Timespec nanaseconds as int32 scalar
  * Element 2-:
    * ElementName: "packet"
    * ElementType: *user defined*
    * Data: *user defined*

#### WirePeekOutValue

Retrieve the current OutValue using a request without creating a WireConnection.

##### Request:

Direction: Client to Service

* EntryType: 1183
* ServicePath: Service path to object
* MemberName: Name of wire
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: *empty*

##### Response:

Direction: Service to Client

* EntryType: 1184
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list:  
  * Element 1-:
    * ElementName: "packettime"
    * ElementType: 101 (structure)
    * ElementTypeName: "RobotRaconteur.TimeSpec"
    * Data: List of message elements
      * Element 1:
        * ElementName: "seconds"
        * ElementType: 9 (int64)
        * Data: Timespec seconds as int64 scalar
      * Element 2:
        * ElementName: "nanoseconds"
        * ElementType: 7 (int32)
        * Data: Timespec nanaseconds as int32 scalar
  * Element 2-:
    * ElementName: "packet"
    * ElementType: *user defined*
    * Data: *user defined*

#### WirePokeOutValue

Set the current OutValue using a request without creating a WireConnection.

##### Request:

Direction: Client to Service

* EntryType: 1185
* ServicePath: Service path to object
* MemberName: Name of wire
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list
  * Element 1-:
    * ElementName: "packettime"
    * ElementType: 101 (structure)
    * ElementTypeName: "RobotRaconteur.TimeSpec"
    * Data: List of message elements
      * Element 1:
        * ElementName: "seconds"
        * ElementType: 9 (int64)
        * Data: Timespec seconds as int64 scalar
      * Element 2:
        * ElementName: "nanoseconds"
        * ElementType: 7 (int32)
        * Data: Timespec nanaseconds as int32 scalar
  * Element 2-:
    * ElementName: "packet"
    * ElementType: *user defined*
    * Data: *user defined*

##### Response:

Direction: Service to Client

* EntryType: 1186
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: *empty* 
  
### Memory

Memories represent a random-access segment of numeric primitive arrays, numeric primitive multi-dim arrays, pod arrays, pod multi-dim arrays, namedarrays arrays, and namedarrays multi-dim arrays. They are accessed using Read and Write requests, which operate on a subset of the array.

Memories may use the following modifiers:

| Modifier | Effect |
| --- | --- |
| readonly | MemoryWrite results in ReadOnlyMember error |
| writeonly | MemoryRead results in WriteOnlyMember error |
| nolock | MemoryRead and MemoryWrite request will not be prevented by object or session locks |
| nolockread | MemoryRead request will not be prevented by object or session locks. PropertySet will still result in an ObjectLockedError |

#### MemoryRead

Read a segment of the memory.

##### Request:

Direction: Client to Service

* EntryType: 1171
* ServicePath: Service path to object
* MemberName: Name of memory
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list
  * Element 1-:
    * ElementName: "memorypos"
    * ElementType: 10 (uint64[])
    * Data: Array offset scalar or array as uint64
  * Element 2-:
    * ElementName: "count"
    * ElementType: 10 (uint64[])
    * Data: Array count to read as scalar or array as uint64

##### Response:

Direction: Service to Client

* EntryType: 1172
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list
  * Element 1-:
    * ElementName: "memorypos"
    * ElementType: 10 (uint64[])
    * Data: Array offset scalar or array as uint64
  * Element 2-:
    * ElementName: "count"
    * ElementType: 10 (uint64[])
    * Data: Array count to read as scalar or array as uint64
  * Element 3-:
    * ElementName: "data"
    * ElementType: *user defined*
    * Data: *user defined*

#### MemoryWrite

Write a segment of the memory.

##### Request:

Direction: Client to Service

* EntryType: 1173
* ServicePath: Service path to object
* MemberName: Name of memory
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list
  * Element 1-:
    * ElementName: "memorypos"
    * ElementType: 10 (uint64[])
    * Data: Array offset scalar or array as uint64
  * Element 2-:
    * ElementName: "count"
    * ElementType: 10 (uint64[])
    * Data: Array count to write as scalar or array as uint64
  * Element 3-:
    * ElementName: "data"
    * ElementType: *user defined*
    * Data: *user defined*

##### Response:

Direction: Service to Client

* EntryType: 1174
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list
  * Element 1-:
    * ElementName: "memorypos"
    * ElementType: 10 (uint64[])
    * Data: Array offset scalar written or array as uint64
  * Element 2-:
    * ElementName: "count"
    * ElementType: 10 (uint64[])
    * Data: Array count written as scalar or array as uint64


#### MemoryGetParam(MaxTransferSize)

Return the maximum number of elements that can be read or written in one operation.

##### Request:

Direction: Client to Service

* EntryType: 1175
* ServicePath: Service path to object
* MemberName: Name of memory
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "parameter"
    * ElementType: 11 (string)
    * Data: "MaxTransferSize" as utf-8 string

##### Response:

Direction: Service to Client

* EntryType: 1176
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "return"
    * ElementType: 8 (uint32)
    * Data: The maximum number of elements as a uint32 scalar

#### MemoryGetParam(Length)

Return the length of the memory array. Only valid for single dimensional arrays.

##### Request:

Direction: Client to Service

* EntryType: 1175
* ServicePath: Service path to object
* MemberName: Name of memory
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "parameter"
    * ElementType: 11 (string)
    * Data: "Length" as utf-8 string

##### Response:

Direction: Service to Client

* EntryType: 1176
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "return"
    * ElementType: 10 (uint64)
    * Data: The length of the array as a uint64 scalar

#### MemoryGetParam(DimCount)

Return the number of array dimensions. Only valid for multi-dimensional arrays.

##### Request:

Direction: Client to Service

* EntryType: 1175
* ServicePath: Service path to object
* MemberName: Name of memory
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "parameter"
    * ElementType: 11 (string)
    * Data: "DimCount" as utf-8 string

##### Response:

Direction: Service to Client

* EntryType: 1176
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "return"
    * ElementType: 10 (uint64)
    * Data: The number of array dimensions as a uint64 scalar

#### MemoryGetParam(Dimensions)

Return the dimensions of the array. Only valid for multi-dimensional arrays.

##### Request:

Direction: Client to Service

* EntryType: 1175
* ServicePath: Service path to object
* MemberName: Name of memory
* RequestID: Assigned by client
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "parameter"
    * ElementType: 11 (string)
    * Data: "Dimensions" as utf-8 string

##### Response:

Direction: Service to Client

* EntryType: 1176
* ServicePath: Same as request
* MemberName: Same as request
* RequestID: Same as request
* Requested QoS: Reliable
* Data: Message element list:
  * Element 1:
    * ElementName: "return"
    * ElementType: 10 (uint64[])
    * Data: The dimensions of the array as a uint64[]

## Errors

Robot Raconteur defines a number of potential error conditions, and can transmit an error raised during a request back to the requestor. This design makes error handling relatively easy, since errors are automatically transmitted by the framework.

A Robot Raconteur error contains the following information:

* ErrorType: An error code as specified in the table below
* ErrorName: A name corresponding to the error code as defined below, or a *user defined* name. Intended to be machine-readable
* ErrorString: A human readable message describing the error
* ErrorSubName (optional): A short sub-name used to describe in more detail the error. Intended to be machine-readable.
* ErrorParameter (optional): A varvalue associated with an ErrorsubName, to carry more information about what caused the error

Note that not all implementations support ErrorSubName and ErrorParameter.

### ErrorType Codes

The following ErrorType codes are currently defined:

| ErrorType | Code | ErrorName | Description |
| --- | --- | --- | --- |
| None | 0 | Success | *MessageEntry* does not contain an error |
| ConnectionError | 1 | RobotRaconteur.ConnectionError | Existing connection or connection creation has failed |
| ProtocolError | 2 | RobotRaconteur.ProtocolError | Protocol failure occurred on transport connection |
| ServiceNotFound | 3 | RobotRaconteur.ServiceNotFound | The requested service was not found |
| ObjectNotFound | 4 | RobotRaconteur.ObjectNotFound | The specified service path was invalid |
| InvalidEndpoint | 5 | RobotRaconteur.InvalidEndpoint | The specified endpoint was not found |
| EndpointCommunicationFatalError | 6 | RobotRaconteur.EndpointCommunicationFatalError | The specified endpoint has failed, and cannot be used for communication |
| NodeNotFound | 7 | RobotRaconteur.NodeNotFound | The requested node was not found |
| ServiceError | 8 | RobotRaconteur.ServiceError | The request service operation has failed |
| MemberNotFound | 9 | RobotRaconteur.MemberNotFound | The requested object member was not found |
| MemberFormatMismatch | 10 | RobotRaconteur.MemberFormatMismatch | The contents of the *message element* are invalid for the requested member |
| DataTypeMismatch | 11 | RobotRaconteur.DataTypeMismatch | The supplied *message element* does not contain the expected value type |
| DataTypeError | 12 | RobotRaconteur.DataTypeError | The supplied data is invalid for the requested usage |
| DataSerializationError | 13 | RobotRaconteur.DataSerializationError | The supplied message could not be serialized to the transform connection |
| MessageEntryNotFound | 14 | RobotRaconteur.MessageEntryNotFound | The required message entry was not found |
| MessageElementNotFound | 15 | RobotRaconteur.MessageElementNotFound | The required message element was not found |
| UnknownError | 16 | RobotRaconteur.UnknownError | An unknown error has occurred |
| InvalidOperation | 17 | RobotRaconteur.InvalidOperation | The requested operation was invalid for the current state |
| InvalidArgument | 18 | RobotRaconteur.InvalidArgument | The supplied argument was invalid for the operation or current state |
| OperationFailed | 19 | RobotRaconteur.OperationFailed | The requested operation has failed |
| NullValue | 20 | RobotRaconteur.NullValue | A value was null when a non-null value was required |
| InternalError | 21 | RobotRaconteur.InternalError | An unexpected internal error has occurred in the Robot Raconteur implementation |
| SystemResourcePermissionDenied | 22 | RobotRaconteur.SystemResourcePermissionDenied | Permission has been denied to a system resource such as file or device |
| OutOfSystemResource | 23 | RobotRaconteur.OutOfSystemResource | A system resource such as memory has been exhausted |
| SystemResourceError | 24 | RobotRaconteur.SystemResourceError | A system resource error has occured |
| ResourceNotFound | 25 | RobotRaconteur.ResourceNotFound | A requested system resource was not found |
| IOError | 26 | RobotRaconteur.IOError | A generic IO error has occurred |
| BufferLimitViolation | 27 | RobotRaconteur.BufferLimitViolation | A buffer overrun has occurred in a transport |
| ServiceDefinitionError | 28 | RobotRaconteur.ServiceDefinitionError | The supplied service definition failed parsing or verification |
| OutOfRange | 29 | RobotRaconteur.OutOfRange | The supplied index was out of range for an array or list |
| KeyNotFound | 30 | RobotRaconteur.KeyNotFound | The supplied key was not found in a map |
| InvalidConfiguration | 31 | RobotRaconteur.InvalidConfiguration | An invalid configuration was specified or encountered |
| InvalidState | 32 | RobotRaconteur.InvalidState | An invalid state was specified or encountered |
| RemoteError | 100 | *user defined* | An (possibly user defined) error or exception has occurred on the remote node |
| RequestTimeout | 101 | RobotRaconteur.RequestTimeout | The request operation has timed out. Either the operation took too long or communication failed |
| ReadOnlyMember | 102 | RobotRaconteur.ReadOnlyMember | An attempt was made to write a read-only member |
| WriteOnlyMember | 103 | RobotRaconteur.WriteOnlyMember | An attempt was made to read a write-only member |
| NotImplementedError | 104 | RobotRaconteur.NotImplementedError | The requested member has not been implemented |
| MemberBusy | 105 | RobotRaconteur.MemberBusy | The requested member is busy. Try again |
| ValueNotSet | 106 | RobotRaconteur.ValueNotSet | The requested value has not been set and is currently undefined |
| AbortOperation | 107 | RobotRaconteur.AbortOperation | Only valid for generators. Abort the current iteration |
| OperationAborted | 108 | RobotRaconteur.OperationAborted | The current operation was aborted |
| StopIteration | 109 | RobotRaconteur.StopIteration | Only valid for generators. Close the generator, or generator is finished |
| OperationTimeout | 110 | RobotRaconteur.OperationTimeout | An operation did not complete in the expected time |
| OperationCancelled | 111 | RobotRaconteur.OperationCancelled | An operation was cancelled before starting |
| AuthenticationError | 150 | Authentication is required, or authentication failed |
| ObjectLockedError | 151 | The requested object is locked by another user or session |
| PermissionDenied | 152 | Permission to the requested resource was denied |

### User Defined Errors

Errors may be defined in service definitions using the *exception* keyword. See [Robot Raconteur Service Definition Standard](service_definition.md) for more details. These user defined exceptions are essentially just a new ErrorName, which is used with RemoteError, code 100. More sophisticated Robot Raconteur implementations are able to map these error names to user defined exceptions, while simper implementations will raise a RemoteError with a different ErrorName field.

### MessageEntry Error Storage

As defined in [Message Entries](#Message-Entries), message entries contain an ErrorCode. If this is zero, there is no error, and the entry behaves normally. If ErrorCode is not zero, then an error has occurred, the message entry contains an error, and it cannot be processed normally be the recipient. The message elements contain information about the error, and no longer contain the expected elements.

An error message entry has the following contents:

* EntryType: Unchanged
* ServicePath: Unchanged
* MemberName: Unchanged
* RequestID: Unchanged
* ErrorType: The ErrorType code of the raised error
* Requested QoS: Reliable
* Data: Message element list
  * Element 1-:
    * ElementName: "errorname"
    * ElementType: 11 (string)
    * Data: The ErrorName of the raised error as a utf-8 string. May be *user defined*
  * Element 2-:
    * ElementName: "errorstring"
    * ElementType: 11 (string)
    * Data: A message for the user as a utf-8 string
  * Element 3- (optional):
    * ElementName: "errorsubname"
    * ElementType: 11 (string)
    * Data: A short sub-name as a utf-8 string
  * Element 4- (optional):
    * ElementName: "errorparam"
    * ElementType: *any* (varvalue)
    * Data: Parameters associated with the ErrorSubName, stored as a varvalue.

**errorparam must not use any named types.**

### Undeliverable Messages

Occasionally a message may arrive with invalid node address information, invalid endpoint number, or invalid EntryType. In these cases, it is impossible to forward the message to a recipient operation to be processed. In these cases, if the EntryType is odd, the EntryType should be incremented by one, an error applied to the entry, and the message entry error returned to the sender. If the EntryType is even, the message should be discarded.

## Conventions

### Long Running Operations

Requests will typically have a default timeout of 15 seconds. This can be adjusted by clients, but it should be assumed that requests should complete within 15 seconds. Long running operations should use events, pipe, or function generators to avoid this timeout limitation.

When using a generator for a long running operation, if an abort is issued, it should be considered an urgent situation. If the operation represents a physical action, the action should be immediately suspended.

### Transferring Large Data

Most transports have a limitation of around 10 MB per message. Any data larger than this will result in an error. To transfer data larger than this 10 MB limit, it needs to be broken down into smaller messages. This can be cleanly accomplished using a function generator, or using pipes which send the data in several packets.

### Broadcasters

Pipes and Wires are unicast by design, transmitting or receiving data only from one client. In many situations, this is undesirable, and the data should be sent to all connected clients. This is implemented using a PipeBroadcaster, or WireBroadcaster. These take data and broadcast it to all connected clients that have connected the appropriate members. Since these broadcasters do not affect the object protocol, they are conventions rather than part of the official standard.

Flow control for the PipeBroadcaster can be accomplished by limiting the number of packets in-flight by monitoring the RequestPacketAck responses. If too many packets are in flight, new packets should be dropped. Wires automatically drop stale data, so there is no need for this feature in WireBroadcaster.

### Message Field Conventions

Message field names currently used a variety of CamelCase, snake_case, and works without separation. There is no technical requirement for a specific convention, however going forward it is recommended that snake_case be used.
