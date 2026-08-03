<!-- harness: 2026-07-31 -->
# System Design Document: spring-petclinic

<!-- AGENT: Current state only. The path to each decision lives in adr/. -->
<!-- AGENT: Source code is authoritative for types, interfaces, parameters, and constant values. Name each contract once, say what it guarantees and which requirement it implements, and point at the source file. Do not transcribe fields, parameters, or constant literals — in a table OR in prose. They rot when the code changes and add no design information. -->
<!-- AGENT: Cross-reference prd.md for requirements, adr/ for decisions. -->

> **Provenance: derived from code, partly confirmed.** Every statement in this document was read off the working tree at the bootstrap survey and describes what the code demonstrably does. A human has since confirmed the demonstration framing, the persistence-entity exception, and four behaviors as defects — see [Known Defects](#known-defects). Everything else remains unconfirmed against intent. Where the code shows a decision but not its reason, the reason is absent here by design — it is not inferred. The `Implements` column links each contract to the requirements it serves. Those requirements are themselves derived and provisional. A link therefore records that this code satisfies that statement, not that it was built to it. Open questions from the survey are listed under [Open Questions from the Survey](#open-questions-from-the-survey).

## Overview

PetClinic is a single-deployable Spring Boot web application managing a veterinary practice's records. It covers the owners who bring animals in, the pets they own, the visits those pets make, and the veterinarians and specialties the clinic employs. It is a server-rendered application — the browser receives complete HTML pages, and there is no client-side application layer and no JSON API for the primary workflows.

The architecture is a conventional layered Spring MVC application organized by feature package rather than by technical layer. Each feature package holds its own entities, its Spring Data repository interfaces, and its controllers together; there is no service layer between controller and repository. Controllers depend on repository interfaces directly, and the repository interfaces are declarative — Spring Data derives their implementations at runtime from method names and a small number of explicit queries. Persistence is JPA over a relational database, and the JPA mapping annotations live on the domain classes themselves rather than on a separate persistence model.

Two consequences of that shape are load-bearing. First, the `Owner` entity is the entry point for everything in the owner feature. Pets are reached through their owner, and visits through their pet. Cascade and eager-fetch settings on the associations make the whole graph available from a single owner lookup. There is no `PetRepository` and no `VisitRepository` — writes to pets and visits go through `OwnerRepository`.

Second, business rules that a domain-centric design would place inside the model live instead in the controllers and in the framework's validation machinery. Form-level rules sit in bean-validation annotations on the entities. Cross-field and cross-entity rules sit in controller methods and in a standalone `Validator`.

The `system` package holds cross-cutting web and infrastructure configuration — caching, locale resolution, and two trivial controllers. The `model` package holds mapped superclasses shared by the feature packages; it is the one package every other package depends on, and it depends on none of them.

The application is designed to run against three relational databases — an embedded in-memory H2 by default, and MySQL or PostgreSQL under a Spring profile. Schema and seed data are supplied as per-vendor SQL scripts run at startup; JPA schema generation is switched off. The build additionally supports GraalVM native-image compilation, which is why a runtime-hints registrar is present.

**Not present in the codebase:** there is no authentication, no authorization, no session-based login, and no CSRF configuration anywhere in `src`, `build.gradle`, or `pom.xml`. Every route is open to every caller. This is stated as an observation, not as an approval — see [Security Context](#security-context).

## Package Structure

All production code sits under the single Gradle/Maven root project `spring-petclinic`, in one source root, under the base package `org.springframework.samples.petclinic`. There are no submodules.

```text
org.springframework.samples.petclinic
├── PetClinicApplication          — Spring Boot entry point
├── PetClinicRuntimeHints         — GraalVM native-image reflection and resource hints
├── model/                        — mapped superclasses shared by the feature packages; depends on nothing else here
├── owner/                        — owners, pets, pet types, and visits: entities, repositories, controllers, validation
├── vet/                          — veterinarians and their specialties: entities, repository, controller
└── system/                       — cross-cutting web and infrastructure config; welcome and crash controllers
```

Non-Java production assets live under `src/main/resources`:

```text
src/main/resources
├── application.properties            — base configuration; H2 is the default database
├── application-mysql.properties      — MySQL profile: datasource from environment variables
├── application-postgres.properties   — PostgreSQL profile: datasource from environment variables
├── db/{h2,mysql,postgres}/           — per-vendor schema.sql and data.sql, run at startup
├── messages/                         — message bundles, one per supported locale
├── templates/                        — Thymeleaf server-side templates, grouped by feature
└── static/                           — static web assets
```

**Package dependency direction, as observed.** `owner`, `vet`, and `system` each depend on `model`. `owner` and `vet` do not reference each other. `system` does not reference `owner` or `vet`. `model` references none of them. The dependency graph is therefore acyclic today. There is no automated modularity test enforcing this — it holds by current fact, not by construction.

**Standing against `architecture-principles.md`.** Two of the three gaps that survey found are now settled, and one remains open.

The mutable-entity gap is **resolved by a scoped exception**, granted 2026-07-31. That brief's *Scoped exception: framework-mapped persistence entities* covers the persisted types here on three counts: mutability, absent construction-time invariants, and the mapping they carry. Every other closed property still binds them, and every non-entity type realizes the closed properties in full.

Two gaps remain, and the exception covers **neither**. No modularity test enforces the package boundaries, so the acyclic dependency graph holds by fact rather than by construction. Business rules sit in controllers rather than in an independently testable core. That breaches the *Web controller* row of the pattern catalog, which admits no business rule. [testing-principles.md](testing-principles.md#test-pyramid) records the same gap as the reason the test-shape target is not met. Both are listed under [Open Questions from the Survey](#open-questions-from-the-survey).

## Constants

<!-- Name each constant and cite the source file that owns its value; do not copy the value (source is authoritative). -->

| Name | Source | Description |
|------|--------|-------------|
| `REQUIRED` | `src/main/java/org/springframework/samples/petclinic/owner/PetValidator.java` | Error code and default message used for every missing-field rejection in pet validation |
| `unique_owner_pet_name` | `src/main/resources/db/{h2,postgres}/schema.sql` | Name of the pet-name uniqueness constraint. Load-bearing beyond the schema: `PetController` matches this string inside an integrity-violation message to detect a duplicate, so every vendor schema and the controller must agree. The MySQL schema declares the constraint **unnamed**, so the string is absent there and the match fails — see [Known Defects](#known-defects) |

Page size for owner listing and for vet listing is a local variable in each controller's pagination helper, not a named constant, and the two are declared independently. The controllers' view-name constants are private routing details and are deliberately not listed here.

## Contracts

Each row names a public type once and points at the file that owns its signature. `Implements` cites the requirements in `docs/prd.md` that the type serves.

An `Implements` value of `—` marks a contract serving no single requirement. Three kinds appear: the bootstrap entry point, the native-image registrar, and the mapped superclasses. The superclasses carry shared state for every persisted type rather than implementing a behavior.

**One requirement has no contract.** `REQ-LANG-002` — no hard-coded user-facing text, and no partly translated language — is a property of the message bundles and templates, not of any type. It is enforced at build time by a test that walks the templates and compares every bundle's keys. The guarantee lives in the test and the resources; nothing in the Contracts table can carry it.

**Invariants the rows cannot carry.** `Owner` is the aggregate entry point for the owner feature: `Pet` and `Visit` are persisted only through cascade from `Owner`, and no repository exists for either. `Vets` is a serialization wrapper that exists to give the vet list a single root element for content negotiation; it is not a persisted entity. `PetTypeFormatter` and `PetValidator` are Spring MVC binding and validation extension points, not domain services — `PetValidator` is instantiated directly per data binder rather than injected.

| Contract | Purpose | Source | Implements |
|----------|---------|--------|------------|
| `PetClinicApplication` | Spring Boot entry point; imports the native-image runtime hints | `src/main/java/org/springframework/samples/petclinic/PetClinicApplication.java` | — |
| `PetClinicRuntimeHints` | Registers reflection and resource hints so the app builds and runs as a GraalVM native image | `src/main/java/org/springframework/samples/petclinic/PetClinicRuntimeHints.java` | — |
| `BaseEntity` | Mapped superclass giving every persisted type a generated identity and a "not yet persisted" test | `src/main/java/org/springframework/samples/petclinic/model/BaseEntity.java` | — |
| `NamedEntity` | Mapped superclass adding a validated name to `BaseEntity` | `src/main/java/org/springframework/samples/petclinic/model/NamedEntity.java` | — |
| `Person` | Mapped superclass adding validated first and last names to `BaseEntity` | `src/main/java/org/springframework/samples/petclinic/model/Person.java` | — |
| `Owner` | Persisted owner; owns its pets by cascade and is the entry point for adding a visit to one of them | `src/main/java/org/springframework/samples/petclinic/owner/Owner.java` | REQ-OWN-001, REQ-OWN-003, REQ-PET-002, REQ-VIS-001 |
| `Pet` | Persisted pet; owns its visits by cascade and references its type | `src/main/java/org/springframework/samples/petclinic/owner/Pet.java` | REQ-PET-001, REQ-OWN-003, REQ-VIS-002 |
| `PetType` | Persisted lookup value naming a species | `src/main/java/org/springframework/samples/petclinic/owner/PetType.java` | REQ-PET-001 |
| `Visit` | Persisted appointment record against a pet; defaults its date forward on construction | `src/main/java/org/springframework/samples/petclinic/owner/Visit.java` | REQ-VIS-001 |
| `OwnerRepository` | Spring Data JPA repository for owners; the sole write path for the owner–pet–visit graph. Supports prefix search by last name with paging | `src/main/java/org/springframework/samples/petclinic/owner/OwnerRepository.java` | REQ-OWN-001, REQ-OWN-002, REQ-OWN-003, REQ-OWN-004, REQ-PET-001, REQ-VIS-001 |
| `PetTypeRepository` | Spring Data JPA repository for pet types, returning them in name order | `src/main/java/org/springframework/samples/petclinic/owner/PetTypeRepository.java` | REQ-PET-001 |
| `OwnerController` | Server-rendered owner workflows: create, edit, search with paging, and detail | `src/main/java/org/springframework/samples/petclinic/owner/OwnerController.java` | REQ-OWN-001, REQ-OWN-002, REQ-OWN-003, REQ-OWN-004 |
| `PetController` | Server-rendered pet workflows nested under an owner: create and edit, including duplicate-name and future-birth-date rejection | `src/main/java/org/springframework/samples/petclinic/owner/PetController.java` | REQ-PET-001, REQ-PET-002, REQ-PET-003, REQ-PET-004 |
| `VisitController` | Server-rendered visit booking for a pet, rejecting non-future dates | `src/main/java/org/springframework/samples/petclinic/owner/VisitController.java` | REQ-VIS-001, REQ-VIS-002 |
| `PetTypeFormatter` | Spring MVC formatter translating between a pet type and its display name during form binding | `src/main/java/org/springframework/samples/petclinic/owner/PetTypeFormatter.java` | REQ-PET-001 |
| `PetValidator` | Spring `Validator` enforcing the required-field rules for pet forms that bean validation does not cover | `src/main/java/org/springframework/samples/petclinic/owner/PetValidator.java` | REQ-PET-001 |
| `Vet` | Persisted veterinarian; exposes specialties sorted by name | `src/main/java/org/springframework/samples/petclinic/vet/Vet.java` | REQ-VET-001 |
| `Specialty` | Persisted lookup value naming a veterinary specialty | `src/main/java/org/springframework/samples/petclinic/vet/Specialty.java` | REQ-VET-001 |
| `Vets` | Serialization wrapper giving the vet collection a single root element for the non-HTML representation | `src/main/java/org/springframework/samples/petclinic/vet/Vets.java` | — |
| `VetRepository` | Spring Data repository for veterinarians; results are cached | `src/main/java/org/springframework/samples/petclinic/vet/VetRepository.java` | REQ-VET-001 |
| `VetController` | Serves the paged HTML vet list and a serialized vet collection from a second route | `src/main/java/org/springframework/samples/petclinic/vet/VetController.java` | REQ-VET-001 |
| `CacheConfiguration` | Enables caching and declares the vet cache through the JCache API | `src/main/java/org/springframework/samples/petclinic/system/CacheConfiguration.java` | REQ-VET-001 |
| `WebConfiguration` | Session-scoped locale resolution plus a request-parameter locale switch | `src/main/java/org/springframework/samples/petclinic/system/WebConfiguration.java` | REQ-LANG-001 |
| `WelcomeController` | Serves the landing page | `src/main/java/org/springframework/samples/petclinic/system/WelcomeController.java` | REQ-SYS-001 |
| `CrashController` | Deliberately throws on a fixed route to exercise the error page | `src/main/java/org/springframework/samples/petclinic/system/CrashController.java` | REQ-SYS-002 |

### Persistence

The relational schema is not generated from the JPA mapping. `spring.jpa.hibernate.ddl-auto` is set to `none`, and the schema is created by the per-vendor `schema.sql` scripts under `src/main/resources/db/`, with seed rows from the matching `data.sql`. The domain classes carry the JPA mapping annotations directly — there is no separate persistence model and no mapper between them. Under `architecture-principles.md` this is the sanctioned direct-mapping case: the project owns both the model and the schema, and the stored shape tracks the model closely.

The schemas are hand-maintained per vendor and are not identical in their constraints. Pet-name uniqueness within an owner is the case in point. H2 and MySQL express it as a table constraint; PostgreSQL uses a functional unique index over the lower-cased name. Case-insensitive matching is reached three different ways — H2 types the column `VARCHAR_IGNORECASE`, MySQL relies on its default case-insensitive collation, and PostgreSQL lower-cases in the index expression. All three therefore enforce the rule at the database, and the stored data is protected on every vendor.

Detection of that violation is where they diverge. `PetController` recognizes it by matching the constraint name inside the exception message, which couples the controller to a schema identifier. H2 and PostgreSQL name the constraint `unique_owner_pet_name`; MySQL declares it unnamed and is auto-assigned a name derived from its first column, so the match fails and the violation is rethrown rather than converted to a field error. The consequence is recorded under [Known Defects](#known-defects).

Open sessions are not held across view rendering (`spring.jpa.open-in-view` is disabled), so associations that a view needs are fetched eagerly at the mapping level.

## Dependency Policy

Minimize external dependencies. Every dependency is an attack surface and a maintenance burden.

> **Provenance note.** The Approved Sources table records the dependency *sources the code currently draws on*. It is a description of the present state, not a policy a human has ratified. Confirming or narrowing it is one of the open questions.

### Approved Sources

| Source | Examples | Rationale |
|--------|----------|-----------|
| Spring Boot managed starters | web MVC, Data JPA, Thymeleaf, validation, cache, actuator, devtools | Versions are resolved by the Spring Boot dependency-management plugin, so the set moves as one audited unit |
| JDBC drivers for the three supported databases | H2, MySQL, PostgreSQL | Required to satisfy the multi-vendor persistence target; all runtime-scoped |
| Jakarta EE API artifacts | cache API, XML binding API | Standard APIs the Spring stack programs against |
| WebJars for front-end assets | Bootstrap, Font Awesome, the WebJars locator | Front-end assets are served from the JAR rather than a CDN; all runtime-scoped |
| Build and quality plugins | Spring Boot, Spring dependency management, Spring Java Format, Checkstyle, NoHttp, GraalVM native build tools, CycloneDX | Build-time only; none reach the runtime classpath |
| Test-only libraries | Spring Boot test starters, Testcontainers, Spring Boot Docker Compose support | Test-scoped; excluded from the runtime SBOM |

All artifacts resolve from Maven Central; no private or mirrored repository is configured.

### Adding a New Dependency

Before adding a dependency, verify:

1. **Necessity** — Can the standard library solve the problem?
2. **Source** — Is it from a source listed under Approved Sources? If not, create an ADR.
3. **Audit** — Review transitive dependencies. Flag unknown modules.
4. **Verification** — Verify checksums and commit the lockfile.

**Observed gap against step 4.** Neither build system pins a lockfile. Gradle resolves Spring Boot's managed versions at build time and no `gradle.lockfile` is committed; the Maven build likewise relies on managed versions. Version pinning therefore comes from the Spring Boot BOM, not from a verified lock. A CycloneDX SBOM is generated by the build with test configurations excluded.

### Prohibited

<!-- Dependency classes this project rejects, with the reason. -->

- **Plain-HTTP artifact sources and plain-HTTP URLs in source or resources** — enforced mechanically: the NoHttp Checkstyle configuration is wired into the build and fails it. This is the one prohibition the codebase enforces rather than merely states.
- Everything else is undetermined. The code demonstrates no other rejected dependency class, and none is invented here.

## Security Context

<!-- PROJECT: Describe this application's security profile: what it connects to, what it exposes, how it handles credentials, and how it runs (systemd, container, etc.). The security-reviewer reads this section before reviewing. -->

> **Provenance: derived from code, unconfirmed.** This section describes the application's observed security posture. It records what is there. It does not assert that the posture is intended or acceptable — that judgment needs a human.

- **Inputs it processes:** HTTP form submissions and query parameters for owner, pet, and visit workflows. Path variables carrying owner and pet identifiers. A locale-selection request parameter. Environment variables supplying the datasource credentials under the MySQL and PostgreSQL profiles. SQL schema and seed scripts read from the classpath at startup.
- **Outputs it produces:** server-rendered HTML pages via Thymeleaf; a serialized veterinarian collection on a second vet route; Spring Boot Actuator endpoints.
- **External services it connects to:** exactly one relational database — embedded H2 by default, or MySQL or PostgreSQL when the matching profile is active. No outbound HTTP calls, no message broker, no third-party API.
- **Credential handling:** database credentials are read from environment variables in the MySQL and PostgreSQL profile files. Each carries a **committed plaintext fallback default**, used whenever the variable is unset. The same credentials appear in plaintext in `docker-compose.yml`, which additionally starts MySQL with an empty root password and `MYSQL_ALLOW_EMPTY_PASSWORD` enabled. There is no secret store integration in the application itself; the Kubernetes manifest mounts a binding secret into the container.
- **Runtime:** an executable Spring Boot JAR, or a container image built by the Spring Boot build plugin (no `Dockerfile` in the repository), or a GraalVM native image. The Kubernetes manifest runs a single replica behind a NodePort service with liveness and readiness probes on actuator health paths. Local development uses Docker Compose for the database.

**Two observations a reviewer must see first.** There is no authentication or authorization of any kind — a repository-wide search for security configuration, filter chains, method-level authorization, and CSRF settings returns nothing across `src`, `build.gradle`, and `pom.xml`. Every route, including every mutating POST, is reachable by any caller. Separately, `management.endpoints.web.exposure.include` is set to expose **all** actuator endpoints, with a source comment stating this is for development and testing only; no profile in the repository narrows it.

## Threat Model

> **Provenance: derived from code, unconfirmed.** Rows record attack surface that is observably present and the mitigation that is observably in place. A blank or qualified mitigation means the code shows none — not that the threat is accepted.

| Threat | Attack Vector | Mitigation |
|--------|--------------|------------|
| Unauthenticated data modification | Any HTTP client POSTs to the owner, pet, or visit creation and edit routes | **None observed.** No authentication, authorization, or CSRF protection exists anywhere in the codebase |
| Sensitive operational disclosure | All actuator endpoints are exposed over HTTP by configuration | **None observed** beyond the source comment restricting the setting to development and testing |
| Mass assignment / identifier tampering via form binding | Attacker submits an `id` field alongside legitimate form fields | Every controller's data binder explicitly disallows `id` and nested `id` binding; owner edit additionally rejects a form/URL identifier mismatch |
| SQL injection | User-supplied search text and identifiers reach the database | Data access is exclusively through Spring Data JPA derived queries and one parameterized JPQL query; no string-concatenated SQL exists in production code |
| Cross-site scripting through rendered user data | Owner, pet, and visit fields are echoed into HTML pages | Thymeleaf escapes expression output by default; the application does not disable it |
| Duplicate pet name for one owner under concurrent submission | Two simultaneous create requests for the same owner and pet name | A database uniqueness constraint on owner plus pet name in all three vendor schemas, so exactly one write wins on every vendor. A concurrency test asserts this. The controller's conversion of the violation into a field error is vendor-dependent and does not hold under MySQL — see [Known Defects](#known-defects). Integrity is enforced regardless; only the presentation of the refusal differs |
| Credential exposure from committed defaults | The profile property files and `docker-compose.yml` carry plaintext database credentials, and the compose file permits an empty MySQL root password | **None observed.** The fallback values are committed and take effect whenever the environment variables are unset |
| Unvalidated input reaching persistence | Form fields for owner, pet, and visit | Bean-validation constraints on the entities cover required fields, length, and telephone format; `PetValidator` and controller checks cover pet required fields, future birth dates, and non-future visit dates |
| Deserialization of untrusted data | Java serialization is enabled by native-image hints for three entity types | **Not assessed.** The hints register Java serialization for those types; no untrusted deserialization entry point was found in the survey, but this was not exhaustively verified |

## Implementation Order

| ID | Name | Depends On |
|----|------|------------|
<!-- Empty by construction: the codebase predates the PRD, so no requirement has an implementation order yet. Populated when docs/prd.md defines requirements. -->

## State Machine

<!-- Define state transitions as parseable tables when the system carries state. -->

The application carries no explicit state machine. The only lifecycle distinction is whether an entity has been persisted. `BaseEntity` exposes it as a single test, and the pet workflows branch on it to choose between insert and update. No status field, workflow, or transition table exists in the code.

## Known Defects

Behaviors confirmed 2026-07-31 as defects rather than intended demonstration properties. Each contradicts a requirement or serves none. All remain in the code; none is fixed here. The final row is derived from code and has not been put to a human — it is listed because the contradiction is demonstrable in the source, not because intent was established.

| Defect | Breaches | Detail |
|---|---|---|
| PostgreSQL owner search is case-sensitive | `REQ-OWN-002` | The PostgreSQL schema stores the last name case-sensitively, while H2 and MySQL match case-insensitively. The same search returns different results per database |
| The error page renders the exception message | `REQ-SYS-002` | Internal failure text reaches the reader, under a source comment marking it "for developers" |
| The machine-readable veterinarian route serves no requirement | — | Nothing consumes it. The requirement derived from it was withdrawn to the PRD's Superseded list; the route remains pending removal |
| Two message keys are dead vocabulary | — | Wording for a duplicate form submission and for a non-numeric value is translated into all eleven locales but produced by no code |
| Duplicate pet names are not detected under MySQL *(derived, unconfirmed)* | `REQ-PET-002` | Detection matches the constraint name `unique_owner_pet_name` in the integrity-violation message, but the MySQL schema declares that constraint unnamed. The match fails, the exception is rethrown, and the reader is shown the error page instead of the form carrying a field error. The database still rejects the duplicate, so no data is corrupted — only the refusal the requirement describes is not delivered. H2 and PostgreSQL are unaffected, and the H2-backed test suite cannot observe it |

## Open Questions from the Survey

Gaps found during the bootstrap survey that a human has not yet settled. None has an inferred rationale attached.

1. **No modularity test.** `architecture-principles.md` requires module boundaries verified at test time. The package graph is acyclic today by fact, not by enforcement.
2. **No service layer.** Controllers depend on repositories directly, and business rules live in controllers and validators. This breaches the pattern catalog's *Web controller* row; whether the code or the pattern moves is undecided.
3. **Constraint-name coupling.** Duplicate pet-name detection matches a constraint name inside an exception message, and the three vendor schemas express that constraint differently. The coupling is already broken under MySQL ([Known Defects](#known-defects)). Whether to repair it by naming the MySQL constraint, or to remove the coupling by detecting the violation without reference to a schema identifier, is undetermined — as is whether the H2-only default test suite should be able to catch a divergence of this kind.
4. **Two parallel build systems.** Both Gradle and Maven configurations are committed and both run in CI. Which is authoritative is undetermined.
5. **Vet cache invalidation.** The vet repository's read methods are cached with no eviction configured. The application has no write path for veterinarians, so the question is whether vet data is expected to change out of band.
6. **No dependency lockfile.** Version pinning relies on the Spring Boot BOM. The dependency policy's own verification step calls for a committed lockfile.
7. **Visits are unrelated to veterinarians.** Nothing connects the two, which leaves the veterinarian directory a standalone page rather than part of the clinical workflow.

**Settled since the survey.** Requirements now exist and the `Implements` column is wired. The mutable-entity conflict is covered by a scoped exception. The security posture is confirmed as a demonstration's, and `security-principles.md` § Realization is written against it. The domain vocabulary is written and confirmed.
