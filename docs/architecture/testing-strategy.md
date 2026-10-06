# Testing strategy

> Status: **Proposed** (Phase 0). Language: English · [Русский](testing-strategy.ru.md)
>
> Related: [ARCHITECTURE](../ARCHITECTURE.md) · [Provisioning engine](provisioning-engine.md) · [Threat model](../security/THREAT_MODEL.md) · ADR-0016 (testing without own hardware)

"Don't write the application without tests" (prompt §61). A feature is done only when it has tests at the right level (Definition of Done, prompt §125). This document defines those levels, the test environments, the mandatory E2E scenario and the failure-testing matrix. It also covers how we test infrastructure provisioning **without owning any hardware**.

## 1. Test pyramid

| Level | Scope | Tools | Runs | Target |
|---|---|---|---|---|
| Static | Formatting, lint, types, architecture rules (import boundaries), forbidden patterns (`sh -c`), secret scan | gofumpt, golangci-lint v2 (incl. gosec, depguard, custom analyzers), `tsc --noEmit` (TypeScript 7), Oxlint, Prettier, gitleaks | every commit (pre-commit) + CI | 0 findings |
| Unit (Go) | Domain state machines, planner/DAG, scheduler, retry/backoff, compatibility engine, decision engine, validators, redactor, crypto envelope, error catalog | `go test`, table-driven tests, golden files, Go native fuzzing (`go test -fuzz`) for parsers/validators/quoting | every commit | ≥ 80 % lines in `internal/domain`, `engine`, `catalog`, `recommend`, `secrets` |
| Unit (web) | Components, hooks, form logic, schema → form rendering | Vitest + Testing Library + axe | every commit | Key components covered; 0 axe violations |
| Integration (Go) | Repositories with real PostgreSQL (RLS on), River jobs, transactional outbox, API handlers end-to-end through HTTP with real DB, OpenAPI contract validation of every response, SSE streaming and resume | `testcontainers-go` (PostgreSQL 18), `httptest`, kin-openapi validator | every PR | All endpoints have happy-path + authz tests |
| Integration (web) | Feature flows against mocked API (MSW) generated from OpenAPI examples | Vitest + MSW | every PR | Wizard, operation view, credentials, errors |
| Provider / distribution | SSH adapter, bootstrap layer, kubeadm driver against **container nodes** (section 3.2); provider contract tests against simulated + recorded HTTP fixtures (cloud) | Go tests + Docker | every PR (subset), nightly (full) | Each provider/distribution passes the shared contract suite |
| Add-on | Install/upgrade/uninstall/health per add-on and catalog version on **kind/k3d** | `make test-cluster` + Go tests using Helm service | nightly + on catalog changes | Every catalog entry tested before release |
| Compatibility | Catalog consistency (no cycles, every entry has implementation, version ranges valid), matrix tests (K8s minor × CNI × OS) | Go tests over catalog data | every PR touching catalog | 100 % catalog entries validated |
| E2E (system) | Full user flows through UI and API: simulated provider (fast), container nodes (real kubeadm), real VMs (nightly/manual) | Playwright + Go E2E harness + CLI | see section 3 | Mandatory scenario green |
| Security | Authz matrix (endpoint × role × tenant), tenant-escape, SSRF, injection fuzzing, secret-leak canaries, dependency/image scanning | Go tests, fuzzing, govulncheck, OSV-Scanner (dependencies), Grype (images; pinned Trivy as a second opinion in nightly runs) | every PR (fast), nightly (full) | 0 high/critical unaccepted |
| Failure / chaos | Fault injection in SSH, provider, Helm, Kubernetes API, DB; worker kill during operations | Fault-injecting adapters + test harness | every PR (simulated), nightly (container nodes) | Every scenario in section 5 produces its expected outcome: correct report, plus resume, rollback or rejection as specified |

## 2. Principles

- **Real dependencies over mocks** at integration level: real PostgreSQL, real River, real Helm SDK against a real cluster (kind/k3d), real SSH server in containers. Mocks are only for external SaaS APIs (recorded fixtures) and for fault injection.
- **Determinism:** time is injected (`Clock`), randomness seeded, plans are golden-file tested, and parallel DAG execution is tested with a deterministic scheduler mode.
- **No fake production code:** the simulated provider and fault injectors live in clearly separated packages (`plugins/providers/simulated`, `test/…`), carry `TestOnly` metadata, and are refused in production profiles (prompt §115).
- **Tests are documentation:** each failure scenario test asserts the user-facing error code, remediation and available actions, not just "returns error".
- **Flaky tests are bugs:** quarantining needs an issue and an owner, and is time-boxed.

## 3. Test environments — testing without own hardware

The owner has no physical or virtual machines. We use three tiers.

### 3.1 Tier 1 — Simulated infrastructure (every PR, seconds to minutes)

- The `simulated` provider creates fake hosts with configurable facts (OS, CPU, RAM, disk), latencies and **fault scripts** ("fail `runtime.install` on node w-2 on attempt 1 with transient error", "drop SSH connection after 3 s", "disk full").
- The engine, API, SSE, UI and CLI run for real. Only the node side is simulated.
- This tier proves the orchestration logic: DAG ordering, parallelism, retries, pause/resume, rollback, cancellation, crash recovery (killing the worker process mid-operation), idempotency keys, concurrency guards.
- Playwright E2E runs here: Auto Mode → plan → deploy → live progress → success; failure → report → retry → success.

### 3.2 Tier 2 — Container nodes (every PR subset, nightly full; minutes)

- **"Container nodes"** are privileged Docker containers that run **systemd + sshd** with a base image per supported OS (Ubuntu 24.04/26.04, Debian 13; Rocky/Alma later). They behave enough like VMs for kubeadm, as the `kind` project proves by running kubeadm inside containers.
- The platform reaches them **over real SSH** with generated throwaway keys and runs the **real bootstrap layer and kubeadm driver**: install containerd and Kubernetes packages (from a local package cache/mirror to keep CI fast and offline-capable), `kubeadm init/join` (single and 3-CP HA with kube-vip in a Docker network), Cilium, then add-ons. Then `kubectl get nodes` / `get pods -A` checks.
- Known limits, documented and handled with capability flags in tests:
  - kernel modules and many sysctls belong to the CI host's kernel. Tasks detect "already loaded / read-only in container" and treat it as satisfied in a test profile.
  - no real L2 for kube-vip ARP and MetalLB L2 across hosts. The Docker bridge network works for single-host tests.
  - no real disks for Longhorn/Ceph performance (functional tests only).
  - cgroup v2 nesting requires a cgroup-v2 CI host (GitHub-hosted Ubuntu runners are cgroup v2).
- This tier catches most real-world bootstrap bugs (packages, config files, kubeadm flags, join ordering, idempotency on re-run) without any cloud account.

### 3.3 Tier 3 — Real virtual machines (nightly or manual; tens of minutes)

- Option A (recommended once a budget exists): a **Hetzner Cloud project** used by CI through a token stored as a GitHub Actions secret, with VMs created and destroyed per run and spend capped. This validates real networking and kernels, the Hetzner Cloud load balancer for the API endpoint (and that L2/ARP VIPs such as MetalLB L2 and kube-vip ARP are correctly blocked on Hetzner Cloud's layer-3 networks), real disks for Longhorn, and the Phase 7 Hetzner provider itself.
- Option B: **nested VMs on CI runners** (QEMU/KVM, when the runner exposes `/dev/kvm`) using cloud images and cloud-init. This is slower but has no cloud bill, and suits OS-matrix smoke tests.
- Option C: the owner or contributors run `make e2e-real` against any SSH-reachable machines they have, using the same harness.
- Until Tier 3 runs, phase reports state clearly which scenarios were verified only on Tiers 1–2 (**no claims of "tested on bare metal" without evidence**).

### 3.4 Local developer environments

- `make dev`: docker compose with PostgreSQL, api, worker and the Vite dev server (simulated provider enabled in the dev profile).
- `make test-cluster`: creates a local **kind** cluster (default) or **k3d** (`TEST_CLUSTER=k3d`) for add-on and Helm-service tests (prompt §78).
- `make test-nodes`: starts N container nodes for Tier 2 tests locally.
- `make e2e`: runs Tier 1 E2E (UI + API) headless.

## 4. Mandatory E2E scenario (prompt §62)

```
Create cluster (UI / API / CLI)
  → Provision infrastructure (simulated | container nodes | real VMs)
  → Install Kubernetes (kubeadm)
  → Install CNI (Cilium)
  → Install Gateway API controller
  → Install storage (local-path; Longhorn on Tier 3)
  → Install monitoring (kube-prometheus-stack)
  → Health check
  → Cluster READY
  → Get kubeconfig (user-scoped)
  → kubectl get nodes            (all Ready, expected roles/versions)
  → kubectl get pods -A          (no CrashLoopBackOff; add-on pods Ready)
  → Deploy sample app behind a Gateway + HTTPRoute, curl it (Tier 2/3)
  → Delete cluster → nodes reset, records DESTROYED, audit complete
```

The harness also checks the audit log entries, operation events (no secrets — canary scan), webhook deliveries (to a local receiver), and the UI state after a browser reload mid-deployment (prompt §66).

## 5. Failure testing matrix (prompt §63)

Each scenario runs at least on Tier 1 (fault injection). Scenarios marked ★ also run on Tier 2 with real faults (stopping sshd, filling a tmpfs, iptables drops, killing containers).

| Scenario | Injection | Expected behaviour |
|---|---|---|
| SSH failure ★ | Stop sshd / drop connection mid-command | Transient retry with backoff; after exhaustion, PAUSED with `SSH_UNREACHABLE` (what/why/where/how to fix), Resume works after restoring sshd |
| Node unavailable ★ | Kill node container before/while joining | Preflight catches before start; mid-operation → task fails with `NODE_UNREACHABLE`; other nodes continue to safe point |
| Disk full ★ | Small tmpfs on `/var/lib/containerd` | Preflight warns on low disk; runtime install fails with `DISK_FULL` remediation (free space / resize) |
| Wrong credentials | Bad key / password / sudo needs password | `SSH_AUTH_FAILED` / `SUDO_PASSWORD_REQUIRED` (UserAction, no retries), credential link in remediation |
| Network unavailable ★ | iptables DROP between nodes / to package mirror | `PACKAGE_REPO_UNREACHABLE` or `K8S_JOIN_API_UNREACHABLE` with port/route checks |
| Helm failure | Chart render error / timeout / failed hook | Add-on task fails; with policy `rollback` → helm rollback; report shows chart, values diff and failing resource |
| Add-on failure | Add-on pods CrashLoop (bad values) | Health check fails → rollback of that add-on only (reversible); cluster stays usable |
| Kubernetes bootstrap failure ★ | `kubeadm init` fails (port in use, bad config) | Irreversible-step handling: PAUSED, recommendations, `ResetNode` offered explicitly |
| Partial deployment | Kill worker process at random points (100 iterations) | Lease expiry → another worker resumes; no task runs twice concurrently; final state identical to an uninterrupted run |
| Worker join failure ★ | Block 6443 from one worker | Only that node's tasks fail; cluster READY with WARNING if policy allows partial, or PAUSED; Retry for that node works |
| Control-plane join failure ★ | Break etcd peer port on cp-2 | Serialized CP joins stop; etcd member cleanup task documented; no quorum loss on cp-1 |
| DB unavailable | Pause PostgreSQL container | API returns 503 on readiness; workers stop starting tasks and keep leases until timeout; operations resume after DB returns |
| Duplicate submit | Same Idempotency-Key twice, concurrent | Exactly one operation |
| Concurrent operation | Start upgrade while apply runs | `409 OPERATION_IN_PROGRESS` |

Every failure test asserts: error code, localized title/remediation (en and ru), `location` (task/node), available actions (Retry/Resume/Rollback), no secrets in messages, correct cluster phase and audit event.

## 6. Security testing

- **Authorization matrix**: generated from the OpenAPI spec and the permission catalogue. Every endpoint × built-in role × (same tenant, other tenant, anonymous) asserts the expected status (200/202, 403, 404, 401).
- **Tenant escape**: per repository and per endpoint, using ids from another org, including SSE subscribe and resume, logs, audit, search and kubeconfig.
- **SSRF**: URLs to loopback, private ranges, metadata IPs, DNS names resolving to private IPs, redirects to private IPs, IPv6 tricks (`::ffff:127.0.0.1`), DNS rebinding simulation.
- **Injection**: fuzz validators (hostnames, labels, paths, versions) and the POSIX quoting function against a real `/bin/sh` in a sandbox container. The property: arbitrary input must arrive at the remote program as exactly one argument.
- **Secret leakage**: canary secrets (unique random strings) in credentials. After each E2E run, scan DB tables (except `secrets`), logs, events, API responses, webhook payloads and error reports for the canaries.
- **Dependency and image scanning**: govulncheck, OSV-Scanner or Grype on Go and npm deps, image scan of our images, SBOM diff per release.
- **Headers and session**: automated checks for CSP, HSTS, cookie flags, CSRF enforcement, session rotation and timeouts.

## 7. Frontend testing details

- **Component tests** for the design system and domain widgets (TaskTree, LogViewer, SpecDiff, StatusBadge, schema-driven forms), with axe checks in both themes.
- **Integration tests** with MSW: the wizard (validation, back navigation, dependent fields, draft persistence), Auto Mode (explanations, overrides, warnings), the operation view (SSE replay and live), credential forms (write-only behaviour) and error rendering from problem+json.
- **Playwright E2E** against the docker compose stack with the simulated provider. It covers the critical paths in Chromium (all PRs), Firefox and WebKit (nightly), keyboard-only navigation, and light/dark screenshots for visual review.
- **i18n checks**: no missing keys in `ru`, pseudo-locale run to catch hard-coded strings and truncation.

## 8. CI pipeline (prompt §80)

```
lint ─▶ unit ─▶ integration ─▶ build ─▶ security scan ─▶ e2e (simulated; container-nodes subset) ─▶ docker build ─▶ publish (tags only: signed images/binaries, SBOM, provenance)
                                                     nightly: full container-nodes matrix, add-on matrix on kind/k3d, browser matrix, fuzzing (time-boxed), Tier 3 when configured
```

- Required checks on `main`: lint, unit, integration, build, security scan, E2E simulated.
- Caching: Go build cache, pnpm store, package mirror for container nodes, image cache.
- Test reports (JUnit), coverage, and Playwright traces are uploaded as artifacts. Failures link to the logs.

## 9. Definition of Done — testing part

A change is done only if: unit tests cover the new logic; integration tests cover new endpoints/repositories (incl. authz and tenancy); failure paths have tests asserting user-facing errors; E2E is updated when a user flow changes; security tests are updated when the change touches a threat in the [threat model](../security/THREAT_MODEL.md); lint/type/format pass; `make dev` still works; and the phase report says honestly which tiers verified it.
