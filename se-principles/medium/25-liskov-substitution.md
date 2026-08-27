# P-25 · Liskov Substitution

> A subtype must be usable anywhere its supertype is expected, without the caller needing to know which one it actually got.

**Tier:** Medium · **Scope:** any inheritance hierarchy or interface implementation — the check for whether "is-a" is actually true, not just declared

## What it means

Liskov Substitution says a subtype has to honor its supertype's contract completely — it can't strengthen preconditions (demanding more than the supertype required), weaken postconditions (guaranteeing less than the supertype promised), or throw new exception types the caller wasn't told to expect. The classic teaching example is a `Square` extending `Rectangle`: mathematically a square is a rectangle, but if `Rectangle` has independent `setWidth`/`setHeight` methods, a `Square` that keeps both sides equal breaks the contract callers rely on — calling `setWidth` on what they hold as a `Rectangle` unexpectedly changes the height too. The type system said "is-a" was fine; the behavior said otherwise. This is the practical test for whether inheritance (or an interface implementation) is actually sound: can every caller that only knows about the supertype use any subtype interchangeably, with no surprises and no need to check which concrete type it has.

## Why it matters

A Liskov violation is a trap that's invisible at the type level and only detonates at runtime, in whichever caller happens to exercise the specific behavior that diverges — and because the type system approved the substitution, nothing catches it until a test (or production) actually calls the method that behaves differently than the contract promised. `UnsupportedOperationException` thrown from an override is the sharpest version of this: the type system says the subtype supports the operation, and the runtime says it doesn't, which means every caller now has to defensively check the concrete type before calling a method the interface told them was safe to call — exactly the kind of type-check sprawl polymorphism was supposed to eliminate.

## What good looks like

- Every override honors the same preconditions (or weaker) and same postconditions (or stronger) as the method it overrides — never the reverse.
- No override throws a checked or unchecked exception type the caller wasn't already told to expect from the supertype's contract.
- No implementation of an interface method throws `UnsupportedOperationException` (or an equivalent "not implemented") for a method the interface declares as part of its contract.
- A caller working only against the supertype/interface type never needs an `instanceof` check to use a subtype correctly.
- Subtype behavior is a specialization, not a contradiction, of the supertype's documented behavior.

## Violation signatures

- `throw new UnsupportedOperationException()` inside an interface method implementation or an override.
- An override that adds a stricter precondition than the method it overrides (e.g., rejecting `null` where the parent allowed it).
- An override that returns a narrower or different-shaped result than callers of the supertype's contract expect.
- Caller code with an `instanceof` check specifically to decide whether it's "safe" to call a particular method on an object typed as the interface/supertype.
- A subtype that silently no-ops a method the supertype's contract says has an effect.
- Collection implementations (e.g., an "immutable list" extending a mutable `List` interface) that throw on mutation methods the interface declares as supported.

## Code: violation → fix

```java
// Violation: ReadOnlyRepository "is-a" Repository, but breaks the contract at runtime
interface Repository<T> {
    T save(T entity);
    Optional<T> findById(String id);
}

class ReadOnlyRepository<T> implements Repository<T> {
    public T save(T entity) {
        throw new UnsupportedOperationException("read-only"); // silently violates the interface
    }
    public Optional<T> findById(String id) { /* ... */ return Optional.empty(); }
}
```

```java
// Fix: split the interface so the contract actually matches what's substitutable
interface ReadableRepository<T> {
    Optional<T> findById(String id);
}

interface WritableRepository<T> extends ReadableRepository<T> {
    T save(T entity);
}

class ReadOnlyRepository<T> implements ReadableRepository<T> {
    public Optional<T> findById(String id) { /* ... */ return Optional.empty(); }
    // no save() to fail to honor — the type itself says it can't be written to
}
```

The fix makes the type system tell the truth: a caller holding a `ReadableRepository` never has a `save` method to call in the first place, so there's no contract to violate at runtime — the interface segregation from [P-14](../high/14-contract-and-api-design.md) is what makes the substitution honest.

## Review checklist

1. Does any override or implementation throw `UnsupportedOperationException` or similar for a method its interface/supertype declares?
2. Does an override require a stronger precondition than its parent method did?
3. Does an override deliver a weaker guarantee (a narrower postcondition) than callers of the supertype expect?
4. Is there caller code that uses `instanceof` to decide whether a method is safe to call on a supertype-typed reference?
5. Would a caller be surprised by this subtype's behavior if it only knew about the supertype's documented contract?

## How AI-generated code violates this

When a model is asked to implement an interface for a case that only partially fits it — a read-only data source implementing a read/write repository interface, because that's the interface already in the codebase — reaching for `UnsupportedOperationException` on the unsupported methods is a common, syntactically valid way to make the code compile without restructuring the interface, which is the smaller, more locally contained edit from the model's perspective. This is a specific case of the model correctly avoiding [speculative abstraction](../cross-cutting/ai-code-failure-modes.md#speculative-abstraction--layers-invented-for-a-single-caller) (it doesn't invent a new interface) while producing a worse problem instead — the fix (interface segregation, per [P-14](../high/14-contract-and-api-design.md)) requires recognizing that the *existing* interface is wrong for this case, a judgment call that's easy to defer in favor of the code that compiles today.

## Guardrail snippet

```
Never implement an interface method by throwing UnsupportedOperationException
or a similar "not implemented" error — if a type can't honor part of an
interface's contract, split the interface instead of implementing it
partially. Never strengthen a precondition or weaken a postcondition in an
override relative to the method it overrides. If calling code needs an
instanceof check to safely call a method, the substitution isn't sound.
```

## Scoring

- **0 — Violated:** an override or implementation throws for a method the interface declares, or strengthens a precondition callers aren't expecting.
- **1 — Partial:** the hierarchy mostly holds, but one implementation needs a defensive instanceof check somewhere in calling code.
- **2 — Met:** every subtype is fully substitutable for its supertype with no surprises; interfaces are segregated where a full implementation isn't possible.
- **3 — Exemplary:** substitutability is verified by shared contract tests run against every implementation, so a future violation fails a test rather than surfacing at runtime.

## Related

- [P-24 Composition over Inheritance](24-composition-over-inheritance.md) — this principle is the litmus test for whether an inheritance relationship being considered is actually sound.
- [P-14 Contract & API Design](../high/14-contract-and-api-design.md) — interface segregation is frequently the actual fix for a Liskov violation, not a workaround inside the failing implementation.
- [P-16 Open/Closed](../high/16-open-closed.md) — an extension mechanism only stays safe if every new implementation genuinely substitutes for the abstraction it extends.

## Going deeper

- Liskov & Wing, *"A Behavioral Notion of Subtyping"* — the original formal paper defining the substitution principle.
- Martin, *Agile Software Development, Principles, Patterns, and Practices*, ch. 10 (The Liskov Substitution Principle).
- Meyer, *Object-Oriented Software Construction*, 2nd ed., ch. 11 — pre/postcondition rules for redefinition in inheritance.
