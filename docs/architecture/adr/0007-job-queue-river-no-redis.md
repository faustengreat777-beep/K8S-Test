# ADR-0007: River (PostgreSQL-backed) job queue; PostgreSQL for pub/sub and locks; no Redis requirement

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0007-job-queue-river-no-redis.ru.md)

## Context

Deployments must run asynchronously: API → job queue → worker → deployment engine (prompt §41). The spec mentions Redis + BullMQ, RabbitMQ, NATS and Temporal as options and asks for a reasoned choice. It also lists `REDIS_URL` among settings (prompt §84). Research (2026-10-06):
- **River v0.49** (MPL-2.0 core, pre-1.0) offers transactional enqueue, unique jobs, retries with backoff, scheduled/periodic jobs, priorities, multiple queues and leader election. Its *workflows*, batching and concurrency limits are only in the commercial River Pro, which we don't need.
- **asynq** needs Redis and can't enqueue transactionally with PostgreSQL.
- **Temporal** (MIT) needs a multi-service cluster plus schema upgrades.
- **NATS** stays Apache-2.0 under the Linux Foundation.
- **Redis 8** is tri-licensed RSAL/SSPL/**AGPL**.
- **Valkey 9.1** is BSD-3 (Linux Foundation).
- **BullMQ** is a Node.js library.

## Decision

- **Job queue: River** with the pgx v5 driver. Jobs are inserted **in the same transaction** as the domain changes that need them, so there are no lost or phantom jobs. Uses: `operation.run` (unique per operation, ADR-0008), health polling (periodic), webhook and notification delivery, retention and maintenance, catalog sync.
- We implement DAGs ourselves (ADR-0008) and **do not depend on River Pro** features. River is used unmodified (MPL-2.0 file-level copyleft is satisfied by using it unmodified, and is tracked in NOTICE). The River version is pinned because of pre-1.0 breaking changes, and upgrades are tested.
- **Pub/sub**: PostgreSQL `LISTEN/NOTIFY` (SSE fan-out, cancellation signals, session revocation broadcast).
- **Locks and coordination**: PostgreSQL advisory locks (migrations, per-org audit chain) and lease rows with fencing (operations).
- **No Redis requirement**: `REDIS_URL` from the spec is dropped. **Valkey** (BSD-3) is an *optional* backend for shared rate limiting and caching in very large HA installs. Redis itself is not bundled because of its license.
- The queue sits behind a `JobQueue` port, so replacing River later only touches one adapter.

## Alternatives considered

| Option | Why not |
|---|---|
| Redis + BullMQ | Node.js; we use Go |
| Redis/Valkey + asynq | Extra stateful service, no transactional enqueue with our DB (dual-write problem), slow release cadence |
| RabbitMQ | Extra stateful broker and operational burden, no transactional enqueue with PostgreSQL (needs an outbox anyway) |
| NATS JetStream | Excellent for future edge or agent messaging, but an extra cluster today and the same dual-write problem. Revisit for fleet or edge agents |
| Temporal | See ADR-0008: heavy for self-hosted and air-gap. Possible optional integration later |
| DBOS Transact Go (MIT, durable workflows on PostgreSQL) | A promising library-only alternative, but young (1.x in 2026). It is worth a spike before Phase 2 as a possible replacement for parts of our executor |

## Consequences

- Positive: one stateful dependency, atomic state + job changes, simpler air-gap and HA, no copyleft or source-available datastore in the default install.
- Negative: queue throughput is bounded by PostgreSQL. That is plenty for provisioning workloads (operations are long and coarse), and per-replica rate limits need Valkey at large scale. River is pre-1.0, so upgrades need care.

## References

- River docs and CHANGELOG; Redis and Valkey license files; NATS/CNCF statement (2025-05-01)
- [Provisioning engine](../provisioning-engine.md), ADR-0008
