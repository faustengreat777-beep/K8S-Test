# Technology stack and version baseline

> Status: **Proposed** (Phase 0). Research date: **2026-10-06**. Language: English · [Русский](technology-stack.ru.md)
>
> Related: [ARCHITECTURE](../ARCHITECTURE.md) · [DECISIONS](../DECISIONS.md) · [Auto Mode, catalog and compatibility](auto-mode-and-catalog.md)

This document records **what we use, why, and which versions were current when Phase 0 research was done**. The research checked primary sources (release pages, `go.mod`/`Chart.yaml` at release tags, package registries, the Go module proxy, official docs). Versions here are a **baseline for Phase 1 decisions**, not a pin list. Pins live in `go.mod`, `pnpm-lock.yaml` and the [version catalog](auto-mode-and-catalog.md#1-version-catalog-prompt-49), and are updated continuously.

Confidence: unless marked otherwise, each version was confirmed in at least one primary source (raw release file, registry or Go proxy). "medium" means one credible source only.

## 1. Platform backend

| Component | Choice | Baseline version | License | Why |
|---|---|---|---|---|
| Language | Go | toolchain **1.27.1** (go.mod `go 1.26`) | BSD-3 | Kubernetes ecosystem, client-go/Helm in-process, concurrency, single static binary. Go 1.26 is the floor because client-go v0.37, Helm v4 and x/crypto need it |
| HTTP router | chi v5 | 5.3.2 | MIT | net/http-compatible middleware ecosystem, otelhttp |
| API framework | Huma v2 | 2.39.1 | MIT | Typed operations → OpenAPI **3.1**, validation, RFC 9457 errors (ADR-0005) |
| API contract tooling | oasdiff, vacuum (libopenapi) | 1.33.0, 0.30.x | Apache-2.0, MIT | Breaking-change gate, OpenAPI lint |
| Database | PostgreSQL | **18.6** (min 16) | PostgreSQL | Only required stateful dependency (ADR-0006); PG 19 GA expected late Oct 2026 |
| DB driver | pgx v5 | 5.11.0 | MIT | Best PostgreSQL driver for Go, LISTEN/NOTIFY, pools |
| Queries | sqlc | 1.31.1 | MIT | Typed SQL, no ORM |
| Migrations | goose v3 | 3.28.0 | MIT | Embedded SQL migrations; Atlas avoided (CLI EULA) |
| Job queue | River | 0.49.0 (pre-1.0, pinned) | MPL-2.0 (Pro features unused) | Transactional enqueue on PostgreSQL (ADR-0007) |
| Kubernetes client | client-go / api / apimachinery | 0.37.1 | Apache-2.0 | Matches K8s 1.37; all `k8s.io/*` on one minor |
| Controller helpers | controller-runtime | 0.25.2 (Helm 4.3 pins 0.24.x — align at Phase 1) | Apache-2.0 | Waiters, apply helpers where useful |
| Helm | Helm SDK **v4** | 4.3.0 | Apache-2.0 | Helm 3 security fixes end 2027-02-10 (ADR-0010) |
| SSH | golang.org/x/crypto/ssh (+agent, knownhosts) | x/crypto 0.57.0 | BSD-3 | Jump hosts, agent, host keys, PQ-hybrid KEX (`mlkem768x25519`) |
| SFTP | pkg/sftp | 1.13.11 | BSD-2 | Atomic config/binary uploads |
| ssh_config parsing | kevinburke/ssh_config | 1.6.0 | MIT | Import user SSH configs (optional) |
| Crypto | Google Tink Go v2 (+ gcpkms v2, awskms **v3**, hcvault v2) | 2.8.0 | Apache-2.0 | Envelope encryption, keyset rotation, KMS (ADR-0014) |
| Password hashing | argon2id (x/crypto) | 0.57.0 | BSD-3 | OWASP-recommended |
| File encryption (exports, bundles) | age | 1.3.2 | BSD-3 | Encrypted kubeconfig/backup exports; post-quantum recipients available |
| OIDC (ee) | coreos/go-oidc v3 (+ x/oauth2); zitadel/oidc if we must act as OP | 3.21.0; 3.51.x | Apache-2.0 | Relying party for SSO; OP for device flow later |
| SAML (ee) | russellhaering/gosaml2 + goxmldsig | 0.12.0; 1.6.1 | Apache-2.0 | crewjam/saml is dormant since 2025-05 |
| MFA (ee) | go-webauthn/webauthn; pquerna/otp | 0.18.2; 1.5.0 | BSD-3; Apache-2.0 | Passkeys, TOTP |
| SCIM (ee) | own implementation (or elimity-com/scim pinned to a commit) | untagged | MIT | Library lacks bulk/sort; IdPs exercise those paths |
| Policy (ee) | OPA embedded (`opa/v1/rego`) for policy-as-code; cedar-go for ABAC decisions | 1.21.1; 1.8.0 | Apache-2.0 | Choice per use case in Phase 10 |
| Secrets backend (ee) | OpenBao (API) / Vault (API only, BSL) | OpenBao 2.7.1 | MPL-2.0 | Never bundle Vault binaries |
| CLI | cobra + koanf | 1.10.2; 2.3.7 | Apache-2.0; MIT | koanf is lighter than viper |
| Observability | log/slog + OpenTelemetry Go (traces, metrics, **logs stable**) + Prometheus exporter | OTel 1.47.0 | Apache-2.0 | Self-observability (prompt §60) |
| Lint / vuln | golangci-lint v2, govulncheck | 2.14.0; 1.8.0 | GPL-3 (dev tool only); BSD-3 | v2 config format |
| Tests | Go testing, testcontainers-go | 0.44.0 | MIT | Real PostgreSQL in integration tests |
| Durable workflow alternative (spike) | DBOS Transact Go | 1.5.0 | MIT | Evaluate before Phase 2 (ADR-0007/0008) |
| Optional cache / rate limit | Valkey (+ valkey-go or go-redis) | 9.1.2 | BSD-3 | Only for large HA installs; **not Redis** (Redis 8 is RSAL/SSPL/AGPL) |

Go 1.27 note: `encoding/json` now runs on the json/v2 implementation. Error strings may differ, so tests must not assert on JSON error text.

## 2. Frontend

{{FRONTEND_SECTION}}

## 3. Managed-cluster components (catalog baseline)

### 3.1 Kubernetes and node runtime

| Component | Baseline | Status / notes |
|---|---|---|
| Kubernetes | **1.37.1** (latest), **1.36.5** (default for new clusters), 1.35.9 | EOL: 1.37 → 2027-10-28; 1.36 → 2027-06-28; 1.35 → 2027-02-28; **1.34 → 2026-10-27 (not offered)**; 1.38 planned 2026-12-16 |
| kubeadm config API | `kubeadm.k8s.io/v1beta4` | v1beta3 removed in 1.37 |
| etcd (kubeadm defaults) | 1.36 → 3.6.8-0; 1.37 → 3.7.0-0 (latest 3.7.2) | 3.7 removed the v2 store; going 3.6 → 3.7 requires every member on ≥ 3.6.11; no binary rollback; restore with `etcdutl` |
| CoreDNS (kubeadm) | 1.36 → v1.14.2; 1.37 → v1.14.6 | Managed by kubeadm |
| containerd | **2.3.x LTS** (2.3.6, until 2028-04-30) default; 2.4.1 opt-in | Upstream binaries, not distro packages; static build on glibc < 2.35 |
| runc | 1.5.2 | Pinned by catalog |
| kube-proxy | `nftables` mode (or none with Cilium KPR) | IPVS deprecated (off by default in 1.40, removed in 1.43) |
| k3s / RKE2 (Phase 7) | v1.37.1+k3s1 / v1.37.1+rke2r1 (stable channel 1.36.5) | Bundled components disabled in favour of our add-ons |
| Talos / k0s (candidates) | Talos 1.14.2; k0s 1.36.4 | Future distribution plugins |

### 3.2 Node operating systems

| OS | Status at launch | Notes |
|---|---|---|
| Ubuntu 24.04 LTS | Supported | Kernel 6.8 (HWE 6.17/7.0), cgroup v2 |
| Ubuntu 26.04 LTS | Supported | Kernel 7.0, systemd 259 (no cgroup v1), Rust coreutils/sudo-rs; archive containerd too old → upstream containerd |
| Debian 13 (trixie) | Supported | Stable since 2025; full support until 2028-08 (medium confidence on kernel/systemd details) |
| Debian 12 (bookworm) | Best-effort | LTS since 2026-06 |
| Rocky Linux / AlmaLinux 9.x, 10.x | Phase 7 | SELinux enforcing with container-selinux; Rocky 10 needs x86-64-v3 CPUs (Alma 10 has v2 builds) |

### 3.3 Networking

| Component | Baseline | Notes |
|---|---|---|
| Cilium (default CNI) | **1.20.2** | Tested on K8s 1.33–1.36; Gateway API (v1.6 conformance); needs Gateway API CRDs ≥ 1.6.1; L2 announcements beta; OCI chart `oci://quay.io/cilium/charts/cilium` |
| Calico | 3.33.0 (tigera-operator 1.44.0) | Tested on K8s 1.35–1.37; includes "Calico Ingress Gateway" (Envoy Gateway) and bundles Gateway API CRDs (we disable them) |
| Flannel | 0.28.9 | No NetworkPolicy of its own (add kube-network-policies) |
| Gateway API CRDs | **v1.6.2** (standard channel) | Platform-owned; safe-upgrade VAP blocks downgrades; TLSRoute storage migration guard |
| Envoy Gateway | 1.9.2 | K8s 1.33–1.36; ~6-month support per minor |
| NGINX Gateway Fabric | 2.7.2 | Full v1.6 conformance (5 profiles) |
| Traefik | 3.7.14 | Partial HTTP core conformance |
| ingress-nginx | **retired** (archived 2026-03-24) | Never offered; migration assistant via ingress2gateway 1.2.x |
| MetalLB | 0.16.1 | FRR-K8s default BGP backend; metrics HTTPS-only |
| kube-vip | 1.2.4 | Static pod for kubeadm CP VIP (super-admin.conf bootstrap) |
| Hetzner CCM / CSI | 1.39.0 / 2.23.0 | Block versions ≤ 1.30.0 (they panic against the current API) |

### 3.4 Add-ons

| Area | Component | Baseline | License | Notes |
|---|---|---|---|---|
| Certificates | cert-manager | 1.21.2 | Apache-2.0 | K8s 1.33–1.36 listed (1.37 not yet) → WARNING on 1.37; OCI charts are the source of truth |
| | trust-manager | 0.25.0 | Apache-2.0 | pre-1.0 |
| Metrics | metrics-server | 0.9.0 | Apache-2.0 | Needs K8s ≥ 1.34 |
| | kube-prometheus-stack | chart 91.9.0 (Operator 0.94.1, Prometheus 3.15.0) | Apache-2.0 | Grafana subchart from grafana-community |
| Dashboards | Grafana | 13.2.3 | **AGPL-3.0** | Installed unmodified; never embedded (ADR-0024) |
| Logs | Loki | 3.7.8 (grafana-community chart) | **AGPL-3.0** | Monolithic / HA-monolithic; SSD deprecated |
| | Grafana Alloy | 1.20.1 | Apache-2.0 | Default agent (Promtail EOL 2026-03-02) |
| | Fluent Bit | 5.1.3 (medium) | Apache-2.0 | Alternative agent |
| Traces | Tempo | 3.1.0 (medium) | **AGPL-3.0** | Monolithic only (distributed needs Kafka) |
| | OpenTelemetry Collector / Operator | 0.162.0 / 0.160.0 | Apache-2.0 | Curated distribution for air-gap |
| Apache-2.0 alternative | VictoriaMetrics / VictoriaLogs | 1.153.0 / 1.53.0 | Apache-2.0 | "Apache-2.0 observability" preset |
| Storage | Longhorn | 1.13.0 | Apache-2.0 | Needs K8s ≥ 1.34 |
| | Rook-Ceph | 1.20.8 | Apache-2.0 | K8s 1.31–1.37 |
| | csi-driver-nfs | 4.13.4 | Apache-2.0 | |
| | local-path-provisioner | 0.0.37 | Apache-2.0 | No Helm repo upstream → we host the chart |
| | OpenEBS | 4.6.1 (medium) | Apache-2.0 | Candidate |
| Object storage | **MinIO not bundled** (community edition archived); Rook-Ceph RGW, SeaweedFS, external S3 | — | — | |
| Backup | Velero | 1.18.4 | Apache-2.0 | Kopia uploader, CSI data mover; CNCF Sandbox |
| GitOps | Argo CD | 3.5.3 | Apache-2.0 | Bundled Redis 8 replaced by Valkey in our values |
| | Flux | 2.9.6 | Apache-2.0 | Install via manifests/community chart (Flux Operator is AGPL) |
| Policy | Kyverno | 1.19.1 | Apache-2.0 | Use CEL policy types (ClusterPolicy deprecated) |
| | OPA Gatekeeper | 3.23.1 | Apache-2.0 | Alternative |
| Runtime security | Falco | 0.45.0 | Apache-2.0 | Keep falcosidekick-ui (redis-stack) disabled |
| Vulnerabilities | Trivy Operator | 0.35.0 | Apache-2.0 | Pin by digest (Trivy ecosystem compromise in March 2026) |
| GPU | NVIDIA GPU Operator / device plugin | 26.7.1 / 0.20.1 | Apache-2.0 (+ NVIDIA driver licenses) | DRA (GA since K8s 1.34) as opt-in profile |
| Secrets in clusters | External Secrets Operator | 2.12.0 | Apache-2.0 | Only latest minor supported (~3-week cadence) |
| | Sealed Secrets | 0.40.0 | Apache-2.0 | Optional, simple air-gap choice |

## 4. Supply chain, CI and security tooling

{{SECURITY_SECTION}}

## 5. Notable facts that changed since many 2024–2025 assumptions

1. **ingress-nginx is retired** (final releases 2026-03-19, archived 2026-03-24). Gateway API is the path forward.
2. **Gateway API v1.5/v1.6** made TLSRoute, ListenerSet, TCPRoute and UDPRoute standard, and added a safe-upgrade admission policy.
3. **Kubernetes 1.34 is about to reach EOL** (2026-10-27). 1.37 is current, and kubeadm v1beta3 is gone.
4. **etcd 3.7** is the default in kubeadm 1.37, without binary rollback.
5. **cgroup v1 is effectively unsupported** (kubelet ≥ 1.35 refuses it).
6. **containerd** ships every 4 months with a yearly LTS (2.3), and 1.7 is EOL.
7. **Helm 4 is GA**, and Helm 3's security support ends 2027-02-10.
8. **Bitnami** free versioned images are gone (moved to `bitnamilegacy`).
9. **MinIO** community edition is archived. **Promtail** is EOL. Loki SSD is deprecated. Tempo 3 distributed needs Kafka.
10. **Redis 8** is tri-licensed (RSAL/SSPL/AGPL). **Valkey 9** is the BSD-licensed fork. NATS remained Apache-2.0.
11. **OpenAPI 3.1** server codegen in Go is still partial, so we use Huma code-first with a committed spec.
12. **Go 1.27** is current (json/v2 under `encoding/json`), and the OpenTelemetry Go logs SDK is stable.
13. The **Trivy ecosystem compromise (March 2026)** shows that scanners and CI actions themselves must be pinned by digest/SHA.

## 6. How this baseline is maintained

- The catalog CI job checks upstream sources weekly (Kubernetes release markers, Go proxy, chart registries) and opens update PRs with compatibility test results.
- Library upgrades: Renovate PRs, grouped by ecosystem. `k8s.io/*` + Helm + controller-runtime move together after each Kubernetes minor.
- This document gets a dated "baseline refresh" section at each phase gate.
