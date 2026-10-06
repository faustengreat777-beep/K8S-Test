# ADR-0024: Аддоны наблюдаемости по умолчанию и работа с компонентами под AGPL и source-available лицензиями

- Статус: Предложено
- Дата: 2026-10-06
- Язык: [English](0024-observability-stack-and-licensing.md) · Русский

## Контекст

Спецификация требует метрик (Prometheus, Grafana, metrics-server), логирования (Loki, Fluent Bit), трассировки (OpenTelemetry, Tempo), мониторинга Kubernetes (kube-state-metrics, node-exporter), пресетов (Basic, Standard, Full), а также URL и учётных данных Grafana с дашбордами после развёртывания (промпт §22–23). Исследование (2026-10-06):
- Чарт **kube-prometheus-stack** 91.x (Prometheus Operator 0.94, Prometheus 3.15) распространяется под Apache-2.0. Его сабчарт Grafana теперь берётся из чартов `grafana-community`.
- **Grafana 13.2**, **Loki 3.7** и **Tempo 3.1** распространяются под **AGPL-3.0**. Их OSS-чарты переехали в `grafana-community/helm-charts`.
- **Promtail достиг EOL** (2026-03-02). Режим Simple Scalable в Loki объявлен устаревшим (будет удалён в 4.0).
- Режиму микросервисов в Tempo 3.x нужна Kafka.
- **Grafana Alloy** (Apache-2.0), **Fluent Bit 5.1** и **OpenTelemetry Collector/Operator** распространяются под Apache-2.0.
- **VictoriaMetrics/VictoriaLogs** — альтернативы под Apache-2.0.
- Также важно: Argo CD включает в поставку Redis 8 (RSAL/SSPL/AGPL), falcosidekick-ui использует redis-stack, а Flux Operator распространяется под AGPL.

## Решение

- **Пресеты**:
  - **Basic**: только metrics-server.
  - **Standard**: kube-prometheus-stack (Prometheus, Alertmanager, kube-state-metrics, node-exporter, стандартные дашборды и алерты) + Grafana + Loki (монолитный режим) + Alloy в качестве агента логов.
  - **Full**: Standard + Tempo (монолитный режим) + OpenTelemetry Collector (gateway) + OpenTelemetry Operator (опциональная автоинструментация).
- **Передача Grafana пользователю**: после развёртывания пользователь получает URL Grafana (через Gateway кластера с сертификатом cert-manager), имя администратора и пароль, который хранится как секрет и может быть получен один раз, либо конфигурацию SSO, если она доступна. Дашборды создаются автоматически (кластер, узлы, аддоны, Cilium/Hubble, Longhorn).
- **Лицензионные правила** для компонентов под AGPL и source-available лицензиями (Grafana, Loki, Tempo, Flux Operator, Redis 8 внутри чартов):
  1. Они устанавливаются **без изменений** из артефактов upstream в **пользовательские кластеры** как отдельные программы. Мы никогда не линкуем, не встраиваем и не изменяем их код в core или ee и никогда не встраиваем UI Grafana в UI нашего продукта.
  2. Бандлы для изолированных (air-gapped) установок, которые распространяют эти компоненты, содержат тексты лицензий и предложение предоставить исходный код (source offer). Юридическая проверка проводится до подготовки поставки редакции Enterprise.
  3. В каталоге есть альтернативы под Apache-2.0: VictoriaMetrics k8s-stack и VictoriaLogs (бэкенды), Alloy, Fluent Bit или OTel Collector (агенты).
  4. В values встроенных чартов Redis по возможности заменяется на **Valkey** (Argo CD), а source-available подкомпоненты (redis-stack в falcosidekick-ui) остаются отключёнными.
  5. CI аддонов формирует SBOM и проверяет лицензии каждого образа, а политика завершается ошибкой при обнаружении неожиданных лицензий.
- **Не предлагаются**: Promtail (EOL), режим SSD в Loki, распределённый Tempo (зависимость от Kafka), MinIO в качестве встроенного объектного хранилища.
- **Самонаблюдаемость платформы** (метрики Prometheus, трассы OTel, логи slog) не зависит от этих аддонов (см. модель развёртывания).

## Рассмотренные альтернативы

- **VictoriaMetrics/VictoriaLogs по умолчанию**: полностью под Apache-2.0 и экономны по ресурсам. Однако большинство пользователей ожидают и знают именно стек Prometheus Operator/Grafana/Loki. Предлагается как полноценная альтернатива (пресет «Apache-2.0 observability»).
- **Встроить Grafana в UI продукта**: обязательства по AGPL и сильная связанность; отклонено.
- **Fluent Bit как агент по умолчанию**: тоже хороший вариант (Apache-2.0). У Alloy полноценная поддержка Loki/OTel/Prometheus и конвертер конфигурации promtail.

## Последствия

- Плюсы: привычные, хорошо поддерживаемые значения по умолчанию с чёткими лицензионными границами и путь на Apache-2.0 для клиентов, чувствительных к лицензиям.
- Минусы: приходится отслеживать в каталоге быстрые переезды чартов (grafana → grafana-community) и изменения архитектуры Tempo/Loki.

## Ссылки

- kube-prometheus-stack, grafana-community/helm-charts, режимы развёртывания Loki, уведомление об EOL Promtail, примечания к выпуску Tempo 3.0, Alloy, документация VictoriaMetrics; файлы лицензий компонентов
