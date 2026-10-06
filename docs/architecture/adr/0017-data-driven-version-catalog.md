# ADR-0017: Data-driven, signed version catalog decoupled from releases

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0017-data-driven-version-catalog.ru.md)

## Context

Versions must not be hard-coded across the project (prompt §49). The compatibility engine must prevent known-broken combinations (prompt §48). Phase 0 research confirmed fast churn: three Kubernetes minors a year with monthly patches, containerd minors every 4 months with yearly LTS, etcd 3.7 as the new default, Gateway API v1.5 and v1.6 graduating several kinds, Cilium supporting the last 3 minors with explicit Kubernetes test ranges, and Envoy Gateway minors supported about 6 months. Air-gapped installs need exact image and chart lists with digests.

## Decision

- All version and compatibility knowledge lives in **`catalog/`** as YAML: Kubernetes minors (status, patches, EOL, component pins), distributions, runtimes, OS matrix, add-ons (versions, chart refs **with digests**, images **with digests**, Kubernetes ranges, capabilities, ports, defaults), declarative compatibility rules, presets, and duration estimates.
- The catalog is **embedded** in each release and can be updated independently through a **signed catalog bundle** (Ed25519, monotonic version, channels `stable`/`lts`/`beta`). An invalid or unsigned bundle is rejected, and the previous catalog stays active.
- **Status semantics**: `current/recommended`, `supported`, `untested` combinations (WARNING), `deprecated` (not offered for new clusters), `blocked` (never selectable).
- The catalog is **validated at load**: schema, references, every entry has an implementation in the plugin registry, acyclic capability graph, well-formed ranges.
- **Automation**: CI bots propose catalog updates (Kubernetes markers from `dl.k8s.io/release/stable-1.x.txt` and the release schedule, chart versions from registries, image lists from rendered charts). Compatibility tests gate the merge.
- A ResolvedSpec records the catalog version used, so plans are reproducible.

## Alternatives considered

- **Versions as Go constants**: every patch release needs a platform release, and versions get scattered and drift.
- **Fetching versions live from upstream at plan time**: not reproducible, breaks air-gap, and exposes the platform to upstream outages and tampering.
- **Unsigned remote catalog**: a supply-chain risk (T12/T15 in the threat model).

## Consequences

- Positive: fast reaction to upstream releases and CVEs, reproducible plans, air-gap-ready image lists, a single source of truth for UI options, the compatibility engine and the decision engine.
- Negative: the catalog is a product in itself, with maintenance effort for curation and tests, and signing key management is required.

## References

- [Auto Mode, catalog and compatibility](../auto-mode-and-catalog.md), [Technology stack — version baseline](../technology-stack.md)
