# Структура репозитория

> Статус: **Предложено** (Phase 0). Язык: [English](repository-structure.md) · Русский
>
> Связанные документы: [ARCHITECTURE](../ARCHITECTURE.ru.md) · [Ключевые интерфейсы](core-interfaces.ru.md) · ADR-0004 (структура монорепозитория)

Промпт предлагал структуру в стиле TypeScript (`apps/`, `packages/`) и допускал более удачный вариант, если он архитектурно правильнее (промпт §76). Бэкенд написан на Go, поэтому мы используем идиоматичную для Go структуру. В `cmd/` лежат точки входа, в `internal/` — код, импорт которого из других модулей запрещает компилятор, а в `pkg/` — небольшой, намеренно публичный API. Провайдеры, дистрибутивы и аддоны остаются на виду, на верхнем уровне в `plugins/`, как и задумывалось в промпте.

## 1. Дерево

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
│   ├── api/                        # HTTP server: generated handlers, middleware, SSE, problem+json, rate limiting
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
│   └── client/                     # generated Go API client (CLI, Terraform provider, integrations)
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
├── api/openapi/                    # OpenAPI 3.1 spec split by domain; bundled + linted in CI
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

## 2. Правила зависимостей

Правила обеспечиваются проверками `depguard`/границ импорта в golangci-lint и механизмом `internal/` в Go:

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

Нарушение любого правила роняет CI. Так ядро остаётся независимым от провайдеров (промпт §99, «инверсия зависимостей», Dependency inversion), а будущий внепроцессный протокол плагинов сможет обернуть `pkg/sdk` без изменений в движке.

## 3. Варианты сборки

| Сборка | Команда | Что входит |
|---|---|---|
| Community | `make build` (`go build ./cmd/...`) | ядро + плагины Community |
| Enterprise | `make build EDITION=ee` (`-tags ee`) | + пакеты и миграции из `ee/`; функции разблокируются правами (entitlements) лицензии |
| Dev | `make dev` | Community + симулированный провайдер + профиль dev |

Enterprise-код регистрирует свои реализации точек расширения в функциях `init()` в файлах с тегом сборки `ee`. В сборках Community этого кода нет вовсе. Проверка редакции во время выполнения идёт через `EntitlementService`, а не через разбросанные по коду `if edition == …` (промпт §218).

## 4. Генерация кода

| Генератор | Вход | Выход | Проверка |
|---|---|---|---|
| OpenAPI → Go-сервер/клиент | `api/openapi/` | `internal/api/gen`, `pkg/client` | `make generate && git diff --exit-code` в CI |
| OpenAPI → типы TypeScript | `api/openapi/` | `web/src/lib/api/gen` | то же |
| sqlc | `internal/adapters/postgres/queries/*.sql` + миграции | типизированные запросы на Go | то же |
| JSON Schema из типов Go | `pkg/spec` | `api/openapi/schemas/cluster-spec.json`, схемы каталога | то же |
| Каталог ошибок | `internal/errcatalog/*.yaml` | константы Go + `web/src/i18n/*/errors.json` | то же |

## 5. Соглашения

- Go: `gofumpt`, конфигурация `golangci-lint` v2 в репозитории, ошибки оборачиваются через `%w`, `context.Context` — первым параметром, никакого глобального изменяемого состояния, кроме реестра, собираемого при запуске, табличные тесты рядом с кодом (`*_test.go`), интеграционные тесты — за тегом сборки `integration`.
- TypeScript: `strict`, ESLint + Prettier, никаких `any` без обоснования, компоненты в PascalCase, хуки — `useX`.
- Коммиты: Conventional Commits (`feat:`, `fix:`, `docs:` …), для внешних вкладов — с подписью signed-off (DCO).
- Документация: английский `*.md` и рядом с ним русский `*.ru.md`. Оба обновляются в одном PR.
- Никаких `TODO`/`FIXME` в критичном для production коде без ссылки на issue и пояснения (промпт §82). Линтер проверяет формат `TODO(#123): …`.
