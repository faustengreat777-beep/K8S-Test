# ADR-0012: Frontend stack

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0012-frontend-stack.ru.md)

## Context

The spec prefers React, TypeScript, Vite, Tailwind, shadcn/ui, TanStack Query, React Hook Form and Zod (prompt §44). It requires a modern, clean, enterprise-grade UI (not an old admin panel), a design system with light/dark themes (prompt §45, §95), a Monaco YAML editor with validation, autocomplete, schema validation, formatting and diff (prompt §96), a wizard (prompt §46), accessibility (prompt §90), desktop/tablet support (prompt §91), i18n with English and Russian (prompt §92), a command palette (prompt §94) and global search (prompt §93). The frontend must be a stateless client of the API (prompt §89).

## Decision

- **Runtime model**: a client-only SPA built with **Vite 8** (Rolldown/Oxc) and **React 19.3**, written in **TypeScript 7** (strict). It is served same-origin by the Go `api` role from `embed.FS`, with an `index.html` history fallback. No SSR and no React Server Components. Node.js is needed only at build time (Node 24 LTS, 26 LTS from late October 2026).
- **Design system**: **Tailwind CSS 4.3** with CSS-variable design tokens for full light and dark themes, and **shadcn/ui (CLI 4) on Base UI primitives**, vendored into `web/src/components/ui`, so we own the code.
- **Routing and data**: **TanStack Router** (type-safe routes and search params validated with Zod 4) + **TanStack Query 5** (server-state cache updated by SSE). **TanStack Table v9** for resource grids, **TanStack Virtual** for long log views.
- **Forms**: **React Hook Form 7 + Zod 4**. Zod schemas are generated from the committed OpenAPI/JSON Schema where possible and shared with route validation.
- **API client**: types generated from the committed OpenAPI 3.1 document (`openapi-typescript`) with `openapi-fetch`.
- **Editor**: **Monaco Editor + monaco-yaml**, because the spec requires Monaco (prompt §96) and its diff editor fits spec diffs. Mitigations for known 2026 issues:
  - selective `monaco-editor/editor` imports;
  - local bundling via `loader.config({ monaco })` with no CDN or AMD loader (air-gap);
  - a pinned monaco-editor version compatible with monaco-yaml under Vite 8, or the documented worker alias;
  - `enableSchemaRequest: false` with locally bundled schemas;
  - lazy-loaded editor routes.
  The editor sits behind a `CodeEditor` component interface. If Monaco's integration or accessibility becomes a blocker, **CodeMirror 6** (`@codemirror/lang-yaml`, `codemirror-json-schema`, `@codemirror/merge`) replaces it without touching features.
- **i18n**: **Lingui 6**: ICU MessageFormat with CLDR plural categories (Russian one/few/many/other), compile-time catalogs, PO files for translators/TMS. The API returns stable error codes and parameters; the UI renders localized text.
- **Real-time**: native `EventSource` with the same-origin session cookie and one multiplexed SSE stream per tab. A fetch-based parser (`eventsource-parser`) is used only where bearer tokens are needed.
- **Quality**: **Oxlint** (including type-aware rules) + Prettier; **Vitest 5** (+ Browser Mode), Testing Library, **MSW 3**; **Playwright** with axe accessibility checks in CI.
- Exact versions: [Technology stack §2](../technology-stack.md).

## Alternatives considered

| Option | Why not |
|---|---|
| Next.js / React Router framework mode with SSR | The UI is an authenticated SPA served by the Go binary. SSR adds a Node.js runtime to the platform, which is bad for single-binary and air-gapped installs, and SEO is irrelevant |
| Vue / Svelte / Angular | Viable, but the spec prefers React, and the React ecosystem (shadcn/ui, TanStack, Monaco wrappers) fits best |
| Prebuilt admin/UI kits (MUI, Ant Design) | Strong components but a recognizable "admin panel" look, heavier bundles and harder theming. shadcn/ui gives us owned, restylable components on accessible primitives |
| CodeMirror 6 instead of Monaco (research recommendation) | About 3× smaller, no web workers, better touch and screen-reader support. Kept as the drop-in fallback behind the `CodeEditor` interface. Monaco stays primary because the spec names it explicitly and its YAML/diff tooling is richer |
| i18next + react-i18next | Mature and popular. ICU needs the `i18next-icu` plugin, and its JSON catalogs are less translator-friendly than Lingui's PO files |
| ESLint (+ typescript-eslint) | ESLint 9 is EOL; ESLint 10's plugin ecosystem (jsx-a11y, react) lags; typescript-eslint doesn't yet support TypeScript 7. Oxlint covers React, a11y and type-aware rules natively |

## Consequences

- Positive: matches the spec and a mainstream, well-supported ecosystem; type-safe end to end (generated API types → typed router and query hooks → Zod forms); the UI can be embedded in the server binary.
- Negative: a fast-moving JavaScript ecosystem. In the last four months alone: TypeScript 7, Vite 8, Vitest 5, MSW 3, TanStack Table 9, pnpm 12. That needs Renovate, a pinned lockfile and a pinned Node version. Monaco's size (≈1.3 MB gzip with workers) and integration fragility need lazy loading and the CodeMirror fallback. Tailwind v4 raises the browser floor (Firefox 128+, Safari 16.4+), which must be documented for enterprise customers.

## References

- [UI architecture](../ui-architecture.md), [Technology stack §2](../technology-stack.md)
