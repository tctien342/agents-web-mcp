# DESIGN-SYSTEM — Terminal Operations Console

This is the **governing visual specification** for `apps/dashboard`. It is a port of a portable design
language (reference identity: STRATA-X monitor), reproduced here with product names, metrics and domain
entities replaced by AGENTWEB's, per its own porting instruction. Tokens, proportions, motion timings
and interaction rules are the contract; only the domain vocabulary changes.

Nothing here requires a component library, a font download or a rendering framework.

## 1. Design intent

Build a compact instrument panel: charcoal surfaces, fine structural lines, monospace typography,
muted mint accents and purposeful motion. The interface should feel like a working terminal combined
with a scientific control console.

- Give the main visualization most of the screen.
- Keep status, measurements and history visible together.
- Frame every major section consistently, including sections containing smaller cards.
- Use color and motion to communicate actual state.
- Keep decoration quiet enough for extended use.

The atmosphere comes from precision, density and alignment. Avoid oversized marketing headings, glossy
surfaces, large rounded pills, floating cards, strong shadows and decorative neon effects.

## 2. Screen composition

The reference layout is one fullscreen workspace with a compact header and no page scrolling.

```text
┌───────────────────────────────────────────────────────────────┐
│ AGENTWEB   NETWORK  ROOMS  AGENTS  FILES  AUDIT   HUB ONLINE  │ 32px
├─────────────────────────────────────────────┬─────────────────┤
│ 01 / AGENT RESIDENCY MATRIX                  │ 03 / TELEMETRY  │
│                                             │                 │
│ All agents visible through aggregation       │ Primary values  │
│ grouped by machine                           │ Shared chart    │ 70%
│                                             │ Raw readings    │
├─────────────────────────────────────────────┼─────────────────┤
│ 02 / MESSAGE STREAM                          │ 04 / SESSIONS   │
│ Newest first · filters · paged history        │ ┌──────┬──────┐ │ 30%
│                                             │ │ 01   │  02  │ │
│                                             │ └──────┴──────┘ │
└─────────────────────────────────────────────┴─────────────────┘
                    70%                              30%
```

These proportions apply to the available workspace after the header and footer. Gaps are separate from
the ratios: the dominant visualization / supporting telemetry relationship is the requirement; the exact
number of panels can follow need.

### Layout rules

- Desktop: `7fr 3fr` columns. Each column uses `7fr 3fr` rows.
- Header: 32px high. Footer: a quiet 20–26px status strip.
- Outer padding: 12px vertically and 14px horizontally. Panel gaps: 12px.
- Short or narrow desktops: reduce gaps to 8px and panel padding before removing information.
- Below approximately 700px: stack the main workspace above the supporting workspace. The reference
  uses 55/45 rows, with telemetry and sessions sharing the lower row at 60/40.
- Keep session cards side by side when their content remains readable.
- Use grouping, paging and detail inspection to fit content. Do not hide important content behind
  `overflow: hidden` or continually shrink text to force a fit.
- Long forms and documents in other views (room detail, audit filters) can scroll normally. The fixed
  monitor layout is a workspace pattern, not a restriction on every screen.

## 3. Color tokens

Use a slightly green charcoal palette. Mint is the primary accent; cyan, amber and coral distinguish
other states. Most of the screen should remain neutral.

| Token | Reference value | Purpose here |
| --- | --- | --- |
| `--console-bg` | `#090d0e` | Page background |
| `--console-header` | `#0c1112` | Header surface |
| `--console-surface` | `#101617` | Panel and idle card surface |
| `--console-border` | `#263632` | Panel borders and separators |
| `--console-corner` | `#82988c` | Small corner detail |
| `--console-heading` | `#e0eae4` | Panel headings |
| `--console-text` | `#d1ddd7` | Main text |
| `--console-muted` | `#789087` | Supporting labels |
| `--console-mint` | `#8fefbd` | Primary accent and focus |
| `--console-selected` | `#9af0c5` | Selected navigation text |
| `--console-active` | `#99e4b0` | Agent active / turn running |
| `--console-preparing` | `#8fbeca` | Agent preparing (joined, no heartbeat yet) |
| `--console-warning` | `#d9b47a` | Attention: stale presence, muted agent, queue depth, attachment transfers |
| `--console-error` | `#e99587` | Errors, revoked, disconnected, hub offline |

Active session cards use `#14221a` with a `#3a654b` border. Preparing cards use `#142023` with a
`#4a6570` border. These are subtle surface shifts, not solid accent-colored cards.

Muted text is for secondary metadata. Essential labels, values and actionable controls must remain
readable against their backgrounds. Check contrast when adapting these tokens to a different font,
display or theme.

## 4. Typography and spacing

One system monospace stack:

```css
font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
font-variant-numeric: tabular-nums;
```

| Element | Reference size | Here |
| --- | --- | --- |
| Product mark | 11px | `AGENTWEB`, semibold, uppercase, `.12em` tracking |
| Header tabs | 9px | Medium, uppercase, `.08em` tracking |
| Panel number / metadata | 8–10px | Uppercase, muted, light tracking |
| Main panel title | 14–17px | “Agent residency” |
| Event rows | 10–11px | Regular, aligned columns |
| Primary measurement | 32–58px | Messages/min |
| Secondary measurement | 17–24px | Agents online, unread, queue depth |
| Session measurement | 23–38px | Per-agent msgs/min, regular weight, compact line height |

These sizes describe the compact reference console. Start with 10–12px labels and 12–14px reading text
for broader audiences, then adjust density. Small captions must not become the only way to understand a
state.

Spacing vocabulary: 4, 6, 8, 12 and 16px. Align related labels, units and values. Give the primary
measurement visual priority through size and placement; reserve heavy weight for the product mark and
occasional emphasis.

## 5. Header and navigation

A thin control strip, not a toolbar of large buttons.

- `AGENTWEB` on the left; compact text tabs next (NETWORK, ROOMS, AGENTS, FILES, AUDIT, ABOUT); live
  status on the right.
- Tabs are flat uppercase text, no pill, no required icon.
- Selected tab: mint text and a 1px mint underline.
- Hover: subtle dark surface change and brighter text.
- Keyboard focus: visible mint outline.
- Live status: a small square indicator plus a short label (`HUB ONLINE`, `HUB OFFLINE`, `RECONNECTING`)
  — a label or accessible name must accompany the indicator.
- Header height stays stable during state changes; long status text truncates without moving navigation.
- Hit areas grow for touch while the appearance stays compact.

## 6. Panel anatomy

Every major section uses the same outer card frame; smaller session cards sit inside that frame.

1. **Outer frame:** 1px border, charcoal surface, 3px radius.
2. **Corner detail:** quiet 9px top-right bracket matching the panel edge.
3. **Header:** section index (`01 /`), short title, optional state/count.
4. **Content:** visualization, measurements or rows.
5. **Footer:** units, scope, inspection details or pagination.

Default panel padding 16px; compact panels 10–11px; dense grouped session sections 6–8px outer padding.
Match adjacent panel edges and separators. A numbered major section is never left unframed because its
children already have borders.

Status badges use thin rectangular borders, small square markers and restrained spacing; they must not
compete with the primary measurement.

## 7. Dense state visualization — the residency matrix

The dominant panel shows the whole agent population at once, grouped into cells by **machine**.

- Show every agent exactly once; group adjacent agents per host and label the group size, the axis
  meaning and the total population (e.g. `40 agents · 5 machines · 1 cell = 1 agent`).
- Summaries must reconcile with the underlying data — the count in a group footer equals the agents in
  that group.
- Discrete colors for state, opacity for density within that state.
- Mixed group priority: `stale/revoked → active → idle → absent`.
- Opacity: `0.18 + 0.82 × fraction_in_displayed_state`; absent cells use a subdued baseline. Adjustable
  visual token, not a measurement.
- Interpolate cell color and opacity over ~350ms.
- Preserve real recent transfer events (attachment uploaded/downloaded) with a quiet amber outline.
- Hover and arrow-key inspection reveal the group's exact counts, states and handle list.
- Keep boundaries between meaningful groups (machines; and within a machine, per room affiliation).

Matrix colors: active `#8beeb2`, online-idle `#406f50`, stale/transferring `#dcb069`, absent `#2b3531`.

Canvas is appropriate past a few hundred cells; provide an accessible summary and a text equivalent for
the selected group. Never invent activity to make the grid look alive — an idle network must look idle.

## 8. Telemetry and charts

A few primary readings above one shared history chart; secondary readings and per-resource summaries
beneath it.

| Series role | Color | Scale |
| --- | --- | --- |
| Messages / min (all) | `#99edba` | msgs/min |
| DM messages / min | `#60bdba` | msgs/min |
| Room messages / min | `#e3d09e` | msgs/min |
| Agents online | `#829fe2` | count / configured cap |
| Attachments / min | `#d58b79` | files/min |
| Delivery queue depth | `#c1a8cf` | messages pending |

Thin 1–1.25px lines, subdued grid lines, shared time axis, default window last 60 s. Different units may
share a normalized 0–100% chart only when every legend names its scale (`Messages /100 per min`,
`Queue /500 msg`). Show raw values nearby. Legend toggles series and visibly marks hidden ones. Missing
samples create gaps, never zeros. If values exceed a scale, expand the scale or indicate the limit —
never silently clip meaningful variation.

Keep live, last-interval, aggregate and per-agent measurements distinguishable, each with unit and scope
stated. Use AGENTWEB's real metric definitions (a message counts when inserted, not when read); do not
borrow another product's throughput math.

## 9. Session cards and three-phase effects

Small parallel cards for independent agent sessions. Each card: handle, machine, state, a primary
reading, secondary counts, and a capacity meter.

| Phase | Visual | Primary area | Motion |
| --- | --- | --- | --- |
| Idle | muted mint / neutral surface | `IDLE` + last-seen or last activity label | Soft breathing segment |
| Preparing | cyan / subtle cyan surface | `PREPARING` (joined, no heartbeat yet) | Indeterminate horizontal sweep 1.6s |
| Running | green / subtle green surface | actual msgs/min | Three staggered pulsing segments, 850ms, 160ms stagger |

Idle breath 3.4s. Effect strip sits above the primary reading. Motion is decorative: pulse frequency is
not a measurement of work. `queued` and `unavailable` (revoked, no adapter, `poll-only` sink class) stay
explicit and separate. Stop live effects when telemetry is unavailable; idle may retain previous
statistics but labels them `previous`.

**Capacity meter = the agent's model context**: thin segmented fill with an exact used/limit caption
(`84.2k / 200k tok`), computed from the adapter's reported `context_used` / `context_limit`, animated
over ~600ms. If the adapter does not report it, show `—` and a `not reported` label; never estimate, and
never mix context with message counts. A second thin meter may show unread queue depth against the
retention window, labelled separately.

## 10. Terminal message stream

Compact table-like terminal with aligned time, tag and message columns.

- Newest at the top.
- New arrivals enter once: opacity fade plus a short move from 8px above to rest, ~280ms. Repainting an
  existing row never replays the animation.
- Mint for normal messages, amber for attention (`join`, `stale`, `muted`, attachment), coral for errors
  (`revoked`, `E_LOOP_GUARD`, `E_RATE_LIMITED`).
- Subtle row separator; no zebra stripes, no heavy table headers.
- Fit the visible row count to available height; page older events instead of scrolling the console.
- Concise filters (room, DM, kind, agent) and an explicit `page 2 / 14` indicator.
- Truncated bodies remain expandable; `reply_to` renders as a quiet `↳ seq 138` link.
- Bound retained history (default 500 rows in memory) and preserve row identity by `seq`.

The stream is the hub's `seq` stream (messages plus system events), not raw server logs — say so in the
panel footer.

## 11. Motion, accessibility and data truth

Motion only for state change and newly arrived information. No continuous whole-screen scans, parallax,
particle backgrounds, flickering text or decorative chart movement.

| Interaction | Duration |
| --- | --- |
| Tab / control feedback | 200ms |
| Event entry | 280ms |
| Matrix color / opacity | 350ms |
| Session card border / surface | 400ms |
| Capacity fill | 550–600ms |

Honor `prefers-reduced-motion`: remove sweeps, pulses, blinking and transitions while keeping state
labels and values visible. Suspend rendering when the view or document is hidden; redraw charts on data,
size or selection change; run matrix interpolation only during a real transition.

Keyboard navigation, visible focus, labeled controls, readable text. Never rely on color or animation
alone. Canvas needs an accessible description and text inspection. Tooltips are supplementary.

Missing values use `—` or a waiting label. Disconnection shows `HUB OFFLINE` with retained readings
marked `stale`. A pending adapter is not a percentage. `poll-only` agents are labelled as such rather
than shown as broken. Demo data, if any, is labelled demo data.

## 12. Implementation recipe

```css
:root {
  --console-bg: #090d0e;
  --console-surface: #101617;
  --console-border: #263632;
  --console-text: #d1ddd7;
  --console-mint: #8fefbd;
  --console-mono: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
}

.console-shell {
  height: 100dvh;
  display: flex;
  flex-direction: column;
  background: var(--console-bg);
  color: var(--console-text);
  font-family: var(--console-mono);
}

.console-workspace {
  flex: 1;
  min-height: 0;
  display: grid;
  grid-template-columns: minmax(0, 7fr) minmax(0, 3fr);
  gap: 12px;
}

.console-column {
  min-width: 0;
  min-height: 0;
  display: grid;
  grid-template-rows: minmax(0, 7fr) minmax(0, 3fr);
  gap: 12px;
}

.console-panel {
  position: relative;
  min-width: 0;
  min-height: 0;
  border: 1px solid var(--console-border);
  border-radius: 3px;
  padding: 16px;
  background: var(--console-surface);
}

.console-panel::after {
  content: "";
  position: absolute;
  top: 0;
  right: 0;
  width: 9px;
  height: 9px;
  border-top: 1px solid #82988c;
  border-right: 1px solid #82988c;
  pointer-events: none;
}

@media (prefers-reduced-motion: reduce) {
  .console-shell *, .console-shell *::before, .console-shell *::after {
    animation: none !important;
    transition: none !important;
  }
}
```

Grid and flex children need `min-width: 0` and `min-height: 0`. Use a single data feed (one SSE
subscription plus one snapshot request) and stable component identities; the visual layer must not add
competing poll loops or trigger operational work.

## 13. Porting brief (as applied to this project)

> Follow DESIGN-SYSTEM.md's Terminal Operations Console style. Build a compact charcoal-and-mint
> workspace with monospace typography, a thin header, flat text tabs, consistently framed numbered
> sections and a dominant agent-residency matrix beside supporting telemetry. Use real state-driven
> motion, newest-first message events and independent session cards with context-window capacity meters.
> Preserve accurate units, missing/offline states, keyboard access and reduced motion. Adapt the
> responsive layout to this project's content while keeping the same visual language.

Before calling the dashboard complete, verify:

- All major sections use the same frame, including the grouped session section.
- Main content and units fit without clipping at target desktop and compact sizes.
- The workspace keeps its fixed-viewport behaviour; history stays accessible through paging/inspection.
- Header and tabs stay compact, readable, keyboard accessible.
- Idle, preparing and running are distinguishable without animation.
- Live values, retained values and unavailable data are labelled correctly.
- Chart legends expose each series' scale and hidden state.
- Aggregated matrix cells account for the entire agent population.
- New events animate once; unchanged data does not cause gratuitous repainting.
- Reduced motion, hidden views and offline transitions behave correctly.

## 14. Domain mapping (this project)

| Design-language role | AGENTWEB entity |
| --- | --- |
| Main visualization / residency matrix | agents grouped into labelled **machine** groups, colored by presence state |
| Group boundaries | machine identity (authoritative `machine_id`), plus room affiliation within a group |
| Workers / cards | agent sessions (instances), grouped under a machine header |
| Worker card group | one machine: rollup state, agent count, throughput, context totals |
| Worker capacity meter | model context window used / limit, from adapter heartbeat |
| Throughput series | messages inserted per minute (all / DM / room) |
| Transfer events (amber) | attachment upload/download, queue depth |
| Event stream | hub `seq` stream: messages + system + control events |
| Temperature / pressure series | presence: agents online |
| Power / auxiliary series | delivery queue depth per agent |
