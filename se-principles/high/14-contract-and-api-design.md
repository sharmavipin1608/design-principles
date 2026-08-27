# P-14 · Contract & API Design

> A caller can tell what a method requires and guarantees from its signature alone, without reading the implementation.

**Tier:** High · **Scope:** any public method, interface, or API endpoint — the surface other code depends on

## What it means

A contract is everything a caller needs to know to use a method correctly without reading its body: what inputs are valid, what's guaranteed about the output, what can go wrong and how that's signaled. Good contract design makes as much of this explicit in the type system as possible — nullability encoded in the type (`Optional<T>` vs. `T`), expected failures encoded as checked exceptions or sealed result types, preconditions stated and ideally enforced at the boundary. Interface segregation is part of the same discipline at a coarser grain: an interface should expose only the methods a given caller actually needs, not a wide interface that forces every implementer to support operations most of them don't use.

The test for a good contract is whether a new caller, with no access to the implementation, could use the method correctly on the first try, including its error cases, based on the signature and its documentation.

## Why it matters

A contract that under-specifies its behavior pushes the cost of that ambiguity onto every caller, forever — each one either reads the implementation to find out what actually happens (fragile, since the implementation can change without the signature changing) or gets it wrong and finds out in production. This is a High-tier concern rather than Crucial because a single bad contract rarely causes an incident by itself, but a codebase full of them means every integration is a small research project, and the compounding cost lands hardest exactly when it matters most: onboarding a new team member, integrating a new consumer, or debugging an unexpected null three layers deep in a call chain that no signature warned about.

## What good looks like

- Nullable returns are represented as `Optional<T>`, not a `T` that might be `null` with no signal in the signature.
- Expected failure modes are checked exceptions, sealed result types, or clearly documented specific unchecked exceptions — not a bare `throws Exception` or an undocumented `RuntimeException`.
- Preconditions (non-null args, valid ranges) are validated at the start of the method and documented, not silently assumed.
- Interfaces are narrow and focused — a caller implementing one isn't forced to provide methods it has no meaningful implementation for.
- Javadoc/documentation on a public method states behavior a signature can't express (side effects, thread-safety, ordering guarantees) rather than restating the parameter names.

## Violation signatures

- A method returning `T` that can actually return `null`, with no `Optional` and no clear documentation.
- `throws Exception` on a method signature instead of the specific exception types that can actually occur.
- An interface with a method that most implementers throw `UnsupportedOperationException` from (see also [P-25 Liskov Substitution](../medium/25-liskov-substitution.md)).
- A public method with an undocumented precondition that silently produces wrong results instead of failing when violated.
- A boolean parameter with no named-parameter equivalent, so a call site reads as `process(true, false, true)` with no clue what each flag means.
- A "God interface" with many unrelated methods that no single implementer needs all of.
- A method whose name and signature suggest a pure query but which has a side effect (a `getX()` that also mutates state).

## Code: violation → fix

```java
// Violation: nullability, failure modes, and a mystery boolean are all invisible
User findUser(String id, boolean includeDeleted) throws Exception {
    // returns null if not found; throws a generic Exception on a DB error;
    // includeDeleted's effect isn't discoverable from the call site
    ...
}
```

```java
// Fix: the signature is the documentation
Optional<User> findUser(String id, UserLookupOptions options) throws UserLookupException {
    // Optional makes "not found" explicit; UserLookupException replaces the
    // generic Exception; a named options type replaces the mystery boolean
    ...
}

record UserLookupOptions(boolean includeDeleted) {
    static UserLookupOptions defaults() { return new UserLookupOptions(false); }
}
```

A caller now sees, without reading the implementation, that lookup can fail to find a user (`Optional`), can throw a specific documented failure (`UserLookupException`, not an unbounded `Exception`), and what the extra parameter actually controls (`options.includeDeleted()` at the call site, not a bare `true`).

## Review checklist

1. Can a return value be `null` without the signature saying so (`Optional`, documentation)?
2. Does the method declare `throws Exception`/`Throwable` instead of specific expected failure types?
3. Are there boolean parameters whose meaning isn't clear from the call site?
4. Does an interface force implementers to provide a method most of them can't meaningfully support?
5. Are preconditions validated and documented, or silently assumed?
6. Does a method with a query-sounding name (`getX`, `isX`) have a side effect?

## How AI-generated code violates this

Generated method signatures tend to reflect whatever's easiest to make compile for the immediate task — a nullable return without `Optional`, a broad `throws Exception` — because encoding the full contract into the type system requires the model to anticipate every caller's needs up front, and it's optimizing for this one call site working, not for the interface being good for callers it hasn't seen. Boolean-parameter contracts are a specific tell: when a prompt describes a variant behavior ("also include deleted users"), the shortest addition is a new boolean parameter appended to the existing signature, because it requires no restructuring — even though it immediately makes every existing call site ambiguous. This is also where [hallucinated or drifted assumptions](../cross-cutting/ai-code-failure-modes.md#version-drift--patterns-from-an-older-library-version-applied-to-a-newer-one) about a library's contract show up: a model may assume a method returns `null` on absence because that was the older idiom, when the version actually in use returns `Optional`.

## Guardrail snippet

```
Encode nullability, expected failure modes, and preconditions into method
signatures — use Optional for values that may be absent, specific checked
exceptions or sealed result types for expected failures, never a bare
`throws Exception`. Replace boolean parameters that aren't self-explanatory
at the call site with a named options type or enum. Keep interfaces narrow:
if most implementers can't meaningfully implement a method, split the
interface instead of adding UnsupportedOperationException fallbacks.
```

## Scoring

- **0 — Violated:** a signature hides nullability or failure modes that cause real call-site bugs, or an interface forces unsupported methods on implementers.
- **1 — Partial:** contracts are mostly clear, but at least one method has an ambiguous boolean parameter or an overly broad exception type.
- **2 — Met:** signatures fully communicate nullability, failure modes, and preconditions; interfaces are appropriately narrow.
- **3 — Exemplary:** the contract is enforced by the type system (sealed interfaces, exhaustive pattern matching) so a caller cannot compile code that mishandles a documented case.

## Related

- [P-01 Correctness & Edge Cases](../crucial/01-correctness-and-edge-cases.md) — a contract that makes illegal states unrepresentable prevents whole classes of edge-case bugs before they're written.
- [P-25 Liskov Substitution](../medium/25-liskov-substitution.md) — a subtype that can't honor its supertype's contract is a special case of contract violation at the inheritance level.
- [P-18 Backward Compatibility & Versioning](18-backward-compatibility-versioning.md) — the contract is exactly what a compatibility guarantee promises not to break.

## Going deeper

- Meyer, *Object-Oriented Software Construction*, 2nd ed., ch. 11 — Design by Contract, the origin of pre/postcondition thinking.
- Bloch, *Effective Java*, 3rd ed., Item 54 ("Return empty collections or arrays, not nulls") and Item 55 (`Optional`).
- Martin, *Agile Software Development*, ch. 12 (Interface Segregation Principle).
