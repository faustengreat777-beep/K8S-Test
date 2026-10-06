# Farvater

**От «голой» инфраструктуры до готового к продакшену кластера Kubernetes — с минимальным участием пользователя.**

Язык: [English](README.md) · Русский

> **Статус: Phase 0 — предложение архитектуры.** В репозитории пока только продуктовая и архитектурная документация, кода приложения ещё нет. Реализация начнётся после того, как владелец утвердит архитектуру (см. [ROADMAP](docs/ROADMAP.ru.md)).

*Farvater* («фарватер», от нидерл. *vaarwater*) — безопасный размеченный проход, по которому суда идут в порт. Farvater проводит вас от серверов с доступом по SSH или облачных аккаунтов до защищённого, наблюдаемого кластера Kubernetes с резервными копиями по проверенному маршруту, а потом поддерживает кластер в рабочем состоянии.

## Что будет уметь

- **Готовый к продакшену кластер в один клик.** В режиме **Auto** вы отвечаете на пять вопросов (имя, инфраструктура, окружение, размер, доступность), смотрите план с объяснениями и нажимаете **Deploy**. Можно выбрать пресет (режим Simple) или настроить всё в 15-шаговом мастере или YAML (режим Advanced).
- **Настоящий движок провижининга.** Граф идемпотентных задач (DAG) с учётом зависимостей, прогресс в реальном времени, понятные ошибки (что / почему / где / как исправить), повтор, возобновление и безопасный откат. Если закрыть браузер, развёртывание не остановится.
- **Всё нужное в комплекте, с проверенными версиями.** kubeadm (позже k3s и RKE2), containerd, Cilium/Calico/Flannel, **Gateway API** (Cilium Gateway, Envoy Gateway, …), MetalLB / kube-vip, local-path/NFS/Longhorn/Ceph, cert-manager, Prometheus/Grafana/Loki/OpenTelemetry, резервные копии Velero и etcd, Argo CD/Flux. Подписанный каталог версий и движок совместимости не дадут собрать несовместимую конфигурацию.
- **Жизненный цикл.** Обновления с помощником (upgrade advisor), масштабирование, операции с узлами, резервное копирование и восстановление, управление аддонами, дашборд состояния, обзор нескольких кластеров.
- **API-first.** REST API с зафиксированным контрактом OpenAPI 3.1. Им пользуются веб-интерфейс, CLI `farvater` и (позже) Terraform-провайдер.
- **Безопасность в приоритете.** Хранилище учётных данных с конвертным шифрованием, изоляция тенантов через PostgreSQL RLS, никакой интерполяции в shell, подписанные релизы.
- **Open-core.** Редакция Community (Apache-2.0) — полноценный продукт для провижининга. Enterprise добавляет SSO/SCIM, пользовательские роли и ABAC, политики как код, согласования, комплаенс, операции над парком кластеров, KMS/BYOK и установки без интернета (air-gap).

## Документация

| | English | Русский |
|---|---|---|
| Видение продукта, режимы, редакции, MVP | [PRODUCT](docs/PRODUCT.md) | [PRODUCT](docs/PRODUCT.ru.md) |
| Архитектура (центральный документ) | [ARCHITECTURE](docs/ARCHITECTURE.md) | [ARCHITECTURE](docs/ARCHITECTURE.ru.md) |
| Роадмап | [ROADMAP](docs/ROADMAP.md) | [ROADMAP](docs/ROADMAP.ru.md) |
| Решения (индекс ADR) | [DECISIONS](docs/DECISIONS.md) | [DECISIONS](docs/DECISIONS.ru.md) |
| Технологический стек и база версий | [technology-stack](docs/architecture/technology-stack.md) | [technology-stack](docs/architecture/technology-stack.ru.md) |
| Схема базы данных | [data-model](docs/architecture/data-model.md) | [data-model](docs/architecture/data-model.ru.md) |
| Движок провижининга | [provisioning-engine](docs/architecture/provisioning-engine.md) | [provisioning-engine](docs/architecture/provisioning-engine.ru.md) |
| Основные интерфейсы и плагины | [core-interfaces](docs/architecture/core-interfaces.md) | [core-interfaces](docs/architecture/core-interfaces.ru.md) |
| Режим Auto, каталог, совместимость | [auto-mode-and-catalog](docs/architecture/auto-mode-and-catalog.md) | [auto-mode-and-catalog](docs/architecture/auto-mode-and-catalog.ru.md) |
| Дизайн API | [api-design](docs/architecture/api-design.md) | [api-design](docs/architecture/api-design.ru.md) |
| Архитектура UI | [ui-architecture](docs/architecture/ui-architecture.md) | [ui-architecture](docs/architecture/ui-architecture.ru.md) |
| Развёртывание самой платформы | [deployment-model](docs/architecture/deployment-model.md) | [deployment-model](docs/architecture/deployment-model.ru.md) |
| Стратегия тестирования | [testing-strategy](docs/architecture/testing-strategy.md) | [testing-strategy](docs/architecture/testing-strategy.ru.md) |
| Структура репозитория | [repository-structure](docs/architecture/repository-structure.md) | [repository-structure](docs/architecture/repository-structure.ru.md) |
| Модель безопасности | [SECURITY_MODEL](docs/security/SECURITY_MODEL.md) | [SECURITY_MODEL](docs/security/SECURITY_MODEL.ru.md) |
| Модель угроз | [THREAT_MODEL](docs/security/THREAT_MODEL.md) | [THREAT_MODEL](docs/security/THREAT_MODEL.ru.md) |
| Записи архитектурных решений (ADR) | [docs/architecture/adr/](docs/architecture/adr/) | [docs/architecture/adr/](docs/architecture/adr/) (`*.ru.md`) |

`DEVELOPMENT.md`, `DEPLOYMENT.md`, `CONTRIBUTING.md`, `SECURITY.md` и `API.md` появятся вместе с кодом в Phase 1.

## Роадмап кратко

Phase 0 Архитектура (сейчас) → 1 Фундамент → 2 Движок провижининга → 3 Bare metal + kubeadm + Cilium → 4 Продакшен-кластер (**MVP**) → 5 Режим Auto → 6 Жизненный цикл → 7 Несколько провайдеров → 8 GitOps → 9–14 Enterprise.

## Лицензия

Ядро Farvater распространяется по лицензии [Apache License 2.0](LICENSE). Функции Enterprise будут находиться в каталоге `ee/` под отдельной коммерческой лицензией.
