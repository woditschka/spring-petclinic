# Deleting Records and Amending Booked Visits Are Deliberately Out of Scope

**Status:** Accepted

## Context

The Non-Goals table was derived from observed absence during the 2026-07-31 bootstrap survey. Its preamble records the consequence: whether any row was deliberately declined, deferred, or never considered stayed unknown. For NG-4 (deleting an owner, a pet, or a visit) and NG-5 (changing or cancelling a visit once booked), that open provenance left the boundary contestable. A request touching either row could be read as settling the question the survey left open.

The owner has now decided both rows (2026-08-08).

## Options Considered

1. **Leave both rows provisional.** Rejected: the ambiguity is the cost. A boundary that a request can reopen by implication is not a boundary.
2. **Open the capabilities.** Build deletion and visit amendment. Rejected: the table rationales hold — neither flow teaches the stack anything the create and update flows do not, and amendment adds lifecycle state the sample carries nowhere else.
3. **Confirm both rows as deliberate** (chosen).

## Decision

NG-4 and NG-5 are deliberate product decisions, not survey artifacts. The sample corrects records forward — owner and pet details can be changed — but it deletes nothing, and a booked visit is immutable. The table rationales stand as the reasons.

A future request touching either row proceeds only on an explicit owner decision recorded at intake. The request itself never reopens the row.

## Consequences

- NG-4 and NG-5 leave the open provenance consultation; the Non-Goals preamble names them decided. Every other row remains part of it.
- The sample continues to demonstrate forward-only correction. No delete or amend flow is planned.
- Narrowing either row later is a recorded owner decision with its own non-goal ADR, per the table's convention.

## Implementation

**Non-goal:** NG-4, NG-5

- [PRD Non-Goals](../prd.md#non-goals) — the confirmed rows and the narrowed preamble.
