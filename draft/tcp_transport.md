<p align="center"><img src="../images/RRheader2.jpg"></p>

# Robot Raconteur TCP Transport Standard

Version 0.9.2

http://robotraconteur.com

http://github.com/robotraconteur

Copyright &copy; 2020 Wason Technology, LLC

*Robot Raconteur is a communication framework designed for use with robotics, automation, building control, and internet of things applications.*

## Abstract

The TCP Transport is the standard transport for communicating between Robot Raconteur nodes. TCP is a ubiquitous network protocol, and is supported by virtually all networked devices. The TCP Transport implements the Basic Stream Protocol, with extensions for HTTP Web Sockets and Transport Layer Security (TLS). The TCP transport has full support for both IPv4 and IPv6. UDP broadcast and/or multicast is used for node discovery. Port 48653 for TCP and UDP are officially registered with IANA for use with Robot Raconteur.

## Introduction

TCP is a reliable stream protocol used for most internet traffic, including HTTP web browsers. Robot Raconteur uses TCP as its basic transport due to its simplicity and ubiquitousness. The Robot Raconteur TCP transport is designed for use with IPv4 and IPv6, Web Sockets, and Transport Layer Security (TLS). It also has support for node discovery using UDP. The resulting transport is a capable and reliable transport. 

While TCP has many advantages, it has some disadvantages that will cause it to be phased out of use over the next few years (or possibly decades). These disadvantages are discussed in the [Disadvantages](#Disadvantages) section.

## TCP Transport URLs

Robot Raconteur uses URLs to locate services. These URLs have the following format, as discussed in [Robot Raconteur Framework Architecture Standard](framework_architecture.md).

`URL = scheme://[authority]/[path]?[nodeid=<nodeid>&][nodename=<nodename>&]service=<servicename>`

### scheme

The TCP Transport supports the following schemes:

| Scheme     | Description |
| ---        | ---         |
| rr+tcp     | Standard TCP, without encryption |
| rrs+tcp    | Standard TCP, with TLS encrypted Robot Raconteur stream (STARTTLS) |
| rr+ws      | HTTP WebSocket over TCP |
| rrs+ws     | HTTP WebSocket over TCP with encryption Robot Raconteur stream |
| rr+wss     | HTTPS encrypted WebSocket over TCP with unencrypted Robot Raconteur stream |
| rrs+wss    | HTTPS encrypted WebSocket over TCP with encrypted Robot Raconteur stream |

TLS encryption uses STARTTLS to enable encryption and uses Robot Raconteur Node Certificates to identify peers. See [TLS Security](#TLS-Security) and [Robot Raconteur Node Certificates](node_certificates.md) for more details.

WebSockets are used to provide compatibility with Web Browsers and HTTP servers. See [WebSockets](#WebSockets) for more details, and how encryption is used with WebSockets.

### authority

The authority portion of the URL is broken up into two parts for the TCP transport:

`authority = host[:port]`

The authority section of the URL is either a registered name (typically DNS), or an IP address. If the host is a registered name, it may be a familiar DNS format name like `myservice.robotraconteur.com`. On local networks, the host is typically an IP address. For IPv4 addresses, this takes the form of four numbers separated by dots. For example, `192.168.1.211` is an IP address frequently encountered on local networks.

The host may also take the form of an IPv6 address. IPv6 addresses are specified between brackets, and the segments of the address are separated by colons. IPv6 addresses are somewhat complicated, so it is recommended that IETF RFC 4291 be reviewed for more details.

When using Robot Raconteur on a local network, the IPv6 addresses will nearly always be "link-local" addresses. These addresses are only valid on the local network, and are based on the MAC address of the adapter. (See Section 2.5.6 of IETF RFC 4291 for more details.) Link-local addresses always start with `fe80::`, followed by the 64-bit EUI-64 interface address. (The EUI-64 is normally generated from the 48-bit MAC address of the internet adapter. (See Appendix A of IETF RFC 4291.) This address is (almost always) inherently permanent, making it an excellent choice for use on local networks where DHCP can randomly change IP addresses. An example IPv6 address is `[fe80::30b4:9a80:7758:2377]`.

IPv6 addresses also have the concept of "scope-ids" following a percent symbol after the address. For example, `[fe80::30b4:9a80:7758:2377%14]` is the same address as before with a scope-id. The scope-id refers to which physical adapter on the computer can contact the link local address in question. The scope-id is used by the *local* computer, and can vary from computer to computer. Robot Raconteur automatically tests all scope-ids, so scope-ids do not normally need to be specified.

The port component of the URL specifies which TCP port from which the server is accepting connections. If not specified, it default to the official port 48653. This port is officially registered with IANA for use by Robot Raconteur.

### path

The path component is not used for standard TCP, and must be empty or a single slash (/). If using WebSockets, the path can be used to specify the filename of the target WebSocket on the HTTP server. Robot Raconteur ignores the path in this situation. If connection to a Robot Raconteur TCP Transport running in websocket compatibility mode, the path should be empty or a single slash(/).

### nodename and nodeid

The nodename and nodeid component of the URL are not typically used by the TCP Transport. The routing informaiton of "CreateConnection" or "STARTTLS" request may be used to select the target node if multiple nodes are served by the same TCP port or server WebSocket.

### service

The service is not used by the TcpTransport. It is used by the node during connection.

### Additional Query Parameters

Additional query parameters are ignored by the URL. Extra parameters may be useful for passing information used by the HTTP server during connection establishment, for instance a session token may be necessary to authorize the WebSocket connection.

## TCP

TCP is the workhorse transport protocol for the modern internet. It is defined in IETF RFC 293. On modern computers, TCP is access using the Berkeley Sockets API (or WinSock on Windows). The Berkeley Socket API allows clients to connect using an IP address and port, send and receive data, and close connections. For servers, it allows programs to bind to TCP ports, listen, and accept incoming connections. Robot Raconteur uses standard TCP with sockets for its basic communication. The connection is treated as a stream, using the [Robot Raconteur Basic Stream Protocol](basic_stream_protocol.md). Additional features including WebSockets and TLS are layered over the TCP stream for additional functionality.

## WebSockets

All TCP Transport implementations are required to accept WebSocket incoming connections, and to initiate WebSocket client connections, as appropriate. (Some implementations are client or server only.)  Supporting WebSockets allows compatibility with web infrastructure, such as web browsers and HTTP servers. These technologies are becoming ubiquitous to the point that many computer users never leave a web browser during normal computer activities.

WebSockets are a simple wrapper protocol over TCP that allow for persistent connection between web browsers and HTTP servers. The WebSocket protocol is defined in IETF RFC 6455. The [Robot Raconteur Basic Stream Protocol](basic_stream_protocol.md) is encapsulated in WebSocket frames, with these frames passing over the TCP connection. No other changes to the Robot Raconteur protocol are made other than having the data encapsulated in WebSocket frames.

Robot Raconteur uses the following WebSocket protocol name:

`robotraconteur.robotraconteur.com`

Four aspects of the WebSocket protocol are important for the TCP Transport:

* Opening Handshake
* Origin Control
* Data Framing
* Frame Masking
* Control Messages

### Opening Handshake

The opening handshake is discussed in section 4 of IETF RFC 6455. The handshake is an HTTP `GET` request, with an `Upgrade: websocket` field in the request header. The client must respond as specified in the RFC, with additional verification steps as described in Section 4.7 of the RFC (`Sec-WebSocket-Key` header field). The `Origin` of the client *should* also be verified. Once the handshake is complete, WebSocket frames can be used to communicate Robot Raconteur stream data.

### Origin Control

WebSockets are vulnerable to "Cross-Site Scripting (XSS) Attacks". A malicious web page can access servers and devices that it is not supposed to, and make requests to the devices. This is a common vector of attack against devices like home internet routers. A malicious website will direct requests toward 192.168.1.1, and very often will be able to access the home router, even though the user has only visited a web page. WebSockets attempt to protect against this method of attack with the `Origin` field in the request header. The web browser will include the `authority` portion of the URL of the web page attempting to make the request. For instance, a web page served by the Wason Technology server would have the field `Origin: wasontech.com`. See IETF RFC 6455 Section 4.2.1. The `Origin` for webpages served of the filesystem will typically be empty, or the string "null". Robot Raconteur clients initiating connections should omit the `Origin` field. Robot Raconteur nodes accepting WebSocket connections *should* check the `Origin` header to guard against XSS attacks. `Origin` values served from the `robotraconteur.com` domain and subdomains *should* be considered trusted.

### Data Framing

Robot Raconteur message data is encapsulated in "Binary" WebSocket frames, as defined in the RFC. Each frame must contain no more than 64 KB of data. Messages that are larger than 64 KB must be broken up into multiple frames. Messages *should* start on a frame boundary, but this is not required. Messages are broken up into 64 KB to prevent buffering issues in web browsers and HTTP servers.

### Frame Masking

WebSocket frames "mask" frames sent from a client to the server. This is another attempt to guard against XSS attacks. A clever attacker could potentially use unmodified data to attack a proxy or another server. Masking exclusive-ors (XOR) the data with a 32-bit integer to reversibly scramble the data. This masking is not a form of encryption since the mask value is included in the frame. Rather, it is only intended to stymie XSS attackers. Robot Raconteur clients *must* implement masking, or HTTP servers will reject the frames. Services must have the ability to interpret incoming masked frames, but *must not* mask sent frames. See Section 5.3 of the RFC.

### Control Masking

Robot Raconteur *must* correctly respond to Ping-Pong control messages and Close requests, as defined in 5.5 of the RFC. In general Ping-Pong frames should not be generated, since the Robot Raconteur stream protocol is continuously sending heartbeat messages. Robot Raconteur nodes are not required to fully implement the Close control message, and may simply terminate the TCP connection when finished. The TCP connection *should* be terminated if a Close message is received.

## Establishing Connections

Connection establishment using TCP can be done over TCPv4 or TCPv6, and has four potential modes:

* Standard TCP
* TLS Secured TCP
* HTTP WebSocket
* HTTPS WebSocket

Optionally the WebSocket stream data may also be protected by TLS.

The discussion of establishing connections is broken down into three phases:

* Establish TCP socket connection
* Detect and Complete HTTP WebSocket handshake (optional)
* Start TLS encryption (optional)

Special case

* Establish HTTPS WebSocket connection

### Establish TCP socket connection

The first step to establishing a TCP Transport connection is to connect the underlying TCP socket. TCP Transport servers listen to a port to accept connections. This is accomplished using the "bind", "listen", and "accept" socket commands. Servers *should* listen on both IPv4 and IPv6 address families.  If only one can be supported at a time, IPv6 *should* be preferred. If the TCP Transport is accepting WebSockets through an HTTP server, the server will be configured to forward incoming connections at a specific path to the transport.

Clients initiate socket connections using the socket "connect" command. The IP address of the desired service may be specified directly by the user, or may need to be resolved using DNS. Often times, multiple candidate IP addresses are available. When dealing with an IPv6 link-local address, there are typically multiple candidate scope-ids. (The client computer should be interrogated for potential scope-ids using a command like `getifaddrs`.) When the client has multiple potential IP addresses or IP address-scope-ids available, a "king-of-the-hill" approach should be used, where multiple candidate address are attempted, with a short, random back-off period between the initiation of the connections. A typical back-off of 5 - 100 ms has worked well in previous implementations, but the exact behavior necessary will vary.

Once the TCP socket connection has been established, the connection can be upgraded to WebSocket or TLS as necessary. If the scheme `rr+tcp` was used, the client may initialize the Robot Raconteur stream using a `CreateConnection` message.

### Detect and Complete HTTP WebSocket handshake (optional)

If the client has specified the URL scheme `rr+ws` or `rrs+ws`, the client must now initiate a handshake as discussed in [Opening Handshake](#Opening-Handshake). This handshake is an HTTP `GET` request with an `Upgrade` header field. Once the handshake is complete, the stream may be initiated by sending a `CreateConnection` message if the `rr+ws` scheme was specified, or continue to TLS handshake if the `rrs+ws` scheme was specified.

The service node most be able to detect whether the incoming connection is Robot Raconteur standard TCP, or if the incoming connection is an HTTP handshake. This is accomplished by looking at the first four bytes of data received. If the first four bytes are the ASCII text `RRAC`, the incoming connection is a standard Robot Raconteur stream. If the first four bytes are the ASCII text `GET `, then the incoming connection is an HTTP handshake. If an HTTP handshake is the received, the headers must be evaluated to see if an `Upgrade: websocket` header field is present. If it is, a WebSocket handshake should be completed. Once complete, the service should wait for the client to send an initial stream message. If it is not present, the client is requesting a file. The service may optionally respond with a file, if configured to serve resources over TCP.

Once a WebSocket handshake has been complete, or standard Robot Raconteur connection has been detected, the service will wait for the initializing message.

### Start TLS encryption (optional)

If the client has specified the URL scheme `rrs+tcp` or `rrs+ws`, the client must now send a `STARTTLS` message. This `STARTTLS` message is a StreamOp message with the member name `STARTTLS`. The client *may* specify the desired server node by filling in the RemoteNodeID and/or the RemoteNodeName if the TCP port is shared by more than one node. If the client wants to authenticate with a client certificate, it may add a "mutualauth" message element to the request. The server responds with a `STARTTLS` StreamOp response, optionally with a "mutualauth" message element if it agrees to mutual authentication.

##### Request:

Direction: Client to Service

* EntryType: 1
* ServicePath: *empty*
* MemberName: "STARTTLS"
* RequestID: 0
* Request QoS: reliable
* Data: Message element list:
  * Element 1- (optional):
    * ElementName: "mutualauth"
    * ElementType: 11 (string)
    * Data: utf-8 string "true" to enable mutual authentication, or "false" to disable (default "false")


##### Response:

Direction: Service to Client

* EntryType: 2
* ServicePath: *empty*
* MemberName: "STARTTLS"
* RequestID: 0
* Request QoS: reliable
* Data: Message element list:
  * Element 1- (optional):
    * ElementName: "mutualauth"
    * ElementType: 11 (string)
    * Data: utf-8 string "true" to enable mutual authentication, or "false" to disable (default "false"). Must be first enabled by client.

Once both the client and service have sent and received `STARTTLS` messages, a TLS handshake occurs. The TLS certificates must be verified according to  [Robot Raconteur TLS Certificate Standard](node_tls_certificates.md). Any errors in the handshake must result in the TCP connection being immediately terminated.

Once the TLS handshake is complete, all messages are protected using TLS. The stream may be initialized using the `CreateConnection` message as specified in [Robot Raconteur Basic Stream Protocol Standard](framework_architecture.md).

When the `rrs+ws` scheme is used, the Robot Raconteur messages are protected by TLS, but the WebSocket handshake and frame information is not. This is made available for use on a local network since HTTPS is poorly supported, however any data sent over public networks including the Internet *should* be protected using `rrs+tcp` or `rr+wss` since all data is protected, including the handshakes and framing.

### Establish HTTPS WebSocket connection

Establishing a connection over HTTPS is a special case because services cannot easily detect the incoming handshake, and HTTPS certificates are tied to a specific IP address or DNS name. For these reasons, HTTPS is a poor option for use on local networks where Robot Raconteur normally operates. When used with an standard HTTP server with a DNS name however, it is necessary to use HTTPS to establish connections. For this reason, the `rr+wss` and `rrs+wss` schemes are defined. Clients *should* support these schemes to allow for connection to standard HTTPS servers.

When a client uses the scheme `rr+wss` or `rrs+wss`, the client first initiates a standard HTTPS TLS connection, verifying the certificate against standard public root certificates. (These public root certificates are normally specified by operating systems or web browser vendors. Robot Raconteur does not specify the public root certificates.) Once the TLS connection is established, the WebSocket handshake as specified above is used to create the connection. When used with an HTTPS server, a `path` component to the URL will normally need to be specified. If the `rrs+wss` scheme is specified, the client must initiate a `STARTTLS` to enable Robot Raconteur TLS encryption. This may be necessary in specific cases where the stream is forwarded, or the client or server node needs to be authenticated. If the `rr+wss` scheme is specified, the client should initialize the connection using a `CreateConnection` message.

## Normal Operation

Once the connection is established, messages are transmitted using the TCP connection as specified in [Robot Raconteur Basic Stream Protocol Standard](basic_stream_protocol.md).

## Connection Close

Connections are closed as specified in [Robot Raconteur Basic Stream Protocol Standard](basic_stream_protocol.md). The server should close the socket connection after transmitting a  DisconnectClientRet (110).

## Protocol Error

Protocol errors or messages with sizes exceeding the transport should immediately result in the connection being terminated.

## Discovery

Node and Service discovery is described in [Robot Raconteur Node and Service Discovery Standard](discovery.md). Discovery is a three step process:

1. Service nodes announce their existence, including NodeID, NodeName, and connection URL for RobotRaconteurServiceIndex
2. Client nodes collect node announcements, and interrogate the RobotRaconteurServiceIndex to determine what services are available
3. Client connections are formed based on the collected discovery information

Steps 2 and 3 are discussed in [Robot Raconteur Node and Service Discovery Standard](discovery.md) and will not be discussed again here.

Step 1 on a local network involves announcing the existence of a node to all systems on the network. On an IP network, this is accomplished using UDP. UDP discovery packets are sent using broadcast (IPv4) or multicast (IPv6) so that they are delivered to all listening systems. Each UDP packet contains a single Robot Raconteur Node Announce Packet, as discussed in [Robot Raconteur Node and Service Discovery Standard](discovery.md). This announce packet contains a magic, the NodeID, the NodeName, and the URL for connection to the RobotRaconteurServiceIndex of the node. The specified connection URL should have the same IP address as the send field of the UDP packet. Packets that do not follow this rule may be ignored by clients.

### UDP Discovery Packet Addresses and Port

UDP discovery always occurs on port 48653, for both IPv4 and IPv6. By default, only link-local IPv6 multicast is used for discovery.

#### IPv4

Discovery uses the broadcast address 255.255.255.255, on port 48653. By default, IPv4 discovery is disabled, using only IPv6 discovery.

### IPv6

IPv6 discovery uses the following multicast addresses:

| Address | Scope |
| ---     | --- |
| ff01::ba86 | node-local |
| ff02::ba86 | link-local |
| ff03::ba86 | site-local |

The meanings of node-local, link-local, and site-local are defined in IETF RFC 4291. By default, only link-local IPv6 discovery is enabled.

### UDP Discovery Packet Protocol

Using UDP multicast is challenging, because excessive announcements can cause network congestion, but users do not want to wait a significant amount of time for discovery to occur. For this reason, Robot Raconteur discovery over UDP operates in two modes:

1. Passive announce
2. Call and response

When using passive announce, service nodes passively send discovery packets every 10 - 15 seconds at all times. While this is simple and effective, it can cause significant network usage if there are a lot of nodes. Client nodes shall assume that if an announce node has not been received for 60 seconds the node has been closed.

When using call and response, service nodes passively send discovery packets every 55 seconds. Since it is assumed that clients will consider the node lost after 60 seconds, this period will refresh the node announce with clients. Since 55 seconds is a long time for users to wait to detect nodes, clients may also send a Node Announce Request Packet. This packet has the following format:

    Robot Raconteur Discovery Request Packet\n

Where `\n` is a Unix newline. There is no leading whitespace. This packet only contains the magic string, and is sent using the same UDP IP addresses as announce packet. Clients should send three of these request packets, with 900 ms to 1500 ms delay between packets. The exact delay should be selected randomly. When clients receive a request packet, it should send three announce packets, with 250 ms to 1000 ms between packets. The exact delay should also be selected randomly. To prevent excessive sending of announce packets, after the six initial announce packets, the node shall limit announce packet sending to every 15 seconds for one or two minutes. The node shall send the announce packet over the broadcast/multicast address. It shall not unicast the announce packet to the sender of the request packet. It also shall not respond to request packets that are not sent by a device on the local subnet or using link-local IPv6 addresses.

The announce packets shall use IPv4 or IPv6 addresses in the connection URL. The URL shall not use hostnames, DNS names, or include the IPv6 scope-id.
