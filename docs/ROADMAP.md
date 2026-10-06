# Roadmap

> Status: **Proposed** (Phase 0) — awaiting owner approval. Language: English · [Русский](ROADMAP.ru.md)
>
> Related: [PRODUCT](PRODUCT.md) · [ARCHITECTURE](ARCHITECTURE.md) · [DECISIONS](DECISIONS.md)

The product spec contains two roadmaps: phases 0–10 in prompt §101–111 and an updated list of phases 1–14 in §238. This roadmap merges them: Phase 0 (research and architecture) comes from §101, and phases 1–14 follow the authoritative order of §238. The ordering rule is the spec's most important implementation requirement (§239): **enterprise features are not built before the core provisioning engine has proven it can reliably take infrastructure to a healthy cluster and recover from failures.**

Every phase follows the same loop (prompt §122): docs first → code → tests → lint → unit → integration → build → Docker → E2E where possible → fix → update docs → short report (format of prompt §123). Every phase ends with something runnable (`make dev` works, prompt §113) and with a **phase gate**: the exit criteria below, plus the [Definition of Done](#definition-of-done), plus a threat-model review.

## Overview

| Phase | Name | Theme | Edition | Depends on |
|---|---|---|---|---|
| 0 | Research & Architecture | Decide before building | — | — |
| 1 | Core / Foundation | Runnable platform skeleton with real persistence, auth and CI | Community | 0 |
| 2 | Provisioning engine | DAG, tasks, state machines, retries, resume, real-time | Community | 1 |
| 3 | Kubernetes on bare metal | SSH, node bootstrap, kubeadm, Cilium — first real cluster | Community | 2 |
| 4 | Production cluster | HA, Gateway, LB, storage, TLS, monitoring, logging, backup, Simple Mode | Community | 3 |
| **MVP** | = Phases 1–4 | One click from SSH hosts to a production-ready cluster | Community | |
| 5 | Auto Mode | Decision engine, presets, compatibility UI, cost, one-click | Community | 4 |
| 6 | Lifecycle | Upgrade, scale, backup/restore, node replacement, import, drift | Community | 4 |
| 7 | Multi-provider | Hetzner, AWS, GCP, Azure; k3s, RKE2 | Community | 4 (5 for Auto Mode on clouds) |
| 8 | GitOps | Argo CD, Flux, Git-backed specs | Community | 6 |
| 9 | Enterprise foundation | Orgs/teams admin, advanced RBAC, SSO, SCIM, MFA, service accounts | Enterprise | 4 |
| 10 | Enterprise governance | Policy engine, approvals, change requests, golden templates, compliance | Enterprise | 9 |
| 11 | Enterprise fleet | Cluster groups, fleet operations, canary, multi-region | Enterprise | 6, 10 |
| 12 | Enterprise security | Vault/OpenBao, KMS/BYOK, image security, supply chain | Enterprise | 9 |
| 13 | Enterprise DR & air-gap | Cross-region backup, DR, offline bundles | Enterprise | 6, 12 |
| 14 | Enterprise platform | HA platform, SLOs, cost, budgets, quotas, incidents, service catalog | Enterprise | 10, 11 |

Phases 5–8 can partly overlap once Phase 4 is stable. Enterprise phases start only after the MVP gate (§239).

---

## Phase 0 — Research & Architecture (current)

**Deliverables:** `docs/PRODUCT.md`, `docs/ARCHITECTURE.md`, `docs/ROADMAP.md`, `docs/DECISIONS.md`, ADRs in `docs/architecture/adr/`, architecture deep-dives (data model, provisioning engine, core interfaces, API, UI, Auto Mode/catalog, deployment model, testing, repository structure, technology stack with a verified version baseline), `docs/security/SECURITY_MODEL.md`, `docs/security/THREAT_MODEL.md`. All in English and Russian.

**Exit criteria:** owner approves the architecture (or requests changes); open questions are answered or explicitly deferred.

## Phase 1 — Core / Foundation

**Goal:** a runnable, production-shaped skeleton. Real persistence, auth, tenancy, API, UI shell, CI. Clusters can be created as records and "deployed" only with the **simulated provider** (explicitly test-only; prompt §102 allows a stub here, §115 forbids fake deployment in production, and the simulated provider satisfies both).

**Scope:**
- Monorepo layout, Go module, Makefile, golangci-lint, pnpm workspace, pre-commit (gofumpt, lint, gitleaks).
- API v1 skeleton with Huma (committed OpenAPI 3.1, oasdiff/vacuum gates, generated TS types, typed Go client); problem+json errors; error catalog (en/ru).
- PostgreSQL 18 + goose migrations (identity, tenancy, RBAC, credentials/secrets, clusters, spec revisions, operations, events, audit, idempotency, outbox); RLS from the first migration.
- AuthN: local users (argon2id), sessions, CSRF, API keys. AuthZ: permission catalogue, built-in roles, bindings, `Authorizer`.
- Organizations and projects (single default org in UI, multi-tenant underneath), clusters CRUD with ClusterSpec v1alpha1 (strict YAML/JSON, JSON Schema), spec revisions and diff.
- Credentials vault with envelope encryption (local KEK), write-only API, redaction, canary leak tests.
- River integration, transactional outbox, audit log (hash chain), webhooks (Standard Webhooks) with SSRF guard.
- Simulated provider + minimal engine path (linear task list) so a "deployment" round-trip works end to end; real engine in Phase 2.
- Web: app shell, routing, theme (light/dark), i18n (en/ru), login, clusters list/detail, YAML editor (Monaco), credentials, settings; generated API client.
- CLI skeleton: `login`, `cluster create|list|get|delete`, `validate`.
- Docker: multi-stage images, `make dev` (compose: postgres, api, worker, web), `.env.example`.
- CI: lint → unit → integration → build → security scan (govulncheck, OSV/Grype, gitleaks, image scan, SBOM) → E2E (Playwright on simulated) → docker build.
- Docs: README, DEVELOPMENT.md, DEPLOYMENT.md (dev/self-hosted), CONTRIBUTING.md, SECURITY.md, API.md, and the guides for adding a provider, distribution, add-on, CNI, storage provider, ingress/gateway and preset (prompt §75).

**Exit criteria:** In the UI, a user logs in, creates a cluster from a preset/YAML, deploys it with the simulated provider, sees progress, and the record ends READY. Audit entries exist. Authz-matrix and tenant-escape tests are green. CI is green. `make dev` works from a clean checkout.

## Phase 2 — Provisioning engine

**Scope:** planner (contributions, capability graph, topological sort, diff, irreversible flags, estimates), DAG scheduler with concurrency limits, executor with leases/heartbeats/fencing, task contract (`Check/Run/Rollback`), typed errors and retry policies, circuit breakers, failure policies (pause/rollback/abort), resume, retry task, cancel, rollback of reversible tasks, cluster/operation/task state machines, operation events with SSE (`Last-Event-ID`) and LISTEN/NOTIFY fan-out, structured log search and download, failure report UI (what/why/where/how to fix), plan/dry-run endpoint and UI, idempotency keys end to end.

**Exit criteria:** On the simulated provider with fault scripts: every failure scenario of the [testing strategy §5](architecture/testing-strategy.md#5-failure-testing-matrix-prompt-63) gives the correct report and recovers. The kill-the-worker test passes 100/100 iterations with identical final state. The browser can close and reopen mid-deployment and recover.

## Phase 3 — Kubernetes on bare metal (first real cluster)

**Scope:** SSH subsystem (keys, password, agent, jump hosts, proxy, host-key verification and TOFU confirmation UI, pooling, timeouts), bare-metal provider (declared hosts, discovery/facts), preflight checks (OS/CPU/RAM/disk/kernel/cgroup v2/swap/modules/sysctl/ports/DNS/latency/time sync) with ✓/⚠/✗ UI, bootstrap layer with `OSFamily` debian (Ubuntu 24.04, 26.04; Debian 13; Debian 12 best-effort), signed node helper, containerd (catalog-pinned), kubeadm (v1beta4) single control plane + HA (3 CP, stacked etcd, kube-vip static pod with the super-admin.conf bootstrap step), join, reset, kubeconfig (user-scoped, short-lived), Gateway API CRDs (platform-owned), Cilium (default CNI), basic health engine, cluster delete (node reset).

**Exit criteria:** Tier-2 E2E (container nodes) creates a single-CP and a 3-CP cluster over real SSH. `kubectl get nodes` and `get pods -A` are healthy. Re-running the operation is a no-op. Failure tests marked ★ pass on Tier 2. Tier-3 (real VMs) is run if an account is available; otherwise the report states this gap explicitly.

## Phase 4 — Production cluster

**Scope:** Gateway controller (Cilium Gateway by default; Envoy Gateway alternative), MetalLB (L2/BGP) and Cilium LB-IPAM option, storage (local-path, NFS CSI, Longhorn), cert-manager (+ issuers: Let's Encrypt HTTP-01/DNS-01, internal CA, self-signed, existing), metrics-server, observability presets (Basic / Standard / Full: kube-prometheus-stack, Grafana with dashboards and credentials hand-off, Loki + Alloy, optional Tempo/OpenTelemetry), etcd snapshot backups to S3-compatible storage, health engine (all probes), Calico and Flannel after Cilium is stable, Simple Mode (presets), templates (save/reuse/versions), advanced wizard (15 steps) with dependency-aware forms, cluster dashboard, nodes view (cordon/drain/uncordon), add-on management (install/remove), notifications (in-app, email, webhook, Slack, Telegram), CLI deploy/watch/kubeconfig, global search, command palette.

**Exit criteria (= MVP gate):** The main scenario of prompt §120 (with bare metal instead of Hetzner) runs end to end from the UI and from the CLI: Production HA, Cilium, Gateway, MetalLB, Longhorn, Prometheus + Grafana, Loki, cert-manager, etcd backup to S3-compatible storage (SeaweedFS or Ceph RGW in tests), then READY, with Grafana URL and kubeconfig. All failure tests pass. A security review of the MVP is done. Docs are complete in en/ru.

## Phase 5 — Auto Mode

**Scope:** decision engine (rules, reasons, alternatives, pins, safety warnings), "Why this configuration?" UI, overrides with recomputation, presets as engine inputs, compatibility engine surfaced in the UI (hide/disable with reasons), cost estimation where providers price (otherwise "unavailable"), time estimates from history, one-click "Deploy" from the 5-question form (prompt §234), golden-file and property tests.

**Exit criteria:** Production + bare metal + Medium + HA → Deploy works with no further input except credentials and hosts. Every recommendation passes compatibility with zero BLOCK findings (property-tested).

## Phase 6 — Lifecycle

**Scope:** upgrade advisor (readiness score, blockers, deprecated API scan, etcd guard) and orchestrated Kubernetes upgrades (CP node by node with etcd snapshots, then add-ons, then workers in batches with drain), scaling (add/remove workers, pools), backup/restore (Velero with CSI snapshots/data mover + etcd), node replacement, certificate and credential rotation, import of existing clusters by kubeadm/kubeconfig (detect CNI, gateway/ingress including ingress-nginx with migration assistant, storage, add-ons), drift detection (desired vs actual) with alert-only remediation.

**Exit criteria:** On Tier 2 and Tier 3, 1.36→1.37 upgrade E2E passes. Scale 3→6 workers passes. Backup → destroy → restore restores workloads and PVs. Import of a kind/k3d cluster shows correct inventory.

## Phase 7 — Multi-provider

**Scope:** Hetzner Cloud provider (servers, networks, firewalls, placement groups, LB for API endpoint and Services, hcloud CCM + CSI, pricing), then AWS, GCP and Azure, one at a time. Native SDKs or a Cluster API backend, decided per ADR-0013 at phase start. Distributions k3s and RKE2. OS family rhel (Rocky/Alma 9/10, SELinux enforcing).

**Exit criteria:** Each provider passes the provider contract suite and the mandatory E2E on its real API (nightly) with cost caps. Auto Mode uses provider capabilities and pricing.

## Phase 8 — GitOps

**Scope:** Argo CD and Flux add-ons with automatic cluster registration (prompt §27). Git-backed cluster specs ("GitOps-managed cluster": UI changes create commits/PRs or are blocked, prompt §200). Drift management with Git as source of truth. Git credentials in the vault.

**Exit criteria:** M8 demo passes; UI edits of a GitOps-managed cluster create a commit/PR (or are blocked by policy); drift is detected and reported.

## Phase 9 — Enterprise foundation (ee)

**Scope:** `ee/` directory with commercial license and build tag; EntitlementService with signed licenses; multi-org administration; teams; custom roles; ABAC conditions; SSO (OIDC, SAML 2.0) through the IdentityProvider abstraction (Okta, Entra ID, Google Workspace, Keycloak, Auth0, generic); SCIM 2.0 provisioning/deprovisioning with session and API-key revocation; MFA (TOTP, WebAuthn) and MFA policy; service accounts; session management (device list, revoke all, suspicious session detection); enterprise audit search and export.

**Exit criteria:** SSO login via Keycloak (OIDC) and a SAML IdP in E2E; SCIM deprovisioning revokes sessions and API keys within one minute; custom-role and ABAC authorization-matrix tests green; Community build contains no `ee/` code.

## Phase 10 — Enterprise governance (ee)

**Scope:** policy engine (policy as code, levels INFO/WARNING/BLOCK, scopes and inheritance global → org → project → environment → cluster with inherit/override/restrict, OPA/Rego or Cedar adapter), approval policies, change requests (diff, risk, policy results, approvers), deployment approval, maintenance windows, golden templates (locked fields, versions, inheritance), compliance center (frameworks as control mappings, evidence, never claiming compliance), security posture per cluster (CIS via kube-bench/Kubescape).

**Exit criteria:** a BLOCK policy stops a non-compliant production deploy with an explanation; a 2-approver change request gates an upgrade end to end; golden templates enforce locked fields; compliance center shows controls with evidence.

## Phase 11 — Enterprise fleet (ee)

**Scope:** cluster groups; fleet operations (upgrade 100+ clusters in batches with canary, health gates, automatic stop on failure rate); multi-region and topology-aware placement (regions, zones, racks, failure domains); global dashboard (health, deployments, backups, security, compliance); node lifecycle policies (replace nodes older than N days).

**Exit criteria:** a canary fleet upgrade of ≥ 20 simulated + ≥ 3 real clusters runs in batches and stops automatically when the failure threshold is reached.

## Phase 12 — Enterprise security (ee)

**Scope:** OpenBao/Vault (Transit KEK + KV backend), AWS/GCP/Azure KMS, BYOK, key rotation tooling; image security (registry allow/block, signature verification via Sigstore/Kyverno, private default registry, Harbor integration); vulnerability management (Trivy Operator data aggregation); supply-chain verification of catalog artifacts; signed releases with SBOM and provenance (already in CI since Phase 1, now enforced on install/import).

**Exit criteria:** KEK in an external KMS/OpenBao with rotation tested; unsigned or disallowed images blocked by policy in E2E; release artifacts verifiable with documented commands.

## Phase 13 — Enterprise DR & air-gap (ee)

**Scope:** cross-region backup and replication, DR workflows (failover/restore runbooks executed as operations), RPO/RTO tracking and display, backup policies by environment, platform backup/restore; air-gapped installation: `farvater bundle create|verify|import`, private OCI registry, offline Helm charts, offline Kubernetes and OS packages/binaries.

**Exit criteria:** a full cluster is created in a network-isolated CI environment from an imported bundle; backup → restore into another region meets the documented RPO/RTO.

## Phase 14 — Enterprise platform (ee)

**Scope:** HA reference architecture for the platform (api×3, worker×3, PostgreSQL HA), SLO dashboards (API availability, deployment success, backup success), cost management (allocation by org/project/team/environment/cost center), budgets with thresholds and policy actions, quotas (clusters, nodes, CPU, RAM, storage, deployments per scope), incident management (auto-incidents, assignment, timeline, postmortems; PagerDuty/Opsgenie/Slack/Teams), service catalog (templates, add-on bundles, security and observability profiles), self-service within policy, Terraform/OpenTofu provider (over the public API), enterprise integrations (Jira, ServiceNow, SIEMs).

**Exit criteria:** HA platform survives the loss of one zone without failed operations; SLO dashboards and alerts shipped; budgets and quotas enforced in E2E; incidents created automatically for unavailable clusters.

---

## Milestones and demos

| Milestone | Demo |
|---|---|
| M1 (end of Phase 1) | Login → create cluster from YAML → simulated deploy → READY; audit trail; CI badge green |
| M2 (Phase 2) | Simulated deployment with injected failures: pause → explanation → fix → resume; kill worker → automatic continuation |
| M3 (Phase 3) | Real kubeadm HA cluster over SSH on container nodes (and VMs if available); kubeconfig works |
| M4 = MVP (Phase 4) | Main scenario (§120) on bare metal: production-ready cluster with add-ons, from UI and CLI |
| M5 (Phase 5) | 5-question Auto Mode → Deploy, with "Why?" explanations |
| M6 (Phase 6) | Upgrade 1.36 → 1.37, scale, backup/restore, import |
| M7 (Phase 7) | Hetzner Cloud one-click cluster with cost estimate |
| M8 (Phase 8) | GitOps-managed cluster: a spec change merged in Git is planned, validated and applied; drift is detected |
| M9+ | Enterprise milestones per phase (see exit criteria) |

## Definition of Done

A feature is done only if (prompt §125, and §237 for enterprise):
- it is implemented and its integration points work;
- errors are handled and user-facing errors are explainable (what/why/where/how to fix);
- tests exist at the right levels (see [testing strategy](architecture/testing-strategy.md)); lint, type checks and build pass; the main happy path works;
- docs are updated in English and Russian;
- there are no obvious security issues, no hard-coded secrets, no unexplained TODOs;
- for enterprise features also: API + UI + permissions + audit + migration if needed + observability + failure handling + E2E + CLI/API support where applicable + tenant isolation verified + Community ("Standard" in the spec) edition not broken.

## Top risks to the plan

See [ARCHITECTURE §Risks](ARCHITECTURE.md#13-biggest-risks) for the full list with mitigations. The ones that most affect the schedule:
1. Scope is very large compared with team capacity. Mitigations: strict phase gates and MVP focus.
2. No real hardware for testing. Mitigations: Tier-2 container nodes from Phase 3, and a small cloud budget for Tier 3 before calling the MVP done.
3. Ecosystem churn (three Kubernetes minors per year; Gateway API, containerd, etcd changes). Mitigation: a data-driven, signed catalog with automated update PRs.
