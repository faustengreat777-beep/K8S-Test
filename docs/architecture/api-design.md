# API design

> Status: **Proposed** (Phase 0). Language: English · [Русский](api-design.ru.md)
>
> Related: [ARCHITECTURE](../ARCHITECTURE.md) · [Provisioning engine](provisioning-engine.md) · [Security model](../security/SECURITY_MODEL.md) · ADR-0005 (API style and contract)

Farvater is **API-first** (prompt §40, §57). The web UI, the CLI (`farvater`), the future Terraform/OpenTofu provider and customer automation are all clients of the same public HTTP API. No client contains business logic. No client talks to infrastructure directly.

## 1. Style and contract

| Aspect | Decision |
|---|---|
| Protocol | HTTPS, JSON (`application/json`), UTF-8 |
| Contract | **OpenAPI 3.1**, produced **contract-first in Go** with **Huma v2** on the chi router: typed operation definitions (request/response structs with validation tags) generate the OpenAPI 3.1 document, JSON Schemas, request validation and RFC 9457 errors. The generated `api/openapi/openapi.yaml` is **committed**; every PR shows the contract diff, **oasdiff** blocks breaking changes in v1, and **vacuum** lints it. The TypeScript client is generated from the committed document (`openapi-typescript` + `openapi-fetch`); the CLI uses `pkg/client`, a thin typed client over the shared request/response types (or an oapi-codegen client if an external consumer needs one). A CI job fails when the committed spec or generated clients are stale. |
| Base path / versioning | `/api/v1`. Within v1 only **additive** changes (new endpoints, new optional fields, new enum values that clients must tolerate). Breaking changes → `/api/v2` running side by side. Deprecated endpoints send `Deprecation` and `Sunset` headers (RFC 9745 / RFC 8594) for at least two minor releases. |
| Naming | Plural nouns, kebab-case paths, camelCase JSON fields, ids are UUIDv7 strings, timestamps RFC 3339 UTC |
| Long-running work | `202 Accepted` + `Operation` resource + `Location: /api/v1/operations/{id}` |
| Errors | RFC 9457 `application/problem+json` with stable codes (section 6) |
| Real-time | Server-Sent Events for operation progress (section 7) |
| Docs | Rendered reference from the spec (served at `/api/docs` in the product and published with each release); `docs/API.md` is the human guide |

Why code-first with Huma rather than spec-first server generation: in October 2026 Go server generators support OpenAPI 3.1 only partially (oapi-codegen calls its 3.1 support "initial", ogen needs pre-processors). Huma emits valid 3.1 natively from typed Go, and the committed spec plus breaking-change gating keeps the contract reviewable before merge (ADR-0005). Every response in integration tests is also validated against the committed spec.

## 2. Resource model and endpoints (v1)

Tenant scoping: **collections** live under their parent (`/organizations/{orgId}/…`, `/projects/{projectId}/…`), so tenancy is visible in the URL and in access logs. **Items** have globally unique ids and are addressable directly (`/clusters/{clusterId}`), which keeps CLI ergonomics simple. Authorization on every call resolves the item's org/project and checks the principal's permission there. The URL never grants access by itself.

### 2.1 Identity, tenancy, access

| Method & path | Purpose | Permission |
|---|---|---|
| `POST /api/v1/auth/login` · `POST /auth/logout` | Local login/logout (session cookie) | public / authenticated |
| `GET /api/v1/me` · `GET /me/sessions` · `DELETE /me/sessions/{id}` | Current user, session list, revoke | authenticated |
| `GET/POST /api/v1/organizations` · `GET/PATCH /organizations/{orgId}` | Organizations (Community: one) | `organization:read`/`organization:manage` |
| `GET/POST /organizations/{orgId}/projects` · `GET/PATCH/DELETE /projects/{projectId}` | Projects | `project:*` |
| `GET/POST /organizations/{orgId}/members` · `DELETE …/members/{userId}` | Membership | `organization:manage` |
| `GET/POST /organizations/{orgId}/teams` · `…/teams/{teamId}/members` | Teams | `team:*` |
| `GET/POST /organizations/{orgId}/roles` · `GET/PATCH/DELETE /roles/{roleId}` | Built-in + custom roles (custom = ee) | `role:*` |
| `GET/POST /organizations/{orgId}/role-bindings` · `DELETE /role-bindings/{id}` | Grant roles at org/project/cluster scope | `role:bind` |
| `GET/POST /organizations/{orgId}/api-keys` · `DELETE /api-keys/{id}` | API keys (secret returned **once** on create) | `apikey:*` |

### 2.2 Credentials and infrastructure

| Method & path | Purpose | Permission |
|---|---|---|
| `GET/POST /organizations/{orgId}/credentials` · `GET/PATCH/DELETE /credentials/{id}` | Credential vault. Secret fields are **write-only**; reads return metadata (type, fingerprint, last used, expiry) | `credentials:read/create/delete` |
| `POST /credentials/{id}/rotate` | Rotate (new material; old kept decrypt-only until dependent clusters are updated) | `credentials:rotate` |
| `GET/POST /organizations/{orgId}/provider-accounts` · `GET/PATCH/DELETE /provider-accounts/{id}` | Configured infrastructure connections | `provider:*` |
| `POST /provider-accounts/{id}/verify` | Validate config + credentials (async → Operation of type `preflight`) | `provider:read` |
| `GET /provider-accounts/{id}/inventory` | Regions, sizes, images, prices (if the provider supports them) | `provider:read` |
| `GET /api/v1/providers` | Registered provider types, capabilities, config JSON Schemas | authenticated |

### 2.3 Clusters

| Method & path | Purpose | Permission |
|---|---|---|
| `GET /projects/{projectId}/clusters` | List (filter: `phase`, `health`, `environment`, `label`; sort; paginate) | `cluster:read` |
| `POST /projects/{projectId}/clusters` | Create a cluster **record** from a ClusterSpec (phase `DRAFT`); body may be YAML (`application/yaml`) or JSON | `cluster:create` |
| `GET /clusters/{id}` | Cluster with status, health, endpoints, current spec revision | `cluster:read` |
| `PUT /clusters/{id}/spec` | Save a new desired spec revision (requires `If-Match`) — does not change infrastructure | `cluster:update` |
| `GET /clusters/{id}/spec-revisions` · `GET …/spec-revisions/{rev}` · `GET …/spec-revisions/{rev}/diff?against={rev2}` | History and diffs (prompt §97) | `cluster:read` |
| `POST /clusters/{id}/plan` | Dry run: validate → resolve → compatibility → policy → plan. Returns the plan, diff, irreversible steps, estimates. **No side effects** | `cluster:read` |
| `POST /clusters/{id}/deploy` | Apply the latest (or given) spec revision: `create` for a new cluster, `apply` otherwise → `202` Operation. Body: `{revision, planHash, confirmIrreversible}` | `deployment:create` + `cluster:update` |
| `POST /clusters/{id}/upgrade` | Kubernetes upgrade workflow (`targetVersion`) → `202` Operation | `cluster:upgrade` |
| `GET /clusters/{id}/upgrade-advisor?target=1.38` | Readiness score, blockers, warnings, deprecated APIs in use | `cluster:read` |
| `POST /clusters/{id}/scale` | Change worker pool sizes or add/remove declared hosts → `202` | `cluster:scale` |
| `DELETE /clusters/{id}` | Destroy (requires `confirm: <cluster-name>`) → `202` Operation of type `destroy` | `cluster:delete` |
| `GET /clusters/{id}/kubeconfig` | Download a kubeconfig. Default: a **user-scoped, short-lived client certificate** (TTL configurable), audited; admin kubeconfig only with `cluster:admin-kubeconfig` | `cluster:connect` |
| `GET /clusters/{id}/health` | Per-probe health + aggregate | `cluster:read` |
| `GET /clusters/{id}/nodes` · `GET …/nodes/{nodeId}` | Node inventory with facts, conditions, utilization (from metrics if installed) | `node:read` |
| `POST /clusters/{id}/nodes/{nodeId}/cordon` · `/uncordon` · `/drain` · `/restart` · `/replace` · `DELETE …/nodes/{nodeId}` | Node actions (drain/replace/remove → Operations) | `node:cordon` / `node:drain` / `node:delete` |
| `GET /clusters/{id}/addons` · `POST /clusters/{id}/addons` · `PATCH/DELETE /clusters/{id}/addons/{addonId}` | Add-on management (install/upgrade/remove → Operations) | `addon:install/update/delete` |
| `GET /clusters/{id}/backups` · `POST /clusters/{id}/backups` · `POST /clusters/{id}/restore` | Backup/restore (Phase 6) | `cluster:backup` / `cluster:restore` |
| `POST /clusters/{id}/rotate-certificates` · `/rotate-credentials` | Rotations → Operations | `cluster:update` |
| `POST /projects/{projectId}/clusters:import` | Import existing cluster by kubeconfig (Phase 6) | `cluster:create` |

### 2.4 Operations (the product's "Deployments")

| Method & path | Purpose | Permission |
|---|---|---|
| `GET /clusters/{id}/operations` · `GET /projects/{projectId}/operations` | Lists | `deployment:read` |
| `GET /operations/{id}` | Status, plan summary, timing, error report | `deployment:read` |
| `GET /operations/{id}/tasks` | Task tree with statuses, attempts, durations | `deployment:read` |
| `GET /operations/{id}/events` | **SSE** live stream with replay (`Last-Event-ID`) | `deployment:read` |
| `GET /operations/{id}/logs?level=&task=&node=&q=&pageToken=` | Searchable, filterable structured logs (JSON or `text/plain` download) | `deployment:read` |
| `POST /operations/{id}/cancel` · `/resume` · `/retry` (optionally `{taskKey}`) · `/rollback` | Control actions | `deployment:cancel` / `deployment:retry` |

### 2.5 Catalog, validation, Auto Mode

| Method & path | Purpose |
|---|---|
| `GET /api/v1/catalog` | Catalog version and channel |
| `GET /api/v1/catalog/kubernetes-versions` | Minors with status (current/supported/deprecated/blocked), patches, EOL |
| `GET /api/v1/catalog/addons?category=` · `GET /catalog/addons/{id}` | Add-on metadata, versions, config schema, pros/cons |
| `GET /api/v1/catalog/presets` | Presets (Minimal, Development, Staging, Production, High Availability, Edge, GPU, AI/ML, Custom) |
| `GET /api/v1/schemas/cluster-spec` | JSON Schema of ClusterSpec (for Monaco/YAML editors and the CLI) |
| `POST /api/v1/validate` | Validate a ClusterSpec: schema + compatibility + guardrails/policies → findings (no side effects) |
| `POST /api/v1/recommendations` | Auto Mode: answers (environment, size, availability, workload, budget, compliance, provider) → recommended ClusterSpec + per-field explanations + warnings + estimates |
| `POST /api/v1/preflight` | Live preflight against hosts/provider → `202` Operation of type `preflight` with per-check results |

### 2.6 Templates, audit, webhooks, notifications, search

| Method & path | Purpose |
|---|---|
| `GET/POST /organizations/{orgId}/templates` · `GET /templates/{id}` · `POST /templates/{id}/versions` · `PATCH /templates/{id}/versions/{v}` (status) | Cluster templates with versions, inheritance, status (prompt §33, §191–192) |
| `POST /templates/{id}/versions/{v}/instantiate` | Create a cluster draft from a template |
| `GET /organizations/{orgId}/audit-events?actor=&action=&target=&cluster=&project=&ip=&from=&to=&result=` · `GET …/audit-events:export?format=json|csv` | Audit search and export |
| `GET/POST /organizations/{orgId}/webhooks` · `GET /webhooks/{id}/deliveries` · `POST /webhooks/{id}/deliveries/{deliveryId}/replay` · `POST /webhooks/{id}/test` | Webhooks |
| `GET /me/notifications` · `POST /me/notifications/{id}/read` · `GET/POST /organizations/{orgId}/notification-channels` | Notifications |
| `GET /api/v1/search?q=&types=clusters,nodes,operations,addons,users` | Global search (prompt §93), authorization-filtered |

### 2.7 Platform

| Path | Purpose |
|---|---|
| `GET /healthz` | Liveness (process up) — unauthenticated, outside `/api` |
| `GET /readyz` | Readiness (DB reachable, migrations applied, worker heartbeat for `worker` role) |
| `GET /metrics` | Prometheus metrics (separate listener/port, not public by default) |
| `GET /api/v1/version` | Server version, edition, catalog version, API version |

### 2.8 Enterprise (ee, later phases)

`/api/v1/policies`, `/policy-bindings`, `/change-requests` (+ `/approve`, `/reject`, `/execute`), `/compliance`, `/fleet`, `/cluster-groups`, `/integrations`, `/identity-providers`, `/service-accounts`, `/incidents`, `/budgets`, `/quotas`, `/scim/v2/*` (SCIM 2.0, RFC 7643/7644). They use the same conventions. Entitlements decide availability, so a Community server returns `403` with code `FEATURE_NOT_LICENSED`.

## 3. Representations

Cluster (abridged):

```json
{
  "id": "0199a7c2-5b1e-7c3a-9f0e-2d4b6a8c1e3f",
  "name": "production",
  "projectId": "0199a7c2-…",
  "environment": "production",
  "phase": "READY",
  "health": "HEALTHY",
  "kubernetesVersion": "1.37.1",
  "distribution": "kubeadm",
  "apiEndpoint": "https://10.0.0.10:6443",
  "desiredRevision": 4,
  "appliedRevision": 4,
  "activeOperationId": null,
  "nodeCounts": { "controlPlane": 3, "worker": 3 },
  "labels": { "team": "payments" },
  "createdAt": "2026-10-06T12:30:00Z",
  "updatedAt": "2026-10-06T12:47:12Z",
  "etag": "\"17\""
}
```

Operation (abridged):

```json
{
  "id": "0199a7c3-…",
  "clusterId": "0199a7c2-…",
  "type": "create",
  "status": "RUNNING",
  "progress": { "tasksTotal": 64, "tasksSucceeded": 41, "tasksRunning": 2, "tasksFailed": 0, "percent": 66 },
  "currentPhase": "INSTALLING",
  "estimatedSecondsRemaining": 310,
  "irreversibleSteps": [],
  "error": null,
  "links": { "events": "/api/v1/operations/0199a7c3-…/events", "tasks": "/api/v1/operations/0199a7c3-…/tasks" },
  "startedAt": "2026-10-06T12:31:02Z"
}
```

Plan (`POST /clusters/{id}/plan`, abridged):

```json
{
  "planHash": "sha256:9c1…",
  "changes": [
    { "op": "add", "kind": "nodes", "summary": "+ 3 control plane nodes" },
    { "op": "add", "kind": "nodes", "summary": "+ 3 worker nodes" },
    { "op": "add", "kind": "kubernetes", "summary": "+ Kubernetes 1.37.1 (kubeadm)" },
    { "op": "add", "kind": "addon", "id": "cilium", "summary": "+ Cilium 1.x.y" }
  ],
  "irreversibleSteps": [],
  "findings": [ { "severity": "WARNING", "code": "NO_BACKUP_DESTINATION", "path": "/spec/backup" } ],
  "estimate": { "durationSeconds": { "min": 720, "max": 1080 }, "cost": { "available": false, "reason": "PROVIDER_HAS_NO_PRICING" } },
  "tasks": 64
}
```

Values in examples are illustrative. Real versions come from the catalog.

## 4. Pagination, filtering, sorting

- **Cursor pagination** on all list endpoints: `?pageSize=50&pageToken=…`. The response has `{ "items": [...], "nextPageToken": "…" }`. Tokens are opaque (encrypted sort keys + filter hash), and a token used with different filters is rejected with `400 INVALID_PAGE_TOKEN`. Max `pageSize` is 200. Total counts only on request (`?includeTotal=true`), because counting is expensive at scale.
- **Filtering** with explicit, documented query parameters per collection (`phase`, `health`, `environment`, `label=key=value`, `createdAfter`, …). There is no free-form filter language in v1; a CEL-based `filter=` expression is possible in a later version.
- **Sorting** with `orderBy=createdAt desc,name` over an allow-listed set of fields per collection.

## 5. Idempotency, concurrency, async

- **Idempotency-Key** header (UUID or any opaque string ≤ 255 chars) is **required** on `POST` endpoints that create resources or start operations from the CLI and automation. The web UI sends it on every such call. The server stores `(org, principal, key) → request hash + response` for 24 hours:
  - same key + same request → original response replayed (`Idempotent-Replayed: true`);
  - same key + different request → `422 IDEMPOTENCY_KEY_REUSED`;
  - request still in progress → `409 IDEMPOTENCY_IN_PROGRESS`.
- **Optimistic concurrency**: mutable resources return `ETag`. `PUT`/`PATCH` require `If-Match` (otherwise `428 PRECONDITION_REQUIRED`); a mismatch → `412 PRECONDITION_FAILED` with the current version.
- **Async jobs**: all infrastructure-changing endpoints return `202` with an Operation. Synchronous endpoints never block on infrastructure.
- **Operation conflicts**: starting an operation while another mutating operation runs on the cluster → `409 OPERATION_IN_PROGRESS` with `activeOperationId`.

## 6. Errors

`application/problem+json` (RFC 9457) with extensions:

```json
{
  "type": "https://docs.farvater.io/errors/K8S_JOIN_API_UNREACHABLE",
  "title": "Worker node failed to join the cluster",
  "status": 409,
  "code": "K8S_JOIN_API_UNREACHABLE",
  "detail": "Worker node node-04 cannot reach the Kubernetes API server on port 6443.",
  "instance": "/api/v1/operations/0199a7c3-…",
  "requestId": "req_01J…",
  "location": { "operationId": "0199a7c3-…", "taskKey": "k8s.worker.join/node-04", "node": "node-04", "address": "10.0.0.14" },
  "remediation": [
    "Check firewall rules between node-04 and the control-plane endpoint on port 6443.",
    "Check routing from node-04 to 10.0.0.10.",
    "Verify the API server address in the cluster spec."
  ],
  "actions": ["retry", "resume", "view-logs"],
  "errors": []
}
```

- `code` is stable and documented (error catalog). `title`, `detail` and `remediation` are localized by `Accept-Language` (en, ru).
- Validation errors → `422` with `errors: [{ "path": "/spec/networking/cni/provider", "code": "UNSUPPORTED_VALUE", "message": "…" }]`.
- Auth: `401 UNAUTHENTICATED`, `403 FORBIDDEN` (with missing permission name, never with details about resources the caller cannot see; unknown or foreign ids return `404` to prevent enumeration).
- Never in errors: secrets, stack traces, internal hostnames of the platform, raw SQL errors. The server logs the internal cause with the same `requestId`.
- Generic "Something went wrong" is used only for truly unexpected failures (`500 INTERNAL`), always with a `requestId` (prompt §83).

## 7. Real-time updates (SSE)

`GET /api/v1/operations/{id}/events` with `Accept: text/event-stream`:

```
id: 18342
event: task-status
data: {"taskKey":"addon/cilium.install","status":"SUCCEEDED","durationMs":48211}

id: 18343
event: log
data: {"ts":"2026-10-06T12:31:08Z","level":"INFO","taskKey":"node/cp-1/runtime.install","node":"cp-1","message":"containerd installed"}

: heartbeat

event: end
data: {"status":"SUCCEEDED"}
```

- Event types: `log`, `task-status`, `operation-status`, `progress`, `end`.
- Resume after reconnect using the standard `Last-Event-ID` header (or `?after=` for clients that can't set headers). The server replays from the database.
- Authentication: session cookie (same-origin `EventSource`) or `Authorization: Bearer` for the CLI's streaming client.
- Also available: `GET /api/v1/events?clusterId=` (project/cluster-level stream of operation and health changes for dashboards). It is rate-limited and capped per user.

## 8. Authentication and authorization

- **Browser**: `POST /auth/login` sets the cookie `__Host-session` (`HttpOnly; Secure; SameSite=Lax; Path=/`). Unsafe methods require the `X-CSRF-Token` header (synchronizer token bound to the session; the token is provided in `GET /me`). Sessions rotate on login and privilege change.
- **CLI / automation**: `Authorization: Bearer fvt_…` API keys (prefix for lookup + secret verified against a hash). Later: OAuth 2.0 device flow through the configured OIDC provider (ee) and service accounts.
- **Authorization** is checked in the application layer for every request (`Authorizer.Authorize(principal, permission, resource)`) with org/project/cluster scope. Permissions follow `resource:verb` (see [Security model](../security/SECURITY_MODEL.md#authorization)). API keys can only narrow, never widen, their owner's permissions.
- **Kubeconfig downloads** are themselves audited operations and produce short-lived credentials by default.

## 9. Webhooks

- Event types (prompt §68): `deployment.started`, `deployment.completed`, `deployment.failed`, `cluster.created`, `cluster.deleted`, `cluster.upgraded`, `backup.completed`, `backup.failed`. More can be added, for example `cluster.health_changed`, `operation.paused`, `credential.expiring`.
- Format follows the **Standard Webhooks** specification. Headers `webhook-id`, `webhook-timestamp`, `webhook-signature` (`v1,` + HMAC-SHA256 over `id.timestamp.body` with a per-endpoint secret). The JSON body has `type`, `timestamp`, `data` (redacted public representation).
- Delivery runs from the transactional outbox through River jobs: at-least-once, exponential backoff for up to 24 h, then `dead`. Delivery history, manual replay and a test endpoint are available. The target URL is checked by the SSRF guard on save and on every delivery.
- Receivers must deduplicate by `webhook-id`.

## 10. Rate limiting and abuse protection

- Token buckets per **API key / user**, per **organization** and per **source IP** (unauthenticated endpoints, especially login). Separate budgets for read and mutating endpoints, and tighter limits for expensive ones (`plan`, `preflight`, `kubeconfig`).
- Responses: `429 RATE_LIMITED` with `Retry-After` and `RateLimit-*` headers (IETF RateLimit header fields).
- Implementation: in-process limiter per replica in Community/MVP. A shared store (PostgreSQL-backed, or Valkey for large HA installs) when running several API replicas behind a load balancer.
- Login protection: per-account and per-IP backoff, generic error messages ("invalid email or password"), audit of failures.

## 11. CLI mapping

The CLI uses the generated Go client. Commands map 1:1 to API calls and print the same error codes and remediation:

```
farvater login --server https://… [--api-key-stdin]
farvater cluster create -f cluster.yaml [--project payments]      # POST …/clusters (DRAFT)
farvater cluster plan production                                   # POST /clusters/{id}/plan
farvater cluster deploy production [--watch] [--yes]               # POST /clusters/{id}/deploy + SSE stream
farvater cluster list | get production | delete production
farvater cluster upgrade production --to 1.38 [--plan]
farvater cluster scale production --pool workers=6
farvater kubeconfig production > ~/.kube/production
farvater addon install production cert-manager [--set key=value]
farvater operation watch <id> | logs <id> [--follow] | resume <id> | retry <id> [--task …] | cancel <id>
farvater recommend --provider baremetal --env production --size medium --availability ha -o yaml
farvater validate -f cluster.yaml
```

The prompt's names `clusterctl` and `platformctl` are not used, because `clusterctl` is the CLI of the Kubernetes Cluster API project (see ADR-0002).
