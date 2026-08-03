# Script-Managed Schema Across Three Database Vendors

**Status:** Accepted

> **Provenance: derived from code, unconfirmed.** This ADR was reconstructed from the working tree during the bootstrap survey. The file date is the date the decision was *recorded*, not the date it was made — the actual decision date is unknown. Only the Decision section is evidenced by code.

## Context

**Not recorded, and not recoverable.** The code proves this decision was taken. It cannot show what problem it solved, what constraints applied, or when. Nothing is inferred here.

## Options Considered

**Not recorded, and not recoverable.** No alternative schema-management approach — and no migration tool — is evidenced anywhere in the repository. Any list written here would be invention.

This project was adopted as an existing upstream codebase. The reasoning predates this repository's documentation and was never written down here, so no interview can recover it. This section stays empty by decision (2026-07-31), not as pending work.

## Decision

The application supports three relational databases: an embedded in-memory H2 by default, and MySQL or PostgreSQL selected by Spring profile. Each vendor has its own hand-maintained `schema.sql` and `data.sql` under `src/main/resources/db/`, run by Spring at startup.

JPA schema generation is switched off (`spring.jpa.hibernate.ddl-auto` is `none`), so the SQL scripts — not the JPA mapping — define the schema. Datasource URL, username, and password come from environment variables in the MySQL and PostgreSQL profile property files, each with a committed fallback default. The MySQL and PostgreSQL scripts are written to be idempotent, and those profiles run initialization on every start.

No schema-migration tool is present.

## Consequences

Observable from the code:

- The schema and the JPA mapping are maintained independently and can drift; nothing in the build compares them.
- The three schemas are not constraint-identical. Pet-name uniqueness within an owner is a table constraint under H2 and MySQL but a functional unique index over the lower-cased name under PostgreSQL — see [Database-Enforced Pet Name Uniqueness](2026-07-31-database-enforced-pet-name-uniqueness.md).
- Adding a column or constraint means editing three scripts, and any behavior depending on a constraint must hold across all three dialects.
- With no migration tool, there is no recorded path for evolving an existing populated database; idempotent scripts cover re-initialization, not migration.
- The committed plaintext credential fallbacks take effect whenever the environment variables are unset.

## Implementation

**Requirements:** none directly — this decision is structural. It underpins the requirements in `docs/prd.md` without implementing any single one.

## References

- [system-design.md § Persistence](../system-design.md#persistence) — how the scripts relate to the JPA mapping
- [system-design.md § Security Context](../system-design.md#security-context) — the committed credential defaults these profiles carry
