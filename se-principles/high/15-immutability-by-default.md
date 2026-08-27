# P-15 · Immutability by Default

> An object doesn't change state after construction unless there's a specific, stated reason it must.

**Tier:** High · **Scope:** value objects, DTOs, configuration, anything passed between threads or layers — default posture for any new type unless mutability is a deliberate requirement

## What it means

Immutability means once an object is constructed, its observable state never changes — instead of mutating an object, you produce a new one with the changed value. This isn't a stylistic preference; it eliminates entire categories of bugs by construction: no aliasing bug where two references to what looks like "the same object" diverge unexpectedly after one caller mutates it, no concurrency bug from unsynchronized shared state, no defensive-copy discipline needed because there's nothing to defend against. Java's `record` types, `final` fields, and unmodifiable collection factories (`List.of`, `Map.copyOf`) make this close to free to default to for anything that represents a value rather than an identity with a changing lifecycle.

The exception isn't "never use mutability" — an entity that genuinely models something changing over time (an in-progress order, a session) needs mutable state. The discipline is that mutability is the exception you justify, not the default you reach for.

## Why it matters

A mutable object shared across two places that each assume they have exclusive control produces the aliasing bug: caller A hands an object to caller B, B mutates it "locally," and A's copy has silently changed too, because it was never a copy. These bugs are notoriously hard to trace because the mutation and the symptom can be far apart in the code and in time — nothing crashes, a value is just quietly wrong somewhere downstream. This connects directly to [P-05 Concurrency](../crucial/05-concurrency-thread-safety.md): most of the effort that goes into locks and synchronized blocks exists specifically to manage mutable shared state, and immutable objects need none of it — they're safe to share across threads with zero additional code, which makes immutability one of the highest-leverage single design choices in this whole guide.

## What good looks like

- Value objects and DTOs are implemented as `record`s or classes with only `final` fields set in the constructor.
- Collections stored in or returned by an object are unmodifiable (`List.copyOf`, `Collections.unmodifiableList`) rather than mutable references.
- "Changing" an object means producing a new instance (a `withX()` method or a new `record` with one field different), not mutating in place.
- Mutability is reserved for objects with a genuine identity and lifecycle (an entity, a builder mid-construction), and that choice is deliberate, not incidental.
- Static/shared fields are either immutable or explicitly synchronized — no mutable `static` field used as ad hoc shared state.

## Violation signatures

- A class representing a value (money, a date range, coordinates, an address) with setters instead of being constructed once.
- A field that's `final` but points to a mutable collection or object, giving no actual immutability guarantee.
- A method that mutates and returns `this` instead of returning a new instance, making call chains order-dependent in surprising ways.
- A mutable `static` field used to share state across requests or threads with no synchronization.
- A "configuration" object that's mutated after being handed to multiple consumers, so consumers can observe different values depending on when they read it.
- Passing a mutable collection into a constructor and storing the reference directly, without copying.

## Code: violation → fix

```java
// Violation: mutable value type — an aliasing bug waiting for a second caller
class DateRange {
    private LocalDate start;
    private LocalDate end;

    void setStart(LocalDate start) { this.start = start; }
    void setEnd(LocalDate end) { this.end = end; }
}

DateRange shared = getDefaultRange();
DateRange q1 = shared;
q1.setEnd(LocalDate.of(2026, 3, 31)); // shared is now silently mutated too
```

```java
// Fix: a record — immutable by construction, no aliasing possible
record DateRange(LocalDate start, LocalDate end) {
    DateRange withEnd(LocalDate newEnd) { return new DateRange(start, newEnd); }
}

DateRange shared = getDefaultRange();
DateRange q1 = shared.withEnd(LocalDate.of(2026, 3, 31)); // a new instance; shared is untouched
```

The fix makes it structurally impossible for one caller's change to leak into another's reference — `withEnd` returns a new object, so `shared` and `q1` can never silently diverge from what either caller expects.

## Review checklist

1. Does a new type representing a value (not an entity with identity) have setters instead of being immutable?
2. Does a `final` field point to a mutable collection or object with no defensive copy?
3. Is any method mutating `this` and returning it, instead of returning a new instance?
4. Is a mutable `static` field used as shared state with no synchronization?
5. Is a mutable object passed into a constructor and stored by reference without copying?

## How AI-generated code violates this

Generated Java code frequently defaults to the classic getter/setter JavaBean shape for any new class, because it's the most heavily represented pattern for "a class with fields" in training data, independent of whether the type actually needs mutability — a model asked for "an Address class" is more likely to produce setters than a `record`, even though modern idiomatic Java (which the model has also seen extensively) would default to the latter for a pure value type. This is a case of pattern frequency winning over contextual fit, and it's worth flagging explicitly in review because the JavaBean shape isn't wrong syntactically — it compiles, it works in the one example tested — it just reintroduces an aliasing/concurrency risk the alternative would have avoided for free.

## Guardrail snippet

```
Default new types that represent a value (money, dates, coordinates,
DTOs, config) to Java records or final-field classes with no setters.
Reserve mutable classes for objects with genuine identity and a changing
lifecycle, and justify that choice explicitly. Store collections as
unmodifiable copies, never as a live mutable reference. To "change" a
value object, produce a new instance rather than mutating in place.
```

## Scoring

- **0 — Violated:** a value type is mutable and this causes or risks an aliasing bug (shared reference mutated unexpectedly).
- **1 — Partial:** most value types are immutable, but at least one still exposes setters or an unprotected mutable field.
- **2 — Met:** value types are immutable by default; mutability is reserved for genuine entities and is deliberate.
- **3 — Exemplary:** immutability is used as a concurrency strategy explicitly — shared state across threads is immutable specifically to eliminate the need for locking, not just as a style choice.

## Related

- [P-05 Concurrency & Thread Safety](../crucial/05-concurrency-thread-safety.md) — immutable objects need no synchronization at all; this is the cheapest fix for many concurrency problems.
- [P-13 Encapsulation & Information Hiding](13-encapsulation-information-hiding.md) — a leaked mutable reference undermines both encapsulation and immutability simultaneously.
- [P-24 Composition over Inheritance](../medium/24-composition-over-inheritance.md) — immutable value objects compose cleanly; mutable ones often don't.

## Going deeper

- Bloch, *Effective Java*, 3rd ed., Item 17 ("Minimize mutability").
- Goetz et al., *Java Concurrency in Practice*, ch. 3.4 — immutable objects and safe publication.
- Evans, *Domain-Driven Design*, ch. 5 — value objects vs. entities as a modeling distinction that maps directly onto this principle.
