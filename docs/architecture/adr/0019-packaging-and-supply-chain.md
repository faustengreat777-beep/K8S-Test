# ADR-0019: Packaging, distribution and supply-chain security

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0019-packaging-and-supply-chain.ru.md)

## Context

The spec requires production-ready Dockerfiles (multi-stage, non-root, minimal images, health checks, no `latest`; prompt §79), a GitHub Actions pipeline (lint → unit → integration → build → security scan → E2E → docker build → publish; prompt §80), and security scanning: Trivy, dependency scanning, secret scanning, SBOM (prompt §81). Enterprise adds signed artifacts, provenance and reproducible builds where possible (prompt §223, §225–226).

2025–2026 events also matter here: the tj-actions/changed-files compromise (2025) and the **Trivy ecosystem compromise (March 2026)**, which affected the Trivy v0.69.4 release, some Docker Hub images and `trivy-action`/`setup-trivy` tags. The tools that protect the supply chain are themselves part of it.

## Decision

**Artifacts**
- One static server binary (`CGO_ENABLED=0`, `-trimpath`, version metadata via `-ldflags`) with the web UI and default catalog embedded; a CLI binary for linux/macOS/windows × amd64/arm64; a signed node-helper binary (ADR-0015).
- OCI images: multi-stage builds, final image on `gcr.io/distroless/static` (or equivalent), non-root UID 65532, read-only root filesystem, no shell, `HEALTHCHECK` for compose installs, multi-arch manifest, **immutable semver tags + digests, never `latest`**.
- Helm chart for the platform, docker compose bundles, `.env.example`.

**Pipeline (GitHub Actions)**
`lint → unit → integration → build → security scan → E2E (simulated, container-nodes subset) → docker build → publish (tags only)`; nightly extended suites.

**Supply-chain controls**
1. **Pin everything**: Go modules (`go.sum`), pnpm lockfile, base images by digest, GitHub Actions **by full commit SHA** (including security tools). Renovate opens update PRs, which are reviewed.
2. **Least-privilege CI**: per-job `permissions:`, OIDC to registries instead of long-lived tokens, protected branches and tags, required reviews, no `pull_request_target` with checkout of untrusted code, and egress hardening for build jobs where available.
3. **Scanning**:
   - `govulncheck`;
   - OSV-Scanner (Go and npm dependencies);
   - an image scanner — **Grype** by default, Trivy allowed only at a known-good version pinned by digest after the 2026 incident, so two independent scanners can be compared in nightly runs;
   - **gitleaks** (and GitHub push protection);
   - static analysis (golangci-lint with gosec; ESLint security rules);
   - a **license policy** gate (deny GPL/AGPL/SSPL/BSL in the *linked* dependency graph; record MPL-2.0 usage in NOTICE).
4. **SBOM**: Syft-generated SBOMs for binaries and images (SPDX and CycloneDX), attached to releases and to image attestations.
5. **Signing & provenance**: **cosign keyless** signatures (Sigstore, GitHub OIDC) for images, binaries, the Helm chart, catalog bundles and offline bundles; **SLSA build provenance** attestations (GitHub artifact attestations / SLSA generator). Users get documented verification commands.
6. **Reproducible builds where possible**: pinned toolchain, `SOURCE_DATE_EPOCH`, deterministic archives. Rebuild verification runs as a nightly job.
7. **Catalog artifacts** (charts, images, node binaries) are pinned by digest and checksum and verified at use. The catalog bundle is signed with Ed25519 (ADR-0017).

## Alternatives considered

- **Alpine/Ubuntu base images**: larger attack surface and shells. Distroless/static is enough for a static Go binary.
- **Trivy as the only scanner**: the March 2026 compromise showed the risk of trusting one tool's distribution channel. Pinned versions and a second scanner reduce it.
- **GPG-signed releases with long-lived keys**: key management burden. Keyless Sigstore with transparency logs is the current best practice.

## Consequences

- Positive: verifiable releases (signature + SBOM + provenance), a smaller attack surface, a documented chain of custody for enterprise customers.
- Negative: CI is more complex and slower. Pinned-SHA actions need Renovate to stay current.

## References

- Sigstore cosign, SLSA v1.x, Syft, Grype, OSV-Scanner, gitleaks, govulncheck; Trivy security advisory GHSA-69fq-xp46-6x23 (2026)
- [Security model §11](../../security/SECURITY_MODEL.md), [Deployment model](../deployment-model.md)
