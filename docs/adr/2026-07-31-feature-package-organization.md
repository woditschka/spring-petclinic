# Feature-Package Organization Without a Service Layer

**Status:** Accepted

> **Provenance: derived from code, unconfirmed.** This ADR was reconstructed from the working tree during the bootstrap survey. The file date is the date the decision was *recorded*, not the date it was made — the actual decision date is unknown. Only the Decision section is evidenced by code.

## Context

**Not recorded, and not recoverable.** The code proves this decision was taken. It cannot show what problem it solved, what constraints applied, or when. Nothing is inferred here.

## Options Considered

**Not recorded, and not recoverable.** No alternative layering is evidenced anywhere in the repository. Any list written here would be invention.

This project was adopted as an existing upstream codebase. The reasoning predates this repository's documentation and was never written down here, so no interview can recover it. This section stays empty by decision (2026-07-31), not as pending work.

## Decision

Production code is organized by feature, not by technical layer. Each feature package (`owner`, `vet`) holds its own entities, repository interfaces, controllers, and validation together. A `model` package holds the mapped superclasses the feature packages share, and a `system` package holds cross-cutting web and infrastructure configuration.

There is **no service layer**. Controllers depend on repository interfaces directly and are themselves package-private. Business rules live in three places: bean-validation annotations on the entities, a standalone Spring `Validator` for pet forms, and inline checks in controller methods.

## Consequences

Observable from the code:

- The package dependency graph is acyclic today — `owner`, `vet`, and `system` each depend on `model`, and none depends on another feature package — but nothing enforces it. No modularity test exists, so the property holds by current fact rather than by construction.
- Controllers are the transaction boundary by default, since no application service opens one explicitly.
- Business rules that span entities are only reachable through the web layer. Duplicate-pet-name checking, future-birth-date rejection, and non-future-visit-date rejection are all implemented in controllers, so they cannot be unit-tested without MVC infrastructure.
- `PetValidator` is instantiated directly per data binder rather than injected, so it is testable in isolation but is not a managed bean.

**Recorded conflict.** `docs/architecture-principles.md` requires a domain core isolated from infrastructure, with orchestration kept thin and business logic testable without framework context, and requires module boundaries verified at test time. The organization this ADR records does not satisfy either. This ADR records the conflict; it does not resolve it and offers no rationale for it.

## Implementation

**Requirements:** none directly — this decision is structural. It underpins the requirements in `docs/prd.md` without implementing any single one.

## References

- [system-design.md § Package Structure](../system-design.md#package-structure) — the package map and its dependency direction
- [system-design.md § Open Questions from the Survey](../system-design.md#open-questions-from-the-survey) — the unresolved gaps this organization leaves
