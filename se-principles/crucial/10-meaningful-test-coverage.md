# P-10 · Meaningful Test Coverage

> A test asserts observable behavior against the requirement — it should fail if the requirement is violated, and it should still fail if the bug it was written to catch comes back.

**Tier:** Crucial · **Scope:** every test, but especially tests attached to bug fixes and to code generated in the same pass it's meant to verify

## What it means

A test is meaningful if it can fail — if there's a real implementation change that would turn it red. That sounds trivial, but it's the exact property that coverage percentage doesn't measure: a line can be executed by a test without any assertion depending on what that line actually computed. The discipline is to write the test from the *requirement*, independent of how the implementation works internally, and to specifically include the failure path, not just the success path — most production incidents happen on the path that wasn't tested, and that's disproportionately the error/edge-case path rather than the happy path, which gets exercised informally just by running the feature once during development.

This principle doesn't ask for more tests; it asks for tests that would actually catch a regression. A suite with 40% coverage that asserts real behavior is more valuable than one with 95% coverage that mocks out the logic under test and checks that a mock was called.

## Why it matters

A test suite's entire value proposition is catching a regression before a human does. A suite full of tests that assert the implementation back at itself provides the *appearance* of that safety net with none of the substance — it will pass after someone reintroduces the exact bug it was supposedly guarding against, because the test was derived from the buggy (or now-different) implementation rather than the specification. The cost isn't visible until the moment it matters: a refactor or a fix ships confidently because "all tests pass," and the actual regression reaches production because nothing in the suite could have caught it. Diagnosing that failure mode later is expensive specifically because the team's instinct — "we have tests for this" — is the thing actively misleading them.

## What good looks like

- Tests are written against the documented behavior/requirement, not by reading the implementation and describing what it does.
- For any bug fix, there's a test that fails on the pre-fix code and passes on the post-fix code — you can verify this by checking it out against the parent commit.
- Failure and edge-case paths have explicit tests, not just the happy path (empty input, invalid input, a downstream dependency failing).
- Mocks are used for genuine external dependencies (network, time, randomness) — not for the unit of logic actually being verified.
- Assertions check outputs, state changes, or emitted events — not "was this internal method called," except where that call *is* the observable contract (e.g., verifying a message was published).

## Violation signatures

- A test that mocks the exact method under test, or mocks the class it belongs to, so the assertion checks the mock's behavior instead of the real code's.
- `verify(mock).someMethod())` as the only assertion, where the actual output or state change is never checked.
- A test whose expected value was computed by running the implementation once and hardcoding the result, rather than derived independently from the spec.
- 100% line coverage on a module with zero tests for a null, empty, or error-path input.
- A test for a bug fix that would still pass if the fix were reverted (check this directly — it's the single fastest audit).
- Assertions on incidental implementation details (internal field values, private method call order) rather than the public, observable contract.
- Flaky tests disabled or given a generous retry instead of fixed, hiding a real intermittent bug.
- A test file whose test names describe *what the code does* ("testProcessOrder") instead of *what behavior is expected* ("rejectsOrderWithNegativeQuantity").

## Code: violation → fix

```java
// Violation: mocks the logic under test, only checks a method was called
@Test
void testCalculateDiscount() {
    DiscountPolicy policy = mock(DiscountPolicy.class);
    when(policy.eligibleFor(any())).thenReturn(true);
    OrderService service = new OrderService(policy);

    service.applyDiscount(order, "SAVE10");

    verify(policy).eligibleFor(order); // proves nothing about the actual discount math
}
```

```java
// Fix: exercises the real policy, asserts the observable result against the spec
@Test
void appliesTenPercentDiscountWhenOrderExceedsThreshold() {
    Order order = new Order(new BigDecimal("150.00"));
    DiscountPolicy policy = new PercentageDiscountPolicy("SAVE10", 10, new BigDecimal("100"));

    policy.apply(order, "SAVE10");

    assertEquals(new BigDecimal("135.00"), order.getTotal()); // the actual business outcome
}

@Test
void doesNotApplyDiscountBelowThreshold() {
    Order order = new Order(new BigDecimal("50.00"));
    DiscountPolicy policy = new PercentageDiscountPolicy("SAVE10", 10, new BigDecimal("100"));

    policy.apply(order, "SAVE10");

    assertEquals(new BigDecimal("50.00"), order.getTotal()); // the boundary case, not just the happy path
}
```

The fix drops the mock of the logic being tested, asserts the actual computed total against a value derived from the spec (10% off over $100), and adds the boundary case that the original test never considered.

## Review checklist

1. Would this test still pass if the bug it's meant to prevent were reintroduced? (Check by reverting the fix locally.)
2. Does any test mock the exact class or method it's supposed to be verifying?
3. Is there a test for the failure/edge-case path, not just the success path?
4. Are assertions checking observable output/state, or just that an internal method was invoked?
5. Was the expected value in each assertion derived from the spec, or copied from a single run of the implementation?
6. Do test names describe expected behavior, not implementation steps?

## How AI-generated code violates this

When the same generation pass writes the implementation and its tests, the model has full visibility into what it just wrote and the path of least resistance is to describe *that* — mocking the exact collaborator under test, or asserting the exact value the implementation happens to produce, rather than independently deriving the expected value from the requirement. This is the single most consistent failure mode in agent-generated test suites: they achieve high coverage numbers and pass reliably, but a large fraction would still pass against a subtly wrong implementation, because they were never adversarial to begin with — see [tests written against the implementation rather than the requirement](../cross-cutting/ai-code-failure-modes.md#tests-written-against-the-implementation-rather-than-the-requirement) for the general pattern. The fastest audit for this is exactly the one in this page's checklist: revert the fix or the feature logic and rerun the suite. If nothing goes red, the tests were never testing the requirement.

## Guardrail snippet

```
Write tests from the requirement or spec, not by describing what the
implementation currently does. Never mock the exact class or method under
test — only mock genuine external dependencies (network, time, randomness,
other services). Include at least one test for the failure/edge-case path
for every new function, not only the happy path. Before considering a bug
fix complete, verify its test fails against the pre-fix code and passes
against the post-fix code.
```

## Scoring

- **0 — Violated:** a test suite exists but would pass even with the actual bug reintroduced (mocks the logic under test, or asserts on implementation details).
- **1 — Partial:** happy-path behavior is well tested; failure/edge-case paths are missing or shallow.
- **2 — Met:** tests assert observable behavior derived from the spec, including failure paths; a fix's test fails pre-fix and passes post-fix.
- **3 — Exemplary:** tests are written before or independently of the implementation (spec-first, property-based, or by a separate review pass), making it structurally hard for them to encode the implementation's own blind spots.

## Related

- [P-01 Correctness & Edge Cases](01-correctness-and-edge-cases.md) — this principle is the enforcement mechanism; edge cases without a test are edge cases you're only hoping are handled.
- [P-09 Separation of Concerns / SRP](09-separation-of-concerns-srp.md) — logic that isn't separated from its framework/persistence layer usually can't be given a meaningful unit test at all.
- [P-04 Error Handling & Failure Semantics](04-error-handling-failure-semantics.md) — failure-path tests are what actually verify the error-handling contract, not just its presence in the code.

## Going deeper

- Meszaros, *xUnit Test Patterns*, ch. 1–4 — test doubles and the specific failure mode of over-mocking.
- Fowler, *"Mocks Aren't Stubs"* (martinfowler.com) — the classic treatment of when mocking undermines what a test actually verifies.
- Feathers, *Working Effectively with Legacy Code*, ch. 4 — characterization tests and the difference between testing behavior and testing implementation.
