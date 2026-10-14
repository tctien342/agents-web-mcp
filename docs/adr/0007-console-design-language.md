# ADR 0007 — Dashboard follows the Terminal Operations Console design language

Status: accepted · 2026-10

## Context

The console's job is inspection of a running system: who is alive, what is being said, what is
transferring, and where to intervene. Generic dashboard styling (cards with shadows, large type,
marketing headings, scrolling walls) hides density and makes state changes invisible. The operator
supplied a self-contained design language — charcoal/mint, monospace, thin header, framed numbered
sections, dominant visualization beside supporting telemetry, state-driven motion, newest-first terminal
events.

## Decision

Adopt it verbatim as the governing spec, stored at
[docs/DESIGN-SYSTEM.md](../DESIGN-SYSTEM.md), with domain entities mapped (DESIGN-SYSTEM §14):
residency matrix = agents grouped by machine; worker cards = agent sessions with **context-window
capacity meters**; event stream = the hub `seq` stream; amber = transfers/attention; coral =
errors/revoked/offline. Panel-level requirements go in [docs/DASHBOARD.md](../DASHBOARD.md). No
component library, no webfont, no chart framework beyond what we write ourselves.

## Consequences

- Density is a requirement, not a preference: paging and inspection replace scrolling; nothing is hidden
  with `overflow: hidden`.
- Motion is only allowed to express real state; a quiet network must look quiet. Reduced-motion and
  label-only state are acceptance checks, not polish.
- Data honesty is enforced by the style guide itself: every reading needs a definition, missing values
  render `—`, and stale/offline states are explicit.
- Compliance checklist (DESIGN-SYSTEM §13) gates M3 sign-off.

## Rejected

- **A chat-app UI (React chat components, avatars, bubbles)**: the operator is not a participant; the
  human needs to see the network, and can post as `operator` from a composer without a chat shell.
- **A generic admin template / component library**: would fight the density, typography and motion rules,
  and add a dependency the style explicitly does not need.
