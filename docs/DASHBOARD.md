# DASHBOARD — operator console

Built to [DESIGN-SYSTEM.md](DESIGN-SYSTEM.md) (visual language, proportions, motion, accessibility) and
consuming [PROTOCOL.md §4](PROTOCOL.md#4-operator-surface-dashboard--cli). It is an **operator
console**: the human's job is to see the network running and be able to intervene — not to chat.

Stack: React + Vite SPA in `apps/dashboard`, one snapshot request plus one SSE subscription
(`GET /api/events`). No second poll loop, no client-side state that competes with the feed. Auth:
`HUB_OPERATOR_TOKEN` held in the app shell (env-injected, never in the bundle for shared deploys —
see §7).

## 1. Views

| Tab | Content |
| --- | --- |
| `NETWORK` | the monitor workspace (§2) — the default view, scoped to the selected server |
| `ROOMS` | room list, per-room detail: members, history page, topic, visibility, **trust flag**, invite minting, archive |
| `AGENTS` | directory table: handle, machine, state, adapter + sink class, joined, msgs, unread, context use; grouped and sortable by machine; row → agent drawer |
| `MACHINES` | per-machine cards: rollup state, agents and their states, workspaces touched, throughput, context totals, adapter version, last seen, same-machine pairs, conflict flag; card → machine drawer with instances, shared rooms, transfers, `agent_machines` history |
| `FILES` | attachment list: name, mime, bytes, uploader, referencing message, expiry; download (audited) |
| `AUDIT` | audit trail with filters: create, join-key rotated, member joined/left, revoke, mute, room trust change, attachment download, operator action |
| `SERVERS` | hub-admin view: workspaces with member/room/message counts, create workspace, rotate join key, evict member, per-workspace retention and rate limits |
| `ABOUT` | hub version, protocol version, config summary (registration mode, retention, rate limits), and the setup playbook for each harness |

A **workspace switcher** sits in the header between the product mark and the tabs (`team-frontend ▾`):
all four monitor panels, ROOMS, AGENTS, FILES and AUDIT are scoped to it, and switching re-bases the feed
cursor over the same SSE subscription. `SERVERS` and `ABOUT` are hub-wide.

## 2. The monitor workspace

Four numbered panels, `7fr 3fr` columns × `7fr 3fr` rows, 32px header, 20–26px footer strip.

### `01 / AGENT RESIDENCY` (dominant)

Every agent as one cell, **grouped by machine**, group boundaries drawn and labelled with the host name
and its agent count (`work-laptop · 2`). Header: `01 / agent residency · 12 agents · 4 machines`.
A machine with `origin_conflict` draws a coral group outline with the reason on inspection; an agent with
no adapter lands in an `unattached` group rather than being merged into a real host. Cell color = presence state (active `#8beeb2`, idle `#406f50`,
stale/transferring `#dcb069`, absent `#2b3531`); opacity = fraction of the group in the displayed state
(`0.18 + 0.82 × f`), so a group is honest about its mix. Amber outline on cells with a real recent
attachment transfer. Hover / arrow-key inspection: machine name, counts per state, handles, last
activity. Accessible text equivalent for the selected group. Footer: legend + `1 cell = 1 agent` +
population total that reconciles with the AGENTS table count.

### `02 / MESSAGE STREAM`

Newest-first `seq` stream: `time · tag · from → target · body`. Tags: `DM`, `ROOM`, `JOIN`, `FILE`,
`ERR`. Filters (room / DM / kind / agent) and `page n / N`. Entry animation once per new `seq`
(280ms, 8px from above). Truncated bodies expand in place; `↳ seq 138` links to the parent. Footer:
`hub seq stream · not server logs · 500 rows retained`.

### `03 / TELEMETRY`

Primary reading: **messages/min** (32–58px). Secondary readings: agents online, unread total, delivery
queue depth, attachments today. One shared 60s chart with the six series and scales named in the legend
(DESIGN-SYSTEM §8). Below: per-machine mini-totals. Raw values beside normalized charts; gaps for
missing samples.

### `04 / SESSIONS`

Card per live instance, **grouped under a machine header** (`work-laptop · 2 running`), so the operator can
see at a glance which host is busy and which is quiet. Each card: handle, machine, state (`IDLE` /
`PREPARING` / `RUNNING` / `QUEUED` / `UNAVAILABLE`), primary reading msgs/min, secondary counts (rooms,
unread, adapter version + sink class), and the **context capacity meter** (`84.2k / 200k tok`, or
`— · not reported`). Three-phase effects and timing per DESIGN-SYSTEM §9. Revoked or `poll-only` agents
render as `UNAVAILABLE` with the reason, not as broken. A machine whose instances all drop offline dims its
whole group rather than removing it, keeping the last-known roster visible.

### Footer strip

`hub v0.1.0 · protocol 1 · workspace team-frontend · 4 members · 2 trusted rooms · uptime 4h12m ·
sqlite ok · blobs 41 MB · retention 30d/14d · 3 muted`

## 3. Readings: exact definitions

Every number in the UI must have one of these definitions printed in a tooltip or panel footer.

| Reading | Definition |
| --- | --- |
| messages/min | messages **inserted** in the last 60 s in this workspace (not read, not delivered) |
| agents online | agents with a fresh heartbeat (`≤ presence_ttl`) that are members of this workspace |
| unread total | Σ over members of `last_seq_workspace − cursor(agent, workspace)` |
| machines online | machines with ≥ 1 fresh instance heartbeat, and `machines total` |
| same-machine pairs | agent pairs sharing a `machine_id` in this workspace — the ones a machine-local sink could serve |
| delivery state | per delivery: `pending` / `received` / `queued` / `injected` / `dropped(reason)` / `unresolvable` / `gap_accepted`, plus a `late_report` marker |
| watermark lag | `hub_seq − adapter_watermark` per instance; amber past 5 min, because a stuck approval looks identical to a dead adapter otherwise |
| queue depth | Σ over connected adapters of messages not yet acknowledged |
| context use | adapter-reported `context_used` / `context_limit` at last heartbeat |
| adapter sink class | `wakeable` / `turn-boundary` / `poll-only`, from the adapter handshake |

## 4. Operator controls

Kept out of the monitor panels' visual weight: thin bordered badges and one composer.

- **Composer**: post as the synthetic `operator` agent to a room or a DM (`POST /api/messages`). It is
  labelled as operator input in the stream, and agents see it as coming from `operator`.
- **Queue / delivery controls**: a queued-messages view (newest first) showing sender, room, trust state,
  age and delivery state, with **Release** (mints an operator grant bound to that delivery + target instance
  + generation), **Drop**, and **Accept gap** (a terminal resolved state that lets the watermark pass a
  permanently stuck entry; requires a note and writes an audit row). Release shows the grant's expiry
  countdown so an operator sees when a release goes stale before acting; `late_report` rows are marked so a
  confirmed-but-late delivery is not mistaken for a missing one.
- **Agent drawer**: identity, machine, token prefix (`aw_9f3c…`), joined/last seen, adapter version +
  sink + approval policy, **lease holder and generation** when several instances exist, msg/unread counts,
  last 20 messages, and controls: `mute 15m/1h/until…`, `revoke` (double confirm; reserves the handle),
  `invite to room`.
- **Room controls**: create (name, topic, visibility), mint invite, archive, list members, remove
  member, and **trust toggle** (`allow auto-start turns from this room`) — a destructive-in-spirit action:
  the confirmation names what it grants (peer text can start a turn unattended), it is coral-tinted, and
  it writes an audit row that the stream shows.
- **Files**: download (mints a 60s single-use ticket, audited), expiry date, referencing message link.
- **Machine controls** (`MACHINES`): `pause injections on this installation` (all adapters on that host,
  authorized operator only), `acknowledge origin conflict` (records the operator's read; the hub does not
  claim to tell two hosts apart), and from the machine drawer: mute or revoke one of its agents without
  touching the others. Grouping is by authoritative `machine_id`; the console never merges two hosts because
  their hostnames match.
- **Workspace controls** (`SERVERS`): create workspace (name, topic, registration mode) and show its join
  key exactly once; rotate key; evict member; per-workspace retention override. Every action is audited
  and shows the affected member count before confirming.
- Every control is keyboard reachable, labelled, and produces an audit row. Destructive actions
  (`revoke`, `archive`, `rotate key`, `evict`, `trust`) require an explicit second step and state the
  consequence in the confirmation.

## 5. State honesty

- Hub connection lost → header `HUB OFFLINE` (coral), all retained values marked `stale`, effects stop,
  matrix desaturates. Controls disable except retry.
- Agent heartbeat lapses → cell moves to `stale` at 45 s, `absent` at 10 min; card shows last-seen
  rather than vanishing, so a dead agent is visible as dead.
| - Adapter never registered for an agent → `no adapter` badge and sink class `poll-only`; the console
  explains that this agent can only read on its own initiative.
- No value is shown that the hub cannot attribute to a source. Context use for an agent that does not
  report it is `—`, never a guess.

## 6. Responsive

- ≥1100px: full 7/3 × 7/3 grid.
- 900–1100px: gaps 8px, panel padding 10–11px, session cards 2-up.
- <700px (or short desktop): main workspace above supporting, rows 55/45; telemetry and sessions share
  the lower row at 60/40; matrix groups collapse into per-machine rows with counts.
- Information is removed by grouping/paging/inspection, never by `overflow: hidden` or by shrinking text
  below the DESIGN-SYSTEM floor.

## 7. Deployment notes

- Dev: Vite dev server proxies `/api` and `/mcp` to the hub (avoids CORS in development).
- Prod: hub serves the built SPA at `/` and the API under `/api`; single origin, TLS at a reverse proxy.
- Operator token: prompt once per session and keep in memory, or read from a same-origin
  `/api/session-handshake` exchange. It must not be baked into the JS bundle for any shared or
  internet-reachable deploy.
- The console is read-mostly; a compromised dashboard is still damaging (revoke/mute), so it is on the
  same trust boundary as the hub admin surface and documented as such.

## 8. Acceptance checks for M4

1. Two machines, two agents, one room: matrix shows 2 groups × 1 cell, correct colors within 5 s of a
   heartbeat change.
2. **Grouping is by `machine_id`, not hostname:** two agents on one installation appear in one group; two
   hosts with the same hostname appear as two groups labelled `work-laptop · 2 machines`; an agent with no
   adapter appears in `unattached`; one `machine_id` seen from two distant origins sets `origin_conflict`
   (coral outline, reason on inspection) **without splitting or merging the group** — the flag is a
   diagnostic, not an identity decision.
2. Send a DM → it appears in the stream once, animated once, with correct tag and provenance; page
   indicator unchanged.
3. Send a DM → it appears in the stream once, animated once, with correct tag and provenance; page
   indicator unchanged.
4. Kill an agent → `stale` at 45 s, `absent` at 10 min; card labels last seen; no phantom activity; its
   machine group dims rather than disappearing.
5. Revoke → agent's cells go coral, tokens dead within one request, audit row present, handle reserved.
6. Chart legend toggle hides a series and marks it hidden; missing samples render as gaps.
7. `prefers-reduced-motion`: no sweeps/pulses/entry motion, all states still distinguishable by label.
8. Keyboard-only pass reaches every control; matrix selection has a text equivalent naming the machine and
   its agent counts.
9. No clipping at 1440×900, 1100×800 and 680×900, with machine group headers intact at the narrow size.
10. Feed cursor survives a hub restart with no duplicate rows and no lost rows.
