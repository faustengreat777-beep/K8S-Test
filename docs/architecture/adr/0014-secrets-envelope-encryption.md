# ADR-0014: Secrets — envelope encryption with a pluggable key provider

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0014-secrets-envelope-encryption.ru.md)

## Context

The platform stores the most sensitive material its users have: SSH private keys, cloud API tokens, cluster admin kubeconfigs, backup storage keys, Git and registry credentials. The spec requires that credentials are encrypted at rest and never reach logs, the frontend, git or error messages (prompt §25). It asks for envelope encryption or another correct model (prompt §85), and in Enterprise for Vault, cloud secret managers, KMS integration, key rotation and BYOK (prompt §167, §213–214).

## Decision

- **Envelope encryption**: payloads are encrypted with **AES-256-GCM** using a per-organization **data-encryption key (DEK)**. DEKs are stored only **wrapped** by a **key-encryption key (KEK)** obtained from a `KeyProvider`.
- **AAD** = `org_id ‖ secret_id ‖ purpose`. This binds every ciphertext to its tenant, record and use.
- **KeyProvider** implementations: `local` (Community: 256-bit key from `ENCRYPTION_KEY_FILE`/`ENCRYPTION_KEY`, never stored in the database), then `openbao-transit`/`vault-transit`, `aws-kms`, `gcp-kms`, `azure-keyvault` (Enterprise, BYOK).
- **SecretBackend** abstraction for where payloads live: `internal` (encrypted rows in PostgreSQL) by default; OpenBao/Vault KV and cloud secret managers in Enterprise.
- **Only the worker role decrypts.** The api role has no unwrap path. Decryption is just in time per task, and plaintext is cleared after use (best effort).
- **Rotation**: KEK rotation re-wraps DEKs. DEK rotation re-encrypts lazily, with key states `active` → `decrypt-only` → `destroyed`. Credential rotation is a product feature that tracks dependent clusters.
- **API**: secret fields are write-only. Reads return metadata (fingerprint, last 4 characters, timestamps).
- **Redaction**: values fetched through `SecretAccessor` are masked in all logs and events of the operation; pattern- and field-based masking applies as well. Canary leak tests run in CI.
- **Crypto implementation: Google Tink Go** (Apache-2.0). Each organization's DEK is a Tink **AEAD keyset** (AES-256-GCM). It is stored in `data_keys` encrypted by the KEK AEAD (`keyset.Handle` written with a master-key AEAD). Tink keysets support several keys with one primary, so **DEK rotation is native**: a new primary key encrypts, and old keys still decrypt. KEK AEADs:
  - `local`: a Tink AEAD from the local key file;
  - Enterprise: Tink KMS extensions — `tink-go-gcpkms/v2`, `tink-go-awskms/v3` (v3 is on aws-sdk-go-v2; v2 depends on the end-of-life SDK v1) and `tink-go-hcvault/v2` (works against OpenBao/Vault Transit);
  - Azure Key Vault: a small custom `KMSClient`.
  We write no custom cryptographic primitives.

## Alternatives considered

| Option | Why not |
|---|---|
| Single static key encrypting all rows | No per-tenant separation, painful rotation, a large blast radius |
| PostgreSQL `pgcrypto` | The key would pass through SQL and the DB server, and could leak in logs or `pg_stat_statements`. It couples crypto to the DB tier, which we treat as untrusted for plaintext |
| Require HashiCorp Vault for all installs | A heavy dependency for Community/self-hosted. Vault's license is BSL since 2023 (OpenBao is the MPL-2.0 fork). Kept as an optional backend |
| Store secrets in Kubernetes Secrets of a management cluster | There is no management cluster in our architecture, and K8s Secrets are only base64 unless KMS is configured |

## Consequences

- Positive: compromise of the DB alone yields only ciphertext. Tink keeps the primitive choice, nonce handling and key rotation in a well-reviewed library. Tenants are cryptographically separated. Rotation is cheap. KMS/BYOK can be plugged in without changing data formats (wrapped key + provider id + key version are stored).
- Negative: KEK availability becomes critical. Losing it means losing all secrets, so KEK backup must be separate and documented. Workers need KEK access, which raises their sensitivity (mitigated by isolation and KMS audit in Enterprise).

## References

- NIST SP 800-38D (GCM); Google Tink Go v2 (envelope encryption, KMS extensions); [Security model §6](../../security/SECURITY_MODEL.md), [Data model §3.3](../data-model.md)
