# P-13 · Encapsulation & Information Hiding

> Internal state and representation stay hidden; only behavior is exposed.

**Tier:** High · **Scope:** any class with state — especially collections, mutable fields, and anything with invariants that must hold across its lifetime

## What it means

Encapsulation means an object controls access to its own state — callers interact with it through methods that preserve its invariants, not by reaching in and mutating fields directly. Information hiding is the design intent behind it: expose the smallest interface that lets callers do what they need, and hide everything about *how* the object accomplishes it, so that internal representation can change without breaking every caller. The most common way this breaks in practice isn't a public field — it's a getter that returns a live, mutable reference to internal state (a `List`, a `Map`, a mutable date), which lets a caller bypass every invariant the class was trying to enforce, using an API that looks perfectly encapsulated on the surface.

## Why it matters

A leaked mutable reference means the class's invariants are only as strong as every caller's discipline, forever — and that discipline degrades the moment a new caller, written by someone (or something) that never read the class's intent, gets a handle on the internal list and appends to it directly, silently invalidating whatever the class assumed about its own state. The bug that results is usually far from the leak: a class asserts "orders never exceeds 10 items," a caller three modules away mutates the returned list to 11, and the failure shows up as a downstream invariant violation that has nothing obviously to do with the getter that leaked it.

## What good looks like

- Getters that return collections return an unmodifiable view or a defensive copy, never the live internal collection.
- Fields are private by default; visibility is widened only with a specific reason, not habitually.
- Mutation happens through methods that can enforce invariants (`addItem(x)` that checks a limit), not through direct field or collection access.
- Constructors validate and defensively copy mutable arguments before storing them, so external mutation of the original doesn't affect the object's internal state.
- A class's public surface reveals *what* it does, not *how* — no getter exists purely to let another class do the class's own job for it.

## Violation signatures

- A getter returning a live `List`, `Map`, or `Set` field directly instead of `Collections.unmodifiableList(...)` or a copy.
- A public mutable field (`public List<Item> items;`) instead of a private field with controlled access.
- A constructor that stores a passed-in mutable object (a `Date`, a collection) by reference without copying it.
- A class with more getters than behavior methods — an "anemic" object that's really just a bag of exposed data other classes act on.
- `setX()` methods for every field, with no validation, effectively re-exposing the field as writable from anywhere.
- A `final` field pointing to a mutable object, where "final" is doing no actual work because the object it points to can still be mutated externally.

## Code: violation → fix

```java
// Violation: leaks the live collection; any caller can corrupt the invariant
class ShoppingCart {
    private final List<Item> items = new ArrayList<>();

    List<Item> getItems() { return items; } // caller can items.clear(), items.add(), ...

    boolean isValid() { return items.size() <= 10; } // invariant nobody outside can trust
}
```

```java
// Fix: mutation only through methods that can enforce the invariant
class ShoppingCart {
    private final List<Item> items = new ArrayList<>();

    List<Item> getItems() { return List.copyOf(items); } // read-only snapshot

    void addItem(Item item) {
        if (items.size() >= 10) throw new CartLimitExceededException();
        items.add(item); // the only path that can grow the list, and it's guarded
    }
}
```

The fix means `isValid()`'s guarantee actually holds — no caller anywhere in the codebase has a path to append an eleventh item without going through the check.

## Review checklist

1. Does any getter return a live mutable collection or object instead of a copy or unmodifiable view?
2. Are fields private, widened only where there's a specific reason?
3. Does the constructor defensively copy mutable arguments before storing them?
4. Is there a class with many getters/setters and little behavior — an anemic object other code manipulates directly?
5. Does a `final` field still allow external mutation of the object it points to?

## How AI-generated code violates this

Generated data classes very often default to a getter that returns the field directly — `return this.items;` — because it's the shortest, most obviously "correct" implementation of "add a getter for items," and the defensive-copy step requires anticipating a misuse (external mutation) that isn't implied by the immediate task. This tends to surface specifically in generated DTOs and entity classes, where the model produces boilerplate getters/setters for every field uniformly, without differentiating fields that need protection (collections, dates, nested mutable objects) from ones that don't (a `String`, a primitive) — a plausible-looking pattern applied indiscriminately, which is exactly the [plausible-but-wrong logic](../cross-cutting/ai-code-failure-modes.md#plausible-but-wrong-logic-that-reads-correctly) failure mode: nothing about a leaky getter looks wrong on inspection.

## Guardrail snippet

```
Never return a live mutable field (List, Map, Set, Date, or similar)
directly from a getter — return an unmodifiable view or a defensive copy.
Defensively copy mutable constructor arguments before storing them.
Prefer exposing behavior methods that enforce invariants over exposing raw
field access via getters/setters for every field uniformly.
```

## Scoring

- **0 — Violated:** a getter leaks a live mutable collection/object that lets a caller bypass a stated invariant.
- **1 — Partial:** most state is protected, but at least one getter or constructor leaks a mutable reference.
- **2 — Met:** all mutable internal state is protected by copies/unmodifiable views; invariants can't be bypassed externally.
- **3 — Exemplary:** the class is designed so invariant-breaking states are unrepresentable (immutable value objects, builder validation) rather than merely guarded at each access point.

## Related

- [P-15 Immutability by Default](15-immutability-by-default.md) — the strongest form of encapsulation is removing mutability entirely rather than guarding access to it.
- [P-03 Security Fundamentals](../crucial/03-security-fundamentals.md) — authorization is encapsulation enforced across a network boundary instead of a class boundary.
- [P-14 Contract & API Design](14-contract-and-api-design.md) — a leaked mutable reference is an unstated part of the contract that the signature doesn't reveal.

## Going deeper

- Bloch, *Effective Java*, 3rd ed., Item 15 ("Minimize the accessibility of classes and members") and Item 50 ("Make defensive copies when needed").
- Fowler, *Refactoring*, 2nd ed. — "Data Class" smell and encapsulation-restoring refactorings.
- Parnas, *"On the Criteria to Be Used in Decomposing Systems into Modules"* — the original information-hiding argument.
