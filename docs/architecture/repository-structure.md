# Repository structure

> Status: **Proposed** (Phase 0). Language: English · [Русский](repository-structure.ru.md)
>
> Related: [ARCHITECTURE](../ARCHITECTURE.md) · [Core interfaces](core-interfaces.md) · ADR-0004 (monorepo layout)

The prompt suggested a TypeScript-style layout (`apps/`, `packages/`) and invited a better one if it is architecturally more correct (prompt §76). The backend is Go, so we use the Go-idiomatic layout. `cmd/` holds the entry points, `internal/` holds code the compiler forbids other modules from importing, and `pkg/` holds the small, deliberately public API. Providers, distributions and add-ons stay visible at the top level under `plugins/`, as the prompt intended.

## 1. Tree

```
/
├── cmd/
│   ├── farvater-server/                 # server binary: `api | worker | all | migrate | admin | version`
│   └── farvater/                    # CLI: thin client over the public API
│
├── internal/                       # private application code (import-restricted by the Go compiler)
│   ├── domain/                     # entities, value objects, state machines, typed errors — no I/O
│   ├── app/                        # use cases (application services) + ports (interfaces) they depend on
│   ├── engine/                     # provisioning engine: planner, dag, scheduler, executor, leases, retry, rollback, events
│   ├── api/                        # HTTP server: Huma operations on chi, middleware, SSE, problem+json, rate limiting
│   ├── authn/                      # sessions, API keys, password hashing, CSRF
│   ├── authz/                      # permission catalogue, roles, bindings, evaluator
│   ├── tenancy/                    # TenantScope, RLS session settings
│   ├── secrets/                    # envelope encryption, key providers, redaction
│   ├── catalog/                    # catalog loader, signature verification, resolver
│   ├── compat/                     # compatibility engine (rules over catalog + spec + facts)
│   ├── recommend/                  # decision engine (Auto Mode), presets, explanations
│   ├── policy/                     # guardrails (Community) + extension point for the ee policy engine
│   ├── health/                     # health engine and built-in probes
│   ├── audit/                      # append-only, hash-chained audit log
│   ├── notify/                     # notification channels (in-app, email, webhook, Slack, Telegram)
│   ├── webhooks/                   # outbound webhook delivery (Standard Webhooks)
│   ├── entitlements/               # EntitlementService (edition/features/limits, license verification)
│   ├── errcatalog/                 # error codes → i18n keys, remediation, HTTP status mapping
│   ├── config/                     # configuration loading and validation
│   ├── telemetry/                  # slog, OpenTelemetry, Prometheus metrics, health endpoints
│   └── adapters/                   # implementations of ports
│       ├── postgres/               # sqlc-generated queries, repositories, tx manager, outbox
│       ├── queue/                  # River integration (job args, workers, periodic jobs)
│       ├── ssh/                    # NodeHandle over SSH: auth methods, jump hosts, host keys, SFTP, pooling
│       ├── nodehelper/             # client for the signed node helper binary (Phase 3)
│       ├── helm/                   # HelmService over the embedded Helm SDK
│       ├── kube/                   # client-go helpers, server-side apply, waiters
│       ├── kms/                    # KeyProvider implementations (local; OpenBao/Vault Transit; cloud KMS in ee)
│       └── httpx/                  # outbound HTTP client with SSRF guard
│
├── pkg/                            # PUBLIC Go API (semver-stable once v1)
│   ├── sdk/                        # plugin interfaces: provider, distribution, addon, task, probe, node, errors
│   ├── spec/                       # ClusterSpec types (apiVersion/kind) + JSON Schema generation
│   └── client/                     # typed Go API client over shared API types (CLI, Terraform provider, integrations)
│
├── plugins/
│   ├── providers/
│   │   ├── simulated/              # TEST/DEV ONLY (refused in production profile)
│   │   ├── baremetal/              # SSH-reachable declared hosts (also "existing infrastructure")
│   │   └── hetzner/ aws/ gcp/ azure/   # Phase 7
│   ├── distributions/
│   │   ├── kubeadm/                # Phase 3–4
│   │   └── k3s/ rke2/              # Phase 7
│   ├── osfamilies/
│   │   ├── debian/                 # Ubuntu, Debian
│   │   └── rhel/                   # Rocky, AlmaLinux (after Phase 3)
│   └── addons/
│       ├── generic/                # generic chart-based add-on implementation (reads addon.yaml)
│       ├── cilium/ calico/ flannel/
│       ├── gateway-api-crds/ envoy-gateway/
│       ├── kube-vip/ metallb/
│       ├── cert-manager/ metrics-server/
│       ├── local-path/ nfs/ longhorn/ rook-ceph/
│       ├── kube-prometheus-stack/ loki/ alloy/ tempo/ opentelemetry/
│       ├── velero/ argocd/ flux/
│       └── …                       # each: addon.yaml, values.yaml.tmpl, schema.json, optional Go hooks, docs.md, tests
│
├── catalog/                        # DATA: versions, compatibility rules, presets, estimates (see auto-mode-and-catalog.md)
├── api/openapi/                    # generated OpenAPI 3.1 document (committed) + vacuum/oasdiff config
├── migrations/                     # goose SQL migrations (core)
├── web/                            # React SPA (pnpm workspace)
│   ├── src/                        # see ui-architecture.md
│   ├── tests/                      # Playwright
│   └── package.json
├── ee/                             # (later) enterprise code under commercial license, build tag `ee`
│   ├── LICENSE
│   ├── internal/…                  # sso, scim, policy engine, approvals, fleet, …
│   ├── migrations/
│   └── web/                        # enterprise UI modules (lazy-loaded, entitlement-gated)
├── deploy/
│   ├── docker/                     # Dockerfiles (server, node-helper, test container-node images)
│   ├── compose/                    # compose.dev.yaml, compose.yaml (+ .env.example)
│   └── helm/farvater/        # Helm chart of the platform
├── test/
│   ├── e2e/                        # tiered E2E harness (simulated / container-nodes / real VMs)
│   ├── integration/                # cross-package integration suites
│   └── fixtures/                   # isolated fixtures: container-node images, sample specs, recorded provider responses
├── hack/                           # dev scripts: codegen, test-cluster (kind/k3d), catalog update helpers
├── docs/                           # documentation (en + ru), ADRs, guides
├── .github/workflows/              # CI/CD
├── Makefile                        # dev | build | test | lint | e2e | test-cluster | generate | release
├── go.mod / go.sum                 # single Go module
├── LICENSE                         # Apache-2.0 (core)
└── README.md / README.ru.md
```

## 2. Dependency rules

The rules are enforced by `depguard`/import-boundary checks in golangci-lint and by the Go `internal/` mechanism:

```
cmd/*            → may import everything (composition root: wires adapters + plugins into the registry)
internal/api     → internal/app, internal/authn, internal/errcatalog, pkg/spec          (never adapters directly)
internal/app     → internal/domain, internal/engine, ports (interfaces), pkg/sdk, pkg/spec
internal/engine  → internal/domain, pkg/sdk                                              (never plugins, never adapters)
internal/domain  → stdlib only (+ tiny utility libs)
internal/adapters/* → implement ports; may import third-party SDKs (pgx, River, x/crypto/ssh, Helm, client-go)
plugins/*        → pkg/sdk, pkg/spec, small shared helpers (never internal/app or internal/engine)
pkg/*            → stdlib + minimal deps (public API must stay light)
ee/*             → may import internal/* through declared extension points only; core never imports ee/*
web/             → talks to the server only through the generated API client
```

Breaking a rule fails CI. This keeps the core provider-agnostic (prompt §99 "Dependency inversion") and lets a future out-of-process plugin protocol wrap `pkg/sdk` without changing the engine.

## 3. Build variants

| Build | Command | Includes |
|---|---|---|
| Community | `make build` (`go build ./cmd/...`) | core + community plugins |
| Enterprise | `make build EDITION=ee` (`-tags ee`) | + `ee/` packages and migrations; features unlocked by license entitlements |
| Dev | `make dev` | Community + simulated provider + dev profile |

Enterprise code registers its implementations of extension points in `init()` functions inside files with the build tag `ee`. Community builds contain none of that code. The edition check at runtime goes through `EntitlementService`, never through `if edition == …` scattered in code (prompt §218).

## 4. Code generation

| Generator | Input | Output | Check |
|---|---|---|---|
| Go API definitions (Huma) → OpenAPI 3.1 | `internal/api` operations + shared types | `api/openapi/openapi.yaml` (committed) | `make generate && git diff --exit-code`; oasdiff (no breaking changes in v1); vacuum lint |
| OpenAPI → TypeScript types | `api/openapi/openapi.yaml` | `web/src/lib/api/gen` | `make generate && git diff --exit-code` in CI |
| sqlc | `internal/adapters/postgres/queries/*.sql` + migrations | typed Go queries | same |
| JSON Schema from Go types | `pkg/spec` | `api/openapi/schemas/cluster-spec.json`, catalog schemas | same |
| Error catalog | `internal/errcatalog/*.yaml` | Go constants + `web/src/i18n/*/errors.json` | same |

## 5. Conventions

- Go: `gofumpt`, `golangci-lint` v2 config in repo, errors wrapped with `%w`, `context.Context` first, no global mutable state except the registry built at start-up, table-driven tests next to code (`*_test.go`), integration tests behind the `integration` build tag.
- TypeScript: `strict`, Oxlint + Prettier, no `any` without justification, components in PascalCase, hooks `useX`.
- Commits: Conventional Commits (`feat:`, `fix:`, `docs:` …), signed-off (DCO) for contributions.
- Docs: English `*.md` plus Russian `*.ru.md` next to it. Both are updated in the same PR.
- No `TODO`/`FIXME` in production-critical code without an issue link and explanation (prompt §82). The linter checks the format `TODO(#123): …`.
