# Java 17 and Spring Boot as the Application Platform

**Status:** Accepted

> **Provenance: derived from code, unconfirmed.** This ADR was reconstructed from the working tree during the bootstrap survey. The file date is the date the decision was *recorded*, not the date it was made — the actual decision date is unknown. Only the Decision section is evidenced by code.

## Context

**Not recorded, and not recoverable.** The code proves this decision was taken. It cannot show what problem it solved, what constraints applied, or when. Nothing is inferred here.

## Options Considered

**Not recorded, and not recoverable.** No alternative platform is evidenced anywhere in the repository. Any list written here would be invention.

This project was adopted as an existing upstream codebase. The reasoning predates this repository's documentation and was never written down here, so no interview can recover it. This section stays empty by decision (2026-07-31), not as pending work.

## Decision

The application is written in Java and built against a Java 17 toolchain, pinned in `build.gradle`. It runs on Spring Boot with the Spring MVC servlet web stack, bootstrapped from a single `@SpringBootApplication` entry point (`PetClinicApplication`). Dependency versions are governed by the Spring Boot dependency-management plugin rather than declared individually.

The build additionally configures GraalVM native-image compilation, and a `RuntimeHintsRegistrar` (`PetClinicRuntimeHints`) registers the reflection and resource hints that native compilation requires.

## Consequences

Observable from the code:

- Dependencies come as Spring Boot managed starters, so the runtime library set moves as one unit rather than per-artifact.
- Framework mechanisms — dependency injection, declarative transactions, bean validation, caching, MVC binding — are available throughout, and the codebase uses them directly in place of hand-written equivalents.
- Native-image support imposes an ongoing obligation: types reached reflectively or by serialization must be registered in `PetClinicRuntimeHints`, or the native build breaks in ways the JVM build does not.
- Java 17 is the floor for both building and running.

## Implementation

**Requirements:** none directly — this decision is structural. It underpins the requirements in `docs/prd.md` without implementing any single one.

## References

- [system-design.md § Overview](../system-design.md#overview) — how the framework shapes the application's layering
- [system-design.md § Dependency Policy](../system-design.md#dependency-policy) — the managed-starter sourcing this platform implies
