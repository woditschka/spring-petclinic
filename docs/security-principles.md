<!-- harness: 2026-07-31 -->
# Security Principles

This brief specializes the harness's non-negotiable security laws for this project. The four laws — security as an emergent property, defense in depth, least privilege, fail secure — are harness-owned: a project decides *how* it meets them, never *whether*. This document carries that "how": the project's trust-boundary map and the stack's state-of-the-art security defaults. The feature-implementer designs against it; the security-reviewer enforces it; the exhaustive item-by-item checklist lives in the `security-review` skill.

## Trust Boundaries

A trust boundary is any point where data or control crosses from less-trusted to more-trusted. Every boundary the change introduces or crosses gets the same treatment.

| At the boundary | The rule |
|---|---|
| External input arrives (request, file, env, message) | Validate type, range, and shape before use; reject what the contract does not allow |
| A secret is read (token, key, password) | It never reaches a log, an error message, a URL, or a process argument |
| A secret is stored | It lives in the environment or a secret store, never in committed source |
| A privilege is exercised (file, network, process, query) | Grant the minimum scope; deny by default |
| An error crosses back out | The message carries debugging context inward, never internal detail outward |

Internal code, past the boundary, trusts its contracts — defensive checks belong at the boundary, not scattered through the core.

## Realization

The vulnerability classes that matter in this Java/Spring Boot web application, each with the principle it puts at risk and the control that prevents it. Derived from the dependency policy and threat model in [`system-design.md`](system-design.md#threat-model). These are the bar a change starts from, not a floor to argue down. The `security-review` skill holds the exhaustive checklist.

### What this application is, and what that does not excuse

**This is a demonstration application** (confirmed 2026-07-31; recorded in the handoff log). It has no authentication, no authorization, and no CSRF protection; every route including every mutating one is open, all management endpoints are exposed, and database credentials carry committed plaintext fallbacks. A reviewer does **not** raise these as defects — they are the demonstration's shape, recorded in `system-design.md` § Security Context.

What the framing does not excuse is a change that makes things worse than that baseline, or that teaches a reader something false. A demonstration is read and copied; a vulnerable pattern in it propagates further than the same pattern in a private product. Every control in this section applies in full.

| Vulnerability class | Principle at risk | Control that prevents it |
|---|---|---|
| **Injection into data access** — a query built by concatenating user-supplied text | Validate at the boundary | Data access goes through the repository abstraction's derived queries or explicitly parameterized queries. String-concatenated query text containing any request-derived value fails review, with no exception for "it is only a demo" |
| **Cross-site scripting** — user-supplied text rendered into a page as markup | Fail secure | The template engine's default output escaping stays on. A change that disables escaping for a value, or that injects markup into the DOM from a request-derived value, names the reason and the sanitization applied |
| **Mass assignment** — a request binding a field the form never offered, an identifier being the damaging case | Least privilege | Every request-bound type explicitly disallows identifier binding. A new controller or binder that omits the disallow list fails, since the safe state must be the default rather than remembered per-endpoint |
| **Path traversal and unsafe resource resolution** — a request-derived value used to select a file or classpath resource | Least privilege | A request-derived value never composes a filesystem or resource path. Resource selection uses a fixed allowlist, not caller input |
| **Unsafe deserialization** — untrusted bytes reconstructed into objects | Fail secure | Java serialization is enabled by native-image hints for three types. No endpoint may accept serialized object input; deserialization stays confined to framework-internal use |
| **Secret disclosure through logs and errors** — a credential or connection string reaching output | Secrets never surface | No credential, connection string, or session identifier appears in a log line, an exception message, a URL, or a rendered page. The error page today renders the underlying exception message, so a new exception message must not carry sensitive values into it |
| **Newly committed credentials** — a real secret added to the repository | Secrets are stored outside source | The committed development defaults are the demonstration's baseline and stay. Adding a *new* credential that is not an obviously-fake local default fails, whatever file it lands in |
| **Dependency and supply-chain risk** — a new dependency widening the attack surface | Defense in depth | A new dependency clears the four checks in `system-design.md` § Adding a New Dependency. Artifacts resolve over TLS from the declared repository; the build's plain-HTTP check stays enabled |
| **Widening the exposed surface** — a new endpoint, or broader management exposure | Least privilege | A change that adds an endpoint states what it exposes and to whom. Broadening management-endpoint exposure beyond the current baseline, or exposing one that mutates state, fails |
| **Trusting cross-request state** — assuming a prior request's validation still holds | Validate at the boundary | Each request re-resolves and re-validates the entities it acts on. A handler that trusts an identifier because an earlier request validated it fails |

### Applying this section

A reviewer decides pass or fail per row, against the change only, by asking two questions. Does the change introduce a class in this table? Does it leave the application weaker than the baseline in [system-design.md#security-context](system-design.md#security-context)? A "no" to both passes. The pre-existing absences in that baseline are never findings; anything the change itself introduces always is.
