<!-- harness: 2026-07-31 -->
# Ubiquitous Language

<!-- The canonical domain vocabulary for this project — the "ubiquitous language" in the Domain-Driven Design sense (Eric Evans, Chapter 2). The same terms are used by stakeholders, the PRD, the system design, and the source code.

  This file is durable memory. Agents and developers across sessions read it to keep the same vocabulary; drift here ripples into variable, function, and file names across the codebase.

  ENTRY FORMAT

    **TermName**: One-sentence definition. Define what it IS, not what it does. Relationships: (optional) one line naming the related concepts and the cardinality where obvious. Avoid: (optional) other words used in the wild for the same concept that this project rejects. Listing them heads off drift.

  Example:

    **Order**: A customer-placed request for one or more line items, accepted for fulfillment but not yet shipped. Relationships: An Order contains one or more LineItems and references one Customer. Avoid: Purchase, Transaction (those are billing-context terms).

  WHEN TO ADD A TERM

  - The moment a term resolves during a requirements interview or a design discussion. Do not batch. Do not wait for a second use.
  - When recurring domain terms in existing docs or source code aren't yet captured (adoption case — the system-design-expert may write here during a foundational triage to seed the initial vocabulary).

  WHAT NOT TO PUT HERE

  - Harness methodology vocabulary (slice, loop, triage verdict, etc.) — that is the team's method, not this project's domain. This file holds project-domain terms only.
  - Implementation details. Definitions describe domain concepts, not code shape.

  CONSUMERS

  - The product-requirements-expert resolves and writes PRD terms.
  - The system-design-expert resolves design terms; it writes here only during the foundational triage path on adoption.
  - The doc-reviewer lints cross-document term consistency.

  CADENCE

  Slow. The ubiquitous language changes less often than the PRD or system design. Treat updates here as load-bearing — they ripple into variable, function, and file names across the codebase. -->

> **Derived from code, confirmed 2026-07-31.** Every term here was extracted from persisted types, table names, seed data, message-bundle keys, and page text during the bootstrap survey. A human has confirmed it as canonical. This project is a reference sample, so the codebase's own terms are the vocabulary; there is no external domain expert to defer to. The naming collisions recorded at the end stand as known tensions, not as pending questions.

## Domain Terms

**Clinic**: The veterinary practice whose records this system keeps — its owners, their pets, the visits those pets make, and the veterinarians it employs. Relationships: One Clinic; the system does not model more than one. Avoid: Practice, Surgery (the latter collides with the Specialty named "surgery").

**Owner**: A person who owns one or more pets and whose contact details the clinic holds so it can reach them. Relationships: An Owner has zero or more Pets; every Pet has exactly one Owner. Avoid: Customer, Client, Patient (the animal is the patient, not the person).

**Pet**: An animal belonging to exactly one Owner and treated at the clinic. Relationships: A Pet has exactly one Owner, exactly one PetType, and zero or more Visits. Avoid: Animal, Patient.

**PetType**: The species classification of a Pet, chosen from a list the clinic maintains outside the application. Relationships: A PetType classifies zero or more Pets. The seeded list is cat, dog, lizard, snake, bird, and hamster. Avoid: Species, Breed (breed is a finer distinction this system does not make), and bare "Type" — that word is ambiguous in a codebase.

**Visit**: A dated record of one Pet's appointment at the clinic, carrying a short description of what the appointment was for. Relationships: A Visit belongs to exactly one Pet. Seeded descriptions are "rabies shot", "neutered", and "spayed". Avoid: Appointment, Booking, Consultation, Treatment.

**Veterinarian**: A clinician the clinic employs, listed publicly with the specialties they hold. Relationships: A Veterinarian holds zero or more Specialties; a Specialty is held by zero or more Veterinarians. A Veterinarian is not linked to a Visit. Avoid: Doctor, Clinician, Surgeon. **"Vet" is the accepted short form in code and page text; "Veterinarian" is the term for prose.**

**Specialty**: A named field of veterinary practice that a Veterinarian holds. Relationships: A Specialty is held by zero or more Veterinarians. The seeded list is radiology, surgery, and dentistry. Avoid: Skill, Qualification, Discipline.

### Naming collisions recorded during the survey

These are places where the codebase uses one word for two things, or two words for one thing. They are recorded so the vocabulary can settle them; none is resolved here.

- **Vets** names both the plural of Veterinarian and a distinct wrapper type used for serialization. Prose refers to the people as "veterinarians"; the wrapper is referred to only as a code symbol, in backticks, and never as a domain concept. Writing "the vets" and meaning the wrapper is the error to avoid.
- **Type** appears as a form field, **PetType** as the concept, and **types** as the table. "PetType" is the term.
- **Visit date** is called `date` on the record and `visit_date` in the schema. "Visit date" is the term.
- A test-only package implies a **clinic service** concept that does not exist in the application.

## Example Dialogue

> **Illustrative, not a transcript.** This exchange was constructed to show how the Domain Terms interact. It uses only behavior observed in the codebase; no such conversation took place, and it is not evidence of intent.

**Developer:** When someone rings up and says "I need to bring Leo in", what do I need before I can write anything down?

**Domain expert:** You need the Owner first. Leo is a Pet, and every Pet hangs off exactly one Owner — we don't keep an animal that isn't attached to somebody. So you find the Owner by last name, open their record, and Leo is already there under them.

**Developer:** And if there are two Leos?

**Domain expert:** Not under the same Owner, there aren't. One Owner can't have two Pets with the same name — it's how we tell them apart when we're on the phone. Different Owners can both have a Leo, that's fine and it happens.

**Developer:** So I book the appointment against Leo, not against the Owner.

**Domain expert:** Against the Pet. That's a Visit — a date and a line about what it's for, "rabies shot" or similar. It goes on Leo's record, and when you open the Owner's page you see every Pet they have and every Visit under each one.

**Developer:** Do I say which Veterinarian is seeing him?

**Domain expert:** No. A Visit doesn't name a Veterinarian. The Veterinarian list is a separate thing — it's who we employ and what Specialties they hold, radiology or surgery or dentistry. People look at it, but it isn't connected to the Visit.

**Developer:** And Leo's a cat — where does that come from?

**Domain expert:** That's his PetType. You pick it from the list; you don't type it in. Cat, dog, lizard, snake, bird, hamster. If we ever needed a new one, that isn't something you'd do from the screen.
