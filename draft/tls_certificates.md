<p align="center"><img src="../images/RRheader2.jpg"></p>

# Robot Raconteur Public Key Certificates

Version 0.9.2

http://robotraconteur.com

http://github.com/robotraconteur

Copyright &copy; 2020 Wason Technology, LLC

*Robot Raconteur is a communication framework designed for use with robotics, automation, building control, and internet of things applications.*

## Abstract

TLS requires Public Key Certificates to authenticate the server and optionally the client. Robot Raconteur uses a special format for public key certificates and the associated public key infrastructure. All Robot Raconteur certificates are expected to be supplied by Wason Technology, LLC.

## Introduction

Robot Raconteur uses TLS to secure TCP and QUIC transport connections. TLS uses public key certificates to validate the identity of the server and optionally the client. These certificates contain identity information, a public key, and are signed by an identity authority. The certificates and infrastructure to certify distribute and distribute the certificates is called Public Key Infrastructure (PKI). PKI is a complex topic and the reader is highly encouraged to consult additional [references](https://en.wikipedia.org/wiki/Public_key_infrastructure).

HTTPS is the most common use for PKI. Certificate authorities such as Comodo, DigiCert, GlobalSign, and GoDaddy sell certificates tied to a DNS name. Web browsers are distributed with the root certificates for these authorities, and these root certificates are used to test that a certificate is valid. The web browser will then compare the DNS name specified in the certificate to the current web address. If the verification succeeds, the web browser user will be informed that the connection is secure. See [SSL and SSL Certificates Explained For Beginners](http://www.steves-internet-guide.com/ssl-certificates-explained/) for a more detailed introduction to HTTPS certificates.

Robot Raconteur has a very different use case for TLS when compared to HTTPS. Since they are normally used on a local network, Robot Raconteur devices do not have a host name, or a fixed address. Instead, the only fixed identifier is the NodeID. For this reason, Robot Raconteur TLS certificates are tied to the 128-bit UUID NodeID, instead of other information that can potentially change.

## Node Certificates

Robot Raconteur node certificates are standard X509.3 certificates. They are designed to be unusable for HTTPS to avoid any potential attack vector. The X509.3 certificate subject must only contain the `CN` attribute. The `CN` attribute must contain the magic `Robot Raconteur Node` followed by a space, and then the 128-bit UUID in bracketed 8-4-4-4-12 format. The `CN` must match the following regex:

    ^Robot Raconteur Node \{[0-9A-Fa-f]{8}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{12}\}$

Robot Raconteur certificates use custom extensions marked "critical". These exist to mark the certificates for use with Robot Raconteur, and to make them unusable with HTTPS or other purposes. For nodes, the extension with OID `1.3.6.1.4.1.45455.1.1.3.3` is marked critical and boolean true. The X509 full format is:

    1.3.6.1.4.1.45455.1.1.3.3=critical,ASN1:BOOL:TRUE

The following standard critical fields are expected:

* basicConstraints: CA:FALSE
* keyUsage: "digitalSignature, keyEncipherment, dataEncipherment"

Any additional critical extensions may result in verification failures. The absence of critical extension OID `1.3.6.1.4.1.45455.1.1.3.3` will result in verification failure.

As part of the licensing agreement to use the Robot Raconteur trademark, all node certificates must be purchased from Wason Technology, LLC.

## Intermediate Certificate

Intermediate certificates exist between the root certificate and node certificates. These certificates are standard PKI certificates, with the addition of the custom OID extension marked critical:

    1.3.6.1.4.1.45455.1.1.3.2=critical,ASN1:BOOL:TRUE

The subject field is ignored. The following standard critical extensions are expected:

* basicConstraints: CA:TRUE, pathlen:0
* keyUsage: keyCertSign, cRLSign

Note `pathlen` may be a value other than zero.

Any additional critical extensions may result in verification failures. The absence of critical extension OID `1.3.6.1.4.1.45455.1.1.3.2` will result in verification failure.

## Root Certificate

Root certificates anchor the entire certificate chain. They must be distributed with every Robot Raconteur node so that the certificates can be validated. Root certificates must contain the following custom critical extension:

    1.3.6.1.4.1.45455.1.1.3.1=critical,ASN1:BOOL:TRUE

The subject field is ignored. The following standard critical extensions are expected:

* basicConstraints: CA:TRUE, pathlen:1
* keyUsage: keyCertSign, cRLSign

Note `pathlen` may be a value other than one.

Any additional critical extensions may result in verification failures. The absence of critical extension OID `1.3.6.1.4.1.45455.1.1.3.1` will result in verification failure.

All Robot Raconteur nodes are expected to use certificates issued by Wason Technology, LLC. All nodes are expected to have the Wason Technology, LLC root certificate embedded in the software:

    -----BEGIN CERTIFICATE-----
    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
    -----END CERTIFICATE-----
