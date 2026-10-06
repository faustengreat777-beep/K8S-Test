# Threat model

> Status: **Proposed** (Phase 0), to be reviewed at every phase gate. Language: English · [Русский](THREAT_MODEL.ru.md)
>
> Related: [Security model](SECURITY_MODEL.md) · [ARCHITECTURE](../ARCHITECTURE.md) · [Provisioning engine](../architecture/provisioning-engine.md)

Method: we list assets, actors and trust boundaries, then walk through **STRIDE** (Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege) for each component and for the 15 threat scenarios required by the product spec (prompt §224). Each threat has a **likelihood/impact rating** (L/M/H), the **mitigations** planned, and the **phase** in which they must exist. A threat is "covered" only when its mitigation has tests.

## 1. Assets

| # | Asset | Why it matters | Where |
|---|---|---|---|
| A1 | Infrastructure credentials (SSH private keys, cloud tokens, S3 keys, registry and Git credentials) | Full control of customer machines and accounts | `secrets` (encrypted), worker memory (transient) |
| A2 | Cluster admin credentials (admin kubeconfig, CA keys, join tokens, etcd snapshots) | Full control of customer clusters and their data | `secrets`, control-plane nodes, backup targets |
| A3 | Key-encryption keys (KEK) and data keys (DEK) | Decrypt A1/A2 | KEK: worker key file / KMS; DEK: wrapped in `data_keys` |
| A4 | Platform accounts, sessions, API keys | Act as a user | `users`, `sessions`, `api_keys` (hashed) |
| A5 | Tenant data: cluster specs, inventory, logs, audit | Confidentiality between tenants; operational intelligence for attackers | PostgreSQL |
| A6 | Audit log integrity | Accountability, compliance evidence | `audit_events` |
| A7 | Platform availability and the provisioning pipeline | Users depend on it for upgrades and recovery | API, workers, DB |
| A8 | Software supply chain (our binaries/images, catalog: charts, images, packages) | A malicious artifact reaches every managed cluster | CI, registries, catalog bundles |

## 2. Actors

| Actor | Capability |
|---|---|
| Anonymous internet attacker | Reaches the public API/UI endpoint |
| Authenticated low-privilege user (e.g. Viewer, Developer) | Valid session/API key in one tenant |
| Malicious tenant admin | Full control inside their own organization; wants another tenant's data or the platform |
| Compromised user/API key | Same as the owner's rights until revoked |
| Malicious or compromised managed node / cluster | Controls responses to SSH commands and Kubernetes API calls coming from the platform |
| Malicious upstream (Helm repo, image registry, Git repo, add-on author) | Controls content fetched by the platform |
| Insider with infrastructure access | Access to DB backups, logs or hosts of the platform |
| Network attacker | On path between platform and nodes, or between users and platform |

## 3. Trust boundaries

See [Security model §2](SECURITY_MODEL.md#2-trust-boundaries): **A** clients → API, **B** app ↔ database, **C** worker (holds decrypted credentials transiently), **D** platform → customer infrastructure and external services. Plus the **E** supply-chain boundary: CI and catalog → released artifacts.

## 4. Threat scenarios required by the spec (prompt §224)

Ratings: **L**ikelihood / **I**mpact (Low, Medium, High). Phase = the phase in which the mitigation must be implemented and tested.

| # | Threat | STRIDE | L / I | Mitigations | Phase |
|---|---|---|---|---|---|
| T1 | **Compromised user account** (phished password, stolen session) | S, E | M / H | argon2id, breached-password check, login backoff and lock, session rotation, idle + absolute timeouts, session list and revoke-all, audit of logins with IP/UA, MFA and SSO (ee); sensitive actions (credential use, destroy, production upgrade) need recent auth or approval (ee); least-privilege default roles | 1 (base), 9 (MFA/SSO) |
| T2 | **Compromised API key** (leaked in CI logs or a repo) | S, E | H / H | Keys are prefixed (detectable by secret scanners), hashed at rest, scoped, expiring (default 90 d), revocable, IP of last use shown; anomaly alerts (new IP/ASN) later; keys can't exceed the owner's rights and lose them when the owner does; secret-scanning partner revocation later | 1, 9 |
| T3 | **Malicious cluster config** (spec crafted to escalate: weird CIDRs, paths, versions, kubeadm patches, huge node lists) | T, E, D | M / H | Strict schema (unknown fields rejected), semantic validators, versions only from the catalog, "advanced" patches limited to an allow-list of config fields, resource limits (max nodes per cluster, max add-ons), policy/guardrails before plan; everything rendered from typed structs, never concatenated | 1–3 |
| T4 | **Malicious YAML** (billion laughs, deep nesting, huge documents, type confusion) | D, T | M / M | Size limit (1 MiB default), alias/anchor expansion limit, depth limit, strict decoding into typed structs, JSON Schema validation, fuzz tests on the parser path | 1 |
| T5 | **Command injection** (user data reaching a shell on a node or on the platform) | E, T | M / H | No `sh -c` with user data; argv-based `Command` with POSIX quoting; validators for every field type; config files via SFTP from typed structs; optional signed node helper with JSON stdin; custom static analyzer forbidding unsafe patterns; fuzz tests of validators and the quoting function | 1 (lint), 3 |
| T6 | **SSRF** (webhook/Helm/Git/registry/OIDC URLs pointing at internal services or cloud metadata) | I, E | H / H | Central `httpx` client: resolve-then-dial with IP checks, blocks private, loopback, link-local and metadata ranges (allow-list configurable for self-hosted), redirect re-validation, scheme allow-list, timeouts, size limits; workers in a restricted egress zone; tests with DNS-rebinding and redirect tricks | 1 (webhooks), 4 (Helm/Git) |
| T7 | **Credential theft** (DB dump, backup leak, log leak, memory scraping, API response leak) | I | M / H | Envelope encryption with KEK outside the DB; AAD binding; API never returns secrets; redaction of logs/events/errors; canary-secret leak tests; just-in-time decryption only in workers; KEK backups separate from DB backups; KMS/BYOK (ee); audit of credential use | 1 |
| T8 | **Tenant escape** (reading or modifying another organization's clusters, logs, credentials, audit) | I, T, E | M / H | `TenantScope` required in every repository; app-layer authz on every call; PostgreSQL RLS (`FORCE ROW LEVEL SECURITY`) with per-transaction `app.org_id`; `404` for foreign ids; jobs re-validate ownership; tenant-escape tests per repository and endpoint; SSE authz on subscribe and resume | 1 |
| T9 | **Privilege escalation inside the platform** (Developer grants self Admin, API key widens scope, custom role abuse) | E | M / H | Role binding requires `role:bind` and can't grant permissions the granter lacks; API key scopes ⊆ owner's; built-in roles immutable; custom roles validated against the permission catalogue; audit of all role/binding changes; authorization-matrix tests | 1, 9 |
| T10 | **Compromised worker** (RCE in worker or its dependencies) | E, I | L / H | Workers have no inbound network, run non-root on a read-only FS with minimal images; they hold plaintext secrets only during tasks; KEK in KMS (ee) with per-request audit; DB role without DDL; dependency scanning; separate worker pools per tenant for enterprise (later); rapid key rotation runbook | 1 (hardening), 12 (KMS) |
| T11 | **Compromised control plane of the platform** (API tier RCE, DB admin access) | E, I, T | L / H | API tier holds no infrastructure credentials and cannot decrypt them; RLS plus role separation; audit hash chain detects tampering; DB TLS; admin access to DB is out-of-band and audited; backups encrypted | 1 |
| T12 | **Malicious add-on** (catalog entry or community plugin that exfiltrates data or installs a backdoor) | T, E | L / H | Add-ons are compiled in or catalog data reviewed by maintainers; no runtime plugin loading; charts pinned by digest; per-add-on namespace and RBAC review; signed catalog bundles (Ed25519) verified on load; third-party catalogs disabled by default (ee: allow-listed, signed by the org) | 4, 12 |
| T13 | **Supply chain attack** (compromised dependency, CI action, build pipeline) | T | M / H | Pinned deps and actions (SHA), Renovate with review, minimal CI permissions (`permissions:` per job, OIDC instead of long-lived secrets), protected branches, signed commits/tags for releases, SBOM, cosign signatures, SLSA provenance, reproducible builds; verification steps documented for users | 1 (CI), 12 (full) |
| T14 | **Malicious container image** (typosquatted or poisoned image pulled into managed clusters) | T, E | M / H | Images referenced by digest from the catalog; image lists generated per release; optional private mirror; signature verification (Sigstore/Kyverno, ee); registry allow/block policies (ee); vulnerability scanning of catalog images in CI | 4, 12 |
| T15 | **Compromised Helm repository** (chart replaced or new malicious version published) | T | M / H | Only catalog-pinned versions **with digest** are installed; OCI registries preferred; provenance/signature verification where publishers support it; air-gap mirrors; catalog update PRs reviewed and tested before release; rendered manifests can be inspected in plan diff | 4 |

## 5. Additional threats by component

| # | Component | Threat | STRIDE | L / I | Mitigations | Phase |
|---|---|---|---|---|---|---|
| T16 | Web UI | XSS via cluster names, labels, log lines, add-on docs | T, I | M / H | React escaping; never `dangerouslySetInnerHTML` with user data; sanitized Markdown; strict CSP | 1 |
| T17 | Web UI | CSRF on state-changing endpoints | T | M / H | Synchronizer token + `SameSite=Lax` + no CORS | 1 |
| T18 | API | Brute force / credential stuffing on login | S | H / M | Rate limits per IP/account, backoff, lock, breached-password check, MFA (ee) | 1 |
| T19 | API | Resource exhaustion (expensive plans, preflights, SSE connections, log downloads) | D | M / M | Rate limits per principal/org, SSE connection caps, pagination limits, request size limits, timeouts, async operations for anything slow | 1–2 |
| T20 | API | Enumeration of ids, users or clusters | I | M / L | UUIDv7 ids, `404` for foreign ids, uniform login errors, authorization-filtered search | 1 |
| T21 | Engine | Duplicate or concurrent operations corrupting a cluster (race, replay) | T, D | M / H | One active operation per cluster (DB constraint), leases with fencing, idempotency keys, plan hash check on resume | 2 |
| T22 | Engine | Rollback causing data loss | T, D | L / H | Only reversible tasks rolled back; irreversible steps flagged and confirmed; pause-by-default failure policy; etcd snapshot before risky control-plane steps | 2, 6 |
| T23 | SSH | Man-in-the-middle between worker and node | S, I | M / H | Mandatory host-key verification (TOFU with explicit confirmation, known_hosts, SSH CA); mismatch stops the operation | 3 |
| T24 | SSH / nodes | Malicious node returns crafted output to exploit parsers or exfiltrate join tokens | T, I | L / M | Output parsed with strict, bounded parsers (max size); join tokens short-lived and single-cluster scoped; per-node data used only for that node; node facts treated as untrusted input | 3 |
| T25 | Kubernetes API of managed cluster | Compromised cluster attacks the platform (malicious API responses, huge objects, slow responses) | D, T | L / M | Timeouts, response size limits, client-go with strict decoding, per-cluster concurrency limits, circuit breakers | 3–4 |
| T26 | Webhooks | Receivers spoofed / replayed payloads | S, T | M / M | Standard Webhooks signatures with timestamp, rotation of signing secrets, documented verification; outbound only, so no inbound trust | 1 |
| T27 | Audit | Repudiation of destructive actions; tampering by DB insider | R, T | L / H | Append-only role permissions, per-org hash chain with verification job, export to external sinks (ee), time from DB server clock | 1 |
| T28 | Backups (platform and clusters) | Backup leak exposes secrets/etcd data | I | M / H | Platform DB backups contain only ciphertext (KEK separate); etcd snapshots and Velero backups encrypted at rest in the target with customer-controlled keys; backup credentials scoped write-only where possible | 6, 13 |
| T29 | Kubeconfig download | Long-lived admin credentials leak from a laptop | I, E | H / H | User-scoped short-lived client certs (default 8 h); admin kubeconfig gated by separate permission; downloads audited; revocation by cluster CA rotation / OIDC (later) | 3–4 |
| T30 | Air-gapped bundles | Tampered offline bundle | T | L / H | Signed manifests and checksums verified on import; import is a privileged audited action | 13 |
| T31 | License / entitlements (ee) | Forged license enabling enterprise features | T | M / L | Ed25519-signed license files verified offline; no security-relevant behaviour depends on the license being genuine (enterprise features are additive) | 9 |

## 6. Residual risks (accepted)

1. **Worker compromise exposes secrets used by in-flight operations.** This is inherent to an agentless design. Mitigations: shorten exposure (just-in-time decryption), isolate workers, use KMS audit trails. Revisit if customers require per-tenant worker pools. Owner: platform architect.
2. **TOFU host-key confirmation depends on the user checking fingerprints.** UX nudges toward known_hosts import or SSH CAs. Enterprise can enforce "known hosts required" policy. Owner: product owner (UX).
3. **Self-hosted operators with DB + KEK access can decrypt everything.** This is by definition. Documented in operator guidance. KMS/HSM with separate duties is available in ee. Owner: security maintainer.
4. **AGPL-licensed upstream components installed into managed clusters** (e.g. Grafana, Loki) are a legal risk, not a security one. They are tracked in the ADR on licensing. Owner: product owner (legal review).

## 7. Process

- Every PR that touches authentication, authorization, tenancy, secrets, remote execution, outbound HTTP, YAML parsing or the catalog must reference the relevant threat IDs, and either confirm existing mitigations or add new rows.
- Before each phase gate: review this document, verify that the mitigations listed for that phase have tests, and record the result in the phase report.
- Before enterprise GA: external penetration test, threat-model review by a third party, DR and backup-restore exercise (prompt §223).
