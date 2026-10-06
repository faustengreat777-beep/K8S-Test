# ADR-0009: Server-Sent Events for real-time progress, backed by a replayable event log

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0009-real-time-updates-sse.ru.md)

## Context

The UI must show deployment progress and live logs in real time (prompt §15) through WebSocket or SSE (prompt §42). Closing the browser must not lose anything, and on return the UI restores state from the backend (prompt §66). There may be several API replicas.

## Decision

- Use **Server-Sent Events** (`text/event-stream`) for operation progress (`/operations/{id}/events`) and dashboards (`/events?clusterId=`).
- Every event is a row in `operation_events` with a monotonically increasing `id`. The SSE `id:` equals that row id. On reconnect, the client sends `Last-Event-ID` (native `EventSource` does this automatically), and the server replays from the DB, then streams live.
- Cross-replica fan-out: workers `NOTIFY` after committing events. Each API replica `LISTEN`s and pushes new rows to its subscribers. No Redis or message broker is needed.
- Heartbeat comments every 15 s; `retry: 3000`; `event: end` on terminal states; per-user connection caps.
- WebSocket is reserved for a future interactive node/pod terminal, where bidirectional traffic is needed.

## Alternatives considered

| Option | Why not |
|---|---|
| WebSocket | Bidirectional, which we don't need for progress. Its own reconnect and resume protocol is needed anyway, and it is harder through some proxies. No native resume semantics |
| Long polling | Works everywhere but adds latency and server load. Kept as fallback: `GET /operations/{id}/events?after=` without SSE |
| gRPC streaming | Browsers need a proxy (grpc-web), which adds complexity |

## Consequences

- Positive: simple and HTTP-native (works with the cookie session and the CLI). Built-in resume via `Last-Event-ID`. The DB-backed log gives history, search and audit for free.
- Negative: one-way only. Long-lived connections need proxy timeouts configured, which heartbeats help with. Each open stream holds a goroutine and a LISTEN subscription share, so there are per-user caps.

## References

- WHATWG HTML Living Standard — Server-sent events
- [Provisioning engine §9](../provisioning-engine.md), [API design §7](../api-design.md)
