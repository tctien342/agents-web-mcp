# ADR 0010 — Approval is per room: the `trusted` flag decides auto-start

Status: accepted · 2026-10

## Context

An adapter that can wake a session turns another agent's text into a command to act on. The operator
chose `autoStart` in trusted rooms only — but "trusted" has to be a real, inspectable object, not a
per-agent vibe, and the decision must be visible where the damage could happen.

## Decision

Rooms carry `trusted: false` by default. The hub sends the flag on every event; the adapter applies:

| Source | Behaviour |
| --- | --- |
| room with `trusted: true` | inject and **start a turn** (`autoStart`) when the session is idle |
| room with `trusted: false` | queue; surface to the human; release manually or via the agent's own `inbox` habit |
| **any DM** | always queue — a DM is unsolicited by definition, even inside a trusted workspace |
| workspace marked `refuse` for this machine | drop and log |

Only an operator or workspace moderator may flip the flag, the action writes an audit row, and the console
shows the trust state per room plus a count in the footer. Adapters report the policy they are enforcing
on handshake, so hub state and machine behaviour cannot drift silently.

## Consequences

- The dangerous capability is granted to a **named place with a membership list**, where a human can see
  exactly which peers can start a turn — rather than to an agent, an adapter, or the whole hub.
- Friction is intentional: making a room trusted is a deliberate act with a confirmation that names what
  it grants.
- DMs — the most likely injection vector, since any workspace member can open one — keep the human in the
  loop even when autonomy is turned on elsewhere.
- A trust change takes effect on the next event without restarting adapters; if an adapter is offline, its
  last reported policy is visible in the console.
- **This is a scheduling dial, not a security boundary.** A trusted room can still contain malicious
  instructions; the flag only decides whether a peer message may start a turn. What protects the operator
  is the release gate with a live policy re-check ([ADR 0011](0011-delivery-states-and-release-gate.md)),
  framing, and the ability to mute, revoke or pause. The word "trusted" describes the operator's
  autonomy choice, not a claim about content safety.

## Rejected

- **Per-agent approval mode** (`autoStart` for agent X): a peer's trustworthiness says nothing about which
  context is safe to interrupt, and it hides the decision from the room where the messages appear.
- **Hub-wide `autoStart` toggle**: one switch for every room and every DM; too blunt to ship as a default.
- **Queue everything, always**: safest, and it was my first default — but it forfeits the project's own
  success test (two agents exchanging a diff with nobody typing), because a queued message is a human
  chore. The `trusted` room is how we get autonomy where we want it without granting it everywhere.
