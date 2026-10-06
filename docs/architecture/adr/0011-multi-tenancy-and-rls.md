# ADR-0011: Multi-tenancy by organization/project scoping with PostgreSQL Row-Level Security as defence in depth

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0011-multi-tenancy-and-rls.ru.md)

## Context

The architecture must be multi-tenant-ready from the start (prompt §86). Every resource has an ownership chain Organization → Project → Cluster and an authorization boundary. Tenant isolation must be checked at every API level and never rely on the frontend (prompt §211–212). Enterprise adds many organizations, teams and policies later, without a schema redesign.

## Decision

- **Data model**: every tenant-owned table has `org_id`, and project-scoped tables also have `project_id`. Community runs one default organization and project on the same schema.
- **Application layer (primary control)**: every repository method requires a `TenantScope`, derived from the authenticated principal after authorization. Every use case calls `Authorizer.Authorize(principal, permission, resource)` before side effects. Jobs carry `org_id` and re-validate ownership.
- **Database layer (defence in depth)**: PostgreSQL **RLS** with `FORCE ROW LEVEL SECURITY` on tenant tables. Policies compare `org_id` with `current_setting('app.org_id')`, which is set by `SET LOCAL` in each transaction (safe with connection pools). The app role has no `BYPASSRLS`. A separate system role is used only for cross-tenant maintenance jobs.
- **API behaviour**: ids are UUIDv7. Foreign or unknown ids return `404`, not `403`.
- **Tests**: tenant-escape tests for every repository and endpoint, including SSE and search.

## Alternatives considered

| Option | Why not |
|---|---|
| Database (or schema) per tenant | Strong isolation, but heavy operations (migrations × N, connection pools), poor fit for Community single-tenant installs and for SaaS with many small tenants. It could still be offered later for dedicated enterprise deployments, since it is orthogonal |
| App-layer checks only | One missed `WHERE org_id = …` leaks data. RLS turns such bugs into empty results |
| RLS only | Authorization semantics (roles, scopes, ABAC) don't belong in SQL policies, and the error messages and auditing would be poor |

## Consequences

- Positive: two independent layers must both fail for a cross-tenant leak, and the model is ready for enterprise multi-org without redesign.
- Negative: RLS adds query-planning considerations (index on `org_id`), requires careful role setup and `SET LOCAL` discipline in the transaction manager, and complicates ad-hoc SQL debugging. These are mitigated by the tx manager abstraction and tests.

## References

- [Data model §4](../data-model.md), [Security model §5](../../security/SECURITY_MODEL.md)
