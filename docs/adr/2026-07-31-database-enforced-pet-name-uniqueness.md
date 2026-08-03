# Database-Enforced Pet Name Uniqueness Within an Owner

**Status:** Accepted

> **Provenance: derived from code, unconfirmed.** This ADR was reconstructed from the working tree during the bootstrap survey. The file date is the date the decision was *recorded*, not the date it was made — the actual decision date is unknown. Only the Decision section is evidenced by code.

## Context

**Not recorded, and not recoverable.** Git history shows the constraint was added and then adjusted for PostgreSQL, but history records what changed, not why the rule exists or why it is enforced at this layer. Nothing is inferred here.

## Options Considered

**Not recorded, and not recoverable.** No alternative enforcement mechanism is evidenced in the code. Any list written here would be invention.

This project was adopted as an existing upstream codebase. The reasoning predates this repository's documentation and was never written down here, so no interview can recover it. This section stays empty by decision (2026-07-31), not as pending work.

## Decision

A pet's name is unique within its owner, and the rule is enforced **in the database** rather than only in application code. All three vendor schemas carry the constraint, expressed per dialect. H2 and MySQL use a table-level unique constraint. PostgreSQL uses a functional unique index over the lower-cased name.

The constraint is *named* `unique_owner_pet_name` under H2 and PostgreSQL only. The MySQL schema declares it unnamed, so MySQL assigns a name of its own. That divergence defeats the detection described below — recorded as a defect in [system-design.md § Known Defects](../system-design.md#known-defects).

`PetController` keeps an application-level check as well, rejecting a duplicate name before attempting the write, and additionally catches the integrity violation that a concurrent write can still produce. It identifies that violation by matching the constraint name inside the exception message and converts it into a field-level form error; any other integrity violation is rethrown.

## Consequences

Observable from the code:

- Two simultaneous submissions of the same pet name for one owner cannot both succeed. `PetClinicConcurrencyTests` drives two racing requests and asserts exactly one wins.
- The controller is coupled to the constraint's *name* through string matching on an exception message. Renaming the constraint in any vendor schema, leaving it unnamed, or a driver changing its message text, silently turns a handled duplicate into an unhandled error. This has already happened under MySQL.
- Case-insensitivity is achieved differently per vendor — a case-insensitive column type under H2, the default collation under MySQL, a functional index under PostgreSQL — so the rule's exact semantics depend on which database is running.
- The rule is enforced twice, in the schema and in the controller, and lives nowhere in the domain model.

## Implementation

**Requirements:** [`REQ-PET-002`](../prd.md#req-pet-002) — pet-name uniqueness within an owner, case-insensitive, holding under concurrent submission.

## References

- [system-design.md § Persistence](../system-design.md#persistence) — how the constraint is expressed per database vendor
- [system-design.md § Threat Model](../system-design.md#threat-model) — the concurrent-duplicate row this decision mitigates
- [Script-Managed Schema Across Three Database Vendors](2026-07-31-script-managed-multi-vendor-schema.md) — why the constraint is written three times
