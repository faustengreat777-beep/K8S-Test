# ADR-0018: Rule-based, deterministic and explainable decision engine for Auto Mode

- Status: Proposed
- Date: 2026-10-06
- Language: English · [Русский](0018-explainable-decision-engine.ru.md)

## Context

Auto Mode must turn a few answers (environment, infrastructure, size, availability, workload, budget, compliance…) into a complete cluster plan through a separate decision engine, not hard-coded `if/else` (prompt §127–129). It must be transparent ("Why this configuration?", prompt §130), overridable with recomputation of dependencies (prompt §131), and must never silently pick dangerous settings (prompt §132).

## Decision

- `internal/recommend` implements a **pure function** `Recommend(input, catalog, inventory, policies) → Recommendation`. It has no I/O and gives the same output for the same input.
- An **ordered pipeline of small rules** (constraints → topology → Kubernetes → networking → LB → storage → certificates → observability → security → backup → resources → add-on versions → safety review → estimates). Each rule emits `Decision{path, value, reasons[], alternatives[], risk[], pinned, ruleId}`.
- **Explanations** are first-class data (i18n keys + params), rendered by the UI and CLI.
- **Overrides pin** a path and re-run the pipeline. Rules must respect pinned paths, and dependent decisions recompute.
- **Safety review** produces explicit warnings with choices for data-loss, downtime, security-degradation, cost and compatibility risks. In production, acknowledgements are recorded.
- Every recommendation is validated by the compatibility engine (zero BLOCK findings is a property-tested invariant).
- Presets are inputs and defaults for the same engine, not separate code paths.
- **Golden-file tests** cover the input matrix.

## Alternatives considered

| Option | Why not |
|---|---|
| Hard-coded preset templates only | No adaptation to provider capabilities, inventory or budget; explanations can't be produced |
| General rules engine / Rego for recommendations | Powerful, but less readable for contributors. Explanations and alternatives are awkward to express. Rego stays a candidate for the *policy* engine (ee), which evaluates rather than generates |
| ML/LLM-based recommender | Not deterministic or auditable. Explanations can't be verified. Unacceptable for infrastructure decisions |
| Constraint solver (SAT/SMT) | Overkill for the current decision space, and its explanations are hard to make human-friendly. Reconsider if the add-on space grows combinatorially |

## Consequences

- Positive: predictable, testable, explainable recommendations. Easy to extend with a new rule per concern.
- Negative: rule ordering and interactions need care. This is mitigated by golden files and property tests (no BLOCK findings, pins respected).

## References

- [Auto Mode, catalog and compatibility §3](../auto-mode-and-catalog.md)
