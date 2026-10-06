# ADR-0003: Modular monolith with one server binary and process roles

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0003-modular-monolith.ru.md)

## Context

The platform needs an API, background workers for long-running provisioning, a web UI, a CLI, database migrations, and later enterprise modules. The spec asks for production-grade reliability, a simple local workflow (`make dev`), self-hosted and air-gapped installation, and an HA option for enterprise (prompt §43, §77, §179–182). The team is small. The domain is tightly coupled transactionally: creating an operation must atomically write cluster state, tasks, audit, outbox events and the job.

## Decision

- Build a **modular monolith in Go**: one repository, one Go module, one **server binary** with process roles selected by subcommand: `api`, `worker`, `all`, `migrate`, `admin`, `version`. The CLI is a separate small binary.
- Module boundaries are explicit packages under `internal/` (domain, app, engine, api, authn, authz, secrets, catalog, compat, recommend, health, audit, …) with import rules enforced in CI (see ADR-0004).
- Roles share code but differ in privileges. Only `worker` can decrypt credentials and reach customer infrastructure. `api` never can (trust boundary, see the security model).
- Scale out by running more `api` and `worker` replicas. PostgreSQL coordinates them (ADR-0007, ADR-0008).

## Alternatives considered

| Option | Why not (now) |
|---|---|
| Microservices (separate API, engine, notification, audit services) | Distributed transactions or sagas for every state change, more infrastructure (service mesh, message bus), harder self-hosting and air-gap, no clear scaling benefit at our size |
| Single process without roles | Can't separate the credential-holding tier from the internet-facing tier; can't scale workers independently |
| Serverless functions | Long-running SSH/Helm operations, air-gap and self-hosting make it unsuitable |

## Consequences

- Positive: simple deployment (a single binary or image), ACID transactions across modules, easy local development, small air-gap footprint, fewer moving parts to secure.
- Positive: a module can be extracted into a service later because its boundary is already an interface.
- Negative: risk of erosion of module boundaries. This is mitigated by import rules in CI and code review.
- Negative: one release cadence for all modules. This is acceptable at this stage.

## References

- [ARCHITECTURE](../../ARCHITECTURE.md), [Deployment model](../deployment-model.md), [Repository structure](../repository-structure.md)
