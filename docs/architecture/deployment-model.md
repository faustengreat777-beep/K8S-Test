# Deployment model of the platform

> Status: **Proposed** (Phase 0). Language: English · [Русский](deployment-model.ru.md)
>
> Related: [ARCHITECTURE](../ARCHITECTURE.md) · [Security model](../security/SECURITY_MODEL.md) · ADR-0003 (modular monolith), ADR-0019 (packaging and distribution)

This document explains how **Farvater itself** is built, packaged, installed, configured, upgraded and operated: in development, in small self-hosted installations, on Kubernetes, in highly available enterprise setups and in air-gapped environments. For how the platform deploys *customer* clusters, see the [provisioning engine](provisioning-engine.md).

## 1. Artifacts

| Artifact | Contents | Platforms |
|---|---|---|
| `farvater-server` binary | API + worker + migrations + embedded web UI + embedded default catalog | linux/amd64, linux/arm64 |
| `farvater` binary | CLI (API client), offline bundle tooling (ee) | linux, macOS, windows × amd64, arm64 |
| OCI image `…/farvater-server:<version>` | static binary on distroless/static, non-root (UID 65532), read-only rootfs | amd64, arm64 (multi-arch manifest) |
| Helm chart `farvater` | Installs the platform on Kubernetes | — |
| Docker compose bundle | `compose.yaml` + `.env.example` for single-host installs | — |
| Node helper binary (Phase 3) | Small static Go binary uploaded to nodes during operations (signed) | linux/amd64, linux/arm64 |
| Catalog bundle | Signed catalog data (can update independently) | — |
| Release metadata | checksums, cosign signatures, SBOM, SLSA provenance | — |

Images never use `latest`. Tags are immutable semver plus digests (prompt §79). Dockerfiles are multi-stage: a builder with a pinned Go toolchain and Node for the web build, then a final `gcr.io/distroless/static`-class image. Build flags are `-trimpath`, `CGO_ENABLED=0` and `-ldflags` for version metadata.

## 2. Process roles

One binary, several roles (see [ARCHITECTURE](../ARCHITECTURE.md)):

| Command | Role | Scaling | Needs |
|---|---|---|---|
| `farvater-server api` | HTTP API, SSE, web UI | Stateless, N replicas behind a load balancer | PostgreSQL |
| `farvater-server worker` | River workers: operations, health polling, webhooks, notifications, retention | N replicas; leases make it safe | PostgreSQL, KEK (local key file or KMS), egress to customer infrastructure |
| `farvater-server all` | api + worker in one process | 1 (small installs, dev) | both |
| `farvater-server migrate` | Apply DB migrations (advisory lock) then exit | Run once per upgrade (init container / Job / compose one-shot) | PostgreSQL owner role |
| `farvater-server admin …` | Bootstrap first org/admin, rotate keys, verify audit chain, export | Manual | PostgreSQL |

Health endpoints: `/healthz` (liveness), `/readyz` (readiness: DB reachable, schema version matches, for workers also River client healthy), `/metrics` on a separate port.

## 3. Configuration

12-factor configuration through environment variables or files (`*_FILE` variants for secrets), validated at start-up:

| Variable | Purpose |
|---|---|
| `DATABASE_URL` (or `DATABASE_URL_FILE`) | PostgreSQL DSN (TLS `verify-full` required in production profile) |
| `ENCRYPTION_KEY_FILE` / `ENCRYPTION_KEY` | Local KEK (256-bit, base64) — required for workers; never needed by `api` (api has no decrypt path) |
| `KMS_PROVIDER`, `KMS_*` (ee) | External KEK (OpenBao/Vault Transit, AWS/GCP/Azure KMS) |
| `SESSION_SECRET_FILE` | Session/CSRF HMAC key |
| `API_KEY_PEPPER_FILE` | Pepper for API key hashing |
| `PUBLIC_URL` | External URL (cookies, links in notifications, OIDC redirects) |
| `LISTEN_ADDR`, `METRICS_ADDR` | Listeners |
| `TLS_CERT_FILE`, `TLS_KEY_FILE` | Optional built-in TLS |
| `PROFILE` | `dev` \| `production` — production refuses insecure settings and the simulated provider |
| `LOG_LEVEL`, `LOG_FORMAT` | `slog` settings |
| `OTEL_EXPORTER_OTLP_ENDPOINT` etc. | OpenTelemetry (standard env vars) |
| `CATALOG_CHANNEL`, `CATALOG_BUNDLE_URL`, `CATALOG_PUBLIC_KEYS` | Catalog updates (disabled in air-gap) |
| `EGRESS_ALLOW_PRIVATE_CIDRS` | SSRF allow-list for self-hosted environments that need private targets |
| `SMTP_*` | Email notifications |
| `VALKEY_URL` (optional) | Shared rate limiting/caching for large HA installs |

The prompt's `REDIS_URL` and `JWT_SECRET` are **not required**: PostgreSQL covers queueing, pub/sub and locks, and sessions are server-side, so there are no JWTs. ADR-0007 and ADR-0025 explain why. `.env.example` documents every variable (prompt §84).

## 4. Installation targets

### 4.1 Development (`make dev`, prompt §77)

```
docker compose -f deploy/compose/compose.dev.yaml up
  ├── postgres        (PostgreSQL 18, seeded dev DB, port 5432 on localhost)
  ├── migrate         (one-shot)
  ├── api             (air hot-reload of Go code, PROFILE=dev, simulated provider enabled)
  ├── worker
  └── web             (Vite dev server with HMR, proxy /api → api)
optional profiles: valkey · keycloak (SSO testing, Phase 9) · mailpit (email) · container-nodes (Tier-2 test nodes)
```

One command, no host dependencies except Docker and `make`. A dev admin user and org are created by a seed command and printed once. The seed password is random, never hard-coded (prompt §115).

### 4.2 Small self-hosted (single host)

- Docker compose (`deploy/compose/compose.yaml`): `postgres` + `migrate` + `farvater-server all` + optional reverse proxy (Caddy) for TLS.
- Or a systemd unit running the binary with an external PostgreSQL.
- Suitable for teams managing up to tens of clusters. Back up the DB and the KEK separately.

### 4.3 Kubernetes (Helm chart)

```
Deployment  api      (replicas ≥ 2, HPA on CPU/RPS, PDB, topology spread)
Deployment  worker   (replicas ≥ 2, PDB; optional dedicated node pool with egress to customer networks)
Job         migrate  (pre-install/pre-upgrade hook)
Service + Gateway/HTTPRoute (or Ingress) for api
Secret refs: DB credentials, KEK (or KMS config), session secret, pepper
NetworkPolicies: api ingress only from gateway; worker no ingress; egress rules configurable
ServiceMonitor (optional), PrometheusRule (optional)
PostgreSQL: external (managed DB) or CloudNativePG cluster (recommended operator for self-managed HA)
```

Pods run with `runAsNonRoot`, `readOnlyRootFilesystem`, drop all capabilities, `seccompProfile: RuntimeDefault`, and resource requests/limits.

### 4.4 Enterprise high availability (prompt §182)

- api × 3 and worker × 3 across ≥ 2 (better 3) zones; PostgreSQL HA (CloudNativePG with synchronous replica, Patroni, or managed service with multi-AZ); load balancer; object storage for platform backups and offline bundles; KMS for the KEK.
- No single point of failure in the platform. Worker leases and River make losing a worker safe. API replicas are stateless. SSE clients reconnect to any replica, which replays from the DB.
- Zero-downtime upgrades: expand/contract migrations, rolling api/worker updates, workers drain at safe points (`SIGTERM` → stop taking new tasks → finish or checkpoint current tasks within the grace period).
- SLO readiness (prompt §222): documented failure domains, monitoring and alerting rules shipped with the chart, DR runbooks. **No SLA is promised** without matching infrastructure.

### 4.5 Deployment models (prompt §220)

| Model | Description |
|---|---|
| Self-hosted | Customer runs the platform in their infrastructure (compose, VM, Kubernetes) — the first and main model |
| Dedicated | Vendor-operated single-tenant instance |
| SaaS | Vendor-operated multi-tenant "Platform Cloud" (multi-tenancy architecture makes it possible; workers for customer infrastructure need connectivity: outbound-only agent/relay is a future design topic) |
| Air-gapped | No Internet; offline bundles (section 6) |

## 5. Upgrades of the platform

1. Read release notes and verify signatures and checksums (`cosign verify`).
2. `migrate` job runs (idempotent, advisory-locked). Migrations are backward compatible with the previous release (expand phase).
3. Rolling update of workers, then api. Running operations continue: a worker hands over at task boundaries thanks to leases. Long tasks continue on the old worker until the grace period ends, then resume elsewhere.
4. Post-upgrade checks: `/readyz`, smoke test, catalog verification.
5. Rollback: previous binaries work with the expanded schema. Contract migrations run only in a later release.

Release channels: stable, LTS (enterprise, longer support), beta, nightly (prompt §227).

## 6. Air-gapped installation (ee, Phase 13, prompt §179–180)

```
# On a connected machine
farvater bundle create --catalog-version 2026.10.1 --kubernetes 1.36,1.37 --addons default,longhorn,observability \
                      --os ubuntu-24.04,ubuntu-26.04,debian-13 --output bundle.tar
# → OCI layout: images (by digest), Helm charts (OCI), OS packages / static binaries (containerd, runc, CNI plugins, kubeadm/kubelet/kubectl),
#   node helper, catalog subset, checksums, signatures, SBOMs, manifest

# In the air-gapped environment
farvater bundle verify bundle.tar
farvater bundle import bundle.tar --registry registry.internal:5000 --packages https://mirror.internal/farvater
```

- The catalog is rewritten to point at internal mirrors (registry, package/binary mirror, chart OCI repo).
- Nodes get binaries from the internal mirror (or copied over SFTP by the node helper), not from pkgs.k8s.io.
- The import is a privileged, audited action. Signatures are verified against configured public keys.

## 7. Operating the platform

- **Observability of the platform itself** (prompt §60, §231): Prometheus metrics (`farvater_clusters_total`, `…_operations_total{type,status}`, `…_operation_duration_seconds`, `…_tasks_failed_total{kind,code}`, `…_nodes_total`, `…_api_requests_total`, `…_api_request_duration_seconds`, `…_job_queue_depth`, `…_backup_success_total`, `…_backup_failure_total`, `…_webhook_deliveries_total{status}`); OpenTelemetry traces across API → job → tasks → SSH/K8s calls; structured logs with trace ids; dashboards and alert rules shipped with the chart.
- **Backups** (prompt §183): PostgreSQL (logical or physical with PITR), configuration, KEK (separately, offline), catalog state. Audit logs are exported to object storage before partition drop.
- **Runbooks** (Phase 1+): restore from backup, KEK rotation, DEK rotation, revoke all sessions, emergency read-only mode, worker isolation, catalog rollback.
