# Режим Auto, каталог версий, движки совместимости и принятия решений

> Статус: **Предложено** (Phase 0). Язык: [English](auto-mode-and-catalog.md) · Русский
>
> Связанные документы: [ARCHITECTURE](../ARCHITECTURE.ru.md) · [Движок провижининга](provisioning-engine.ru.md) · [Ключевые интерфейсы](core-interfaces.ru.md) · ADR-0017 (каталог, управляемый данными), ADR-0018 (движок принятия решений на основе правил)

Режим Auto (Auto Mode) — основной UX Farvater (промпт §127). Пользователь отвечает на несколько вопросов, а платформа принимает остальные технические решения — **прозрачно** (промпт §130), **с возможностью переопределения** (промпт §131) и **безопасно** (промпт §132). Работу режима обеспечивают три компонента:

1. **Каталог версий**: данные о том, что существует, какие версии поддерживаются и как они сочетаются друг с другом.
2. **Движок совместимости**: проверяет конкретную конфигурацию по каталогу и правилам. Он блокирует нерабочие сочетания и объясняет предупреждения.
3. **Движок принятия решений (рекомендаций)**: превращает высокоуровневые ответы в полную ClusterSpec с обоснованиями.

Пресеты, режим Simple (Simple Mode) и режим Advanced (Advanced Mode) используют те же три компонента.

## 1. Каталог версий (промпт §49)

### 1.1 Принципы

- **Никаких строк с версиями в коде.** Каждая версия Kubernetes, релиз дистрибутива, среда выполнения контейнеров, версия аддона, Helm-чарт, образ и сведения о поддержке ОС хранятся в каталоге (промпт §49). Код ссылается на идентификаторы из каталога.
- **Данные, версионированные и подписанные.** Каталог — это директория с YAML-файлами в `catalog/`. Он встраивается в каждый релиз, а также может обновляться независимо через **подписанный бандл каталога** (подпись Ed25519, монотонно растущая версия, канал `stable`/`lts`/`beta`). Благодаря этому новые патч-версии Kubernetes и обновления аддонов можно поставлять без релиза платформы, а изолированные (air-gapped) установки могут импортировать каталоги офлайн.
- **Закреплённость и воспроизводимость.** У каждого разрешаемого элемента есть точные версии и **digest/контрольные суммы**: digest чартов, digest образов, SHA-256 бинарных файлов. ResolvedSpec ссылается на версию каталога, поэтому повторное планирование с тем же каталогом даёт те же артефакты.
- **Проверка при загрузке.** Валидация по схеме, ссылочная целостность (у каждого аддона есть реализация, каждый список образов полон), ацикличность графа возможностей (capabilities), корректность диапазонов версий. Повреждённый каталог отклоняется, и активным остаётся предыдущий.
- **Генерация, где это возможно.** CI-задания предлагают обновления каталога (патч-версии Kubernetes — из `dl.k8s.io/release/stable-1.x.txt` и графика релизов; версии чартов — из реестров; списки образов — из `helm template`). Человек проводит ревью, и перед слиянием выполняются тесты совместимости.

### 1.2 Структура

```
catalog/
├── catalog.yaml                 # catalog version, channel, minimum platform version, signing metadata
├── kubernetes.yaml              # minors: status, patches, EOL, default flag, per-minor component pins (etcd, CoreDNS, pause)
├── distributions/
│   ├── kubeadm.yaml             # supported minors, package repos/binary URLs + checksums, config API version (v1beta4)
│   ├── k3s.yaml
│   └── rke2.yaml
├── runtimes/containerd.yaml     # versions, LTS flags, K8s compatibility, runc pins, config version
├── os/                          # support matrix: ubuntu.yaml, debian.yaml, rocky.yaml, almalinux.yaml (versions, kernels, notes)
├── addons/                      # one file per add-on: versions, chart refs+digests, images, K8s ranges, capabilities, ports, defaults
│   ├── cilium.yaml
│   ├── calico.yaml
│   ├── flannel.yaml
│   ├── gateway-api-crds.yaml
│   ├── envoy-gateway.yaml
│   ├── metallb.yaml
│   ├── kube-vip.yaml
│   ├── cert-manager.yaml
│   ├── longhorn.yaml
│   └── …
├── rules/                       # compatibility rules (declarative), see §2
├── presets/                     # Minimal, Development, Staging, Production, High Availability, Edge, GPU, AI/ML
└── estimates.yaml               # default task durations for time estimates (until local history exists)
```

Примеры записей (значения соответствуют базовому набору версий по итогам исследования на 2026-10-06; см. [технологический стек §3](technology-stack.ru.md)):

```yaml
# catalog/kubernetes.yaml (excerpt)
minors:
  - minor: "1.37"
    status: supported            # current | supported | deprecated | blocked
    default: false
    latestPatch: "1.37.1"
    eol: "2027-10-28"
    components: { etcd: "3.7.0-0", coredns: "v1.14.6", pause: "3.10.2" }
    notes: [ "kubeadm v1beta3 config removed; v1beta4 only", "etcd 3.7 default: no binary rollback after upgrade" ]
  - minor: "1.36"
    status: current
    default: true                # default for new clusters (widest add-on compatibility today)
    latestPatch: "1.36.5"
    eol: "2027-06-28"
    components: { etcd: "3.6.8-0", coredns: "v1.14.2", pause: "3.10.2" }
  - minor: "1.35"
    status: supported
    latestPatch: "1.35.9"
    eol: "2027-02-28"
  - minor: "1.34"
    status: deprecated           # EOL 2026-10-27 — not offered for new clusters
    latestPatch: "1.34.12"
    eol: "2026-10-27"
```

```yaml
# catalog/addons/cilium.yaml (excerpt)
id: cilium
category: cni
versions:
  - version: "1.20.2"
    chart: { ref: "oci://quay.io/cilium/charts/cilium", version: "1.20.2", digest: "sha256:…" }
    kubernetes: { tested: ">=1.33 <1.37", allowedUntested: ">=1.37 <1.38" }   # untested → WARNING, not BLOCK
    requires: { gatewayApiCRDs: ">=1.6.1" }                                     # for Cilium Gateway API support
    images: [ "quay.io/cilium/cilium:v1.20.2@sha256:…", "quay.io/cilium/operator-generic:v1.20.2@sha256:…", … ]
    status: recommended
  - version: "1.19.8"
    kubernetes: { tested: ">=1.32 <1.36" }
    status: supported
features:
  kubeProxyReplacement: { status: ga }
  hubble: { status: ga }
  encryption.wireguard: { status: ga, note: "pod-to-pod GA; node-to-node beta" }
  l2Announcements: { status: beta }
  gatewayAPI: { status: ga }
ports: [ { port: 4240, protocol: TCP, purpose: health }, { port: 8472, protocol: UDP, purpose: vxlan } ]
```

### 1.3 Каналы релизов и семантика статусов

| Статус | Значение в UI и движках |
|---|---|
| `current` / `recommended` | Выбор по умолчанию; полностью протестировано |
| `supported` | Доступно для выбора; протестировано |
| Сочетание `untested` | Доступно для выбора с WARNING («Cilium 1.20 ещё не протестирован на Kubernetes 1.37»); блокируется корпоративной политикой, если она настроена |
| `deprecated` | Не предлагается для новых кластеров; для существующих кластеров выдаются рекомендации по обновлению |
| `blocked` | Никогда не доступно для выбора (заведомо неработоспособно, проблема безопасности, EOL) — например, HCCM ≤ v1.30.0 |

## 2. Движок совместимости (промпт §48)

Назначение: **никогда не позволять пользователю создать кластер, о котором известно, что он неработоспособен.** Объяснять каждое ограничение.

### 2.1 Входы и выходы

```
Check(resolvedSpec, catalog, [inventoryFacts]) → []Finding{severity: BLOCK|WARNING|INFO, code, path, params, remediation}
```

- Запускается **во время редактирования** (UI обращается к `/validate` по мере ввода), **во время планирования** (жёсткий барьер) и в **советнике по обновлению (upgrade advisor)** (целевая версия против установленных аддонов, ОС и устаревших API).
- При наличии фактов об инвентаре (после обнаружения и предварительных проверок (preflight)) движок также проверяет оборудование и ОС: архитектуру CPU, x86-64-v3 для Rocky 10, версию ядра для kube-proxy в режиме nftables (≥ 5.13), cgroup v2, минимальные объёмы RAM/диска для каждой роли и набора аддонов.

### 2.2 Типы правил

Правила — это **декларативные данные** (`catalog/rules/*.yaml`), которые вычисляет небольшой движок на Go. Правила, которые невозможно выразить данными, пишутся как Go-функции правил, регистрируются по идентификатору и покрываются модульными тестами.

| Тип правила | Пример |
|---|---|
| Диапазон версий | аддон `cilium@1.20.x` требует `kubernetes ∈ [1.33, 1.37)` (протестировано) → иначе WARNING (не протестировано) или BLOCK (`>= 1.38`) |
| Требуемая возможность | `kube-prometheus-stack` с persistence требует `default-storage-class` |
| Конфликты | два CNI; `flannel` + `networkPolicies: enforced` (нет поддержки NetworkPolicy) → BLOCK с рекомендацией «выберите Cilium или Calico либо добавьте kube-network-policies» |
| Возможности провайдера | `loadBalancer: metallb` у провайдера с `l3OnlyNetwork` (Hetzner Cloud) → BLOCK, рекомендация «используйте балансировщик нагрузки провайдера» |
| ОС / архитектура | Rocky Linux 10 требует CPU x86-64-v3; Ubuntu 22.04 не поддерживается для K8s ≥ 1.36 (пример) |
| Минимальные ресурсы | Longhorn с `replicaCount: 3` требует ≥ 3 узлов, доступных для планирования, с ≥ N GiB свободного места на диске |
| Топология | HA требует нечётного числа узлов control plane, не меньше 3; узлы control plane распределяются по доменам отказа, если они есть (промпт §176) |
| Путь обновления | обновление с пропуском минорной версии → BLOCK; etcd 3.6 < 3.6.11 перед переходом на 3.7 → BLOCK с «сначала обновите etcd» |
| Устаревание | целевая версия K8s удаляет API, который всё ещё используют установленные ресурсы (советник по обновлению) → BLOCK со списком ресурсов |
| Статус функции | Cilium L2 announcements → INFO «бета-функция», в продакшене → WARNING |

### 2.3 Интеграция с UI

- UI скрывает или отключает варианты, которые в текущем контексте дали бы результаты проверки (findings) уровня BLOCK, и показывает причину во всплывающей подсказке. Например, MetalLB отключён на Hetzner Cloud с пояснением «Сети Hetzner Cloud работают на уровне L3; используйте балансировщик нагрузки Hetzner».
- Результаты уровня WARNING показываются рядом с полями и на шаге проверки (Review). Для окружений production они требуют подтверждения.
- Каждый результат проверки имеет стабильный код (общий каталог ошибок), параметры и локализованную рекомендацию по исправлению.

## 3. Движок принятия решений — режим Auto (промпт §128–129)

### 3.1 Входные данные

| Вход | Значения |
|---|---|
| Окружение | Development, Staging, Production, Enterprise |
| Доступность | Single Node, Standard, High Availability, Mission Critical |
| Размер | Small, Medium, Large, XLarge (определяет число и размер узлов с учётом провайдера) |
| Тип нагрузки | General, Web, Database, AI/ML, GPU, High Network, High Storage IO, Edge |
| Бюджет | Low, Balanced, Performance, Unlimited |
| Соответствие требованиям (compliance) | None, SOC 2, ISO 27001, HIPAA, PCI DSS, Custom (влияет на выбор профиля безопасности, аудита, резервного копирования и шифрования; никогда не заявляет о соответствии) |
| Инфраструктура | аккаунт провайдера + **возможности и инвентарь** (регионы, типы инстансов, цены, сеть L2/L3, поддержка LB) или объявленные хосты с фактами |
| Регион / домены отказа | из меток провайдера или хостов |
| ОС | обнаружена или выбрана |
| Предпочтения пользователя | значения по умолчанию организации (например, «предпочитать Calico»), закреплённые переопределения из UI |
| Политики (ee) | ограничения из движка политик (например, «production должен использовать Cilium»), которые применяются как жёсткие ограничения до запуска правил |

### 3.2 Конвейер правил

Движок — это **чистая детерминированная функция** без ввода-вывода. Инвентарь и каталог поступают на вход. Движок выполняет **упорядоченный список правил**. Каждое правило читает входные данные и уже принятые решения и выдаёт значения `Decision`:

```go
type Decision struct {
    Path        string      // JSON pointer in ClusterSpec, e.g. "/spec/networking/cni/provider"
    Value       any
    Reasons     []Reason    // i18n keys + params: "environment.production", "availability.ha", "feature.networkPolicies"
    Alternatives []Alternative // other valid values with trade-offs, for the [Change] menu
    Risk        []RiskTag   // data-loss | downtime | security | cost | compatibility
    Pinned      bool        // set by user override; rules must not change pinned paths
    RuleID      string
}
```

Порядок правил (упрощённо):
1. **Ограничения**: применить политики (ee) и закреплённые переопределения пользователя.
2. **Топология**: число узлов control plane по уровню доступности (Single Node → 1, Standard → 1 с предупреждением в продакшене, HA → 3, Mission Critical → 5 в ≥ 3 доменах отказа); число и размер рабочих узлов — по размеру, типу нагрузки и бюджету; распределение по доменам отказа.
3. **Kubernetes**: дистрибутив (bare metal/VPS → kubeadm; Edge с небольшими узлами → k3s, когда он станет доступен), минорная версия = `default` из каталога, если ограничения не требуют иного, патч-версия = последняя в этой минорной версии.
4. **Сеть**: CNI (production/HA/сетевые политики → Cilium; k3s на edge → Flannel + kube-network-policies или Cilium, если позволяют ресурсы); CIDR подов и сервисов выбираются так, чтобы не пересекаться с обнаруженными сетями хостов; замена kube-proxy при Cilium; реализация Gateway (Cilium Gateway, если CNI = Cilium, иначе Envoy Gateway); эндпоинт control plane (kube-vip на bare metal с поддержкой L2, LB провайдера в облаках, внешний LB, если он объявлен).
5. **Балансировщик нагрузки для Services**: MetalLB L2 (bare metal со смежностью на уровне L2), Cilium LB-IPAM + BGP (если заданы BGP-пиры), LB провайдера (облака), NodePort (development без LB).
6. **Хранилище**: development → local-path; production на bare metal с ≥ 3 рабочими узлами → Longhorn (3 реплики; 2 при низком бюджете, с предупреждением); large/high-IO → Rook-Ceph предлагается как альтернатива; облако → CSI провайдера; задаётся StorageClass по умолчанию.
7. **Сертификаты**: cert-manager всегда; Let's Encrypt HTTP-01, если публичный DNS/ingress доступен, иначе внутренний CA / самоподписанный сертификат для development.
8. **Наблюдаемость**: Development → Basic (metrics-server); Staging → Standard (Prometheus + Grafana + Loki); Production → Full (добавляет Tempo/OpenTelemetry, оповещения), если бюджет не Low.
9. **Безопасность**: профиль hardened для Production/Enterprise или при выборе любого требования соответствия; PodSecurity restricted; шаблоны NetworkPolicy с запретом по умолчанию (default-deny); журнал аудита; шифрование секретов при хранении.
10. **Резервное копирование**: production → снапшоты etcd + Velero в S3-совместимое хранилище (запрашивает место назначения; если его нет — WARNING «требуется место назначения для резервных копий», а политика может выдать BLOCK).
11. **Ресурсы**: метки/taints узлов (на GPU-узлы ставятся taints; выделенные узлы ingress для High Network), запросы ресурсов (requests) для аддонов по уровню размера.
12. **Версии аддонов**: из каталога для выбранной минорной версии Kubernetes (рекомендуемые версии; их проверяет движок совместимости).
13. **Проверка рисков (safety review)** (промпт §132): выдаёт предупреждения о вариантах, ведущих к потере данных, простою, снижению защищённости, высоким затратам или несовместимости. Примеры: 1 CP в продакшене; реплик Longhorn < 3; нет места назначения для резервных копий; бюджет «Unlimited» при > N узлов; непротестированные сочетания версий.
14. **Оценки**: время — по оценкам/истории; стоимость — по ценам провайдера, иначе «Оценка стоимости недоступна» (промпт §36: никогда не выдумывать цены).

### 3.3 Прозрачность (промпт §130)

Каждое решение хранит свои обоснования, поэтому UI может ответить на вопрос «Почему такая конфигурация?» для каждого поля:

```
Why Cilium?
  Selected because:
   - Production environment
   - High availability cluster
   - Network policies enabled
   - Hubble observability requested
  Alternatives: Calico (BGP-native networking, Calico Ingress Gateway), Flannel (lightweight; no NetworkPolicy)

Why 3 control-plane nodes?
  Selected because:
   - High availability requested
   - etcd quorum requires an odd number of members (tolerates 1 failure with 3)
```

### 3.4 Переопределения (промпт §131)

- Переопределение пользователя **закрепляет** путь (`Pinned: true`) и перезапускает весь конвейер. Зависимые решения пересчитываются: переключение CNI на Flannel меняет Gateway на Envoy Gateway, убирает Hubble и добавляет предупреждение о NetworkPolicy. UI подсвечивает, что изменилось из-за переопределения.
- Переопределения, нарушающие правило уровня BLOCK, отклоняются с соответствующим результатом проверки. Переопределения, вызывающие предупреждения, в продакшене требуют подтверждения.

### 3.5 Безопасность (промпт §132)

Режим Auto никогда не выбирает рискованные настройки **молча**. Предупреждения предлагают явный выбор:

```
⚠ Warning
You selected 1 control-plane node for a production cluster.
This configuration does not provide control-plane high availability.
[Continue anyway]   [Enable HA]
```

Журнал решений фиксирует подтверждение (кто и когда) в метаданных ревизии спецификации и в журнале аудита.

### 3.6 Тестирование

- Golden-file-тесты: для матрицы входных данных (окружение × доступность × размер × тип нагрузки × бюджет × возможности провайдера) сохраняется снимок полной рекомендации (Recommendation: спецификация + обоснования + предупреждения). Изменения видны как diff, доступный для ревью.
- Property-тесты: каждая рекомендация проходит движок совместимости без единого результата BLOCK; закреплённые пути никогда не меняются; одни и те же входные данные всегда дают один и тот же результат.

## 4. Пресеты (промпт §3, §74)

Пресеты — это **именованные входные данные плюс значения по умолчанию** для движка принятия решений; они хранятся в `catalog/presets/`. Это не отдельные ветви кода. Каждый пресет также можно сохранить как шаблон.

| Пресет | Ключевые решения |
|---|---|
| Minimal | 1 узел (CP+worker), Kubernetes, CNI, CoreDNS, metrics-server |
| Development | 1 CP + 1–2 рабочих узла, CNI, Gateway, хранилище local-path, metrics-server |
| Staging | 1–3 CP, 2–3 рабочих узла, аддоны как в Production, но с меньшими размерами, наблюдаемость уровня Standard |
| Production | 3 CP (HA), ≥ 3 рабочих узлов, Cilium, Gateway + TLS (cert-manager), Longhorn/CSI провайдера, мониторинг, логирование, резервное копирование, усиленная (hardened) безопасность |
| High Availability | Production + 5 CP для Mission Critical, распределение по доменам отказа, PodDisruptionBudget для аддонов, повышенная частота резервного копирования |
| Edge | Минимальное потребление ресурсов (k3s, когда станет доступен), Flannel или минимальный Cilium, local-path, минимальная наблюдаемость, устойчивость к прерывистой связи |
| GPU | Пул GPU-узлов с метками/taints, NVIDIA GPU Operator, мониторинг с дашбордами GPU |
| AI/ML | GPU + хранилище с высоким IO (Rook-Ceph или CSI провайдера), метки узлов для планирования, мониторинг, опционально объектное хранилище |
| Custom | Начинается со значений по умолчанию Production, всё можно редактировать |

## 5. Оценка стоимости (промпт §36)

- Цены возвращают только провайдеры с возможностью `Pricing` (например, API цен Hetzner Cloud, прайс-листы облаков). Расчёт содержит источник и метку времени и кешируется с TTL.
- Разбивка охватывает control plane, рабочие узлы, хранилище, балансировщики нагрузки и прочие позиции (трафик, IP), если провайдер их тарифицирует.
- Для bare metal и существующей инфраструктуры показывается «Оценка стоимости недоступна» (а позже — опционально введённые пользователем внутренние затраты).
- Платформа **никогда не выдумывает и не экстраполирует цены**. Отсутствующие позиции перечисляются как «не включено».
- Корпоративное управление затратами (бюджеты, распределение затрат) строится на тех же расчётах (фаза 14).

## 6. Советник по обновлению (промпт §71, §229)

Советник по обновлению (upgrade advisor) использует те же движки. Он принимает текущее состояние (установленные версии, аддоны, ОС, CNI/CSI/Gateway, CRD) и целевую минорную версию Kubernetes и выдаёт:
- **оценку готовности**: взвешенную долю пройденных проверок;
- **блокеры**: несовместимые версии аддонов, пропуски минорных версий, используемые устаревшие или удалённые API (живое сканирование ресурсов, которые обслуживает кластер), требования к версии etcd, поддержку ОС;
- **предупреждения**: непротестированные сочетания, изменения поведения (например, переход SELinuxMount в GA в 1.37 на узлах с SELinux в режиме enforcing), ожидаемый простой на каждом шаге;
- **план обновления**: control plane по одному узлу (сначала снапшот etcd) → аддоны, которые нужно обновить заранее → рабочие узлы партиями с drain → проверка.
