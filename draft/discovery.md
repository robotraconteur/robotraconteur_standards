<p align="center"><img src="../images/RRheader2.jpg"></p>

# Robot Raconteur Node and Service Discovery Standard

Version 0.9.2

http://robotraconteur.com

http://github.com/robotraconteur

Copyright &copy; 2020 Wason Technology, LLC

*Robot Raconteur is a communication framework designed for use with robotics, automation, building control, and internet of things applications.*

## Abstract

Discovery for Robot Raconteur is a three step process: announce, interrogation, and connection. The method of announce is transport-specific. Interrogation occurs using a special Robot Raconteur service available on every service node. Connection uses the data retrieved from the node to make the connection.

## Introduction

Discovery is an integral part of using devices on a local network. It can be difficult to determine addresses of devices, and these addresses may change over time. Robot Raconteur discovery allows for nodes and services to be discovered. Discovery can find nodes using the following criteria:

* NodeID
* NodeName
* Service root object type and types implemented
* Service attributes

Discovery is a three step process:

1. Service nodes announce their existence, including NodeID, NodeName, and connection URL for RobotRaconteurServiceIndex
2. Client nodes collect node announcements, and interrogate the RobotRaconteurServiceIndex to determine what services are available
3. Client connections are formed based on the collected discovery information

## Node Announce

The first step of discovery is announcing the existence of a service node. This announce contains the NodeID, the NodeName, and a URL to connect to the RobotRaconteurServiceIndex. This announce packet does not contain any additional information. It only contains enough information to connect to the service index to retrieve more information.

An announce packet has the following format:

    Robot Raconteur Node Discovery Packet\n
    <nodeid>,<nodename>\n
    <service_index_url>\n
    ServiceStateNonce: <state_nonce>\n

Where `\n` is a newline, `<nodeid>` is the NodeID in bracket format `{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}`, `<nodename>` is an optionally the NodeName, and `<service_index_url>` is the URL to connect to the service index. The packet may optionally contain extra fields, with the name and value separated by a colon. The only currently used field is `ServiceStateNonce`. This nonce is a random alphanumeric string, regex `[A-Za-z0-9]{16}`. This string represents if the data returned by RobotRaconteurServiceIndex has changed. While the data returned by the service is constant, the nonce shall be unchanged. When the information has changed, the nonce shall change. Clients can monitor the service state nonce, and re-check the index service if it changes. Service nodes shall resend announce packets when the information returned by the index service changes.

The connection URL must include the NodeID, must NOT include the NodeName, and should have service name `RobotRaconteurServiceIndex`.

Announce packets should match the following regex:

    ^Robot Raconteur Node Discovery Packet\n\{[A-Fa-f0-9]{8}\-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{12}\}(?:,[a-zA-Z](?:\w*[a-zA-Z0-9])?(?:\.[a-zA-Z](?:\w*[a-zA-Z0-9])?)*)?\nrrs?\+\w{1,8}\:\/\/.*\/?nodeid=[A-Fa-f0-9]{8}\-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{4}-[A-Fa-f0-9]{12}\&service=RobotRaconteurServiceIndex\n(?:ServiceStateNonce: [A-Za-z0-9]{16}\n)?$

The URL scheme, authority, and path will vary depending on the transport.

How announce packets are transmitted depends on the transport. For example, the TCP transport uses multicast UDP packets, while the local transport uses files. See the individual transport standard for details on how the discovery packets are transmitted.

## Service Interrogation

Once a service node has been detected, the client needs to determine what services are available on the service node. The announce contains the URL to the `RobotRaconteurServiceIndex`, a special service run on all service nodes that provide discovery. This service should not require authentication. The service index uses the following service definition, and has root object type `RobotRaconteurServiceIndex.ServiceIndex`.

    service RobotRaconteurServiceIndex
    struct NodeInfo
        field string NodeName
        field uint8[16] NodeID
        field string{int32} ServiceIndexConnectionURL
    end struct
    struct ServiceInfo
        field string Name
        field string RootObjectType
        field string{int32} RootObjectImplements
        field string{int32} ConnectionURL
        field varvalue{string} Attributes
    end struct
    object ServiceIndex
        function ServiceInfo{int32} GetLocalNodeServices()
        function NodeInfo{int32} GetRoutedNodes()
        function NodeInfo{int32} GetDetectedNodes()
        event LocalNodeServicesChanged()
    end object

The only member that should be used for service interrogation is `GetLocalNodeServices()`. The rest are deprecated.

The function `GetLocalNodesServices()` returns a map of the available services. (This service uses map instead of list because list was not available when it was defined.) The `ServiceInfo` structure contains the name of the service, the root object type, the types the root object implements, the candidate URLs for connection to the service, and a set of attributes.

The candidate connection URLs should be configured to match the transport address used to for interrogation. For example, if a local transport connection is used for interrogation, a connection URL using the local transport should be used. If a TCP address is used for interrogation, the returned URL should used the same IP endpoint that was used to accept the interrogation request.

The `Attributes` field is a varvalue map intended to contain arbitrary information that can be used to identify a device.

The `ServiceInfo` structure does not contain the NodeID and NodeName. For this reason, the Robot Raconteur node will often return a `ServiceInfo2` structure to the user. The `ServiceInfo2` structure has the following contents:

* ServiceName
* RootObjectType
* RootObjectImplements
* ConnectionURL
* Attributes
* NodeID
* NodeName

The `NodeInfo2` structure is used to inform users of detected nodes:

* NodeID
* NodeName
* ConnectionURL (list of candidate URL)

## Establish Connection

After nodes have been detected and interrogated, the client node has a list of available services and the URLs for connection. The user can then filter the `ServiceInfo` and/or `ServiceInfo2` structures to determine which services to connect. Nodes should have a strategy to filter candidate URLs, since a client will often be provided with multiple candidates. The different URLs should be tried, in order of quality of the connection.

A rough strategy for selecting URL priorities are, from best to worst:

1. Local Transport
2. Hardware Transports
3. TCP Loopback
4. QUIC
5. Secure TCP IPv6 link-local
6. Secure TCP local subnet
7. Secure TCP other
8. Insecure TCP IPv6 link-cola
9. Insecure TCP local subnet
10. Insecure TCP other
11. Other

## Discovery API

Four functions are typically provided to the user:

* FindServiceByType: Searches for a service by `RootObjectType` and `RootObjectImplements`. Optionally takes a list of URL schemes to search. Returns a list of `ServiceInfo2` structures.
* FindNodeByID: Searches for a node by NodeID. Returns a list of `NodeInfo2` structures.
* FindNodeByName: Searches for a node by NodeName. Returns a list of `NodeInfo2` structures.
* GetDetectedNodes: Returns a list of `NodeInfo2`structures containing nodes that have been detected from announce packets.

## Subscriptions

Subscriptions are used to create and maintain connections to services whenever they are available. Subscriptions automate the connection process after nodes have been detected. The exact behavior of subscriptions is implementation specific.

For most implementations, a filter is used to determine if a service should be connected. A filter is specified when the subscription instance is created, and the subscription will maintain client connections with all services that match this filter.

A filter has the following fields:

* Root Object Type or Implements
* Nodes
  * NodeID
  * NodeID
  * Username
  * Credentials
* ServiceNames
* TransportSchemes
* Predicate (user defined callback function)
* MaxConnections

All fields have a wildcard setting, which mean to ignore the field. How this filter is implemented and its behavior is implementation specific.
