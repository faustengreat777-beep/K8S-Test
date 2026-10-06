# Проектирование API

> Статус: **Предложено** (Phase 0). Язык: [English](api-design.md) · Русский
>
> Связанные документы: [ARCHITECTURE](../ARCHITECTURE.ru.md) · [Движок провижининга](provisioning-engine.ru.md) · [Модель безопасности](../security/SECURITY_MODEL.ru.md) · ADR-0005 (стиль и контракт API)

Farvater построен по принципу **API-first** (промпт §40, §57). Веб-интерфейс, CLI (`farvater`), будущий провайдер Terraform/OpenTofu и автоматизация на стороне клиентов — все они клиенты одного и того же публичного HTTP API. Ни один клиент не содержит бизнес-логики. Ни один клиент не обращается к инфраструктуре напрямую.

## 1. Стиль и контракт

| Аспект | Решение |
|---|---|
| Протокол | HTTPS, JSON (`application/json`), UTF-8 |
| Контракт | **OpenAPI 3.1**, формируется **по принципу contract-first на Go** с помощью **Huma v2** поверх роутера chi: из типизированных определений операций (структуры запросов и ответов с тегами валидации) генерируются документ OpenAPI 3.1, JSON Schema, валидация запросов и ошибки по RFC 9457. Сгенерированный `api/openapi/openapi.yaml` **хранится в репозитории**: в каждом PR виден diff контракта, **oasdiff** блокирует ломающие изменения в v1, а **vacuum** проверяет его линтером. TypeScript-клиент генерируется из закоммиченного документа (`openapi-typescript` + `openapi-fetch`); CLI использует `pkg/client` — тонкий типизированный клиент поверх общих типов запросов и ответов (или клиент oapi-codegen, если он понадобится внешнему потребителю). Задача CI завершается ошибкой, если закоммиченная спецификация или сгенерированные клиенты устарели. |
| Базовый путь / версионирование | `/api/v1`. В пределах v1 допускаются только **аддитивные** изменения (новые эндпоинты, новые необязательные поля, новые значения enum, которые клиенты обязаны корректно принимать). Ломающие изменения → `/api/v2`, который работает параллельно с v1. Устаревшие эндпоинты отдают заголовки `Deprecation` и `Sunset` (RFC 9745 / RFC 8594) как минимум в течение двух минорных релизов. |
| Именование | Существительные во множественном числе, пути в kebab-case, поля JSON в camelCase, идентификаторы — строки UUIDv7, метки времени — RFC 3339 в UTC |
| Длительная работа | `202 Accepted` + ресурс `Operation` + `Location: /api/v1/operations/{id}` |
| Ошибки | RFC 9457 `application/problem+json` со стабильными кодами (раздел 6) |
| Реальное время | Server-Sent Events для отображения прогресса операций (раздел 7) |
| Документация | Справочник, сгенерированный из спецификации (доступен по `/api/docs` в продукте и публикуется с каждым релизом); `docs/API.md` — руководство для людей |

Почему code-first с Huma, а не генерация сервера по спецификации (spec-first): на октябрь 2026 года генераторы серверного кода на Go поддерживают OpenAPI 3.1 лишь частично (oapi-codegen называет свою поддержку 3.1 «начальной», ogen требует препроцессоров). Huma нативно выдаёт корректную спецификацию 3.1 из типизированного кода на Go, а закоммиченная спецификация вместе с блокировкой ломающих изменений позволяет проверить контракт на ревью до слияния (ADR-0005). Кроме того, каждый ответ в интеграционных тестах проверяется на соответствие закоммиченной спецификации.

## 2. Модель ресурсов и эндпоинты (v1)

Привязка к тенанту: **коллекции** располагаются под родительским ресурсом (`/organizations/{orgId}/…`, `/projects/{projectId}/…`), поэтому принадлежность тенанту видна в URL и в журналах доступа. **Элементы** имеют глобально уникальные идентификаторы и адресуются напрямую (`/clusters/{clusterId}`), что упрощает работу с CLI. При каждом вызове авторизация определяет организацию и проект элемента и проверяет разрешение субъекта (principal) именно в этой области. Сам по себе URL никогда не даёт доступа.

### 2.1 Идентификация, тенанты, доступ

| Метод и путь | Назначение | Разрешение |
|---|---|---|
| `POST /api/v1/auth/login` · `POST /auth/logout` | Локальный вход и выход (сессионная cookie) | публичный доступ / аутентифицированный пользователь |
| `GET /api/v1/me` · `GET /me/sessions` · `DELETE /me/sessions/{id}` | Текущий пользователь, список сессий, отзыв сессии | аутентифицированный пользователь |
| `GET/POST /api/v1/organizations` · `GET/PATCH /organizations/{orgId}` | Организации (в Community — одна) | `organization:read`/`organization:manage` |
| `GET/POST /organizations/{orgId}/projects` · `GET/PATCH/DELETE /projects/{projectId}` | Проекты | `project:*` |
| `GET/POST /organizations/{orgId}/members` · `DELETE …/members/{userId}` | Участники | `organization:manage` |
| `GET/POST /organizations/{orgId}/teams` · `…/teams/{teamId}/members` | Команды | `team:*` |
| `GET/POST /organizations/{orgId}/roles` · `GET/PATCH/DELETE /roles/{roleId}` | Встроенные и пользовательские роли (пользовательские — ee) | `role:*` |
| `GET/POST /organizations/{orgId}/role-bindings` · `DELETE /role-bindings/{id}` | Назначение ролей в области организации, проекта или кластера | `role:bind` |
| `GET/POST /organizations/{orgId}/api-keys` · `DELETE /api-keys/{id}` | API-ключи (секрет возвращается **один раз** при создании) | `apikey:*` |

### 2.2 Учётные данные и инфраструктура

| Метод и путь | Назначение | Разрешение |
|---|---|---|
| `GET/POST /organizations/{orgId}/credentials` · `GET/PATCH/DELETE /credentials/{id}` | Хранилище учётных данных. Секретные поля доступны **только для записи**; при чтении возвращаются метаданные (тип, отпечаток, время последнего использования, срок действия) | `credentials:read/create/delete` |
| `POST /credentials/{id}/rotate` | Ротация (новый материал; старый остаётся доступным только для расшифровки, пока зависящие от него кластеры не будут обновлены) | `credentials:rotate` |
| `GET/POST /organizations/{orgId}/provider-accounts` · `GET/PATCH/DELETE /provider-accounts/{id}` | Настроенные подключения к инфраструктуре | `provider:*` |
| `POST /provider-accounts/{id}/verify` | Проверка конфигурации и учётных данных (асинхронно → Operation типа `preflight`) | `provider:read` |
| `GET /provider-accounts/{id}/inventory` | Регионы, размеры, образы, цены (если провайдер их поддерживает) | `provider:read` |
| `GET /api/v1/providers` | Зарегистрированные типы провайдеров, их возможности, JSON Schema конфигураций | аутентифицированный пользователь |

### 2.3 Кластеры

| Метод и путь | Назначение | Разрешение |
|---|---|---|
| `GET /projects/{projectId}/clusters` | Список (фильтры: `phase`, `health`, `environment`, `label`; сортировка; пагинация) | `cluster:read` |
| `POST /projects/{projectId}/clusters` | Создаёт **запись** о кластере из ClusterSpec (фаза `DRAFT`); тело может быть в YAML (`application/yaml`) или JSON | `cluster:create` |
| `GET /clusters/{id}` | Кластер со статусом, состоянием (health), эндпоинтами и текущей ревизией спецификации | `cluster:read` |
| `PUT /clusters/{id}/spec` | Сохраняет новую ревизию желаемой спецификации (требует `If-Match`) — инфраструктуру не меняет | `cluster:update` |
| `GET /clusters/{id}/spec-revisions` · `GET …/spec-revisions/{rev}` · `GET …/spec-revisions/{rev}/diff?against={rev2}` | История и diff (промпт §97) | `cluster:read` |
| `POST /clusters/{id}/plan` | Пробный прогон (dry run): валидация → разрешение версий → совместимость → политики → план. Возвращает план, diff, необратимые шаги, оценки. **Без побочных эффектов** | `cluster:read` |
| `POST /clusters/{id}/deploy` | Применяет последнюю (или указанную) ревизию спецификации: `create` для нового кластера, иначе `apply` → `202` Operation. Тело: `{revision, planHash, confirmIrreversible}` | `deployment:create` + `cluster:update` |
| `POST /clusters/{id}/upgrade` | Процесс обновления (upgrade) Kubernetes (`targetVersion`) → `202` Operation | `cluster:upgrade` |
| `GET /clusters/{id}/upgrade-advisor?target=1.38` | Оценка готовности, блокеры, предупреждения, используемые устаревшие API | `cluster:read` |
| `POST /clusters/{id}/scale` | Изменяет размеры пулов рабочих узлов или добавляет и удаляет объявленные хосты → `202` | `cluster:scale` |
| `DELETE /clusters/{id}` | Уничтожение (требует `confirm: <cluster-name>`) → `202` Operation типа `destroy` | `cluster:delete` |
| `GET /clusters/{id}/kubeconfig` | Скачивание kubeconfig. По умолчанию — **короткоживущий клиентский сертификат, привязанный к пользователю** (TTL настраивается), скачивание фиксируется в аудите; административный kubeconfig — только с `cluster:admin-kubeconfig` | `cluster:connect` |
| `GET /clusters/{id}/health` | Состояние по каждой пробе + агрегированное состояние | `cluster:read` |
| `GET /clusters/{id}/nodes` · `GET …/nodes/{nodeId}` | Инвентарь узлов с фактами, условиями (conditions) и утилизацией (по метрикам, если они установлены) | `node:read` |
| `POST /clusters/{id}/nodes/{nodeId}/cordon` · `/uncordon` · `/drain` · `/restart` · `/replace` · `DELETE …/nodes/{nodeId}` | Действия с узлами (drain/replace/remove → операции) | `node:cordon` / `node:drain` / `node:delete` |
| `GET /clusters/{id}/addons` · `POST /clusters/{id}/addons` · `PATCH/DELETE /clusters/{id}/addons/{addonId}` | Управление аддонами (установка/обновление/удаление → операции) | `addon:install/update/delete` |
| `GET /clusters/{id}/backups` · `POST /clusters/{id}/backups` · `POST /clusters/{id}/restore` | Резервное копирование и восстановление (фаза 6) | `cluster:backup` / `cluster:restore` |
| `POST /clusters/{id}/rotate-certificates` · `/rotate-credentials` | Ротации → операции | `cluster:update` |
| `POST /projects/{projectId}/clusters:import` | Импорт существующего кластера по kubeconfig (фаза 6) | `cluster:create` |

### 2.4 Операции (в продукте — «Deployments»)

| Метод и путь | Назначение | Разрешение |
|---|---|---|
| `GET /clusters/{id}/operations` · `GET /projects/{projectId}/operations` | Списки | `deployment:read` |
| `GET /operations/{id}` | Статус, сводка плана, тайминги, отчёт об ошибке | `deployment:read` |
| `GET /operations/{id}/tasks` | Дерево задач со статусами, попытками, длительностью | `deployment:read` |
| `GET /operations/{id}/events` | Живой поток **SSE** с повторной доставкой (`Last-Event-ID`) | `deployment:read` |
| `GET /operations/{id}/logs?level=&task=&node=&q=&pageToken=` | Структурированные логи с поиском и фильтрацией (JSON или скачивание в `text/plain`) | `deployment:read` |
| `POST /operations/{id}/cancel` · `/resume` · `/retry` (опционально `{taskKey}`) · `/rollback` | Управляющие действия | `deployment:cancel` / `deployment:retry` |

### 2.5 Каталог, валидация, режим Auto (Auto Mode)

| Метод и путь | Назначение |
|---|---|
| `GET /api/v1/catalog` | Версия и канал каталога |
| `GET /api/v1/catalog/kubernetes-versions` | Минорные версии со статусом (current/supported/deprecated/blocked), патчи, EOL |
| `GET /api/v1/catalog/addons?category=` · `GET /catalog/addons/{id}` | Метаданные аддонов, версии, схема конфигурации, плюсы и минусы |
| `GET /api/v1/catalog/presets` | Пресеты (Minimal, Development, Staging, Production, High Availability, Edge, GPU, AI/ML, Custom) |
| `GET /api/v1/schemas/cluster-spec` | JSON Schema для ClusterSpec (для редакторов Monaco/YAML и для CLI) |
| `POST /api/v1/validate` | Валидация ClusterSpec: схема + совместимость + защитные ограничения (guardrails) и политики → список замечаний (без побочных эффектов) |
| `POST /api/v1/recommendations` | Режим Auto: ответы (окружение, размер, уровень доступности, тип нагрузки, бюджет, требования соответствия (compliance), провайдер) → рекомендуемая ClusterSpec + пояснения по каждому полю + предупреждения + оценки |
| `POST /api/v1/preflight` | Предварительные проверки (preflight) на реальных хостах или у провайдера → `202` Operation типа `preflight` с результатами по каждой проверке |

### 2.6 Шаблоны, аудит, вебхуки, уведомления, поиск

| Метод и путь | Назначение |
|---|---|
| `GET/POST /organizations/{orgId}/templates` · `GET /templates/{id}` · `POST /templates/{id}/versions` · `PATCH /templates/{id}/versions/{v}` (status) | Шаблоны кластеров с версиями, наследованием и статусом (промпт §33, §191–192) |
| `POST /templates/{id}/versions/{v}/instantiate` | Создание черновика кластера из шаблона |
| `GET /organizations/{orgId}/audit-events?actor=&action=&target=&cluster=&project=&ip=&from=&to=&result=` · `GET …/audit-events:export?format=json|csv` | Поиск и экспорт событий аудита |
| `GET/POST /organizations/{orgId}/webhooks` · `GET /webhooks/{id}/deliveries` · `POST /webhooks/{id}/deliveries/{deliveryId}/replay` · `POST /webhooks/{id}/test` | Вебхуки |
| `GET /me/notifications` · `POST /me/notifications/{id}/read` · `GET/POST /organizations/{orgId}/notification-channels` | Уведомления |
| `GET /api/v1/search?q=&types=clusters,nodes,operations,addons,users` | Глобальный поиск (промпт §93) с фильтрацией результатов по правам доступа |

### 2.7 Платформа

| Путь | Назначение |
|---|---|
| `GET /healthz` | Liveness (процесс запущен) — без аутентификации, вне `/api` |
| `GET /readyz` | Readiness (БД доступна, миграции применены, есть heartbeat воркера для роли `worker`) |
| `GET /metrics` | Метрики Prometheus (отдельный listener/порт, по умолчанию не публикуется) |
| `GET /api/v1/version` | Версия сервера, редакция, версия каталога, версия API |

### 2.8 Enterprise (ee, последующие фазы)

`/api/v1/policies`, `/policy-bindings`, `/change-requests` (+ `/approve`, `/reject`, `/execute`), `/compliance`, `/fleet`, `/cluster-groups`, `/integrations`, `/identity-providers`, `/service-accounts`, `/incidents`, `/budgets`, `/quotas`, `/scim/v2/*` (SCIM 2.0, RFC 7643/7644). Они следуют тем же соглашениям. Доступность определяют лицензионные права (entitlements), поэтому сервер редакции Community возвращает `403` с кодом `FEATURE_NOT_LICENSED`.

## 3. Представления ресурсов

Cluster (сокращённо):

```json
{
  "id": "0199a7c2-5b1e-7c3a-9f0e-2d4b6a8c1e3f",
  "name": "production",
  "projectId": "0199a7c2-…",
  "environment": "production",
  "phase": "READY",
  "health": "HEALTHY",
  "kubernetesVersion": "1.37.1",
  "distribution": "kubeadm",
  "apiEndpoint": "https://10.0.0.10:6443",
  "desiredRevision": 4,
  "appliedRevision": 4,
  "activeOperationId": null,
  "nodeCounts": { "controlPlane": 3, "worker": 3 },
  "labels": { "team": "payments" },
  "createdAt": "2026-10-06T12:30:00Z",
  "updatedAt": "2026-10-06T12:47:12Z",
  "etag": "\"17\""
}
```

Operation (сокращённо):

```json
{
  "id": "0199a7c3-…",
  "clusterId": "0199a7c2-…",
  "type": "create",
  "status": "RUNNING",
  "progress": { "tasksTotal": 64, "tasksSucceeded": 41, "tasksRunning": 2, "tasksFailed": 0, "percent": 66 },
  "currentPhase": "INSTALLING",
  "estimatedSecondsRemaining": 310,
  "irreversibleSteps": [],
  "error": null,
  "links": { "events": "/api/v1/operations/0199a7c3-…/events", "tasks": "/api/v1/operations/0199a7c3-…/tasks" },
  "startedAt": "2026-10-06T12:31:02Z"
}
```

Plan (`POST /clusters/{id}/plan`, сокращённо):

```json
{
  "planHash": "sha256:9c1…",
  "changes": [
    { "op": "add", "kind": "nodes", "summary": "+ 3 control plane nodes" },
    { "op": "add", "kind": "nodes", "summary": "+ 3 worker nodes" },
    { "op": "add", "kind": "kubernetes", "summary": "+ Kubernetes 1.37.1 (kubeadm)" },
    { "op": "add", "kind": "addon", "id": "cilium", "summary": "+ Cilium 1.x.y" }
  ],
  "irreversibleSteps": [],
  "findings": [ { "severity": "WARNING", "code": "NO_BACKUP_DESTINATION", "path": "/spec/backup" } ],
  "estimate": { "durationSeconds": { "min": 720, "max": 1080 }, "cost": { "available": false, "reason": "PROVIDER_HAS_NO_PRICING" } },
  "tasks": 64
}
```

Значения в примерах условные. Реальные версии берутся из каталога.

## 4. Пагинация, фильтрация, сортировка

- **Курсорная пагинация** во всех эндпоинтах-списках: `?pageSize=50&pageToken=…`. Ответ имеет вид `{ "items": [...], "nextPageToken": "…" }`. Токены непрозрачны (зашифрованные ключи сортировки + хеш фильтра); токен, переданный с другими фильтрами, отклоняется с ошибкой `400 INVALID_PAGE_TOKEN`. Максимальный `pageSize` — 200. Общее количество возвращается только по запросу (`?includeTotal=true`), потому что подсчёт на больших объёмах обходится дорого.
- **Фильтрация** — через явные, задокументированные параметры запроса, свои для каждой коллекции (`phase`, `health`, `environment`, `label=key=value`, `createdAfter`, …). Языка произвольных фильтров в v1 нет; выражение `filter=` на основе CEL может появиться в одной из следующих версий.
- **Сортировка** — через `orderBy=createdAt desc,name` по разрешённому набору полей, своему для каждой коллекции.

## 5. Идемпотентность, конкурентный доступ, асинхронность

- Заголовок **Idempotency-Key** (UUID или любая непрозрачная строка длиной ≤ 255 символов) **обязателен** для вызовов из CLI и автоматизации к `POST`-эндпоинтам, которые создают ресурсы или запускают операции. Веб-интерфейс отправляет его при каждом таком вызове. Сервер хранит `(org, principal, key) → request hash + response` в течение 24 часов:
  - тот же ключ + тот же запрос → повторно возвращается исходный ответ (`Idempotent-Replayed: true`);
  - тот же ключ + другой запрос → `422 IDEMPOTENCY_KEY_REUSED`;
  - запрос ещё выполняется → `409 IDEMPOTENCY_IN_PROGRESS`.
- **Оптимистичная блокировка**: изменяемые ресурсы возвращают `ETag`. `PUT`/`PATCH` требуют `If-Match` (иначе `428 PRECONDITION_REQUIRED`); при несовпадении → `412 PRECONDITION_FAILED` с текущей версией.
- **Асинхронные задания**: все эндпоинты, меняющие инфраструктуру, возвращают `202` с Operation. Синхронные эндпоинты никогда не блокируются в ожидании инфраструктуры.
- **Конфликты операций**: попытка запустить операцию, пока на кластере выполняется другая изменяющая операция → `409 OPERATION_IN_PROGRESS` с `activeOperationId`.

## 6. Ошибки

`application/problem+json` (RFC 9457) с расширениями:

```json
{
  "type": "https://docs.farvater.io/errors/K8S_JOIN_API_UNREACHABLE",
  "title": "Worker node failed to join the cluster",
  "status": 409,
  "code": "K8S_JOIN_API_UNREACHABLE",
  "detail": "Worker node node-04 cannot reach the Kubernetes API server on port 6443.",
  "instance": "/api/v1/operations/0199a7c3-…",
  "requestId": "req_01J…",
  "location": { "operationId": "0199a7c3-…", "taskKey": "k8s.worker.join/node-04", "node": "node-04", "address": "10.0.0.14" },
  "remediation": [
    "Check firewall rules between node-04 and the control-plane endpoint on port 6443.",
    "Check routing from node-04 to 10.0.0.10.",
    "Verify the API server address in the cluster spec."
  ],
  "actions": ["retry", "resume", "view-logs"],
  "errors": []
}
```

- `code` стабилен и задокументирован (каталог ошибок). `title`, `detail` и `remediation` локализуются по `Accept-Language` (en, ru).
- Ошибки валидации → `422` с `errors: [{ "path": "/spec/networking/cni/provider", "code": "UNSUPPORTED_VALUE", "message": "…" }]`.
- Аутентификация и авторизация: `401 UNAUTHENTICATED`, `403 FORBIDDEN` (с именем недостающего разрешения, но никогда — с подробностями о ресурсах, которые вызывающий не может видеть; на неизвестные или чужие идентификаторы возвращается `404`, чтобы исключить перебор).
- Что никогда не попадает в ошибки: секреты, трассировки стека, внутренние имена хостов платформы, необработанные ошибки SQL. Внутреннюю причину сервер записывает в лог с тем же `requestId`.
- Общее сообщение «Что-то пошло не так» (Something went wrong) используется только для действительно непредвиденных сбоев (`500 INTERNAL`) и всегда сопровождается `requestId` (промпт §83).

## 7. Обновления в реальном времени (SSE)

`GET /api/v1/operations/{id}/events` с `Accept: text/event-stream`:

```
id: 18342
event: task-status
data: {"taskKey":"addon/cilium.install","status":"SUCCEEDED","durationMs":48211}

id: 18343
event: log
data: {"ts":"2026-10-06T12:31:08Z","level":"INFO","taskKey":"node/cp-1/runtime.install","node":"cp-1","message":"containerd installed"}

: heartbeat

event: end
data: {"status":"SUCCEEDED"}
```

- Типы событий: `log`, `task-status`, `operation-status`, `progress`, `end`.
- Продолжение после переподключения — через стандартный заголовок `Last-Event-ID` (или `?after=` для клиентов, которые не могут задавать заголовки). Сервер повторно отдаёт пропущенные события из базы данных.
- Аутентификация: сессионная cookie (`EventSource` с того же origin) или `Authorization: Bearer` для потокового клиента CLI.
- Также доступен `GET /api/v1/events?clusterId=` — поток изменений операций и состояния на уровне проекта или кластера для дашбордов. Для него действуют ограничение частоты запросов и лимит на пользователя.

## 8. Аутентификация и авторизация

- **Браузер**: `POST /auth/login` устанавливает cookie `__Host-session` (`HttpOnly; Secure; SameSite=Lax; Path=/`). Небезопасные методы требуют заголовок `X-CSRF-Token` (synchronizer token, привязанный к сессии; токен выдаётся в ответе `GET /me`). Сессия ротируется при входе и при изменении привилегий.
- **CLI и автоматизация**: API-ключи `Authorization: Bearer fvt_…` (префикс для поиска ключа + секрет, сверяемый с хешем). Позже: OAuth 2.0 device flow через настроенного OIDC-провайдера (ee) и сервисные аккаунты.
- **Авторизация** проверяется на уровне приложения для каждого запроса (`Authorizer.Authorize(principal, permission, resource)`) в области организации, проекта или кластера. Разрешения имеют вид `resource:verb` (см. раздел об авторизации в [Модели безопасности](../security/SECURITY_MODEL.ru.md)). API-ключи могут только сужать права своего владельца, но никогда не расширять их.
- **Скачивание kubeconfig** само по себе является операцией, фиксируемой в аудите, и по умолчанию выдаёт короткоживущие учётные данные.

## 9. Вебхуки

- Типы событий (промпт §68): `deployment.started`, `deployment.completed`, `deployment.failed`, `cluster.created`, `cluster.deleted`, `cluster.upgraded`, `backup.completed`, `backup.failed`. Можно добавить и другие, например `cluster.health_changed`, `operation.paused`, `credential.expiring`.
- Формат соответствует спецификации **Standard Webhooks**. Заголовки `webhook-id`, `webhook-timestamp`, `webhook-signature` (`v1,` + HMAC-SHA256 от `id.timestamp.body` с секретом, отдельным для каждого эндпоинта). JSON-тело содержит `type`, `timestamp`, `data` (публичное представление, из которого удалены чувствительные данные).
- Доставка выполняется из транзакционного outbox через задания River: семантика at-least-once (как минимум один раз), повторы с экспоненциальной задержкой в течение до 24 ч, затем статус `dead`. Доступны история доставок, ручная повторная отправка и тестовый эндпоинт. Целевой URL проверяется защитой от SSRF при сохранении и при каждой доставке.
- Получатели должны отбрасывать дубликаты по `webhook-id`.

## 10. Ограничение частоты запросов и защита от злоупотреблений

- Корзины токенов (token buckets) по **API-ключу / пользователю**, по **организации** и по **IP-адресу источника** (для эндпоинтов без аутентификации, особенно для входа). Отдельные бюджеты для читающих и изменяющих эндпоинтов и более жёсткие лимиты для дорогих (`plan`, `preflight`, `kubeconfig`).
- Ответы: `429 RATE_LIMITED` с заголовками `Retry-After` и `RateLimit-*` (IETF RateLimit header fields).
- Реализация: в Community/MVP — ограничитель внутри процесса на каждой реплике. Общее хранилище (на базе PostgreSQL или Valkey для больших HA-инсталляций) — когда несколько реплик API работают за балансировщиком нагрузки.
- Защита входа: нарастающая задержка (backoff) по аккаунту и по IP, обобщённые сообщения об ошибках («неверный email или пароль»), аудит неудачных попыток.

## 11. Соответствие команд CLI и API

CLI использует сгенерированный клиент на Go. Команды соответствуют вызовам API один к одному и выводят те же коды ошибок и рекомендации по исправлению:

```
farvater login --server https://… [--api-key-stdin]
farvater cluster create -f cluster.yaml [--project payments]      # POST …/clusters (DRAFT)
farvater cluster plan production                                   # POST /clusters/{id}/plan
farvater cluster deploy production [--watch] [--yes]               # POST /clusters/{id}/deploy + SSE stream
farvater cluster list | get production | delete production
farvater cluster upgrade production --to 1.38 [--plan]
farvater cluster scale production --pool workers=6
farvater kubeconfig production > ~/.kube/production
farvater addon install production cert-manager [--set key=value]
farvater operation watch <id> | logs <id> [--follow] | resume <id> | retry <id> [--task …] | cancel <id>
farvater recommend --provider baremetal --env production --size medium --availability ha -o yaml
farvater validate -f cluster.yaml
```

Названия `clusterctl` и `platformctl` из промпта не используются, потому что `clusterctl` — это CLI проекта Kubernetes Cluster API (см. ADR-0002).
