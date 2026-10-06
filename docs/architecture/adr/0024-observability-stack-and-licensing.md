# ADR-0024: Observability add-on defaults and handling of AGPL/source-available components

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0024-observability-stack-and-licensing.ru.md)

## Context

The spec asks for metrics (Prometheus, Grafana, metrics-server), logging (Loki, Fluent Bit), tracing (OpenTelemetry, Tempo), Kubernetes monitoring (kube-state-metrics, node-exporter), presets (Basic, Standard, Full) and Grafana URL/credentials with dashboards after deployment (prompt §22–23). Research (2026-10-06):
- **kube-prometheus-stack** chart 91.x (Prometheus Operator 0.94, Prometheus 3.15) is Apache-2.0. Its Grafana subchart now comes from `grafana-community` charts.
- **Grafana 13.2**, **Loki 3.7** and **Tempo 3.1** are **AGPL-3.0**. Their OSS charts moved to `grafana-community/helm-charts`.
- **Promtail is EOL** (2026-03-02). Loki's Simple Scalable mode is deprecated (removed in 4.0).
- Tempo 3.x microservices mode needs Kafka.
- **Grafana Alloy** (Apache-2.0), **Fluent Bit 5.1** and the **OpenTelemetry Collector/Operator** are Apache-2.0.
- **VictoriaMetrics/VictoriaLogs** are Apache-2.0 alternatives.
- Also relevant: Argo CD bundles Redis 8 (RSAL/SSPL/AGPL), falcosidekick-ui uses redis-stack, and Flux Operator is AGPL.

## Decision

- **Presets**:
  - **Basic**: metrics-server only.
  - **Standard**: kube-prometheus-stack (Prometheus, Alertmanager, kube-state-metrics, node-exporter, default dashboards and alerts) + Grafana + Loki (monolithic) + Alloy as the log agent.
  - **Full**: Standard + Tempo (monolithic) + OpenTelemetry Collector (gateway) + OpenTelemetry Operator (optional auto-instrumentation).
- **Grafana hand-off**: after deployment, the user gets the Grafana URL (through the cluster's Gateway with a cert-manager certificate), the admin username, and a one-time retrievable password stored as a secret, or SSO configuration when available. Dashboards are provisioned automatically (cluster, nodes, add-ons, Cilium/Hubble, Longhorn).
- **Licensing rules** for AGPL and source-available components (Grafana, Loki, Tempo, Flux Operator, Redis 8 inside charts):
  1. They are installed **unmodified** from upstream artifacts into **user clusters** as separate programs. We never link, embed or modify their code in core or ee, and we never embed Grafana UI in our product UI.
  2. Air-gapped bundles that redistribute them include license texts and a source offer. Legal review happens before enterprise packaging.
  3. Apache-2.0 alternatives exist in the catalog: VictoriaMetrics k8s-stack and VictoriaLogs (backends), Alloy, Fluent Bit or OTel Collector (agents).
  4. In bundled chart values, Redis is replaced by **Valkey** where possible (Argo CD), and source-available sub-components (falcosidekick-ui redis-stack) stay disabled.
  5. Add-on CI runs an SBOM and license scan of every image, and the policy fails on unexpected licenses.
- **Not offered**: Promtail (EOL), Loki SSD mode, distributed Tempo (Kafka dependency), MinIO as the bundled object store.
- **Platform self-observability** (Prometheus metrics, OTel traces, slog logs) is independent of these add-ons (see the deployment model).

## Alternatives considered

- **VictoriaMetrics/VictoriaLogs as the default**: fully Apache-2.0 and resource-efficient. However, the Prometheus Operator/Grafana/Loki stack is what most users expect and know. It is offered as a first-class alternative ("Apache-2.0 observability" preset).
- **Bundle Grafana into the product UI**: AGPL obligations and coupling; rejected.
- **Fluent Bit as the default agent**: also good (Apache-2.0). Alloy has first-class Loki/OTel/Prometheus support and a promtail config converter.

## Consequences

- Positive: familiar, well-supported defaults with clear license boundaries, and an Apache-2.0 path for licence-sensitive customers.
- Negative: we track the fast-moving chart relocations (grafana → grafana-community) and Tempo/Loki architecture changes in the catalog.

## References

- kube-prometheus-stack, grafana-community/helm-charts, Loki deployment modes, Promtail EOL notice, Tempo 3.0 notes, Alloy, VictoriaMetrics docs; license files of the components
