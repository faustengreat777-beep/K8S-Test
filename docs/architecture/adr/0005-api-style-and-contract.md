# ADR-0005: REST/JSON API with a committed OpenAPI 3.1 contract (Huma, contract-first in Go), async operations and problem+json errors

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0005-api-style-and-contract.ru.md)

## Context

The UI, the CLI, a future Terraform/OpenTofu provider and customer automation must all use the same API, with no business logic in clients (prompt §40, §56, §57, §198). The API needs authentication, authorization, validation, pagination, filtering, sorting, idempotency, async jobs, webhooks, audit and versioning (prompt §40, §87). Deployments are long-running and must not run inside HTTP requests (prompt §41).

## Decision

- **REST over HTTPS with JSON**, resource-oriented paths under `/api/v1`. Only additive changes within v1. Breaking changes go to `/api/v2`, served side by side, with `Deprecation`/`Sunset` headers.
- **OpenAPI 3.1 contract, authored as typed Go with Huma v2** (on the chi router): operations are declared with request/response structs and validation tags. Huma generates the OpenAPI 3.1 document, JSON Schemas, request validation and RFC 9457 errors.
- The generated document `api/openapi/openapi.yaml` is **committed** and treated as the contract:
  - every PR shows its diff, so API changes are reviewed like code;
  - **oasdiff** blocks breaking changes within v1;
  - **vacuum** lints style and consistency;
  - a CI job fails when the committed spec or generated clients are stale.
- **Clients**: TypeScript types are generated from the committed spec (`openapi-typescript` + `openapi-fetch`). The CLI uses `pkg/client`, a thin typed client over the shared request/response types. External consumers (e.g. the Terraform provider) may generate clients from the spec (oapi-codegen client mode).
- Every response in integration tests is validated against the committed spec (kin-openapi or libopenapi validator).
- **Long-running work returns `202 Accepted` + an `Operation` resource**. Progress is read by polling or SSE (ADR-0009).
- **Errors**: RFC 9457 `application/problem+json` with stable `code`, localized `title`/`detail`, `requestId`, field `errors[]`, and for operation failures `location`, `remediation[]` and `actions[]`.
- **Idempotency-Key** header on creating/starting POSTs (24 h replay window). `ETag`/`If-Match` for optimistic concurrency.
- **Cursor pagination**, explicit filter parameters and allow-listed `orderBy`.
- **Webhooks** in the Standard Webhooks format (HMAC-SHA256 signatures).
- Details: [API design](../api-design.md).

## Alternatives considered

| Option | Why not |
|---|---|
| gRPC (+ grpc-gateway) | Great for service-to-service, but browsers need a gateway anyway; REST + OpenAPI is friendlier for the CLI, Terraform, curl and third parties |
| GraphQL | Flexible reads, but complex authorization per field, caching and rate limiting; long-running operations and SSE still need REST-like endpoints |
| Spec-first with server code generation (oapi-codegen, ogen) | The most "pure" contract-first approach, but in Oct 2026 Go generators support OpenAPI 3.1 only partially: oapi-codegen calls its 3.1 support "initial" (v2.8.0, Jul 2026) and ogen needs pre-processors for 3.1 nullability. We would either stay on 3.0 or fight the generator. Huma emits valid 3.1 natively, and committing the generated spec with breaking-change gating keeps the contract reviewed before merge. Revisit when 3.1 server generation matures |
| Hand-written handlers + hand-written spec | Drift between code and spec is inevitable |
| Kubernetes-style aggregated API / CRDs as the product API | Elegant for in-cluster tools, but the platform is not itself a Kubernetes controller and must run without a management cluster |

## Consequences

- Positive: one contract for all clients, generated code instead of hand-written DTOs, reviewable API changes, consistent errors.
- Negative: the contract lives in Go types, so non-Go contributors review the generated YAML diff rather than editing the spec directly. Huma is led by a single maintainer (MIT, v2 stable since 2023). That concentration risk is mitigated because the committed spec is portable to other generators.

## References

- Huma v2 (huma.rocks), oasdiff, vacuum; RFC 9457 (Problem Details), RFC 9745 (Deprecation header), RFC 8594 (Sunset), Standard Webhooks, IETF RateLimit header fields draft
- [API design](../api-design.md)
