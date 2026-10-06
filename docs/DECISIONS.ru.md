# Решения

> Статус: **Предложено** (Phase 0) — ожидает утверждения владельцем. Язык: [English](DECISIONS.md) · Русский
>
> Указатель записей об архитектурных решениях (ADR). Формат и процесс: [ADR-0001](architecture/adr/0001-record-architecture-decisions.ru.md). ADR со статусом *Предложено* получают статус *Принято*, когда владелец утверждает архитектуру Фазы 0.

## Решения владельца, принятые в Фазе 0 (2026-10-06)

| № | Вопрос | Ответ |
|---|---|---|
| 1 | Объём этой сессии | Только Фаза 0; остановиться и дождаться утверждения перед Фазой 1 |
| 2 | Название продукта | Архитектор выбирает оригинальное название, ещё не занятое на рынке (см. ADR-0002) |
| 3 | Лицензирование | Open-core: ядро под Apache-2.0 + коммерческий `ee/` |
| 4 | Оборудование для тестов | Собственных машин у владельца нет → симулированный провайдер, узлы в контейнерах, позже — облачные ВМ (ADR-0016) |
| 5 | Ingress по умолчанию | Gateway API (ADR-0020) |
| 6 | Язык документации | Английский и русский бок о бок (ADR-0026) |
| 7 | Работа с Git | Пуш в рабочую ветку; без pull request |

## Указатель ADR

| ADR | Название | Статус | Кратко |
|---|---|---|---|
| [0001](architecture/adr/0001-record-architecture-decisions.ru.md) | Фиксация архитектурных решений | Принято | Облегчённый MADR в `docs/architecture/adr/`, на двух языках, после принятия не изменяются |
| [0002](architecture/adr/0002-product-name-and-open-core-licensing.ru.md) | Название продукта, лицензирование open-core и редакции | Предложено | Название продукта; ядро под Apache-2.0 + коммерческий `ee/`; права (entitlements) в одном сервисе; Enterprise никогда не урезает ядро |
| [0003](architecture/adr/0003-modular-monolith.ru.md) | Модульный монолит с одним серверным бинарником и ролями процессов | Предложено | Роли `api` / `worker` / `all` / `migrate` одного Go-бинарника; учётные данные есть только у воркеров |
| [0004](architecture/adr/0004-repository-layout.ru.md) | Идиоматичная для Go структура монорепозитория с одним Go-модулем | Предложено | `cmd/`, `internal/`, `pkg/`, `plugins/`, `catalog/`, `web/`, `ee/`; правила импорта контролируются принудительно |
| [0005](architecture/adr/0005-api-style-and-contract.ru.md) | REST/JSON API с закоммиченным контрактом OpenAPI 3.1 (Huma) | Предложено | Типизированные операции на Go генерируют OpenAPI 3.1; закоммиченная спецификация с проверкой oasdiff; 202 + Operation для асинхронных действий; ошибки по RFC 9457 |
| [0006](architecture/adr/0006-persistence-postgresql.ru.md) | PostgreSQL — единственная обязательная зависимость с состоянием | Предложено | PG 18 (минимум 16), pgx, sqlc, goose; без ORM; без Bitnami |
| [0007](architecture/adr/0007-job-queue-river-no-redis.ru.md) | Очередь заданий River; PostgreSQL для pub/sub и блокировок; без Redis | Предложено | Транзакционная постановка в очередь; LISTEN/NOTIFY; advisory-блокировки; опционально Valkey, но никогда Redis |
| [0008](architecture/adr/0008-provisioning-engine-execution-model.ru.md) | Собственный DAG-движок провижининга; одно задание с арендой (lease) на операцию | Предложено | Планировщик → DAG → исполнитель с арендами, повторами, возобновлением и безопасным откатом; Temporal пока не используем |
| [0009](architecture/adr/0009-real-time-updates-sse.ru.md) | Server-Sent Events на основе журнала событий с воспроизведением | Предложено | SSE + воспроизведение (replay) по `Last-Event-ID` из PostgreSQL; WebSocket только для будущего терминала |
| [0010](architecture/adr/0010-plugin-model-and-helm.ru.md) | Плагины на этапе компиляции; декларативные аддоны через встроенный Helm 4 SDK | Предложено | Интерфейсы `pkg/sdk`; аддоны как данные; Helm v4 SDK; OCI-чарты, закреплённые по дайджесту; без Bitnami |
| [0011](architecture/adr/0011-multi-tenancy-and-rls.ru.md) | Мультитенантность с PostgreSQL RLS как эшелонированной защитой | Предложено | Разграничение по организациям и проектам, авторизация на уровне приложения, `FORCE ROW LEVEL SECURITY`, 404 для чужих id |
| [0012](architecture/adr/0012-frontend-stack.ru.md) | Фронтенд-стек | Предложено | React + TypeScript + Vite + Tailwind + shadcn/ui + TanStack + RHF/Zod + Monaco + i18next |
| [0013](architecture/adr/0013-infrastructure-tooling-boundaries.ru.md) | Границы инфраструктурного инструментария; Cluster API отложен | Предложено | Нативные провайдеры на Go; без Terraform/Ansible в ядре; Cluster API для облаков оценивается в Фазе 7 |
| [0014](architecture/adr/0014-secrets-envelope-encryption.ru.md) | Секреты: конвертное шифрование (envelope encryption) с подключаемыми провайдерами ключей | Предложено | Наборы ключей Tink AEAD для каждой организации, обёрнутые ключом шифрования ключей (KEK) (local / KMS / OpenBao); расшифровывают только воркеры |
| [0015](architecture/adr/0015-remote-execution-ssh.ru.md) | Безагентный SSH с типизированными командами и временным подписанным помощником на узле (node helper) | Предложено | Без интерполяции в shell; конфигурационные файлы по SFTP; обязательная проверка ключа хоста |
| [0016](architecture/adr/0016-testing-without-own-hardware.ru.md) | Тестирование без собственного оборудования | Предложено | Симулированный провайдер, узлы в контейнерах с настоящим SSH, уровень с реальными ВМ при наличии бюджета |
| [0017](architecture/adr/0017-data-driven-version-catalog.ru.md) | Подписанный каталог версий, управляемый данными | Предложено | Все версии и совместимость — подписанные данные, обновляемые без релизов, дайджесты для air-gap |
| [0018](architecture/adr/0018-explainable-decision-engine.ru.md) | Детерминированный объяснимый движок принятия решений на основе правил | Предложено | Упорядоченные правила выдают решения с обоснованиями; переопределения фиксируют значение и запускают пересчёт; эталонные (golden) тесты |
| [0019](architecture/adr/0019-packaging-and-supply-chain.ru.md) | Упаковка, распространение и безопасность цепочки поставок | Предложено | Статический бинарник + distroless-образы + Helm-чарт + compose; подписанные релизы, SBOM, provenance |
| [0020](architecture/adr/0020-networking-defaults-gateway-api.ru.md) | Сетевые настройки по умолчанию: Cilium; только Gateway API; без ingress-nginx | Предложено | CRD Gateway API под управлением платформы; Cilium Gateway или Envoy Gateway |
| [0021](architecture/adr/0021-kubernetes-version-and-runtime-defaults.ru.md) | Политика версий Kubernetes и среда выполнения на узлах по умолчанию | Предложено | Поддерживаются 1.35–1.37, по умолчанию 1.36; kubeadm v1beta4; containerd 2.3 LTS; cgroup v2; nftables |
| [0022](architecture/adr/0022-load-balancing-and-control-plane-endpoint.ru.md) | Балансировка нагрузки и endpoint control plane | Предложено | kube-vip / внешний LB / LB провайдера; MetalLB или Cilium LB-IPAM; L2 запрещён в облаках с L3-сетями |
| [0023](architecture/adr/0023-storage-defaults.ru.md) | Хранилище по умолчанию | Предложено | local-path (dev), Longhorn (prod, bare metal), Rook-Ceph (большие объёмы/высокий IO), облачные CSI; MinIO не входит в поставку |
| [0024](architecture/adr/0024-observability-stack-and-licensing.ru.md) | Наблюдаемость по умолчанию и работа с AGPL | Предложено | Пресеты Basic/Standard/Full; AGPL-компоненты устанавливаются без изменений; альтернативы под Apache-2.0 |
| [0025](architecture/adr/0025-authentication-sessions-and-api-keys.ru.md) | Аутентификация: сессии, API-ключи, SSO в Enterprise | Предложено | Серверные сессии (без JWT), хешированные API-ключи с ограниченной областью действия, argon2id; OIDC/SAML/SCIM/MFA в ee |
| [0026](architecture/adr/0026-bilingual-documentation.ru.md) | Двуязычная документация | Принято | `X.md` + `X.ru.md`, исходный текст на английском, обновление в том же PR, проверка структуры в CI |
| [0027](architecture/adr/0027-operation-instead-of-deployment.ru.md) | Моделировать «Deployment» как Operation / OperationTask | Предложено | Однозначные имена в коде и API; в текстах UI и именах вебхуков остаётся «deployment» |

## Отступления от продуктовой спецификации (и их причины)

| Пункт спецификации | Решение | Причина | ADR |
|---|---|---|---|
| Kubernetes 1.34 в примерах | По умолчанию 1.36, последняя — 1.37; 1.34 не предлагается | Поддержка 1.34 заканчивается (EOL) 2026-10-27; матрицы тестирования аддонов заканчиваются на 1.36 | 0021 |
| NGINX Ingress по умолчанию (§19, §120) | Только Gateway API; ingress-nginx не предлагается никогда | ingress-nginx выведен из эксплуатации и архивирован (2026-03) | 0020 |
| Hetzner + MetalLB в основном сценарии (§120) | Hetzner LB через hcloud CCM в Hetzner Cloud; MetalLB для L2 на bare metal | MetalLB в режиме L2/ARP не работает в L3-сетях Hetzner Cloud | 0022 |
| Redis + очередь заданий (§43, §84 `REDIS_URL`) | River на PostgreSQL; без Redis; опционально Valkey | Транзакционная постановка в очередь, меньше компонентов, лицензирование | 0007 |
| `JWT_SECRET` (§84) | Серверные сессии; без JWT для аутентификации собственных (first-party) клиентов | Мгновенный отзыв, просмотр списка сессий | 0025 |
| Имена CLI `clusterctl` / `platformctl` (§56, §197) | CLI с названием продукта | `clusterctl` — это CLI проекта Cluster API | 0002 |
| Модели «Deployment»/«DeploymentTask» (§58) | `Operation`/`OperationTask` | Охватывает все действия жизненного цикла; нет конфликта с `Deployment` из Kubernetes | 0027 |
| Структура `apps/` + `packages/` (§76) | Идиоматичная для Go `cmd/internal/pkg/plugins` | Инкапсуляция в Go, обеспечиваемая компилятором; явно разрешено §76 | 0004 |
| Использование Terraform/Ansible (§55) | Не в ядре; нативные провайдеры на Go; позже — собственный Terraform-провайдер для API | Единая модель исполнения с типизированными ошибками и прогрессом | 0013 |
| MinIO как хранилище резервных копий (§28) | Поддерживается как *внешний* S3-endpoint; в поставку не входит | Community-редакция MinIO архивирована | 0023 |
| Prometheus/Grafana/Loki/Tempo (§22) | Сохранены, с соблюдением лицензионных границ и альтернативами под Apache-2.0 | Grafana/Loki/Tempo распространяются под AGPL | 0024 |
| «Mock/stub-развёртывание» в Фазе 1 против «никаких фиктивных развёртываний» в §115 | Провайдер `simulated`, только для тестов, запрещён в production-профиле | Удовлетворяет обоим требованиям | 0016 |
