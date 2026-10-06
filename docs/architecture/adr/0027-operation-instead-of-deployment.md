# ADR-0027: Model "Deployment" from the spec as Operation / OperationTask

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0027-operation-instead-of-deployment.ru.md)

## Context

The spec names the long-running unit of change `Deployment` and its steps `DeploymentTask` (prompt §11, §58). The same mechanism is also needed for upgrades, scaling, add-on changes, backups, restores, rotations, node replacement, imports and destroys (prompt §29). In a Kubernetes product, "Deployment" also means the `apps/v1 Deployment` kind, which causes ambiguity in code, API and logs.

## Decision

- In code, the database and the API, the unit is **`Operation`** with a `type` (`create`, `apply`, `upgrade`, `scale`, `addon-install`, `addon-remove`, `backup`, `restore`, `rotate-certificates`, `rotate-credentials`, `node-replace`, `import`, `destroy`, `preflight`), and its steps are **`OperationTask`**.
- UI copy uses natural language: "Deploying…", "Deployment in progress", "Upgrading…".
- Webhook event names from the spec are kept for compatibility: `deployment.started|completed|failed` (fired for operations of types that change cluster infrastructure). Permission names use `deployment:*` as in the spec (prompt §138).

## Alternatives considered

- **Keep `Deployment` everywhere**: ambiguous next to Kubernetes `Deployment`, and misleading for destroy, backup and restore.
- **`Run` / `Job`**: `Job` collides with Kubernetes and River jobs, and `Run` is vague.

## Consequences

- Positive: unambiguous code and API. One mechanism covers all lifecycle actions (Google AIP-151 "long-running operations" is a similar concept).
- Negative: a mapping to the spec's wording must be documented. This ADR and the data model record it.
