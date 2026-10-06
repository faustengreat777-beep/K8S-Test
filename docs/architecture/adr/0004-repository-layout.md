# ADR-0004: Go-idiomatic monorepo layout with a single Go module

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0004-repository-layout.ru.md)

## Context

The spec proposes a TypeScript-style layout (`apps/`, `packages/`, top-level `providers/`, `distributions/`, `addons/`) and explicitly invites a more correct structure if there is one (prompt §76). The backend is Go, the frontend is TypeScript, and they live in one repository. Plugin boundaries (providers, distributions, add-ons) must be clean enough that new implementations don't require rewriting the core (prompt §114).

## Decision

- Layout: `cmd/` (binaries), `internal/` (private code, compiler-enforced), `pkg/` (small public API: `sdk`, `spec`, `client`), `plugins/` (providers, distributions, OS families, add-ons), `catalog/` (data), `api/openapi/`, `migrations/`, `web/` (pnpm workspace), `ee/` (enterprise, later), `deploy/`, `test/`, `hack/`, `docs/`. Full tree: [repository structure](../repository-structure.md).
- **One Go module** at the repository root. `pkg/sdk` may become its own module once third parties build plugins against it.
- Dependency rules (domain → nothing; engine → domain + sdk; plugins → sdk only; core never imports `ee/`) are enforced with `depguard`/import-boundary linting in CI.
- Enterprise code lives in `ee/` in the same repository under a separate license and the `ee` build tag (ADR-0002).
- Go module path: `github.com/faustengreat777-beep/k8s-test` until the repository is renamed to the product name. Renaming is recommended before Phase 1, when it is still cheap.

## Alternatives considered

- **`apps/` + `packages/` as proposed**: idiomatic for Node monorepos, but in Go it loses the `internal/` encapsulation and creates many tiny modules.
- **Multi-module Go workspace (one module per plugin)**: independent versioning, but heavy release management, `replace` directives and slower builds. Premature for now.
- **Separate repositories for web, CLI and plugins**: cross-repo changes become slow, and API/codegen drifts.

## Consequences

- Positive: compiler-enforced encapsulation, a clear public surface, one `go build`, atomic cross-cutting changes.
- Negative: the frontend and backend share CI and release cadence. This is acceptable and even desirable, because the UI is a client of the same API version.

## References

- [Repository structure](../repository-structure.md)
