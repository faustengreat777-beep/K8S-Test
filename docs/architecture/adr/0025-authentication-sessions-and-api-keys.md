# ADR-0025: Authentication — server-side sessions for the UI, hashed API keys for automation, SSO in Enterprise

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0025-authentication-sessions-and-api-keys.ru.md)

## Context

The spec lists `JWT_SECRET` among environment variables (prompt §84) and requires authentication for UI, API and CLI, session expiration, refresh, revocation, a session list, MFA and SSO in Enterprise (prompt §142–146). API keys must be scoped, expiring, revocable, shown once and stored hashed (prompt §146).

## Decision

- **Browser**: opaque random session tokens in `__Host-` cookies (`HttpOnly; Secure; SameSite=Lax`). Only the SHA-256 is stored server-side, with idle and absolute expiry, rotation on login and privilege change, a per-user session list, revoke and revoke-all. CSRF uses a synchronizer token.
- **API keys**: `fvt_<public id>_<secret>`. The prefix is used for lookup and the secret is verified with HMAC-SHA256 using a server pepper. Keys are scoped (subset of the owner's permissions), expiring, revocable, record last-used IP and time, and are shown once.
- **No JWTs for first-party sessions.** Revocation must be immediate and sessions must be listable, which is simpler and safer with server-side state. JWTs may appear only where a protocol requires them: OIDC ID tokens from IdPs, and short-lived tokens for the future OAuth device flow.
- **Passwords**: argon2id with versioned parameters, breached-password checks, login backoff and lock.
- **Enterprise**: OIDC and SAML 2.0 through an `IdentityProvider` abstraction; SCIM 2.0 provisioning with immediate session and key revocation on deprovisioning; MFA (TOTP, WebAuthn/passkeys) with org policy; service accounts.

## Alternatives considered

- **Stateless JWT sessions**: revocation needs deny-lists (which brings state back), signing keys become a high-value secret, and token theft is harder to contain.
- **External auth proxy only (oauth2-proxy)**: not enough for API keys, CLI and fine-grained authorization; still useful as an *optional* deployment pattern in front of the UI.

## Consequences

- Positive: immediate revocation, simple mental model, fewer secrets (`JWT_SECRET` not needed; `SESSION_SECRET_FILE` and `API_KEY_PEPPER_FILE` instead).
- Negative: a session lookup per request. This is cheap with an index and a small per-replica cache with short TTL, with revocation broadcast via `NOTIFY`.

## References

- OWASP ASVS (session management), [Security model §3](../../security/SECURITY_MODEL.md), [API design §8](../api-design.md)
