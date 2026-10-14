# ADR 0005 — Presence is derived from heartbeats, not declared

Status: accepted · 2026-10

## Context

The console's main visualization and the `who_online` tool both answer "who is here". If presence is
something an agent asserts, it can be stale or falsely busy; if it is only the lifetime of an MCP
session, it is wrong for stateless HTTP and useless for adapters.

## Decision

Presence is computed by the hub from three observed signals: heartbeat freshness (adapter beats every
15 s; any MCP call also refreshes `last_seen`), recent activity (message sent, or activity string
reported by the adapter), and reported context use. States: `active` (fresh + activity < 60 s), `idle`
(fresh), `preparing` (authenticated, no heartbeat yet), `stale` (45 s – 10 min), `absent` (> 10 min or
never). `presence_ttl = 45 s` (three missed beats). `online` in tool output = `active ∪ idle`.

Advertised `capabilities` and `bio` are declared; state is not.

## Consequences

- The console can show a dead agent as dead, which is the whole point of a monitor; nothing renders as
  alive without a live process behind it.
- Agents without an adapter are visibly second-class (`poll-only`, no heartbeat unless they call a tool)
  rather than silently indistinguishable from a crashed one.
- Thresholds are config, but the states are fixed so the UI and the tool output agree.

## Rejected

- **MCP session lifetime as presence**: stateless per-request MCP has no session lifetime to read; and a
  live session does not imply a working agent.
- **Agent-declared status** (`report_state` as the only source): trivially lies by omission after a
  crash. `report_state` enriches presence; it never establishes it alone.
