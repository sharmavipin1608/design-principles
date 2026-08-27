# P-12 · Dependency Inversion & Injection

> Business logic depends on abstractions supplied from outside — it never constructs the concrete things it needs itself.

**Tier:** High · **Scope:** any class that needs a collaborator (a repository, a client, a clock, a config source) to do its job

## What it means

Dependency inversion says high-level policy shouldn't depend on low-level detail — a pricing service shouldn't know it's talking to Postgres, only that it's talking to something that satisfies a repository interface. Dependency injection is the mechanism: the object doesn't build its own collaborators with `new`, it receives them from outside, usually through a constructor. The two together mean the direction of dependency runs from concrete to abstract, not the other way around, and it's what makes a class replaceable, mockable, and reusable across contexts (production, test, a different backend) without editing its internals.

The tell that this is missing isn't philosophical — it's mechanical: if you can't unit test a class without spinning up a real database, HTTP client, or file system, it's because the class built its own concrete dependency instead of accepting one.

## Why it matters

A class that constructs its own dependencies is welded to whatever it constructed — swapping an implementation, injecting a test double, or reusing the class in a new context all require editing the class itself, which is exactly the coupling this guide is trying to keep out of business logic. The compounding cost shows up first in test suites: without injection, "unit" tests become integration tests that need real infrastructure, which makes them slow and flaky, which then makes teams skip writing them — turning a design problem into a test-coverage gap, which is a Crucial-tier consequence traced back to a High-tier cause.

## What good looks like

- Collaborators are passed into a constructor (or factory), not instantiated inside business-logic methods with `new`.
- A class's dependencies are visible in its constructor signature — you can tell what it needs without reading the method bodies.
- Business logic depends on an interface or abstract type, and the concrete implementation is wired up in one place (a DI container, a factory, a `main`/composition-root method).
- A class can be unit tested by passing in a fake/mock implementation of every dependency, with no real I/O.
- Cross-cutting concerns (time, randomness, config) are also injected (`Clock`, a config object) rather than called statically (`Instant.now()`, `System.getenv()`) inside logic that needs to be deterministic in tests.

## Violation signatures

- `new SomeRepository()`, `new HttpClient()`, or similar inside a method that also contains business logic.
- A class with a no-arg constructor that internally wires up its own real dependencies.
- Static method calls to infrastructure (`Instant.now()`, `System.getenv()`, a static DB connection holder) scattered through business logic instead of injected.
- A unit test that needs a real database, file system, or network call to pass.
- A constructor that takes zero dependencies for a class that clearly needs collaborators — the collaborators are probably hidden as static/global lookups instead.
- Business logic depending on a concrete class from a specific vendor SDK, rather than an interface the codebase owns.

## Code: violation → fix

```java
// Violation: constructs its own dependency; untestable without a real HTTP client
class PricingService {
    BigDecimal getExternalRate(String currency) {
        HttpClient client = HttpClient.newHttpClient(); // built here, every call
        // ... make a real network call ...
        return fetchRate(client, currency);
    }
}
```

```java
// Fix: dependency is injected; PricingService doesn't know or care how rates are fetched
interface ExchangeRateProvider {
    BigDecimal getRate(String currency);
}

class PricingService {
    private final ExchangeRateProvider rateProvider;

    PricingService(ExchangeRateProvider rateProvider) { // supplied from outside
        this.rateProvider = rateProvider;
    }

    BigDecimal getExternalRate(String currency) {
        return rateProvider.getRate(currency);
    }
}
```

The fix lets `PricingService` be unit tested with a fake `ExchangeRateProvider` returning a fixed rate, and lets the real HTTP-backed implementation be swapped for a cached or different-vendor one without touching `PricingService` at all.

## Review checklist

1. Does any business-logic method construct a concrete collaborator with `new` instead of receiving it?
2. Are a class's dependencies visible in its constructor, or hidden behind static/global lookups?
3. Can this class be unit tested without real I/O (network, database, file system, clock)?
4. Does business logic depend on a concrete vendor/SDK type where an owned interface would decouple it?
5. Is `Instant.now()`, `System.getenv()`, or similar called directly inside logic that would benefit from being deterministic in tests?

## How AI-generated code violates this

Generated code for a single, self-contained task tends to construct exactly what it needs inline, because that's the shortest correct-looking snippet for the task in front of the model — dependency injection requires anticipating a need (testability, swappability) that isn't stated in the prompt and has no immediate payoff within that one generation. This is especially common with time and randomness: a model asked to implement expiry logic will reach for `Instant.now()` directly rather than an injected `Clock`, because the direct call is simpler and passes the one example the model is implicitly checking itself against — and only becomes a problem when someone tries to write a deterministic test for the expiry boundary.

## Guardrail snippet

```
Business logic classes must receive their collaborators through a
constructor, never construct them internally with `new` or look them up
via static/global accessors. Depend on an interface the codebase owns,
not a concrete vendor/SDK class, wherever the logic needs to be testable
or swappable. Inject time (Clock) and configuration rather than calling
Instant.now() or System.getenv() directly inside business logic.
```

## Scoring

- **0 — Violated:** business logic constructs a real, concrete dependency internally, making it untestable without live infrastructure.
- **1 — Partial:** most dependencies are injected, but time, randomness, or one collaborator is still accessed statically.
- **2 — Met:** all dependencies are injected via constructor against owned interfaces; the class is fully unit-testable with fakes.
- **3 — Exemplary:** wiring is centralized in a single composition root/DI configuration, and architecture tests enforce that business logic never imports a concrete infrastructure class.

## Related

- [P-11 Coupling & Cohesion](11-coupling-and-cohesion.md) — dependency inversion is the primary technique for keeping coupling low between layers.
- [P-10 Meaningful Test Coverage](../crucial/10-meaningful-test-coverage.md) — untestable-without-real-I/O classes are the most common reason a team can't write meaningful unit tests in the first place.
- [P-14 Contract & API Design](14-contract-and-api-design.md) — the interface a dependency is injected against is itself a contract that needs the same design care.

## Going deeper

- Martin, *Agile Software Development, Principles, Patterns, and Practices*, ch. 11 (The Dependency Inversion Principle).
- Fowler, *"Inversion of Control Containers and the Dependency Injection pattern"* (martinfowler.com).
- Freeman & Pryce, *Growing Object-Oriented Software, Guided by Tests*, ch. 2–3 — dependency injection as a design driver for testability.
