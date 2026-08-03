# Server-Side HTML Rendering with Thymeleaf

**Status:** Accepted

> **Provenance: derived from code, unconfirmed.** This ADR was reconstructed from the working tree during the bootstrap survey. The file date is the date the decision was *recorded*, not the date it was made — the actual decision date is unknown. Only the Decision section is evidenced by code.

## Context

**Not recorded, and not recoverable.** The code proves this decision was taken. It cannot show what problem it solved, what constraints applied, or when. Nothing is inferred here.

## Options Considered

**Not recorded, and not recoverable.** No alternative rendering approach is evidenced anywhere in the repository. Any list written here would be invention.

This project was adopted as an existing upstream codebase. The reasoning predates this repository's documentation and was never written down here, so no interview can recover it. This section stays empty by decision (2026-07-31), not as pending work.

## Decision

The user interface is delivered as complete HTML pages rendered on the server by Thymeleaf. Controllers return view names, and the templates live under `src/main/resources/templates`, grouped by feature and sharing a common layout fragment. There is no client-side application layer and no JSON API backing the primary workflows.

Two supporting choices are part of the same decision. Front-end assets (Bootstrap, Font Awesome) are served from WebJars packaged into the application rather than fetched from a CDN. User-facing text is externalized into per-locale message bundles under `src/main/resources/messages`, with locale resolved per session and switchable through a request parameter (`WebConfiguration`).

One route departs from the pattern: `VetController` exposes a second vet endpoint that returns a serialized `Vets` wrapper rather than a view.

## Consequences

Observable from the code:

- Thymeleaf's default expression escaping is the application's cross-site-scripting control for rendered user data; nothing in the codebase disables it.
- Open sessions are not held across rendering, since `spring.jpa.open-in-view` is disabled. Any association a template reads must therefore be fetched first, which is why the owner-to-pet and pet-to-visit associations are mapped eager.
- Adding user-facing text means adding it to every message bundle. A test (`I18nPropertiesSyncTest`) fails the build on hard-coded strings in templates and on bundles that are missing a key another bundle has.
- The `Vets` wrapper type exists solely to give the serialized vet collection a single root element.

## Implementation

**Requirements:** none directly — this decision is structural. It underpins the requirements in `docs/prd.md` without implementing any single one.

## References

- [system-design.md § Overview](../system-design.md#overview) — where rendering sits in the request path
- [system-design.md § Package Structure](../system-design.md#package-structure) — where templates and message bundles live
