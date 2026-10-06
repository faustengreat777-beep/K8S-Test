# ADR-0013: Infrastructure tooling boundaries — native Go providers, no Terraform/Ansible in the core, Cluster API evaluated for clouds in Phase 7

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0013-infrastructure-tooling-boundaries.ru.md)

## Context

The spec allows Terraform/OpenTofu and Ansible "if useful" but warns against turning the product into a UI around Terraform. It asks for clear responsibilities: Terraform/OpenTofu → infrastructure provisioning, Ansible/bootstrap engine → OS/node configuration, Kubernetes engine → cluster, Helm → add-ons (prompt §55). It also requires a provider-agnostic `InfrastructureProvider` abstraction with bare metal, VPS (Hetzner, DigitalOcean, OVH, Vultr, Linode), cloud (AWS, GCP, Azure) and existing infrastructure (prompt §5–6). **Cluster API (CAPI)** is the upstream Kubernetes project for declarative cluster lifecycle across providers. It has infrastructure providers (CAPA, CAPG, CAPZ, CAPH for Hetzner, Metal3/BYOH for bare metal) and bootstrap/control-plane providers (kubeadm, k3s, RKE2, Talos), but it needs a **management cluster**.

## Decision

| Responsibility | Owner in Farvater | Not used in core |
|---|---|---|
| Infrastructure provisioning (instances, networks, firewalls, LBs, volumes) | `InfrastructureProvider` plugins written in Go against provider APIs/SDKs (hcloud-go, aws-sdk-go-v2, Google Cloud Go, Azure SDK for Go), or **declared hosts** for bare metal/existing infrastructure | Terraform/OpenTofu |
| OS and node configuration | Bootstrap layer in Go over SSH: typed commands, SFTP-rendered configs, signed node helper (ADR-0015) | Ansible |
| Kubernetes cluster bootstrap and lifecycle | `KubernetesDistribution` plugins (kubeadm first) driven by our engine (ADR-0008) | — |
| Add-ons | Helm service over the Helm 4 SDK (ADR-0010) | `helm` CLI |
| Declarative desired state | ClusterSpec + spec revisions + planner diff (dry run/plan everywhere) | — |

Reasons for native providers:
1. **One execution model**: every step is a task in our DAG with typed errors, retries, events, resume and explainable failures. Terraform runs are coarse-grained (plan/apply of a whole state), and their errors are text. Ansible playbooks are untyped and hard to resume step by step.
2. **No extra runtimes**: no Terraform/OpenTofu binary or provider plugins, no Python/Ansible on workers. That means simpler air-gap bundles and a smaller attack surface.
3. **State ownership**: our PostgreSQL is the single source of truth. Terraform state files would be a second state store holding secrets.

Cluster API:
- **Not embedded in the MVP.** Research (2026-10, CAPI v1.14.2) shows:
  - a **management cluster** is mandatory (kube-apiserver, CRDs, webhooks, cert-manager), which is a chicken-and-egg problem on bare metal and weight for self-hosted/air-gapped installs;
  - there is **no maintained CAPI provider for existing SSH hosts** with kubeadm/k3s/RKE2. BYOH has been dormant since 2024, and k0smotron RemoteMachine is tested with k0s only. Metal3 and Tinkerbell need BMC/PXE. This gap is exactly our MVP target;
  - CAPI's default upgrade model is immutable machine replacement, which doesn't fit fixed hosts, and **ClusterClass, Runtime SDK and in-place updates are still alpha** feature gates;
  - **v1beta1 stops being served in CAPI v1.16** (≈ April 2027) while CAPA, CAPZ, CAPG and CAPH still declared the v1beta1 contract in October 2026, so a provider-compatibility cliff is ahead;
  - only `util/*` and `cmd/clusterctl/client` are stable Go packages, so CAPI must be integrated through its CRDs, not embedded as a library.
- **Re-evaluated at the start of Phase 7** for cloud providers, as a possible `InfrastructureProvider` implementation (`capi-*`). Preferred pattern if adopted: bootstrap through a temporary management plane, then **pivot CAPI into each workload cluster** (self-managed, as Spectro Cloud Palette does). That avoids a central management cluster as a single point of failure. The integration would use v1beta2 CRDs only, pinned provider versions managed by Cluster API Operator or clusterctl, and cert-manager plus image overrides in the air-gap bundle. Criteria: v1beta2 contract readiness of CAPA/CAPG/CAPZ/CAPH, the upgrade orchestration we would reuse, ClusterClass maturity, effort vs native SDKs, air-gap impact. The `InfrastructureProvider` and `KubernetesDistribution` interfaces stay coarse enough to allow a CAPI-backed implementation.
- **CAPI-compatible vocabulary now**: internal status conditions follow CAPI v1beta2 style (`Ready`, `Available`, `UpToDate` with reason and message), and node pools map to MachineDeployment concepts. An "export to Cluster API manifests" path (later) can then be a mapping rather than a rewrite, which supports our no-lock-in story.
- **Interoperability**: the import feature (Phase 6) should recognise CAPI-managed clusters and avoid fighting their controllers.

Terraform the other way round: we will ship a **Terraform/OpenTofu provider for the Farvater API** (Enterprise roadmap, prompt §198). It only calls the public API and contains no business logic.

OpenTofu as an optional provider: if customers need exotic infrastructure that we don't support natively, an `opentofu` provider plugin that runs user-supplied modules in a sandbox can be considered later. It would be opt-in and never the default path.

## Alternatives considered

| Option | Why not |
|---|---|
| Terraform/OpenTofu for all infrastructure (core) | Coarse progress and errors, a second state store, binary and plugin distribution for air-gap; the spec warns against becoming a "UI around Terraform" |
| Ansible (e.g. Kubespray) for node and cluster bootstrap | A proven tool, but it imports Kubespray's whole model and Python runtime, makes typed resume and explainable failures hard, and duplicates our engine |
| Cluster API as the core from day one | Requires a management cluster even for the first bare-metal cluster; heavier self-hosting; weak SSH bare-metal support. It may be the right engine for clouds later |
| Crossplane as the provisioning layer | Needs a management cluster and pulls in a large dependency set. Its abstraction overlaps with our catalog/spec model |

## Consequences

- Positive: uniform UX and failure semantics across providers, minimal runtime dependencies, full control of state.
- Negative: we write and maintain provider integrations ourselves. This is mitigated by capability-driven interfaces, a shared provider contract test suite, one provider at a time (Phase 7), and the CAPI option for clouds.

## References

- Cluster API book (management cluster, providers, ClusterClass); Kubespray; OpenTofu; Crossplane
- [Core interfaces §3](../core-interfaces.md), [ROADMAP Phase 7](../../ROADMAP.md)
