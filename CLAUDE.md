# CLAUDE.md

Guidance for Claude Code (and other agents) working in the AIRE specification repo.

## Purpose

This repo holds the AIRE protocol specification. **The spec IS the product.** Wire-format and semantic decisions live here first, in `SPEC.md`. Implementations follow the spec — never the other way around.

## Working principles

- **The spec leads.** Don't propose changes that haven't been specced. If you're tempted to add behavior in an implementation, draft it in `SPEC.md` first.
- **No normative references to specific implementations.** Vega, MCP, A2A, etc. may appear in non-normative comparison tables and examples. Never in normative MUST/SHOULD/MAY clauses.
- **RFC 2119 keywords.** When normative, use **MUST / MUST NOT / SHOULD / SHOULD NOT / MAY** in capitals. Tie them to the section you're in (e.g., "An AIRE node MUST send a HELLO frame before any other frame").
- **Wire-format changes require test vectors.** No frame-format change ships without canonical byte-level vectors in the appendix. Implementations conform by matching these.
- **Section numbering is stable.** §1 Introduction · §2 Wire format · §3 Frame types · §4 Handshake · §5 Identity · §6 URI scheme · §7 Cancellation · §8 Budget · §9 Resumability · §10 Security · §11 Versioning. Don't renumber.

## Versioning rules

- **v0.x:** breaking changes expected. Document in commit messages.
- **v1.0:** wire format frozen. Wire-incompatible changes bump major.
- **Minor bumps** (v1.1, v1.2…) are for capability-additive changes only — new optional frame fields, new optional capabilities. Anything that an old implementation would silently mishandle is a major bump.

## How to add or change a section

1. Open an issue first. Wire-format and semantic decisions need air time before they land.
2. Update `SPEC.md` with the change.
3. If wire-format-affecting, add canonical test vectors to the appendix.
4. Cross-reference from related sections.

## Style

- Numbered subsections (§2.1, §2.2…)
- Tables for frame-type registries, comparison data
- Code-fenced blocks for wire-format byte layouts and example URIs
- Don't be cute. Specs are dry on purpose.

## Governance

See `GOVERNANCE.md`. BDFL approves spec changes during v0.x. Issues are the medium for proposals; PRs land changes.

## Apache 2.0 / patent grant

By contributing, you grant a patent license under Apache 2.0 §3. If a contribution requires patents you cannot license under those terms, do not submit it.
