# ADR-0023: Storage defaults

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0023-storage-defaults.ru.md)

## Context

The spec requires a storage abstraction with local-path, NFS, Longhorn, Ceph and cloud CSI, and a suitable default for the production preset (prompt §21). Research (2026-10-06):
- **local-path-provisioner** v0.0.37 publishes no Helm repo.
- **csi-driver-nfs** is at v4.13.x.
- **Longhorn 1.13.0** (2026-09-29) requires **Kubernetes ≥ 1.34**, and its V2 (SPDK) data engine is GA with hugepage/kernel requirements.
- **Rook-Ceph 1.20.8** supports Kubernetes 1.31–1.37.
- **OpenEBS 4.6** has LocalPV and Mayastor.
- **MinIO community is archived and unmaintained**, so it is not suitable as a bundled S3.

## Decision

| Context | Default | Alternatives |
|---|---|---|
| Development, Minimal, Edge | **local-path** (we host its chart), default StorageClass | — |
| Production on bare metal with ≥ 3 workers | **Longhorn** (V1 data engine, replicas 3, default StorageClass, backup target = cluster backup destination) | Rook-Ceph for large/high-IO or when the user has Ceph skills; NFS CSI for existing NAS |
| Production on bare metal with < 3 workers | Longhorn with replicas = workers, plus a WARNING (reduced redundancy), or local-path plus a WARNING (no replication) | NFS |
| AI/ML, High Storage IO | Rook-Ceph (block + RGW object storage) | Longhorn V2 engine (opt-in, kernel/hugepages checks) |
| Clouds | Provider CSI (e.g. hcloud CSI for RWO volumes) | Longhorn on cloud disks |
| Object storage needed in-cluster (Loki, Tempo, Velero) | External S3-compatible endpoint (recommended), or **Rook-Ceph RGW** | SeaweedFS (Apache-2.0); **MinIO not bundled** (archived) |

- The compatibility engine enforces add-on requirements: Longhorn ≥ 3 schedulable nodes for replica 3, Longhorn 1.13 → K8s ≥ 1.34, the iSCSI/NFS client packages installed by the bootstrap layer, disk space minimums, and exactly one default StorageClass.
- Health probes include an optional PVC smoke test during `health.verify`.

## Alternatives considered

- **OpenEBS as default**: a capable alternative (Apache-2.0); kept as a catalog candidate. Longhorn has a simpler UX and a built-in backup integration.
- **Ceph by default**: powerful but heavy for small clusters (resource and operations cost).
- **MinIO for in-cluster S3**: archived upstream, with an unpatched CVE noted by Grafana. Not acceptable.

## Consequences

- Positive: sensible defaults per size and environment; the risky "no replication in production" choices are explicit.
- Negative: several storage add-ons to test (Tier-3 real disks are needed for meaningful Longhorn/Ceph tests).

## References

- Longhorn 1.13 release notes and chart; Rook 1.20 prerequisites; csi-driver-nfs; local-path-provisioner; MinIO repository status
