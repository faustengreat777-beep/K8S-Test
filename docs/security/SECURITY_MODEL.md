# Security model

> Status: **Proposed** (Phase 0). Language: English · [Русский](SECURITY_MODEL.ru.md)
>
> Related: [Threat model](THREAT_MODEL.md) · [ARCHITECTURE](../ARCHITECTURE.md) · [Data model](../architecture/data-model.md) · [API design](../architecture/api-design.md) · ADR-0011 (tenancy), ADR-0014 (secrets and encryption), ADR-0015 (remote execution)

Farvater holds the keys to its users' infrastructure: root-capable SSH keys, cloud API tokens, cluster admin credentials, backup storage keys. Compromising the platform means compromising every cluster it manages. Security is therefore a first-class design concern from Phase 1, not an enterprise add-on (prompt §54, §99).

## 1. Principles

| Principle | Meaning in this product |
|---|---|
| Least privilege | Every principal (user, API key, service account, worker, DB role, platform component) gets the minimum permissions; API keys can only narrow their owner's rights; DB roles are split (`app`, `app_system`, `app_owner`) |
| Deny by default | No permission → no access; unknown routes, fields and enum values are rejected; outbound network to private ranges is denied unless allow-listed |
| Defence in depth | Authorization in the app layer **and** Row-Level Security in PostgreSQL; input validation **and** typed commands **and** no shell interpolation; encryption **and** access control **and** audit |
| Secure defaults | TLS everywhere, host-key verification on, hardened cluster profile for production, PodSecurity `restricted`, default-deny NetworkPolicies offered, secrets encryption at rest in clusters, short-lived kubeconfigs |
| Zero trust where possible | Every request authenticated and authorized; internal hops (API ↔ DB, worker ↔ nodes) encrypted and authenticated; no implicit trust in network location |
| Secrets are first-class | Dedicated vault with envelope encryption, write-only API fields, redaction everywhere, just-in-time decryption in workers only |
| Auditability | Every security-relevant action recorded append-only with actor, source, target and result |
| Minimal attack surface | Single binary, few dependencies, no runtime plugin loading, no arbitrary script execution, metrics on a separate listener |

## 2. Trust boundaries

```
 Internet / corporate network
   │  (TLS)
   ▼
┌───────────────────────┐   Boundary A: untrusted clients → API
│ api role              │   authn, CSRF, rate limits, input validation, authz, audit
│ (no infra credentials)│
└──────────┬────────────┘
           │ PostgreSQL (TLS, role `app`, RLS)          Boundary B: app ↔ database
┌──────────▼────────────┐
│ PostgreSQL            │   encrypted secret blobs only; KEK never stored here
└──────────▲────────────┘
           │ jobs (River) + leases
┌──────────┴────────────┐   Boundary C: worker holds decrypted credentials transiently
│ worker role           │   KEK access (local key file / KMS), SSRF-guarded egress
└───┬─────────┬─────────┘
    │ SSH     │ HTTPS (provider APIs, K8s API, Helm/OCI registries, Git, webhooks)
    ▼         ▼                                         Boundary D: platform → customer infrastructure
 Customer nodes & clusters, cloud provider APIs, external services
```

Design consequences:
- The **api role never holds infrastructure credentials and never connects to customer infrastructure**. It cannot decrypt credential secrets. Envelope-decryption is wired only into the worker's `SecretAccessor`. The one exception is webhook signing for *outgoing* platform events, which is done by the worker. If the API tier is compromised, the attacker still has no plaintext keys.
- Workers can run in a separate network zone with egress only to customer infrastructure and the database, and no ingress at all.
- The KEK (key-encryption key) is never stored in PostgreSQL. It comes from a file or environment variable mounted only into workers (Community), or from an external KMS/Transit engine (Enterprise).

## 3. Identity and authentication

**Users (browser)**
- Local accounts (Phase 1): email + password hashed with **argon2id** (memory ≥ 64 MiB, iterations ≥ 3, parallelism 1–4, 16-byte salt; parameters versioned in the PHC string and re-hashed on login when they change). Minimum length 12, checked against a breached-password list (k-anonymity range API optional/offline list), no composition rules.
- Sessions: random 256-bit token in a `__Host-session` cookie (`HttpOnly; Secure; SameSite=Lax; Path=/`). Only its SHA-256 is stored. Idle timeout (default 30 min) and absolute lifetime (default 12 h). The session ID rotates on login and on privilege change. Users can list and revoke their sessions, and admins can revoke all of a user's sessions.
- CSRF: synchronizer token bound to the session and required in `X-CSRF-Token` for unsafe methods. `SameSite=Lax` adds a second layer. CORS is disabled by default.
- Login hardening: per-account and per-IP exponential backoff, a temporary lock after repeated failures (with an audit event), generic error messages, constant-time comparisons.
- Phase 9 / ee: MFA (TOTP, WebAuthn/passkeys) with an admin policy "MFA required", OIDC and SAML 2.0 SSO through the `IdentityProvider` abstraction (Okta, Microsoft Entra ID, Google Workspace, Keycloak, Auth0, generic OIDC), SCIM 2.0 provisioning and deprovisioning. Deprovisioning disables the user, revokes sessions and revokes API keys (prompt §143).

**Machines**
- API keys: format `fvt_<12-char public id>_<32-byte random secret, base62>`. The server stores the prefix and an HMAC-SHA256 of the secret, keyed with a server-side pepper held in config, not in the DB. The full key is shown once. Keys have scopes (permission subsets), expiry (default 90 days, max configurable), last-used time and IP, and can be revoked instantly. Keys found in the gitleaks/GitHub secret-scanning format can be auto-revoked later via secret-scanning partner integration.
- Service accounts (Phase 9): non-human principals with their own role bindings, used by CI/CD, Terraform and GitOps.
- Workers authenticate to PostgreSQL with their own credentials. Worker instance ids are recorded in leases and audit (`actor_type=system`).

## 4. Authorization

- **Model:** permissions are strings `resource:verb`. Roles are named sets of permissions. Role bindings attach a role to a subject (user, team, service account, API key) at a **scope** (organization, project or cluster). A binding at org scope applies to all projects and clusters in the org.
- **Built-in roles (Community):**

| Role | Summary |
|---|---|
| Owner | Everything in the org, including org deletion, billing/license and role management |
| Admin | Everything except org deletion and ownership transfer |
| Operator | Create/update/scale/upgrade clusters, run operations, manage add-ons, nodes, backups; use (not read) credentials |
| Developer | Read clusters and operations; download kubeconfig (user-scoped); create clusters only in projects/environments allowed by bindings (development by default) |
| Viewer | Read-only, no kubeconfig, no credential metadata beyond names |

- **Enterprise:** the extended default roles from prompt §139 (Organization Owner, Organization Admin, Platform Admin, Cluster Admin, SRE, DevOps, Developer, Security Admin, Auditor, Viewer), **custom roles** (prompt §140), and **ABAC conditions** evaluated by the policy engine over principal, resource and environment attributes (for example `project.environment == "development"`, `cluster.region == "eu"`, `cluster.ownerTeam in principal.teams`; prompt §141).
- **Enforcement:** a single `Authorizer.Authorize(principal, permission, resource)` call in every application-service method, before any side effect. Handlers can't skip it, because services require an authorized context type. The permission catalogue documents each permission and the endpoints that use it. **Authorization-matrix tests** cover every endpoint × role × tenant combination.
- **Separation of duties hooks (ee):** approval workflows ensure the requester of a production change can't be its only approver.
- **Credential use vs read:** using a credential in an operation (`credentials:use`) is a different permission from reading its metadata (`credentials:read`). No permission ever returns secret material.

## 5. Tenant isolation

- Every tenant-owned row carries `org_id` and, where relevant, `project_id`. Every repository call requires a `TenantScope`, derived from the authenticated principal after authorization and never from client input alone.
- PostgreSQL **RLS** policies on all tenant tables use `current_setting('app.org_id')`, set with `SET LOCAL` per transaction (see [data model §4](../architecture/data-model.md#4-tenant-isolation)). The application role cannot bypass RLS.
- Background jobs carry `org_id` and re-validate ownership of every referenced id.
- Logs, audit, operation events, secrets, kubeconfigs and search results are tenant-scoped. SSE streams check authorization at subscribe time and on every resumed connection.
- Object ids are unguessable (UUIDv7 has 74 random bits). Even so, a foreign or unknown id returns `404`, not `403`, so callers can't enumerate resources.
- Tests: tenant-escape tests per repository and per endpoint. Fuzzed id substitution in API tests.
- Never rely on the frontend for isolation (prompt §212).

## 6. Secrets management

### 6.1 Envelope encryption (prompt §85)

```
KEK  (key-encryption key)  — from KeyProvider: local key file / env (Community) | OpenBao/Vault Transit | AWS KMS | GCP KMS | Azure Key Vault (ee, BYOK)
 └── wraps  DEK (data-encryption key, 256-bit, per organization, versioned)  — stored wrapped in `data_keys`
       └── encrypts  secret payload with AES-256-GCM, random 96-bit nonce, AAD = org_id ‖ secret_id ‖ purpose  — stored in `secrets`
```

- **AAD binding** stops ciphertext swapping between tenants, records or purposes. Copying a ciphertext into another org's row makes decryption fail.
- **Nonce safety:** random 96-bit nonces under one DEK are fine for far more encryptions than one org will ever do. DEKs are rotated on a schedule (default yearly) or on demand, which also bounds nonce use.
- **Rotation:**
  - KEK rotation re-wraps DEKs only, which is cheap.
  - DEK rotation creates a new active DEK. New writes use it, and a background job re-encrypts old payloads lazily. The old DEK becomes `decrypt-only`, then `destroyed` after re-encryption completes.
  - Credential rotation (new SSH key or token) is a product feature with dependent-cluster tracking (prompt §169).
- **Implementation:** Google Tink Go AEAD (AES-256-GCM) keysets per organization, encrypted by the KEK AEAD. Rotation of keys inside a keyset is native, and Tink KMS extensions cover GCP KMS, AWS KMS and OpenBao/Vault Transit (ADR-0014). There are no custom cryptographic primitives, and known-answer tests run in CI.
- **Local KEK (Community):** 256-bit key from `ENCRYPTION_KEY` (base64) or a key file (`ENCRYPTION_KEY_FILE`, recommended, mounted from a Kubernetes Secret or a file with 0400 permissions). The server refuses to start without it in non-dev profiles. Key ID and version are recorded with each wrapped DEK, so several KEKs can coexist during rotation.
- **External backends (ee):** `SecretBackend` implementations can store payloads in OpenBao/Vault KV or cloud secret managers instead of PostgreSQL. The `secrets` row then holds only a reference.

### 6.2 Handling rules (prompt §25)

Credentials and secrets must:
1. **be encrypted at rest**, as above.
2. **never reach the frontend.** Secret fields in the API are write-only. Responses carry metadata: type, fingerprint (e.g. `SHA256:…` for SSH keys), last four characters of tokens, created, rotated, last used and expiry.
3. **never appear in logs.** The redactor masks every value obtained through `SecretAccessor` in all log and event output of that operation. Pattern-based masking covers PEM blocks, bearer tokens, `password=`/`token=` pairs and kubeconfig `client-key-data`. Structured fields on a denylist (`password`, `token`, `secret`, `privateKey`, …) are dropped.
4. **never appear in error messages.** Typed errors carry codes and safe parameters. Raw upstream errors pass through the same redactor before they are stored.
5. **never be committed to git.** gitleaks runs in CI and in pre-commit. Test fixtures use generated throwaway keys. `.env` is git-ignored and `.env.example` holds placeholders only.
6. **be decrypted just in time**, only in worker memory, only for the task that needs them. Plaintext buffers are cleared after use, best effort in Go. Keys are never written to disk. The SSH adapter parses keys in memory, and agent forwarding is disabled by default.

Canary tests plant known secret values in test runs, then scan logs, events, API responses and error payloads for leaks.

### 6.3 Secrets inside managed clusters

- kubeadm clusters are configured with **encryption at rest for Secrets** (`EncryptionConfiguration`). The `aescbc`/`secretbox` local provider is used by default, and **KMS v2** is used when an external KMS is configured (ee).
- Join tokens are short-lived (default 1 h) and deleted after the operation. kubeadm certificate keys are uploaded only for the duration of control-plane joins.
- The admin kubeconfig is stored encrypted and never shown in the UI by default. Users download **user-scoped short-lived kubeconfigs**: a client certificate signed by the cluster CA with a TTL (default 8 h), or OIDC later. Each download is audited.
- The CA private keys of managed clusters stay on control-plane nodes (kubeadm default). The platform keeps an encrypted copy only if the user enables "platform-managed PKI backup" for disaster recovery. Certificate rotation is an operation (prompt §29).

## 7. Remote execution safety (SSH, kubectl, Helm)

- **No shell interpolation.** Remote commands are argv lists (`Command{Program, Args}`). The SSH adapter quotes each argument with strict POSIX single-quote escaping and runs it under `/bin/sh -c` only as a transport wrapper, with no user-controlled text outside quoted arguments. A lint rule (custom `go/analysis` analyzer) forbids `exec.Command("sh", "-c", …)` and string concatenation into `Command.Program` (prompt §116).
- **Validated inputs.** Hostnames (RFC 1123), IPs/CIDRs (`net/netip`), ports, Kubernetes names/labels/taints, versions (catalog-resolved only), paths (allow-listed prefixes), URLs (scheme allow-list, SSRF guard). Validators are fuzz-tested.
- **Files over SFTP.** Configuration (kubeadm, containerd, sysctl, systemd units, kube-vip manifest) is rendered from typed Go structs, marshalled with real encoders (YAML/TOML/INI), uploaded atomically (temp + fsync + rename) with explicit ownership and mode. `echo`/heredoc file writing is never used.
- **Ephemeral node helper (recommended, Phase 3).** The bootstrap layer may upload a small, signed, statically linked helper binary per operation. It is invoked as `helper <step> < json-input` and performs facts collection, file and package operations with structured input and output. This avoids differences between distro shell tools (for example Rust coreutils and `sudo-rs` on Ubuntu 26.04) and shrinks the command surface further. The helper is removed after the operation (agentless model kept). See ADR-0015.
- **Host keys.** Unknown host keys are never accepted silently. The user confirms the fingerprint (or supplies known_hosts/SSH CA), and it is stored in `ssh_known_hosts`. A mismatch stops the operation with `SSH_HOST_KEY_MISMATCH`.
- **Privilege.** `sudo -n` is used only for specific steps, root login is not required, and the doc recommends a dedicated provisioning user with restricted sudo.
- **Kubernetes API.** client-go with the cluster's CA pinned. Server-side apply with a dedicated field manager. RBAC of the platform's in-cluster identity limited to what add-on management needs.
- **Helm.** Embedded SDK (no shell). Charts come only from catalog-pinned references with digests. User value overrides are structured data validated against the chart schema, never template text.
- **User-provided YAML.** Strict decoding (unknown fields error), size limit (default 1 MiB), alias/anchor expansion limit (billion-laughs protection), max depth, then JSON Schema validation. ClusterSpec "advanced" patches are limited to documented kubeadm/kubelet/containerd configuration fields.

## 8. Network security of the platform

- **TLS** on all external listeners (TLS 1.2+, modern ciphers; TLS 1.3 preferred). The built-in server can use provided certificates or sit behind a TLS-terminating proxy, with HSTS when serving HTTPS directly.
- **PostgreSQL** over TLS with `verify-full` in production profiles.
- **SSRF guard** (`internal/adapters/httpx`) for every outbound request to a user-influenced URL: webhooks, Helm/OCI repositories, Git, registries, OIDC discovery, notification endpoints. It:
  - resolves DNS, then dials the resolved IP (no TOCTOU),
  - blocks loopback, link-local, multicast, unspecified, private ranges (configurable allow-list for self-hosted installs that need them), cloud metadata endpoints (`169.254.169.254`, `fd00:ec2::254`, `metadata.google.internal`, …),
  - re-checks on every redirect (max 3),
  - enforces scheme allow-list (`https`, plus `http` only if explicitly allowed), timeouts and response size limits.
- **Path traversal**: the only file serving is the embedded web UI (`embed.FS`, no filesystem access). Archive extraction (offline bundles, Helm chart cache) checks for `..`, absolute paths and symlinks.
- **Rate limiting** per user, API key, org and IP; separate stricter limits for login, plan, preflight and kubeconfig.
- **Security headers**: CSP (`default-src 'self'`, no inline scripts, `frame-ancestors 'none'`), `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` minimal.
- **Metrics and debug endpoints** run on a separate listener bound to localhost or the pod network, never on the public port.

## 9. Audit

- Recorded actions (prompt §118, §156): authentication (success/failure), authorization denials, cluster create/update/delete/upgrade/scale, backup/restore, add-on install/remove, node actions, kubeconfig downloads, credential create/rotate/delete/use, role and binding changes, user changes, API key lifecycle, webhook changes, policy changes and approvals (ee), settings changes.
- Each event records **who** (actor type, id and display name at that time), **what** (action), **when** (server timestamp), **from where** (IP, user agent, request id), **target** (type, id, project, cluster), **result** (success / failure / denied) and **details** (redacted, e.g. spec diff summary).
- Integrity: `audit_events` is append-only. The app role has `INSERT` only, with no `UPDATE`/`DELETE`. Rows are **hash-chained per organization**, and a verification job and API endpoint detect tampering. Retention is by partition drop after export (configurable; enterprise retention lock).
- Export: JSON and CSV in Community. SIEM webhook, syslog (RFC 5424 over TLS), S3, Splunk, Datadog, Elastic and Sentinel in Enterprise (prompt §158).

## 10. Security of managed clusters (what the platform configures)

Production and "hardened" profiles turn on:

| Area | Default in hardened profile |
|---|---|
| Pod Security Admission | `restricted` enforced for user namespaces, `baseline` audit/warn cluster-wide; system namespaces exempted explicitly |
| NetworkPolicies | Default-deny per namespace template offered; CNI with policy support required for production |
| API server | Anonymous auth limited to health endpoints, audit logging with a sensible policy, secrets encryption at rest, admission plugins (NodeRestriction, etc.), profiling disabled |
| Kubelet | Webhook authn/authz, read-only port disabled, rotate certificates, protect kernel defaults |
| etcd | TLS peer and client auth (kubeadm default), snapshots encrypted at rest in the backup target |
| Certificates | cert-manager for workload TLS (Let's Encrypt HTTP-01/DNS-01, internal CA, existing, self-signed), expiry monitoring for control-plane certs |
| RBAC | No cluster-admin bindings for users by default; user kubeconfigs mapped to groups with least privilege |
| Image policy (later/ee) | Allowed/blocked registries, signature verification (Kyverno `verifyImages`/Sigstore), private default registry |
| Scanning (later/ee) | kube-bench / Kubescape for CIS benchmark, Trivy Operator for vulnerabilities, security posture score per cluster |

The platform never claims that a cluster is "compliant" with any framework. It reports controls, status, evidence and recommendations (prompt §160).

## 11. Supply chain and release security

- Dependencies are pinned: `go.sum`, the pnpm lockfile, and GitHub Actions pinned by full commit SHA. Renovate/Dependabot open update PRs, and every update runs the full test suite.
- CI security gates: `govulncheck`, OSV-Scanner/Grype on dependencies, image scanning (scanner choice in ADR-0019), gitleaks, static analysis (gosec rules in golangci-lint, ESLint security rules), license policy check.
- Release artifacts get checksums, **cosign keyless signatures** (Sigstore, GitHub OIDC), an **SBOM** (SPDX or CycloneDX, generated by Syft), **SLSA provenance** attestations, version and build metadata (prompt §226). The goal is reproducible builds (`-trimpath`, pinned toolchain, `SOURCE_DATE_EPOCH`).
- Catalog content (charts, images, binaries for nodes) is pinned by digest/checksum and verified at use. Offline bundles carry signatures and checksums and are verified on import.
- Container images: minimal (distroless/static), non-root UID, read-only root filesystem, no shell, `HEALTHCHECK` where the runtime supports it, no `latest` tags (prompt §79).

## 12. Operational security

- **Configuration:** all secrets via files or env (`DATABASE_URL`, `ENCRYPTION_KEY_FILE`, `SESSION_SECRET_FILE`, `API_KEY_PEPPER_FILE`, …). `.env.example` is documented. The server validates config at start-up and refuses insecure combinations in production profiles (missing KEK, plaintext DB connection, debug mode).
- **Logging:** structured JSON (`slog`), request ids, no secrets, no full request bodies for sensitive endpoints.
- **Backups of the platform:** database (including encrypted secrets), configuration, audit logs. KEK backups are kept **separately** from database backups. Restore tests are part of the release checklist (prompt §183).
- **Vulnerability disclosure:** `SECURITY.md` (Phase 1) with a private reporting channel, supported versions and response targets.
- **Threat model** maintained in [THREAT_MODEL.md](THREAT_MODEL.md) and reviewed at every phase gate. New features that touch credentials, tenancy, remote execution or outbound network access need a threat-model update in the same PR.
