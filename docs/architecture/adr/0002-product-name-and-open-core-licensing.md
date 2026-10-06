# ADR-0002: Product name, open-core licensing and editions

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0002-product-name-and-open-core-licensing.ru.md)

## Context

The owner asked the architect to choose an interesting name that does not yet exist on the market (Phase 0 answer 2), and chose an **open-core** model: Apache-2.0 core plus a commercial `ee/` directory (answer 3a). The spec names the CLI `clusterctl` (prompt §56) and `platformctl` (prompt §197). `clusterctl` is already the CLI of the Kubernetes **Cluster API** project, so using it would cause confusion and installation conflicts. The spec also requires two logical editions, Community/Standard and Enterprise. Enterprise must not be "a feature flag over chaotic code", and entitlements must be centralized (prompt §133, §218–219).

## Decision

### Name: **Farvater**

- **Meaning.** *Farvater* (Russian «фарватер», from Dutch *vaarwater*) is the fairway: the safe, marked navigable channel that ships follow through difficult waters into port. That is exactly the product's promise:
  - it guides users from bare infrastructure to a production-ready cluster along a proven, safe channel;
  - the version catalog and the compatibility rules are the "buoys" marking where it is safe to go;
  - it fits the nautical vocabulary of Kubernetes (κυβερνήτης, "helmsman"), Helm and harbor.
- **Works in both languages.** The word is common in Russian and easy to pronounce in English ("FAR-vah-ter"). It has no unfortunate meanings in either language.
- **Availability check (2026-10-06).** Web searches found no software product, open-source project or company in the DevOps/Kubernetes/cloud space using the name. Candidates rejected because the space already uses them:

| Candidate | Conflict found |
|---|---|
| Nauarch / Navarch | *Nauarchos* (Kubernetes control plane for agent fleets), *Navarch* (multi-cloud GPU fleet provisioning), *Navarcos* (Kubernetes CaaS/PaaS manager on Cluster API) |
| Slipway | *Slipway* Kubernetes GitOps controller; slipwayhq images |
| Stapel | Image build syntax in **werf** (a Kubernetes CI/CD tool) |
| Werf («верфь») | **werf** itself |
| Keelson | No direct hit, but too close to **Keel** (keel.sh, Kubernetes update automation) |
| Shipwright, Flotilla, Armada, Helmsman, Tiller, Drydock | Existing CNCF/OSS projects |
| Lotsman, Kormchiy, Rumb | Free, but awkward for English speakers or with weak associations |

  A formal trademark search and domain registration are **action items for the owner** before public launch. Until then `farvater.io` is used as the API group and docs domain placeholder.
- **Identifiers**:

| Item | Value |
|---|---|
| Product | Farvater |
| Server binary | `farvater-server` (subcommands: `api`, `worker`, `all`, `migrate`, `admin`, `version`) |
| CLI | `farvater` (users can alias it) |
| ClusterSpec API group | `farvater.io/v1alpha1` |
| Go module | `github.com/faustengreat777-beep/k8s-test` until the repository is renamed (recommended: `farvater`) |
| API key prefix | `fvt_` |
| Container images | `…/farvater-server`, `…/farvater-node-helper` |
| Metrics prefix | `farvater_` |
| Helm chart | `farvater` |

### Licensing and editions

- **Community edition**: everything outside `ee/`, licensed **Apache-2.0** (`LICENSE` at the root). It is a complete, production-grade provisioning and lifecycle product: all providers, distributions, add-ons, Auto/Simple/Advanced modes, upgrades, scaling, backup/restore, health, templates, API, CLI, webhooks, audit log.
- **Enterprise edition**: code in `ee/` under a **separate commercial license** (`ee/LICENSE`). The license text must be drafted by legal counsel before the first `ee/` code lands. Until then `ee/` doesn't exist. Enterprise code compiles only with the `ee` build tag and plugs into documented extension points (identity providers, authorizer, policy evaluator, approval gate, audit sinks, key providers, fleet controller).
- **Open-core boundary rule**: Enterprise adds governance, scale and integrations: SSO/SCIM/MFA policy, custom roles/ABAC, policy engine, approvals, compliance, fleet, KMS/BYOK, air-gap bundles, HA reference architecture, cost management, incidents. It **never removes or cripples** core provisioning capabilities, and it never gates security fixes.
- **Entitlements**: a single `EntitlementService` verifies an **Ed25519-signed license file** offline (edition, features, limits, expiry). Code asks `Has(feature)`, never `if edition == "enterprise"`. Feature keys: `enterprise.sso`, `enterprise.scim`, `enterprise.rbac.custom`, `enterprise.policy`, `enterprise.approvals`, `enterprise.audit.export`, `enterprise.compliance`, `enterprise.fleet`, `enterprise.kms`, `enterprise.airgap`, … Feature flags for gradual rollout are a separate mechanism and never replace architecture (prompt §219).
- **Public commitments** (trust is a differentiator after KubeSphere's 2025 licence change, Omni's BSL and Kamaji's paywalled stable releases): the core will not be relicensed away from Apache-2.0, stable Community releases and security fixes are free, and the edition boundary is documented in [PRODUCT §5](../../PRODUCT.md).
- **Contributions**: Developer Certificate of Origin (DCO) sign-off for the Apache-2.0 core, with no CLA required for core contributions. `ee/` contributions are by the vendor only.
- **Third-party licenses**: no GPL/AGPL/SSPL/BSL code is linked into our binaries (CI license gate). AGPL components (Grafana, Loki, Tempo) are only installed unmodified into user clusters (ADR-0024).

## Alternatives considered

- **Keep `clusterctl`/`platformctl`**: name clash with Cluster API, and too generic to be a brand.
- **Fully proprietary**: limits adoption and trust for an infrastructure tool that holds root credentials.
- **AGPL core**: protects against SaaS free-riding, but many enterprises ban AGPL. Apache-2.0 maximizes adoption, and enterprise value lives in `ee/`.
- **BSL/source-available core**: community backlash (cf. HashiCorp → OpenTofu/OpenBao forks). Not aligned with the owner's choice.
- **Separate private repository for ee**: cleaner IP separation, but cross-repo changes and CI get harder. One repository with a separate directory license is the established pattern (GitLab, Grafana, Mattermost).

## Consequences

- Positive: a distinctive, meaningful name that works in both target languages; a clear open-core contract that builds community trust; a single place for edition logic.
- Negative: trademark and domain must be secured. The commercial license needs legal work. The repository should be renamed before code exists, otherwise module-path churn follows.

## References

- Cluster API `clusterctl`; Apache License 2.0; DCO 1.1; open-core precedents (GitLab EE, Grafana Enterprise, Mattermost)
