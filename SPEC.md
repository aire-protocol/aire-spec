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

#### Vector 5 — CANCEL on operation 5

```
05 00 05 00
```

| Field      | Bytes | Decoded             |
|------------|-------|---------------------|
| Type       | `05`  | CANCEL (0x05)       |
| Flags      | `00`  | 0                   |
| OpID       | `05`  | 5 (1-byte varint)   |
| PayloadLen | `00`  | 0                   |
| Payload    | —     | empty (v0.1 CANCEL has no payload; reason codes are reserved for v0.3) |

#### Vector 6 — GOODBYE (connection-level)

```
09 00 00 00
```

| Field      | Bytes | Decoded                   |
|------------|-------|---------------------------|
| Type       | `09`  | GOODBYE (0x09)            |
| Flags      | `00`  | 0                         |
| OpID       | `00`  | 0 (connection-level)      |
| PayloadLen | `00`  | 0                         |
| Payload    | —     | empty                     |

#### Vector 7 — INVOKE with 8-byte varint OpID

Exercises the maximum varint length for OpID (here, 2⁶⁰).

```
03 00 D0 00 00 00 00 00 00 00 00
```

| Field      | Bytes                          | Decoded                              |
|------------|--------------------------------|--------------------------------------|
| Type       | `03`                           | INVOKE (0x03)                        |
| Flags      | `00`                           | 0                                    |
| OpID       | `D0 00 00 00 00 00 00 00`      | 1 152 921 504 606 846 976 (2⁶⁰)      |
| PayloadLen | `00`                           | 0                                    |
| Payload    | —                              | empty                                |

Total: 11 bytes.

### 2.7 Machine-readable test vectors

A canonical JSON representation of every vector in §2.6 (and §4.7) is published at [`vectors/v0.1.json`](./vectors/v0.1.json) in this repository. Conformance suites SHOULD load this file directly rather than re-encoding the hex tables above. The JSON schema:

```json
{
  "version": "v0.1",
  "vectors": [
    {
      "id": "empty-hello",
      "section": "§2.6",
      "description": "Connection-level HELLO frame with no payload",
      "encoded_hex": "01000000",
      "frame": {
        "type": 1,
        "type_name": "HELLO",
        "flags": 0,
        "op_id": 0,
        "payload_hex": ""
      }
    }
  ]
}
```

Every implementation MUST round-trip every vector byte-for-byte. New vectors are added as the spec evolves; the JSON is the authoritative wire-level conformance contract.

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

Every AIRE connection begins with a HELLO exchange on the *control stream*. The control stream is the first client-initiated bidirectional QUIC stream (stream ID `0` under QUIC's standard stream-ID numbering, RFC 9000 §2.1). Both peers MUST send exactly one HELLO frame as the first frame on the control stream, before sending any other frame on any stream.

A peer MUST NOT send a second HELLO on the same connection. A second HELLO MUST be treated as a protocol violation: the receiver emits ERROR with code `PROTOCOL_VIOLATION` (§4.6) and closes the connection.

### 4.1 HELLO frame payload

The HELLO frame's Payload (see §2.1) carries the following fields, in order:

```
+----------+----------+--------+----------+----------+
| VerMajor | VerMinor | NodeID | NumCaps  | Caps[]   |
| varint   | varint   | string | varint   | NumCaps× |
+----------+----------+--------+----------+----------+
```

| Field    | Type   | Description                                                                                                                           |
|----------|--------|---------------------------------------------------------------------------------------------------------------------------------------|
| VerMajor | varint | Major protocol version proposed by the sender (e.g., `0` for v0.x).                                                                   |
| VerMinor | varint | Minor protocol version proposed by the sender (e.g., `1` for v0.1).                                                                   |
| NodeID   | string | Sender's Node ID. Encoding per §4.2. Opaque to the protocol; interpretation defined by §5 (identity model).                           |
| NumCaps  | varint | Count of capability entries that follow.                                                                                              |
| Caps[]   | array  | `NumCaps` capability entries, each encoded per §4.3.                                                                                  |

### 4.2 String encoding

Strings within AIRE frame payloads are encoded as `<length: varint> <bytes: UTF-8>`. The length is the byte count of the UTF-8 representation, not the character count. The empty string is encoded as a single varint of value `0`.

This encoding is used wherever a frame payload field is typed as `string` in this specification.

### 4.3 Capability entry encoding

A capability entry consists of:

```
<name: string> <version: varint> <required: 1 byte>
```

| Field    | Description                                                                                                                       |
|----------|-----------------------------------------------------------------------------------------------------------------------------------|
| name     | Capability identifier in `reverse-domain.dotted` style, e.g., `core.streaming`, `com.example.budget.v2`. UTF-8.                   |
| version  | Capability version, varint.                                                                                                       |
| required | `0x01` if the sender requires the peer to support this capability; `0x00` if optional. Other values MUST be rejected as malformed. |

### 4.4 Version negotiation

Upon receiving the peer's HELLO:

1. **Major version check.** If `peer.VerMajor` does not equal the receiver's major version, the receiver MUST emit ERROR with code `INCOMPATIBLE_VERSION` and close the connection.
2. **Minor version selection.** The negotiated minor version is `min(local.VerMinor, peer.VerMinor)`. Both peers proceed using the negotiated minor version for the duration of the connection.

### 4.5 Capability negotiation

For each capability advertised by either peer:

- If both peers list the capability and the `version` values match exactly, the capability is **active** for the connection.
- If a peer lists a capability with `required = 0x01` and the other peer does not list it (or lists it with a non-matching version), the receiver MUST emit ERROR with code `MISSING_REQUIRED_CAPABILITY` and close the connection.
- Capabilities listed by only one peer with `required = 0x00` are **inactive** for the connection. They MUST NOT cause connection failure.

For v0.1, each capability advertises a single version; multi-version range matching is reserved for future minor versions.

### 4.6 Handshake error codes

| Code   | Name                          | Condition                                                |
|--------|-------------------------------|----------------------------------------------------------|
| `0x01` | `INCOMPATIBLE_VERSION`        | Major version mismatch in HELLO.                         |
| `0x02` | `MISSING_REQUIRED_CAPABILITY` | Peer required a capability the receiver lacks.           |
| `0x03` | `MALFORMED_FRAME`             | HELLO payload could not be parsed.                       |
| `0x04` | `PROTOCOL_VIOLATION`          | HELLO was not the first frame, or HELLO was sent twice.  |

These codes are carried in the `code` field of the ERROR frame (defined in §3).

### 4.7 Handshake test vector

A minimal HELLO frame at v0.1 from a node identified as `"node1"`, advertising one required capability `core.streaming` at version 1:

```
01 00 00 1A
00 01 05 6E 6F 64 65 31
01 0E 63 6F 72 65 2E 73 74 72 65 61 6D 69 6E 67 01 01
```

| Bytes                                     | Field                              |
|-------------------------------------------|------------------------------------|
| `01`                                      | Type = HELLO                       |
| `00`                                      | Flags = 0                          |
| `00`                                      | OpID = 0 (control stream)          |
| `1A`                                      | PayloadLen = 26                    |
| `00`                                      | VerMajor = 0                       |
| `01`                                      | VerMinor = 1                       |
| `05`                                      | NodeID length = 5                  |
| `6E 6F 64 65 31`                          | NodeID = `"node1"`                 |
| `01`                                      | NumCaps = 1                        |
| `0E`                                      | `cap[0].name` length = 14          |
| `63 6F 72 65 2E 73 74 72 65 61 6D 69 6E 67` | `cap[0].name` = `"core.streaming"` |
| `01`                                      | `cap[0].version` = 1               |
| `01`                                      | `cap[0].required` = `true` (0x01)  |

Total: 30 bytes.

After a successful handshake, peers MAY open additional QUIC streams to begin operations (§2.4).

## 5. Identity model

### 5.1 Identifiers

From v0.2 onward, the `NodeID` field in HELLO (§4.1) carries a **W3C Decentralized Identifier (DID)** as defined by [DID Core 1.0](https://www.w3.org/TR/did-1.0/). The DID is the stable, cryptographically anchored identity of the sender.

For v0.1 only, `NodeID` was opaque UTF-8. From v0.2 onward, receivers MUST validate that `NodeID` parses as a DID per DID Core syntax and MUST emit ERROR with code `MALFORMED_FRAME` (§4.6) on parse failure. v0.1 implementations interoperating with v0.2 nodes SHOULD migrate `NodeID` to a DID.

Identity is bound at the **frame level**, not the **connection level**. A single AIRE connection MAY carry operations on behalf of one or more agent identities; per-Operation identity is asserted by the agent's DID in the INVOKE payload (§3) and verified against signatures introduced in §5.4.

### 5.2 Required DID method support

A conforming AIRE v0.2 implementation MUST support both:

- **`did:web`** — for DNS-rooted, human-administrable identities. See [W3C CCG `did-method-web`](https://w3c-ccg.github.io/did-method-web/).
- **`did:key`** — for ephemeral, self-asserted, no-DNS-required identities. See [W3C CCG `did-method-key`](https://w3c-ccg.github.io/did-method-key/).

Implementations MAY support additional methods (`did:plc`, `did:ethr`, `did:ion`, …). Methods supported by an implementation MAY be advertised at handshake time as capabilities (§4.5) under the namespace `core.did-method.<methodname>`.

### 5.3 DID resolution

Resolution of a `did:web:<host>[:<port>][:<path-segments>]` proceeds per the `did-method-web` specification:

- `:` separators between path segments map to `/` in the resolution URL.
- With no path segments, fetch `https://<host>/.well-known/did.json`.
- With path segments, fetch `https://<host>/<seg1>/<seg2>/.../did.json` (no `.well-known`).
- A non-default port is conveyed by percent-encoding the colon in the DID: `did:web:example.com%3A8443` → `https://example.com:8443/.well-known/did.json`.
- HTTPS is mandatory; cleartext fetches MUST fail.

`did:key` resolution is purely algorithmic per `did-method-key` (no network fetch).

### 5.4 Signing and replay protection

*TODO (v0.2):* signature scheme (Ed25519), per-frame signing rules, replay protection. This sub-section is tracked separately from §5.1–§5.3 (which define naming-identity only) and will be filled in before v0.2 is finalized.

### 5.5 Examples

A v0.2 HELLO from a `did:web` node:

```
NodeID = "did:web:aire.example.com"
```

A v0.2 HELLO from an ephemeral `did:key` node:

```
NodeID = "did:key:z6MkpTHR8VNsBxYAAWHut2Geadd9jSwuBV8xRoAnwWsdvktH"
```

## 6. URI scheme

AIRE addresses are expressed as URIs with the `aire` scheme.

### 6.1 Grammar

The grammar follows RFC 3986. In ABNF:

```
aire-uri    = "aire://" authority [ "/" agent-id [ "/" operation ] ]
authority   = [ userinfo "@" ] host [ ":" port ]
userinfo    = 1*( unreserved / pct-encoded / sub-delims )
host        = <host as defined by RFC 3986 §3.2.2>
port        = <port as defined by RFC 3986 §3.2.3>
agent-id    = 1*pchar
operation   = 1*pchar
pchar       = <as defined by RFC 3986 §3.3>
```

`host` MAY be a DNS name, an IPv4 dotted-quad literal, or a bracketed IPv6 literal. `agent-id` and `operation` are case-sensitive UTF-8 strings, percent-encoded as required by RFC 3986.

When `userinfo` is present, the `userinfo "@" host` substring is a **handle** per §6.8 — a human-readable alias requiring off-wire resolution to a DID before connection. Canonical wire forms (e.g., the `uri` in a DID Document service entry, §6.7) MUST omit `userinfo` and address the resolved endpoint directly. Handles are sugar for documentation and CLI use; they are never authoritative on the wire.

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

### 6.6 Discovery

v0.1 implementations rely on DNS or IP literals for addressing. v0.2 introduces two layered discovery mechanisms on top of §6.1:

- **DID-based discovery** (§6.7): a DID resolves to one or more `aire://` endpoints via an `AIREv1` service entry in the DID Document.
- **Handle resolution** (§6.8): a human-readable `agent@domain` handle resolves to a DID, then to endpoints via §6.7.

The wire-level URI form in §6.1 is unchanged; §6.7 and §6.8 define how an off-wire identifier (DID or handle) becomes a wire-level URI.

### 6.7 DID Document service entry

An AIRE-addressable DID publishes its wire endpoint via a `service` array entry in its DID Document, of type **`AIREv1`**.

#### 6.7.1 Service entry shape

```json
{
  "id":   "<DID>#<fragment>",
  "type": "AIREv1",
  "serviceEndpoint": [{
    "uri":     "<aire-uri>",
    "accept":  ["aire/v0.2", ...],
    "agentId": "<agent-id>"
  }]
}
```

| Field             | Required | Description                                                                                                     |
|-------------------|----------|-----------------------------------------------------------------------------------------------------------------|
| `type`            | yes      | The exact case-sensitive string `"AIREv1"`.                                                                     |
| `serviceEndpoint` | yes      | Non-empty array of endpoint objects. The string form (DIDComm v2.0-style) is NOT supported.                     |
| `…[].uri`         | yes      | An `aire://` URI per §6.1. The host portion is the QUIC endpoint to dial. `userinfo` MUST NOT appear here.      |
| `…[].accept`      | yes      | Non-empty array of supported AIRE protocol version strings, format `"aire/v<major>.<minor>"`.                   |
| `…[].agentId`     | no       | The §6.1 `agent-id` when the DID resolves to a single agent. If absent, the resolver supplies it from input.    |

The `accept` array enables off-wire pre-negotiation; it does NOT replace the on-wire HELLO version negotiation (§4.4), which remains authoritative.

A DID Document MAY contain multiple `AIREv1` service entries (e.g., for redundancy across regions). Clients SHOULD attempt them in array order.

> Mediator routing (DIDComm-style `routingKeys`) is reserved for a future AIRE version. v0.2 assumes direct connection.

#### 6.7.2 Registry

A future revision will register `AIREv1` at the [W3C DID Spec Registries](https://www.w3.org/TR/did-spec-registries/), following the precedent of DIDComm Messaging's `DIDCommMessaging` entry. Until that submission lands, the type string is owned by this specification.

#### 6.7.3 Example DID Document

```json
{
  "id": "did:web:aire.example.com:agents:summarizer",
  "service": [{
    "id":   "did:web:aire.example.com:agents:summarizer#aire-1",
    "type": "AIREv1",
    "serviceEndpoint": [{
      "uri":     "aire://aire.example.com:4433",
      "accept":  ["aire/v0.2", "aire/v0.1"],
      "agentId": "summarizer"
    }]
  }],
  "alsoKnownAs": [
    "aire://summarizer@aire.example.com"
  ]
}
```

### 6.8 Handle resolution

A **handle** is a human-readable, mutable alias for a DID, of the form `<localpart>@<domain>`.

#### 6.8.1 Grammar

```
handle    = localpart "@" domain
localpart = 1*( ALPHA / DIGIT / "_" / "-" / "." )
domain    = <DNS name as defined by RFC 1035>
```

ASCII-only at v0.2. Internationalized handles (IDN) are reserved for a future revision to defer the homograph-attack surface.

#### 6.8.2 Resolution methods

A client resolving a handle to a DID MUST attempt at least one of the following methods. Implementations SHOULD attempt both in parallel and accept the first valid response.

**Method A — DNS TXT.**

```
QUERY:    _aire.<localpart>.<domain>  IN  TXT
RESPONSE: "did=<DID>"
```

The TXT record value MUST be the exact ASCII string `did=` followed immediately by the DID. If multiple TXT records are returned at the same name, resolution MUST fail.

**Method B — HTTPS well-known.**

```
GET https://<domain>/.well-known/aire-did?name=<localpart>
```

Response MUST be `200 OK` with body the bare DID followed by a single `\n` (LF). `Content-Type` is ignored. The query-string form lets a single well-known endpoint serve every agent under a domain. HTTPS is mandatory; cleartext requests MUST fail. The endpoint MUST NOT require CORS; it is a server-to-server resolution.

#### 6.8.3 Bidirectional verification (mandatory)

After a handle resolves to a DID, the client MUST resolve the DID's Document and verify that `alsoKnownAs` contains the handle's URI form:

```
"aire://<localpart>@<domain>"
```

If `alsoKnownAs` is absent, empty, or does not contain the handle URI, the handle MUST be treated as invalid for that DID and resolution MUST fail.

This bidirectional binding prevents impersonation: control of the handle's domain is necessary but not sufficient to claim an arbitrary DID. Without the binding, anyone with control of a domain could associate any DID with any handle they served.

#### 6.8.4 End-to-end resolution

```
Input handle: @summarizer@aire.example.com

1. Resolve handle → DID
   DNS TXT: _aire.summarizer.aire.example.com
     → "did=did:web:aire.example.com:agents:summarizer"
   (or HTTPS: https://aire.example.com/.well-known/aire-did?name=summarizer
     → "did:web:aire.example.com:agents:summarizer\n")

2. Resolve DID → DID Document
   GET https://aire.example.com/agents/summarizer/did.json

3. Verify bidirectional binding
   assert "aire://summarizer@aire.example.com" ∈ document.alsoKnownAs
   else: FAIL

4. Pick AIREv1 service entry
   service[type=="AIREv1"].serviceEndpoint[0]
     → uri "aire://aire.example.com:4433"
     → agentId "summarizer"
     → accept ["aire/v0.2", "aire/v0.1"]

5. Connect
   QUIC dial aire.example.com:4433
   §4 handshake: HELLO carries our DID; peer HELLO carries the agent's DID
   INVOKE on agentId "summarizer"
```

Steps 1–4 are off-wire (DNS / HTTPS). Step 5 is AIRE proper.

#### 6.8.5 Test vectors

Conforming resolvers MUST accept the following triple as a valid resolution of the handle `@summarizer@aire.example.com`.

**TXT record:**

```
_aire.summarizer.aire.example.com.  IN  TXT  "did=did:web:aire.example.com:agents:summarizer"
```

**Well-known response body** (43 bytes, ASCII):

```
64 69 64 3A 77 65 62 3A 61 69 72 65 2E 65 78 61
6D 70 6C 65 2E 63 6F 6D 3A 61 67 65 6E 74 73 3A
73 75 6D 6D 61 72 69 7A 65 72 0A
```

(`did:web:aire.example.com:agents:summarizer\n`.)

**Required `alsoKnownAs` entry in the resolved DID Document:**

```
"aire://summarizer@aire.example.com"
```

#### 6.8.6 Security note

Handle ownership equals domain ownership. DNS hijack or TLS compromise on the handle's domain enables handle takeover but, given §6.8.3 verification, does NOT enable impersonation of an existing DID — only creation of a new DID under attacker control. Threat-model details are deferred to §10.

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
