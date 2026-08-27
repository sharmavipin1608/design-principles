# P-16 · Open/Closed

> New behavior gets added by extension, without editing code that already works.

**Tier:** High · **Scope:** anywhere behavior varies by type or case and is likely to grow a new variant — payment methods, notification channels, pricing rules, file format handlers

## What it means

A module is open for extension and closed for modification when you can add a new case — a new payment provider, a new discount type, a new export format — without touching the code that handles the existing cases. The mechanism is almost always some form of polymorphism: a strategy interface, a plugin registry, a `sealed` hierarchy with exhaustive handling, rather than a single method with a growing conditional that branches on type. The signal that a design has stopped being open/closed is a switch or if/else-if chain on a type discriminator that every new case requires editing — the exact code that handled every previous case correctly is now back in play for review and regression risk on every future addition.

This principle is explicitly in tension with [KISS](../medium/22-kiss-complexity-budget.md): building an extension point for a single current case is speculative complexity, not open/closed design. The principle earns its keep when new cases are a *known, recurring* pattern of change, not a hypothetical one.

## Why it matters

A type-switch that grows with every new case means every addition is a modification to shared, already-tested code — the exact code path every existing case depends on is now edited again, and a mistake in the new branch, or in how it interacts with the shared logic around the switch, can regress a case that had nothing to do with the change. This is High-tier because the damage compounds specifically with scale: the tenth payment provider added to a growing if/else chain is riskier to add than the second was, because the surrounding logic has accreted assumptions from nine prior additions that the new author has to hold in their head simultaneously.

## What good looks like

- Adding a new case (a new type, a new provider, a new format) means adding a new class/implementation, not editing an existing method's branches.
- A `sealed` interface with pattern matching is used where the compiler should enforce that every case is handled — not an open-ended `if`/`else` chain with no exhaustiveness check.
- A registry or strategy map dispatches to the right implementation by key/type, so adding a case is a registration, not a code edit inside the dispatcher.
- Shared logic that every case needs lives in the abstraction (a default method, a template method) rather than being copy-pasted per case.

## Violation signatures

- An `if (type == X) ... else if (type == Y) ... else if (type == Z)` chain that's grown past two or three cases and keeps growing.
- A `switch` on a type/enum with a `default` that throws, meaning the compiler doesn't catch a missing case.
- A method whose diff, across the file's history, is dominated by additions of new branches for new cases rather than changes to existing logic.
- Business logic that has to know about every implementation of an interface to decide which one to use, instead of a registry/factory doing that.
- Copy-pasted logic across several "case" implementations that should share a common base or default.

## Code: violation → fix

```java
// Violation: adding provider #4 means editing this method again
BigDecimal calculateFee(String provider, BigDecimal amount) {
    if (provider.equals("stripe")) return amount.multiply(new BigDecimal("0.029"));
    else if (provider.equals("paypal")) return amount.multiply(new BigDecimal("0.034"));
    else if (provider.equals("square")) return amount.multiply(new BigDecimal("0.026"));
    else throw new IllegalArgumentException("unknown provider: " + provider);
}
```

```java
// Fix: adding a provider means adding a class — this method never changes again
sealed interface FeeStrategy permits StripeFee, PayPalFee, SquareFee {
    BigDecimal calculate(BigDecimal amount);
}

record StripeFee() implements FeeStrategy {
    public BigDecimal calculate(BigDecimal amount) { return amount.multiply(new BigDecimal("0.029")); }
}
// PayPalFee, SquareFee follow the same shape

BigDecimal calculateFee(FeeStrategy strategy, BigDecimal amount) {
    return strategy.calculate(amount); // closed for modification, permanently
}
```

Adding `SquareFee`, or a fourth provider, is now a new file implementing `FeeStrategy` — `calculateFee` never needs to be touched, reviewed, or regression-tested again for that change, and the `sealed` interface means the compiler flags any switch elsewhere that forgot to handle the new case.

## Review checklist

1. Does adding the next expected case require editing an existing method's branches, or adding a new implementation?
2. Is there a growing if/else-if or switch on a type discriminator with no exhaustiveness enforcement?
3. Would a `sealed` type with pattern matching catch a missed case at compile time here?
4. Is shared per-case logic duplicated across implementations that should share a base or default method?
5. Does business logic need to know about every implementation, or does a registry/factory abstract that away?

## How AI-generated code violates this

When asked to "add support for a new provider/type," a model working incrementally on an existing file will very often extend the existing conditional in place — it's the smallest, most locally coherent diff for the stated task, and recognizing that the *pattern itself* has outgrown its shape requires stepping back from the immediate request to consider the whole method's trajectory, which a single-turn edit doesn't naturally do. This is a case where the AI-specific risk is under-refactoring, not over-abstracting: unlike [speculative abstraction](../cross-cutting/ai-code-failure-modes.md#speculative-abstraction--layers-invented-for-a-single-caller), which invents unnecessary structure for one caller, this failure mode is the model correctly avoiding new abstraction for what looks like "just one more case" — repeated across several sessions until the conditional is nine cases deep and nobody made the call to convert it.

## Guardrail snippet

```
Before adding a new case to an existing if/else-if chain or switch on a
type discriminator, check how many cases it already has. At three or
more, convert it to a sealed interface with one implementation per case
(or a registry/strategy map) instead of adding another branch. Use sealed
types with exhaustive pattern matching wherever the compiler can enforce
that every case is handled.
```

## Scoring

- **0 — Violated:** adding the next expected case requires editing shared, already-tested branching logic, with no exhaustiveness check.
- **1 — Partial:** an extension point exists, but at least one case still requires touching shared logic beyond registration.
- **2 — Met:** new cases are added purely by extension (new class/registration); existing dispatch logic is untouched and exhaustiveness is enforced.
- **3 — Exemplary:** the extension mechanism is discoverable and self-registering (a plugin system, a `sealed` hierarchy with compiler-enforced exhaustiveness) so misuse is caught before runtime.

## Related

- [P-22 KISS / Complexity Budget](../medium/22-kiss-complexity-budget.md) — in tension: building an extension point before a second case exists is speculative complexity; this principle only pays off once variation is a real, recurring pattern.
- [P-25 Liskov Substitution](../medium/25-liskov-substitution.md) — an extension point only stays safe if every new implementation actually honors the shared interface's contract.
- [P-09 Separation of Concerns / SRP](../crucial/09-separation-of-concerns-srp.md) — a growing type-switch is usually a sign the dispatch responsibility and the per-case logic haven't been separated.

## Going deeper

- Martin, *Agile Software Development, Principles, Patterns, and Practices*, ch. 9 (The Open-Closed Principle).
- Gamma, Helm, Johnson, Vlissides, *Design Patterns*, Strategy and Template Method chapters.
- Meyer, *Object-Oriented Software Construction*, 2nd ed., ch. 2 — the origin of the open/closed formulation.
