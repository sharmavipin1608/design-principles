# P-18 · Backward Compatibility & Versioning

> A change to a shared contract doesn't break the consumers who are already relying on it.

**Tier:** High · **Scope:** any API, schema, message format, or shared library interface with consumers you don't control or can't force to update simultaneously

## What it means

The moment a contract has more than one consumer, or any consumer you can't deploy in lockstep with, changing it stops being a local decision. Backward compatibility means new versions of a contract don't break clients written against the old one — additive changes (new optional field, new endpoint, new enum value handled by a documented default) are safe; removing a field, narrowing a type, renaming a required parameter, or changing existing behavior are not, unless there's an explicit migration path. The standard technique for the changes that genuinely can't be additive is expand/contract: add the new shape alongside the old one, migrate consumers over time, and only remove the old shape once nothing depends on it anymore — never in the same change that introduces the new one.

This applies as much to database schemas and internal message formats as it does to public HTTP APIs — anywhere a producer and a consumer can be deployed at different times, which in practice is almost everywhere in a system with more than one deployable unit.

## Why it matters

A breaking change to a shared contract doesn't fail at the point of the change — it fails at every consumer, at a time the person who made the change doesn't control and often isn't watching. That's what makes this a High-tier concern with a uniquely bad blast radius: the failure surfaces as someone else's outage, in someone else's service, hours or days after your deploy, and by the time it's traced back to your change, the fix requires either an emergency rollback or an emergency compatibility patch under incident pressure — the exact opposite of the low-cost, unhurried fix that an expand/contract migration would have been if done up front.

## What good looks like

- New fields added to a schema/API response are optional/nullable with a sensible default, so old consumers that don't know about them keep working.
- A required field is never removed or renamed in a single change — it's deprecated, consumers are migrated, and only then removed in a later, separate change.
- New enum/union values are handled by clients with an explicit default/unknown case, not an exhaustive switch that breaks on any unrecognized value.
- API versioning (a version in the URL, a header, or a schema evolution strategy) exists and is used deliberately when a change genuinely can't be additive.
- A migration plan and timeline are documented for any deprecation, with visibility into which consumers still depend on the old shape before it's removed.

## Violation signatures

- A field removed or renamed on a shared response/schema in the same change that also changes the logic using it.
- A previously optional field becoming required, breaking any existing caller that didn't send it.
- A numeric or string type narrowed (e.g., `long` to `int`, unbounded string to a fixed enum) without checking whether existing data or callers violate the new constraint.
- An enum consumer with an exhaustive switch and no default/unknown case, guaranteed to break the moment a new value is added upstream.
- A database migration that drops or renames a column in the same deploy that also ships code depending on the old name, with no rollback window.
- "We'll just tell the other team to update" as the entire migration plan, with no compatibility window.

## Code: violation → fix

```java
// Violation: renames a field outright — every existing consumer breaks on deploy
record OrderResponse(String orderId, BigDecimal total) {} // was: `totalAmount`
```

```java
// Fix: expand/contract — add the new name, keep the old one until consumers migrate
record OrderResponse(
    String orderId,
    @Deprecated BigDecimal totalAmount, // kept for one deprecation cycle
    BigDecimal total                    // new name, same value
) {
    static OrderResponse of(String orderId, BigDecimal amount) {
        return new OrderResponse(orderId, amount, amount);
    }
}
```

The fix lets both old and new consumers keep working during the transition — old ones read `totalAmount`, new ones read `total` — and `totalAmount` is removed only in a later, separate change once nothing depends on it, verified by usage metrics rather than assumption.

## Review checklist

1. Does this change remove, rename, or narrow a field/type on a contract with external consumers?
2. Is a newly required field going to break any existing caller that doesn't send it?
3. Does an enum consumer have a default/unknown case, or will a new value break it?
4. If this can't be additive, is there an expand/contract plan with a defined migration window, not a same-change removal?
5. Is there visibility into who still depends on the old shape before it's removed?

## How AI-generated code violates this

A model asked to "rename this field to be clearer" or "fix this response shape" will very often make the change directly and completely, because from inside a single-repo, single-change view, the rename is obviously correct and the old name looks like dead weight — the model typically has no visibility into external consumers that live outside the diff it's editing, so nothing signals that this "cleanup" is actually a breaking change. This is a specific instance of [silent scope creep](../cross-cutting/ai-code-failure-modes.md#silent-scope-creep-into-files-that-werent-part-of-the-task) in reverse: the change is properly scoped to the files it touches, but the *blast radius* extends far outside those files in a way nothing in the diff reveals, which is exactly why this category needs a human who knows the system's actual consumers, not just its code, in the loop before merge.

## Guardrail snippet

```
Never remove, rename, or narrow a field or type on a schema, API, or
message format that has external consumers — add the new shape alongside
the old one and deprecate the old one separately, on its own timeline
(expand/contract). Treat any new required field as a breaking change
unless every existing consumer is confirmed to already send it. Give enum
consumers a default/unknown case so new values don't break them.
```

## Scoring

- **0 — Violated:** a shared contract has a field removed, renamed, or narrowed in a way that breaks existing consumers with no migration path.
- **1 — Partial:** the change is additive but an enum/union consumer isn't defensive against future values, or the deprecation timeline for an old field is undefined.
- **2 — Met:** changes are additive, or a proper expand/contract migration with a defined window is in place for breaking ones.
- **3 — Exemplary:** compatibility is enforced by tooling (schema-compatibility checks in CI, contract tests against real consumers) so a breaking change fails the build before merge.

## Related

- [P-14 Contract & API Design](14-contract-and-api-design.md) — this principle is about *changing* a contract; P-14 is about designing it well in the first place.
- [P-08 Transaction & Data Integrity Boundaries](../crucial/08-transaction-data-integrity.md) — a schema migration that isn't backward compatible during rollout can corrupt data written by old and new code simultaneously.
- [P-21 YAGNI](../medium/21-yagni.md) — in tension at the margin: keeping a deprecated field "just in case" past its migration window is its own form of unnecessary complexity.

## Going deeper

- Fowler, *"Evolutionary Database Design"* and the expand/contract pattern (martinfowler.com).
- Newman, *Building Microservices*, 2nd ed., ch. 5 (Implementing Microservice Communication) — schema evolution, compatibility, and avoiding lockstep releases.
- Google, *"API Design Guide"* (Google Cloud APIs design guide) — resource versioning and compatibility conventions.
