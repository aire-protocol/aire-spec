# AIRE Addressing — Design Memo

> **Status:** design proposal, pre-spec.
> **Author:** Etienne de Bruin.
> **Date:** 2026-05-08.
> **Scope:** proposed binding for §5 (Identity) and §6 (URI scheme) toward v0.2.
> **Relates to:** [aire-spec#6](https://github.com/aire-protocol/aire-spec/issues/6), [aire-spec#14](https://github.com/aire-protocol/aire-spec/issues/14).

This memo is **non-normative**. It exists to align on an addressing model before normative wording lands in `SPEC.md`. Once accepted, the substance moves into §5 and §6.

## 1. Question

Can AIRE provide a global, federated, human-friendly agent addressing scheme **without inventing a new standard**?

## 2. Recommendation

Yes — by *binding* four existing standards:

1. **W3C DIDs** — the cryptographic identity root.
2. **DID Document service entries** — discovery of AIRE endpoints, modeled on DIDComm's `DIDCommMessaging` (registered in the W3C DID Spec Registries).
3. **Atproto-style handle resolution** — human-friendly aliases (`agent@host.example.com`) resolving to DIDs via DNS TXT or HTTPS `.well-known`, with mandatory bidirectional verification.
4. **AIRE's existing §6 `aire://` URI** — the on-the-wire address.

AIRE's contribution is the **binding**: a few pages of normative spec describing how these pieces compose. AIRE invents no new identity, no new resolution mechanism, no new transport. Every layer has been deployed at internet scale (DIDComm, Bluesky, did:web).

## 3. The three layers

### 3.1 Identity (DID)

Every AIRE-addressable agent is identified by a **W3C DID**. The DID is the stable, cryptographic root. Handles change; DIDs do not.

- DID method is **not constrained** by AIRE. `did:web`, `did:key`, `did:plc`, `did:ethr`, etc. are all valid AIRE identities.
- The reference v0.2 implementation supports **`did:web`** by default (operationally simplest, DNS-rooted, matches AIRE's deployment shape) and **`did:key`** for ephemeral / development identities.
- The DID is what HELLO carries (§4) and what frames are signed against (§5). Handles never appear on the wire as identity — only as a lookup key off-wire.

`did:web` resolution recap (W3C CCG `did-method-web`):

| DID                                        | Resolves to                                          |
|--------------------------------------------|------------------------------------------------------|
| `did:web:example.com`                      | `https://example.com/.well-known/did.json`           |
| `did:web:example.com:agents:alice`         | `https://example.com/agents/alice/did.json`          |
| `did:web:example.com%3A8443:u:bob`         | `https://example.com:8443/u/bob/did.json`            |

Note: `.well-known` appears **only** in the zero-path case. Path-bearing DIDs do not include it.

### 3.2 Discovery (DID Document service entry)

The DID Document carries a `service` array. AIRE registers a new service type — call it **`AIREv1`** — analogous to DIDComm v2.1's `DIDCommMessaging`.

Proposed DID Document fragment:

```json
{
  "id": "did:web:aire.example.com:agents:summarizer",
  "service": [{
    "id": "did:web:aire.example.com:agents:summarizer#aire-1",
    "type": "AIREv1",
    "serviceEndpoint": [{
      "uri": "aire://aire.example.com:4433",
      "accept": ["aire/v0.2", "aire/v0.1"],
      "agentId": "summarizer"
    }]
  }],
  "alsoKnownAs": [
    "aire://summarizer@aire.example.com"
  ]
}
```

Field rationale:

- `type: "AIREv1"` — case-sensitive string. Once §5 is finalized, register at the W3C DID Spec Registries (same playbook DIDComm followed).
- `serviceEndpoint` — an **array of objects** (DIDComm v2.1 normalized this; we adopt the array-of-objects form from day one to skip the v2.0/v2.1 drift).
- `uri` — an `aire://` URI per §6, optionally including port. The canonical wire endpoint.
- `accept` — array of supported AIRE protocol versions. Lets clients pre-negotiate before opening QUIC.
- `agentId` — optional. If the DID resolves to a single agent, names it explicitly so resolvers don't need a separate hop. If absent, the resolver uses the agent identifier from the original input (handle local-part, URI path, etc.).
- `alsoKnownAs` — required when handles are in use. The `aire://localpart@host` form binds the DID back to the handle. **Bidirectional verification (§3.3) requires this.**

DIDComm precedent for routing keys / mediator indirection is **deferred** to a later AIRE version. v0.2 assumes direct connection to the listed `uri`.

### 3.3 Naming (handle resolution)

Handles are **mutable, human-friendly aliases** for DIDs. Modeled on atproto.

**Grammar:** `<localpart> "@" <domain>`. ASCII-only at v1; IDN deferred to avoid homograph attacks.

**Resolution methods** — clients SHOULD attempt both in parallel; either succeeding is sufficient:

1. **DNS TXT** at `_aire.<localpart>.<domain>`, value `did=<did>`.
   ```
   _aire.summarizer.aire.example.com.  IN  TXT  "did=did:web:aire.example.com:agents:summarizer"
   ```
2. **HTTPS well-known** at `https://<domain>/.well-known/aire-did?name=<localpart>`, response body the bare DID followed by a single newline. `Content-Type` is ignored.
   ```
   GET https://aire.example.com/.well-known/aire-did?name=summarizer
   200 OK
   did:web:aire.example.com:agents:summarizer\n
   ```

**Bidirectional verification is mandatory.** After handle → DID, fetch the DID Document and confirm `alsoKnownAs` contains `aire://<localpart>@<domain>`. If absent, the handle is invalid for that DID. Without this check, anyone with control of a domain can claim any DID.

**Rationale for choosing atproto's design over NIP-05's:**

| Concern                  | atproto                                          | NIP-05                                          |
|--------------------------|--------------------------------------------------|-------------------------------------------------|
| Bidirectional binding    | yes (`alsoKnownAs`)                              | no — vulnerable to DNS/host hijack              |
| DNS path                 | yes (`_atproto.<handle>`)                        | no                                              |
| Well-known body          | bare DID, no JSON                                | JSON with relays — easy to misimplement         |
| CORS                     | not needed (server-to-server)                    | required `Access-Control-Allow-Origin: *` — silently broken in many deployments |
| Identity layer           | DID                                              | bare pubkey                                     |
| Field record             | scaled to millions of users (Bluesky)            | smaller scale, more drift                       |

We adopt atproto's pattern with two adaptations: query-string at the well-known (`?name=`) instead of path, so a single domain can serve many agents under one well-known endpoint.

### 3.4 End-to-end resolution

Given handle `@summarizer@aire.example.com`, a client:

```
1. Resolve handle to DID:
     try DNS TXT: _aire.summarizer.aire.example.com
       → "did=did:web:aire.example.com:agents:summarizer"
     OR HTTPS:    GET https://aire.example.com/.well-known/aire-did?name=summarizer
       → "did:web:aire.example.com:agents:summarizer\n"

2. Resolve DID to DID Document:
     GET https://aire.example.com/agents/summarizer/did.json
       → { "id": "did:web:...", "service": [...], "alsoKnownAs": [...] }

3. Verify bidirectional binding:
     assert "aire://summarizer@aire.example.com" ∈ document.alsoKnownAs
     (else: handle does not authoritatively belong to this DID — ABORT)

4. Pick AIREv1 service entry:
     find service[type == "AIREv1"]
     → uri "aire://aire.example.com:4433", agentId "summarizer", accept ["aire/v0.2", "aire/v0.1"]

5. Connect:
     QUIC-dial aire.example.com:4433
     §4 handshake (HELLO with our DID; peer HELLO carries summarizer's DID)
     INVOKE on agentId "summarizer"
```

Steps 1–4 are off-wire (DNS / HTTPS). Step 5 is AIRE proper.

## 4. Proposed spec changes

| Section                  | Change                                                                                                                                                       | Issue                                  |
|--------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------|
| §5 Identity              | Define DID-based identity. Specify HELLO NodeID semantics: NodeID is a DID. Reference `did:web` and `did:key` as MUST-support for v0.2 reference impl.       | resolves [#6](../../../issues/6)       |
| §6.7 (new)               | DID Document service-type registration. Define `AIREv1` shape, fields, JSON example. Note future W3C DID Spec Registries submission.                        | new                                    |
| §6.8 (new)               | Handle resolution. Grammar, DNS TXT and HTTPS well-known methods, bidirectional verification.                                                                | revises [#14](../../../issues/14)      |
| §6.1 grammar (extend)    | Allow optional `userinfo` in the `authority` (i.e., `aire://localpart@host[:port]`) as a sugar form. Resolution unchanged: still ends with a §4 handshake.   | new                                    |
| §10 Security             | DNS-rooted trust caveats; bidirectional verification mandate; CORS lessons from NIP-05 (do not require, eliminate the surface); handle squatting → operator policy. | partial overlap with [#10](../../../issues/10) |

**Milestone proposal:** pull issue #14 (handles) from v0.4 to v0.2. Without handles, "talk to another agent by username" — the user-facing UX that motivates this whole memo — is not deliverable in v0.2. Handles are 80% of the value of the binding.

## 5. Open decisions

1. **DID method support floor for v0.2 reference impl.** Memo proposes `did:web` (MUST) + `did:key` (MUST). Should `did:plc` be MAY at v0.2 to keep door open for atproto-adjacent runtimes? No strong opinion.
2. **Sister spec or core.** Issue #14 raised this. Memo recommends **core** — handles are too central to AIRE's UX to defer. But: keeps the wire protocol (§2–§4) untouched; handles are §6 only, and §6 is already non-wire.
3. **`userinfo` in `aire://` URI.** Memo proposes allowing it as a sugar form. Alternative: keep §6 grammar untouched, treat `agent@host` strictly as a handle (off-wire), and require canonicalization to `aire://host/agentId` before any wire use. Cleaner separation but uglier in CLIs and docs. Recommend allowing userinfo.
4. **Well-known query-string vs path.** Memo proposes query-string (`?name=summarizer`) over path (`/.well-known/aire-did/summarizer`). Query-string lets one well-known endpoint serve many agents without per-agent routing config; path is more REST-cute but operationally heavier. Recommend query-string.
5. **`alsoKnownAs` URI form.** `aire://summarizer@aire.example.com` (handle form) vs `aire://aire.example.com/summarizer` (canonical form). Memo proposes handle form because that's what's being verified. Both could be allowed.
6. **IANA port assignment.** §6.2 currently uses 4433/udp by convention. Once the binding is stable, request IANA assignment in conjunction with the DID Spec Registries entry. Not blocking v0.2.

## 6. Tradeoffs

- **Pure-DID vs DID + handle.** Pure-DID is simpler; no DNS, no squatting, no handle-moderation surface. Handles are what humans actually use. Atproto's mutable-handle / immutable-DID split is the field-tested compromise. Memo picks the compromise.
- **DNS-rooted trust.** Whoever controls a domain controls handles under it. This is a feature (federated, no central registry) and a risk (DNS hijack = handle hijack). Bidirectional verification limits the blast radius — a hijacker can claim a handle but not impersonate an established DID. Acknowledge in §10.
- **Standards adoption lag.** A W3C DID Spec Registries entry takes time to finalize. The reference impl can use `AIREv1` immediately; the registry submission follows. DIDComm did the same — the type was deployed before the registry entry was final.
- **Doing nothing.** AIRE works today without addresses richer than `aire://host/agent`. Handles and DID identity are not blockers for the wire-level v0.1 conformance. The cost of waiting is that runtime adapters (Vega-side AIRE adapter and friends) ship without a friendly addressing model and reinvent it incoherently. Better to get the binding right now.

## 7. Prior art (citations)

- **DIDComm Messaging v2.1**, §Service Endpoint — https://identity.foundation/didcomm-messaging/spec/v2.1/#service-endpoint
- **W3C DID Spec Registries**, `DIDCommMessaging` entry — https://www.w3.org/TR/did-spec-registries/#didcommmessaging
- **atproto Handle Resolution** — https://atproto.com/specs/handle
- **NIP-05** — https://github.com/nostr-protocol/nips/blob/master/05.md
- **W3C CCG `did-method-web`** — https://w3c-ccg.github.io/did-method-web/
- **W3C DID Core 1.0** — https://www.w3.org/TR/did-1.0/

## 8. Next steps if accepted

1. Open a tracking issue: "Addressing binding: §5 identity + §6 service entry + §6 handles". Label `v0.2`. Reference this memo. Subsume #6; pull #14 forward.
2. Promote §3 of this memo into normative §5 and §6 wording in `SPEC.md`. Add JSON / TXT / handle examples as test vectors (analogous to §2.6, §4.7).
3. `aire-go` reference impl: a small `addressing/` package that takes a handle or DID and returns `(did string, endpoint string, agentId string, accept []string)`. Used by the Vega-side AIRE adapter.
4. After v0.2 ships: prepare a W3C DID Spec Registries PR registering `AIREv1`.
