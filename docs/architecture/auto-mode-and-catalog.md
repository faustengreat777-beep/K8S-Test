# Auto Mode, version catalog, compatibility and decision engines

> Status: **Proposed** (Phase 0). Language: English · [Русский](auto-mode-and-catalog.ru.md)
>
> Related: [ARCHITECTURE](../ARCHITECTURE.md) · [Provisioning engine](provisioning-engine.md) · [Core interfaces](core-interfaces.md) · ADR-0017 (data-driven catalog), ADR-0018 (rule-based decision engine)

Auto Mode is the main UX of Farvater (prompt §127). The user answers a few questions and the platform makes the remaining technical decisions, **transparently** (prompt §130), **overridably** (prompt §131) and **safely** (prompt §132). Three components make it work:

1. **Version catalog**: the data about what exists, which versions are supported, and how they fit together.
2. **Compatibility engine**: evaluates a concrete configuration against the catalog and rules. It blocks broken combinations and explains warnings.
3. **Decision (recommendation) engine**: turns high-level answers into a full ClusterSpec with reasons.

Presets, Simple Mode and Advanced Mode use the same three components.

## 1. Version catalog (prompt §49)

### 1.1 Principles

- **No version strings in code.** Every Kubernetes version, distribution release, runtime, add-on version, Helm chart, image and OS support statement lives in the catalog (prompt §49). Code references catalog ids.
- **Data, versioned and signed.** The catalog is a directory of YAML files in `catalog/`. It is embedded into each release, and can also be updated independently through a **signed catalog bundle** (Ed25519 signature, monotonic version, channel `stable`/`lts`/`beta`). New Kubernetes patches and add-on updates can then ship without a platform release, and air-gapped installs can import catalogs offline.
- **Pinned and reproducible.** Every resolvable item has exact versions plus **digests/checksums**: chart digests, image digests, binary SHA-256. A ResolvedSpec references a catalog version, so re-planning with the same catalog yields the same artifacts.
- **Validated at load.** Schema validation, reference integrity (every add-on has an implementation, every image list is complete), capability graph acyclic, version ranges well-formed. A broken catalog is rejected, and the previous one stays active.
- **Generated where possible.** CI jobs propose catalog updates (Kubernetes patches from `dl.k8s.io/release/stable-1.x.txt` and the release schedule; chart versions from registries; image lists from `helm template`). A human reviews and the compatibility tests run before merge.

### 1.2 Layout

```
catalog/
├── catalog.yaml                 # catalog version, channel, minimum platform version, signing metadata
├── kubernetes.yaml              # minors: status, patches, EOL, default flag, per-minor component pins (etcd, CoreDNS, pause)
├── distributions/
│   ├── kubeadm.yaml             # supported minors, package repos/binary URLs + checksums, config API version (v1beta4)
│   ├── k3s.yaml
│   └── rke2.yaml
├── runtimes/containerd.yaml     # versions, LTS flags, K8s compatibility, runc pins, config version
├── os/                          # support matrix: ubuntu.yaml, debian.yaml, rocky.yaml, almalinux.yaml (versions, kernels, notes)
├── addons/                      # one file per add-on: versions, chart refs+digests, images, K8s ranges, capabilities, ports, defaults
│   ├── cilium.yaml
│   ├── calico.yaml
│   ├── flannel.yaml
│   ├── gateway-api-crds.yaml
│   ├── envoy-gateway.yaml
│   ├── metallb.yaml
│   ├── kube-vip.yaml
│   ├── cert-manager.yaml
│   ├── longhorn.yaml
│   └── …
├── rules/                       # compatibility rules (declarative), see §2
├── presets/                     # Minimal, Development, Staging, Production, High Availability, Edge, GPU, AI/ML
└── estimates.yaml               # default task durations for time estimates (until local history exists)
```

Example entries (values reflect the research baseline of 2026-10-06; see [technology stack](technology-stack.md#3-managed-cluster-components-catalog-baseline)):

```yaml
# catalog/kubernetes.yaml (excerpt)
minors:
  - minor: "1.37"
    status: supported            # current | supported | deprecated | blocked
    default: false
    latestPatch: "1.37.1"
    eol: "2027-10-28"
    components: { etcd: "3.7.0-0", coredns: "v1.14.6", pause: "3.10.2" }
    notes: [ "kubeadm v1beta3 config removed; v1beta4 only", "etcd 3.7 default: no binary rollback after upgrade" ]
  - minor: "1.36"
    status: current
    default: true                # default for new clusters (widest add-on compatibility today)
    latestPatch: "1.36.5"
    eol: "2027-06-28"
    components: { etcd: "3.6.8-0", coredns: "v1.14.2", pause: "3.10.2" }
  - minor: "1.35"
    status: supported
    latestPatch: "1.35.9"
    eol: "2027-02-28"
  - minor: "1.34"
    status: deprecated           # EOL 2026-10-27 — not offered for new clusters
    latestPatch: "1.34.12"
    eol: "2026-10-27"
```

```yaml
# catalog/addons/cilium.yaml (excerpt)
id: cilium
category: cni
versions:
  - version: "1.20.2"
    chart: { ref: "oci://quay.io/cilium/charts/cilium", version: "1.20.2", digest: "sha256:…" }
    kubernetes: { tested: ">=1.33 <1.37", allowedUntested: ">=1.37 <1.38" }   # untested → WARNING, not BLOCK
    requires: { gatewayApiCRDs: ">=1.6.1" }                                     # for Cilium Gateway API support
    images: [ "quay.io/cilium/cilium:v1.20.2@sha256:…", "quay.io/cilium/operator-generic:v1.20.2@sha256:…", … ]
    status: recommended
  - version: "1.19.8"
    kubernetes: { tested: ">=1.32 <1.36" }
    status: supported
features:
  kubeProxyReplacement: { status: ga }
  hubble: { status: ga }
  encryption.wireguard: { status: ga, note: "pod-to-pod GA; node-to-node beta" }
  l2Announcements: { status: beta }
  gatewayAPI: { status: ga }
ports: [ { port: 4240, protocol: TCP, purpose: health }, { port: 8472, protocol: UDP, purpose: vxlan } ]
```

### 1.3 Release channels and status semantics

| Status | Meaning in UI and engines |
|---|---|
| `current` / `recommended` | Default choice; fully tested |
| `supported` | Selectable; tested |
| `untested` combination | Selectable with WARNING ("Cilium 1.20 is not yet tested on Kubernetes 1.37"); blocked by enterprise policy if configured |
| `deprecated` | Not offered for new clusters; existing clusters get upgrade recommendations |
| `blocked` | Never selectable (known-broken, security issue, EOL) — e.g. HCCM ≤ v1.30.0 |

## 2. Compatibility engine (prompt §48)

Purpose: **never let the user create a cluster that is known to be broken.** Explain every restriction.

### 2.1 Inputs and outputs

```
Check(resolvedSpec, catalog, [inventoryFacts]) → []Finding{severity: BLOCK|WARNING|INFO, code, path, params, remediation}
```

- Runs at **edit time** (UI asks `/validate` as the user types), at **plan time** (hard gate) and in the **upgrade advisor** (target version against installed add-ons, OS and deprecated APIs).
- With inventory facts (after discovery/preflight), it also checks hardware and OS: CPU architecture, x86-64-v3 for Rocky 10, kernel version for nftables kube-proxy (≥ 5.13), cgroup v2, RAM/disk minimums per role and add-on set.

### 2.2 Rule types

Rules are **declarative data** (`catalog/rules/*.yaml`) evaluated by a small Go engine. Rules that data cannot express are written as Go rule functions, registered by id and unit-tested.

| Rule type | Example |
|---|---|
| Version range | add-on `cilium@1.20.x` requires `kubernetes ∈ [1.33, 1.37)` tested → else WARNING (untested) or BLOCK (`>= 1.38`) |
| Requires capability | `kube-prometheus-stack` with persistence requires `default-storage-class` |
| Conflicts | two CNIs; `flannel` + `networkPolicies: enforced` (no NetworkPolicy support) → BLOCK with remediation "choose Cilium or Calico, or add kube-network-policies" |
| Provider capability | `loadBalancer: metallb` on provider with `l3OnlyNetwork` (Hetzner Cloud) → BLOCK, remediation "use provider load balancer" |
| OS / arch | Rocky Linux 10 requires x86-64-v3 CPU; Ubuntu 22.04 not supported for K8s ≥ 1.36 (example) |
| Resource minimums | Longhorn with `replicaCount: 3` requires ≥ 3 schedulable nodes with ≥ N GiB free disk |
| Topology | HA requires odd number of control-plane nodes ≥ 3; control-plane nodes across failure domains when available (prompt §176) |
| Upgrade path | upgrade skips a minor → BLOCK; etcd 3.6 < 3.6.11 before 3.7 → BLOCK with "upgrade etcd first" |
| Deprecation | target K8s removes an API still used by installed resources (upgrade advisor) → BLOCK with resource list |
| Feature status | Cilium L2 announcements → INFO "beta feature" in production → WARNING |

### 2.3 UI integration

- The UI hides or disables options that would produce BLOCK findings for the current context and shows the reason in a tooltip. For example, MetalLB is disabled on Hetzner Cloud with "Hetzner Cloud networks are layer 3; use the Hetzner load balancer".
- WARNING findings appear inline and in Review. They need acknowledgement for production environments.
- Every finding has a stable code (shared error catalog), parameters and localized remediation.

## 3. Decision engine — Auto Mode (prompt §128–129)

### 3.1 Inputs

| Input | Values |
|---|---|
| Environment | Development, Staging, Production, Enterprise |
| Availability | Single Node, Standard, High Availability, Mission Critical |
| Size | Small, Medium, Large, XLarge (maps to node counts and sizes, provider-aware) |
| Workload | General, Web, Database, AI/ML, GPU, High Network, High Storage IO, Edge |
| Budget | Low, Balanced, Performance, Unlimited |
| Compliance | None, SOC 2, ISO 27001, HIPAA, PCI DSS, Custom (affects security profile, audit, backup, encryption choices; never claims compliance) |
| Infrastructure | provider account + **capabilities and inventory** (regions, instance types, prices, L2/L3 network, LB support) or declared hosts with facts |
| Region / failure domains | from provider or host labels |
| OS | detected or chosen |
| User preferences | org defaults (e.g. "prefer Calico"), pinned overrides from the UI |
| Policies (ee) | constraints from the policy engine (e.g. "production must use Cilium") applied as hard constraints before rules run |

### 3.2 Rule pipeline

The engine is a **pure, deterministic function** with no I/O. Inventory and catalog come in as inputs. It runs an **ordered list of rules**. Each rule reads the inputs and the decisions so far, and emits `Decision`s:

```go
type Decision struct {
    Path        string      // JSON pointer in ClusterSpec, e.g. "/spec/networking/cni/provider"
    Value       any
    Reasons     []Reason    // i18n keys + params: "environment.production", "availability.ha", "feature.networkPolicies"
    Alternatives []Alternative // other valid values with trade-offs, for the [Change] menu
    Risk        []RiskTag   // data-loss | downtime | security | cost | compatibility
    Pinned      bool        // set by user override; rules must not change pinned paths
    RuleID      string
}
```

Rule order (simplified):
1. **Constraints**: apply policies (ee) and pinned user overrides.
2. **Topology**: control-plane count from availability (Single Node → 1, Standard → 1 with warning in production, HA → 3, Mission Critical → 5 across ≥ 3 failure domains); worker count and size from size, workload and budget; failure-domain spreading.
3. **Kubernetes**: distribution (bare metal/VPS → kubeadm; Edge with small nodes → k3s once available), minor = catalog `default` unless constraints say otherwise, patch = latest in that minor.
4. **Networking**: CNI (production/HA/network policies → Cilium; k3s edge → Flannel + kube-network-policies, or Cilium if resources allow); pod/service CIDRs chosen to avoid collisions with discovered host networks; kube-proxy replacement when Cilium; Gateway implementation (Cilium Gateway when CNI = Cilium, otherwise Envoy Gateway); control-plane endpoint (kube-vip on L2-capable bare metal, provider LB on clouds, external LB if declared).
5. **Load balancer for Services**: MetalLB L2 (bare metal with L2 adjacency), Cilium LB-IPAM + BGP (if BGP peers given), provider LB (clouds), NodePort (development without LB).
6. **Storage**: development → local-path; production bare metal with ≥ 3 workers → Longhorn (replicas 3; 2 if budget low with warning); large/high-IO → Rook-Ceph suggested as alternative; cloud → provider CSI; default StorageClass set.
7. **Certificates**: cert-manager always; Let's Encrypt HTTP-01 if public DNS/ingress reachable, otherwise internal CA / self-signed for development.
8. **Observability**: Development → Basic (metrics-server); Staging → Standard (Prometheus + Grafana + Loki); Production → Full (adds Tempo/OpenTelemetry, alerting) unless budget is Low.
9. **Security**: profile hardened for Production/Enterprise or any compliance selection; PodSecurity restricted; default-deny NetworkPolicy templates; audit logging; secrets encryption at rest.
10. **Backup**: production → etcd snapshots + Velero to S3-compatible target (asks for destination; if none, WARNING "backup destination required" and policy may BLOCK).
11. **Resources**: node labels/taints (GPU nodes tainted; dedicated ingress nodes for High Network), resource requests for add-ons by size tier.
12. **Add-on versions**: from catalog per chosen Kubernetes minor (recommended versions; compatibility engine validates).
13. **Safety review** (prompt §132): emits warnings for data-loss, downtime, security-degradation, high-cost and incompatible choices. Examples: 1 CP in production; Longhorn replicas < 3; no backup destination; budget "Unlimited" with > N nodes; untested version combinations.
14. **Estimates**: time from estimates/history; cost from provider pricing, otherwise "Cost estimation unavailable" (prompt §36: never invent prices).

### 3.3 Transparency (prompt §130)

Every decision stores its reasons, so the UI can answer "Why this configuration?" field by field:

```
Why Cilium?
  Selected because:
   - Production environment
   - High availability cluster
   - Network policies enabled
   - Hubble observability requested
  Alternatives: Calico (BGP-native networking, Calico Ingress Gateway), Flannel (lightweight; no NetworkPolicy)

Why 3 control-plane nodes?
  Selected because:
   - High availability requested
   - etcd quorum requires an odd number of members (tolerates 1 failure with 3)
```

### 3.4 Overrides (prompt §131)

- A user override **pins** a path (`Pinned: true`) and re-runs the whole pipeline. Dependent decisions are recomputed: switching CNI to Flannel changes Gateway to Envoy Gateway, removes Hubble, and adds a NetworkPolicy warning. The UI highlights what changed because of the override.
- Overrides that violate a BLOCK rule are rejected with the finding. Overrides that trigger warnings need acknowledgement in production.

### 3.5 Safety (prompt §132)

Auto Mode never **silently** selects risky settings. Warnings carry explicit choices:

```
⚠ Warning
You selected 1 control-plane node for a production cluster.
This configuration does not provide control-plane high availability.
[Continue anyway]   [Enable HA]
```

The decision log records the acknowledgement (who, when) in the spec revision metadata and the audit log.

### 3.6 Testing

- Golden-file tests: for a matrix of inputs (environment × availability × size × workload × budget × provider capabilities), the full Recommendation (spec + reasons + warnings) is snapshotted. Changes show up as reviewable diffs.
- Property tests: every recommendation passes the compatibility engine with zero BLOCK findings; pinned paths are never changed; the same input always gives the same output.

## 4. Presets (prompt §3, §74)

Presets are **named inputs plus defaults** for the decision engine, stored in `catalog/presets/`. They are not separate code paths. Each preset can also be saved as a template.

| Preset | Key choices |
|---|---|
| Minimal | 1 node (CP+worker), Kubernetes, CNI, CoreDNS, metrics-server |
| Development | 1 CP + 1–2 workers, CNI, Gateway, local-path storage, metrics-server |
| Staging | 1–3 CP, 2–3 workers, Production-like add-ons with smaller sizing, Standard observability |
| Production | 3 CP (HA), ≥ 3 workers, Cilium, Gateway + TLS (cert-manager), Longhorn/provider CSI, monitoring, logging, backup, hardened security |
| High Availability | Production + 5 CP for Mission Critical, failure-domain spreading, PodDisruptionBudgets for add-ons, backup frequency up |
| Edge | Small footprint (k3s when available), Flannel or Cilium minimal, local-path, minimal observability, tolerant of intermittent connectivity |
| GPU | GPU node pool with labels/taints, NVIDIA GPU Operator, monitoring with GPU dashboards |
| AI/ML | GPU + high-IO storage (Rook-Ceph or provider CSI), node labels for scheduling, monitoring, optional object storage |
| Custom | Starts from Production defaults with everything editable |

## 5. Cost estimation (prompt §36)

- Only providers with the `Pricing` capability return prices (e.g. Hetzner Cloud pricing API, cloud price lists). The quote includes source and timestamp, and is cached with a TTL.
- The breakdown covers control plane, workers, storage, load balancers and other items (traffic, IPs) where the provider prices them.
- Bare metal and existing infrastructure show "Cost estimation unavailable" (and optional user-entered internal costs later).
- The platform **never invents or extrapolates prices**. Missing items are listed as "not included".
- Enterprise cost management (budgets, allocation) builds on the same quotes (Phase 14).

## 6. Upgrade advisor (prompt §71, §229)

The upgrade advisor uses the same engines. It takes the current state (installed versions, add-ons, OS, CNI/CSI/Gateway, CRDs) and the target Kubernetes minor, and produces:
- **readiness score**: weighted checks passed;
- **blockers**: incompatible add-on versions, minor skips, deprecated or removed APIs in use (live scan of the cluster's served resources), etcd version requirements, OS support;
- **warnings**: untested combinations, behaviour changes (e.g. SELinuxMount GA in 1.37 on SELinux-enforcing nodes), expected downtime per step;
- **upgrade plan**: control plane node by node (etcd snapshot first) → add-ons that must move first → workers in batches with drain → validation.
