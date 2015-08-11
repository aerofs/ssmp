Stupid-Simple Messaging Protocol
================================

SSMP, aka the Stupid-Simple Messaging Protocol is an application-level
protocol for 1:1 and 1:many messaging which aims to be a lightweight
alternative to open messaging protocols such as XMPP or STOMP.

Key design goals:
  - Text-based, for easy debugging
  - Interleave request/responses and server events on a single connection
  - Simple enough that a complete and efficient client or server can be
    written in pretty much any programming language within a few hours


History
-------

  - 1.0: Initial version


Assumptions
-----------

SSMP is designed to run atop a reliable 2-way streaming transport such as TCP.


Message format
--------------

Each message is a `LF`-delimited sequence of UTF-8 encoded unicode codepoints
that MUST NOT exceed 1024 bytes, `LF` delimiter included. Messages MUST be
separated by a single `LF` delimiter.

A message is composed of space-separated fields of the following types:

 Type         |  Syntax
--------------|-------------------------
 `VERB`       |  `[A-Z]+`
 `IDENTIFIER` |  `[a-zA-Z0-9.:@/_-+=~]+`
 `PAYLOAD`    |  `[^\n]+`
 `CODE`       |  `[0-9]{3}`

Messages fall in three categories:

  - client request <br/>
    `<VERB> <IDENTIFIER>* <PAYLOAD>?`
  - server response <br/>
    `<CODE> <PAYLOAD>?`
  - server event <br/>
    `000 <IDENTIFIER> <VERB> <IDENTIFIER>? <PAYLOAD>?`


Response codes
--------------

Response code values are borrowed from HTTP where appropriate:

  - `200` OK
  - `400` Bad Request
  - `401` Unauthorized
  - `404` Not Found
  - `405` Not Allowed
  - `501` Not Implemented

The special `000` code is used to distinguish server events from request/responses,
thereby allowing events to be freely interleaved with regular responses on the same
connection.

Login
-----

The first client request in any connection MUST be a `LOGIN`.

The format of a `LOGIN` request is:

    LOGIN <id> <scheme> <credential>

where

  - `id` is of type `IDENTIFIER`
  - `scheme` is of type `IDENTIFIER`
  - `credential` is of type `PAYLOAD`


If the authentication is successful, the server MUST send a `200` response
with no payload.

If no request is received after a reasonable period of time, typically a few
seconds, the server MUST close the connection without sending any response.

If the first request is not a `LOGIN` the server MUST send a `400` response
and immediately close the connection.

If the `scheme` value is not supported or if the authentication fails for
any other reason, the server MUST send a `401` response with with a space-
separated list of supported authentication schemes as payload and immediately
close the connection.

After a successful `LOGIN` request, servers MUST reject any subsequent `LOGIN`
request on the same connection with code `405`.

### Anonymous connections

Servers MAY allow login with the reserved `.` user identifier.

Anonymous connections are not able to subscribe to any topic and cannot be the
destination of unicast messages but they can publish messages to existing
topics.


### Authentication schemes

#### Client certificate

All servers that accept connection over SSL/TLS MUST allow authentication through
client certificates.

The `scheme` value for cert-based authentication is `cert`.

When a connection is made with a client certificate, `LOGIN` should succeed for
any `id` matching either the Common Name or one of the Subject Alternative Names
specified in the certificate.

To accommodate multiple connections being opened using the same certificate,
servers MAY accept identifiers consisting of a valid CN or altName followed by
a forward slash (`/`) and an arbitrary sequence of `IDENTIFIER` characters.

#### Shared secret

Servers MAY allow authentication through a pre-shared secret.

The `scheme` value for shared secret authentication is `secret`.

#### Open login

Servers MAY allow unauthenticated `LOGIN`.

The `scheme` value for open login is `open`.

The client SHOULD NOT include a `credential` field in an `open` login request
and the server MUST ignore its content if it is present.

Open login is subject to easy abuse and SHOULD therefore only be enabled for
debugging purposes.


Ping
----

### Client-initiated

Clients SHOULD send a `PING` request after an implementation-defined period where
no server event is received, typically about 30s.

Upon reception of a `PING` request, servers MUST send a `PONG` event using the
anonymous identifier as its provenance.

    Client          Server

    PING    ---->

            <----   000 . PONG

### Server-initiated

Servers SHOULD send a `PING` event after an implementation-defined period where
no client request is received, typically about 30s. The provenance MUST be the
anonymous identifier.

Upon reception of a `PING` event, clients MUST send a `PONG` message.

Servers MUST NOT send any message in response to a `PONG` message.

    Client          Server

            <----   000 . PING

    PONG    ---->


Topic subscription
-------------------


### Subscribe to multicast topic

Opt in to receiving events for messages sent to a multicast topic.

    SUBSCRIBE <topic> [PRESENCE]

Any `SUBSCRIBE` request from an anonymous user MUST be rejected with code `405`.

If the caller was already subscribed to the given topic, the server MUST respond
with code `409`.

The optional `PRESENCE` flag can be used to subscribe to presence notifications,
i.e. receive events when other peer subscribe to or unsubscribe from the topic.

#### Presence notifications

Upon successful subscription, the server MUST send the following event to each
client who specified the `PRESENCE` flag when subscribing to the given topic:

    000 <from> SUBSCRIBE <topic>

where

  - `from` is the identifier of the peer having sent the `SUBSCRIBE` request
  - `topic` is the name of the topic in the `SUBSCRIBE` request


When the `PRESENCE` flag is provided, the caller will receive an initial batch
of `SUBSCRIBE` events for all existing subscribers and subsequently, `SUBSCRIBE`
and `UNSUBSCRIBE` events as topic membership changes.

The server must ensure that presence notifications are delivered in a safe order.
Crucially, if peer A subscribe to a topic T and peer B unsubscribes from it, the
server must ensure that peer A either does not receive a `SUBSCRIBE` event about
B or receives it before the `UNSUBSCRIBE` event.


### Unsubscribe from multicast topic

Opt out of receiving events for messages sent to a multicast topic.

    UNSUBSCRIBE <topic>

Any `UNSUBSCRIBE` request from an anonymous user MUST be rejected with code `405`.

If the caller was not subscribed to the given topic, the server MUST respond with
code `404`.

#### Presence notifications

Upon successful unsubscription, the server MUST send the following event to each
client who specified the `PRESENCE` flag when subscribing to the given topic:

    000 <from> UNSUBSCRIBE <topic>

where

  - `from` is the identifier of the peer having sent the `UNSUBSCRIBE` request
  - `topic` is the name of the topic in the `UNSUBSCRIBE` request


Messages
--------

Message delivery is:
  - in-order: two messages from the same sender to the same recipient MUST arrive
    in-order at the recipient
  - at most once: recipients MUST NOT receive duplicate messages
  - best effort with no acknowledgment: a successful response from the server
    indicates that the message was sent but it may not be received


### Unicast

Send message to a single peer.

    UCAST <to> <payload>

where

  - `to` is the identifier used by the remote peer in its `LOGIN` request
  - `payload` is the message payload to be sent to the peer


If no peer with the requested identifier is currently connected, the server
MUST send a `404` response.

Otherwise it MUST send the following event to the given peer:

    000 <from> UCAST <payload>

where

  - `from` is the identifier of the peer having sent the `UCAST` request
  - `payload` is the message payload from the `UCAST` request

### Multicast

Send message to all peers subscribed to a given topic.

    MCAST <topic> <payload>

where

  - `topic` is the name of the topic
  - `payload` is the message payload to be sent to the topic subscribers

The server MUST NOT send a 404 response, even if no peer has subscribed to the
given topic.

The server MUST send the following event to every peer having subscribed to the
given topic:

    000 <from> MCAST <topic> <payload>

where

  - `from` is the identifier of the peer having sent the `MCAST` request
  - `topic` is the name of the topic given in the `MCAST` request
  - `payload` is the message payload from the `MCAST` request

NB: One does not need to be subscribed to a topic to send messages to it.

### Broadcast

Broadcast to all peers sharing at least one topic.

    BCAST <payload>

where

  - `payload` is the message payload to be broadcasted

Any `BCAST` request from an anonymous user MUST be rejected with code `405`.

The server MUST send the following event to every peer sharing at least one
topic with the sender of the `BCAST` request.

    000 <from> BCAST <payload>

where

  - `from` is the identifier of the peer having sent the `BCAST` request
  - `payload` is the message payload from the `BCAST` request

Peers that share multiple topics with the sender MUST NOT receive multiple
identical `BCAST` events.


Forward compatibility
---------------------

Upon reception of a request with an unrecognized `VERB` field, servers MUST
send a `501` response.

This is intended to allow client to safely detect whether servers support
any new or optional requests that may be added in future versions of this
specification.

