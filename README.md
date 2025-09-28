# LABUBU - A Lightweight Multiplexing WebTransport Proxy Protocol

Version 1.0 - written by [@jargdev](https://github.com/jargdev)

Inspired by [Wisp](https://github.com/MercuryWorkshop/wisp-protocol)

## About

LABUBU is a low-overhead, simple-to-implement protocol, where multiple UDP/QUIC streams can be proxied over a single WebTransport session.  Wisp's concepts are expanded upon by LABUBU, which is tailored for WebTransport and makes use of its low-latency, dependable and unreliable datagram streams. LABUBU has strong error handling and is easier to use than alternatives.

## Packet format

| Field Name  | Field Type | Notes                                           |
| ----------- | ---------- | ----------------------------------------------- |
| Packet Type | `uint8_t`  | The packet type, described in the next section. |
| Stream ID   | `uint32_t` | Random stream ID assigned by the client.        |
| Payload     | `char[]`   | Payload occupies the rest of the packet.        |

All packets follow this format. Data types are **little-endian**.

---

## Packet Types

### `0x01` - CONNECT

#### Payload Format

| Field Name           | Field Type | Notes                                         |
| -------------------- | ---------- | --------------------------------------------- |
| Destination Port     | `uint16_t` | Destination UDP/QUIC port for the new stream. |
| Destination Hostname | `char[]`   | Destination hostname, UTF-8 string.           |

#### Behavior

* The client sends a CONNECT packet to create a new UDP stream under the WebTransport session.
* The Stream ID is associated with this UDP stream for all future packets.
* The server validates the destination; if invalid, it sends a CLOSE packet.
* Upon successful validation, the server establishes a UDP socket or QUIC connection to the destination.

> **Note:** Unlike Wisp, there is no TCP option. LABUBU is strictly UDP/QUIC.

---

### `0x02` - DATA

#### Payload Format

| Field Name     | Field Type | Notes                                    |
| -------------- | ---------- | ---------------------------------------- |
| Stream Payload | `char[]`   | Data to be sent to/from the destination. |

#### Behavior

* All DATA packets are sent over WebTransport to the server, then forwarded to the associated UDP socket.
* Incoming UDP data is packaged in DATA packets and sent to the client.
* Optional: DATA packets can be marked as **reliable** or **unreliable** using a header bit in the future.

> **Note:** WebTransport datagrams can be unreliable; LABUBU does not implement a TCP-style buffer, unlike Wisp.

---

### `0x03` - CLOSE

#### Payload Format

| Field Name   | Field Type | Notes                          |
| ------------ | ---------- | ------------------------------ |
| Close Reason | `uint8_t`  | Reason for closing the stream. |

#### Behavior

* A CLOSE packet closes the associated UDP stream immediately.
* Optional close reasons may provide debugging info.

#### Close Reason Codes

* `0x01` - Reason unspecified or unknown.
* `0x02` - Voluntary stream closure by client or server.
* `0x03` - Unexpected closure due to network error.
* `0x41` - Stream creation failed (invalid hostname or port).
* `0x42` - Destination unreachable.

---

## WebTransport Behavior

* All LABUBU traffic is multiplexed over a single WebTransport session.
* The client initiates a WebTransport session using standard HTTP/3 handshake.
* Each UDP stream corresponds to a **unidirectional or bidirectional WebTransport stream**.
* Reliable messages can use **WebTransport streams**, while low-latency, unreliable messages use **WebTransport datagrams**.

---

## Server Architecture

* Must implement a WebTransport server capable of handling multiple streams and datagrams concurrently.
* Each stream is mapped to a UDP socket, maintaining minimal buffering for high performance.
* No CONTINUE packet is needed (unlike Wisp), since UDP/QUIC streams don’t rely on TCP-style flow control.

---

## Choosing a WebTransport URL

* URL format: `https://example.com/wut/`
* The server may optionally use the URL prefix for authentication or gatekeeping.

---

## Establishing a WebTransport Connection

1. Client performs a standard WebTransport handshake over HTTP/3.
2. Server responds with acceptance of the session.
3. The client may immediately send CONNECT packets for any UDP streams.
4. DATA packets follow for active streams; CLOSE packets terminate streams.
