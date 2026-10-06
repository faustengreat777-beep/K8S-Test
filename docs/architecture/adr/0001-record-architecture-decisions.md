# ADR-0001: Record architecture decisions

- Status: Accepted
- Date: 2026-10-06
- Language: English · [Русский](0001-record-architecture-decisions.ru.md)

## Context

The product spec requires architecture decision records (prompt §101) and a decisions log (`docs/DECISIONS.md`, prompt §112). The platform will evolve over many phases. Contributors and the owner need to know why things are the way they are, which alternatives were rejected, and when a decision can be revisited.

## Decision

- We record every architecturally significant decision as an ADR in `docs/architecture/adr/NNNN-short-title.md`, in English, with a Russian translation `NNNN-short-title.ru.md` next to it.
- Format: a lightweight MADR variant with these sections: Status, Date, Context, Decision, Alternatives considered, Consequences, References.
- Statuses: `Proposed` → `Accepted` → (`Deprecated` | `Superseded by ADR-XXXX`). ADRs are immutable once accepted. A change of mind is a new ADR that supersedes the old one.
- `docs/DECISIONS.md` is the index: one line per ADR with title, status and a one-sentence summary.
- An ADR is required when a change affects module boundaries, public interfaces (`pkg/sdk`, `pkg/spec`, the API), persistence, security model, the dependency set, licensing, or the supported version matrix.
- Phase 0 ADRs are `Proposed` until the owner approves the architecture. Then they become `Accepted` in one commit.

## Alternatives considered

- **No ADRs, only design docs.** Rationale gets lost in large documents, and superseded thinking is hard to see.
- **Full MADR template.** Too heavy for the volume of decisions here. The lightweight variant keeps the key fields.

## Consequences

- Positive: decisions are traceable, reviewable and linkable from code reviews and docs.
- Negative: some writing overhead per decision, and two languages double it. This is accepted because the owner works in Russian.

## References

- Michael Nygard, "Documenting Architecture Decisions" (2011)
- MADR — Markdown Architectural Decision Records
