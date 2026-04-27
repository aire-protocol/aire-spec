# AIRE Specification — Draft v0.1

> **Author:** Etienne de Bruin ([@etdebruin](https://github.com/etdebruin)).
> **Status:** Draft. Breaking changes expected until v1.0.
> **Last updated:** 2026-04-27.

## 1. Introduction

AIRE (Agent Interchange Runtime Envelope) is an application-layer protocol for agent-to-agent communication. It runs on QUIC (RFC 9000) and provides agent-native semantics — capability negotiation, identity-bound operations, semantic cancellation, and cost-aware backpressure — that are absent from HTTP-based agent protocols.

### 1.1 Goals

- First-class support for agent migration across hosts and networks.
- Multiplexed, non-HOL-blocking streams for parallel fan-out.
- Semantic cancellation: kill one logical operation without nuking the channel.
- Budget-aware backpressure: tokens and dollars are first-class flow-control units.
- Identity bound to streams, not connections.
- Capability negotiation at handshake time.
- Resumability of logical operations across new connections.

### 1.2 Non-goals

- Replacing TCP, UDP, or QUIC.
- Defining LLM model behavior, prompt formats, or tool schemas.
- Providing transport-level reliability beyond what QUIC offers.
- Mandating a specific identity provider.

### 1.3 Terminology

- **Node** — a host process that participates in an AIRE mesh. Identified by a Node ID.
- **Agent** — a logical unit of behavior. Identified by an Agent ID, scoped to a Node.
- **Operation** — a single logical task (e.g. "answer this query"). Identified by an Operation ID, scoped to a Stream.
- **Stream** — a QUIC stream carrying frames for one Operation.
- **Connection** — a QUIC connection between two Nodes.
- **Capability** — a named, versioned, optionally-signed unit of agent functionality.

## 2. Wire format

Every AIRE message is a self-delimiting *frame*. Frames are sent as the byte-stream payload of QUIC streams (RFC 9000 §2.1). A single QUIC stream MAY carry one or more frames concatenated end-to-end with no separators.

### 2.1 Frame envelope

```
+--------+--------+----------+-------------+----------+
| Type   | Flags  | OpID     | PayloadLen  | Payload  |
| 1 byte | 1 byte | varint   | varint      | bytes    |
+--------+--------+----------+-------------+----------+
```

| Field      | Size               | Description                                                                                                                                            |
|------------|--------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Type       | 1 byte (uint8)     | Frame type code (see §3).                                                                                                                              |
| Flags      | 1 byte (uint8)     | Type-specific flags. Reserved bits MUST be zero on send and MUST be ignored on receive.                                                                |
| OpID       | varint (§2.2)      | Operation identifier scoped to the connection. Value `0` is reserved for connection-level frames (HELLO, GOODBYE). Operation-scoped frames MUST use a non-zero OpID. |
| PayloadLen | varint (§2.2)      | Length of the Payload field, in bytes.                                                                                                                 |
| Payload    | `PayloadLen` bytes | Frame-type-specific payload (see §3).                                                                                                                  |

All multi-byte integers are encoded in network byte order (big-endian).

### 2.2 Variable-length integers (varint)

AIRE varints reuse the QUIC variable-length integer encoding (RFC 9000 §16). The two most significant bits of the first byte indicate the total length:

| 2 MSB | Length  | Value range                              |
|-------|---------|------------------------------------------|
| `00`  | 1 byte  | 0 to 63 (2⁶ − 1)                         |
| `01`  | 2 bytes | 0 to 16 383 (2¹⁴ − 1)                    |
| `10`  | 4 bytes | 0 to 1 073 741 823 (2³⁰ − 1)             |
| `11`  | 8 bytes | 0 to 4 611 686 018 427 387 903 (2⁶² − 1) |

The remaining bits hold the value, big-endian. Senders SHOULD use the shortest encoding that fits the value. Receivers MUST accept any valid encoding — longer-than-minimal encodings are tolerated for forward compatibility, though senders are not required to emit them.

### 2.3 Self-delimiting concatenation

Multiple frames MAY be concatenated within a single QUIC stream. A receiver parses frames sequentially: read Type, Flags, OpID, PayloadLen, then exactly PayloadLen bytes of Payload. The next frame begins immediately at the byte position following the previous frame's Payload.

A QUIC stream MUST NOT close (FIN) mid-frame. An implementation that detects a partial frame at FIN MUST treat the connection as malformed and emit ERROR (§3) before tearing down.

### 2.4 Operation lifetime within a stream

By convention, all frames belonging to a single Operation share one QUIC stream and the same OpID. The Operation begins with an INVOKE frame (§3) and terminates with one of:

- a STREAM frame indicating end-of-operation (see §3),
- a CANCEL frame, or
- an ERROR frame indicating a terminal error.

After the terminal frame for an Operation, the QUIC stream's send-side MUST be closed (FIN). The receive-side closes when the peer's terminal frame arrives.

Connection-level frames (HELLO, GOODBYE; OpID = 0) travel on a dedicated control stream, defined in §4.

### 2.5 Maximum frame size

Implementations MUST accept frames whose PayloadLen is up to 65 536 bytes (2¹⁶) without prior negotiation. Implementations MAY accept larger frames; senders SHOULD NOT exceed 1 048 576 bytes (2²⁰) without first negotiating a higher `max_frame_size` capability via CAPABILITY (§4). Receivers MAY emit ERROR with code `FRAME_TOO_LARGE` if a received frame exceeds their configured limit.

### 2.6 Test vectors

The following byte sequences are canonical encodings. Implementations MUST round-trip these byte-for-byte.

#### Vector 1 — empty HELLO

A connection-level HELLO frame with no payload.

```
01 00 00 00
```

| Field      | Bytes    | Decoded               |
|------------|----------|-----------------------|
| Type       | `01`     | HELLO (0x01)          |
| Flags      | `00`     | 0                     |
| OpID       | `00`     | 0 (connection-level)  |
| PayloadLen | `00`     | 0                     |
| Payload    | *(none)* | empty                 |

#### Vector 2 — INVOKE with short payload

An INVOKE frame on Operation 42, payload `"hi"` (UTF-8).

```
03 00 2A 02 68 69
```

| Field      | Bytes   | Decoded                |
|------------|---------|------------------------|
| Type       | `03`    | INVOKE (0x03)          |
| Flags      | `00`    | 0                      |
| OpID       | `2A`    | 42 (1-byte varint)     |
| PayloadLen | `02`    | 2                      |
| Payload    | `68 69` | `"hi"`                 |

#### Vector 3 — STREAM with 4-byte OpID and 2-byte length

A STREAM frame on Operation 16 384, payload of 100 bytes (all `0xAB`).

```
04 00 80 00 40 00 40 64
AB AB AB ... (100 × 0xAB)
```

| Field      | Bytes           | Decoded                    |
|------------|-----------------|----------------------------|
| Type       | `04`            | STREAM (0x04)              |
| Flags      | `00`            | 0                          |
| OpID       | `80 00 40 00`   | 16 384 (4-byte varint)     |
| PayloadLen | `40 64`         | 100 (2-byte varint)        |
| Payload    | 100 × `AB`      | 100 bytes                  |

Total frame length: 108 bytes.

#### Vector 4 — two empty HELLO frames concatenated

Demonstrates self-delimiting concatenation. The byte sequence parses as two consecutive HELLO frames.

```
01 00 00 00 01 00 00 00
```

A receiver MUST parse this as two complete HELLO frames, not as one malformed frame.

> **Note.** Additional vectors covering each frame type, edge-case varint lengths, and a machine-readable JSON encoding will be added under issue [#4 (test vectors for wire format)](https://github.com/aire-protocol/aire-spec/issues/4).

## 3. Frame types

| Code | Name        | Direction       | Purpose                                                    |
|------|-------------|-----------------|------------------------------------------------------------|
| 0x01 | HELLO       | both            | Initial handshake — Node ID, version, supported capabilities |
| 0x02 | CAPABILITY  | both            | Capability advertisement and negotiation                   |
| 0x03 | INVOKE      | client → server | Begin an Operation on a target Agent                       |
| 0x04 | STREAM      | both            | Streamed payload data for an active Operation              |
| 0x05 | CANCEL      | both            | Cancel a specific Operation (propagates to delegated sub-ops) |
| 0x06 | BUDGET      | both            | Budget update — tokens remaining, cost remaining, deadline |
| 0x07 | DELEGATE    | server → client | Forward an Operation to another Node                       |
| 0x08 | ERROR       | both            | Typed error frame (rate limit, budget, auth, etc.)         |
| 0x09 | GOODBYE     | both            | Graceful shutdown                                          |

Frame codes 0x80+ reserved for vendor extensions; 0x10–0x7F reserved for future versions.

## 4. Handshake

*TODO (v0.1):* `HELLO` exchange flow, capability negotiation, version selection.

## 5. Identity model

*TODO (v0.2):* DID-based identity. Frames carry signatures bound to the issuer's DID. Identity is per-stream (per-operation), not per-connection — a single connection can carry operations on behalf of many distinct agent identities.

## 6. URI scheme

AIRE addresses are expressed as URIs with the `aire` scheme.

### 6.1 Grammar

The grammar follows RFC 3986. In ABNF:

```
aire-uri    = "aire://" authority [ "/" agent-id [ "/" operation ] ]
authority   = host [ ":" port ]
host        = <host as defined by RFC 3986 §3.2.2>
port        = <port as defined by RFC 3986 §3.2.3>
agent-id    = 1*pchar
operation   = 1*pchar
pchar       = <as defined by RFC 3986 §3.3>
```

`host` MAY be a DNS name, an IPv4 dotted-quad literal, or a bracketed IPv6 literal. `agent-id` and `operation` are case-sensitive UTF-8 strings, percent-encoded as required by RFC 3986.

### 6.2 Default port

The default AIRE port is **`4433/udp`** (QUIC). When `port` is omitted from the URI, implementations MUST connect to UDP/4433.

> Port `4433` is unassigned by IANA at this time and is conventional in QUIC-based tooling. Once AIRE adoption warrants, a dedicated IANA assignment will be requested and the default revised in a major version bump.

### 6.3 Resolution

Given an `aire://` URI, the connecting peer:

1. Resolves `host` to one or more IP addresses (DNS A/AAAA, IPv4 literal, or IPv6 literal).
2. Opens a QUIC connection to a resolved IP on `port` (default `4433/udp`).
3. Proceeds with the §4 handshake.
4. Uses `agent-id` and `operation` (when present) inside the INVOKE frame's payload (defined in §3); these components are not transmitted at the QUIC layer.

### 6.4 Examples

| URI                                          | Interpretation                                        |
|----------------------------------------------|-------------------------------------------------------|
| `aire://node1.example.com`                   | host `node1.example.com`, default port 4433           |
| `aire://node1.example.com:5000`              | host `node1.example.com`, port 5000                   |
| `aire://10.0.0.5`                            | IPv4 literal `10.0.0.5`, port 4433                    |
| `aire://[2001:db8::1]:5000`                  | IPv6 literal, port 5000                               |
| `aire://node1.example.com/inbox`             | agent `inbox` on the host                             |
| `aire://node1.example.com/inbox/answer`      | operation `answer` on agent `inbox`                   |

### 6.5 Equivalence

Two AIRE URIs are equivalent if and only if their `host`, `port`, `agent-id`, and `operation` components match after URI normalization (RFC 3986 §6). DNS-name hosts are case-insensitive; `agent-id` and `operation` are case-sensitive.

### 6.6 Future work

Opaque node identifiers (host components without DNS resolution) and a standardized discovery mechanism are reserved for v0.2. v0.1 implementations rely on DNS or IP literals for addressing.

## 7. Cancellation semantics

*TODO (v0.3):* `CANCEL` frame kills a single Operation. If that operation has been delegated (via `DELEGATE`), the cancellation propagates to the delegate. Cancellation is best-effort but must be acknowledged within an implementation-defined deadline.

## 8. Budget and backpressure

*TODO (v0.3):* `BUDGET` frames are bidirectional. Senders advertise remaining budget (tokens, dollars, deadline). Receivers MAY refuse work that exceeds advertised budget. Budget is per-Operation, not per-Connection.

## 9. Resumability

*TODO (v0.4):* Operations may be resumable across connection loss. Resumable operations carry a resumption token issued by the server in `INVOKE` ACK. Client may reconnect (possibly to a different node via DNS-level migration) and present the resumption token to continue.

## 10. Security considerations

*TODO:* Threat model, replay protection, authentication, authorization, capability-scoped tokens, denial-of-service resistance.

## 11. Versioning

AIRE uses semantic versioning at the protocol level. Wire-incompatible changes bump the major version. Capability-additive changes bump the minor version. A node MUST refuse a `HELLO` from an incompatible major version.
