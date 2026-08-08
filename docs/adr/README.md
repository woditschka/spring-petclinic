<!-- harness: 2026-07-31 -->
# Architecture Decision Records

This directory is the project's decision log. ADRs document the path to each decision — the options considered, trade-offs evaluated, and rationale for the choice. The current state of all accepted decisions is reflected in [`system-design.md`](../system-design.md), the authoritative reference for implementation.

## Format

Each ADR is a markdown file named `YYYY-MM-DD-title-in-kebab-case.md`.

### Template

```markdown
# [Title]

**Status:** Proposed | Accepted | Deprecated | Superseded by [ADR-YYYY-MM-DD]

## Context

[Why is this decision needed? What problem are we solving?]

## Options Considered

1. **Option A** - [Brief description]
2. **Option B** - [Brief description]

## Decision

[What did we choose and why?]

## Consequences

[What are the results? Both positive and negative.]

## Implementation

**Requirements:** REQ-XX-NNN

## References

- [Link to the system-design.md section realizing this decision](../system-design.md)
```

### Guidelines

- One decision per file
- Document the path: what options existed, why we chose this one
- Keep it concise: aim for under 60 lines
- Write in present tense ("We use X" not "We will use X")
- Link related ADRs when decisions interact
- Update status when decisions change; supersede, don't delete

### Non-Goal ADRs

A non-goal ADR records a *product* decision not to build something — distinct from an architectural ADR. Two conventions apply:

- **Filename:** `YYYY-MM-DD-non-goal-<slug>.md` — the `non-goal-` infix is load-bearing: the product-requirements-expert agent's write scope matches this pattern; standard ADRs remain owned by the system-design-expert.
- **Implementation section:** use `**Non-goal:** NG-X` (referencing the PRD's Non-Goals table) instead of `**Requirements:**`.

## Index

> **Provenance of the 2026-07-31 entries: derived from code, unconfirmed.** These were reconstructed from the working tree during the bootstrap survey, before any requirements existed. Each records only the *decision* the code proves; its Context and Options Considered are deliberately empty and marked for human interview. The date is when the ADR was recorded, not when the decision was made.

| Date | Decision | Status |
|------|----------|--------|
| 2026-07-31 | [Java 17 and Spring Boot as the Application Platform](2026-07-31-java-17-and-spring-boot.md) | Accepted |
| 2026-07-31 | [Server-Side HTML Rendering with Thymeleaf](2026-07-31-server-side-rendering-with-thymeleaf.md) | Accepted |
| 2026-07-31 | [Relational Persistence via Spring Data JPA with Mapping on the Domain Types](2026-07-31-relational-persistence-with-spring-data-jpa.md) | Accepted |
| 2026-07-31 | [Script-Managed Schema Across Three Database Vendors](2026-07-31-script-managed-multi-vendor-schema.md) | Accepted |
| 2026-07-31 | [Feature-Package Organization Without a Service Layer](2026-07-31-feature-package-organization.md) | Accepted |
| 2026-07-31 | [Database-Enforced Pet Name Uniqueness Within an Owner](2026-07-31-database-enforced-pet-name-uniqueness.md) | Accepted |
| 2026-07-31 | [Dual Gradle and Maven Build Definitions](2026-07-31-dual-gradle-and-maven-builds.md) | Accepted |
| 2026-08-08 | [Deleting Records and Amending Booked Visits Are Deliberately Out of Scope](2026-08-08-non-goal-deletion-and-visit-amendment.md) | Accepted |
