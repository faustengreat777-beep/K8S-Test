# ADR-0021: Kubernetes version policy and node runtime defaults

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0021-kubernetes-version-and-runtime-defaults.ru.md)

## Context

The spec's examples use Kubernetes 1.34 (prompt §34, §120). As of 2026-10-06, research shows: 1.37.1 is the latest stable release (1.37.0 on 2026-08-26), 1.36.5 and 1.35.9 are supported, and **1.34 reaches EOL on 2026-10-27**. 1.38 is planned for 2026-12-16. Add-ons lag: Cilium 1.20 is tested on 1.33–1.36, and Envoy Gateway 1.9 on 1.33–1.36. kubeadm 1.37 removed the v1beta3 config API (only v1beta4 is usable), defaults to etcd 3.7.0 (v2 store removed, no binary rollback), and kubelets ≥ 1.35 refuse cgroup v1. kube-proxy IPVS is deprecated (removal in 1.43), and nftables mode has been GA since 1.33. containerd moved to a 4-month cadence with yearly LTS (2.3 LTS until 2028-04-30), and 1.38 kubelets need containerd 2.x (RuntimeConfig RPC).

## Decision

- **Supported Kubernetes minors at launch**: 1.35, 1.36, 1.37 (catalog-driven).
  - **Default for new clusters: 1.36** (the newest minor with full add-on compatibility today; matches the k3s/RKE2 `stable` channel).
  - **1.37 offered as "latest"**. Combinations untested upstream (e.g. Cilium 1.20 on 1.37) are allowed with a WARNING, until the add-ons qualify it.
  - 1.34 is not offered for new clusters (EOL 2026-10-27). 1.38 is to be qualified within about 4–6 weeks of its release.
- **Upgrades**: one minor at a time, control plane before workers, drain before a kubelet minor upgrade, `kubeadm upgrade plan` plus an etcd snapshot before each control-plane step. **etcd guard**: before moving etcd 3.6 → 3.7, every member must be on ≥ 3.6.11; the upgrade advisor enforces this.
- **kubeadm config**: generate only `kubeadm.k8s.io/v1beta4`, with a converter layer ready for the upcoming `v1`. Always use the kubeadm binary of the target minor.
- **Container runtime**: upstream **containerd 2.3.x LTS** by default (2.4.x opt-in), pinned in the catalog together with runc 1.5.x, CNI plugins and crictl. Distro containerd packages are not used (too old on Ubuntu, Debian and EL). Use the static build where glibc < 2.35 (EL9). Config uses `version = 3` (compatible with 2.0–2.4). Registry mirrors use `config_path`/`hosts.toml`. `SystemdCgroup = true`.
- **cgroup v2 is required**. There is no `failCgroupV1=false` escape hatch.
- **kube-proxy**: always write the mode explicitly. Use `nftables` (kernel ≥ 5.13 on all supported OS), or none when Cilium's kube-proxy replacement is on. IPVS is never offered.
- **Swap**: disabled by default. NodeSwap (GA) can be configured explicitly in Advanced Mode.
- **Node OS at launch**: Ubuntu 24.04 and 26.04 LTS, Debian 13 (Debian 12 best-effort, since it has been in LTS since 2026-06). Then Rocky Linux and AlmaLinux 9.x/10.x (Phase 7, SELinux enforcing with container-selinux; Rocky 10 needs x86-64-v3 CPUs).
- **Control-plane endpoint**: see ADR-0022.

## Alternatives considered

- **Default to the latest minor (1.37)**: produces add-on combinations that upstream hasn't tested (Cilium, Envoy Gateway). This is risky for "production-ready by default".
- **Use distro packages for containerd**: outdated versions, inconsistent across OSes, and some are EOL.
- **IPVS for performance**: deprecated upstream, and Cilium KPR or nftables cover the use cases.

## Consequences

- Positive: safe defaults aligned with upstream support and add-on test matrices. Explicit guards for known upgrade traps (etcd 3.7, SELinuxMount in 1.37, cgroup driver fallback removal in 1.38).
- Negative: the catalog needs monthly maintenance (patch releases). The default minor moves on a schedule (about every 4 months), which needs a documented process.

## References

- kubernetes.io releases and version-skew policy, kubeadm CHANGELOG 1.35–1.37, containerd RELEASES.md, etcd upgrade guides 3.6/3.7
- [Technology stack — version baseline](../technology-stack.md)
