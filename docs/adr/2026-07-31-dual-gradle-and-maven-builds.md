# Dual Gradle and Maven Build Definitions

**Status:** Accepted

> **Provenance: derived from code, unconfirmed.** This ADR was reconstructed from the working tree during the bootstrap survey. The file date is the date the decision was *recorded*, not the date it was made — the actual decision date is unknown. Only the Decision section is evidenced by code.

## Context

**Not recorded, and not recoverable.** The code proves both builds are maintained. It cannot show why two exist, which is authoritative, or whether the duplication is deliberate. Nothing is inferred here.

## Options Considered

**Not recorded, and not recoverable.** No record of a build-tool evaluation exists in the repository. Any list written here would be invention.

This project was adopted as an existing upstream codebase. The reasoning predates this repository's documentation and was never written down here, so no interview can recover it. This section stays empty by decision (2026-07-31), not as pending work.

## Decision

The project ships **two** complete build definitions and their wrappers: `build.gradle` with `gradlew`, and `pom.xml` with `mvnw`. Both are exercised in CI by separate GitHub Actions workflows.

The Gradle build wires a quality toolchain into the build: Spring Java Format, Checkstyle, NoHttp, CycloneDX SBOM generation with test configurations excluded, and the GraalVM native-build-tools plugin. NoHttp fails the build on plain-HTTP URLs in source or resources. Six format and Checkstyle tasks over AOT-generated sources are explicitly disabled.

`CLAUDE.md` documents the Gradle commands as this project's build interface.

## Consequences

Observable from the code:

- Any dependency, plugin, or version change must be applied in two places to keep both CI workflows green; nothing enforces that they agree.
- Neither build commits a lockfile. Version pinning comes from the Spring Boot dependency-management BOM, which leaves the dependency policy's verification step unsatisfied — see [system-design.md § Adding a New Dependency](../system-design.md#adding-a-new-dependency).
- The NoHttp check is the only mechanically enforced dependency-and-source prohibition in the project.
- Native-image compilation is available from the Gradle build, which is what makes the runtime-hints registrar necessary at build time rather than only at runtime.

## Implementation

**Requirements:** none directly — this decision is structural. It underpins the requirements in `docs/prd.md` without implementing any single one.

## References

- [system-design.md § Dependency Policy](../system-design.md#dependency-policy) — the approved sources both builds resolve from
- [Java 17 and Spring Boot as the Application Platform](2026-07-31-java-17-and-spring-boot.md) — the platform both builds target
