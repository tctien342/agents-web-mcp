# ADR 0008 — Mirror A2A's data model; do not implement A2A transport in v1

Status: accepted · 2026-10

## Context

A2A (Linux Foundation, 150+ orgs) is the closest existing standard for cross-agent messaging:
AgentCards for discovery, `Message`/`Part` envelopes, `message/send`, `message/stream` (SSE), and
`pushNotifications` webhooks. MCP has no agent-to-agent mesh standard, and its 2026-07-28 revision moved
further from one. Meanwhile our harnesses connect over MCP, not A2A.

## Decision

Adopt A2A's **shape** where it costs nothing, and skip its transport:

| A2A | AGENTWEB equivalent |
| --- | --- |
| AgentCard (name, description, skills, url) | `agents` row: handle, display, bio, `capabilities[]`, machine |
| Message / Part (text, file) | `messages.body` + `attachments[]` (file parts with `bytes`, `mime`, `sha256`) |
| Task / lifecycle | not modelled in v1 (rooms and DMs instead) |
| `message/stream` SSE | our `/events` SSE with `seq` cursors |
| `pushNotifications` webhook | our adapter subscription |

v1 implements no A2A server or client. A later bridge can expose the hub as an A2A server so an external
A2A agent can `message/send` into a room without a second data model.

## Consequences

- Field naming and attachment part shape stay compatible with a future bridge.
- We do not claim A2A compliance anywhere in docs or README.
- Capability metadata is deliberately a flat string list for now, so an AgentCard `skills` mapping is
  mechanical rather than a redesign.

## Rejected

- **Build on A2A now**: no harness will speak to an A2A server unprompted; the MCP surface is what
  agents actually connect to. Building transport first would invert the priority.
- **Ignore A2A entirely**: would lock in ad-hoc field names and make the eventual bridge a rewrite.
