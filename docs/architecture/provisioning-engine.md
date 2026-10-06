# Provisioning engine

> Status: **Proposed** (Phase 0). Language: English · [Русский](provisioning-engine.ru.md)
>
> Related: [ARCHITECTURE](../ARCHITECTURE.md) · [Core interfaces](core-interfaces.md) · [Data model](data-model.md) · [Auto Mode, catalog and compatibility](auto-mode-and-catalog.md) · ADR-0007 (job queue), ADR-0008 (engine model), ADR-0009 (real-time updates)

The provisioning engine turns a desired **ClusterSpec** into a working cluster and then keeps changing it safely: upgrades, scaling, add-ons, backup, restore and destroy. It is the most important component of Farvater, so this document sets its contract precisely.

## 1. Design goals

| Goal | How the engine meets it |
|---|---|
| No giant `if/else` | The work is a **DAG of small tasks** contributed by plugins (provider, bootstrap layer, distribution, add-ons). The engine core knows nothing about kubeadm, Cilium or Hetzner. |
| Idempotent | Every task has *ensure / converge* semantics and an optional `Check` probe. Re-running a finished task is safe and cheap. |
| Resumable | Every state change is persisted **before** and **after** a task runs. A crashed worker or a closed browser loses nothing. Resume continues from the first unfinished task. |
| Observable | Every transition and log line becomes an `OperationEvent`. It is streamed to clients over SSE and kept for audit and search. |
| Safe rollback | Only tasks declared `Reversible` are rolled back automatically. Irreversible steps are flagged in the plan, and the user confirms them. |
| Explainable failures | Typed errors carry *what / why / where / how to fix*, plus the actions the user can take. |
| Deterministic plans | Same ResolvedSpec + same observed state → same plan. The plan is stored and hashed, so you can review it before applying it (dry run). |
| Testable | The engine core is pure Go with interfaces for time, storage, queue and plugins. The simulated provider and fault injection run full scenarios in unit/integration tests. |

## 2. Lifecycle of a change

```
ClusterSpec (UI / YAML / CLI / API / template / Auto Mode)
   │
   ▼
1. Validate        schema (strict) → semantic validators → secrets-free check
2. Resolve         catalog: minor → pinned patch, chart versions + digests, images  → ResolvedSpec
3. Check           compatibility engine (BLOCK/WARNING/INFO) → guardrails / policies → (ee) approvals
4. Preflight       live checks against hosts / provider (async operation of type `preflight`)
5. Plan            planner builds DAG from ResolvedSpec + observed state → diff, estimates, irreversible flags
6. Confirm         user reviews plan ("Deploy Cluster"); idempotency key binds the request
7. Execute         worker runs the DAG with leases, retries, events
8. Verify          health probes; cluster → READY (or PAUSED/FAILED with remediation)
```

Steps 1–5 are side-effect-free with respect to infrastructure. They power **Plan / Preview / Apply** everywhere ("dry-run everywhere", prompt §117). Step 4 reads hosts (SSH facts, ports) but changes nothing.

## 3. Domain objects

- **ClusterSpec**: the user-authored desired state (see [ARCHITECTURE §ClusterSpec](../ARCHITECTURE.md#clusterspec)). It never contains secrets, only `credentialRef`s.
- **ResolvedSpec**: the ClusterSpec with every version pinned and every default materialised, plus the catalog version used. It is stored in `cluster_spec_revisions.resolved_spec`.
- **ObservedState**: what the platform knows about reality, i.e. nodes and their facts, installed add-ons and versions, API endpoint and health. It is refreshed by discovery tasks and health probes.
- **Plan**: a DAG of `PlannedTask`s, plus a human-readable **diff** (`+ 3 control-plane nodes`, `~ cilium 1.19.4 → 1.20.2`, `- addon loki`), irreversible steps, time estimate and cost estimate (provider pricing or "unavailable"). It is hashed (`plan_hash`).
- **Operation**: one execution of a plan (`create`, `apply`, `upgrade`, `scale`, `addon-install`, `addon-remove`, `backup`, `restore`, `rotate-*`, `node-replace`, `import`, `destroy`, `preflight`). This is what the product spec calls a *Deployment*.
- **OperationTask**: one node of the DAG with its own status, attempts, timings, error and outputs (the product spec's *DeploymentTask*).
- **OperationEvent**: an append-only event (log line, status change, progress tick). Its `id` is the SSE event id.

## 4. Planner

The planner is a pure function:

```
Plan(resolved ResolvedSpec, observed ObservedState, registry PluginRegistry) (Plan, error)
```

It asks each participating plugin for **task contributions**:

| Contributor | Example task keys | Notes |
|---|---|---|
| Engine (pre) | `preflight.verify` | Re-validates the facts gathered by the preflight operation; fails fast if they are stale |
| Infrastructure provider | `infra.network.ensure`, `infra.instance.ensure/cp-1`, `infra.lb.ensure` | Bare metal: `instance.ensure` = adopt and verify a declared host |
| Bootstrap layer (per node) | `node/cp-1/os.prepare`, `…/kernel.modules`, `…/sysctl`, `…/swap.disable`, `…/time.sync`, `…/runtime.install`, `…/k8s.packages` | Shared by all distributions that need it; distributions can opt out (k3s ships its own runtime) |
| Distribution | `k8s.controlplane.init`, `k8s.controlplane.join/cp-2`, `k8s.worker.join/w-1`, `k8s.kubeconfig.fetch`, `k8s.node.upgrade/cp-1` | kubeadm first |
| Add-ons | `addon/gateway-api-crds.install`, `addon/cilium.install`, `addon/metallb.install`, `addon/cert-manager.install`, … | Ordering derived from add-on `requires`/`provides` capabilities |
| Engine (post) | `health.verify`, `cluster.ready` | Health probes from all plugins |

**Dependency resolution for add-ons.** Each add-on manifest declares capabilities:

```yaml
# catalog/addons/cert-manager.yaml (excerpt)
id: cert-manager
provides: [cert-issuer-crds, certificates]
requires: [cni]                 # nothing installs before networking is up
optionalRequires: [metrics]     # if metrics present, install ServiceMonitor after it
conflicts: []
reversible: true
```

```yaml
# catalog/addons/kube-prometheus-stack.yaml (excerpt)
id: kube-prometheus-stack
provides: [metrics, alerting, dashboards]
requires: [cni, default-storage-class]
```

The planner maps `requires` to the add-on that `provides` the capability *in this cluster* (for example `default-storage-class` → `longhorn` or `local-path`). It then builds edges and topologically sorts the graph. It rejects:
- unsatisfied requirements ("Prometheus needs a default StorageClass; choose a storage provider or disable persistence"),
- conflicts (two CNIs, two default StorageClasses),
- cycles. These are rejected when the catalog loads, so a cycle never reaches users.

**Diff, not rebuild.** For `apply`, `scale` and `upgrade`, the planner compares ResolvedSpec with ObservedState and emits only the needed tasks. Adding two workers emits `infra.instance.ensure/w-4,w-5` → bootstrap → `k8s.worker.join/w-4,w-5` → `health.verify`. Changing Longhorn values emits a single `addon/longhorn.upgrade`.

**Irreversible flags.** Tasks declare `Reversible: false` together with a risk class (`data-loss`, `downtime`, `security`, `cost`). The plan lists them under "Irreversible / risky steps". The API requires explicit confirmation (`confirmIrreversible: true`) for plans that contain them, except for the initial `create` of an empty cluster.

**Estimates.** Time estimates use the median duration of the same task kind from previous operations of this installation. Without history, they fall back to catalog defaults. The critical path through the DAG gives "Estimated deployment time: 12–18 min". Cost estimates come only from a provider's `Pricing` capability; otherwise the plan says "Cost estimation unavailable". The engine never invents prices.

### Example: create HA kubeadm cluster on bare metal (simplified DAG)

```
preflight.verify
  └─▶ node/*/os.prepare ─▶ node/*/kernel.modules ─▶ node/*/sysctl ─▶ node/*/swap.disable ─▶ node/*/runtime.install ─▶ node/*/k8s.packages
                                                                                                      │ (all nodes, in parallel)
        node/cp-1/k8s.packages ─▶ addon/kube-vip.static-pod(cp-1) ─▶ k8s.controlplane.init(cp-1) ─▶ k8s.kubeconfig.fetch
                                                                                                      │
              addon/gateway-api-crds.install ◀────────────────────────────────────────────────────────┤
              addon/cilium.install  ◀── requires gateway-api-crds (Cilium Gateway) ───────────────────┘
                     │
        ┌────────────┼───────────────────────────────┐
        ▼            ▼                               ▼
 k8s.controlplane.join(cp-2)  k8s.controlplane.join(cp-3)   k8s.worker.join(w-1..w-3)   (CP joins serialized: one etcd member at a time)
        └────────────┴───────────────┬───────────────┘
                                     ▼
          addon/coredns.verify ─▶ addon/metallb.install ─▶ addon/gateway.install ─▶ addon/longhorn.install
                 ─▶ addon/cert-manager.install ─▶ addon/metrics-server.install ─▶ addon/kube-prometheus-stack.install
                 ─▶ addon/loki.install ─▶ addon/backup.configure ─▶ health.verify ─▶ cluster.ready
```

Control-plane joins are serialized, because etcd membership changes must happen one at a time to keep quorum safe. Worker joins run in parallel up to the per-cluster concurrency limit.

## 5. Executor

### 5.1 Job model

- Creating an operation is **one database transaction**: insert the `operations` row, insert the `operation_tasks` rows from the plan, set the cluster phase, write the audit event, add an outbox event (`deployment.started`), and **enqueue the River job** `operation.run{operationID}`. All of this commits atomically, so there is never a job without an operation, or an operation without a job.
- `operation.run` is a River job with `unique` options keyed by operation id. Duplicates are impossible.
- The job handler acquires an **execution lease** on the operation row: `lease_owner = <worker-id>`, `lease_expires_at = now() + 30s`, renewed every 10 s by a heartbeat. If the worker dies, the lease expires. River retries the job (or the stuck-operation sweeper re-enqueues it), and a new worker continues.
- Inside the job, an in-process **DAG scheduler** runs ready tasks concurrently, with semaphores:
  - global per worker (`ENGINE_MAX_PARALLEL_TASKS`, default 32),
  - per cluster (default 10),
  - per node (default 1 mutating task at a time, so package managers do not fight over locks).
- Long waits (for example a `kubeadm init` that takes minutes) keep the lease alive through the heartbeat. Tasks receive a `context.Context` that is cancelled on lease loss, operation cancel or timeout.

Why a single job per operation instead of one job per task: the DAG scheduling, concurrency limits and failure policy need a global view of the operation. One owner simplifies them. Persistence after every task transition still gives task-level resume. ADR-0008 records this decision and the alternative (Temporal workflows) that was rejected.

### 5.2 Task contract

```go
// pkg/sdk/task.go (sketch)
type Task interface {
    Key() TaskKey                       // stable id inside the plan, e.g. "node/cp-1/runtime.install"
    Meta() TaskMeta                     // name i18n key, kind, node, reversible, risk class, timeout, retry policy
    // Check reports whether the desired end state is already reached. Optional (ErrNotImplemented → treated as false).
    Check(ctx context.Context, tc TaskContext) (done bool, err error)
    // Run converges the system to the desired state. Must be idempotent and safe to re-run after partial completion.
    Run(ctx context.Context, tc TaskContext) error
    // Rollback undoes Run for reversible tasks. Must be idempotent. Called only if Meta().Reversible.
    Rollback(ctx context.Context, tc TaskContext) error
}

type TaskContext interface {
    Logger() *slog.Logger               // pre-tagged with cluster/operation/task/node ids; secret values redacted
    Progress(pct int, msgKey string, args ...any)
    Node(id NodeID) (NodeHandle, error) // SSH-backed or simulated
    Kube() (KubeClient, error)          // client for the target cluster (after kubeconfig is fetched)
    Helm() HelmService
    Secrets() SecretAccessor            // just-in-time decryption, scoped to this operation's tenant
    Outputs() OutputStore               // pass non-secret outputs to dependent tasks (e.g. join endpoint)
    Spec() *ResolvedSpec
    Observed() ObservedStateReader
}
```

Rules for task authors (enforced by review, by tests, and where possible by linters):
1. **Idempotent.** Use "ensure" logic: check the current state, then act. For example, write a config file only if its checksum differs, and run `kubeadm init` only if `/etc/kubernetes/admin.conf` does not exist and the API is not reachable.
2. **No shell strings with data.** Remote commands are `Command{Program, Args[], Env, Stdin, Timeout}`. Arguments are typed values that pass validators. Config files are rendered from Go structs and uploaded via SFTP; they are never built with `echo … >`.
3. **Typed errors.** Return `errs.Transient(...)`, `errs.Permanent(...)`, `errs.UserAction(code, remediation...)`. Unknown errors are treated as permanent.
4. **Secrets** come only from `tc.Secrets()`. Never log them, put them in outputs or embed them in error text. The redactor also masks every value fetched through `Secrets()` in all log lines of the operation.
5. **Bounded.** Every remote call has a timeout. Polling loops use backoff and honour `ctx.Done()`.

### 5.3 Task state machine

```
PENDING ──▶ RUNNING ──▶ SUCCEEDED
   │           │  └──▶ RETRYING ──▶ RUNNING (attempt+1, after backoff)
   │           ├─────▶ FAILED ──(user Retry / Resume)──▶ PENDING
   │           └─────▶ CANCELLED (operation cancelled; stopped at a safe point)
   ├──▶ SKIPPED            (Check() == done, or dependency not needed in this plan)
   └──▶ CANCELLED          (operation cancelled before start)
SUCCEEDED ──(rollback)──▶ ROLLED_BACK   (only reversible tasks)
```

A task becomes *ready* when all of its `depends_on` are `SUCCEEDED` or `SKIPPED`. A `FAILED` dependency blocks dependents, which stay `PENDING`.

### 5.4 Operation and cluster state machines

Operation: `PENDING → RUNNING → SUCCEEDED | FAILED | CANCELLED`, with `RUNNING → PAUSED` (failure policy *pause*), `PAUSED → RUNNING` (resume/retry), `PAUSED|FAILED → ROLLING_BACK → ROLLED_BACK`, and `PAUSED → CANCELLED` (abort).

Cluster phase is derived from the active operation's progress and persisted for queries:

| Operation context | Cluster phase while running |
|---|---|
| create: validation/preflight | `VALIDATING → VALIDATED` |
| create: infra tasks | `PROVISIONING` |
| create: node prep + control plane | `BOOTSTRAPPING` |
| create: add-on installs | `INSTALLING` |
| create: post-install config (issuers, gateways, dashboards, GitOps bootstrap) | `CONFIGURING` |
| any: health verification | `HEALTH_CHECK` |
| success | `READY` |
| apply / upgrade / scale / add-on ops on a READY cluster | `UPDATING → READY` |
| failure with policy pause | `PAUSED` |
| unrecoverable failure / abort of create | `FAILED` |
| rollback in progress | `ROLLING_BACK` |
| destroy | `DESTROYING → DESTROYED` |

All transitions go through `domain.Cluster.Transition(to, reason)`, which checks an explicit allowed-transition table and bumps `version` (optimistic lock). Illegal transitions are programming errors: they are logged as ERROR and fail the operation. Ad-hoc `UPDATE … SET phase` statements are not allowed.

## 6. Retries, timeouts, circuit breakers

- **Retry policy per task** (defaults from task kind, overridable in catalog): `maxAttempts` (default 3; package installs 5; cloud API calls 8), exponential backoff `base=2s, factor=2, max=2m`, full jitter. Only `Transient` errors retry.
- **Error classification** happens in adapters, close to the source:
  - SSH: connection refused, timeout, `EOF` during handshake → transient; auth failure → `UserAction(SSH_AUTH_FAILED)`; host key mismatch → `UserAction(SSH_HOST_KEY_MISMATCH)` (never auto-accepted).
  - Package managers: lock held (`dpkg lock`) → transient; repository 404 → permanent with "catalog/mirror" remediation.
  - Kubernetes API: 429/5xx/connection reset → transient; 4xx validation → permanent.
  - Helm: timeout waiting for readiness → transient (bounded); chart render error → permanent.
  - Cloud providers: rate limits → transient honouring `Retry-After`; quota exceeded → `UserAction(PROVIDER_QUOTA)`.
- **Timeouts** at three levels: per remote command, per task (`Meta().Timeout`), per operation (soft limit: the operation pauses with "timed out" so a human can look).
- **Circuit breakers** wrap provider API clients and outbound webhooks. When a provider API fails repeatedly, new tasks fail fast with a clear "Provider API unavailable" message instead of each task retrying separately.

## 7. Failure handling, resume and rollback

**Default failure policy: `pause`.** On a non-retryable task failure (or exhausted retries):
1. Running sibling tasks finish or are cancelled at safe points. No new tasks start.
2. The outcome depends on the operation's failure policy:
   - `pause` (default): the operation becomes `PAUSED` and the cluster becomes `PAUSED`, waiting for a user decision;
   - `rollback`: the operation moves to `ROLLING_BACK` and reversible tasks are undone (see below), ending in `ROLLED_BACK`;
   - `abort`: the operation becomes `FAILED`. The cluster becomes `FAILED` if a `create` produced nothing usable yet; otherwise it returns to its previous phase (e.g. `READY`) with health reflecting the partial change.
3. A **failure report** is stored in `operations.error` and shown to the user:

```
Worker node node-04 failed to join the cluster.
Reason:   The node cannot reach the Kubernetes API server on port 6443.
Where:    task k8s.worker.join/node-04 · node 10.0.0.14 · attempt 3/3
How to fix:
  1. Check firewall rules between 10.0.0.14 and 10.0.0.10:6443.
  2. Check routing (traceroute from node-04 to the API endpoint).
  3. Verify the API server address (control-plane endpoint 10.0.0.10).
[Retry task] [Resume] [Roll back add-ons] [View logs] [Abort]
```

The text comes from the **error catalog**: a stable code (`K8S_JOIN_API_UNREACHABLE`), i18n keys for title, reason and remediation steps, and parameters (node, address, port) filled from typed error fields. The catalog is shared by API (`problem+json`), UI, CLI and notifications. A raw `exit status 1` is never shown. The raw stderr is attached as a log detail for experts.

**Resume** (also automatic after worker crash):
1. Re-acquire the lease. Reload the operation and its tasks.
2. Verify `plan_hash` against the current ResolvedSpec. If the user changed the spec since the plan was made, resume is refused with "spec changed, re-plan".
3. `SUCCEEDED`/`SKIPPED` tasks stay done. Tasks found `RUNNING` or `RETRYING` (crash mid-task or mid-backoff) are reset to `PENDING`; their attempt counters are kept. Because tasks are idempotent and `Check()` short-circuits, re-running them is safe. `FAILED` tasks are reset to `PENDING` only by an explicit user Retry/Resume.
4. Continue scheduling.

**Rollback** (user action, or automatic with policy `rollback`):
- The engine walks `SUCCEEDED` **reversible** tasks in reverse topological order and calls `Rollback`. For example, an add-on install rolls back to the previous Helm revision, or is uninstalled if it was newly installed.
- Irreversible tasks are never rolled back automatically. If any exist after the rollback boundary, the UI explains the remaining state and offers documented options: keep the partial cluster and fix it forward, or `destroy`.
- Example (prompt §13): `Install addon → Health check failed → Rollback addon`, implemented per add-on by `addon/<id>.install` (Reversible) + `addon/<id>.health` (failure triggers rollback of that add-on only, when the policy is `rollback`).

**Cancel**: `POST /operations/{id}/cancel` sets `cancel_requested`, sends `NOTIFY operation_cancel`, and the executor cancels task contexts. Running tasks stop at their next safe point and become `CANCELLED`, pending tasks become `CANCELLED`, and the operation ends in `CANCELLED`, a terminal state. Cancel never runs rollback implicitly. Because every task is idempotent, the user continues later by planning and starting a **new** operation (`apply`): completed work is detected by `Check()` and skipped, and partially applied steps converge. The UI offers "Rollback reversible steps" as a separate, explicit action.

## 8. Concurrency and consistency guarantees

| Concern | Mechanism |
|---|---|
| Two operations mutate the same cluster | Partial unique index `operations_one_active_per_cluster` (DB-enforced); API returns `409 OPERATION_IN_PROGRESS` with a link to the running operation |
| Duplicate submissions (double click, retrying client) | `Idempotency-Key` header → same operation returned; River unique jobs |
| Two workers execute the same operation | Execution lease with heartbeat and fencing (each write checks `lease_owner = me`) |
| Two tasks hit the same node's package manager | Per-node semaphore inside the executor + `flock` on the node for package operations |
| Lost updates on cluster rows | Optimistic locking (`version`), domain transition guards |
| Events visible in UI before commit | Events are written in the same transaction as the state change they describe; NOTIFY fires on commit |

## 9. Events, logs and real-time delivery

- Every task transition and log line → `operation_events` (see [data model](data-model.md#36-operations-the-prompts-deployment--deploymenttask)). It is structured: `ts, level, cluster_id, operation_id, task_id, node_id, type, message, fields`.
- Logs pass through the **redactor** before persistence. It masks values registered by `Secrets()`, it masks well-known patterns (private keys, bearer tokens, `password=`), and it drops fields on a denylist.
- **SSE** endpoint `GET /api/v1/operations/{id}/events` (`text/event-stream`):
  - On connect, the server replays events with `id > Last-Event-ID` (or from the beginning), then streams live events.
  - Live wake-up: the worker issues `NOTIFY operation_events, '<operation_id>:<event_id>'` after commit. Every API replica `LISTEN`s and pushes new rows to its subscribers. This scales horizontally without Redis.
  - Heartbeat comment every 15 s keeps proxies from closing the stream. The `retry:` hint is 3 s.
  - The stream ends with `event: end` when the operation reaches a terminal state.
- The UI rebuilds the full task tree from `GET /operations/{id}` + `/tasks` and then applies SSE deltas. Closing the browser and coming back shows "Deployment in progress" with the full history (prompt §66).
- ADR-0009 records why SSE over WebSocket. WebSocket is reserved for an interactive node terminal later.

## 10. Bootstrap layer (node preparation)

The bootstrap layer is a set of reusable, OS-aware task builders used by distributions:

```
Node ─▶ facts (OS, kernel, arch, CPU, RAM, disks, NICs, time sync, cgroup v2, SELinux/AppArmor)
     ─▶ os.prepare        (packages: conntrack, socat, ebtables/nftables, chrony; hostname; /etc/hosts entries)
     ─▶ kernel.modules     (overlay, br_netfilter; CNI-specific modules from CNIRequirements)
     ─▶ sysctl             (net.ipv4.ip_forward=1, bridge-nf-call-iptables=1, CNI extras) via /etc/sysctl.d/90-farvater.conf
     ─▶ swap.disable       (or configure NodeSwap if the spec enables it and the K8s version supports it)
     ─▶ time.sync          (chrony enabled and synced; skew check)
     ─▶ runtime.install    (containerd from catalog version; config rendered from struct; SystemdCgroup=true)
     ─▶ k8s.packages       (kubeadm/kubelet/kubectl from pkgs.k8s.io or offline mirror; held at catalog version)
     ─▶ distribution steps (init / join)
```

OS handling lives behind an `OSFamily` strategy (`debian` covers Ubuntu and Debian; `rhel` covers Rocky and Alma, after Phase 3). Each strategy maps abstract steps (`InstallPackages`, `HoldPackages`, `EnableService`, `AddRepository`) to typed commands. New OSes are added by adding a strategy, not by editing tasks.

## 11. Health engine

- `HealthProbe` interface: `Key()`, `Interval()`, `Run(ctx, ClusterAccess) ProbeResult{Status, Reason, Details, Remediation}`.
- Built-in probes: API server (`/readyz`), etcd (member health through the API server or etcdctl on control-plane nodes), scheduler, controller-manager, node conditions, CNI (agent DaemonSet ready + connectivity check pod), DNS (resolve `kubernetes.default` from a probe pod), Gateway (GatewayClass accepted, Gateway programmed), storage (default StorageClass + provisioner ready; optional PVC smoke test during `health.verify` only), metrics-server, monitoring stack, certificates (expiry of control-plane certs and cert-manager Certificates; WARNING < 30 days, CRITICAL < 7 days), backups (last successful within policy).
- During operations, `health.verify` runs all probes with strict thresholds. Periodically, a River periodic job per cluster (default every 60 s for READY clusters) updates `cluster_health_checks` and the cluster's aggregate health: worst-of with `UNKNOWN` handling. Transitions emit notifications and outbox events (incidents later in ee).

## 12. Extending the engine

- **New task kinds** are plain Go types implementing `Task`. They are registered by a plugin's planner contribution, and the engine core is not changed.
- **New operation types** need a planner function `(ResolvedSpec, ObservedState) → []PlannedTask`, an entry in the operation-type table (cluster phase mapping, required permission, audit action, default failure policy) and an API endpoint.
- **New plugins** (providers, distributions, add-ons) only implement interfaces from `pkg/sdk` (see [Core interfaces](core-interfaces.md)).

## 13. What the engine deliberately does not do

- It does not run arbitrary user scripts on nodes. An expert "advanced" escape hatch is limited to validated kubeadm, kubelet and containerd config patches and typed hooks.
- It does not hide irreversible actions behind automatic rollback.
- It does not keep state only in memory. If it isn't in PostgreSQL, it didn't happen.
- It does not talk to infrastructure from the API process. Only workers execute tasks, which keeps credentials out of the internet-facing tier.
