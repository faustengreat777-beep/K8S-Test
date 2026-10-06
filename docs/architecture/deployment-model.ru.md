# Модель развёртывания платформы

> Статус: **Предложено** (Phase 0). Язык: [English](deployment-model.md) · Русский
>
> Связанные документы: [ARCHITECTURE](../ARCHITECTURE.ru.md) · [Модель безопасности](../security/SECURITY_MODEL.ru.md) · ADR-0003 (модульный монолит), ADR-0019 (упаковка и распространение)

Этот документ описывает, как **сам Farvater** собирается, упаковывается, устанавливается, настраивается, обновляется и эксплуатируется: при разработке, в небольших self-hosted-инсталляциях, в Kubernetes, в высокодоступных корпоративных конфигурациях и в изолированных (air-gapped) средах. О том, как платформа развёртывает *клиентские* кластеры, см. [движок провижининга](provisioning-engine.ru.md).

## 1. Артефакты

| Артефакт | Содержимое | Платформы |
|---|---|---|
| Бинарник `farvater-server` | API + worker + миграции + встроенный веб-интерфейс + встроенный каталог по умолчанию | linux/amd64, linux/arm64 |
| Бинарник `farvater` | CLI (клиент API), инструменты для офлайн-бандлов (ee) | linux, macOS, windows × amd64, arm64 |
| OCI-образ `…/farvater-server:<version>` | статический бинарник на distroless/static, запуск не от root (UID 65532), корневая файловая система только для чтения | amd64, arm64 (мультиархитектурный манифест) |
| Helm-чарт `farvater` | Устанавливает платформу в Kubernetes | — |
| Бандл Docker compose | `compose.yaml` + `.env.example` для установки на один хост | — |
| Вспомогательный бинарник для узлов (node helper, Фаза 3) | Небольшой статический бинарник на Go, который загружается на узлы во время операций (подписан) | linux/amd64, linux/arm64 |
| Бандл каталога | Подписанные данные каталога (могут обновляться независимо) | — |
| Метаданные релиза | контрольные суммы, подписи cosign, SBOM, SLSA provenance | — |

Образы никогда не используют `latest`. Теги — неизменяемые версии semver плюс digest (промпт §79). Dockerfile многоэтапные: сборочный этап с зафиксированной версией тулчейна Go и Node для сборки веб-интерфейса, затем финальный образ класса `gcr.io/distroless/static`. Флаги сборки: `-trimpath`, `CGO_ENABLED=0` и `-ldflags` для метаданных версии.

## 2. Роли процессов

Один бинарник, несколько ролей (см. [ARCHITECTURE](../ARCHITECTURE.ru.md)):

| Команда | Роль | Масштабирование | Что требуется |
|---|---|---|---|
| `farvater-server api` | HTTP API, SSE, веб-интерфейс | Без состояния, N реплик за балансировщиком нагрузки | PostgreSQL |
| `farvater-server worker` | Воркеры River: операции, опрос состояния (health), вебхуки, уведомления, очистка по срокам хранения (retention) | N реплик; безопасность обеспечивают аренды (leases) | PostgreSQL, KEK (локальный файл ключа или KMS), исходящий доступ (egress) к инфраструктуре клиента |
| `farvater-server all` | api + worker в одном процессе | 1 (небольшие инсталляции, разработка) | и то и другое |
| `farvater-server migrate` | Применяет миграции БД (под advisory lock) и завершается | Запускается один раз при каждом обновлении (init-контейнер / Job / одноразовый сервис compose) | Роль владельца в PostgreSQL |
| `farvater-server admin …` | Первичная настройка первой организации и администратора, ротация ключей, проверка цепочки аудита, экспорт | Вручную | PostgreSQL |

Эндпоинты состояния: `/healthz` (liveness), `/readyz` (readiness: БД доступна, версия схемы совпадает, а для воркеров — ещё и клиент River исправен), `/metrics` на отдельном порту.

## 3. Конфигурация

Конфигурация в духе 12-factor задаётся переменными окружения или файлами (варианты `*_FILE` для секретов) и проверяется при запуске:

| Переменная | Назначение |
|---|---|
| `DATABASE_URL` (или `DATABASE_URL_FILE`) | DSN PostgreSQL (в профиле production обязателен TLS `verify-full`) |
| `ENCRYPTION_KEY_FILE` / `ENCRYPTION_KEY` | Локальный KEK (256 бит, base64) — обязателен для воркеров; роли `api` никогда не нужен (у api нет пути расшифровки) |
| `KMS_PROVIDER`, `KMS_*` (ee) | Внешний KEK (OpenBao/Vault Transit, AWS/GCP/Azure KMS) |
| `SESSION_SECRET_FILE` | HMAC-ключ для сессий и CSRF |
| `API_KEY_PEPPER_FILE` | Pepper для хеширования API-ключей |
| `PUBLIC_URL` | Внешний URL (cookie, ссылки в уведомлениях, редиректы OIDC) |
| `LISTEN_ADDR`, `METRICS_ADDR` | Адреса для прослушивания |
| `TLS_CERT_FILE`, `TLS_KEY_FILE` | Необязательный встроенный TLS |
| `PROFILE` | `dev` \| `production` — в production небезопасные настройки и симулированный провайдер отклоняются |
| `LOG_LEVEL`, `LOG_FORMAT` | Настройки `slog` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` и т. д. | OpenTelemetry (стандартные переменные окружения) |
| `CATALOG_CHANNEL`, `CATALOG_BUNDLE_URL`, `CATALOG_PUBLIC_KEYS` | Обновления каталога (отключены в air-gap) |
| `EGRESS_ALLOW_PRIVATE_CIDRS` | Allow-list защиты от SSRF для self-hosted-сред, которым нужны приватные адреса назначения |
| `SMTP_*` | Уведомления по email |
| `VALKEY_URL` (необязательно) | Общие ограничение частоты запросов (rate limiting) и кеширование для крупных HA-инсталляций |

Упомянутые в промпте `REDIS_URL` и `JWT_SECRET` **не нужны**: PostgreSQL покрывает очереди, pub/sub и блокировки, а сессии хранятся на сервере, поэтому JWT не используются. Обоснование — в ADR-0007 и ADR-0025. В `.env.example` описана каждая переменная (промпт §84).

## 4. Варианты установки

### 4.1 Разработка (`make dev`, промпт §77)

```
docker compose -f deploy/compose/compose.dev.yaml up
  ├── postgres        (PostgreSQL 18, seeded dev DB, port 5432 on localhost)
  ├── migrate         (one-shot)
  ├── api             (air hot-reload of Go code, PROFILE=dev, simulated provider enabled)
  ├── worker
  └── web             (Vite dev server with HMR, proxy /api → api)
optional profiles: valkey · keycloak (SSO testing, Phase 9) · mailpit (email) · container-nodes (Tier-2 test nodes)
```

Одна команда и никаких зависимостей на хосте, кроме Docker и `make`. Команда seed создаёт администратора и организацию для разработки и один раз выводит их данные. Пароль seed генерируется случайно и никогда не зашивается в код (промпт §115).

### 4.2 Небольшая self-hosted-инсталляция (один хост)

- Docker compose (`deploy/compose/compose.yaml`): `postgres` + `migrate` + `farvater-server all` + необязательный обратный прокси (Caddy) для TLS.
- Или юнит systemd, который запускает бинарник с внешним PostgreSQL.
- Подходит командам, которые управляют несколькими десятками кластеров. Резервные копии БД и KEK делайте по отдельности.

### 4.3 Kubernetes (Helm-чарт)

```
Deployment  api      (replicas ≥ 2, HPA on CPU/RPS, PDB, topology spread)
Deployment  worker   (replicas ≥ 2, PDB; optional dedicated node pool with egress to customer networks)
Job         migrate  (pre-install/pre-upgrade hook)
Service + Gateway/HTTPRoute (or Ingress) for api
Secret refs: DB credentials, KEK (or KMS config), session secret, pepper
NetworkPolicies: api ingress only from gateway; worker no ingress; egress rules configurable
ServiceMonitor (optional), PrometheusRule (optional)
PostgreSQL: external (managed DB) or CloudNativePG cluster (recommended operator for self-managed HA)
```

Поды запускаются с `runAsNonRoot`, `readOnlyRootFilesystem`, без каких-либо capabilities (drop all), с `seccompProfile: RuntimeDefault` и с заданными requests/limits ресурсов.

### 4.4 Высокая доступность для Enterprise (промпт §182)

- api × 3 и worker × 3 в ≥ 2 (лучше в 3) зонах; HA PostgreSQL (CloudNativePG с синхронной репликой, Patroni или управляемый сервис с multi-AZ); балансировщик нагрузки; объектное хранилище для резервных копий платформы и офлайн-бандлов; KMS для KEK.
- В платформе нет единой точки отказа. Аренды воркеров и River делают потерю воркера безопасной. Реплики API не хранят состояния. SSE-клиенты переподключаются к любой реплике, а та воспроизводит события из БД.
- Обновления без простоя: миграции по схеме expand/contract, поочерёдное (rolling) обновление api и worker, воркеры завершают работу (drain) в безопасных точках (`SIGTERM` → перестать брать новые задачи → завершить текущие задачи или сохранить по ним контрольную точку в пределах grace period).
- Готовность к SLO (промпт §222): задокументированные домены отказа, правила мониторинга и оповещений в составе чарта, ранбуки DR. **SLA не обещается** без соответствующей инфраструктуры.

### 4.5 Модели развёртывания (промпт §220)

| Модель | Описание |
|---|---|
| Self-hosted (собственное размещение) | Клиент запускает платформу в своей инфраструктуре (compose, VM, Kubernetes) — первая и основная модель |
| Выделенная (Dedicated) | Однотенантный экземпляр, которым управляет вендор |
| SaaS | Мультитенантный «Platform Cloud», которым управляет вендор (это возможно благодаря архитектуре мультитенантности; воркерам для инфраструктуры клиентов нужна сетевая связность: агент или ретранслятор только с исходящими соединениями — тема для будущего проектирования) |
| Изолированная (air-gapped) | Без доступа в Интернет; офлайн-бандлы (раздел 6) |

## 5. Обновление платформы

1. Прочитайте примечания к релизу и проверьте подписи и контрольные суммы (`cosign verify`).
2. Запускается задание `migrate` (идемпотентное, под advisory lock). Миграции обратно совместимы с предыдущим релизом (фаза expand).
3. Поочерёдное (rolling) обновление: сначала воркеры, затем api. Выполняющиеся операции продолжаются: благодаря арендам воркер передаёт работу на границах задач. Длинные задачи продолжают выполняться на старом воркере до конца grace period, а затем возобновляются на другом.
4. Проверки после обновления: `/readyz`, smoke-тест, проверка каталога.
5. Откат: предыдущие бинарники работают с расширенной схемой. Миграции фазы contract выполняются только в одном из следующих релизов.

Каналы релизов: stable, LTS (enterprise, увеличенный срок поддержки), beta, nightly (промпт §227).

## 6. Изолированная (air-gapped) установка (ee, Фаза 13, промпт §179–180)

```
# On a connected machine
farvater bundle create --catalog-version 2026.10.1 --kubernetes 1.36,1.37 --addons default,longhorn,observability \
                      --os ubuntu-24.04,ubuntu-26.04,debian-13 --output bundle.tar
# → OCI layout: images (by digest), Helm charts (OCI), OS packages / static binaries (containerd, runc, CNI plugins, kubeadm/kubelet/kubectl),
#   node helper, catalog subset, checksums, signatures, SBOMs, manifest

# In the air-gapped environment
farvater bundle verify bundle.tar
farvater bundle import bundle.tar --registry registry.internal:5000 --packages https://mirror.internal/farvater
```

- Каталог переписывается так, чтобы он указывал на внутренние зеркала (реестр образов, зеркало пакетов и бинарников, OCI-репозиторий чартов).
- Узлы получают бинарники из внутреннего зеркала (или их копирует по SFTP node helper), а не из pkgs.k8s.io.
- Импорт — привилегированное действие, которое фиксируется в журнале аудита. Подписи проверяются по настроенным открытым ключам.

## 7. Эксплуатация платформы

- **Наблюдаемость самой платформы** (промпт §60, §231): метрики Prometheus (`farvater_clusters_total`, `…_operations_total{type,status}`, `…_operation_duration_seconds`, `…_tasks_failed_total{kind,code}`, `…_nodes_total`, `…_api_requests_total`, `…_api_request_duration_seconds`, `…_job_queue_depth`, `…_backup_success_total`, `…_backup_failure_total`, `…_webhook_deliveries_total{status}`); трассировки OpenTelemetry по всей цепочке API → задание → задачи → вызовы SSH/K8s; структурированные логи с идентификаторами трассировок; дашборды и правила оповещений в составе чарта.
- **Резервное копирование** (промпт §183): PostgreSQL (логическое или физическое с PITR), конфигурация, KEK (отдельно, офлайн), состояние каталога. Журналы аудита экспортируются в объектное хранилище перед удалением партиции.
- **Ранбуки** (Фаза 1+): восстановление из резервной копии, ротация KEK, ротация DEK, отзыв всех сессий, аварийный режим только для чтения, изоляция воркера, откат каталога.
