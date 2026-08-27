# P-11 · Coupling & Cohesion

> Modules depend on as little outside themselves as possible, and everything inside a module belongs together.

**Tier:** High · **Scope:** module, package, and service boundaries — anywhere two units of code know about each other's internals or unrelated responsibilities sit side by side

## What it means

Coupling is how much a module needs to know about another to work; cohesion is how much what's inside a module actually belongs together. You want low coupling — a module should depend on a narrow, stable interface, not another module's internal structure or its transitive dependencies — and high cohesion, where everything inside a module changes for the same reasons and serves the same purpose. These pull in the same direction more than they're independent axes: a poorly cohesive module (grab-bag responsibilities) tends to accumulate high coupling too, because unrelated responsibilities each drag in their own unrelated dependencies.

The practical tell for excess coupling is "shotgun surgery" — a single conceptual change requires edits across many files/modules because the responsibility is scattered — and "feature envy" — a method that spends more time reaching into another object's data than using its own.

## Why it matters

Tightly coupled modules turn a local change into a system-wide one: fixing a bug in one place requires understanding and touching several others, and every touched module is a chance to introduce a new bug in something that wasn't supposed to change. This is what makes coupling a High-tier, not Medium-tier, concern — it doesn't ship an immediate incident the way a Crucial violation does, but it compounds silently, and by the time it's visible (a team afraid to touch a module because nobody can predict the blast radius of a change) the cost to unwind it is a multi-sprint untangling effort, not a code review comment.

## What good looks like

- A module exposes a narrow interface and hides how it accomplishes its work; callers depend on that interface, not on internal fields or helper methods.
- A single business change (add a field, adjust a rule) touches one module, not a scattered set of files.
- Methods primarily operate on their own class's data; reaching into another object's internals to do work is rare and deliberate.
- Dependencies point in one direction consistently (e.g., domain doesn't depend on infrastructure) rather than forming cycles.
- A module can be tested in isolation with a small number of mocked collaborators, not a deep dependency graph.

## Violation signatures

- A single logical change requires edits across three or more unrelated files/modules (shotgun surgery).
- A method that calls more getters on a parameter than it calls on `this` (feature envy).
- Two modules importing each other, directly or transitively (a dependency cycle).
- A class with unrelated method groups that never call each other (low cohesion — it's really two or three classes wearing one name).
- A "utility" or "common" module imported by nearly everything, becoming a de facto coupling hub.
- Business logic reaching directly into another module's internal data structure instead of calling its public method.

## Code: violation → fix

```java
// Violation: OrderProcessor reaches deep into Customer's internals — feature envy
class OrderProcessor {
    BigDecimal computeShippingDiscount(Order order) {
        Customer c = order.getCustomer();
        if (c.getLoyaltyProgram().getTier().getLevel() >= 3
                && c.getLoyaltyProgram().getPointsBalance() > 500) {
            return new BigDecimal("0.15");
        }
        return BigDecimal.ZERO;
    }
}
```

```java
// Fix: the decision moves to where the data lives; OrderProcessor just asks
class Customer {
    boolean qualifiesForShippingDiscount() {
        return loyaltyProgram.getTier().getLevel() >= 3
            && loyaltyProgram.getPointsBalance() > 500;
    }
}

class OrderProcessor {
    BigDecimal computeShippingDiscount(Order order) {
        return order.getCustomer().qualifiesForShippingDiscount()
            ? new BigDecimal("0.15") : BigDecimal.ZERO;
    }
}
```

`OrderProcessor` no longer needs to know `Customer` has a `LoyaltyProgram` at all — the eligibility rule moved next to the data it depends on, and a future change to loyalty tiers touches one class instead of every caller that inspected it.

## Review checklist

1. Does this change require edits across multiple unrelated modules for one conceptual change?
2. Does any method call more accessors on a parameter/collaborator than on its own state?
3. Does this introduce a new dependency cycle between modules?
4. Could this module be unit tested with only one or two mocked collaborators?
5. Does everything in this class actually serve the same responsibility, or are there two unrelated method groups?

## How AI-generated code violates this

A model generating one function at a time optimizes for that function working, and the shortest path to a working result is often to reach directly into whatever object already has the needed data, rather than asking whether that data's owner should expose a method instead — the coupling cost is invisible in a single-function view. Across a longer agentic session, this compounds with [inconsistency across turns](../cross-cutting/ai-code-failure-modes.md#inconsistency-across-turns--the-same-concept-implemented-two-different-ways-in-one-session): a concern that should live in one cohesive module ends up implemented piecemeal across files the agent touched in sequence, each edit locally reasonable, the aggregate low-cohesion.

## Guardrail snippet

```
Before reaching into another object's data across more than one level
(a.getB().getC()), check whether the owning object should expose a method
that does the work instead. Keep dependencies pointing in one direction —
never introduce a cycle between modules. If a single change requires
editing more than two unrelated files, stop and check whether the
responsibility should be consolidated into one module first.
```

## Scoring

- **0 — Violated:** a dependency cycle exists, or a single conceptual change requires edits across many unrelated modules.
- **1 — Partial:** coupling is mostly reasonable, but at least one clear feature-envy method or low-cohesion class exists.
- **2 — Met:** modules expose narrow interfaces, dependencies are acyclic, changes stay localized.
- **3 — Exemplary:** module boundaries are enforced by build-level rules (module system, architecture tests) so a coupling violation fails the build, not just the review.

## Related

- [P-09 Separation of Concerns / SRP](../crucial/09-separation-of-concerns-srp.md) — SRP is cohesion applied at the class/method level; this principle is the same idea scaled to modules.
- [P-12 Dependency Inversion & Injection](12-dependency-inversion.md) — depending on abstractions rather than concretions is one of the main tools for reducing coupling.
- [P-26 Law of Demeter](../medium/26-law-of-demeter.md) — the specific "don't reach through objects" discipline that prevents feature envy at the call-site level.

## Going deeper

- Martin, *Clean Architecture*, ch. 13–14 (component coupling and cohesion principles).
- Fowler, *Refactoring*, 2nd ed. — "Feature Envy" and "Shotgun Surgery" as named smells with corresponding refactorings.
- Parnas, *"On the Criteria to Be Used in Decomposing Systems into Modules"* — the foundational paper on information hiding as a module-boundary principle.
