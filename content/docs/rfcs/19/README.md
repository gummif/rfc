---
slug: 19
title: 19/FILEMQ
name: File Message Queuing Protocol
status: retired
editor: Pieter Hintjens <ph@imatix.com>
---

The File Message Queuing Protocol (FILEMQ) governs the delivery of files between a 'client' and a 'server'. FILEMQ runs over the ZeroMQ [ZMTP protocol](http://rfc.zeromq.org/spec:15/ZMTP).

## License

Copyright (c) 2009-2012 iMatix Corporation

This Specification is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation; either version 3 of the License, or (at your option) any later version.

This Specification is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with this program; if not, see <http://www.gnu.org/licenses>.

## Change Process

This Specification is a free and open standard (see "[Definition of a Free and Open Standard](http://www.digistan.org/open-standard:definition)") and is governed by the Digital Standards Organization's Consensus-Oriented Specification System (COSS) (see "[Consensus Oriented Specification System](http://www.digistan.org/spec:1/COSS)").

## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 (see "[Key words for use in RFCs to Indicate Requirement Levels](http://tools.ietf.org/html/rfc2119)").

## Goals

The FILEMQ protocol defines a mechanism for wide-area file distribution using a publish-subscribe pattern. Its goals are:

* To allow clients to "subscribe" to server-hosted directories with zero out-of-band knowledge of the files that will be created on those directories.
* To provide high-speed chunked file delivery to clients.
* To allow the clients to cancel and restart file transfers arbitrarily.
* To be fully secure.

## Implementation

### Formal Grammar

The following ABNF grammar defines the FILEMQ protocol:

```
filemq-protocol = open-peering *use-peering [ close-peering ]

open-peering    = C:OHAI *( S:ORLY C:YARLY ) ( S:OHAI-OK / error )

use-peering     = C:ICANHAZ ( S:ICANHAZ-OK / error )
                / C:NOM
                / S:CHEEZBURGER
                / C:HUGZ S:HUGZ-OK
                / S:HUGZ C:HUGZ-OK

close-peering   = C:KTHXBAI / S:KTHXBAI

error           = S:SRSLY / S:RTFM

;   The client opens peering to the server
OHAI            = signature %x01 protocol version
signature       = %xAA %xA3
protocol        = string        ; Must be "FILEMQ"
string          = size *VCHAR
size            = OCTET
version         = %x01

;   The server challenges the client using the SASL model
ORLY            = signature %x02 mechanisms challenge
mechanisms      = size 1*mechanism
mechanism       = string
challenge       = *OCTET        ; ZeroMQ frame

;   The client responds with SASL authentication information
YARLY           = %signature x03 mechanism response
response        = *OCTET        ; ZeroMQ frame

;   The server grants the client access
OHAI-OK         = signature %x04

;   The client subscribes to a virtual path
ICANHAZ         = signature %x05 path options cache
path            = string        ; Full path or path prefix
options         = dictionary
dictionary      = size *key-value
key-value       = string        ; Formatted as name=value
cache           = dictionary    ; File SHA-1 signatures

;   The server confirms the subscription
ICANHAZ-OK      = signature %x06

;   The client sends credit to the server
NOM             = signature %x07 credit
credit          = 8OCTET        ; 64-bit integer, network order

;   The server sends a chunk of file data
CHEEZBURGER     = signature %x08 sequence operation filename
                  offset headers chunk
sequence        = 8OCTET        ; 64-bit integer, network order
operation       = create | delete
create          = %x01          ; Octet
delete          = %x02          ; Octet
filename        = string
offset          = 8OCTET        ; 64-bit integer, network order
headers         = dictionary
chunk           = FRAME

;   Client or server sends a heartbeat
HUGZ            = signature %x09

;   Client or server responds to a heartbeat
HUGZ-OK         = signature %x0A

;   Client closes the peering
KTHXBAI         = signature %x0B

;   Server error reply - refused due to access rights
S:SRSLY         = signature %x80 reason
reason          = string

;   Server error reply - client sent an invalid command
S:RTFM          = signature %x81 reason
```

### Interconnection Model

#### ZeroMQ Socket Types

The server SHALL create a ROUTER socket and SHOULD bind it to port 5670, which is the registered Internet Assigned Numbers Authority (IANA) port for FILEMQ. The server MAY bind its ROUTER socket to other ports in the ephemeral port range (%C000x - %FFFFx). The client SHALL create a DEALER socket and connect it to the server ROUTER host and port.

Note that the ROUTER socket provides the caller with the connection identity of the sender for any message received on the socket, as an identity frame that precedes other frames in the message.

#### Protocol Signature

Every ZeroMQ message SHALL start with the FILEMQ protocol signature, %xAA %xA3. The server and client SHALL silently discard any message received that does not start with these two octets.

This mechanism is designed particularly for servers that bind to ephemeral ports which may have been previously used by other protocols, and to which there are still peers attempting to connect. It is also a general fail-fast mechanism to detect ill-formed messages.

#### Connection State

The server SHALL reply to an unexpected command with a RTFM command. The client SHALL respond to an RTFM command by closing its DEALER connection and starting a new connection.

### FILEMQ Commands

#### The OHAI Command

The client SHALL start a new connection by sending the OHAI command to the server. This command identifies the protocol and version. This is designed to allow version detection.

If the server does not support the request protocol version it SHALL reply with a RTFM command. The client MAY try again with a lower protocol version.

If the server accepts the OHAI command it SHALL reply with either a OHAI-OK command or, if authentication is required, an ORLY command.

#### The ORLY Command

When a client requests access, the server MAY reply with an ORLY command to initiate a Simple Authentication and Security Layer (SASL) challenge and response exchange. The server SHALL provide a list of mechanisms that it accepts, and a frame of binary data containing a challenge.

#### The YARLY Command

The client SHOULD respond to an ORLY command with a YARLY command that has calculated response data. Note that the SASL challenge/response model uses external SASL libraries that process the challenge data and return suitable response data.

The client or server MUST at least support the SASL "PLAIN" mechanism.

#### The OHAI-OK Command

When the server grants the client access after an OHAI or YARL command, it SHALL reply with an OHAI-OK command. If the server does not grant access it SHALL reply with a SRSLY command.

#### The ICANHAZ Command

The client MAY subscribe to any number of virtual paths by sending ICANHAZ commands to the server. The client MAY specify a full path, or it MAY specify a partial path, which is used as a prefix match. Paths MUST start with "/", thus the path "/" subscribes to *everything*.

The 'path' does not have to exist in the server. That is, clients can request paths which will exist in the server at some future time.

The 'options' field provides additional information to the server. The server SHOULD implement these options:

* <tt>RESYNC=1</tt> - if the client sets this, the server SHALL send the full contents of the virtual path to the client, except files the client already has, as identified by their SHA-1 digest in the 'cache' field.

When the client specifies the RESYNC option, the 'cache' dictionary field tells the server which files the client already has. Each entry in the 'cache' dictionary is a "filename=digest" key/value pair where the digest SHALL be a SHA-1 digest in printable hexadecimal format. If the filename starts with '/' then it SHOULD start with the path, otherwise the server MUST ignore it. If the filename does not start with '/' then the server SHALL treat it as relative to the path.

#### The ICANHAZ-OK Command

When a server accepts a subscription it MUST reply with an ICANHAZ-OK command. If the server refuses a subscription it SHALL reply with a SRSLY command, and discard any further commands from this client.

#### The NOM Command

The client MUST initiate the transfer of data by sending credit to the server. The server SHALL only send as much data to the client as it has credit for. The credit is an amount in bytes that corresponds to actual file content (but not bytes used by commands themselves).

The client MAY sent NOM commands at any point after it has received an OHAI-OK from the server. The server SHALL not respond directly to NOM commands.

#### The CHEEZBURGER Command

The server SHALL send file content to the client using CHEEZBURGER commands. Each CHEEZBURGER command shall deliver a chunk of file data starting at a specific offset. The server MUST send the content of a single file as consecutive chunks and clients MAY depend on this behavior.

The headers field is reserved for future use.

#### The HUGZ Command

The server or client MAY sent heartbeat commands at any point after the server has sent OHAI-OK to the client, which has received it.

The HUGZ command acts as a heartbeat, indicating that the peer is alive. The server and client SHALL treat *any* command from a peer as a sign that the peer is alive.

A peer may thus choose to only send HUGZ to another peer when it is not sending any other traffic to that peer.

#### The HUGZ-OK Command

A peer SHALL respond to a HUGZ command with a HUGZ-OK command. This allows one peer to be responsible for all heartbeating.

#### The KTHXBAI Command

The client MAY end a connection by sending the KTHXBAI command to the server. The server SHALL not respond to this command.

#### The SRSLY Command

The server SHALL respond to any failed attempt to access a resource on the server with a SRSLY command. This includes failed authentication and failed subscriptions. When a client receives a SRSLY command it SHOULD close the connection and if needed, reconnect with new authentication credentials.

#### The RTFM Command

The server SHALL respond to an invalid command by sending RTFM. Note that the server SHALL not send RTFM to clients which send an invalid protocol signature. When a client receives a RTFM command it SHOULD close the connection and not reconnect.

## Security Aspects

FILEMQ uses the Simple Authentication and Security Layer (SASL) for authentication and encryption. The SHA-1 digest used for file cache identification has no security implications.
