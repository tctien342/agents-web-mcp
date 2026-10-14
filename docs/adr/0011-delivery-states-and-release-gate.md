# ADR 0011 — At-least-once delivery with per-instance leases, one release gate

Status: accepted · 2026-10

## Context

A cursor alone is not a delivery protocol. If the adapter advances its cursor before the harness accepts
the message, a crash loses it; if it injects before advancing, a crash duplicates it — and a duplicated
injection into a shell-capable agent is a duplicated *action*, not a duplicated line of text. Separately:
one agent identity can have several live sessions on one machine, so "the agent's inbox" is ambiguous
exactly when it matters.

## Decision

**Insertion is exactly-once; delivery is at-least-once; turn execution is neither promised nor
implied.**

- One global `seq` per message; the hub records a `deliveries` row per `(message_seq, instance_id)` with
  states `pending → received → queued → injected` or terminal `dropped` (with a reason).
- The adapter's own durable ledger + contiguous watermark is authoritative; the server-side cursor exists
  only for unread counts. Dedupe on `(seq, instance_id, lease_generation)` absorbs replays.
- **One identity is one target:** a message is delivered to exactly one instance of an identity (the
  active lease holder). Two sessions wanting separate inboxes must use two handles.
- A cursor older than retention returns `E_CURSOR_EXPIRED` with `earliest_seq`; the adapter must choose
  backfill or skip explicitly rather than silently resuming across a gap.
- **One release gate for every injection path.** Before injecting, the adapter obtains a **grant minted by
  the hub** (see amendment item 3), which evaluates room trust, membership, mute, revocation, loop budget
  and lease generation at that moment. A grant binds to one immutable message, one recipient instance and
  one lease generation. The `trusted` flag carried on the SSE event is a log hint, never the authority.

## Consequences

- "Did that message reach that session?" is answerable from the console, and stuck `queued` rows make a
  non-compliant or broken adapter visible rather than mysterious.
- Reconnect after any crash is a replay, so adapters must dedupe; the alternative (server-side ack
  bookkeeping with no client cursor) would put durable state in a process that restarts often.
- Policy changes take effect on queued messages, which means an un-trust or a revoke lands before an old
  queue entry can inject — the case that "must not auto-start" phrasing would have missed.
- Cost: a `deliveries` table that must be written on every send, and adapters that must implement the
  gate correctly. A sink that injects without the gate is non-compliant by definition, not merely
  different.

## Amendment (2026-10, after review) — the parts that had to get precise

1. **Cursor = contiguous resolved watermark over a durable pending ledger.** "Advance after the sink
   accepts" is not enough: a message waiting on a human would either block the stream or be lost, and a
   terminally dropped message could never satisfy acceptance. Entries resolve on `injected` **or**
   `dropped`; the watermark advances only across contiguous resolved seqs; unresolved seqs survive restart
   on disk and are re-evaluated against current policy. Console shows delivered seq and watermark, with lag
   past 5 min as an attention state.
2. **Transport ack ≠ harness ack.** `received` (frame arrived) is recorded separately from `injected` (the
   sink took the text). Only `injected` is delivery.
3. **Approvals are minted by the hub, never asserted by the adapter.** `POST /api/deliveries/:seq/attempt`
   answers `allow` + a single-use, TTL-60 s, generation-bound grant, or `deny` + reason. Human release goes
   through `…/release` with the **operator token**; an agent token cannot authorize its own injection. A
   minimal `agentweb release <seq>` CLI exists from M2 so the gate works before the console does.
4. **Fail closed.** No answer, timeout, or 5xx → no injection, entry stays queued, bounded backoff, never
   terminal `dropped`. Unavailability is not a delivery outcome.
5. **Un-trust ≠ revoke, in state terms.** Un-trusting a room cancels *automatic* attempts and leaves the
   entry `queued` and manually releasable; revoking the sender or the recipient's token ends delivery with
   `dropped(reason=policy_denied)`.
6. **Leases carry a generation.** Grants bind to `(delivery_id, instance_id, lease_generation)`; a takeover
   bumps the generation and stale holders get `E_STALE_GENERATION`. Residual window (a former holder with a
   grant already in flight can inject once, bounded by grant TTL) is documented, not hidden.
7. **Two positions, not one cursor.** *Fetch position* advances as entries are durably recorded locally
   (ingestion never stalls); the *resolved watermark* advances only across contiguous `injected`/`dropped`
   entries (restart never loses). A held approval sits in the ledger as `unresolvable` if it falls behind
   hub retention and is surfaced, never silently discarded to move a cursor. Both are scoped to
   `(agent, workspace)`.
8. **`delivery_id` is the delivery identity** — stable per (message, target instance), carried on every
   event, used for reporting, grants, dedupe and console rows. A bare `seq` is never a delivery address, and
   release cannot target "someone": no live lease means `E_NO_LEASE_HOLDER` and the message stays queued.
9. **Grants are minted idempotently and consumed once.** A lost response plus a retry returns the same
   unexpired grant rather than a second usable one; consumption happens when the adapter reports `injected`
   (or `dropped`) with the grant attached.
10. **Injection is at-least-once processing with best-effort durable dedupe keyed by `delivery_id`.** A hub
    grant answers *whether* to inject; it cannot answer *whether this already happened*. Two distinct windows:
    the **stale-holder** window is bounded by lease generation and the 60 s grant TTL; the
    **uncertain-side-effect** window is not bounded by TTL at all — a lost durable marker plus a much later
    grant can duplicate. A hub-side pre-injection write does not fix sink atomicity; it trades duplicate
    injection for possible lost injection, and M2's spike chooses which trade the project accepts.
11. **`delivery_id` is per (message, agent, workspace), never per instance** — `instance_id` does not survive
    restart or takeover, so an instance-derived delivery identity would move the dedupe boundary under a
    crash. Instance and generation bind *authorization*, `delivery_id` identifies *the work*.
12. **Report lifecycle is part of the contract:** terminal reports are idempotent; late reports (after expiry
    or takeover) are recorded `late_report` and never re-inject; operator drop / accept-gap need no grant and
    no live lease. `gap_accepted` is a resolved terminal state and therefore advances the watermark.
13. **Ledgers are per `(agent, workspace, machine_id)` with a single writer** (the lease holder); two
    installations hosting one identity never advance each other's progress. Recovery: intact ledger → resume
    from fetch position; lost ledger → resubscribe from watermark and replay idempotently; both ledger and
    sink record gone → duplicate injection is possible, and that is stated.

## Rejected

- **Server-side ack only (hub tracks everything, adapter is dumb):** clean semantics, but a hub restart
  mid-flight or an adapter that dies between inject and ack still needs a client-side cursor; two sources
  of cursor truth is worse than one authoritative client cursor plus a reporting copy.
- **Advance-before-inject (at-most-once):** no duplicates, but a crash silently drops a message between
  agents — the one failure mode a messaging hub cannot afford.
- **Per-session delivery for one identity:** doubles turns for DMs and room posts addressed to a handle,
  which is precisely how a peer agent ends up starting two turns of the same work.
