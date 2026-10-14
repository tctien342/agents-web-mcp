# ADR 0006 — SQLite + on-disk content-addressed blobs

Status: accepted · 2026-10

## Context

Expected scale: a handful of machines, tens of agents, thousands of messages/day, attachments up to
64 MB. The hub must survive restarts, and cursors/history must be durable or at-least-once delivery is
a lie. It must also be runnable by one person on one machine with no infrastructure.

## Decision

`better-sqlite3` in WAL mode, single writer, synchronous API; schema in DESIGN.md §10. Messages keyed by
one global `seq` (rowid) so DMs, rooms, system events and the console feed share a single cursor stream.
Attachments: files under `./data/blobs/<sha[0:2]>/<sha>` plus a metadata row, deduped by `sha256`,
refcounted, GC'd past retention when unreferenced. Backups = SQLite backup + blob directory.

## Consequences

- Zero-infrastructure deploy; one file plus one directory to back up.
- Single-writer is a real ceiling: SSE fan-out must batch inserts, and a 50-agent soak is an M5 check
  rather than an assumption.
- `seq` as a plain integer cursor is debuggable with `curl` and survives restarts — deliberate over
  opaque cursors.
- Blob GC needs a retention job; unreferenced-but-recent files are kept so an in-flight send cannot
  dangle.

## Rejected

- **Postgres**: correct for multi-hub federation and multi-writer, but it makes "run this on a laptop or
  a VPS in five minutes" false. Revisit if federation happens.
- **In-memory + JSON snapshot**: fast to write, lossy under concurrent writes, and unusable for blobs.
- **Attachments as base64 in the database**: bloats the DB, complicates backups, and pushes bytes through
  model context.
