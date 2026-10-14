# ADR 0003 — Delivery: pull-first over MCP, push via harness adapters

Status: accepted · 2026-10

## Context

The value of the hub is a message reaching a *waiting* agent. MCP's model is request/response driven by
the client's model: a tool call happens only when the model decides to make one. Investigating current
harness behaviour (2026-10) established a distinction worth keeping straight — MCP does define
server→client messaging, but what this project needs is a host that turns a server signal into model
context or a new turn, and that is the part that is missing:

- MCP 2026-07-28 steers away from push-style primitives: Roots/Sampling/Logging deprecated, MRTR replaces
  server-initiated elicitation/sampling, HTTP+SSE as an MCP *transport* deprecated.
- Codex receives `notifications/message` and routes it to stderr — the model never sees it.
- Claude Code: a plain HTTP MCP server pushes nothing; its "channels" extension
  (`notifications/claude/channel`) is stdio-only, a research preview, and its cross-session inbox socket
  is machine-local.
- pi can inject into a live session only from inside the process (an extension calling
  `pi.sendUserMessage()` / `pi.sendMessage(..., {triggerTurn:true})`).

## Decision

Three layers, with (a) guaranteed for everyone:

1. **Pull**: `inbox(after_seq, wait_ms)` over MCP — long-poll, restart-safe, works in every harness.
2. **Adapter push**: per-machine process subscribed to our own HTTP/SSE `/events`, injecting into the
   live session through whatever that harness supports, with fixed provenance framing, per-room approval
   policy, and a single release gate that re-checks policy before injecting
   ([ADR 0011](0011-delivery-states-and-release-gate.md)). This is the delivery mechanism the project is
   really about.
3. **MCP push notifications**: not used. Kept as a possible future path only if a client's support is
   verified.

The adapter feed is deliberately **our own HTTP protocol**, not MCP, because MCP cannot express it and
because the harness process — not the model — must own the connection.

`setup_guide` is the entry point: it returns a harness-specific playbook naming the sink, the approval
mode, and the delivery class (`wakeable` / `turn-boundary` / `poll-only`).

## Consequences

- One `seq` stream + durable cursors give at-least-once delivery across hub restarts and network drops.
- We own a small protocol surface (`/events`, `/api/inbox/poll`, `/api/heartbeat`, `/api/adapter/hello`)
  in addition to MCP. Accepted: it is the only way to get inbound delivery.
- Per-harness glue is unavoidable and version-fragile; sinks are isolated packages whose claims are
  tiered and re-verified per sink — pi at M2, the rest at M5
  ([ADAPTERS.md §5](../ADAPTERS.md#5-open-risks-to-re-verify-before-shipping-each-sink)).
- **pi-first framing is part of the decision:** pi is the one harness with a documented in-process
  injection API, so the product promises reliable delivery there and states a weaker delivery class
  elsewhere rather than implying parity.
- Honesty requirement: on Codex and most other harnesses an *idle* agent cannot be woken, so "another
  agent will answer you" must be qualified per harness.

## Rejected

- **Stateful MCP sessions + server notifications**: needs sessions the 2026 spec is moving away from, and
  clients that act on unsolicited notifications; verified support is limited to a stdio-only research
  preview in one harness.
- **Poll-only, no adapters** (an `AGENTS.md` instruction and nothing else): least machinery, but the
  success test — two agents exchanging a diff without a human typing — does not pass.
- **A2A as the agent-facing transport**: right shape for the data model, but no harness will speak to our
  A2A server unprompted; MCP is what harnesses actually connect to. See [ADR 0008](0008-align-message-model-with-a2a.md).
