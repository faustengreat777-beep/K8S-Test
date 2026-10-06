# Движок провижининга

> Статус: **Предложено** (Phase 0). Язык: [English](provisioning-engine.md) · Русский
>
> Связанные документы: [ARCHITECTURE](../ARCHITECTURE.ru.md) · [Ключевые интерфейсы](core-interfaces.ru.md) · [Модель данных](data-model.ru.md) · [Режим Auto, каталог и совместимость](auto-mode-and-catalog.ru.md) · ADR-0007 (очередь заданий), ADR-0008 (модель движка), ADR-0009 (обновления в реальном времени)

Движок провижининга (provisioning engine) превращает желаемую **ClusterSpec** в работающий кластер, а затем безопасно вносит в него изменения: обновления (upgrade), масштабирование, аддоны, резервное копирование, восстановление и удаление. Это самый важный компонент Farvater, поэтому в этом документе его контракт задан точно.

## 1. Цели проектирования

| Цель | Как движок её достигает |
|---|---|
| Никаких гигантских `if/else` | Работа — это **DAG (граф задач) из небольших задач**, которые поставляют плагины (провайдер, слой bootstrap, дистрибутив, аддоны). Ядро движка ничего не знает о kubeadm, Cilium или Hetzner. |
| Идемпотентность | У каждой задачи семантика *ensure / converge* и необязательная проверка `Check`. Повторный запуск завершённой задачи безопасен и дёшев. |
| Возобновляемость | Каждое изменение состояния сохраняется **до** и **после** выполнения задачи. Упавший воркер (worker) или закрытый браузер ничего не теряют. Возобновление (Resume) продолжает работу с первой незавершённой задачи. |
| Наблюдаемость | Каждый переход и каждая строка лога становятся `OperationEvent`. Событие передаётся клиентам по SSE и хранится для аудита и поиска. |
| Безопасный откат | Автоматически откатываются только задачи, объявленные как `Reversible`. Необратимые шаги помечаются в плане, и пользователь их подтверждает. |
| Понятные причины сбоев | Типизированные ошибки сообщают *что / почему / где / как исправить*, а также перечисляют действия, доступные пользователю. |
| Детерминированные планы | Та же ResolvedSpec + то же наблюдаемое состояние → тот же план. План сохраняется и хешируется, поэтому его можно просмотреть до применения (пробный прогон, dry run). |
| Тестируемость | Ядро движка написано на чистом Go с интерфейсами для времени, хранилища, очереди и плагинов. Симулированный провайдер и внедрение сбоев (fault injection) позволяют прогонять полные сценарии в модульных и интеграционных тестах. |

## 2. Жизненный цикл изменения

```
ClusterSpec (UI / YAML / CLI / API / template / Auto Mode)
   │
   ▼
1. Validate        schema (strict) → semantic validators → secrets-free check
2. Resolve         catalog: minor → pinned patch, chart versions + digests, images  → ResolvedSpec
3. Check           compatibility engine (BLOCK/WARN/INFO) → guardrails / policies → (ee) approvals
4. Preflight       live checks against hosts / provider (async operation of type `preflight`)
5. Plan            planner builds DAG from ResolvedSpec + observed state → diff, estimates, irreversible flags
6. Confirm         user reviews plan ("Deploy Cluster"); idempotency key binds the request
7. Execute         worker runs the DAG with leases, retries, events
8. Verify          health probes; cluster → READY (or PAUSED/FAILED with remediation)
```

Шаги 1–5 не имеют побочных эффектов для инфраструктуры. На них построены **план / предпросмотр / применение** (Plan / Preview / Apply) повсюду в продукте («dry run везде», промпт §117). Шаг 4 читает данные с хостов (факты по SSH, порты), но ничего не меняет.

## 3. Доменные объекты

- **ClusterSpec**: желаемое состояние, которое задаёт пользователь (см. [ARCHITECTURE, §ClusterSpec](../ARCHITECTURE.ru.md)). Она никогда не содержит секретов — только ссылки `credentialRef`.
- **ResolvedSpec**: ClusterSpec, в которой закреплены все версии и материализованы все значения по умолчанию, плюс версия каталога, использованная при разрешении. Хранится в `cluster_spec_revisions.resolved_spec`.
- **ObservedState**: то, что платформа знает о реальном положении дел, т. е. узлы и их факты, установленные аддоны и их версии, адрес API и состояние (health). Обновляется задачами обнаружения (discovery) и пробами состояния (health probes).
- **Plan**: DAG из `PlannedTask`, а также понятный человеку **diff** (`+ 3 control-plane nodes`, `~ cilium 1.19.4 → 1.20.2`, `- addon loki`), необратимые шаги, оценка времени и оценка стоимости (по ценам провайдера или «недоступно»). План хешируется (`plan_hash`).
- **Operation**: одно выполнение плана (`create`, `apply`, `upgrade`, `scale`, `addon-install`, `addon-remove`, `backup`, `restore`, `rotate-*`, `node-replace`, `import`, `destroy`, `preflight`). Именно это в промпте называется *Deployment*.
- **OperationTask**: одна вершина DAG со своим статусом, попытками, таймингами, ошибкой и выходными данными (*DeploymentTask* в терминах промпта).
- **OperationEvent**: событие, которое только дописывается (append-only): строка лога, смена статуса, отметка прогресса. Его `id` служит идентификатором события SSE.

## 4. Планировщик

Планировщик (planner) — это чистая функция:

```
Plan(resolved ResolvedSpec, observed ObservedState, registry PluginRegistry) (Plan, error)
```

Он запрашивает у каждого участвующего плагина **вклад в задачи** (task contributions):

| Источник | Примеры ключей задач | Примечания |
|---|---|---|
| Движок (в начале) | `preflight.verify` | Повторно проверяет факты, собранные операцией preflight; сразу завершается ошибкой, если они устарели |
| Провайдер инфраструктуры | `infra.network.ensure`, `infra.instance.ensure/cp-1`, `infra.lb.ensure` | Bare metal: `instance.ensure` = взять под управление (adopt) и проверить объявленный хост |
| Слой bootstrap (на каждом узле) | `node/cp-1/os.prepare`, `…/kernel.modules`, `…/sysctl`, `…/swap.disable`, `…/time.sync`, `…/runtime.install`, `…/k8s.packages` | Общий для всех дистрибутивов, которым он нужен; дистрибутив может от него отказаться (k3s поставляет собственную среду выполнения) |
| Дистрибутив | `k8s.controlplane.init`, `k8s.controlplane.join/cp-2`, `k8s.worker.join/w-1`, `k8s.kubeconfig.fetch`, `k8s.node.upgrade/cp-1` | В первую очередь kubeadm |
| Аддоны | `addon/gateway-api-crds.install`, `addon/cilium.install`, `addon/metallb.install`, `addon/cert-manager.install`, … | Порядок выводится из возможностей (capabilities) аддонов `requires`/`provides` |
| Движок (в конце) | `health.verify`, `cluster.ready` | Пробы состояния от всех плагинов |

**Разрешение зависимостей аддонов.** Манифест каждого аддона объявляет возможности:

```yaml
# catalog/addons/cert-manager.yaml (excerpt)
id: cert-manager
provides: [cert-issuer-crds, certificates]
requires: [cni]                 # nothing installs before networking is up
optionalRequires: [metrics]     # if metrics present, install ServiceMonitor after it
conflicts: []
reversible: true
```

```yaml
# catalog/addons/kube-prometheus-stack.yaml (excerpt)
id: kube-prometheus-stack
provides: [metrics, alerting, dashboards]
requires: [cni, default-storage-class]
```

Планировщик сопоставляет `requires` с аддоном, который `provides` эту возможность *в данном кластере* (например, `default-storage-class` → `longhorn` или `local-path`). Затем он строит рёбра и топологически сортирует граф. Планировщик отклоняет:
- неудовлетворённые требования («Prometheus нужен StorageClass по умолчанию; выберите провайдер хранилища или отключите постоянное хранение»),
- конфликты (два CNI, два StorageClass по умолчанию),
- циклы. Они отклоняются ещё при загрузке каталога, поэтому цикл никогда не доходит до пользователей.

**Diff, а не пересборка.** Для `apply`, `scale` и `upgrade` планировщик сравнивает ResolvedSpec с ObservedState и порождает только нужные задачи. Добавление двух рабочих узлов порождает `infra.instance.ensure/w-4,w-5` → bootstrap → `k8s.worker.join/w-4,w-5` → `health.verify`. Изменение values Longhorn порождает единственную задачу `addon/longhorn.upgrade`.

**Флаги необратимости.** Задачи объявляют `Reversible: false` вместе с классом риска (`data-loss`, `downtime`, `security`, `cost`). План перечисляет их в разделе «Необратимые / рискованные шаги». Для планов с такими задачами API требует явного подтверждения (`confirmIrreversible: true`); исключение — первоначальный `create` пустого кластера.

**Оценки.** Оценка времени опирается на медианную длительность задач того же вида в предыдущих операциях данной инсталляции. Если истории нет, используются значения по умолчанию из каталога. Критический путь через DAG даёт «Ориентировочное время развёртывания: 12–18 мин». Оценка стоимости берётся только из возможности `Pricing` провайдера; в противном случае план сообщает «Оценка стоимости недоступна». Движок никогда не выдумывает цены.

### Пример: создание HA-кластера kubeadm на bare metal (упрощённый DAG)

```
preflight.verify
  └─▶ node/*/os.prepare ─▶ node/*/kernel.modules ─▶ node/*/sysctl ─▶ node/*/swap.disable ─▶ node/*/runtime.install ─▶ node/*/k8s.packages
                                                                                                      │ (all nodes, in parallel)
        node/cp-1/k8s.packages ─▶ addon/kube-vip.static-pod(cp-1) ─▶ k8s.controlplane.init(cp-1) ─▶ k8s.kubeconfig.fetch
                                                                                                      │
              addon/gateway-api-crds.install ◀────────────────────────────────────────────────────────┤
              addon/cilium.install  ◀── requires gateway-api-crds (Cilium Gateway) ───────────────────┘
                     │
        ┌────────────┼───────────────────────────────┐
        ▼            ▼                               ▼
 k8s.controlplane.join(cp-2)  k8s.controlplane.join(cp-3)   k8s.worker.join(w-1..w-3)   (CP joins serialized: one etcd member at a time)
        └────────────┴───────────────┬───────────────┘
                                     ▼
          addon/coredns.verify ─▶ addon/metallb.install ─▶ addon/gateway.install ─▶ addon/longhorn.install
                 ─▶ addon/cert-manager.install ─▶ addon/metrics-server.install ─▶ addon/kube-prometheus-stack.install
                 ─▶ addon/loki.install ─▶ addon/backup.configure ─▶ health.verify ─▶ cluster.ready
```

Узлы control plane присоединяются строго по очереди: изменения членства в etcd должны происходить по одному, чтобы не поставить под угрозу кворум. Рабочие узлы присоединяются параллельно — в пределах лимита параллелизма на кластер.

## 5. Исполнитель

### 5.1 Модель заданий

- Создание операции — это **одна транзакция БД**: вставить строку `operations`, вставить строки `operation_tasks` из плана, установить фазу кластера, записать событие аудита, добавить событие в outbox (`deployment.started`) и **поставить в очередь задание River** `operation.run{operationID}`. Всё это фиксируется атомарно, поэтому не бывает ни задания без операции, ни операции без задания.
- `operation.run` — задание River с опциями `unique`, ключом которых служит id операции. Дубликаты невозможны.
- Обработчик задания захватывает **аренду выполнения** (execution lease) на строке операции: `lease_owner = <worker-id>`, `lease_expires_at = now() + 30s`; аренда продлевается каждые 10 с через heartbeat. Если воркер умирает, аренда истекает. River повторяет задание (или его заново ставит в очередь чистильщик зависших операций, stuck-operation sweeper), и работу продолжает новый воркер.
- Внутри задания встроенный в процесс **диспетчер DAG** (DAG scheduler) параллельно запускает готовые задачи, ограничивая их семафорами:
  - глобальным на воркер (`ENGINE_MAX_PARALLEL_TASKS`, по умолчанию 32),
  - на кластер (по умолчанию 10),
  - на узел (по умолчанию 1 изменяющая задача за раз, чтобы пакетные менеджеры не боролись за блокировки).
- Во время долгих ожиданий (например, `kubeadm init`, который длится минуты) аренда остаётся живой благодаря heartbeat. Задачи получают `context.Context`, который отменяется при потере аренды, отмене операции или по таймауту.

Почему одно задание на операцию, а не по заданию на каждую задачу: диспетчеризации DAG, лимитам параллелизма и политике обработки сбоев нужен глобальный взгляд на операцию. Единственный владелец всё это упрощает. При этом сохранение после каждого перехода задачи всё равно даёт возобновление на уровне отдельных задач. ADR-0008 фиксирует это решение и отклонённую альтернативу (workflows на Temporal).

### 5.2 Контракт задачи

```go
// pkg/sdk/task.go (sketch)
type Task interface {
    Key() TaskKey                       // stable id inside the plan, e.g. "node/cp-1/runtime.install"
    Meta() TaskMeta                     // name i18n key, kind, node, reversible, risk class, timeout, retry policy
    // Check reports whether the desired end state is already reached. Optional (ErrNotImplemented → treated as false).
    Check(ctx context.Context, tc TaskContext) (done bool, err error)
    // Run converges the system to the desired state. Must be idempotent and safe to re-run after partial completion.
    Run(ctx context.Context, tc TaskContext) error
    // Rollback undoes Run for reversible tasks. Must be idempotent. Called only if Meta().Reversible.
    Rollback(ctx context.Context, tc TaskContext) error
}

type TaskContext interface {
    Logger() *slog.Logger               // pre-tagged with cluster/operation/task/node ids; secret values redacted
    Progress(pct int, msgKey string, args ...any)
    Node(id NodeID) (NodeHandle, error) // SSH-backed or simulated
    Kube() (KubeClient, error)          // client for the target cluster (after kubeconfig is fetched)
    Helm() HelmService
    Secrets() SecretAccessor            // just-in-time decryption, scoped to this operation's tenant
    Outputs() OutputStore               // pass non-secret outputs to dependent tasks (e.g. join endpoint)
    Spec() *ResolvedSpec
    Observed() ObservedStateReader
}
```

Правила для авторов задач (их соблюдение обеспечивают ревью, тесты и, где возможно, линтеры):
1. **Идемпотентность.** Используйте логику «ensure»: сначала проверьте текущее состояние, затем действуйте. Например, записывайте конфигурационный файл, только если его контрольная сумма отличается, и запускайте `kubeadm init`, только если `/etc/kubernetes/admin.conf` не существует и API недоступен.
2. **Никаких shell-строк с данными.** Удалённые команды имеют вид `Command{Program, Args[], Env, Stdin, Timeout}`. Аргументы — типизированные значения, прошедшие валидаторы. Конфигурационные файлы рендерятся из структур Go и загружаются по SFTP; они никогда не собираются через `echo … >`.
3. **Типизированные ошибки.** Возвращайте `errs.Transient(...)`, `errs.Permanent(...)`, `errs.UserAction(code, remediation...)`. Неизвестные ошибки считаются постоянными.
4. **Секреты** берутся только из `tc.Secrets()`. Никогда не пишите их в лог, не помещайте в выходные данные и не встраивайте в текст ошибок. Кроме того, маскировщик (redactor) скрывает каждое значение, полученное через `Secrets()`, во всех строках лога операции.
5. **Ограниченность.** У каждого удалённого вызова есть таймаут. Циклы опроса используют backoff и учитывают `ctx.Done()`.

### 5.3 Конечный автомат задачи

```
PENDING ──▶ RUNNING ──▶ SUCCEEDED
   │           │  └──▶ RETRYING ──▶ RUNNING (attempt+1, after backoff)
   │           └─────▶ FAILED ──(user Retry / Resume)──▶ PENDING
   ├──▶ SKIPPED            (Check() == done, or dependency not needed in this plan)
   └──▶ CANCELLED          (operation cancelled before start)
SUCCEEDED ──(rollback)──▶ ROLLED_BACK   (only reversible tasks)
```

Задача становится *готовой*, когда все её `depends_on` находятся в состоянии `SUCCEEDED` или `SKIPPED`. Зависимость в состоянии `FAILED` блокирует зависящие от неё задачи, и они остаются в `PENDING`.

### 5.4 Конечные автоматы операции и кластера

Операция: `PENDING → RUNNING → SUCCEEDED | FAILED | CANCELLED`, а также `RUNNING → PAUSED` (политика обработки сбоев *pause*), `PAUSED → RUNNING` (возобновление/повтор), `PAUSED|FAILED → ROLLING_BACK → ROLLED_BACK` и `PAUSED → CANCELLED` (прерывание, abort).

Фаза кластера выводится из прогресса активной операции и сохраняется, чтобы по ней можно было делать запросы:

| Контекст операции | Фаза кластера во время выполнения |
|---|---|
| create: валидация/preflight | `VALIDATING → VALIDATED` |
| create: инфраструктурные задачи | `PROVISIONING` |
| create: подготовка узлов + control plane | `BOOTSTRAPPING` |
| create: установка аддонов | `INSTALLING` |
| create: настройка после установки (issuers, gateways, дашборды, bootstrap GitOps) | `CONFIGURING` |
| любая: проверка состояния | `HEALTH_CHECK` |
| успех | `READY` |
| apply / upgrade / scale / операции с аддонами на кластере в состоянии READY | `UPDATING → READY` |
| сбой при политике pause | `PAUSED` |
| неустранимый сбой / прерывание create | `FAILED` |
| идёт откат | `ROLLING_BACK` |
| destroy | `DESTROYING → DESTROYED` |

Все переходы проходят через `domain.Cluster.Transition(to, reason)`: метод сверяется с явной таблицей разрешённых переходов и увеличивает `version` (оптимистичная блокировка). Недопустимый переход — это ошибка программирования: он логируется с уровнем ERROR и приводит к сбою операции. Произвольные запросы `UPDATE … SET phase` не допускаются.

## 6. Повторы, таймауты, circuit breakers

- **Политика повторов для каждой задачи** (значения по умолчанию зависят от вида задачи и переопределяются в каталоге): `maxAttempts` (по умолчанию 3; для установки пакетов — 5; для вызовов облачных API — 8), экспоненциальный backoff `base=2s, factor=2, max=2m`, full jitter. Повторяются только ошибки `Transient`.
- **Классификация ошибок** выполняется в адаптерах, как можно ближе к источнику:
  - SSH: connection refused, таймаут, `EOF` во время handshake → временная (transient); ошибка аутентификации → `UserAction(SSH_AUTH_FAILED)`; несовпадение ключа хоста → `UserAction(SSH_HOST_KEY_MISMATCH)` (никогда не принимается автоматически).
  - Пакетные менеджеры: блокировка занята (`dpkg lock`) → временная; 404 от репозитория → постоянная (permanent) с рекомендацией проверить «catalog/mirror».
  - Kubernetes API: 429/5xx/сброс соединения → временная; ошибка валидации 4xx → постоянная.
  - Helm: таймаут ожидания готовности → временная (с ограничением числа попыток); ошибка рендеринга чарта → постоянная.
  - Облачные провайдеры: rate limit → временная с соблюдением `Retry-After`; превышение квоты → `UserAction(PROVIDER_QUOTA)`.
- **Таймауты** действуют на трёх уровнях: на удалённую команду, на задачу (`Meta().Timeout`), на операцию (мягкий лимит: операция приостанавливается с пометкой «время истекло», чтобы человек мог разобраться).
- **Circuit breakers** оборачивают клиенты API провайдеров и исходящие вебхуки. Когда API провайдера раз за разом отказывает, новые задачи сразу завершаются с понятным сообщением «API провайдера недоступен», а не повторяют попытки каждая по отдельности.

## 7. Обработка сбоев, возобновление и откат

**Политика обработки сбоев по умолчанию: `pause`.** При сбое задачи, который нельзя повторить (или после исчерпания повторов):
1. Уже выполняющиеся параллельные задачи завершаются или отменяются в безопасных точках. Новые задачи не запускаются.
2. Операция переходит в `PAUSED`, кластер — тоже в `PAUSED` (или в `FAILED`, если пригодного к использованию ещё ничего нет и действует политика `abort`).
3. **Отчёт о сбое** сохраняется в `operations.error` и показывается пользователю:

```
Worker node node-04 failed to join the cluster.
Reason:   The node cannot reach the Kubernetes API server on port 6443.
Where:    task k8s.worker.join/node-04 · node 10.0.0.14 · attempt 3/3
How to fix:
  1. Check firewall rules between 10.0.0.14 and 10.0.0.10:6443.
  2. Check routing (traceroute from node-04 to the API endpoint).
  3. Verify the API server address (control-plane endpoint 10.0.0.10).
[Retry task] [Resume] [Roll back add-ons] [View logs] [Abort]
```

Текст берётся из **каталога ошибок**: стабильный код (`K8S_JOIN_API_UNREACHABLE`), ключи i18n для заголовка, причины и шагов по исправлению, а также параметры (узел, адрес, порт), которые заполняются из полей типизированной ошибки. Каталог общий для API (`problem+json`), UI, CLI и уведомлений. «Голый» `exit status 1` никогда не показывается. Исходный stderr прикладывается к записи лога как подробности для экспертов.

**Возобновление (Resume)** (выполняется и автоматически после падения воркера):
1. Заново захватить аренду. Перезагрузить операцию и её задачи.
2. Сверить `plan_hash` с текущей ResolvedSpec. Если пользователь изменил спецификацию после построения плана, возобновление отклоняется с сообщением «спецификация изменилась, постройте план заново».
3. Задачи в `SUCCEEDED`/`SKIPPED` остаются выполненными. Задачи, застигнутые в `RUNNING` (падение посреди задачи), сбрасываются в `PENDING`. Поскольку они идемпотентны, а `Check()` позволяет сразу пропустить уже сделанное, их повторный запуск безопасен. Задачи в `FAILED` сбрасываются в `PENDING` только явным действием пользователя — повтором (Retry) или возобновлением (Resume).
4. Продолжить диспетчеризацию.

**Откат** (действие пользователя или автоматически при политике `rollback`):
- Движок обходит **обратимые** задачи в состоянии `SUCCEEDED` в обратном топологическом порядке и вызывает `Rollback`. Например, установка аддона откатывается к предыдущей ревизии Helm, а если аддон был установлен впервые, он удаляется.
- Необратимые задачи никогда не откатываются автоматически. Если такие задачи есть за границей отката, UI объясняет, в каком состоянии всё осталось, и предлагает документированные варианты: оставить частично созданный кластер и довести его до нужного состояния исправлениями (fix forward) или выполнить `destroy`.
- Пример (промпт §13): `Install addon → Health check failed → Rollback addon`; для каждого аддона реализуется парой `addon/<id>.install` (Reversible) + `addon/<id>.health` (при политике `rollback` сбой запускает откат только этого аддона).

**Отмена**: `POST /operations/{id}/cancel` устанавливает `cancel_requested`, отправляет `NOTIFY operation_cancel`, и исполнитель отменяет контексты задач. Задачи останавливаются в безопасных точках. Частично применённые шаги остаются в состоянии, из которого может продолжить `Resume`.

## 8. Гарантии параллелизма и согласованности

| Проблема | Механизм |
|---|---|
| Две операции изменяют один и тот же кластер | Частичный уникальный индекс `operations_one_active_per_cluster` (обеспечивается БД); API возвращает `409 OPERATION_IN_PROGRESS` со ссылкой на выполняющуюся операцию |
| Повторные отправки (двойной клик, клиент с повторами) | Заголовок `Idempotency-Key` → возвращается та же операция; уникальные задания River |
| Два воркера выполняют одну и ту же операцию | Аренда выполнения с heartbeat и fencing (каждая запись проверяет `lease_owner = me`) |
| Две задачи одновременно обращаются к пакетному менеджеру одного узла | Семафор на узел внутри исполнителя + `flock` на узле для операций с пакетами |
| Потерянные обновления строк кластера | Оптимистичная блокировка (`version`), защитные проверки переходов в доменном слое |
| События видны в UI до фиксации транзакции | События записываются в той же транзакции, что и описываемое ими изменение состояния; NOTIFY срабатывает при фиксации |

## 9. События, логи и доставка в реальном времени

- Каждый переход задачи и каждая строка лога → `operation_events` (см. [модель данных, §3.6](data-model.ru.md)). Событие структурировано: `ts, level, cluster_id, operation_id, task_id, node_id, type, message, fields`.
- Перед сохранением логи проходят через **маскировщик** (redactor). Он скрывает значения, зарегистрированные через `Secrets()`, маскирует известные шаблоны (приватные ключи, bearer-токены, `password=`) и отбрасывает поля из denylist.
- **SSE**-эндпоинт `GET /api/v1/operations/{id}/events` (`text/event-stream`):
  - При подключении сервер воспроизводит события с `id > Last-Event-ID` (или с самого начала), а затем транслирует новые события в реальном времени.
  - Пробуждение в реальном времени: после фиксации транзакции воркер выполняет `NOTIFY operation_events, '<operation_id>:<event_id>'`. Каждая реплика API выполняет `LISTEN` и отправляет новые строки своим подписчикам. Такая схема масштабируется горизонтально без Redis.
  - Комментарий-heartbeat каждые 15 с не даёт прокси закрыть поток. Подсказка `retry:` — 3 с.
  - Поток завершается событием `event: end`, когда операция достигает терминального состояния.
- UI восстанавливает полное дерево задач по `GET /operations/{id}` + `/tasks`, а затем применяет дельты из SSE. Если закрыть браузер и вернуться, отображается «Идёт развёртывание» с полной историей (промпт §66).
- ADR-0009 фиксирует, почему выбран SSE, а не WebSocket. WebSocket оставлен для будущего интерактивного терминала узла.

## 10. Слой bootstrap (подготовка узлов)

Слой bootstrap — это набор переиспользуемых построителей задач, учитывающих особенности ОС; ими пользуются дистрибутивы:

```
Node ─▶ facts (OS, kernel, arch, CPU, RAM, disks, NICs, time sync, cgroup v2, SELinux/AppArmor)
     ─▶ os.prepare        (packages: conntrack, socat, ebtables/nftables, chrony; hostname; /etc/hosts entries)
     ─▶ kernel.modules     (overlay, br_netfilter; CNI-specific modules from CNIRequirements)
     ─▶ sysctl             (net.ipv4.ip_forward=1, bridge-nf-call-iptables=1, CNI extras) via /etc/sysctl.d/90-<product>.conf
     ─▶ swap.disable       (or configure NodeSwap if the spec enables it and the K8s version supports it)
     ─▶ time.sync          (chrony enabled and synced; skew check)
     ─▶ runtime.install    (containerd from catalog version; config rendered from struct; SystemdCgroup=true)
     ─▶ k8s.packages       (kubeadm/kubelet/kubectl from pkgs.k8s.io or offline mirror; held at catalog version)
     ─▶ distribution steps (init / join)
```

Работа с ОС скрыта за стратегией `OSFamily` (`debian` охватывает Ubuntu и Debian; `rhel` — Rocky и Alma, после фазы 3). Каждая стратегия отображает абстрактные шаги (`InstallPackages`, `HoldPackages`, `EnableService`, `AddRepository`) на типизированные команды. Новая ОС добавляется новой стратегией, а не правкой задач.

## 11. Движок состояния (health engine)

- Интерфейс `HealthProbe`: `Key()`, `Interval()`, `Run(ctx, ClusterAccess) ProbeResult{Status, Reason, Details, Remediation}`.
- Встроенные пробы: API server (`/readyz`), etcd (состояние членов кластера etcd через API server или etcdctl на узлах control plane), scheduler, controller-manager, условия узлов (node conditions), CNI (DaemonSet агента готов + под проверки связности), DNS (резолв `kubernetes.default` из пода пробы), Gateway (GatewayClass принят, Gateway в состоянии Programmed), хранилище (StorageClass по умолчанию + provisioner готов; необязательный smoke-тест PVC — только во время `health.verify`), metrics-server, стек мониторинга, сертификаты (срок действия сертификатов control plane и ресурсов Certificate cert-manager; WARN < 30 дней, CRITICAL < 7 дней), резервные копии (последняя успешная — в пределах политики).
- Во время операций `health.verify` запускает все пробы со строгими порогами. Помимо этого, для каждого кластера работает периодическое задание River (по умолчанию каждые 60 с для кластеров в состоянии READY), которое обновляет `cluster_health_checks` и агрегированное состояние кластера: берётся худшее из всех с отдельной обработкой `UNKNOWN`. Переходы порождают уведомления и события outbox (позже в ee — инциденты).

## 12. Расширение движка

- **Новые виды задач** — это обычные типы Go, реализующие `Task`. Они регистрируются через вклад плагина в планировщик, а ядро движка не меняется.
- **Новые типы операций** требуют функции планировщика `(ResolvedSpec, ObservedState) → []PlannedTask`, записи в таблице типов операций (соответствие фазам кластера, требуемое разрешение, действие аудита, политика обработки сбоев по умолчанию) и эндпоинта API.
- **Новые плагины** (провайдеры, дистрибутивы, аддоны) лишь реализуют интерфейсы из `pkg/sdk` (см. [Ключевые интерфейсы](core-interfaces.ru.md)).

## 13. Чего движок намеренно не делает

- Не запускает на узлах произвольные пользовательские скрипты. Экспертная «продвинутая» лазейка (advanced) ограничена валидируемыми патчами конфигурации kubeadm, kubelet и containerd, а также типизированными хуками.
- Не прячет необратимые действия за автоматическим откатом.
- Не хранит состояние только в памяти. Если этого нет в PostgreSQL, этого не было.
- Не обращается к инфраструктуре из процесса API. Задачи выполняют только воркеры, поэтому учётные данные не попадают на уровень, доступный из интернета.
