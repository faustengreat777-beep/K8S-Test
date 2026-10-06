# ADR-0022: Load balancing and control-plane endpoint

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0022-load-balancing-and-control-plane-endpoint.ru.md)

## Context

HA clusters need a stable control-plane endpoint, and `Service type=LoadBalancer` needs an implementation (prompt §20: MetalLB, cloud LB, external LB, NodePort; MetalLB suggested for bare metal). Research (2026-10-06): MetalLB 0.16.1 is maintained but still 0.x. FRR-K8s became its default BGP backend and classic FRR mode is deprecated. kube-vip 1.2.x is the common control-plane VIP for kubeadm. As a static pod it needs `super-admin.conf` on the first control-plane node during `kubeadm init` (≥ 1.29), then `admin.conf`. Cilium LB-IPAM and the BGP control plane are stable, but Cilium L2 announcements are still beta. On Hetzner Cloud (layer-3 networks), ARP-based VIPs (MetalLB L2, kube-vip ARP, keepalived) do not work. There, the Hetzner Load Balancer via hcloud-cloud-controller-manager, or Primary/Floating IPs moved through the API, must be used.

## Decision

- **Control-plane endpoint** (chosen by the decision engine from provider capabilities and the declared network):
  - bare metal / VPS with L2 adjacency → **kube-vip** static pod (ARP), written before `kubeadm init --control-plane-endpoint`, using the super-admin.conf → admin.conf switch on the first node;
  - BGP-capable networks → kube-vip in BGP mode;
  - **external load balancer** declared by the user (existing HAProxy/F5/etc.) → use its address;
  - cloud providers → provider load balancer (Hetzner LB, AWS NLB, …);
  - single control plane → node address (no HA, with a WARNING in production).
- **Service load balancer**:
  - bare metal with L2 → **MetalLB** (L2 mode by default; BGP via FRR-K8s when peers are given);
  - Cilium clusters with BGP peers → **Cilium LB-IPAM + BGP control plane** as the preferred alternative. Cilium L2 announcements are offered only as "beta" and are not chosen by Auto Mode for production;
  - clouds → provider CCM load balancers (e.g. hcloud-cloud-controller-manager ≥ 1.38; versions ≤ 1.30.0 are blocked in the catalog);
  - development without LB → NodePort.
- The compatibility engine **blocks** MetalLB L2 and kube-vip ARP on providers flagged as L3-only (Hetzner Cloud), with a remediation pointing at the provider LB.

## Alternatives considered

- **keepalived + HAProxy on control-plane nodes**: proven, but more moving parts to configure and upgrade. Possible later as an "external LB on nodes" option.
- **Cilium L2 announcements as the default**: fewer components on Cilium clusters, but beta status and extra API-server load (one lease per service).
- **MetalLB everywhere**: fails on L3 clouds.

## Consequences

- Positive: correct defaults per environment, and known traps blocked by rules.
- Negative: several LB implementations to maintain. Each needs E2E coverage (L2 tests need Tier-3 or a special network setup).

## References

- MetalLB 0.16 release notes and cloud compatibility page; kube-vip static pod documentation; Cilium 1.20 LB-IPAM, BGP, L2 announcement docs; Hetzner HCCM docs
