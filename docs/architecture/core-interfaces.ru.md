# Ключевые интерфейсы и архитектура плагинов

> Статус: **Предложено** (Phase 0). Язык: [English](core-interfaces.md) · Русский
>
> Связанные документы: [ARCHITECTURE](../ARCHITECTURE.ru.md) · [Движок провижининга](provisioning-engine.ru.md) · [Режим Auto, каталог и совместимость](auto-mode-and-catalog.ru.md) · ADR-0010 (модель плагинов)

Этот документ фиксирует основные точки расширения Farvater. Код на Go ниже — это **эскиз проекта**: имена, зоны ответственности и контракты обязательны начиная с Фазы 1, а точные списки полей будут уточняться на код-ревью. Публичные интерфейсы находятся в `pkg/sdk` (плагины) и `pkg/spec` (типы ClusterSpec). Внутренние порты находятся в `internal/…`.

## 1. Принципы

1. **Плагины подключаются на этапе компиляции.** Провайдеры, дистрибутивы и аддоны — это Go-пакеты в `plugins/`, которые реализуют интерфейсы `pkg/sdk` и регистрируются в реестре при запуске (`registry.MustRegisterProvider(...)`). Нет ни `plugin.Open`, ни загрузки кода во время выполнения, ни скриптового языка. Благодаря этому плагины типобезопасны, тестируемы и легко поставляются в составе одного бинарника, а динамический код из недоверенных источников исключён. Внепроцессные плагины (gRPC) можно добавить позже за теми же интерфейсами, если они понадобятся сторонним разработчикам.
2. **Ядро не импортирует плагины.** `internal/engine`, `internal/app` и `internal/domain` зависят только от интерфейсов `pkg/sdk` (инверсия зависимостей, промпт §99). Конкретные плагины подключает к реестру только `cmd/*`. Это правило проверяет линтер (`depguard`).
3. **Сначала декларативность.** Большинство аддонов — это чистые данные (`addon.yaml` + шаблон Helm values), которые обрабатывает одна универсальная реализация. Код на Go пишется только для поведения, которое нельзя выразить данными, например для предварительной установки CRD или вычисления values по фактам о кластере.
4. **Идемпотентная семантика «Ensure»** для всего, что меняет внешний мир. Методы можно безопасно вызывать повторно с теми же входными данными.
5. **Контекст первым аргументом, типизированные результаты, типизированные ошибки** (`pkg/sdk/errs`). Никаких `panic` через границы плагинов: реестр оборачивает вызовы плагинов в recover с последующей классификацией ошибки.
6. **Возможности (capabilities) вместо проверок типа.** Плагины объявляют свои возможности (например, `CreateInstances`, `LoadBalancers`, `Pricing`, `Regions`). Движок и UI подстраиваются под них, а не проверяют `provider == "hetzner"`.

## 2. Реестр

```go
// pkg/sdk/registry.go
type Registry interface {
    RegisterProvider(p InfrastructureProvider) error
    RegisterDistribution(d KubernetesDistribution) error
    RegisterAddon(a Addon) error
    RegisterOSFamily(o OSFamily) error
    RegisterHealthProbe(p HealthProbe) error
    RegisterIntegration(i Integration) error

    Provider(id string) (InfrastructureProvider, bool)
    Distribution(id string) (KubernetesDistribution, bool)
    Addon(id string) (Addon, bool)
    Addons(filter AddonFilter) []Addon
}
```

При запуске реестр **сверяется с каталогом версий**. У каждой записи каталога должна быть реализация, графы возможностей должны быть ацикличными, схемы конфигурации должны компилироваться. Сломанный набор плагинов приводит к немедленному сбою при старте, а не во время развёртывания у клиента.

## 3. Провайдеры инфраструктуры

```go
// pkg/sdk/provider.go
type InfrastructureProvider interface {
    Metadata() ProviderMetadata
    ValidateConfig(ctx context.Context, cfg ProviderConfig) ValidationResult
    Discover(ctx context.Context, cfg ProviderConfig) (*Inventory, error)

    EnsureNetwork(ctx context.Context, c ClusterRef, spec NetworkSpec) (*NetworkStatus, error)
    EnsureInstance(ctx context.Context, c ClusterRef, spec InstanceSpec) (*Instance, error)
    ListInstances(ctx context.Context, c ClusterRef) ([]Instance, error)
    DeleteInstance(ctx context.Context, c ClusterRef, id InstanceID) error

    EnsureLoadBalancer(ctx context.Context, c ClusterRef, spec LBSpec) (*LoadBalancer, error)
    DeleteLoadBalancer(ctx context.Context, c ClusterRef, id string) error

    Status(ctx context.Context, c ClusterRef) (*InfraStatus, error)
    Teardown(ctx context.Context, c ClusterRef) error

    // Pricing returns ErrPricingUnavailable when the provider has no reliable price source. Never estimate.
    Pricing(ctx context.Context, cfg ProviderConfig, q PriceQuery) (*PriceQuote, error)
}

type ProviderMetadata struct {
    ID           string            // "baremetal", "simulated", "hetzner", "aws", "gcp", "azure"
    DisplayName  string            // i18n key
    Kind         ProviderKind      // BareMetal | Existing | VPS | Cloud | Simulated
    Capabilities ProviderCapabilities
    ConfigSchema json.RawMessage   // JSON Schema for ProviderConfig (rendered as a form in UI)
    TestOnly     bool              // true for "simulated": refused unless the server runs with dev/test profile
}

type ProviderCapabilities struct {
    CreateInstances  bool   // false for bare metal / existing hosts (EnsureInstance = adopt + verify)
    DeleteInstances  bool
    PrivateNetworks  bool
    LoadBalancers    bool   // provider-managed LB for the API endpoint and Services
    FloatingIPs      bool
    Regions          bool
    Zones            bool
    Pricing          bool
    GPUInstances     bool
    CloudControllerManager string // e.g. "hcloud-ccm" add-on id to install, "" if none
    CSIDriver              string // e.g. "hcloud-csi"
}
```

Реализации и их статус:

| Провайдер | Фаза | Примечания |
|---|---|---|
| `simulated` | 1 | Фиктивные хосты с настраиваемыми задержками и внедрением сбоев (fault injection). Только для разработки и тестирования (`TestOnly`). Позволяет прогнать весь путь UI → API → движок → SSE без инфраструктуры; в продакшен-сборках никогда не выдаёт себя за настоящий (промпт §115) |
| `baremetal` | 3 | Объявленные хосты, доступные по SSH. `Discover` собирает факты. `EnsureInstance` проверяет доступность, ключ хоста, поддержку ОС и ресурсы. Инстансы не создаются. Также покрывает «существующую инфраструктуру» (промпт §5): предоставленные пользователем узлы CP/worker, внешний LB, эндпоинты хранилищ |
| `hetzner` | 7 | Hetzner Cloud API (серверы, сети, файрволы, балансировщики нагрузки, группы размещения), API цен, аддоны `hcloud-cloud-controller-manager` + CSI |
| `aws`, `gcp`, `azure` | 7 | Реализации на нативных SDK или реализация на базе Cluster API (решение — по ADR-0013 к началу Фазы 7) |
| `digitalocean`, `ovh`, `vultr`, `linode` | позже | Тот же интерфейс. Вклад сообщества приветствуется |

## 4. Дистрибутивы Kubernetes

```go
// pkg/sdk/distribution.go
type KubernetesDistribution interface {
    Metadata() DistributionMetadata
    Validate(ctx context.Context, spec *ResolvedSpec) ValidationResult

    // Contribute returns the distribution's tasks for a plan (init/join/upgrade/remove), wired by the planner.
    Contribute(ctx context.Context, p PlanContext) ([]PlannedTask, error)

    PrepareNode(ctx context.Context, n NodeHandle, spec *ResolvedSpec) error
    BootstrapControlPlane(ctx context.Context, n NodeHandle, spec *ResolvedSpec) (*JoinMaterial, error)
    JoinControlPlane(ctx context.Context, n NodeHandle, jm *JoinMaterial, spec *ResolvedSpec) error
    JoinWorker(ctx context.Context, n NodeHandle, jm *JoinMaterial, spec *ResolvedSpec) error
    Kubeconfig(ctx context.Context, cp NodeHandle) (*Kubeconfig, error)

    UpgradePlan(ctx context.Context, observed ObservedState, to Version) (*UpgradePlan, error)
    UpgradeNode(ctx context.Context, n NodeHandle, role NodeRole, to Version) error
    RemoveNode(ctx context.Context, cp NodeHandle, n NodeHandle) error
    ResetNode(ctx context.Context, n NodeHandle) error

    HealthProbes() []HealthProbe
}

type DistributionMetadata struct {
    ID                 string     // "kubeadm", "k3s", "rke2"
    SupportedMinors    []string   // from catalog, not hard-coded
    SupportedOS        []OSID
    BundledComponents  []string   // e.g. k3s: traefik, servicelb, local-path, flannel → can be disabled
    NeedsBootstrapLayer bool      // kubeadm: true (runtime + packages); k3s/rke2: false (ship their own)
    HA                 HASupport  // stacked etcd, external etcd, embedded etcd, …
}

// JoinMaterial is secret. It is stored only as an encrypted Secret and handed to join tasks via TaskContext.Secrets().
type JoinMaterial struct {
    Endpoint          string
    TokenRef          SecretRef  // bootstrap token
    CACertHashes      []string   // public
    CertificateKeyRef SecretRef  // kubeadm --certificate-key for control-plane join
    ExpiresAt         time.Time
}
```

Реализации: `kubeadm` (Фазы 3–4, полная: один CP, HA со stacked etcd, join, reset, kubeconfig, состояние (health), основа для обновлений (upgrade) — в Фазе 6), `k3s` и `rke2` (Фаза 7), позже — кандидаты `talos` и `k0s`. Логика, специфичная для дистрибутива, никогда не просачивается в движок (промпт §7).

## 5. Доступ к узлам и абстракция ОС

```go
// pkg/sdk/node.go
type NodeHandle interface {
    ID() NodeID
    Address() netip.AddrPort
    Facts(ctx context.Context) (*NodeFacts, error)                 // cached per operation; refresh on demand
    Run(ctx context.Context, cmd Command) (*CommandResult, error)  // non-zero exit → typed error with stderr tail
    Upload(ctx context.Context, dst string, content []byte, mode fs.FileMode) error  // atomic: temp file + rename
    Download(ctx context.Context, src string, maxBytes int64) ([]byte, error)
}

// Command is argv-based. There is no field that accepts a shell script built from user data.
type Command struct {
    Program string            // absolute path or allow-listed binary name
    Args    []string          // each argument validated by its producer; quoted with strict POSIX quoting by the SSH adapter
    Env     map[string]string // allow-listed keys only
    Stdin   []byte
    Sudo    bool              // run via sudo -n (non-interactive); fails clearly if sudo needs a password
    Timeout time.Duration
}

type NodeFacts struct {
    OS            OSInfo        // id (ubuntu/debian/rocky/almalinux), version, codename
    Kernel        string
    Arch          string        // amd64 / arm64
    CPUs          int
    MemoryBytes   uint64
    Disks         []DiskInfo    // mount points, free bytes, filesystem; optional IOPS probe result
    Interfaces    []NetInterface
    CgroupVersion int
    SwapEnabled   bool
    TimeSync      TimeSyncInfo  // service, synced, offset
    Hostname      string
    SELinux, AppArmor string
    Virtualization string
}

// OSFamily maps abstract steps to concrete typed commands for a distro family.
type OSFamily interface {
    ID() string                                   // "debian" (Ubuntu, Debian), "rhel" (Rocky, AlmaLinux)
    Matches(os OSInfo) bool
    Supported(os OSInfo) SupportLevel             // from catalog OS matrix
    InstallPackages(pkgs []PackageRef) []Command
    HoldPackages(names []string) []Command
    AddRepository(repo RepositoryRef) ([]FileWrite, []Command)
    EnableService(name string) []Command
}
```

SSH-адаптер (`internal/adapters/ssh`) реализует `NodeHandle`. Он поддерживает приватный ключ, пароль, SSH-агент, jump-хосты (цепочки bastion) и HTTP/SOCKS-прокси. Проверка ключа хоста обязательна: хост должен быть известен, либо пользователь подтверждает отпечаток ключа (подтверждение TOFU сохраняется в `ssh_known_hosts`). У адаптера есть пул соединений для каждого узла, keep-alive, а также тайм-ауты команд и сессий. Приватные ключи расшифровываются в памяти воркера непосредственно перед использованием (just in time) и никогда не записываются на диск. У симулированного провайдера есть `NodeHandle` в памяти с внедрением сбоев.

## 6. Аддоны

```go
// pkg/sdk/addon.go
type Addon interface {
    Manifest() AddonManifest
    // Render produces what to install for this cluster: Helm release(s) and/or raw manifests.
    Render(ctx context.Context, c ClusterView, cfg AddonConfig) (*RenderResult, error)
    // Optional hooks. Default implementations are no-ops.
    PreInstall(ctx context.Context, k KubeClient, c ClusterView, cfg AddonConfig) error
    PostInstall(ctx context.Context, k KubeClient, c ClusterView, cfg AddonConfig) error
    PreUninstall(ctx context.Context, k KubeClient, c ClusterView) error
    HealthProbes() []HealthProbe
}

type AddonManifest struct {
    ID, Name, Category string      // category: cni, gateway, load-balancer, storage, certificates, metrics, logging,
                                   // tracing, backup, gitops, security, gpu, registry, other
    Description        I18nText
    Pros, Cons         []I18nText  // shown in UI comparison cards (prompt §17)
    UseCases           []I18nText
    Provides           []Capability
    Requires           []Capability
    OptionalRequires   []Capability
    Conflicts          []string    // add-on ids or capabilities
    Ports              []PortRequirement   // for preflight port checks
    ConfigSchema       json.RawMessage     // JSON Schema 2020-12 with if/then for dependent fields (prompt §47)
    Reversible         bool
    Namespace          string
    Docs               string      // relative path to docs page
}

type RenderResult struct {
    Charts    []HelmRelease          // chart ref (OCI or repo) + version + digest + values (map) + namespace + wait options
    Manifests []unstructured.Unstructured // server-side applied, field manager "<product>"
    Images    []string               // full image list for air-gap bundles (also in catalog)
}

// Capability-specific extensions (type-asserted by the planner/UI):
type CNI interface {
    Addon
    CNIRequirements(spec *ResolvedSpec) CNIRequirements // pod CIDR constraints, kube-proxy replacement, kernel modules, ports, MTU
}
type GatewayProvider interface {
    Addon
    GatewayClassName(cfg AddonConfig) string
    SupportedRoutes() []string // HTTPRoute, GRPCRoute, TLSRoute, …
}
type LoadBalancerProvider interface {
    Addon
    Mode() LBMode // l2, bgp, provider, external
}
type StorageProvider interface {
    Addon
    StorageClasses(cfg AddonConfig) []StorageClassInfo
    MinNodes() int // e.g. Longhorn replica count needs ≥ N schedulable nodes
}
```

**Универсальный чарт-аддон** (`plugins/addons/generic`) реализует `Addon` на основе файла:

```yaml
# plugins/addons/cert-manager/addon.yaml (illustrative; versions come from catalog)
id: cert-manager
category: certificates
chart:
  ref: oci://quay.io/jetstack/charts/cert-manager   # resolved + pinned by digest from catalog
namespace: cert-manager
provides: [cert-issuer-crds, certificates]
requires: [cni]
reversible: true
values: values.yaml.tmpl          # Go text/template over typed AddonConfig + ClusterView, NO user strings concatenated into YAML
configSchema: schema.json
health:
  - kind: Deployment
    namespace: cert-manager
    names: [cert-manager, cert-manager-webhook, cert-manager-cainjector]
```

Шаблоны values рендерятся в Go map, которая проверяется по `values.schema.json` чарта, если такой файл есть. Пользовательские переопределения в режиме Advanced (Advanced Mode) объединяются как структурированные данные и никогда — как текст.

## 7. Сервис Helm

```go
// internal/adapters/helm (port defined in pkg/sdk for add-on hooks)
type HelmService interface {
    Template(ctx context.Context, k ClusterAccess, r HelmRelease) ([]byte, error)          // dry-run / plan diff
    Diff(ctx context.Context, k ClusterAccess, r HelmRelease) (*ReleaseDiff, error)
    InstallOrUpgrade(ctx context.Context, k ClusterAccess, r HelmRelease) (*ReleaseStatus, error) // atomic, wait, timeout
    Rollback(ctx context.Context, k ClusterAccess, name, namespace string, revision int) error
    Uninstall(ctx context.Context, k ClusterAccess, name, namespace string) error
    Status(ctx context.Context, k ClusterAccess, name, namespace string) (*ReleaseStatus, error)
}
```

Это единственная реализация поверх встроенного Helm SDK (мажорная версия Helm SDK — см. ADR-0010). Чарты загружаются только по ссылкам, закреплённым в каталоге (с проверкой digest; поддерживаются зеркала для air-gap), и кешируются на воркере в кеше с адресацией по содержимому. Helm никогда не вызывается через shell (промпт §51).

## 8. Проверки состояния (health probes)

```go
type HealthProbe interface {
    Key() string                       // "apiserver", "etcd", "cni", "dns", "gateway", "storage", "certificates", …
    Interval() time.Duration
    Run(ctx context.Context, c ClusterAccess) ProbeResult
}
type ProbeResult struct {
    Status      HealthStatus           // HEALTHY | WARNING | CRITICAL | UNKNOWN
    ReasonCode  string                 // error-catalog code, e.g. CERT_EXPIRING_SOON
    Params      map[string]string      // for i18n rendering
    Details     map[string]any
}
```

## 9. Каталог версий, совместимость, рекомендации (порты)

```go
// internal/catalog
type Catalog interface {
    Version() string                                  // catalog bundle version (semver + date)
    KubernetesMinors() []KubernetesMinor              // status: current | supported | deprecated | blocked; patches; EOL
    Resolve(spec *spec.ClusterSpec) (*ResolvedSpec, []Finding, error)
    Addon(id string, version string) (*AddonRelease, bool)
    OSSupport(os OSInfo, dist string, k8sMinor string) SupportLevel
}

// internal/compat
type CompatibilityEngine interface {
    Check(ctx context.Context, rs *ResolvedSpec, facts *InventoryFacts) []Finding  // facts optional (pre-preflight)
}

type Finding struct {
    Severity    Severity      // BLOCK | WARNING | INFO
    Code        string        // e.g. CILIUM_K8S_UNSUPPORTED
    Path        string        // JSON pointer into the spec, e.g. /spec/networking/cni
    Params      map[string]string
    Remediation []string      // i18n keys
}

// internal/recommend
type Recommender interface {
    Recommend(ctx context.Context, in RecommendationInput) (*Recommendation, error)
}
type Recommendation struct {
    Spec         *spec.ClusterSpec
    Decisions    []Decision      // per field: value, reasons, alternatives, risk, pinned?
    Warnings     []Finding       // safety warnings (prompt §132)
    Estimate     Estimate        // time, cost (or unavailable)
}
```

Подробнее: [Режим Auto, каталог и совместимость](auto-mode-and-catalog.ru.md).

## 10. Секреты, ключи, интеграции

```go
// internal/secrets
type KeyProvider interface {                 // KEK operations for envelope encryption
    ID() string                              // "local", "openbao-transit", "aws-kms", "gcp-kms", "azure-keyvault"
    Wrap(ctx context.Context, dek []byte, aad []byte) (WrappedKey, error)
    Unwrap(ctx context.Context, wk WrappedKey, aad []byte) ([]byte, error)
    Rotate(ctx context.Context) (newKeyVersion string, err error)
}

type SecretBackend interface {               // where secret payloads live
    ID() string                              // "internal" (encrypted rows in PostgreSQL), "openbao", "vault", "aws-sm", …
    Put(ctx context.Context, scope TenantScope, ref SecretRef, value []byte) error
    Get(ctx context.Context, scope TenantScope, ref SecretRef) ([]byte, error)
    Delete(ctx context.Context, scope TenantScope, ref SecretRef) error
}

// pkg/sdk/integration.go
type Integration interface {
    Type() string                            // "slack", "telegram", "email", "webhook", "jira", "splunk", …
    ConfigSchema() json.RawMessage
    Validate(ctx context.Context, cfg IntegrationConfig) ValidationResult
    TestConnection(ctx context.Context, cfg IntegrationConfig) (*TestResult, error)
}
type Notifier interface {
    Integration
    Notify(ctx context.Context, cfg IntegrationConfig, n Notification) error
}
```

Исходящий HTTP-трафик интеграций, вебхуков, Helm-репозиториев и Git всегда проходит через `internal/adapters/httpx` (защита от SSRF, тайм-ауты, ограничения размера). См. [модель безопасности](../security/SECURITY_MODEL.ru.md).

## 11. Порты прикладного слоя (внутренние)

```go
// internal/app/ports.go (selection)
type TxManager interface {
    InTx(ctx context.Context, scope TenantScope, fn func(ctx context.Context, tx Tx) error) error // sets app.org_id for RLS
}
type JobQueue interface {
    EnqueueTx(ctx context.Context, tx Tx, job JobArgs, opts JobOpts) error  // River InsertTx
}
type EventPublisher interface {
    PublishTx(ctx context.Context, tx Tx, e OperationEvent) error          // insert + NOTIFY on commit
}
type Authorizer interface {
    Authorize(ctx context.Context, p Principal, perm Permission, res ResourceRef) error // ErrForbidden with reason
}
type AuditLogger interface {
    RecordTx(ctx context.Context, tx Tx, e AuditEvent) error
}
type EntitlementService interface {          // single place for edition/feature checks (prompt §218)
    Has(ctx context.Context, f Feature) bool // "enterprise.sso", "enterprise.fleet", …
    Limits(ctx context.Context) Limits
}
type Clock interface { Now() time.Time }
```

## 12. Точки расширения Enterprise

Код Enterprise в `ee/` (коммерческая лицензия, тег сборки `ee`) подключается к тому же ядру через интерфейсы и никогда — через ветвления `if enterprise`:

| Точка расширения | Реализация в ядре по умолчанию (Community) | Реализация в ee |
|---|---|---|
| `IdentityProvider` (способы входа) | локальный пароль | OIDC, SAML 2.0 (Okta, Entra ID, Google Workspace, Keycloak, Auth0, универсальный провайдер) |
| `Authorizer` | RBAC со встроенными ролями | Пользовательские роли + условия ABAC (движок политик) |
| `PolicyEvaluator` (проверки перед развёртыванием) | встроенные ограничители (guardrails) — предупреждения о безопасности | политики как код (policy as code) с областями действия, наследованием, уровнями BLOCK/WARN/INFO; адаптеры OPA/Rego или Cedar |
| `ApprovalGate` (перед выполнением операции) | всегда «одобрено» | запросы на изменение (change requests), политики согласования, окна обслуживания |
| `AuditSink` | таблица аудита в PostgreSQL + экспорт в JSON/CSV | SIEM-вебхук, syslog, S3, Splunk/Datadog/Elastic/Sentinel |
| `KeyProvider` | локальный KEK | OpenBao/Vault Transit, AWS/GCP/Azure KMS, BYOK |
| `FleetController` | операции над одним кластером | группы кластеров, пакетное и канареечное выкатывание на парк кластеров |
| `EntitlementService` | права (entitlements) редакции Community | проверяет файл лицензии с подписью Ed25519, предоставляет доступные функции и лимиты |

Редакция Community остаётся полнофункциональной для провижининга (provisioning) и управления жизненным циклом. Enterprise добавляет корпоративное управление (governance), масштабирование и интеграции и никогда не урезает базовые возможности (см. ADR-0002).
