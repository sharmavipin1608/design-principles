# P-09 · Separation of Concerns / SRP

> Each unit of code — class, method, module — has exactly one reason to change.

**Tier:** Crucial · **Scope:** every class and method, but especially the entry points where requests, business logic, and persistence meet (controllers, handlers, service methods)

## What it means

A "reason to change" is an axis of variation: presentation format, business rule, persistence mechanism, external protocol. When one method or class handles more than one of these, a change to any single axis risks touching code that has nothing to do with that change, and testing one concern requires dragging in the others. This is Crucial-tier, not just craftsmanship, because the most damaging violation isn't stylistic — it's business logic bleeding into a layer that wasn't built to enforce it correctly: a controller that decides authorization instead of delegating it, a persistence method that also sends an email, a request handler that inlines a pricing calculation instead of calling a pricing service. When that happens, the same rule ends up implemented — and drifting — in multiple places, or implemented in a layer that isn't tested the way business logic needs to be tested (a controller test suite usually verifies routing and status codes, not business invariants).

The test for SRP isn't "how many lines" — a long method that does one cohesive thing violates [P-22 KISS](../medium/22-kiss-complexity-budget.md), not this. The test is "how many *unrelated* reasons" would cause someone to open this file.

## Why it matters

When business logic lives inside a controller or handler, it inherits that layer's blast radius: a change to the HTTP framework, a routing refactor, or an API versioning effort now has to carefully extract business rules that were never meant to move independently, and often misses one. Worse, it inherits that layer's *testing* gap — logic buried in a controller is exercised only by integration/end-to-end tests hitting real HTTP, which are slower and less precise than a unit test against an isolated service method, so edge cases go unverified. The cost compounds over the life of the system: every new feature that needs the same business rule either duplicates it (because it's stuck in a controller nobody wants to import from) or triggers a larger refactor than the feature justified.

## What good looks like

- A controller/handler method's body reads as a thin sequence: parse request, call one service method, map result to response — no `if` branches encoding business rules.
- Business logic lives in a service/domain layer that has no framework dependency and can be unit tested without spinning up HTTP, a database, or a message broker.
- Persistence code only persists — it doesn't also validate business rules, send notifications, or make decisions about *what* to save, only *how*.
- A method's name accurately describes its one job, and nothing in its body does something the name doesn't imply.
- Cross-cutting concerns (logging, auth checks, transactions) are applied via a consistent mechanism (middleware, decorators, aspects) rather than hand-written inline in every method that needs them.

## Violation signatures

- A controller method containing an `if` that encodes a business rule (discount eligibility, pricing, access rules beyond basic authz) rather than delegating it.
- A repository/DAO method that also triggers a side effect unrelated to persistence (sending an email, calling another service).
- A class whose name is generic (`Manager`, `Processor`, `Handler`, `Service` with no qualifying domain word) and whose methods don't obviously belong together.
- A method that both computes a result and formats it for a specific output channel (HTML, a specific API's JSON shape) baked in.
- The same business rule implemented independently in two files because neither the controller nor a shared service owned it clearly.
- A unit test for "business logic" that requires mocking an HTTP request/response object to run.
- A single method whose diff touches unrelated behavior when only one requirement changed (a request handler edited for both a new validation rule and an unrelated response field rename).

## Code: violation → fix

```java
// Violation: controller owns business rules, persistence, and response shaping
@PostMapping("/orders/{id}/apply-discount")
ResponseEntity<?> applyDiscount(@PathVariable String id, @RequestParam String code) {
    Order order = orderRepo.findById(id);
    if (order.getTotal().compareTo(new BigDecimal("100")) > 0 && code.equals("SAVE10")) {
        order.setTotal(order.getTotal().multiply(new BigDecimal("0.9"))); // business rule, in a controller
    }
    orderRepo.save(order);
    return ResponseEntity.ok(Map.of("total", order.getTotal(), "id", order.getId()));
}
```

```java
// Fix: controller delegates; the discount rule is owned, named, and testable on its own
@PostMapping("/orders/{id}/apply-discount")
ResponseEntity<OrderView> applyDiscount(@PathVariable String id, @RequestParam String code) {
    Order order = discountService.applyDiscount(id, code); // one call, one reason to change here
    return ResponseEntity.ok(OrderView.from(order));
}

// In DiscountService — testable without HTTP, owns the actual rule
Order applyDiscount(String orderId, String code) {
    Order order = orderRepo.findById(orderId);
    discountPolicy.apply(order, code); // the rule itself, unit-testable in isolation
    return orderRepo.save(order);
}
```

The fix isolates the discount rule so it can be unit tested against every eligibility edge case without an HTTP layer, and so a future channel (batch job, admin tool) can reuse the exact same rule instead of reimplementing it.

## Review checklist

1. Does a controller/handler method contain a business rule (a threshold, an eligibility check, a calculation) instead of delegating it?
2. Can the business logic in this diff be unit tested without mocking HTTP, a database, or a message broker?
3. Does a persistence method do anything beyond persisting?
4. Is the same rule implemented in more than one place because no single layer clearly owns it?
5. Does this diff touch two unrelated concerns for what should have been one requirement?
6. Does the class/method name accurately describe everything it does — no silent extra responsibility?

## How AI-generated code violates this

Given a single-turn prompt like "add an endpoint that applies a discount," a model has every incentive to write the entire thing in the controller method it's already editing — it's the shortest path to a working response, and the model has no felt cost for making the business rule harder to reuse or test later, because it isn't going to be the one reusing it. This produces SRP violations that look complete and correct in isolation, since the endpoint works, and only show up as a problem when a second feature needs the same rule and a human discovers it's landlocked inside a controller. It compounds with [speculative abstraction](../cross-cutting/ai-code-failure-modes.md#speculative-abstraction--layers-invented-for-a-single-caller) in a specific way: models sometimes overcorrect on a *different* file in the same session by inventing a service layer with excessive ceremony for a trivial case, while under-separating the concern that actually needed it — the separation instinct isn't reliably calibrated to where it's actually needed.

## Guardrail snippet

```
Controllers, handlers, and persistence methods must not contain business
rules — no eligibility checks, calculations, or decision logic beyond
parsing input and mapping output. Put business logic in a dedicated
service/domain method that has no framework or persistence dependency and
can be unit tested directly. Before adding a rule to a controller or
repository method, check whether an existing service already owns that
concern — extend it there instead of duplicating the rule.
```

## Scoring

- **0 — Violated:** business logic is implemented inline in a controller, handler, or persistence method.
- **1 — Partial:** logic is mostly separated, but one rule leaked into the wrong layer, or the same rule exists in two places.
- **2 — Met:** each layer has exactly one concern; business logic is independently unit-testable.
- **3 — Exemplary:** the separation is structurally enforced (module boundaries, build-time dependency rules) so a future change can't easily reintroduce the leak.

## Related

- [P-11 Coupling & Cohesion](../high/11-coupling-and-cohesion.md) — SRP at the method/class level is the local instance of the cohesion half of this High-tier principle at the module level.
- [P-10 Meaningful Test Coverage](10-meaningful-test-coverage.md) — logic that isn't separated usually can't be unit tested at all, which is often the first symptom you notice.
- [P-16 Open/Closed](../high/16-open-closed.md) — a well-separated business rule is what makes extending it (a new discount type) additive instead of an edit to a shared controller.

## Going deeper

- Martin, *Clean Architecture*, ch. 7 (SRP) and ch. 22 (The Clean Architecture — the layering this principle scales up to).
- Fowler, *Patterns of Enterprise Application Architecture*, ch. 1–2 — layering rationale (domain logic vs. presentation vs. data source).
- Evans, *Domain-Driven Design*, ch. 4 — the case for isolating a domain layer from infrastructure concerns.
