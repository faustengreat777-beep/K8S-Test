# ADR-0010: Compile-time plugin model; declarative add-ons installed through an embedded Helm 4 SDK

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0010-plugin-model-and-helm.ru.md)

## Context

Providers, distributions and add-ons must plug in through interfaces (prompt §99). New add-ons must not require changing the core engine (prompt §50). Helm must be used through a single service, not chaotically through the shell (prompt §51). Research (2026-10-06):
- **Helm 4** has been GA since November 2025 and v4.3.0 is current. **Helm 3** got its last feature release (v3.22.0) on 2026-09-09, and its security fixes end on **2027-02-10**.
- The Helm 4 SDK (`helm.sh/helm/v4/pkg/...`) defaults installs to **server-side apply**, has kstatus-based waits (`WaitStrategy: watcher`), renames atomic to `RollbackOnFailure`, and keeps an in-process `PostRenderer` interface. Chart API v3 is still experimental and internal.
- Helm 4.3 pins `k8s.io/*` v0.37 and needs Go ≥ 1.26.
- Bitnami charts and images are no longer usable for reproducible or air-gapped installs. local-path-provisioner publishes no Helm repository.

## Decision

- **Plugins are compiled-in Go packages** implementing `pkg/sdk` interfaces (`InfrastructureProvider`, `KubernetesDistribution`, `OSFamily`, `Addon` plus the capability interfaces `CNI`, `GatewayProvider`, `LoadBalancerProvider`, `StorageProvider`). They register in a registry at start-up, and the registry is validated against the catalog. There is no runtime code loading. Out-of-process (gRPC) plugins may come later behind the same interfaces.
- **Add-ons are declarative by default**: `addon.yaml` (metadata, capabilities, ports, config JSON Schema, health), a values template rendered from typed config (never from user strings), and optional Go hooks. One generic implementation installs them.
- **One Helm service over the embedded Helm v4 SDK**:
  - install/upgrade with explicit `ServerSideApply` per release;
  - `WaitStrategy=watcher` plus `RollbackOnFailure` for reversible add-ons;
  - `Template`/`Diff` for plans;
  - Helm logs bridged into our `slog`;
  - in-process post-rendering where needed;
  - release state stays in the target cluster (Secrets driver, so a break-glass `helm` CLI still works) and is mirrored to our DB;
  - no Helm v3 SDK anywhere.
- **Chart sourcing rules**: only catalog-pinned chart references **with digests**, preferring upstream OCI registries (quay.io/jetstack, quay.io/cilium, ghcr.io/prometheus-community, ghcr.io/grafana-community, ghcr.io/argoproj, ghcr.io/kyverno, …). **No `bitnami/*` charts or images.** Where upstream has no chart (local-path-provisioner) or lags (Velero chart), we package and host the chart ourselves. First-party charts use `apiVersion: v2`.
- **Raw manifests** (Gateway API CRDs, Gateway/ClusterIssuer objects) are server-side applied with a dedicated field manager.
- The Helm and client-go versions move in lockstep with each Kubernetes minor (about three bumps a year).

## Alternatives considered

| Option | Why not |
|---|---|
| Shelling out to the `helm` CLI | Explicitly forbidden by the spec; no typed errors; injection surface |
| Helm v3 SDK | End of life (security fixes end 2027-02-10, no client-go for K8s 1.38+) |
| Go `plugin` / dynamic `.so` loading | Fragile ABI, platform-specific, unsafe for untrusted code |
| Kustomize / raw manifests only | Most upstream add-ons ship Helm charts; re-packaging all of them is unsustainable. Kustomize remains possible inside post-rendering |
| Flux/Argo CD as the add-on installer for every cluster | Needs GitOps components in every cluster from the first minute and adds a dependency to bootstrap. GitOps stays an optional add-on (Phase 8) |

## Consequences

- Positive: a single, typed, testable path for add-ons; reproducible and air-gap-friendly artifacts; new add-ons are mostly data.
- Negative: the Helm SDK pulls in a large dependency tree pinned to `k8s.io/*` versions. Mixed client-side/server-side apply ownership on releases first created by Helm 3 (imported clusters) needs care with `ForceConflicts`/`TakeOwnership`.

## References

- Helm 4 release blog, Helm 3 EOL blog (2026-06-02), Helm v4.3.0 sources; [Core interfaces](../core-interfaces.md)
