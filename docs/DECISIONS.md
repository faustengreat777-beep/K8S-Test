# Decisions

> Status: **Proposed** (Phase 0) — awaiting owner approval. Language: English · [Русский](DECISIONS.ru.md)
>
> Index of Architecture Decision Records (ADRs). Format and process: [ADR-0001](architecture/adr/0001-record-architecture-decisions.md). ADRs marked *Proposed* become *Accepted* when the owner approves the Phase 0 architecture.

## Owner decisions taken in Phase 0 (2026-10-06)

| # | Question | Answer |
|---|---|---|
| 1 | Scope of this session | Phase 0 only; stop and wait for approval before Phase 1 |
| 2 | Product name | Architect chooses an original name not used on the market (see ADR-0002) |
| 3 | Licensing | Open-core: Apache-2.0 core + commercial `ee/` |
| 4 | Test hardware | The owner has no own machines → simulated provider, container nodes, cloud VMs later (ADR-0016) |
| 5 | Default ingress | Gateway API (ADR-0020) |
| 6 | Documentation language | English and Russian side by side (ADR-0026) |
| 7 | Git workflow | Push to the working branch; no pull request |

## ADR index

| ADR | Title | Status | Summary |
|---|---|---|---|
| [0001](architecture/adr/0001-record-architecture-decisions.md) | Record architecture decisions | Accepted | Lightweight MADR in `docs/architecture/adr/`, bilingual, immutable once accepted |
| [0002](architecture/adr/0002-product-name-and-open-core-licensing.md) | Product name, open-core licensing and editions | Proposed | Product name; Apache-2.0 core + commercial `ee/`; entitlements in one service; enterprise never cripples core |
| [0003](architecture/adr/0003-modular-monolith.md) | Modular monolith with one server binary and process roles | Proposed | `api` / `worker` / `all` / `migrate` roles of one Go binary; only workers hold credentials |
| [0004](architecture/adr/0004-repository-layout.md) | Go-idiomatic monorepo layout with a single Go module | Proposed | `cmd/`, `internal/`, `pkg/`, `plugins/`, `catalog/`, `web/`, `ee/`; enforced import rules |
| [0005](architecture/adr/0005-api-style-and-contract.md) | REST/JSON API with a committed OpenAPI 3.1 contract (Huma) | Proposed | Typed Go operations generate OpenAPI 3.1; committed spec with oasdiff gate; 202 + Operation for async; RFC 9457 errors |
| [0006](architecture/adr/0006-persistence-postgresql.md) | PostgreSQL as the only required stateful dependency | Proposed | PG 18 (min 16), pgx, sqlc, goose; no ORM; no Bitnami |
| [0007](architecture/adr/0007-job-queue-river-no-redis.md) | River job queue; PostgreSQL for pub/sub and locks; no Redis | Proposed | Transactional enqueue; LISTEN/NOTIFY; advisory locks; optional Valkey, never Redis |
| [0008](architecture/adr/0008-provisioning-engine-execution-model.md) | Own DAG provisioning engine; one leased job per operation | Proposed | Planner → DAG → executor with leases, retries, resume, safe rollback; Temporal not adopted for now |
| [0009](architecture/adr/0009-real-time-updates-sse.md) | Server-Sent Events backed by a replayable event log | Proposed | SSE + `Last-Event-ID` replay from PostgreSQL; WebSocket only for a future terminal |
| [0010](architecture/adr/0010-plugin-model-and-helm.md) | Compile-time plugins; declarative add-ons via embedded Helm 4 SDK | Proposed | `pkg/sdk` interfaces; add-ons as data; Helm v4 SDK; digest-pinned OCI charts; no Bitnami |
| [0011](architecture/adr/0011-multi-tenancy-and-rls.md) | Multi-tenancy with PostgreSQL RLS as defence in depth | Proposed | Org/project scoping, app-layer authz, `FORCE ROW LEVEL SECURITY`, 404 for foreign ids |
| [0012](architecture/adr/0012-frontend-stack.md) | Frontend stack | Proposed | React 19 + TypeScript 7 + Vite 8 + Tailwind 4 + shadcn/ui (Base UI) + TanStack + RHF/Zod 4 + Monaco (CodeMirror fallback) + Lingui; Oxlint |
| [0013](architecture/adr/0013-infrastructure-tooling-boundaries.md) | Infrastructure tooling boundaries; Cluster API deferred | Proposed | Native Go providers; no Terraform/Ansible in core; Cluster API evaluated for clouds in Phase 7 |
| [0014](architecture/adr/0014-secrets-envelope-encryption.md) | Secrets: envelope encryption with pluggable key providers | Proposed | Tink AEAD keysets per org wrapped by a KEK (local / KMS / OpenBao); only workers decrypt |
| [0015](architecture/adr/0015-remote-execution-ssh.md) | Agentless SSH with typed commands and an ephemeral signed node helper | Proposed | No shell interpolation; SFTP config files; mandatory host-key verification |
| [0016](architecture/adr/0016-testing-without-own-hardware.md) | Testing without own hardware | Proposed | Simulated provider, container nodes over real SSH, real-VM tier when budget exists |
| [0017](architecture/adr/0017-data-driven-version-catalog.md) | Data-driven, signed version catalog | Proposed | All versions/compatibility as signed data, updatable without releases, digests for air-gap |
| [0018](architecture/adr/0018-explainable-decision-engine.md) | Rule-based, deterministic, explainable decision engine | Proposed | Ordered rules emit decisions with reasons; overrides pin and recompute; golden tests |
| [0019](architecture/adr/0019-packaging-and-supply-chain.md) | Packaging, distribution and supply-chain security | Proposed | Static binary + distroless images + Helm chart + compose; signed releases, SBOM, provenance |
| [0020](architecture/adr/0020-networking-defaults-gateway-api.md) | Networking defaults: Cilium; Gateway API only; no ingress-nginx | Proposed | Platform-owned Gateway API CRDs; Cilium Gateway or Envoy Gateway |
| [0021](architecture/adr/0021-kubernetes-version-and-runtime-defaults.md) | Kubernetes version policy and node runtime defaults | Proposed | 1.35–1.37 supported, 1.36 default; kubeadm v1beta4; containerd 2.3 LTS; cgroup v2; nftables |
| [0022](architecture/adr/0022-load-balancing-and-control-plane-endpoint.md) | Load balancing and control-plane endpoint | Proposed | kube-vip / external LB / provider LB; MetalLB or Cilium LB-IPAM; L2 blocked on L3 clouds |
| [0023](architecture/adr/0023-storage-defaults.md) | Storage defaults | Proposed | local-path (dev), Longhorn (prod bare metal), Rook-Ceph (large/IO), cloud CSI; MinIO not bundled |
| [0024](architecture/adr/0024-observability-stack-and-licensing.md) | Observability defaults and AGPL handling | Proposed | Basic/Standard/Full presets; AGPL components installed unmodified; Apache-2.0 alternatives |
| [0025](architecture/adr/0025-authentication-sessions-and-api-keys.md) | Authentication: sessions, API keys, SSO in Enterprise | Proposed | Server-side sessions (no JWT), hashed scoped API keys, argon2id; OIDC/SAML/SCIM/MFA in ee |
| [0026](architecture/adr/0026-bilingual-documentation.md) | Bilingual documentation | Accepted | `X.md` + `X.ru.md`, English source, same-PR updates, CI structure check |
| [0027](architecture/adr/0027-operation-instead-of-deployment.md) | Model "Deployment" as Operation / OperationTask | Proposed | Unambiguous naming in code/API; UI copy and webhook names keep "deployment" |

## Deviations from the product spec (and why)

| Spec item | Decision | Reason | ADR |
|---|---|---|---|
| Kubernetes 1.34 in examples | Default 1.36, latest 1.37; 1.34 not offered | 1.34 reaches EOL 2026-10-27; add-on test matrices end at 1.36 | 0021 |
| NGINX Ingress as default (§19, §120) | Gateway API only; ingress-nginx never offered | ingress-nginx retired and archived (2026-03) | 0020 |
| Hetzner + MetalLB in the main scenario (§120) | Hetzner LB via hcloud CCM on Hetzner Cloud; MetalLB for L2 bare metal | MetalLB L2/ARP doesn't work on Hetzner Cloud's L3 networks | 0022 |
| Redis + job queue (§43, §84 `REDIS_URL`) | River on PostgreSQL; no Redis; optional Valkey | Transactional enqueue, fewer components, licensing | 0007 |
| `JWT_SECRET` (§84) | Server-side sessions; no JWT for first-party auth | Immediate revocation, session listing | 0025 |
| `clusterctl` / `platformctl` CLI names (§56, §197) | Product-named CLI | `clusterctl` is Cluster API's CLI | 0002 |
| "Deployment"/"DeploymentTask" models (§58) | `Operation`/`OperationTask` | Covers all lifecycle actions; avoids clash with Kubernetes `Deployment` | 0027 |
| `apps/` + `packages/` layout (§76) | Go-idiomatic `cmd/internal/pkg/plugins` | Compiler-enforced encapsulation in Go; explicitly allowed by §76 | 0004 |
| Terraform/Ansible usage (§55) | Not in core; native Go providers; our own Terraform provider for the API later | One execution model with typed errors and progress | 0013 |
| MinIO as backup target (§28) | Supported as an *external* S3 endpoint; not bundled | MinIO community edition is archived | 0023 |
| Prometheus/Grafana/Loki/Tempo (§22) | Kept, with license boundaries and Apache-2.0 alternatives | Grafana/Loki/Tempo are AGPL | 0024 |
| Phase 1 "mock/stub deployment" vs §115 "no fake deployment" | `simulated` provider, test-only, refused in production profile | Satisfies both requirements | 0016 |
