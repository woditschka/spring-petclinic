# Relational Persistence via Spring Data JPA with Mapping on the Domain Types

**Status:** Accepted

> **Provenance: derived from code, unconfirmed.** This ADR was reconstructed from the working tree during the bootstrap survey. The file date is the date the decision was *recorded*, not the date it was made — the actual decision date is unknown. Only the Decision section is evidenced by code.

## Context

**Not recorded, and not recoverable.** The code proves this decision was taken. It cannot show what problem it solved, what constraints applied, or when. Nothing is inferred here.

## Options Considered

**Not recorded, and not recoverable.** No alternative persistence approach is evidenced anywhere in the repository. Any list written here would be invention.

This project was adopted as an existing upstream codebase. The reasoning predates this repository's documentation and was never written down here, so no interview can recover it. This section stays empty by decision (2026-07-31), not as pending work.

## Decision

Persistence is JPA over a relational database, accessed through Spring Data repository interfaces. The repositories are declarative: `OwnerRepository` and `PetTypeRepository` extend `JpaRepository`, `VetRepository` extends the narrower `Repository`, and their implementations are derived at runtime from method names plus one explicit JPQL query. No repository implementation is hand-written.

The JPA mapping annotations sit **directly on the domain types** — `Owner`, `Pet`, `Visit`, `Vet`, `Specialty`, `PetType`, and the shared mapped superclasses in the `model` package. There is no separate persistence model and no mapper between the two.

Aggregate boundaries follow from the mapping. `Owner` cascades to its pets and each `Pet` cascades to its visits, both fetched eagerly. No `PetRepository` or `VisitRepository` exists: pets and visits are written only by saving their owner.

## Consequences

Observable from the code:

- Under `architecture-principles.md` this is the sanctioned direct-mapping case: the project owns both the model and the schema, and the stored shape tracks the model closely. The absence of a mapper is compliant, not a missing anti-corruption layer.
- Eager fetching plus disabled open-session-in-view means an owner lookup materializes the whole owner–pet–visit graph. A batch fetch size is configured to bound the query count.
- The domain types carry infrastructure concerns (persistence annotations, generated identity) directly, so they are not free of framework coupling.
- Every write to a pet or a visit passes through `OwnerRepository`. That makes `Owner` the single consistency entry point, but it also means controllers mutate the fetched graph in place before saving.

## Implementation

**Requirements:** none directly — this decision is structural. It underpins the requirements in `docs/prd.md` without implementing any single one.

## References

- [system-design.md § Persistence](../system-design.md#persistence) — the mapping and schema-ownership consequences
- [system-design.md § Contracts](../system-design.md#contracts) — the repository and entity contracts this decision produces
