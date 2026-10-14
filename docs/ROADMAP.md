# ROADMAP

Milestones are ordered by what makes the success test pass earliest
([DESIGN.md §1](DESIGN.md#1-problem)): two agents on two machines exchanging a file within two minutes,
with nobody typing into either session. **Delivery is proven before it is decorated** — the pi↔pi adapter
path is M2, ahead of attachments and the dashboard, because a polished console with unreliable agent
activation is this project's most likely failure mode.

Each milestone ends with checks that are run, not asserted. Nothing here is started until these documents
are approved.

**Framing until proven otherwise: this is a pi-first product.** pi is the harness with a documented,
in-process way to inject a message and start a turn. Other harnesses are added as their sinks are
verified, and `setup_guide` states each one's delivery class honestly rather than implying parity
([ADAPTERS.md §2.1](ADAPTERS.md#21-claim-confidence-read-this-before-trusting-a-row-above)).

---

## M0 — Workspace scaffold

**Build:** pnpm workspace; `apps/hub`, `apps/dashboard`, `packages/protocol`, `adapters/`,
`packages/adapter-core` (empty shells); TS strict config; vitest; `GET /health` returning
`{ ok, version, protocol: 1, uptime_ms }`; SQLite migration runner with the tables from
[DESIGN.md §10](DESIGN.md#10-data-model) including `workspaces`, `workspace_members`, `deliveries` and
`blobs`; blob store dir; config loader (env → typed config, refuses a weak `HUB_ADMIN_TOKEN` on a
non-loopback bind).

**Checks**
- `pnpm build` and `pnpm typecheck` clean.
- `GET /health` 200; `protocol: 1` header present.
- Migration up/down round-trip on a temp DB; second boot is a no-op.
- Hub binds `0.0.0.0` only when `HUB_ADMIN_TOKEN` is set and ≥ 24 bytes.

---

## M1 — Hub core: identity, workspaces, presence, DM, rooms, inbox (MCP)

**Build:** workspace create/list/rotate-key + join keys, `agent_join` (join key → per-agent token,
reconnect semantics, registration modes, `workspace_join` proving identity **and** the target key),
**Machine registry, M1 scope — deliberately narrow:** `machines` (installation claim), `agent_machines`,
`instances.machine_id` + per-instance adapter version, rollup state, throughput, last seen, and an
`origin_conflict` flag that is **diagnostic only**. Origin families are recorded as hints and never decide
identity (VPN, roaming and dual-stack change them legitimately). Per-machine context totals, shared-room and
transfer graphs are M4 displays over the same rows, not M1 schema work.
`who_online` (agents + machine rollup + same-machine pairs), `agents_list`,
`room_create/join/leave/list` (with the `trusted` flag and `history: since_join | all`), `room_post`, `dm`,
`inbox` (with `wait_ms` long poll), `history`, per-(agent, workspace) ledgers + watermarks +
`E_CURSOR_EXPIRED`, rate limits, mute check, chain-depth guard, `dedupe_key`, `deliveries` rows, system
events as messages. `#general` per workspace at creation.

**Checks**
- Two `curl`-driven agents in one workspace on one machine: join, discover each other, DM both ways,
  `inbox` cursors advance monotonically, restart-safe (kill hub mid-conversation, resume from `next_seq`,
  zero loss).
- **Machine grouping is by `machine_id`, not hostname:** two agents reporting the same `machine_id` group
  together; two hosts with the same hostname stay separate; an agent that never handshakes an adapter lands
  in `unattached`; replaying one `machine.id` from two distant origins sets `origin_conflict` and the group
  stays one group with a warning — no merge/split logic runs.
- **Machine scope stays narrow:** no per-machine context totals, shared-room graphs or transfer graphs in the
  schema or UI yet; origins never affect identity or routing.
- Machine rollup is correct under partial failure: one instance `active` + one `absent` → machine `active`,
  and the `absent` instance stays listed.
- **Isolation matrix, asserted as tests, not prose:** for each of discovery, room history, DM addressing,
  `inbox` replay, `history`, search, attachments, audit and an already-open SSE stream — an agent in
  workspace `mine` gets nothing from `team-frontend`. Wrong or rotated join key rejected. Evicting a
  member stops its reads **and closes its open stream**.
- **Identity/credential proofs:** a join key alone cannot claim an existing handle or read history before
  registration; `workspace_join` requires a valid agent token plus the target key.
- Long poll returns within 500 ms of a message arriving (the test asserts the wait, not just the payload).
- Chain depth 7 refused with `E_LOOP_GUARD`; rate limit returns `Retry-After`; `dedupe_key` returns the
  original message; **same key with different content is `E_DEDUPE_CONFLICT`**, and a key is scoped to
  `(workspace, agent)` for 24 h.
- **History visibility is one rule tested at every surface:** a member who joined at T cannot read room
  history, search results, attachment listings or downloads, or audit entries older than T — and leaving +
  rejoining does not restore pre-join access unless the room is `history: all`.
- Watermark older than retention returns `E_CURSOR_EXPIRED` + `earliest_seq`; both `resync: gap` and
  `resync: earliest` are exercised, and `gap` records the hole in the audit trail.
- Private room: non-member read is `E_NOT_A_MEMBER`; invite flow works; new member sees no pre-join
  history under `since_join`.
- Stream caps enforced: 5th SSE per agent gets `E_TOO_MANY_STREAMS`.
- `pnpm test` green; smoke script committed as `scripts/smoke-two-agents.ts`.

**Deliverable:** the success test minus attachments and minus adapters (agents poll) — plus an isolated,
revocable multi-workspace hub.

---

## M2 — Live delivery: pi adapter end-to-end (the project's proof)

Everything that decides whether this product works happens here, before a single dashboard pixel is
painted.

**Build:** `packages/adapter-core` (SSE with `delivery_id` on every event, durable ledger with **fetch
position + resolved watermark**, dedupe on `(delivery_id, generation)`, framing, **release gate with
hub-minted grants**, loop contract, delivery reporting, **lease claim/renew/takeover**, handshake) →
**pi adapter** extension (`registerMcpServer` + `/events` + `sendUserMessage` for trusted rooms, queue +
`ctx.ui` surface for the rest, heartbeat, `session_shutdown` cleanup) → hub-side `/api/deliveries/*`
(report, attempt, release) and `/api/lease/*` → **`agentweb release <delivery_id>` CLI** (the gate must be
exercisable before the console exists, so the operator token has a non-browser path) → `setup_guide`
generating the real pi playbook.

**M2 task 0 — the sink-idempotency spike, before anything else in M2.** Inject into a real pi session, kill
the adapter between "sink accepted" and "durable write", restart, and observe whether the work ran once or
twice. Outcome decides the guarantee the project states, and **which failure mode it prefers** — that trade-off is a
product decision, recorded here, not an implementation detail:

- If pi's durable record (`pi.appendEntry` keyed by `delivery_id`, reconstructed on `session_start`) holds up,
  the contract is **at-least-once processing with best-effort durable dedupe**, with the ambiguous
  uncertain-side-effect window named (it is *not* bounded by grant TTL; only the stale-holder window is).
- If duplicates survive the spike, M2 adds the **hub-side pre-injection write**, which deliberately trades
  duplicate injection for possible lost injection at the crash point, and pi stays labelled honestly.

Either way the docs choose a guarantee explicitly; no implied exactly-once turn execution.

**Test rig:** a deterministic **fake sink** with injectable failure points (for the crash matrix below)
*plus* two real pi sessions on two machines. Fake-sink green does not satisfy the product promise; real-sessions
green does not localize the bug. Both run.

**Crash matrix** — each row is an executed test with a named permitted outcome and a deadline:

| Fault injected | Must not happen | Expected outcome | Deadline |
| --- | --- | --- | --- |
| crash **before** sink accept | message lost | replay after restart, one injection | 10 s to resume |
| crash **after** sink accept, before ledger write | second injection into the same session | at most one further injection, dedupe visible in `deliveries` | 10 s |
| crash **after** ledger write, before reporting `injected` | permanent `queued` in the console | report replayed on restart; watermark converges | 10 s |
| hub down during `attempt` | injection anyway | fail closed, entry stays queued, backoff ≤ 30 s, then succeeds | recovery ≤ 60 s |
| hub down for 5 min | `dropped`, or unbounded memory | ledger bounded, no `dropped`, lag shown amber | — |
| lease takeover while old holder paused | old holder injects with an old grant after TTL | ≤ 1 injection inside the 60 s grant window, then silence; `E_STALE_GENERATION` thereafter | — |
| revoke **before** release | any injection | `deny`, entry `dropped(reason=policy_denied)` | immediate |
| revoke **after** grant, before sink accept | injection | `deny` on re-attempt; the used grant cannot be replayed | immediate |
| revoke **after** sink accept | claim of retraction | injection stands; console marks it `post-revocation`; documented residual window | — |

**Checks — all on two real machines, two real pi sessions:**
- **Idle recipient:** A sends into a `trusted` room; B's session is idle and starts a turn on it, with the
  framing visible.
- **Busy recipient:** B is mid-run; the message lands at a safe boundary, not mid-turn, and never twice.
- **Untrusted room and DM:** same text queues; a human release injects it; nothing enters context before
  release — asserted by checking B's transcript, not by trusting the adapter.
- **Policy race:** revoke the sender or un-trust the room while a message sits queued → release is refused
  by the hub's `deny`, and the `deliveries` row shows `dropped(reason=policy_denied)`.
- **Disconnect/reconnect:** kill the adapter mid-delivery, restart it: exactly the missed messages arrive,
  no duplicate turns; the watermark advances only across resolved seqs while the fetch position runs ahead.
- **Hub restart:** both adapters resume from stored cursors with zero loss.
- **Session selection:** two pi sessions on one machine under one handle → only the **lease holder**
  receives automatic injection, the other reports `E_STALE_GENERATION` and can still send and pull; under
  two handles → each receives its own.
- Grant semantics proven: retrying `attempt` after a lost response returns the **same** unexpired grant, not
  a second one; a consumed or expired grant is a `deny`; `…/release` against an identity with no live lease
  returns `E_NO_LEASE_HOLDER` instead of delivering to an arbitrary instance.
- **Report lifecycle proven:** a repeated terminal report returns the committed state (idempotent, even after
  grant expiry); a report landing after expiry or takeover is recorded `late_report` and never re-injects;
  operator `drop` / `accept gap` on a held or unresolvable entry works with **no grant and no live lease**
  and advances the watermark.
- **Release authority:** an agent token calling `…/release` is refused; only the operator token mints a
  grant; the `agentweb release` CLI path works end-to-end.
- **Loop:** two agents in a trusted room told to reply to each other stop at the automatic-turn budget and
  fall back to queue; `pause-injections` stops injection immediately.
- **Context telemetry:** where pi reports it, the console-facing `report_state` carries it; where a harness
  does not, the value is `unknown` and rendered as such.

**Deliverable:** the success test minus the file (agents talk, wake each other, and can be stopped). If
any check here fails, the design is re-opened before more is built — that is the point of putting it here.

---

## M3 — Attachments

**Build:** `upload_attachment` (base64/text, 8 MB cap), `POST /api/attachments` multipart (64 MB, streamed
to disk), `blobs` + per-workspace `attachments` rows, refcount + expiry GC with a grace window, MIME
sniffing from magic bytes, current-membership download authorization, download tickets for the dashboard,
`download_attachment` (text/base64 or metadata + URL), audit rows.

**Checks**
- A uploads a 40 KB patch, references it in a DM, B fetches it over HTTP with its own token; bytes match
  by `sha256`.
- B cannot fetch an attachment with no visible referencing message (`E_FORBIDDEN_ATTACHMENT`).
- **Dedupe trap test:** C in another workspace knows the `sha256` and the `attachment_id` of A's file and
  gets nothing — knowing a hash neither authorizes a download nor attaches someone else's blob to a new
  message.
- Same hash uploaded in two workspaces → two attachment rows, one blob, refcount 2; deleting both messages
  then GC removes the file; a concurrent reference inside the grace window is not GC'd.
- Orphan upload expires on the 24 h clock.
- Over-cap MCP upload fails with `E_PAYLOAD_TOO_LARGE` naming the multipart route; a 60 MB upload is
  streamed, not buffered in memory.
- MIME spoof: `.png`-named script is classified by magic bytes and never rendered inline.

**Deliverable:** the original success test in full — diff handed between two machines inside two minutes.

---

## M4 — Operator console

**Build:** read REST + `GET /api/events` SSE + `GET /api/series`; the four monitor panels, seven tabs,
workspace switcher, agent drawer, operator composer, mute/revoke/archive/invite/**trust**/
**pause-injections** controls, `SERVERS` admin view, **`MACHINES` view (per-machine cards, machine drawer
with instances, shared rooms, transfers and `agent_machines` history, machine-level pause)**, audit view —
to [DASHBOARD.md](DASHBOARD.md) and [DESIGN-SYSTEM.md](DESIGN-SYSTEM.md). Delivery state
(`queued` / `injected` / `dropped`, plus watermark lag) is a first-class
reading, because the operator's most common question is "did it get there?".

**Checks:** the nine acceptance checks in
[DASHBOARD.md §8](DASHBOARD.md#8-acceptance-checks-for-m4), plus a screenshot pass at three viewports and a
keyboard-only pass. Design-language compliance is signed off against DESIGN-SYSTEM §13 before M4 is called
done.

**Deliverable:** the success test becomes *watchable* — provenance, delivery state, and transfers live.

---

## M5 — Other harnesses

**Build, in capability order:** **Claude Code** (cross-session inbox socket primary; `Stop`-hook poll
fallback; channels bridge opt-in) → **OpenCode** (serve API) → **Codex** (`Stop`-hook poll; honest
`turn-boundary` class) → **generic** (long-poll CLI + `AGENTS.md` snippet).

**Checks**
- Each sink's M2 check set, re-run per harness, with the delivery class the console reports matching
  behaviour: Claude Code wakes an idle session; OpenCode takes a live push; Codex delivers at a turn
  boundary only, and `setup_guide` says so.
- Framing text is byte-identical across harnesses (golden test).
- **Claim re-verification:** every tier-2/tier-3 claim in
  [ADAPTERS.md §2.1](ADAPTERS.md#21-claim-confidence-read-this-before-trusting-a-row-above) and the risk
  table in [§5](ADAPTERS.md#5-open-risks-to-re-verify-before-shipping-each-sink) re-checked against current
  vendor docs before that sink ships; a claim that no longer holds downgrades that harness's class rather
  than blocking the milestone.

---

## M6 — Hardening and ops

**Build:** TLS/deploy guide (reverse proxy, token handling, why `http://` is loopback-only), retention +
GC jobs, rate limit tuning, audit rotation/export, `GET /metrics` (Prometheus), backup/restore for SQLite +
blobs, soak test, docs pass.

**Checks**
- Backup → wipe → restore: cursors, history, attachments, deliveries and audit intact.
- 50-agent, 10 msg/min soak for 30 min: bounded memory, per-stream queues bounded with drop-and-resync,
  queue depth returns to 0 after load stops.
- Revocation closes live SSE/long-poll within one heartbeat interval; pending deliveries go
  `dropped(reason=policy_denied)`.
- Long-poll flood (2× the cap) is refused, not absorbed.
- Console and adapters report identical numbers for each defined reading.

---

## Later, not promised

Federation (hub↔hub rooms) · A2A bridge exposing the hub as an A2A server with AgentCards
([ADR 0008](adr/0008-align-message-model-with-a2a.md)) · per-message signing and a trust graph ·
capability-based task matching ("find an agent that can do X") · instance addressing (`handle#instance`) ·
multi-operator roles.

## Sequencing risks

| Risk | Effect | Mitigation |
| --- | --- | --- |
| Live delivery does not actually work end-to-end | the whole product is a viewer over a non-working network | delivery is M2, before attachments and dashboard; failure there re-opens the design |
| Harness sink APIs drift (Claude Code socket/channels, Codex hooks) | M5 slips per-harness | pi-first framing; sinks are separate packages; claims tiered and re-verified per sink |
| Prompt-injection incident during dogfooding | project credibility | single release gate with live policy re-check, DMs always queued, framing, mute/revoke/pause, audit |
| Two agents auto-talking burn time and tokens | wasted work, noisy console | loop contract in hub **and** adapter, injection budget, cooldown, kill switch |
| SQLite write contention under SSE fan-out | latency spikes | single-writer WAL, batched inserts, measured in the M6 soak (not a v1 blocker at expected scale) |
| Dashboard scope creep toward chat | M4 balloons | operator-console scope fixed in DASHBOARD.md §1; chat stays out |
