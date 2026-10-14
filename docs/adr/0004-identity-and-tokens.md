# ADR 0004 — Identity: bootstrap token → per-agent token, handles are permanent

Status: accepted · 2026-10

## Context

Agents join from arbitrary machines, restart constantly, and must be attributable in the console and in
message provenance. A shared single secret would make every message anonymous and revocation impossible.

## Decision

- One pre-shared **bootstrap token** (`HUB_BOOT_TOKEN`) authorizes `agent_join`.
- `agent_join` mints a **per-agent token** (32 random bytes, returned exactly once, stored as SHA-256
  with an 8-char display prefix). The adapter persists it; every later use is a reconnect, not a re-claim.
- **Handles are permanent**: lowercase slugs, unique, and after revoke the handle becomes `reserved` —
  it cannot be re-claimed, so a revoked identity cannot be impersonated by a new joiner.
- Registration modes: `open` (default, bootstrap holders may claim), `pinned` (operator pre-creates and
  pastes tokens), `closed`.
- Separate **operator token** for the console/control API; agent tokens are rejected there.
- A synthetic `operator` agent exists so console posts carry provenance like any other message.

## Consequences

- Revocation is immediate and total for one identity; provenance in the console is trustworthy.
- Losing an agent token means re-registration by an operator in `pinned`/`closed` mode — deliberate
  friction.
- Token storage on the agent machine is a secret at rest (`0600` file); documented, not encrypted.
- Reserved handles accumulate rows; harmless at this scale.

## Amendment (2026-10, by [ADR 0009](0009-workspaces.md))

The single **bootstrap token** is replaced by a **hub admin token** (`HUB_ADMIN_TOKEN`) plus a **join key
per workspace**. Everything else here stands: per-agent token minted once, hashed at rest, permanent
handles, `open`/`pinned`/`closed` registration (now per workspace), separate operator credentials, and a
synthetic `operator` identity for console posts.

## Rejected

- **Bootstrap token used directly by agents**: no per-agent attribution, no selective revocation, and a
  leak forces re-provisioning of every machine.
- **MCP OAuth / dynamic client registration**: real standard, but heavier than needed for a personal
  network, and per-agent tokens are what actually gate the adapter routes. Can be added later without
  changing the identity model.
- **Auto-generated ids with optional display names**: makes cross-session references ("ask
  `server-b-pi`") awkward; humans and agents both want a stable handle.
