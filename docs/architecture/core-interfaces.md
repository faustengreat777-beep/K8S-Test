# Core interfaces and plugin architecture

> Status: **Proposed** (Phase 0). Language: English · [Русский](core-interfaces.ru.md)
>
> Related: [ARCHITECTURE](../ARCHITECTURE.md) · [Provisioning engine](provisioning-engine.md) · [Auto Mode, catalog and compatibility](auto-mode-and-catalog.md) · ADR-0010 (plugin model)

This document fixes the main extension points of Farvater. The Go code below is a **design sketch**: names, responsibilities and contracts are binding for Phase 1+, while exact field lists will be refined in code review. Public interfaces live in `pkg/sdk` (plugins) and `pkg/spec` (ClusterSpec types). Internal ports live under `internal/…`.

## 1. Principles

1. **Compile-time plugins.** Providers, distributions and add-ons are Go packages under `plugins/` that implement `pkg/sdk` interfaces and register themselves in a registry during start-up (`registry.MustRegisterProvider(...)`). There is no `plugin.Open`, no runtime code loading and no scripting language. This keeps them type-safe, testable and simple to ship as a single binary, and it means no dynamic code from untrusted sources. Out-of-process plugins (gRPC) can be added later behind the same interfaces if third parties need them.
2. **Core does not import plugins.** `internal/engine`, `internal/app` and `internal/domain` depend only on `pkg/sdk` interfaces (dependency inversion, prompt §99). Only `cmd/*` wires concrete plugins into the registry. A linter rule enforces this (`depguard`).
3. **Declarative first.** Most add-ons are pure data (`addon.yaml` + Helm values template) handled by one generic implementation. Go code is written only for behaviour that data cannot express, such as pre-install CRDs or computing values from cluster facts.
4. **Idempotent "Ensure" semantics** for anything that changes the world. Methods are safe to call again with the same input.
5. **Context-first, typed results, typed errors** (`pkg/sdk/errs`). No `panic` across plugin boundaries; the registry wraps plugin calls with recover-and-classify.
6. **Capabilities over type switches.** Plugins declare capabilities (for example `CreateInstances`, `LoadBalancers`, `Pricing`, `Regions`). The engine and UI adapt to them instead of checking `provider == "hetzner"`.

## 2. Registry

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

At start-up the registry is **validated against the version catalog**. Every catalog entry must have an implementation; capability graphs must be acyclic; config schemas must compile. A broken plugin set fails fast at boot, not during a customer deployment.

## 3. Infrastructure providers

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

Implementations and status:

| Provider | Phase | Notes |
|---|---|---|
| `simulated` | 1 | Fake hosts with configurable latencies and fault injection. Development and test only (`TestOnly`). Lets the full UI → API → engine → SSE flow run without infrastructure; it never pretends to be real in production builds (prompt §115) |
| `baremetal` | 3 | Declared hosts reached over SSH. `Discover` collects facts. `EnsureInstance` verifies reachability, host key, OS support and resources. No instance creation. Also covers "existing infrastructure" (prompt §5): user-provided CP/worker nodes, external LB, storage endpoints |
| `hetzner` | 7 | Hetzner Cloud API (servers, networks, firewalls, load balancers, placement groups), pricing API, `hcloud-cloud-controller-manager` + CSI add-ons |
| `aws`, `gcp`, `azure` | 7 | Native SDK implementations, or a Cluster-API-backed implementation (decision per ADR-0013 when Phase 7 starts) |
| `digitalocean`, `ovh`, `vultr`, `linode` | later | Same interface. Community contributions welcome |

## 4. Kubernetes distributions

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

Implementations: `kubeadm` (Phase 3–4, full: single CP, stacked-etcd HA, join, reset, kubeconfig, health, upgrade foundation in Phase 6), `k3s` and `rke2` (Phase 7), `talos` and `k0s` as candidates later. Distribution-specific logic never leaks into the engine (prompt §7).

## 5. Node access and OS abstraction

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

The SSH adapter (`internal/adapters/ssh`) implements `NodeHandle`. It supports private key, password, SSH agent, jump hosts (bastion chains) and an HTTP/SOCKS proxy. Host-key verification is mandatory: a known host or a user-confirmed fingerprint (TOFU confirmation stored in `ssh_known_hosts`). The adapter has connection pooling per node, keep-alives, and command and session timeouts. Private keys are decrypted just in time in worker memory and never written to disk. The simulated provider has an in-memory `NodeHandle` with fault injection.

## 6. Add-ons

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

The **generic chart add-on** (`plugins/addons/generic`) implements `Addon` from a file:

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

Values templates render to a Go map that is validated against the chart's `values.schema.json` when one exists. User overrides (Advanced Mode) are merged as structured data, never as text.

## 7. Helm service

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

This is one implementation over the embedded Helm SDK (Helm SDK major version: see ADR-0010). Charts are pulled only from catalog-pinned references (digest-verified; air-gap mirrors supported), and are cached on the worker in a content-addressed cache. Helm is never called through a shell (prompt §51).

## 8. Health probes

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

## 9. Version catalog, compatibility, recommendations (ports)

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

Details: [Auto Mode, catalog and compatibility](auto-mode-and-catalog.md).

## 10. Secrets, keys, integrations

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

Outbound HTTP from integrations, webhooks, Helm repositories and Git always goes through `internal/adapters/httpx` (SSRF guard, timeouts, size limits). See the [security model](../security/SECURITY_MODEL.md).

## 11. Application-layer ports (internal)

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

## 12. Enterprise extension points

Enterprise code in `ee/` (commercial license, build tag `ee`) plugs into the same core through interfaces, never through `if enterprise` branches:

| Extension point | Core default (Community) | ee implementation |
|---|---|---|
| `IdentityProvider` (login methods) | local password | OIDC, SAML 2.0 (Okta, Entra ID, Google Workspace, Keycloak, Auth0, generic) |
| `Authorizer` | RBAC with built-in roles | Custom roles + ABAC conditions (policy engine) |
| `PolicyEvaluator` (pre-deploy checks) | built-in guardrails (safety warnings) | policy as code with scopes, inheritance, BLOCK/WARN/INFO; OPA/Rego or Cedar adapters |
| `ApprovalGate` (before executing an operation) | always "approved" | change requests, approval policies, maintenance windows |
| `AuditSink` | PostgreSQL audit table + JSON/CSV export | SIEM webhook, syslog, S3, Splunk/Datadog/Elastic/Sentinel |
| `KeyProvider` | local KEK | OpenBao/Vault Transit, AWS/GCP/Azure KMS, BYOK |
| `FleetController` | single-cluster operations | cluster groups, batched/canary fleet rollouts |
| `EntitlementService` | Community entitlements | verifies Ed25519-signed license file, exposes features and limits |

The Community edition stays fully functional for provisioning and lifecycle. Enterprise adds governance, scale and integrations, and never removes core capability (see ADR-0002).
