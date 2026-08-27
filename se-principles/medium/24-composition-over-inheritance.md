# P-24 · Composition over Inheritance

> Prefer assembling behavior from independent parts over extending a base class to get it.

**Tier:** Medium · **Scope:** any time reuse is achieved by `extends` rather than by injecting a collaborator — especially hierarchies more than one level deep

## What it means

Inheritance couples a subclass to its superclass's implementation, not just its interface — a subclass inherits every field, every method, and every future change the superclass makes, whether or not that change was intended to affect subclasses. Composition achieves the same reuse by having an object hold a reference to another object and delegate to it, which keeps the two independently changeable: the "has-a" relationship can be swapped, mocked, or reconfigured at runtime in a way "is-a" cannot. This isn't a blanket rejection of inheritance — a shallow, well-designed hierarchy modeling a genuine is-a relationship with a stable contract is fine — it's a default preference: reach for composition first, and justify inheritance specifically when the is-a relationship is real, stable, and the subclass genuinely needs to be substitutable everywhere the superclass is used (see [P-25 Liskov Substitution](25-liskov-substitution.md)).

## Why it matters

The classic failure this principle prevents is the fragile base class problem: a change to a superclass, made for reasons entirely unrelated to a particular subclass, silently changes that subclass's behavior because it inherited a method or field the change touched. This gets worse with hierarchy depth — a change three levels up a chain can have effects at the leaf that are nearly impossible to predict by reading any single class in isolation, because understanding a leaf class requires understanding its entire ancestor chain. Composition doesn't have this problem structurally: a composed collaborator's behavior is fixed at the interface it exposes, and changing its internals can't silently ripple into the composing object's behavior the way a superclass change can ripple into a subclass.

## What good looks like

- Reuse is achieved by injecting a collaborator and delegating to it, not by extending a class to inherit its methods.
- Inheritance hierarchies, where they exist, are shallow (one level, rarely two) and model a genuine, stable is-a relationship.
- A class needing several independent behaviors (logging, validation, caching) composes them as separate collaborators rather than inheriting from a base class that bundles all three.
- Behavior can be swapped at runtime by injecting a different collaborator, without subclassing.
- A subclass, where used, doesn't override a method just to disable or contradict what the superclass does (a sign the relationship isn't really is-a).

## Violation signatures

- An inheritance hierarchy three or more levels deep.
- A subclass that overrides a parent method to throw, return early, or otherwise cancel the parent's behavior rather than extend it.
- A base class created purely to share a few utility methods among unrelated subclasses that don't share a real conceptual relationship.
- A class extending a framework/library base class primarily to reuse its implementation, when composing with an instance of it would work as well.
- Diamond-shaped hierarchies or multiple levels of abstract classes each adding one small piece of behavior.

## Code: violation → fix

```java
// Violation: inherits from a base class just to reuse retry logic;
// couples ReportGenerator to everything RetryableTask does and might change
abstract class RetryableTask {
    protected int maxRetries = 3;
    protected void executeWithRetry(Runnable action) { /* retry logic */ }
}

class ReportGenerator extends RetryableTask {
    void generate() { executeWithRetry(this::buildReport); }
}
```

```java
// Fix: retry logic is a collaborator, not an ancestor — ReportGenerator
// depends on its interface, and either can change independently
class RetryExecutor {
    private final int maxRetries;
    RetryExecutor(int maxRetries) { this.maxRetries = maxRetries; }
    void executeWithRetry(Runnable action) { /* retry logic */ }
}

class ReportGenerator {
    private final RetryExecutor retryExecutor;
    ReportGenerator(RetryExecutor retryExecutor) { this.retryExecutor = retryExecutor; }
    void generate() { retryExecutor.executeWithRetry(this::buildReport); }
}
```

The fix removes the inheritance coupling entirely — `ReportGenerator` no longer inherits fields or future behavior changes from a class it has no real is-a relationship with, and `RetryExecutor` can be reused by any class regardless of its own hierarchy.

## Review checklist

1. Is inheritance being used here to reuse implementation, or to model a genuine, stable is-a relationship?
2. Is the hierarchy involved more than one or two levels deep?
3. Does a subclass override a method to cancel or contradict the parent's behavior?
4. Would injecting a collaborator and delegating to it work just as well as extending a class?
5. Is a base class shared by subclasses that don't actually have much in common conceptually?

## How AI-generated code violates this

When a model is asked to "add a variant of X that also does Y," extending the existing class is often the most locally coherent, smallest-diff way to satisfy the request — inheritance requires no restructuring of the original class, whereas composition requires refactoring the original to accept an injected collaborator, which is more invasive to a change the model is trying to keep minimal. This produces hierarchies that grow one `extends` at a time, each individually reasonable as the smallest fix for its own prompt, converging on a deep, fragile hierarchy nobody deliberately designed — a variant of [inconsistency across turns](../cross-cutting/ai-code-failure-modes.md#inconsistency-across-turns--the-same-concept-implemented-two-different-ways-in-one-session) where the *design decision itself* (favor composition) isn't reliably remembered from one generation to the next without being restated.

## Guardrail snippet

```
Default to composition: inject collaborators and delegate to them rather
than extending a class to reuse its behavior. Reserve inheritance for a
genuine, stable is-a relationship where the subclass must be fully
substitutable for the superclass everywhere it's used. Keep any hierarchy
that does exist to one level. Never subclass purely to reuse a few
utility methods.
```

## Scoring

- **0 — Violated:** inheritance is used purely for implementation reuse with no genuine is-a relationship, or a subclass overrides a method to cancel the parent's behavior.
- **1 — Partial:** the hierarchy is reasonable but deeper than necessary, or one case of reuse-via-inheritance exists where composition would fit better.
- **2 — Met:** reuse is achieved via composition; any inheritance present models a genuine, shallow is-a relationship.
- **3 — Exemplary:** behavior is fully assembled from small, independently testable, composed collaborators, with no inheritance hierarchy deeper than a single level anywhere in the affected code.

## Related

- [P-25 Liskov Substitution](25-liskov-substitution.md) — a prerequisite check for when inheritance is actually appropriate: if the subclass can't honor the supertype's contract, it shouldn't be a subclass at all.
- [P-12 Dependency Inversion & Injection](../high/12-dependency-inversion.md) — composition depends on the same injection discipline this High-tier principle establishes.
- [P-16 Open/Closed](../high/16-open-closed.md) — composed strategies are usually a cleaner extension mechanism than growing a subclass hierarchy.

## Going deeper

- Gamma, Helm, Johnson, Vlissides, *Design Patterns*, ch. 1 — "Favor object composition over class inheritance," stated as a core principle of the book.
- Bloch, *Effective Java*, 3rd ed., Item 18 ("Favor composition over inheritance").
- Snyder, *"Encapsulation and Inheritance in Object-Oriented Programming Languages"* — early formal treatment of the fragile base class problem.
