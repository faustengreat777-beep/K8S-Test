# ADR-0026: Bilingual documentation (English + Russian) side by side

- Status: Accepted
- Date: 2026-10-06
- Language: English · [Русский](0026-bilingual-documentation.ru.md)

## Context

The owner works in Russian and asked for documentation in both languages (Phase 0 answer: «Можешь делать на 2х языках сразу»). An open-source Community edition needs English documentation to reach contributors. The UI must support en and ru from day one (prompt §92).

## Decision

- Every document exists as `X.md` (English) and `X.ru.md` (Russian) in the same directory. Each starts with a language switcher line.
- **English is the source language** for technical precision and for contributors. Russian is a full translation, not a summary. Both are updated **in the same PR**, and a CI check fails when a document has no counterpart or when headings, code blocks or tables differ structurally.
- Code, identifiers, API, commit messages, code comments and log messages are English only. User-facing UI strings, error titles and remediation texts are localized (en, ru) through i18n catalogs.
- A shared glossary keeps terminology consistent. Established English terms stay in English, e.g. control plane, Gateway API, Helm chart.
- Russian docs link to Russian counterparts. Cross-document anchor links are dropped in Russian, because heading slugs differ by language.

## Alternatives considered

- **English only**: excludes the owner and the primary target audience.
- **Russian only**: limits open-source reach.
- **Separate `docs/en` and `docs/ru` trees**: drift is harder to notice, and relative links get longer.

## Consequences

- Positive: the owner reads everything natively, and contributors get English.
- Negative: double maintenance effort and a risk of drift. This is mitigated by same-PR updates and CI structure checks.
