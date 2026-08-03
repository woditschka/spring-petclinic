<!-- harness: 2026-07-31 -->
# Architecture Principles

This document carries the tactical pattern catalog this project builds with. It specializes the strategic properties the harness works from — one canonical vocabulary, bounded modules, an isolated unit-testable domain core, and the state-vs-history document split.

**This brief is the single surface for adapting the architecture style**, and it has two layers. The **closed properties** under *Domain Core* are the kernel every brief realizes but does not rewrite. The **open pattern catalog** ships an opinionated default the project is free to adapt. The `design-validation` skill reads this file and enforces it as written — it holds no competing copy, so changing a pattern here changes what is enforced.

## Design Principles

The high-level laws designs are evaluated against. The `design-validation` skill enforces them at triage. Each entry states the principle, why it holds, and how a reviewer applies it — the application clause is the part that decides pass or fail.

1. **Security and reliability are designed in, not retrofitted.** A control added after the fact covers only the paths its author remembered; one designed into a boundary covers every path through it. *Apply:* a change that introduces or crosses a trust boundary names the boundary and its control in the design before code is written. A new boundary with no named control fails.

2. **Consistency over novelty.** A second way of doing the same thing doubles what a reader must hold in mind and halves how often each way is exercised. *Apply:* when a pattern in the Pattern Catalog covers the case, use it. Departing requires an ADR that names the pattern departed from. "It reads better" is not a departure reason.

3. **Explicit dependencies.** A collaborator reached implicitly cannot be substituted in a test or audited at a boundary. *Apply:* a type receives its collaborators as constructor arguments. A type that constructs its own collaborator, or reaches global or ambient state, fails.

4. **Layer respect.** Layers exist so a change lands in one place; logic that drifts upward stops being reachable without the layer above it. *Apply:* three checks, each readable from imports and method bodies. The domain core imports no infrastructure type. An application service branches on no domain state. An entry point contains no business rule.

5. **Minimal surface.** Every exported name is a promise to every future caller, and a promise is cheaper to make than to withdraw. *Apply:* a type or member is module-internal unless another module uses it today. Widening visibility requires naming the caller that needs it.

6. **Understandable systems.** What cannot be reasoned about cannot be secured, reviewed, or changed safely. *Apply:* a reviewer states what a unit does from its name and signature alone. If the body is required to answer that, the name is wrong.

7. **Fail secure.** A failure that leaves work half-applied is worse than one that does nothing, because the caller cannot tell which happened. *Apply:* on an error path, no partially-applied write survives, no secret reaches the message, and the caller receives the failure rather than a default value.

## Module Boundaries

The unit of architecture is the module: a bounded context with a public API and hidden internals.

| Rule | Rationale |
|------|-----------|
| Modules depend only on other modules' public APIs | Internal implementation changes don't cascade |
| No circular dependencies between modules | Keeps the dependency graph a hierarchy; cycles make modules un-reasonable in isolation |
| Each module owns its configuration | Prevents god-config; the module stays independently understandable |
| Cross-module orchestration lives in a dedicated service or the entry point | Modules stay independent of each other's execution order |

**Default shape: modulith.** A single deployable unit with enforced internal module boundaries. It provides the separation of microservices without network boundaries, deployment pipelines, or distributed consistency problems — costs that outweigh the benefits at single-team scale.

**Enforcement.** Module boundaries are verified at test time: a modularity test fails the build when a module reaches into another's internals or a dependency cycle appears. Boundaries are not advisory.

**Module = bounded context.** Each module typically maps to one bounded context. If two modules share domain types, decide explicitly: they are one context, or they are two contexts needing a translation layer between them.

## Domain Core

Business logic lives in the domain core, isolated from infrastructure and testable without it. Orchestration stays thin: services coordinate sequencing and error handling; the core decides. When logic leaks into orchestration, it stops being unit-testable in isolation — that is the failure signal.

These properties are **closed**: adapt the *how* in the Pattern Catalog, not these.

| Property | Rationale |
|----------|-----------|
| Domain objects are immutable; collections use defensive copies; no setters | Eliminates shared-mutable-state bugs; safe to pass between pipeline steps without synchronization |
| Invariants are enforced when a domain object is first constructed; rebuilding one from stored state restores an already-valid instance | The domain is valid by construction, and only valid states are ever persisted |
| The domain core holds no infrastructure logic — no I/O, queries, transactions, or DI wiring | The same model runs under any infrastructure; swap the store and the domain is unchanged |
| Aggregates are the consistency boundary: outside code enters only through the root, and aggregates reference each other by identity | Invariants enforced in one place; boundaries stay boundaries |
| Anti-corruption guards every boundary the project does not control; infrastructure mechanics never dictate domain shape | The domain is the fixed point; infrastructure is a swappable boundary |
| Configuration is typed and immutable, validated at startup | Fail fast on invalid configuration; no hidden defaults buried in code |
| Errors flow outward; each layer wraps with context; log only at boundaries | Callers decide handling; no double-logging; per-item failure never aborts a batch |

### Scoped exception: framework-mapped persistence entities

**Granted 2026-07-31 by human decision, recorded in the handoff log.** This project's persisted types are framework-mapped entities. The mapping lives on the type, and the store assigns identity on insert. Rebuilding an instance requires a no-argument constructor and field mutability. Those requirements are incompatible with the first three closed properties in the Domain Core table.

The exception is narrow, and the boundary is what makes it safe to grant:

| | |
|---|---|
| **Covers** | Types the persistence framework instantiates and populates directly, and only the three properties the framework forces: mutability, absence of construction-time invariant enforcement, and the persistence mapping the type carries |
| **Does not cover** | Every other domain type. A value object, a domain service, a configuration type, or any type the framework does not instantiate realizes the closed properties in full — no exception applies |
| **Does not cover** | The remaining closed properties, which bind persistence entities too: no I/O, queries, transactions, or DI wiring on the type; aggregates entered only through the root and referencing each other by identity; anti-corruption at every boundary the project does not control; errors flowing outward |
| **Reviewer applies** | An entity outside the covered set, or a covered entity that violates an uncovered property, fails. A new mutable type that the framework does not instantiate is not covered and fails |

The exception records where this codebase stands; it does not endorse spreading the style. A design that could satisfy the closed properties and chooses not to still fails — the exception is a boundary, not a default.

## Pattern Catalog

This section is **open**: it ships a harness default that the project adapts to its own style. Replace or add a pattern only with one that still realizes the Domain Core properties. The catalog is split by whether a pattern is actually used here. A catalog listing patterns the codebase does not contain gives a reviewer nothing to match a change against.

### In force

| Pattern | Rule | Realizes |
|---------|------|----------|
| **Value object** | Immutable data defined by its attributes; equality by value; no identity | Cheap real objects for tests; no shared mutable state |
| **Aggregate** | Root owns the consistency boundary for its children; outside code enters only through the root | Invariants enforced in one place |
| **Repository** | Persistence gateway, one per aggregate root; the default persistence boundary | The core never touches I/O directly |
| **Web controller** | Adapts one HTTP surface to the domain: binds the request, delegates, selects the response. Holds no business rule | The domain stays reachable without the web layer |
| **Form validator** | Stateless check over one bound type, reporting field-level rejections; expresses rules the type's own constraints cannot | Validation is testable without the web layer |
| **Type formatter** | Bidirectional conversion between a domain type and its display form, for request binding and rendering | Parsing lives in one place per type, not per form |
| **Mapped superclass** | Shared persistence-mapped state factored out of sibling entities; never instantiated directly | One definition of identity and shared fields |
| **Framework configuration** | Declares beans and framework wiring for one concern; carries no business rule | Wiring is grouped by concern, not scattered |

**Controllers carry rules today that this catalog places elsewhere.** Duplicate-name, future-birth-date, and non-future-visit-date checks live in controller methods. The *Web controller* row states the rule; the deviation is recorded in [`system-design.md`](system-design.md#package-structure) and stays an open question. A new rule added to a controller is a fresh violation, not covered by the existing one.

### Available, not currently used

These remain sanctioned. A change that needs one adopts it without an ADR; a change that reaches for something else instead needs one.

| Pattern | Rule | Realizes |
|---------|------|----------|
| **Application service** | Thin; sequences the use case and owns the transaction boundary; no business logic | Orchestration never absorbs the core; one place opens and closes the transaction |
| **Domain service** | Stateless; business logic that belongs to no single entity | Logic stays in the testable core |
| **Anti-corruption mapper** | A single pure function taking the source values as arguments and returning the mapped object, or an error / fallback; imports no foreign type | A foreign-format change touches one mapper, never the domain |

### Persistence and boundary mapping

Persistence is a spectrum; choose per project, and the domain core is identical across all of them:

1. **Event-sourced / in-memory** — the model object graph is materialized by folding an event stream (e.g. a log or broker), with no other persistence layer. A relational store is equally valid; neither is privileged.
2. **Repository with an anti-corruption mapper** — the default when the store's shape diverges from the model.
3. **Direct mapping** — when the project **owns both ends** and persistence **follows the model closely**, the model may carry persistence or serialization mapping metadata directly. This is the sanctioned substitute for a hand-written mapper at that controlled boundary — compliant, not a missing anti-corruption layer.

Direct mapping has two gates: the project owns both ends, **and** the stored shape tracks the model closely enough that a separate mapper would be pure boilerplate. Otherwise keep a separate persistence model behind a mapper. Anti-corruption is mandatory only at boundaries the project does **not** control — external APIs, foreign schemas, another system's events.

## Naming

Names come from the project's canonical vocabulary (`ubiquitous-language.md`): if the PRD calls it a "feed item", the code says `FeedItem`, never `Entry` or `Record`.

| Concept | Rule |
|---------|------|
| Value objects, aggregates | Domain noun, no suffix |
| Repositories | Suffix `Repository`, one per aggregate root |
| Web controllers | Suffix `Controller`, prefixed by the domain noun or the surface it serves |
| Form validators | Suffix `Validator`, prefixed by the type validated |
| Type formatters | Suffix `Formatter`, prefixed by the type converted |
| Mapped superclasses | Suffix `Entity`, prefixed by what it factors out; never a bare domain noun |
| Framework configuration | Suffix `Configuration`, prefixed by the concern wired |
| Configuration values | Suffix `Properties`, immutable after construction |
| Domain services | Verb or action name, stateless |
| Anti-corruption mappers | `from{Source}()` / `to{Target}()`, static, pure; source values in, mapped object or fallback out |

The last two rows name concepts this codebase does not yet contain; they bind the first change that introduces one.

**Configuration is two concepts, and they take different suffixes.** A *framework configuration* declares beans and wiring — `CacheConfiguration`, `WebConfiguration`. A *configuration value* holds typed settings validated at startup. The harness default collapsed both into `Properties`/`Config`, which named neither of the classes this project has.

**Prohibited suffixes:** `Manager`, `Helper`, `Utility`, `Handler`, `Processor`, `Base`, `Info`, `Data` (as a type suffix). These names are vague, attract unrelated responsibilities, and grow into god objects. Use specific domain nouns and verbs instead. `Base` is banned as a *suffix* only: `BaseEntity` is a sanctioned mapped superclass, while `EntityBase` would not be.

## Design Validation Checklist

Before approving a design, verify:

- [ ] Placement follows the module structure; no reach into another module's internals
- [ ] No circular dependencies introduced; the modularity test passes
- [ ] Each new type matches a pattern in the catalog's *In force* or *Available* list; a type matching none needs an ADR
- [ ] New types follow the naming rules; no prohibited suffixes
- [ ] No business rule added to a web controller — the *Web controller* row is the bar, and the existing deviation does not extend to new rules
- [ ] Value objects immutable and equal by value; aggregates entered only through the root, referenced by identity. Invariants enforced at construction unless the type falls under the persistence-entity exception — in which case confirm the type is genuinely framework-instantiated and still meets every uncovered property
- [ ] Anti-corruption guards every boundary the project does not control; an owned, closely-tracked model may be mapped directly
- [ ] Persistence/serialization choices follow this brief's catalog; the domain core holds no infrastructure logic
- [ ] Domain logic testable without framework context; real objects usable in tests
- [ ] New dependencies justified against the dependency policy
- [ ] Terms match the canonical vocabulary
