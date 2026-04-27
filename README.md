# AIRE

**Agent Interchange Runtime Envelope** — a QUIC-native protocol for agent-to-agent communication.

> Status: draft v0.1 · Reference implementation: [aire-go](../aire-go)

## The bet

The agent era is moving across HTTP. HTTP is wrong for agents — wrong for streaming, wrong for fan-out, wrong for migration, wrong for identity, wrong for cancellation, wrong for cost-aware backpressure. We can do better, and the right altitude is QUIC + agent-semantic frames, not a new L4.

AIRE is that protocol. Open spec, Apache 2.0, pluralist by design.

## Why now

Today's agent protocols — MCP, A2A, HTTP-based ACPs — all live on HTTP. That makes adoption easy and ceilings adoption hard:

|         | Transport               | Migration | Stream HOL       | Cancel        | Budget BP   | Identity        |
|---------|-------------------------|-----------|------------------|---------------|-------------|-----------------|
| MCP     | stdio / Streamable HTTP | ✗         | inherits HTTP    | crude         | ✗           | bolted OAuth    |
| A2A     | HTTP + SSE              | ✗         | inherits HTTP/2  | HTTP-cancel   | ✗           | bolted OAuth    |
| HTTP-ACPs | HTTP                  | ✗         | inherits HTTP    | ✗             | ✗           | bolted          |
| **AIRE** | **QUIC**               | ✓         | ✓                | ✓ semantic    | ✓ tokens+$  | ✓ DID per stream |

QUIC fixed the transport. AIRE fixes the semantics on top: connection IDs that survive host migration, multiplexed streams that don't head-of-line block, semantic cancellation per logical operation, identity bound to streams, capability negotiation, budget-aware flow control.

## Design at a glance

- **Transport:** QUIC (RFC 9000)
- **Frames:** typed agent verbs — `HELLO`, `CAPABILITY`, `INVOKE`, `STREAM`, `CANCEL`, `BUDGET`, `DELEGATE`, `ERROR`, `GOODBYE`
- **Identity:** DID-based, signed, bound to streams
- **Addressing:** `aire://node-id/agent-id/operation`
- **Backpressure:** semantic — tokens and dollars, not bytes
- **Cancellation:** per-stream, propagating to delegated sub-calls
- **Resumability:** logical-operation resume across new connections

See [SPEC.md](./SPEC.md) for the full draft.

## Status & roadmap

This is **draft v0.1**. Expect breaking changes until v1.0.

- v0.1 — frame types, handshake, basic semantics *(in progress)*
- v0.2 — capability negotiation, identity model
- v0.3 — budget / cancel / delegate semantics
- v0.4 — resumability, multipath
- v1.0 — frozen wire format, foundation donation

## Governance

Currently BDFL — see [GOVERNANCE.md](./GOVERNANCE.md). Path to a neutral foundation (Linux Foundation / IETF) once the spec is proven and adopted.

## License

Apache 2.0 — see [LICENSE](./LICENSE). Includes explicit patent grant.
