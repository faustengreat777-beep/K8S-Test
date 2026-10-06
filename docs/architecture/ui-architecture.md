# UI architecture

> Status: **Proposed** (Phase 0). Language: English · [Русский](ui-architecture.ru.md)
>
> Related: [ARCHITECTURE](../ARCHITECTURE.md) · [API design](api-design.md) · [Auto Mode, catalog and compatibility](auto-mode-and-catalog.md) · ADR-0012 (frontend stack)

The web UI is **one client of the public API** (prompt §57). It holds no business rules that the server doesn't also enforce, and no critical state that exists only in the browser (prompt §89). Its job is to make a complex system feel simple. It should be "simple by default, powerful when needed" (prompt §119) and look like a modern product, not an old admin panel (prompt §44).

## 1. Stack

| Concern | Choice | Notes |
|---|---|---|
| Language / framework | TypeScript (strict) + React | Function components, hooks, no class components |
| Build | Vite | Dev server proxies `/api` to the Go server; production build is embedded into the Go binary with `go:embed` |
| Styling | Tailwind CSS v4 + CSS variables as design tokens | Tokens defined once for light and dark themes |
| Components | shadcn/ui (copied into `web/src/components/ui`, owned by us) on accessible primitives | Lets us restyle and extend without a vendor UI kit |
| Routing | TanStack Router (file-based, type-safe search params) | Search params hold filters/pagination so views are shareable and restorable |
| Server state | TanStack Query | Single cache for API data; SSE events update the cache; optimistic UI only for cheap, reversible actions |
| Forms | React Hook Form + Zod | Zod schemas are **generated** from the OpenAPI/JSON Schema where possible, so client validation matches the server's |
| API client | Generated from OpenAPI (`openapi-typescript` types + small fetch wrapper) | No handwritten request types |
| YAML / code | Monaco Editor + `monaco-yaml` (JSON Schema validation, completion, hover docs) + Monaco diff editor | Loaded lazily (large bundle), only on routes that need it |
| Terminal-style logs | xterm.js for raw command output; virtualized structured log table (TanStack Virtual) for events | Logs can reach 100k lines per operation |
| Command palette | `cmdk` | Ctrl/Cmd + K |
| i18n | i18next + ICU message format | en and ru from day one; Russian plural rules handled by ICU |
| Charts | Lightweight SVG charts (utilization, history) | Respect the design tokens; no heavy BI library |
| Tests | Vitest + Testing Library + MSW; Playwright E2E; axe-core accessibility checks | See [testing strategy](testing-strategy.md) |

Exact versions are in the [technology stack](technology-stack.md) document.

## 2. Source layout

```
web/
├── index.html
├── src/
│   ├── app/                    # router, providers (QueryClient, i18n, theme), layout shell, error boundaries
│   ├── routes/                 # route files (TanStack Router file-based routing)
│   ├── features/
│   │   ├── auth/               # login, session expiry handling
│   │   ├── clusters/           # list, dashboard, settings, YAML view, spec history/diff
│   │   ├── create/             # Auto / Simple / Advanced creation flows, review, deploy
│   │   ├── operations/         # live progress, task tree, logs, failure report, actions
│   │   ├── nodes/              # node list/detail, cordon/drain/replace
│   │   ├── addons/             # catalog browser, install/configure/remove
│   │   ├── templates/          # templates and versions
│   │   ├── credentials/        # vault (write-only secret forms)
│   │   ├── providers/          # provider accounts, inventory
│   │   ├── audit/              # audit log search/export
│   │   ├── settings/           # org, members, roles, API keys, webhooks, notifications
│   │   └── search/             # global search, command palette
│   ├── components/
│   │   ├── ui/                 # design-system primitives (shadcn-based)
│   │   └── domain/             # reusable domain widgets: StatusBadge, HealthPill, TaskTree, LogViewer, SpecDiff, PlanSummary…
│   ├── lib/
│   │   ├── api/                # generated client + query keys + hooks
│   │   ├── sse/                # EventSource wrapper with Last-Event-ID resume and backoff
│   │   ├── schema/             # JSON Schema → form helpers, Zod generation glue
│   │   └── format/             # dates, durations, bytes, percentages (locale-aware)
│   ├── i18n/
│   │   ├── en/*.json
│   │   └── ru/*.json
│   └── styles/                 # tokens.css (light/dark), tailwind entry
└── tests/                      # Playwright E2E
```

Rules: features don't import each other's internals. Shared code goes to `components/domain` or `lib`. Each feature owns its routes, components, hooks and i18n namespace.

## 3. Information architecture

```
Top bar:  org/project switcher · global search · command palette · notifications · help · user menu (theme, language)
Sidebar:  Overview · Clusters · Operations · Templates · Add-on catalog · Credentials · Providers · Audit · Settings
          (enterprise later: Fleet · Policies · Change requests · Compliance · Incidents · Cost)
```

Main screens:

| Screen | Content |
|---|---|
| **Overview** (multi-cluster, prompt §37) | Cluster cards/table with phase and health (Production HEALTHY, Staging HEALTHY, Development WARNING…), running operations, recent failures, certificate expiry warnings |
| **Cluster dashboard** (prompt §31) | Status, Kubernetes version, nodes/pods/namespaces counts, CPU/memory/disk utilization (when metrics are available), per-subsystem health (control plane, networking, storage, gateway, monitoring, backup), endpoints (API, Grafana), kubeconfig download, quick actions |
| **Nodes** (prompt §30) | Grouped by role: control plane / workers / pools; per node CPU, RAM, disk, IP, OS, kubelet version, status, conditions, pods, utilization; actions cordon, drain, restart, remove, replace |
| **Operations** | History with filters; live operation view (section 6) |
| **Add-ons** | Installed add-ons with health and version; catalog cards with description, pros/cons, compatibility, use cases (prompt §17); configure via schema-driven form or YAML |
| **Settings → Configuration** | Advanced tree (prompt §4): General, Kubernetes, Control Plane, Workers, Networking, DNS, Container Runtime, Storage, Gateway/Ingress, Load Balancer, Certificates, Security, Observability, Logging, Backup, GitOps, Autoscaling, GPU, Registry, Add-ons, Scheduling, Policies, Advanced — with **YAML** tab and **History/Diff** tab |
| **Upgrade** | Upgrade advisor: current vs target, compatibility, preflight, plan, potential downtime, warnings → confirm |

## 4. Cluster creation: three modes, one model

All modes produce the same **ClusterSpec** and end in the same **Review → Plan → Deploy** step. Switching mode keeps the data. The user can start in Auto, open "Customize" (Simple/Advanced) and then edit YAML, and nothing is lost.

### 4.1 Auto Mode (default entry point, prompt §127, §234)

1. A short form: **Name, Infrastructure (provider account or "add hosts"), Environment, Size, Availability**. Optional "More options": workload type, budget, compliance, region.
2. "Analyzing infrastructure…" → checks stream in (provider reachable, region capacity, Kubernetes compatibility).
3. "Generating cluster plan…" → the Recommendation card: Kubernetes, control plane, workers, CNI, Gateway, LB, storage, monitoring, logging, TLS, backup, security profile, estimated cost (or "Cost estimation unavailable"), estimated time.
4. Each line has **[Why?]**: a popover with the reasons from the decision engine ("Selected because: production environment, HA requested, network policies enabled…"). It also has **[Change]**: pick an alternative, and the engine re-runs and highlights dependent changes (prompt §131).
5. Safety warnings appear inline with explicit choices ("You selected 1 control-plane node for a production cluster… [Continue anyway] [Enable HA]", prompt §132).
6. "Validating policies…" → **Deploy**.

### 4.2 Simple Mode (prompt §3)

Create Cluster → Choose infrastructure → Choose preset (Development, Staging, Production, High Availability, Minimal, Edge, GPU, AI/ML, Custom) → Review (adjust a few parameters) → Deploy.

### 4.3 Advanced Mode wizard (prompt §46)

Steps: 1 Basics · 2 Infrastructure · 3 Kubernetes · 4 Nodes · 5 Networking · 6 Load Balancer · 7 Storage · 8 Gateway (Ingress) · 9 Security · 10 Observability · 11 Backup · 12 GitOps · 13 Add-ons · 14 Review · 15 Deploy.

The wizard:
- **saves a draft on the server** after every step (`PUT /clusters/{id}/spec` on a DRAFT cluster). Closing the tab loses nothing, and drafts are listed under Clusters.
- **validates each step** with client-side Zod (instant), then server `POST /validate` (compatibility and policies, debounced). Errors block, warnings are shown but don't block.
- **allows going back** freely. A stepper shows completion and problems per step.
- has **smart defaults** from the selected preset or the recommendation.
- **hides or disables incompatible options** with an explanation. For example, with k3s and Flannel the "Network policies" toggle shows "requires a CNI with NetworkPolicy support".
- shows **dependency-aware fields** (prompt §47):
  - CNI = Cilium → Hubble, kube-proxy replacement, encryption (WireGuard/IPsec), Gateway via Cilium
  - Storage = Longhorn → replica count, default StorageClass, backup target
  - Gateway = Envoy Gateway / NGINX Gateway Fabric → service type (LoadBalancer/NodePort), TLS (issuer), listeners
  - Load balancer = MetalLB → address pools, L2/BGP mode, BGP peers
  These come from add-on JSON Schemas (`if/then/else`, `dependentSchemas`) rendered by a schema-to-form layer, with hand-written components for the most important steps.

### 4.4 YAML mode (prompt §34)

- A split view: form ↔ YAML (Monaco). Edits in either side update the other. YAML is parsed and validated against the ClusterSpec JSON Schema from `/schemas/cluster-spec`.
- Unknown fields, type errors and enum violations are underlined with messages. Completion and hover docs come from the schema.
- Before saving, a **diff** is shown (current revision vs new, prompt §97). Formatting runs on save.

### 4.5 Review and Deploy (prompt §35, §65)

- Plan summary (`POST /clusters/{id}/plan`): `+ 3 control plane nodes`, `+ Kubernetes 1.36.x`, `+ Cilium`, … The estimated time and cost come from the plan.
- Checklist: ✓ Configuration valid · ✓ Infrastructure reachable · ✓ Credentials valid · ✓ Compatibility verified · ✓ Required resources available. Each item links to details.
- Irreversible or risky steps need explicit acknowledgement.
- **[Deploy Cluster]** sends `POST /clusters/{id}/deploy` with `Idempotency-Key` and `planHash`, then navigates to the live operation view.

## 5. State management rules

- **Backend is the source of truth.** Server data lives only in the TanStack Query cache. Query keys mirror API resources (`['cluster', id]`, `['operation', id, 'tasks']`).
- **URL holds view state**: filters, sorting, pagination, selected tab, selected task. Reloading or sharing a link restores the view.
- **Local component state** only for ephemeral UI: open dialogs, unsaved input before debounce.
- **No secrets in the browser state**: credential forms submit and immediately clear fields. The API never returns secrets.
- **Resilience**: on reconnect or focus, queries refetch. SSE resumes from the last event id. The app shows a non-blocking "connection lost — reconnecting" banner.
- **Recovery after closing the browser** (prompt §66): opening a cluster with an active operation shows "Deployment in progress" and attaches to the stream. History is replayed from the server.

## 6. Live operation view (prompt §15)

```
Deploying "production"                                    ⏱ 08:41 elapsed · ~6 min remaining
[✓] Validating infrastructure
[✓] Preparing nodes (6/6)
[✓] Installing containerd
[✓] Installing Kubernetes
[✓] Bootstrapping control plane
[✓] Installing Cilium
[✓] Installing CoreDNS
[→] Installing Gateway (Cilium)            ▸ 3 tasks
[ ] Installing cert-manager
[ ] Installing monitoring
[ ] Running health checks
───────────────────────────────────────────────────────────────────────────────
Logs  [All levels ▾] [All tasks ▾] [All nodes ▾] [Search…]          [Follow ✓] [Download]
12:31:02 INFO  cp-1  Connecting to node 10.0.0.11
12:31:03 INFO  cp-1  Installing containerd
12:31:08 INFO  cp-1  containerd installed
```

- The **task tree** groups DAG tasks into user-friendly stages (stage mapping comes from task metadata). Expanding a stage shows per-node tasks with attempts and durations.
- The **log panel** is virtualized, with level/task/node filters, full-text search (server-side for history, client-side for the live tail), follow mode, and download (JSON or text).
- **Failure state**: the failure report card (what / why / where / how to fix) with action buttons Retry, Resume, Roll back, View logs, Abort. The relevant logs are pre-filtered to the failed task.
- **Success state**: "🎉 Cluster Ready" with Kubernetes API URL, Grafana URL and credentials hint (secret retrievable once, or SSO), **Download kubeconfig**, **Open cluster**.

## 7. Design system

Components (prompt §45): buttons, inputs, selects, comboboxes, dialogs, drawers, tables (sortable, selectable, virtualized), cards, badges (phase/health), alerts, progress (linear, stepper, task tree), wizard/stepper, log viewer, terminal, code editor, YAML editor, diff viewer, empty states, skeleton loaders, toasts, command palette.

- **Tokens** (`styles/tokens.css`): color roles (`--bg`, `--surface`, `--text`, `--muted`, `--primary`, `--success`, `--warning`, `--danger`, `--info`, `--border`, `--focus-ring`), spacing, radius, typography, elevation, motion. Light and dark sets are designed separately and tested for contrast. Dark mode is a full theme, not inverted colors (prompt §95). Theme follows the OS by default; the user can override it.
- **Status semantics** are consistent everywhere: phase badges (READY, UPDATING, PAUSED, FAILED…) and health pills (HEALTHY/WARNING/CRITICAL/UNKNOWN) always pair an icon and a label with color, never color alone.
- **Density**: comfortable by default, compact for tables of nodes, logs and audit.
- **Copy**: plain language first, with technical details on demand ("Show technical details").

## 8. Accessibility (prompt §90)

Target **WCAG 2.2 AA**:
- Full keyboard navigation (logical tab order, visible focus ring, no keyboard traps, shortcuts listed in the help dialog and the palette).
- Proper labels and descriptions for every form control. Errors are linked with `aria-describedby`.
- Focus management: dialogs trap focus and restore it on close. Route changes move focus to the page heading.
- Screen readers: live regions announce operation progress milestones and failures, without reading every log line.
- Sufficient contrast in both themes. Respect `prefers-reduced-motion`.
- Automated axe checks in component tests and Playwright, plus a manual screen-reader pass before each major release.

## 9. Responsiveness (prompt §91)

Desktop is first-class, tablet (≥ 768 px) is fully supported. Below that, read-only views (status, operations, notifications) work, but complex editing such as the wizard and YAML shows a "best on larger screens" hint while staying usable.

## 10. Internationalization (prompt §92)

- i18next with namespaces per feature. Message format is ICU (`{count, plural, one {# node} few {# узла} many {# узлов} other {# узла}}`).
- Server error codes map to localized messages. The API also localizes `title`/`detail` by `Accept-Language`, and the UI prefers its own catalog to stay consistent.
- Dates, numbers, durations and bytes are formatted with `Intl` in the user's locale. All times are shown in local time with UTC on hover.
- English is the source language. Russian is maintained in the same PR as any copy change (CI fails on missing keys). Adding a language means adding a folder.

## 11. Global search and command palette (prompt §93–94)

- **Search** (`/` or the top bar): clusters, nodes, operations, add-ons, logs, users, via `GET /search`. Results are grouped by type and filtered by the user's permissions on the server.
- **Command palette** (Ctrl/Cmd + K): actions such as Create cluster, Open production, View deployments, Add node, Install add-on, Upgrade Kubernetes and Download kubeconfig. Commands are registered by features with permission and context checks; disabled commands explain why.

## 12. Performance budget

- Initial JS for the shell ≤ 250 KB gzip. Monaco, xterm.js and charts are lazy chunks.
- Time to interactive on a mid-range laptop ≤ 2 s for the dashboard (cached API).
- Lists are paginated server-side and virtualized client-side when long.
- SSE updates are batched into animation frames for smooth rendering during noisy phases.

## 13. Security in the UI

- Same-origin only: the UI is served by the API binary. Cookies are `HttpOnly`, and the CSRF token goes in a header.
- A strict **Content-Security-Policy** without inline scripts. Monaco workers are loaded from the same origin.
- No secrets rendered or stored. Credential inputs use `autocomplete="off"` where appropriate.
- External links (Grafana URL, docs) use `rel="noopener noreferrer"`.
- User-provided strings are always rendered as text, never as HTML. Markdown (for example add-on docs from the catalog) is rendered with a sanitizer.
