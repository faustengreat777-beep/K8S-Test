# ADR-0020: Networking defaults — Cilium as default CNI; Gateway API as the only ingress model with platform-owned CRDs; no ingress-nginx

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0020-networking-defaults-gateway-api.ru.md)

## Context

The spec asks for a CNI choice (Cilium, Calico, Flannel, with Cilium features: kube-proxy replacement, Hubble, NetworkPolicies, encryption, load balancing; prompt §17–18). It also asks for an ingress choice (NGINX Ingress, Traefik, HAProxy, Gateway API implementations; prompt §19). The owner decided in Phase 0 to default to **Gateway API**. Research (2026-10-06) established:
- **ingress-nginx is retired.** The final releases were on 2026-03-19, the repository was archived on 2026-03-24, and there are no further fixes, including CVEs. Its successor InGate was retired as well. Kubernetes SIG Network and the Steering Committee recommend Gateway API implementations.
- **Gateway API v1.6.2** is current. HTTPRoute, GRPCRoute, TLSRoute, ListenerSet, BackendTLSPolicy, TCPRoute and UDPRoute are in the standard channel. A `safe-upgrades` ValidatingAdmissionPolicy (since v1.5) blocks CRD downgrades and experimental-over-standard installs. Moving TLSRoute objects stored as v1alpha2 to the v1.6 standard CRD needs a storage-version migration.
- **Cilium 1.20.2** is current, with Kubernetes tested range 1.33–1.36. Its Gateway API passes v1.6 conformance (HTTP, GRPC, TLS, TCP, UDP, Mesh) and needs Gateway API CRDs ≥ v1.6.1 installed first. Envoy runs as a shared DaemonSet, so there are no per-Gateway proxies.
- **Envoy Gateway 1.9.x** (Kubernetes 1.33–1.36, Gateway API v1.6.1) passes conformance, with each minor supported about 6 months. NGINX Gateway Fabric 2.7.x has full 5-profile conformance. Traefik 3.7 has partial HTTP core conformance. Contour is stuck on Gateway API v1.3. HAProxy Unified Gateway is new and has no conformance report.
- **Calico 3.33** (Kubernetes 1.35–1.37) ships "Calico Ingress Gateway" (an Envoy Gateway build) and bundles Gateway API CRDs. **Flannel 0.28.x** has no NetworkPolicy of its own.

## Decision

1. **Default CNI: Cilium** (kube-proxy replacement on, Hubble on). It is the alternative for clusters where Calico is preferred (BGP-centric networks), and Flannel (+ kube-network-policies) is offered for minimal/edge profiles.
2. **Gateway API is the only first-class ingress model.** No `Ingress`-only controllers are offered for new clusters. **ingress-nginx is never offered.** Imported clusters running it are flagged as a security liability, and a migration assistant based on `kubernetes-sigs/ingress2gateway` is planned (Phase 6).
3. **The platform owns the Gateway API CRDs** (`addon/gateway-api-crds`, standard channel, one pinned version per catalog release, server-side applied, never downgraded, one minor at a time, with a TLSRoute storage-version migration guard). CRD installation is disabled in implementation charts, and bundled CRDs are disabled in k3s/RKE2 (`rke2-gateway-api-crd`).
4. **Default Gateway implementation**: **Cilium Gateway** when the CNI is Cilium, otherwise **Envoy Gateway**. Optional: NGINX Gateway Fabric, Traefik, Istio (gateway-only). The implementation is swappable per cluster, and the GatewayClass name is a parameter.
5. Feature maturity labels come from the catalog (e.g. Cilium L2 announcements = beta, WireGuard node-to-node = beta). Beta features are not enabled by Auto Mode for production.

## Alternatives considered

- **Keep NGINX Ingress as the default** (as the spec suggested): the upstream project is archived with no CVE fixes, so this is unacceptable.
- **F5 NGINX Ingress Controller** (still maintained, `Ingress` API): a vendor product. Ingress is frozen upstream, and the spec's direction is Gateway API. NGINX users get NGINX Gateway Fabric instead.
- **Envoy Gateway as the default everywhere**: portable, but it adds per-Gateway proxies and a controller next to Cilium, whose built-in Gateway is already conformant.
- **Let each implementation install its CRDs**: multiple owners plus safe-upgrade VAPs can block upgrades or silently pin versions.

## Consequences

- Positive: a secure, future-proof default and fewer components on Cilium clusters, with a consistent API across implementations.
- Negative: some users expect `Ingress`. The UI and docs explain Gateway API, and the migration assistant helps. Gateway/CNI upgrade cadence (Envoy Gateway ~6-month support) has to be automated.

## References

- Kubernetes blog: "Ingress NGINX Retirement: What You Need to Know" (2025-11-11); Steering Committee statement (2026-01-29)
- Gateway API v1.5 and v1.6 changelogs; conformance reports v1.6
- Cilium 1.20 docs (requirements, Gateway API, upgrade notes); Envoy Gateway compatibility matrix
- [Technology stack — version baseline](../technology-stack.md)
