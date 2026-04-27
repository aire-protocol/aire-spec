# AIRE Specification — Draft v0.1

> **Author:** Etienne de Bruin ([@etdebruin](https://github.com/etdebruin)).
> **Status:** Draft. Breaking changes expected until v1.0.
> **Last updated:** 2026-04-26.

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

*TODO (v0.1):* Frame envelope encoding, length prefixes, varints, version byte. Target: self-delimiting frames so multiple frames can share a QUIC stream cleanly.

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

```
aire://<node-id>[/<agent-id>[/<operation>]]
```

- `aire://` — protocol scheme
- `<node-id>` — opaque, globally unique node identifier
- `<agent-id>` — agent identifier scoped to the node
- `<operation>` — optional named operation

DNS resolution and discovery: *TODO*.

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
