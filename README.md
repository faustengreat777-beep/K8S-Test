# Farvater

**From bare infrastructure to a production-ready Kubernetes cluster — with minimum user interaction.**

Language: English · [Русский](README.ru.md)

> **Status: Phase 0 — architecture proposal.** This repository contains the product and architecture design only: no application code yet. Implementation starts after the owner approves the architecture (see [ROADMAP](docs/ROADMAP.md)).

*Farvater* (Russian «фарватер», "fairway") is the safe, marked channel that ships follow into port. Farvater guides you from SSH-reachable servers or cloud accounts to a secure, observable, backed-up Kubernetes cluster along a proven path, and keeps the cluster healthy afterwards.

## What it will do

- **One click to a production-ready cluster.** Answer five questions in **Auto Mode** (name, infrastructure, environment, size, availability), review an explained plan and press **Deploy**. Or pick a preset (Simple Mode), or control everything in the 15-step wizard or YAML (Advanced Mode).
- **Real provisioning engine.** A dependency-aware DAG of idempotent tasks with live progress, explainable failures (what / why / where / how to fix), retry, resume and safe rollback. Closing the browser never stops a deployment.
- **Batteries included, versions verified.** kubeadm (k3s and RKE2 later), containerd, Cilium/Calico/Flannel, **Gateway API** (Cilium Gateway, Envoy Gateway, …), MetalLB / kube-vip, local-path/NFS/Longhorn/Ceph, cert-manager, Prometheus/Grafana/Loki/OpenTelemetry, Velero and etcd backups, Argo CD/Flux. A signed version catalog and a compatibility engine prevent broken combinations.
- **Lifecycle.** Upgrades with an upgrade advisor, scaling, node operations, backup/restore, add-on management, health dashboard, multi-cluster overview.
- **API-first.** The REST API with a committed OpenAPI 3.1 contract is used by the web UI, the `farvater` CLI and (later) a Terraform provider.
- **Security-first.** Envelope-encrypted credential vault, tenant isolation with PostgreSQL RLS, no shell interpolation, signed releases.
- **Open core.** The Community edition (Apache-2.0) is a complete provisioning product. Enterprise adds SSO/SCIM, custom roles/ABAC, policy as code, approvals, compliance, fleet operations, KMS/BYOK and air-gapped installs.

## Documentation

| | English | Русский |
|---|---|---|
| Product vision, modes, editions, MVP | [PRODUCT](docs/PRODUCT.md) | [PRODUCT](docs/PRODUCT.ru.md) |
| Architecture (hub) | [ARCHITECTURE](docs/ARCHITECTURE.md) | [ARCHITECTURE](docs/ARCHITECTURE.ru.md) |
| Roadmap | [ROADMAP](docs/ROADMAP.md) | [ROADMAP](docs/ROADMAP.ru.md) |
| Decisions (ADR index) | [DECISIONS](docs/DECISIONS.md) | [DECISIONS](docs/DECISIONS.ru.md) |
| Technology stack and version baseline | [technology-stack](docs/architecture/technology-stack.md) | [technology-stack](docs/architecture/technology-stack.ru.md) |
| Database schema | [data-model](docs/architecture/data-model.md) | [data-model](docs/architecture/data-model.ru.md) |
| Provisioning engine | [provisioning-engine](docs/architecture/provisioning-engine.md) | [provisioning-engine](docs/architecture/provisioning-engine.ru.md) |
| Core interfaces and plugins | [core-interfaces](docs/architecture/core-interfaces.md) | [core-interfaces](docs/architecture/core-interfaces.ru.md) |
| Auto Mode, catalog, compatibility | [auto-mode-and-catalog](docs/architecture/auto-mode-and-catalog.md) | [auto-mode-and-catalog](docs/architecture/auto-mode-and-catalog.ru.md) |
| API design | [api-design](docs/architecture/api-design.md) | [api-design](docs/architecture/api-design.ru.md) |
| UI architecture | [ui-architecture](docs/architecture/ui-architecture.md) | [ui-architecture](docs/architecture/ui-architecture.ru.md) |
| Deployment of the platform | [deployment-model](docs/architecture/deployment-model.md) | [deployment-model](docs/architecture/deployment-model.ru.md) |
| Testing strategy | [testing-strategy](docs/architecture/testing-strategy.md) | [testing-strategy](docs/architecture/testing-strategy.ru.md) |
| Repository structure | [repository-structure](docs/architecture/repository-structure.md) | [repository-structure](docs/architecture/repository-structure.ru.md) |
| Security model | [SECURITY_MODEL](docs/security/SECURITY_MODEL.md) | [SECURITY_MODEL](docs/security/SECURITY_MODEL.ru.md) |
| Threat model | [THREAT_MODEL](docs/security/THREAT_MODEL.md) | [THREAT_MODEL](docs/security/THREAT_MODEL.ru.md) |
| Architecture decision records | [docs/architecture/adr/](docs/architecture/adr/) | [docs/architecture/adr/](docs/architecture/adr/) (`*.ru.md`) |

`DEVELOPMENT.md`, `DEPLOYMENT.md`, `CONTRIBUTING.md`, `SECURITY.md` and `API.md` arrive with the code in Phase 1.

## Roadmap at a glance

Phase 0 Architecture (now) → 1 Foundation → 2 Provisioning engine → 3 Bare metal + kubeadm + Cilium → 4 Production cluster (**MVP**) → 5 Auto Mode → 6 Lifecycle → 7 Multi-provider → 8 GitOps → 9–14 Enterprise.

## License

The Farvater core is licensed under the [Apache License 2.0](LICENSE). Enterprise features will live in `ee/` under a separate commercial license.
