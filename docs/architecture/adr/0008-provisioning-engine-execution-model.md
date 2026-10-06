# ADR-0008: Own DAG provisioning engine; one leased job per operation; Temporal not adopted for now

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0008-provisioning-engine-execution-model.ru.md)

## Context

The deployment engine is the most important component (prompt §10). Deployments must be decomposed into tasks with dependencies (DAG, prompt §11). They must be idempotent and resumable after interruption (prompt §12), support rollback where safe (prompt §13), follow a controlled state machine (prompt §14), stream progress in real time (prompt §15) and run outside HTTP requests (prompt §41). The spec lists Redis+BullMQ, RabbitMQ, NATS and Temporal as queue candidates and asks for a reasoned choice.

## Decision

- Build an **own provisioning engine** in `internal/engine`:
  - a **planner** turns ResolvedSpec + observed state into a DAG of tasks contributed by plugins (provider, bootstrap layer, distribution, add-ons via capability dependencies);
  - an **executor** runs the DAG with bounded concurrency (global / per cluster / per node), typed errors, retry policies, timeouts, circuit breakers;
  - a **task contract** `Check / Run / Rollback` with mandatory idempotency;
  - **persistence-first**: every task transition is written to PostgreSQL before and after execution; events are appended for SSE.
- **Execution unit = one River job per Operation** (ADR-0007), protected by an **execution lease** (heartbeat, expiry, fencing on every write). A crashed worker's lease expires and another worker resumes from the persisted task states.
- Default failure policy is **pause** (with explanation and user actions). Rollback is only automatic for reversible tasks.
- Engine core has no knowledge of specific providers, distributions or add-ons.
- Details: [Provisioning engine](../provisioning-engine.md).

## Alternatives considered

| Option | Assessment |
|---|---|
| **Temporal** (durable workflows) | Strong durability and retries. But it adds a separately operated cluster (frontend/history/matching services + its own DB, optionally Elasticsearch). That is heavy for self-hosted and air-gapped installs. Workflow determinism constraints complicate dynamic DAGs built from data, and versioning workflow code across releases is non-trivial. Our needs (DAG, leases, resume, events) fit comfortably in PostgreSQL. **Revisit** if we need cross-region durable orchestration at fleet scale (Phase 11+); the engine's interfaces would let an executor backed by Temporal replace the current one. |
| One queue job per task (fan-out in the queue) | Simple per-task retries, but global concurrency limits, failure policy and ordering need cross-job coordination, which makes it harder to reason about and to pause or resume as a unit. |
| Argo Workflows / Tekton in a management cluster | Requires a management Kubernetes cluster before the first cluster exists (chicken-and-egg for bare metal); YAML workflow definitions conflict with our typed tasks. |
| Ansible playbooks as the engine | Weak typing and error classification, no first-class resume or DAG state, Python runtime dependency, injection surface via templated YAML. |

## Consequences

- Positive: full control over semantics (pause/resume, irreversible steps, explainable failures), PostgreSQL-only operations, testability with a simulated provider and deterministic scheduling.
- Negative: we own the complexity of a workflow engine (leases, fencing, recovery). This is mitigated by a small, well-tested core, chaos tests (kill-the-worker) and keeping the engine generic.

## References

- [Provisioning engine](../provisioning-engine.md), ADR-0007 (queue), ADR-0009 (real-time)
