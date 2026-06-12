# AIRE Specification — Draft v0.1

> **Author:** Etienne de Bruin ([@etdebruin](https://github.com/etdebruin)).
> **Status:** Draft. Breaking changes expected until v1.0.
> **Last updated:** 2026-06-12.

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

Implementations MUST accept frames whose PayloadLen is up to 65 536 bytes (2¹⁶) without prior negotiation. Implementations MAY accept larger frames; senders SHOULD NOT exceed 1 048 576 bytes (2²⁰) without first negotiating a higher limit via a capability (§4.5). No such capability is defined in this revision; future minor versions may register one in §4.6. Receivers MAY emit ERROR with code `FRAME_TOO_LARGE` if a received frame exceeds their configured limit.

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

A canonical JSON representation of every vector in §2.6 (and §4.8) is published at [`vectors/v0.1.json`](./vectors/v0.1.json) in this repository. Conformance suites SHOULD load this file directly rather than re-encoding the hex tables above. The JSON schema:

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
| 0x03 | INVOKE      | client → server | Begin an Operation on a target Agent                       |
| 0x04 | STREAM      | both            | Streamed payload data for an active Operation              |
| 0x05 | CANCEL      | both            | Cancel a specific Operation (propagates to delegated sub-ops) |
| 0x06 | BUDGET      | both            | Budget update — tokens remaining, cost remaining, deadline |
| 0x07 | DELEGATE    | server → client | Forward an Operation to another Node                       |
| 0x08 | ERROR       | both            | Typed error frame (rate limit, budget, auth, etc.)         |
| 0x09 | GOODBYE     | both            | Graceful shutdown                                          |

Frame code `0x02` is reserved for a future mid-connection capability-update mechanism; v0.2 implementations MUST treat a received `0x02` frame as a protocol violation (see §4.5.5). Codes `0x0A–0x7F` are reserved for future versions of this specification. Codes `0x80–0xFF` are reserved for vendor extensions.

## 4. Handshake

Every AIRE connection begins with a HELLO exchange on the *control stream*. The control stream is the first client-initiated bidirectional QUIC stream (stream ID `0` under QUIC's standard stream-ID numbering, RFC 9000 §2.1). Both peers MUST send exactly one HELLO frame as the first frame on the control stream, before sending any other frame on any stream.

A peer MUST NOT send a second HELLO on the same connection. A second HELLO MUST be treated as a protocol violation: the receiver emits ERROR with code `PROTOCOL_VIOLATION` (§4.7) and closes the connection.

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

Capabilities declared in HELLO announce features the sender supports or requires. Negotiation produces an **active capability set** that both peers respect for the lifetime of the connection. The procedure on receiving the peer's HELLO is:

1. Validate each capability entry's syntax (§4.5.1). A malformed entry MUST cause ERROR `MALFORMED_FRAME` (§4.7) and close the connection.
2. Identify matching capabilities. Two entries refer to the same capability if and only if their full `name` fields (including the `/<major>` suffix per §4.5.1) are byte-for-byte equal. Different majors of the same namespace are different capabilities.
3. Compute the active set (§4.5.4). For each name advertised by *both* peers, the negotiated minor version is `min(local.minor, peer.minor)`.
4. Enforce required entries (§4.5.3). If either peer advertised a name with `required = 0x01` and that name is not in the active set, the receiver MUST emit ERROR `MISSING_REQUIRED_CAPABILITY` (§4.7) and close the connection.

After step 4 completes without error, the active set is fixed for the connection (§4.5.5).

#### 4.5.1 Naming

Capability names take the form `<namespace>/<major>`:

```
capability-name = namespace "/" major
namespace       = label *("." label)
label           = ALPHA *(ALPHA / DIGIT / "-")
major           = 1*DIGIT                ; no leading zeros except "0"
```

- `namespace` SHOULD be a reverse-DNS identifier the sender controls (e.g. `com.example.feature`). Reverse-DNS allocation is self-administered; no central registry is required for third-party capabilities.
- Names whose `namespace` begins with `aire.` (i.e. the first label is exactly `aire`) are **reserved** for capabilities defined in this specification. Implementations other than this specification MUST NOT advertise capabilities under the `aire.` namespace. Capabilities defined by this specification are listed in §4.6.
- `major` is a decimal integer without leading zeros, except that `0` MAY be used to mark experimental, pre-stable capabilities. Stable capabilities SHOULD begin at major `1`.
- The byte length of `name` (the full `<namespace>/<major>` string, UTF-8) MUST NOT exceed 255 bytes.
- Senders MUST emit names that conform to this grammar. Receivers MUST accept any UTF-8 byte sequence in the `name` field of a capability entry (§4.3): names that do not conform to the grammar are treated as unrecognized — they cannot match anything in the peer's advertisement and so will not appear in the active set. Such names still participate in the `required` check of §4.5.3.

#### 4.5.2 Versioning

A capability's identity has two parts: the **major** version embedded in the `name` (§4.5.1) and the **minor** version carried in the `version` field of the capability entry (§4.3).

- Different majors are different capabilities. `com.example.foo/1` and `com.example.foo/2` MUST NOT match.
- Within a given major, minor versions are **additive**: a sender advertising minor `N` MUST behave compatibly with every minor `0..N` of the same major.
- Both peers MUST operate at the negotiated minor `min(local.minor, peer.minor)`. A peer MUST NOT use functionality introduced in a minor higher than the negotiated minor.
- A peer that requires a specific minor's behavior MAY advertise that minor with `required = 0x01`; this only ensures the name+major is in the active set. If the negotiated minor is lower than the advertiser needs, the advertiser MUST decline to use the functionality and SHOULD close the connection at the application layer rather than transmit frames that depend on it.
- An implementation MAY advertise multiple majors of the same namespace in one HELLO (e.g. both `com.example.foo/1` and `com.example.foo/2`), in which case each major participates in negotiation independently.

#### 4.5.3 Required vs optional

The `required` byte (§4.3) declares whether the *advertising* peer needs the *receiving* peer to support the same name+major.

- `required = 0x01` — the advertiser cannot operate without the peer also advertising this name+major. If the peer did not advertise a matching name, the receiver MUST emit ERROR `MISSING_REQUIRED_CAPABILITY` and close the connection.
- `required = 0x00` — the advertiser will use the capability if the peer also advertised it, but will continue otherwise.
- Either peer setting `required = 0x01` is sufficient to fail negotiation when the other peer has not advertised the name. The check is symmetric: each peer evaluates the other's `required` entries against its own advertisement list.
- A name advertised by both peers with conflicting `required` values is still in the active set; the connection does not fail solely because the bits disagree.

#### 4.5.4 Active capability set

Both peers compute the same active set after negotiation: every `(name, negotiated-minor)` pair where `name` was advertised by both peers. The set is invariant for the lifetime of the connection. All protocol behavior conditioned on capabilities MUST consult the active set rather than raw HELLO contents.

- The order of capability entries in HELLO is **not** semantically significant. Senders SHOULD emit entries in lexical order of `name` to ease debugging; receivers MUST accept any order.
- A single HELLO MUST NOT contain two entries with the same full `name` (i.e. same namespace and same major). A receiver detecting a duplicate MUST emit ERROR `MALFORMED_FRAME` (§4.7) and close the connection. Multiple majors of the same namespace are distinct names and so are permitted.

#### 4.5.5 Mid-connection updates

v0.2 does not permit changes to the active capability set after the handshake. The set computed in §4.5.4 is the set for the connection's lifetime. Frame code `0x02` is reserved (§3) for a mid-connection update mechanism to be defined in a future minor version; a v0.2 receiver encountering a frame of type `0x02` MUST emit ERROR `PROTOCOL_VIOLATION` (§4.7) and close the connection.

### 4.6 Standard capabilities

This specification defines the following capabilities under the reserved `aire.` namespace. Implementations MAY advertise them in HELLO to signal support to a peer. Advertisement is optional; the capabilities listed here that have mandatory behavior elsewhere in the spec remain mandatory regardless of whether they are advertised.

| Capability                | Defined in | Notes                                                                                                            |
|---------------------------|------------|------------------------------------------------------------------------------------------------------------------|
| `aire.did-method.web/1`   | §5.2       | Resolution of `did:web` NodeIDs. Mandatory to implement (§5.2); advertisement is optional and informational.     |
| `aire.did-method.key/1`   | §5.2       | Resolution of `did:key` NodeIDs. Mandatory to implement (§5.2); advertisement is optional and informational.     |

Additional standard capabilities will be registered here as future minor versions of this specification land. Capabilities defined by parties other than this specification MUST be named under a namespace they control (§4.5.1) and MUST NOT use the `aire.` prefix.

### 4.7 Handshake error codes

| Code   | Name                          | Condition                                                |
|--------|-------------------------------|----------------------------------------------------------|
| `0x01` | `INCOMPATIBLE_VERSION`        | Major version mismatch in HELLO.                         |
| `0x02` | `MISSING_REQUIRED_CAPABILITY` | Peer required a capability the receiver lacks.           |
| `0x03` | `MALFORMED_FRAME`             | HELLO payload could not be parsed.                       |
| `0x04` | `PROTOCOL_VIOLATION`          | HELLO was not the first frame, HELLO was sent twice, or a reserved frame code (e.g. `0x02`) was received. |

These codes are carried in the `code` field of the ERROR frame (defined in §3).

### 4.8 Handshake test vector

A minimal HELLO frame at v0.1 from a node identified as `"node1"`, advertising one required capability `core.streaming` at version 1. The capability name in this vector predates the v0.2 naming convention (§4.5.1) and is preserved here for v0.1 conformance; v0.2+ senders MUST use the `<namespace>/<major>` form, and MUST NOT advertise capabilities under the reserved `aire.` namespace unless this specification defines them in §4.6.

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

For v0.1 only, `NodeID` was opaque UTF-8. From v0.2 onward, receivers MUST validate that `NodeID` parses as a DID per DID Core syntax and MUST emit ERROR with code `MALFORMED_FRAME` (§4.7) on parse failure. v0.1 implementations interoperating with v0.2 nodes SHOULD migrate `NodeID` to a DID.

Identity is bound at the **frame level**, not the **connection level**. A single AIRE connection MAY carry operations on behalf of one or more agent identities; per-Operation identity is asserted by the agent's DID in the INVOKE payload (§3) and verified against signatures introduced in §5.4.

### 5.2 Required DID method support

A conforming AIRE v0.2 implementation MUST support both:

- **`did:web`** — for DNS-rooted, human-administrable identities. See [W3C CCG `did-method-web`](https://w3c-ccg.github.io/did-method-web/).
- **`did:key`** — for ephemeral, self-asserted, no-DNS-required identities. See [W3C CCG `did-method-key`](https://w3c-ccg.github.io/did-method-key/).

Implementations MAY support additional methods (`did:plc`, `did:ethr`, `did:ion`, …). Methods defined by *this specification* are advertised under the reserved namespace `aire.did-method.<methodname>/<major>` and listed in §4.6. Methods defined by *other parties* MUST use a namespace the implementer controls (§4.5.1).

### 5.3 DID resolution

Resolution of a `did:web:<host>[:<port>][:<path-segments>]` proceeds per the `did-method-web` specification:

- `:` separators between path segments map to `/` in the resolution URL.
- With no path segments, fetch `https://<host>/.well-known/did.json`.
- With path segments, fetch `https://<host>/<seg1>/<seg2>/.../did.json` (no `.well-known`).
- A non-default port is conveyed by percent-encoding the colon in the DID: `did:web:example.com%3A8443` → `https://example.com:8443/.well-known/did.json`.
- HTTPS is mandatory; cleartext fetches MUST fail.

`did:key` resolution is purely algorithmic per `did-method-key` (no network fetch).

### 5.4 Signing and replay protection

AIRE v0.2 defines a frame-level signature mechanism that lets the receiver of selected frames cryptographically verify the sender controls the DID it claims. At v0.2, only HELLO is mandated to carry a signature; the wire format and verification rules are designed to extend to INVOKE, DELEGATE, and other frames in later minor versions without revisiting primitives.

#### 5.4.1 Algorithm

At v0.2, the sole signature algorithm is **Ed25519** (RFC 8032). Other algorithms are reserved. Algorithm identifiers are 1-byte codes:

| Code   | Algorithm   |
|--------|-------------|
| `0x01` | Ed25519     |
| others | reserved    |

#### 5.4.2 Key resolution

To verify a signature claimed by DID `D` and verification-method identifier `M`:

1. Resolve `D` to its DID Document (§5.3 for `did:web`; algorithmic for `did:key`).
2. Locate the verification method whose `id` equals `M` in the Document's `verificationMethod` array.
3. The verification method MUST be of type `Ed25519VerificationKey2020`. Decode its `publicKeyMultibase` (per the W3C Multikey conventions) to obtain the raw 32-byte Ed25519 public key.

If any step fails — DID unresolvable, method missing, type mismatch, decode failure — the receiver MUST emit ERROR `UNRESOLVABLE_DID` (§5.4.7) and close the connection.

For `did:key`, the DID itself encodes the public key directly; `M` is the canonical fragment form `<DID>#<multibase-identifier>` and resolution is purely local (no network fetch).

#### 5.4.3 Signature block

A signature appears in a frame's payload as a **signature block** appended after the frame's normal payload contents. The block layout:

```
+--------+----------+----------+-----------+---------+----------+
| AlgID  | VMID     | Nonce    | Timestamp | SigLen  | SigBytes |
| 1 byte | string   | 16 bytes | 8 bytes   | varint  | bytes    |
+--------+----------+----------+-----------+---------+----------+
```

| Field     | Encoding                                                                                              |
|-----------|-------------------------------------------------------------------------------------------------------|
| AlgID     | 1-byte algorithm identifier (§5.4.1).                                                                 |
| VMID      | Verification-method identifier in DID URL fragment form (e.g. `did:web:example.com#key-1`). UTF-8 string per §4.2. |
| Nonce     | Exactly 16 random bytes. Each signed frame MUST use a fresh nonce.                                    |
| Timestamp | Signed 64-bit two's-complement big-endian integer: Unix milliseconds at signing time.                 |
| SigLen    | varint length of `SigBytes` in bytes.                                                                 |
| SigBytes  | Algorithm-defined signature bytes. For Ed25519, exactly 64 bytes.                                     |

The block immediately follows the frame's domain-defined payload contents; the frame envelope's `PayloadLen` (§2.1) covers both the inner payload and the signature block. A receiver detects the block's presence by the frame's signing rules (§5.4.5) and by the presence of bytes remaining in the payload after parsing the inner payload.

#### 5.4.4 Signed message

The byte sequence over which the signature is computed (and verified) is:

```
domain_separator || inner_payload || meta
```

Where:

- `domain_separator` is the 13-byte sequence `"AIRE-SIG-v1\x00"` (12 bytes ASCII plus a NUL) concatenated with the 1-byte frame Type (§2.1, §3). This prevents a signature collected from one frame type being replayed against another.
- `inner_payload` is the bytes of the frame's payload *preceding* the signature block (i.e. the v0.1-style payload for HELLO).
- `meta` is the byte-for-byte serialization of `AlgID || encoded(VMID) || Nonce || Timestamp` exactly as it appears in the signature block, up to but not including `SigLen`. A verifier reconstructs `meta` by reading those fields directly from the block before checking the signature.

`SigLen` and `SigBytes` are *not* part of the signed material.

#### 5.4.5 Frame-specific signing rules

For v0.2:

| Frame      | Block presence  | Verification                                                                       |
|------------|-----------------|------------------------------------------------------------------------------------|
| HELLO      | MUST (v0.2+)    | The DID portion of `VMID` MUST be byte-for-byte equal to the HELLO's `NodeID`.    |
| INVOKE     | reserved (v0.3) | —                                                                                  |
| DELEGATE   | reserved (v0.3) | —                                                                                  |
| CANCEL     | MAY             | If present, the DID portion of `VMID` MUST equal the issuing peer's `NodeID`.      |
| Other      | MAY             | Application-defined.                                                               |

A v0.2 HELLO without a signature block MUST be rejected with ERROR `BAD_SIGNATURE` (§5.4.7). If the connection's negotiated minor (§4.4) is `0` — i.e. the peer is v0.1 — the HELLO follows the v0.1 format with no signature block, and v0.2 verification rules do not apply. Because the negotiated minor is not known when constructing one's own HELLO, a v0.2 implementation MUST include a signature block in its HELLO unconditionally; a v0.1 peer will fail to parse the trailing bytes and the resulting interop failure is acceptable. For HELLO, `inner_payload` is the entire v0.1-style HELLO payload (`VerMajor || VerMinor || NodeID || NumCaps || Caps[]`, §4.1).

If the VMID-DID equality check fails for HELLO or CANCEL, the receiver MUST emit ERROR `KEY_MISMATCH` (§5.4.7).

#### 5.4.6 Replay protection

Three classes of replay are addressed:

1. **Cross-frame replay** — a signature from one frame type being reused on another. Prevented by `domain_separator` in §5.4.4.
2. **Within-connection replay** — generally impossible: QUIC streams are reliable and ordered, so the same frame cannot be delivered twice. Implementations MAY treat a duplicate signature within a single connection as a protocol violation.
3. **Cross-connection replay** — addressed by `Nonce` and `Timestamp`:
   - `Timestamp` MUST be within ±300 seconds (5 minutes) of the receiver's UTC clock. Outside that window: ERROR `STALE_TIMESTAMP` (§5.4.7).
   - The receiver MUST maintain a cache of `(signing-DID, Nonce)` pairs seen within the past 300 seconds. A repeat: ERROR `REPLAYED_NONCE` (§5.4.7). Implementations SHOULD bound cache memory; a per-DID FIFO of the most recently seen nonces within the window is sufficient.

Implementations are responsible for clock sync (NTP or equivalent). Wide skew between peers manifests as `STALE_TIMESTAMP` and is a deployment concern, not a protocol bug.

#### 5.4.7 Error codes

The following codes extend the §4.7 handshake-error registry. They share the same code space; ERROR frames carry them in the same `code` field.

| Code   | Name                | Condition                                                                |
|--------|---------------------|--------------------------------------------------------------------------|
| `0x05` | `BAD_SIGNATURE`     | Signature failed to verify, or required signature block was missing.     |
| `0x06` | `STALE_TIMESTAMP`   | Timestamp outside the ±300-second window.                                |
| `0x07` | `REPLAYED_NONCE`    | `(DID, Nonce)` pair seen previously within the replay window.            |
| `0x08` | `UNRESOLVABLE_DID`  | DID could not be resolved, or its verification method was not usable.    |
| `0x09` | `KEY_MISMATCH`      | VMID's DID does not match the identity the frame is meant to assert.     |

#### 5.4.8 Test vector

A canonical signed-HELLO test vector is published at [`vectors/v0.2.json`](./vectors/v0.2.json). The vector includes:

- A fixed Ed25519 secret seed (and the corresponding `did:key` DID and public key).
- A v0.2 HELLO carrying that DID, one required capability (`aire.did-method.key/1`), a fixed 16-byte nonce, a fixed timestamp, and the resulting signature.
- The exact byte sequence covered by Ed25519 (domain separator + inner payload + meta) so implementations can verify or regenerate the signature without rebuilding it from scratch.

Conforming v0.2 implementations MUST verify the vector successfully and MUST reject a tampered copy (any single-bit alteration anywhere in the signed region).

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

The canonical **display form** of a handle prepends a single `@` to the grammar above — `@summarizer@aire.example.com` — mirroring fediverse and atproto convention. The leading `@` disambiguates a handle from an email address or bare DNS label in UX surfaces (CLI prompts, documentation, copy-paste flows) and is RECOMMENDED whenever a handle is rendered for human consumption. The leading `@` is not part of the grammar; tools normalizing user input MUST strip exactly one leading `@` if present before applying the grammar, and MUST NOT strip more than one. On the wire, handles never appear at all — only the resolved DID and `aire://` URI do (see §6.1 and §6.7).

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

### 7.1 v0.1 contract — stream close as cancellation

At v0.1, peers MAY cancel an in-flight operation by closing the corresponding QUIC stream — either gracefully (FIN before the operation's terminal frame) or abruptly (`STREAM_RESET`). A receiver observing unexpected stream termination for an active OpID MUST treat it as cancellation of that operation and abort the associated work, releasing any resources held on its behalf. No CANCEL frame is required at v0.1; the QUIC stream lifetime is the operation's lifetime, and closing the stream is the only cancellation primitive.

Because v0.1 has no in-band CANCEL frame, a peer wishing to cancel one operation but keep the connection open for others simply closes that one stream while leaving the connection's other streams intact. This works correctly under QUIC's stream-per-operation model (§2.4).

### 7.2 v0.3 contract — CANCEL frame

CANCEL provides in-band, per-operation cancellation that supersedes the v0.1 stream-close mechanism when the canceller wants to signal *why* the operation is ending and distinguish it from other forms of stream termination. The v0.1 stream-close cancellation (§7.1) remains valid forever — it is the QUIC-native default and any v0.3+ implementation MUST continue to treat unexpected stream termination as cancellation.

On receiving CANCEL for an active operation, a receiver MUST cease work on it, release any resources held on its behalf, and emit a terminal frame (the operation's normal terminal STREAM, or an ERROR with code `CANCELLED` per §7.2.6) before closing the stream's send side.

#### 7.2.1 CANCEL frame payload

```
+--------+-----------+----------+
| Reason | DetailLen | Detail   |
| varint | varint    | bytes    |
+--------+-----------+----------+
```

| Field     | Encoding                                                                                |
|-----------|-----------------------------------------------------------------------------------------|
| Reason    | Reason code (§7.2.2), varint.                                                           |
| DetailLen | Length of Detail in bytes (varint).                                                     |
| Detail    | Optional UTF-8 human-readable detail. MAY be empty. SHOULD NOT exceed 1024 bytes.       |

For backward compatibility with v0.1: a CANCEL frame with `PayloadLen = 0` (no Reason byte) is a legal "USER_CANCELLED, no detail" form, equivalent to a v0.3 CANCEL with `Reason = 0x00` and `DetailLen = 0`. v0.3 senders SHOULD include the Reason field explicitly; v0.3 receivers MUST accept both forms.

#### 7.2.2 Reason codes

| Code      | Name                 | Meaning                                                            |
|-----------|----------------------|--------------------------------------------------------------------|
| `0x00`    | `USER_CANCELLED`     | Caller explicitly cancelled (default when no reason is given).     |
| `0x01`    | `DEADLINE_EXCEEDED`  | The operation's deadline (§8) has passed.                          |
| `0x02`    | `BUDGET_EXCEEDED`    | The operation's budget (§8) has been exhausted.                    |
| `0x03`    | `UPSTREAM_CANCELLED` | A parent operation was cancelled; propagated via §7.2.4.           |
| `0x04`    | `RESOURCE_EXHAUSTED` | Server-side resource limit (memory, queue depth, …) hit.           |
| `0x05`    | `INTERNAL`           | Implementation-specific failure ended the operation.               |
| 0x06–0x3F | reserved             | Reserved for future minor versions.                                |
| 0x40+     | vendor               | Vendor-defined reasons. MUST be tolerated by receivers.            |

#### 7.2.3 Acknowledgement and timing

- CANCEL is fire-and-forget at the wire level. QUIC's stream ordering guarantees the receiver observes CANCEL after any previously-sent frames on the same operation.
- A receiver of CANCEL MUST emit a terminal frame for the operation within an implementation-defined deadline. The RECOMMENDED upper bound is **5 seconds**.
- A canceller SHOULD wait for the terminal frame before considering the operation complete; if no terminal frame arrives within the deadline, the canceller MAY close the stream abruptly (§7.1).
- A receiver that has already emitted a terminal frame for the operation MUST silently discard a subsequent CANCEL — the operation is already complete.

#### 7.2.4 DELEGATE propagation

When an operation O₁ has been delegated (§3, DELEGATE frame) to a sub-operation O₂, the delegating party MUST propagate cancellation downstream:

- On receiving CANCEL for O₁, the delegator MUST emit CANCEL on O₂'s stream with `Reason = UPSTREAM_CANCELLED`. The Detail field MAY carry O₁'s original Reason and Detail, or any additional context.
- Propagation is best-effort. The delegator MUST initiate the downstream CANCEL within the §7.2.3 deadline.
- Propagation is **transitive**: if O₂ itself delegated to O₃, the same rule applies recursively along the chain.
- A delegator that has already emitted a terminal frame for O₁ and subsequently observes a stale CANCEL MUST still attempt downstream propagation, since O₂ may not yet have completed.

#### 7.2.5 Test vectors

**Vector — CANCEL on operation 5 with reason USER_CANCELLED, no detail:**

```
05 00 05 02 00 00
```

| Bytes | Field        |
|-------|--------------|
| `05`  | Type = CANCEL |
| `00`  | Flags = 0    |
| `05`  | OpID = 5     |
| `02`  | PayloadLen = 2 |
| `00`  | Reason = USER_CANCELLED |
| `00`  | DetailLen = 0 |

**Vector — CANCEL on operation 10 with reason UPSTREAM_CANCELLED, detail "parent: ABORT" (13 bytes UTF-8):**

```
05 00 0A 0F 03 0D 70 61 72 65 6E 74 3A 20 41 42 4F 52 54
```

| Bytes        | Field |
|--------------|-------|
| `05`         | Type = CANCEL |
| `00`         | Flags = 0 |
| `0A`         | OpID = 10 |
| `0F`         | PayloadLen = 15 |
| `03`         | Reason = UPSTREAM_CANCELLED |
| `0D`         | DetailLen = 13 |
| `70 61 72 65 6E 74 3A 20 41 42 4F 52 54` | `"parent: ABORT"` |

The v0.1 zero-payload CANCEL vector (§2.6 Vector 5) remains valid under §7.2.1's compatibility rule.

#### 7.2.6 Operation error codes

The following codes extend the §4.7 / §5.4.7 error-code registry; they share the same code space and travel in the same `code` field of the ERROR frame.

| Code   | Name                  | Condition                                                                    |
|--------|-----------------------|------------------------------------------------------------------------------|
| `0x0A` | `CANCELLED`           | Operation terminated in response to a CANCEL frame.                          |
| `0x0B` | `BUDGET_EXCEEDED`     | Operation cannot continue without exceeding the §8 BUDGET advertised by the peer. |
| `0x0C` | `DEADLINE_EXCEEDED`   | Operation cannot continue without exceeding the §8 deadline.                 |
| `0x0D` | `DELEGATE_FAILED`     | Delegation to a downstream operation failed (see §7.2.4).                    |

## 8. Budget and backpressure

AIRE provides **semantic** backpressure on top of QUIC's byte-level flow control. A peer can declare how many tokens, how much money, and how much wall-clock time an operation may still consume; the other peer MUST honor the declaration. Budget is the actuals-aware counterpart to QUIC's bytes-aware window — they operate independently and both must be satisfied.

### 8.1 BUDGET frame

The BUDGET frame (§3, code `0x06`) carries a snapshot of the budget remaining for an operation. Either peer MAY send it at any time during the operation's lifetime. The frame's OpID identifies the operation it constrains.

BUDGET payload — a length-prefixed sequence of TLV entries:

```
+----------+----------+----------+
| NumEntry | Entry[0] | Entry[1] | …
| varint   |          |          |
+----------+----------+----------+
```

Each entry:

```
+----------+----------+----------+
| FieldID  | Length   | Value    |
| 1 byte   | varint   | bytes    |
+----------+----------+----------+
```

A BUDGET payload MUST contain at least one entry. Multiple entries with the same `FieldID` in a single BUDGET MUST cause the receiver to emit ERROR `MALFORMED_FRAME` (§4.7). Receivers MUST tolerate unknown `FieldID` values by reading `Length` and skipping `Length` bytes — unknown fields do not invalidate the frame.

### 8.2 Defined fields

| Code      | Name                       | Value encoding                                                          |
|-----------|----------------------------|-------------------------------------------------------------------------|
| `0x01`    | `TOKENS_REMAINING`         | varint — count of tokens the producer may still emit.                   |
| `0x02`    | `COST_MICROUNITS_REMAINING`| varint — cost remaining in micro-units of the currency in `0x03` (default USD if `0x03` is absent). |
| `0x03`    | `CURRENCY`                 | 3-byte ASCII ISO 4217 code (e.g. `"USD"`, `"EUR"`).                     |
| `0x04`    | `DEADLINE_MS`              | 8-byte big-endian signed int64 — UTC milliseconds at which the operation MUST be terminated. |
| 0x05–0x7F | reserved                   | Reserved for future minor versions.                                     |
| 0x80–0xFF | vendor                     | Vendor-defined. Receivers MUST tolerate (skip via `Length`).            |

Implementations MAY emit BUDGET with any subset of fields; missing fields mean "no constraint declared on this dimension." A peer that has not sent BUDGET is treated as having declared no constraints, subject to application-layer policy.

### 8.3 Scope

A BUDGET frame is **per-Operation**: it is sent on the operation's QUIC stream and applies only to that operation. Connection-level budget scope (e.g. "this entire connection has 100K tokens remaining") is reserved for a future minor version; it will be carried in a BUDGET frame with `OpID = 0` (control stream) when defined.

### 8.4 Update rules

- Either peer MAY send BUDGET at any time during the operation. There is no maximum frequency at the protocol level; implementations SHOULD batch updates to avoid wire churn.
- The **latest** BUDGET received is the authoritative snapshot. Older snapshots are discarded immediately on receipt of a newer one.
- BUDGET is **absolute, not delta**. Each frame carries the remaining budget, not a change. Senders that wish to delta-encode internally MUST translate to absolutes before emitting.
- A BUDGET that *increases* any field relative to the previous snapshot is valid — it represents a top-up. A BUDGET that decreases a field is also valid and immediately constrains the receiver.

### 8.5 Refusal and exhaustion

- A peer that has received BUDGET MUST NOT continue work that would exceed the snapshot in any declared dimension.
- A peer that detects exhaustion (continuing would exceed the snapshot) MUST cease work, emit a terminal frame, and either:
  - Send CANCEL with the matching reason code (`BUDGET_EXCEEDED` or `DEADLINE_EXCEEDED`, §7.2.2), or
  - Send ERROR with the matching error code (`BUDGET_EXCEEDED` or `DEADLINE_EXCEEDED`, §7.2.6).
- After a `BUDGET_EXCEEDED` outcome the sender that originally advertised the budget MAY emit a fresh BUDGET with higher values to grant headroom. If the operation has not yet been terminated by a terminal frame, the receiver MUST re-evaluate against the new snapshot.

### 8.6 Interaction with QUIC flow control

QUIC stream flow control (RFC 9000 §4) is byte-based and operates independently of AIRE BUDGET. Both layers apply concurrently:

- QUIC may stall a stream when the receiver's byte-window closes; AIRE BUDGET is unaffected.
- BUDGET may cause a producer to stop work even when QUIC has window available. This is the intended semantic: the producer is rate-limited by *what the work means* (tokens, dollars, time) rather than by *how big the bytes are*.

When both layers would stall the producer, the stricter wins. There is no protocol-level coordination between the two; implementations SHOULD report which layer caused a stall in operator-facing telemetry.

### 8.7 Test vectors

**Vector — BUDGET on operation 5 with 1000 tokens remaining, 5000 micro-USD remaining, currency USD:**

```
06 00 05 0E 03 01 02 43 E8 02 02 53 88 03 03 55 53 44
```

| Bytes              | Field |
|--------------------|-------|
| `06`               | Type = BUDGET |
| `00`               | Flags = 0 |
| `05`               | OpID = 5 |
| `0E`               | PayloadLen = 14 |
| `03`               | NumEntry = 3 |
| `01`               | Entry 1 FieldID = TOKENS_REMAINING |
| `02`               | Entry 1 Length = 2 |
| `43 E8`            | Entry 1 Value = varint(1000) |
| `02`               | Entry 2 FieldID = COST_MICROUNITS_REMAINING |
| `02`               | Entry 2 Length = 2 |
| `53 88`            | Entry 2 Value = varint(5000) |
| `03`               | Entry 3 FieldID = CURRENCY |
| `03`               | Entry 3 Length = 3 |
| `55 53 44`         | Entry 3 Value = `"USD"` |

**Vector — BUDGET on operation 7 with deadline 2026-12-31T23:59:59.999Z (UTC ms = 1798761599999 = `0x000001A2CE8BD3FF`):**

```
06 00 07 0B 01 04 08 00 00 01 A2 CE 8B D3 FF
```

| Bytes                          | Field |
|--------------------------------|-------|
| `06`                           | Type = BUDGET |
| `00`                           | Flags = 0 |
| `07`                           | OpID = 7 |
| `0B`                           | PayloadLen = 11 |
| `01`                           | NumEntry = 1 |
| `04`                           | Entry 1 FieldID = DEADLINE_MS |
| `08`                           | Entry 1 Length = 8 |
| `00 00 01 A2 CE 8B D3 FF`      | Entry 1 Value = int64 BE = 1798761599999 |

Both vectors are published in machine-readable form at [`vectors/v0.3.json`](./vectors/v0.3.json). Implementations MUST round-trip them byte-for-byte.

## 9. Resumability

*TODO (v0.4):* Operations may be resumable across connection loss. Resumable operations carry a resumption token issued by the server in `INVOKE` ACK. Client may reconnect (possibly to a different node via DNS-level migration) and present the resumption token to continue.

## 10. Security considerations

This section catalogs the threats AIRE's protocol-level mechanisms address, the threats it explicitly does not address, and the deployment-level assumptions implementations must hold. References point back to the normative section that defines each defense; §10 does not introduce new wire behavior.

### 10.1 Threat model

AIRE assumes a network attacker capable of observing, modifying, dropping, replaying, and injecting packets on the path between peers. AIRE does **not** assume the attacker controls the QUIC TLS root store, the DID resolution endpoints (DNS, HTTPS), or either peer's host. An attacker who compromises either peer's host cannot be defended against at the protocol layer.

The protocol explicitly does not provide:

- **Authorization.** What an authenticated peer is permitted to do is an application concern.
- **Audit / non-repudiation as an archival format.** Signatures (§5.4) cover individual frames; the spec defines no long-term attestation format.
- **Anonymity.** AIRE peers always carry an addressable identity (DID + endpoint) on the wire; AIRE is not an anonymity network.
- **Confidentiality beyond QUIC's.** Frame contents are protected only by the QUIC TLS session; once decrypted at either endpoint, content is plaintext to the application.

### 10.2 Transport security

QUIC (RFC 9000) mandates TLS 1.3 for connection establishment; AIRE inherits this and does **not** define a second transport-layer security mechanism.

- Implementations MUST verify the server's certificate against a configured trust root. The trust root is a deployment choice (Web PKI, private CA, key pinning); AIRE does not constrain it.
- For `did:web` resolution (§5.3) and handle resolution (§6.8.2), HTTPS is mandatory — cleartext fetches MUST fail.
- QUIC connection-ID migration MUST preserve the established TLS session; there is no "fresh handshake on migration" exposure.

### 10.3 Identity spoofing

A node's claimed identity is the `NodeID` field in HELLO (§4.1, §5.1). From v0.2 onward, `NodeID` is a DID and HELLO MUST be signed (§5.4.5). The signature binds HELLO bytes to the DID's verification key; an attacker who does not control the DID's private key cannot impersonate the node. A v0.2 receiver of an unsigned HELLO MUST reject it with `BAD_SIGNATURE` (§5.4.7).

`did:web` identity binding rests on DNS + HTTPS to the DID Document host. An attacker controlling both can substitute a verification key and impersonate the DID — this is the Web PKI trust assumption. Operators of high-value `did:web` identities SHOULD pin keys out of band or use a DID method (`did:key`, `did:plc`, …) that removes the DNS operator from the trust path.

`did:key` is algorithmic and not exposed to DNS hijack, but a `did:key` does not rotate — a leaked private key is a permanent compromise of that identity.

v0.1 has no signing primitive; v0.1 deployments wishing to authenticate peers MUST add an out-of-protocol mechanism (see Appendix A.2). v0.1 ↔ v0.1 connections without such a mechanism authenticate the peer's QUIC transport but not the agent identity claimed in HELLO.

### 10.4 Replay

Three replay surfaces:

1. **Within-connection.** Prevented by QUIC's stream ordering and TLS counters; impossible at the wire layer.
2. **Cross-frame, same signing key.** Prevented by the §5.4.4 domain separator binding a signature to a single frame type.
3. **Cross-connection.** Prevented by §5.4.6 — every signed frame carries a fresh nonce and a recent timestamp; receivers maintain a `(DID, nonce)` cache within the 300-second freshness window.

A network attacker who captures a v0.2 signed HELLO cannot replay it on a fresh connection more than 300 seconds later (timestamp window) and cannot replay it on the same connection (QUIC ordering). Deployments with tighter clock sync MAY shorten the window; loosening it widens the attacker's surface and is NOT RECOMMENDED.

Capability entries and the negotiated active set (§4.5.4) live inside the HELLO whose signature covers them, so capability advertisements cannot be tampered with in transit when signing is in use.

### 10.5 Handle and DID hijack

Handles (§6.8) are mutable aliases for DIDs and are rooted in the handle domain's DNS + HTTPS. A network attacker who controls the handle domain can cause a new handle to resolve to an attacker-controlled DID, or cause an existing handle to resolve to an attacker-controlled DID.

The §6.8.3 bidirectional binding (`alsoKnownAs` in the resolved DID Document MUST contain the handle's URI) prevents the *existing-DID* hijack: the attacker can claim a fresh DID under their own control but cannot hijack the binding of an established DID without also compromising that DID's Document.

DID hijack — taking over an established DID — is method-specific:

- `did:web` is vulnerable to combined DNS + TLS compromise of the Document host. Operators SHOULD use registrar locking, DNSSEC where deployable, and certificate-transparency monitoring.
- `did:key` cannot be hijacked but can be permanently compromised by private-key loss; rotation requires issuing a new DID.

Implementations SHOULD honor the `Cache-Control` directives of the HTTPS response for `did:web`; aggressive over-caching widens the window during which a stale Document with a revoked key remains accepted.

### 10.6 Capability and authorization

A capability (§4.5) advertises that a feature is *supported*. It is not a permission token. Specifically:

- An advertised capability does not grant the peer the right to use it; authorization is an application concern.
- Required capabilities (`required = 0x01`) force connection-level agreement on a feature, not on its use against any particular Operation.
- The capability registry (§4.6) and capability names (§4.5.1) are public; learning that a peer advertises `aire.foo/1` is not a confidentiality breach.

Implementations MUST NOT use capability presence as the sole authorization signal for sensitive operations. Use the authenticated DID plus out-of-band policy.

### 10.7 Denial of service

QUIC provides baseline DoS resistance — address validation, retry tokens, amplification limits — that AIRE inherits. AIRE adds the following surfaces:

- **HELLO signature verification.** Ed25519 verification is cheap (≈50 µs on modern CPUs) but unbounded inbound HELLOs from random peers still constitute a CPU drain. Implementations SHOULD rate-limit unauthenticated inbound connections per source address.
- **DID resolution.** A malicious HELLO carrying a `did:web` value can force the receiver to make an HTTPS request. Implementations MUST cache resolved DID Documents (respecting Cache-Control) and SHOULD rate-limit resolution per source.
- **Stream amplification.** A peer opening many concurrent streams forces the receiver to allocate per-stream state. QUIC's `MAX_STREAMS` transport parameter (RFC 9000 §4.6) is the primary control; implementations MUST set a finite limit and MAY lower it under load.
- **Replay-cache memory.** The §5.4.6 `(DID, nonce)` cache is bounded by the freshness window times nonce arrival rate. Implementations SHOULD bound cache memory per DID and evict FIFO; under attack, exceeding the bound MUST cause `REPLAYED_NONCE` rejection rather than uncontrolled memory growth.
- **Budget exhaustion (v0.3, forward reference).** §8 (BUDGET) lets a peer claim budget; a misbehaving peer claiming effectively infinite budget can drive the producer into uncapped work. Production deployments MUST cap accepted budgets at the application layer regardless of advertised values.

### 10.8 Observability and side channels

Frame envelopes (§2.1) leak frame types, OpIDs, and payload lengths to a network observer of the QUIC cipher-text layer. An attacker observing traffic patterns can infer:

- The rate and burst pattern of operations on a connection.
- Approximate payload sizes (which correlate with LLM turn boundaries or tool-call shapes).

Padding and traffic shaping are out of scope for v0.x. Deployments concerned about traffic analysis SHOULD layer that on at the application layer or via QUIC transport parameters.

ERROR codes related to authentication and key resolution (§5.4.7) SHOULD be returned without further distinguishing detail in production — e.g. a receiver SHOULD NOT separately signal "this DID does not exist" vs. "this DID has a different key" beyond what the four codes `BAD_SIGNATURE`, `UNRESOLVABLE_DID`, `KEY_MISMATCH`, and `STALE_TIMESTAMP` already encode, since finer-grained signals aid enumeration attacks.

### 10.9 Open security work

- **§7.2 (v0.3) CANCEL frame** exposes a small "did this cancellation arrive?" timing signal; threat is minor (cancellation is application-observable anyway) and will be reviewed when §7.2 is normalized.
- **§8 (v0.3) BUDGET frames** add the budget-exhaustion surface (§10.7) and will need their own threat-model elaboration when normalized.
- **§9 (v0.4) Resumability** introduces a long-lived resumption token whose theft would permit operation hijack; v0.4 will specify token entropy and validation requirements.
- Formal protocol analysis (e.g. Tamarin, ProVerif) of §5.4 signing + §4 handshake against an active network attacker is targeted before v1.0; it is not part of the v0.2 deliverable.

## 11. Versioning

AIRE uses semantic versioning at the protocol level. Wire-incompatible changes bump the major version. Capability-additive changes bump the minor version. A node MUST refuse a `HELLO` from an incompatible major version.

## Appendix A — Non-normative recommendations

The recommendations in this appendix are **non-normative**. Protocol behavior MUST NOT depend on compliance with them; they exist to give implementations a shared shape for adjacent concerns that are out of scope for the wire protocol itself.

### A.1 Operation cost accounting

§8 (BUDGET, v0.3) covers *prospective* accounting — declaring what an operation is allowed to consume. *Retrospective* accounting — recording what an operation actually consumed — has no wire-level convention in v0.2. To reduce ad-hoc divergence between implementations building audit logs, chargeback, or marketplace settlement, the following shape is RECOMMENDED.

When an operation's work has a meaningful unit cost (LLM tokens, compute time, paid API calls, …), the producing peer SHOULD include an `accounting` object in the operation's terminal frame payload (a final STREAM, or an ERROR frame) with the following keys, omitting any that do not apply:

| Key           | Type                                              | Meaning                                                  |
|---------------|---------------------------------------------------|----------------------------------------------------------|
| `tokens_in`   | unsigned integer                                  | Input tokens consumed.                                   |
| `tokens_out`  | unsigned integer                                  | Output tokens produced.                                  |
| `cost`        | object `{ amount: number, currency: string }`     | Monetary cost. `currency` is an ISO 4217 code.           |
| `duration_ms` | unsigned integer                                  | Wall-clock duration of the operation in milliseconds.    |

Receivers MAY surface these values for audit, chargeback, or analytics. Protocol behavior MUST NOT depend on them — they are informational.

This recommendation is intentionally orthogonal to BUDGET (§8): a receiver may observe BUDGET frames advertising what was allowed without ever receiving an `accounting` object, and may receive an `accounting` object on an operation that never advertised a budget. The two together let a peer reconcile commitments against actuals end-to-end without inventing a third channel.

The encoding of `accounting` within a frame payload is left to the application — JSON, CBOR, protobuf, or another scheme negotiated out of band. A future minor version may promote this to a normative payload field once production deployment confirms the shape.

### A.2 Authentication on a v0.1 connection

At v0.1, NodeID is opaque UTF-8 and frames are unsigned; there is no protocol-level authentication primitive. v0.2 supersedes this with DIDs (§5) and Ed25519 HELLO signing (§5.4). Implementations still operating over a v0.1 wire format — for instance, a stable runtime that has not yet migrated its peer set — need *some* way to authenticate the peer on the other end of a HELLO. Absent guidance, every v0.1 implementer will roll their own incompatible variant. This appendix documents one workable pattern; it is **non-normative** and explicitly **superseded by §5 + §5.4 once v0.2 is reachable**.

#### A.2.1 Capability-gated auth handshake

Both peers advertise a custom capability (under a namespace they control per §4.5.1) with `required = 0x01`. A typical name is `com.example.shared-secret/1`. Because the bit is required, an unauthenticated peer that omits the capability fails the §4.5 active-set check and the connection closes before any operation can run. The capability advertises the *intent* to authenticate; the actual handshake runs in a dedicated post-handshake operation.

Sketch:

```
1. QUIC connect → AIRE §4 handshake.
   Both peers' HELLOs include {com.example.shared-secret/1, required: true}.
   §4.5 negotiation succeeds → both sides know auth is expected.

2. The initiating peer opens an Operation against the well-known agent
   "_aire/auth" with operation name "challenge".

3. The acceptor returns, in a STREAM frame, a fresh 16-byte nonce and an
   8-byte timestamp.

4. The initiator returns, in a STREAM frame:
       HMAC-SHA256(shared_secret,
                   peer_node_id || nonce || timestamp || direction)
   where direction is a single byte distinguishing initiator-MAC from
   acceptor-MAC. The same exchange runs symmetrically (acceptor → initiator)
   to mutually authenticate.

5. Both sides verify the HMAC. On success, the connection is marked
   authenticated; both sides accept subsequent INVOKE frames. On failure,
   either side MUST emit ERROR and close the connection.
```

#### A.2.2 Replay protection

- The acceptor MUST reject a `timestamp` outside a ±5-minute (300 s) freshness window relative to its UTC clock.
- The acceptor MUST cache `(peer_node_id, nonce)` pairs seen within the window and reject repeats.
- The shared secret MUST be at least 256 bits and exchanged out of band (operator-administered).

#### A.2.3 Migration path to v0.2

Implementations adopting v0.2 SHOULD drop this pattern in favor of §5.4 HELLO signing, which provides equivalent (and stronger) per-frame identity verification at the protocol level and removes the need for a post-handshake auth round-trip. A v0.2 peer connecting to a v0.1 peer that still advertises a shared-secret capability MAY continue to honor it for compatibility during the transition; once both ends are on v0.2 the capability MUST be dropped.

The exact HMAC inputs, nonce length, freshness window, and capability namespace are implementation choices. The interop-relevant decisions documented here are (a) that the existing `required` flag of §4.3 is the right signal to commit both sides to running an auth exchange, and (b) that the exchange itself runs as a normal Operation, since v0.1 capability entries carry no payload room for inline challenge bytes.
