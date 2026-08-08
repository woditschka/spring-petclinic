<!-- harness: 2026-07-31 -->
# Product Requirements Document: spring-petclinic

<!-- AGENT: This PRD is narrative prose. State WHAT the system does and WHY it matters. Never HOW — mechanism (flags, exit codes, file layouts, algorithms) lives in system-design.md. Never the decision trail — alternatives and trade-offs live in adr/. -->
<!-- AGENT: Annotate each requirement inline with its [REQ-XX-NNN] tag where the prose expresses it, and give it one "Done when" acceptance bullet carrying the same tag. The prose is the intent; the tagged bullet is the bounded, testable contract. Drop an <a id="req-xx-nnn"></a> anchor at first mention so other docs deep-link to it. -->
<!-- AGENT: A requirement is active by being in the narrative — there is no per-requirement Status field. Retire one by moving it to the Superseded list; never renumber an ID. -->

> **Provenance: derived from observed behavior, largely unconfirmed.** Every requirement in this document was reconstructed from the running system's boundary surface during a bootstrap survey — not from any statement of intent. **Observed behavior is not an intended requirement.** Each item may be a deliberate requirement, an accident of implementation, or a shipped bug, and the code cannot tell which.
>
> One thing has since been confirmed: this is a demonstration rather than a product (2026-07-31), which settles the Context and Non-Goals framing. Every individual requirement remains provisional, and ten further questions stay open — see [Open Questions](#open-questions).

## Context

The system keeps a veterinary clinic's records. It holds the people who bring animals in, the animals themselves, the appointments those animals attend, and the clinicians the clinic employs. Staff work with it through a browser. They look an owner up by name, then open that owner's record to see every pet and every past appointment. From there they add a pet or book the next visit. A separate page lists the clinic's veterinarians and the specialties they hold.

The domain vocabulary these requirements use — Owner, Pet, PetType, Visit, Veterinarian, Specialty — is defined in [`ubiquitous-language.md`](ubiquitous-language.md).

**This is a demonstration, not a product with users** (confirmed 2026-07-31). It exists to show a complete, idiomatic implementation of its stack across a domain small enough to read in full, which is what G-1 records. That framing is load-bearing for the whole document. The requirements describe what the reference implementation does, not commitments made to anyone. The capabilities it lacks are bounded by what a reader needs to see, not by what a clinic would need to run. Three consequences follow from the framing rather than counting as defects against it: the absent access control, the wide-open management surface, and the committed development credentials recorded in [system-design.md#security-context](system-design.md#security-context).

What the framing does not settle is whether each individual behavior was intended. It still cannot say whether a given rule is a deliberate illustration, an accident, or a bug. It settles only the stakes: getting one wrong is pedagogical, not operational.

## Goals

| ID | Goal | Success Metric |
|----|------|----------------|
| G-1 | Serve as a reference sample: show a complete, idiomatic implementation of the stack across a domain small enough to read in full | A reader can follow one capability end to end — from the page through to the stored record — without leaving the codebase |
| G-2 | *(further goals pending interview)* | *(pending interview)* |

<!-- G-1 records the demonstration framing confirmed 2026-07-31; its metric is the readability property that framing implies, not a measured quantity. Whether the project holds goals beyond being a reference sample is unanswered. -->

## Non-Goals

<!-- Declined scope with the reason it was declined; never silently dropped. A non-goal ADR records the path to each decision. -->

> **One reason covers this table, and it is narrower than it looks.** The demonstration framing confirmed 2026-07-31 (G-1) makes scope a function of what a reader must see. A capability that would add breadth without teaching anything new about the stack is out. That reason genuinely explains each row, but it was supplied *after* the rows were derived from absence. It is therefore a framing applied to observed gaps, not a decision recorded at the time. Two rows have since been decided: NG-4 and NG-5 are confirmed deliberate (2026-08-08) — [the non-goal ADR](adr/2026-08-08-non-goal-deletion-and-visit-amendment.md) records the decision. For every other row, whether it was deliberately declined, deferred, or never considered is still unknown, and remains part of the open consultation.

| ID | Non-Goal | Rationale |
|----|----------|-----------|
| NG-1 | User accounts, sign-in, and any restriction on who may read or change records | An access-control layer would dominate a sample this size and obscure the domain it exists to show. The consequence is recorded in [system-design.md#security-context](system-design.md#security-context), not hidden |
| NG-2 | Managing veterinarians or their specialties through the application | The veterinarian directory demonstrates reading and presenting a cached collection; adding write flows would repeat what the owner and pet capabilities already teach |
| NG-3 | Managing the list of pet types through the application | Same reason as NG-2 — pet types demonstrate a constrained reference list, and a maintenance UI adds no new pattern |
| NG-4 | Deleting an owner, a pet, or a visit | Cascading deletion across an aggregate raises questions a reference sample would have to answer at length without illustrating anything the create and update flows do not. Confirmed deliberate 2026-08-08 — [ADR](adr/2026-08-08-non-goal-deletion-and-visit-amendment.md) |
| NG-5 | Changing or cancelling a visit once booked | Booking already demonstrates validation against a date boundary; amendment would add lifecycle state the sample deliberately has none of. Confirmed deliberate 2026-08-08 — [ADR](adr/2026-08-08-non-goal-deletion-and-visit-amendment.md) |
| NG-6 | Assigning a veterinarian to a visit, or any scheduling and availability | Scheduling is a domain of its own. Including it would make the sample about calendars rather than about the stack |
| NG-7 | Billing, invoicing, and payment | A second bounded context, with money and its own correctness demands, in a sample sized to be read in one sitting |
| NG-8 | Clinical records beyond a visit's one-line description | The description field demonstrates the association; richer clinical data would add domain depth without adding a pattern |
| NG-9 | Searching for a pet, a visit, or a veterinarian — only owners are searchable | Owner search demonstrates paged prefix search once; repeating it for other entities would add surface, not understanding |

## Requirements

### Owner records

<a id="req-own-001"></a><a id="req-own-002"></a><a id="req-own-003"></a><a id="req-own-004"></a>

The clinic records each owner it deals with, holding the person's name, where they live, and a telephone number to reach them on `[REQ-OWN-001]`. Staff find an owner by last name, matching the beginning of the name and disregarding letter case. A partial name is enough, and searching for nothing brings back every owner `[REQ-OWN-002]`. An owner's record shows their contact details, every pet they own, and every visit each pet has made. One page answers "what has happened with this household" `[REQ-OWN-003]`. Contact details can be corrected at any time `[REQ-OWN-004]`.

**Done when:**
- `[REQ-OWN-001]` given a new owner with every detail supplied, when it is submitted, then the owner is recorded and their record is shown.
- `[REQ-OWN-001]` given a blank name, address, city, or telephone, when the owner is submitted, then the entry is refused and the blank field is named.
- `[REQ-OWN-001]` given a telephone that is not a number of the required length, when the owner is submitted, then the entry is refused and the telephone is named.
- `[REQ-OWN-002]` given a last name matching the start of exactly one owner's last name, when the search runs, then that owner's record opens directly.
- `[REQ-OWN-002]` given a last name matching more than one owner, when the search runs, then the matches are listed a page at a time.
- `[REQ-OWN-002]` given a last name matching no owner, when the search runs, then the search reports that no owner was found.
- `[REQ-OWN-002]` given an empty search, when it runs, then every owner is listed.
- `[REQ-OWN-002]` given a search with leading or trailing spaces, when it runs, then the result matches the same search without them.
- `[REQ-OWN-002]` given a last name differing from the stored name only by letter case, when the search runs, then it matches.
- `[REQ-OWN-003]` given an existing owner, when their record is opened, then their contact details are shown.
- `[REQ-OWN-003]` given an owner with pets, when their record is opened, then each pet is shown with its type and birth date, and each pet's visits with date and description.
- `[REQ-OWN-004]` given an existing owner, when a detail is changed and validation passes, then the record is updated and the change is confirmed.

**Edge cases:**
1. A search whose text is entirely spaces behaves as an empty search and returns every owner.
2. A request for an owner that does not exist is refused rather than showing an empty record.
3. **Known defect.** Case-insensitive matching holds under H2 and MySQL but not under PostgreSQL, whose schema stores the last name case-sensitively. The PostgreSQL schema is the defect; the requirement is the bar.

**Design:** [system-design.md#contracts](system-design.md#contracts)

### Pet records

<a id="req-pet-001"></a><a id="req-pet-002"></a><a id="req-pet-003"></a><a id="req-pet-004"></a>

Each pet is recorded against exactly one owner, with a name, a birth date, and a species chosen from a list the clinic maintains rather than typed in free-hand `[REQ-PET-001]`. Within one owner, no two pets may share a name, and letter case does not make two names different. The rule holds even when two people submit the same pet at the same moment `[REQ-PET-002]`. A pet cannot have been born in the future `[REQ-PET-003]`. A pet's details can be corrected later `[REQ-PET-004]`.

**Done when:**
- `[REQ-PET-001]` given an existing owner, when a name, birth date, and species are supplied, then the pet is recorded against that owner.
- `[REQ-PET-001]` given a missing name, birth date, or species, when the pet is submitted, then the entry is refused and the missing field is named.
- `[REQ-PET-002]` given an owner who already has a pet named Leo, when a second pet named "leo" is added, then the entry is refused as a duplicate.
- `[REQ-PET-002]` given two simultaneous attempts to add the same pet name to one owner, when both are submitted, then exactly one succeeds and the other is refused as a duplicate.
- `[REQ-PET-003]` given a birth date later than today, when the pet is submitted, then the entry is refused and the birth date is named.
- `[REQ-PET-004]` given an existing pet, when its name, birth date, or species is changed, then the record is updated.
- `[REQ-PET-004]` given an existing pet renamed to another of the same owner's pet names, when it is submitted, then the change is refused as a duplicate.
- `[REQ-PET-004]` given an existing pet keeping its own name, when it is submitted, then the change is allowed.

**Edge cases:**
1. Two different owners may each have a pet with the same name.
2. A species that is not on the clinic's list cannot be chosen.
3. **Known defect.** Under PostgreSQL and H2 a duplicate is refused on the form, naming the pet name. Under MySQL the refusal reaches the reader as an error page instead. The duplicate is never stored on any database — only the way the refusal is presented differs. The requirement is the bar; see [system-design.md#known-defects](system-design.md#known-defects).

**Design:** [system-design.md#persistence](system-design.md#persistence) · **ADR:** [ADR: Database-Enforced Pet Name Uniqueness Within an Owner](adr/2026-07-31-database-enforced-pet-name-uniqueness.md)

### Visits

<a id="req-vis-001"></a><a id="req-vis-002"></a>

A visit is booked against a particular pet and carries the date it is for and a short description of what it is for `[REQ-VIS-001]`. When booking, the pet's earlier visits are shown alongside, so whoever is booking can see what has already been done `[REQ-VIS-002]`.

**Done when:**
- `[REQ-VIS-001]` given a pet, when a description and a date later than today are supplied, then the visit is recorded and the booking is confirmed.
- `[REQ-VIS-001]` given a blank description, when the visit is submitted, then the booking is refused and the description is named.
- `[REQ-VIS-001]` given a date of today or earlier, when the visit is submitted, then the booking is refused and the date is named.
- `[REQ-VIS-002]` given a pet with earlier visits, when a new visit is being booked, then those earlier visits are shown with their dates and descriptions.

**Edge cases:**
1. Booking a visit for a pet that does not belong to the named owner is refused.
2. The date offered by default is the earliest date that would be accepted.

### Veterinarian directory

<a id="req-vet-001"></a>

The clinic publishes the veterinarians it employs with the specialties each holds, listed a page at a time. A veterinarian holding no specialty is shown as having none rather than left blank `[REQ-VET-001]`.

**Done when:**
- `[REQ-VET-001]` given the clinic's veterinarians, when the directory is opened, then each name and its specialties are listed a page at a time.
- `[REQ-VET-001]` given a veterinarian holding no specialty, when the directory is opened, then that veterinarian is shown as having none.

**Edge cases:**
1. A veterinarian's specialties are presented in a stable order rather than an arbitrary one.
2. **Known defect.** A second route returns the same list in a machine-readable form. Nothing consumes it, and it carries no requirement — see the Superseded list.

**Design:** [system-design.md#contracts](system-design.md#contracts)

### Language

<a id="req-lang-001"></a><a id="req-lang-002"></a>

The interface is presented in the reader's language, chosen from the set the clinic supports and defaulting to English. A reader can switch on request, and the choice holds for the session `[REQ-LANG-001]`. No user-facing wording is fixed to a single language, and no supported language is left partly translated `[REQ-LANG-002]`.

**Done when:**
- `[REQ-LANG-001]` given a reader with no stated preference, when any page is opened, then it is presented in English.
- `[REQ-LANG-001]` given a reader who asks for a supported language, when that request is made, then that page and every subsequent page in the session are presented in it.
- `[REQ-LANG-002]` given the supported languages, when the wording is checked, then no user-facing text is hard-coded into a page.
- `[REQ-LANG-002]` given the supported languages, when the wording is checked, then every piece of wording is present in every one of them.

**Edge cases:**
1. A request for a language the clinic does not support falls back rather than failing.

### Entry point and failures

<a id="req-sys-001"></a><a id="req-sys-002"></a>

The system opens on a landing page, and every page carries navigation to the owner and veterinarian areas `[REQ-SYS-001]`. When something goes wrong, the reader is shown an error page carrying a message appropriate to what failed, and no internal technical detail `[REQ-SYS-002]`.

**Done when:**
- `[REQ-SYS-001]` given a reader arriving at the system, when the landing page opens, then a welcome page is shown.
- `[REQ-SYS-001]` given any page of the system, when it is rendered, then it carries navigation to owner search and to the veterinarian directory.
- `[REQ-SYS-002]` given an unhandled failure, when the request ends, then an error page distinguishes a missing page from an internal failure.
- `[REQ-SYS-002]` given an unhandled failure, when the error page is shown, then it carries no exception message, stack trace, or other internal detail.

**Edge cases:**
1. **Known defect.** The error page renders the underlying exception message to the reader today, under a source comment marking it "for developers". Confirmed 2026-07-31 as a defect, not an intended demonstration behavior. The second acceptance bullet is the bar.

## Superseded

<!-- Retired requirements: each ID maps to its successor (or to the reason it was withdrawn) so existing links still resolve. Keep this a list, so every retired ID stays in a list item. -->

- `REQ-VET-002` (machine-readable veterinarian list) — **withdrawn 2026-07-31.** The endpoint exists but no consumer does, in this repository or any named client. It was derived from observed behavior and confirmed to be an implementation artifact, not a capability anyone asked for. No successor. The route remains in the code pending removal.

## Open Questions

<!-- Unresolved product questions. Each resolves into a requirement, a non-goal, or an ADR. -->

- ~~**Is this a demonstration or a product?**~~ **Answered 2026-07-31: a demonstration / reference sample.** Recorded in Context and G-1, and it supplies the framing reason across the Non-Goals table. It does not settle whether any individual behavior was intended.
- ~~**Is the vocabulary right?**~~ **Answered 2026-07-31: correct as-is.** For a reference sample the codebase's own terms are canonical; there is no external domain expert to defer to. [`ubiquitous-language.md`](ubiquitous-language.md) is confirmed, and its recorded naming collisions stand.
- ~~**Is owner search meant to be case-insensitive?**~~ **Answered 2026-07-31: yes.** Stated in `REQ-OWN-002`. The PostgreSQL schema stores the last name case-sensitively and is the defect — recorded as edge case 3 of Owner records.
- ~~**Is `REQ-VET-002` a real requirement?**~~ **Answered 2026-07-31: no.** Withdrawn to the Superseded list as an implementation artifact.
- ~~**Should the error page show technical failure detail to readers?**~~ **Answered 2026-07-31: no.** Stated in `REQ-SYS-002`; the current behavior is recorded as a defect.
- ~~**Are two message keys dead vocabulary?**~~ **Answered 2026-07-31: yes.** The keys for a duplicate form submission and for a non-numeric value are produced by no code and are translated into all eleven languages. They are dead vocabulary pending removal.
- **Is a visit really unrelated to a veterinarian?** Nothing connects the two, which makes the veterinarian directory a standalone page rather than part of the clinical workflow.
- **Are the seeded pet types and specialties the real lists,** or placeholders?
- **Does the project hold goals beyond being a reference sample?** G-2 is unfilled.
