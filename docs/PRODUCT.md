# Product

> Status: **Proposed** (Phase 0) — awaiting owner approval. Language: English · [Русский](PRODUCT.ru.md)
>
> Related: [ARCHITECTURE](ARCHITECTURE.md) · [ROADMAP](ROADMAP.md) · [DECISIONS](DECISIONS.md)

## 1. Vision

> **"From bare infrastructure to a production-ready Kubernetes cluster with minimum user interaction."**

**Farvater** is one platform, one configuration model and one deployment engine. It deploys a fully prepared Kubernetes cluster almost with one click, and then keeps managing it.

Ideal result for a single user: *"I choose Production + my servers (or Hetzner) + Medium → press Deploy → some time later I get a fully ready, secure, observable, backed-up and manageable Kubernetes cluster."*

Ideal result for a company (Enterprise): *"We centrally manage hundreds of Kubernetes clusters, policies, access, security, compliance evidence, backups, upgrades and infrastructure from one control plane."*

The core principle is to **hide Kubernetes complexity from users who don't need it, and never limit users who need full control.** Simple by default, powerful when needed.

## 2. Who it is for

| Persona | Needs | Main mode |
|---|---|---|
| **Developer / small team** without deep Kubernetes knowledge | A working, secure cluster on their servers or VPS without learning kubeadm, CNI, Helm, cert-manager… | Auto, Simple |
| **DevOps / SRE engineer** | Repeatable, explainable cluster builds with full control over components, versions and YAML; lifecycle automation (upgrades, scaling, backup) | Advanced, YAML, CLI |
| **Platform team** | Templates, standards and self-service for internal teams; many clusters; API/CLI/Terraform automation | Templates, API, Enterprise |
| **Security / compliance officer** (Enterprise) | Policies enforced before deployment, approvals, audit, evidence, posture | Enterprise governance |
| **Enterprise admin** | SSO, RBAC/ABAC, tenancy, quotas, budgets, air-gapped installs | Enterprise |

## 3. What the user can do (main capabilities)

From the product spec (prompt §1):

1. Choose the infrastructure type: bare metal / existing servers over SSH first, then VPS and cloud providers.
2. Choose the Kubernetes version (from a catalog of supported, compatible versions).
3. Choose the topology: single node, standard, high availability, mission critical.
4. Choose networking (CNI): Cilium, Calico, Flannel.
5. Choose storage: local-path, NFS, Longhorn, Ceph, cloud CSI.
6. Choose ingress, using the **Kubernetes Gateway API**: Cilium Gateway, Envoy Gateway, NGINX Gateway Fabric, Traefik.
7. Choose observability: metrics, logs, traces (presets Basic / Standard / Full).
8. Choose security components: Pod Security, NetworkPolicies, cert-manager, audit logging, hardened profile.
9. Choose backup: etcd snapshots, Kubernetes resources, persistent volumes, to S3-compatible storage (AWS S3, Ceph RGW, SeaweedFS, existing MinIO/AIStor endpoints…), NFS or cloud object storage.
10. Choose additional components from the add-on catalog.
11. Review the configuration as a **plan** (dry run) with diff, warnings, estimated time and cost (when the provider publishes prices).
12. Press **Deploy Cluster**.
13. Watch the deployment in real time: a task tree and live structured logs.
14. Get a fully working cluster that passes health checks.
15. Download a kubeconfig (user-scoped, short-lived by default).
16. Manage the cluster after creation from its dashboard.
17. Upgrade Kubernetes with an upgrade advisor.
18. Add or remove add-ons.
19. Scale the cluster.
20. Run backup and restore.
21. Delete the cluster.

The system detects dependencies between components and the correct installation order automatically (DAG engine). The user is never asked to know kubeadm, systemd, containerd, CNI internals, CoreDNS, kubelet, iptables/nftables, Helm, cert-manager, StorageClasses, Prometheus, Grafana, Loki, OpenTelemetry, RBAC, etcd, cloud-init, Ansible, Terraform, SSH details, TLS or PKI (prompt §2). Advanced users can open **Advanced configuration** and control everything.

## 4. Modes

| Mode | Flow | For whom |
|---|---|---|
| **Auto** (main UX) | Name, infrastructure, environment, size, availability → the decision engine generates the full plan with reasons ("Why this configuration?") → optional overrides → Deploy | Anyone; default entry point |
| **Simple** | Infrastructure → preset (Development, Staging, Production, High Availability, Minimal, Edge, GPU, AI/ML, Custom) → review → Deploy | Users who want a known standard |
| **Advanced** | 15-step wizard (Basics … Deploy) + full configuration tree + YAML editor with UI ↔ YAML round-trip | DevOps/SRE |
| **Enterprise** | Service catalog → golden template → allowed options → policy and security validation → cost → approval → deployment → continuous health, drift detection, backup, upgrade, compliance | Organizations |

All modes produce the same declarative **ClusterSpec** and use the same engine. Switching modes keeps the data.

## 5. Editions

| | **Community** (Apache-2.0) | **Enterprise** (commercial, `ee/`) |
|---|---|---|
| Provisioning & lifecycle (all providers, distributions, add-ons, Auto/Simple/Advanced, upgrades, scaling, backup/restore, health, templates, CLI, API, webhooks, notifications) | ✓ | ✓ |
| Organizations / projects | Single org UI (multi-tenant data model) | Many orgs, teams, quotas |
| Authentication | Local accounts, API keys | + SSO (OIDC, SAML), SCIM, MFA policy, service accounts, session management |
| Authorization | Built-in roles | + custom roles, ABAC |
| Governance | Built-in safety guardrails | + policy engine (policy as code), approvals, change requests, golden templates, maintenance windows |
| Audit | Audit log, JSON/CSV export | + SIEM/syslog/S3 export, retention lock |
| Compliance & security | Hardened profiles | + compliance center, security posture, image security, vulnerability aggregation |
| Secrets | Envelope encryption with local key | + OpenBao/Vault, cloud KMS, BYOK |
| Fleet | Multi-cluster dashboard | + cluster groups, canary fleet upgrades, multi-region |
| DR & offline | Backup/restore | + cross-region DR, RPO/RTO, air-gapped bundles |
| Platform | Single-instance or simple HA | + HA reference architecture, SLOs, cost management, budgets, incidents, service catalog, Terraform provider |

The open-core boundary rule: **Community is a complete, production-grade provisioning product. Enterprise adds governance, scale and integrations. It never removes or cripples core provisioning capability** (ADR-0002). Features are gated in one place (`EntitlementService`), never by scattered `if enterprise` checks (prompt §218).

## 6. MVP

**MVP = Phases 1–4** ([ROADMAP](ROADMAP.md)): *one click from SSH-reachable Ubuntu/Debian hosts to a production-ready kubeadm cluster, observable and resumable.*

Included:
- local auth + API keys, a single org/project UI on a multi-tenant data model, audit log
- credentials vault (SSH keys, S3 keys) with envelope encryption
- bare-metal/existing-hosts provider over SSH with host-key verification and preflight checks
- kubeadm: single control plane and HA (3 control-plane nodes with kube-vip), containerd
- Cilium (default), then Calico and Flannel
- Gateway API: platform-owned CRDs + Cilium Gateway (Envoy Gateway as the alternative)
- MetalLB (or Cilium LB-IPAM)
- local-path, NFS CSI, Longhorn
- cert-manager with Let's Encrypt / internal CA / self-signed
- metrics-server, kube-prometheus-stack + Grafana dashboards, Loki
- etcd snapshot backups to S3-compatible storage
- health engine and cluster dashboard
- real-time progress with resume, retry and explainable errors
- plan/dry run; templates; Simple Mode presets; Advanced wizard; YAML ↔ UI
- kubeconfig download; cluster delete
- CLI (`create`, `plan`, `deploy --watch`, `list`, `get`, `kubeconfig`, `destroy`)
- docker compose install; CI with security scanning
- documentation in English and Russian

Not in the MVP: cloud providers, k3s/RKE2, Kubernetes upgrades and Velero-based restore (Phase 6), Auto Mode decision engine (Phase 5; presets exist), GitOps, all Enterprise features.

## 7. Non-goals

- Not "yet another UI over kubeadm" or a shell-script runner. The value is in the engine, the catalog, the compatibility rules and explainability.
- Not a general-purpose infrastructure-as-code tool. Terraform/OpenTofu and Ansible are not wrapped in the core (ADR-0013).
- Not a workload PaaS (no app build/deploy pipelines). GitOps integration covers workload delivery.
- Not a generic Kubernetes resource browser. For in-cluster browsing we link to or integrate **Headlamp** (Apache-2.0, the official successor of the archived Kubernetes Dashboard). Our UI effort goes into lifecycle, plans, diffs, explanations and governance.
- No promise of legal compliance. The platform reports controls and evidence; it never claims that an organization "is compliant".
- No invented prices or SLAs. Cost estimates appear only when the provider publishes prices, and SLA targets only with matching infrastructure.

## 8. Market context and differentiation (research, 2026-10)

| Tool | What it does well | Gap we address |
|---|---|---|
| Rancher (SUSE) | Multi-cluster management, CAPI-native since 2.13, free | Heavy (needs a Kubernetes "local" cluster); bare metal via an agent and RKE2/K3s only; no kubeadm path; little explanation of decisions |
| KubeOne / Kubermatic KKP | kubeadm over SSH (CLI), open-core KKP | KubeOne is CLI-only with no UI or recommendations; KKP targets large hosted-control-plane setups |
| Kubespray / Kubean | Battle-tested Ansible installer, offline packages | No API, UI or state store; slow and opaque on failure; stale add-ons (e.g. cert-manager 1.15, MetalLB 0.13 in v2.32) |
| k0s + k0sctl, KubeKey | Simple SSH installers | Single distribution or ecosystem; no lifecycle platform |
| Spectro Cloud Palette, Platform9, Nutanix NKP | Enterprise multi-infrastructure lifecycle on CAPI | Proprietary and expensive |
| Talos Omni | Excellent UX for Talos | Talos-only (boot an ISO, no SSH onto existing OS); BSL license; ownership changed in 2026 |
| Portainer, Headlamp, Lens | Management UIs | Do not provision clusters (Portainer removed provisioning in 2026) |

**Farvater's position**: the shortest path from *plain SSH hosts or a VPS account* to a *production-ready, current, compatibility-checked* cluster, with every decision and every failure explained, on an open core that will not be relicensed.

## 9. Success metrics

| Metric | Target (MVP) |
|---|---|
| Time from "hosts ready" to READY production cluster (6 nodes, standard add-ons) | ≤ 20 min, with zero manual steps after Deploy |
| Deployment success rate on supported OS/hardware (Tier 2/3 E2E) | ≥ 98 % |
| Recovery: interrupted deployments that resume to READY without manual node intervention | ≥ 95 % |
| Errors shown with what/why/where/how-to-fix | 100 % of known error classes |
| User actions to create a cluster in Auto Mode | ≤ 6 inputs + Deploy |
| Security | 0 secrets in logs/API (canary tests), 0 high/critical unaccepted vulnerabilities at release |

## 10. Principles that shape the product (from the spec)

- **Explain everything.** Recommendations have reasons, failures have causes and fixes, plans show diffs and irreversible steps.
- **Never create a known-broken cluster.** Compatibility engine plus version catalog.
- **Never choose risky settings silently.** Safety warnings with explicit choices.
- **Dry run everywhere.** Plan, then preview, then apply.
- **Backend is the source of truth.** Closing the browser never stops or loses a deployment.
- **Everything is API-first.** The UI and CLI are clients of the same API.
- **Security first.** Credentials are the crown jewels: encrypted, redacted, audited.
- **Production mindset from day one.** No fake deployments, no hard-coded secrets, IPs or passwords outside isolated test fixtures.
