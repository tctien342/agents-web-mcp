# ADR 0001 — Monorepo layout with a shared protocol package

Status: accepted · 2026-10

## Context

Four things ship together and must stay in step: the hub, the dashboard, the per-harness adapters, and
the wire contract between them. A contract drift between hub and adapter is the most likely class of
bug in this project.

## Decision

```text
agents-web-mcp/
  apps/hub/              Fastify + MCP SDK + SQLite + blob store
  apps/dashboard/        React + Vite operator console
  packages/protocol/     zod schemas + TS types for every MCP tool and HTTP route (single source)
  packages/adapter-core/ cursor, SSE, framing, approval modes, heartbeat
  adapters/pi/           pi extension
  adapters/claude-code/  socket sink + Stop-hook + optional channel bridge
  adapters/codex/        Stop-hook poller
  adapters/opencode/     serve-API sink
  adapters/generic/      long-poll CLI + AGENTS.md snippet
  docs/                  this suite
  scripts/               smoke and operator CLIs
```

pnpm workspaces. `packages/protocol` is imported by hub, dashboard and every adapter; tool input/output
schemas are defined once and the MCP tool registrations are generated from them.

## Consequences

- One lockfile, one `pnpm build`, contract changes fail typecheck in all consumers at once — the point.
- Adapters are separately installable packages so an agent's machine installs only its own.
- Dashboard dev needs a proxy to the hub (documented in DASHBOARD.md §7).

## Rejected

- **Single package**: adapters would drag hub dependencies onto every agent machine.
- **Separate repos**: contract drift becomes a release-orchestration problem with three consumers.
