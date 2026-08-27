# P-04 · Error Handling & Failure Semantics

> Failures are classified, surfaced, and handled on purpose — never caught and discarded.

**Tier:** Crucial · **Scope:** any code that calls something that can fail — I/O, external services, parsing, another module's contract

## What it means

Every operation that can fail has to have a deliberate answer to two questions: what happens to the caller, and what happens to the system's state. "Deliberate" rules out the default LLM and human shortcut of catching a broad exception type and logging it — that answers neither question. Error handling is a contract, not a courtesy: a checked exception, a `Result`/`Either` return type, or a documented thrown-exception list tells the caller what can go wrong and forces them to decide what to do about it. The decision to fail fast (propagate immediately, stop the operation) versus degrade gracefully (fall back, retry, return partial results) has to be made per failure type, based on what the failure means for correctness — not applied uniformly because one pattern was easiest to type.

Distinguish between errors that are truly exceptional (a bug, a violated invariant) and errors that are expected outcomes of normal operation (a not-found lookup, a validation failure, a network timeout). The former should generally fail loudly; the latter are part of the method's contract and belong in its return type or a specific, checked exception — not a `RuntimeException` indistinguishable from a real bug.

## Why it matters

A swallowed exception doesn't just hide one failure — it converts a fail-fast bug into a silent-corruption bug, because the system keeps running on the assumption that the failed operation succeeded. That's the single most common cause of "we don't know how the data got into this state" incidents: something failed, was logged (if you're lucky) at a level nobody watches, and everything downstream proceeded as if it hadn't. The cost-to-fix curve is asymmetric — proper error handling at the point of failure is a few extra lines; reconstructing what state a system ended up in after a silently swallowed failure, weeks later, from partial logs, is a forensic exercise.

## What good looks like

- Caught exceptions are either handled meaningfully (retry, fallback, translate into a domain error) or rethrown — never caught and only logged with no further action.
- The exception type or return type communicates what actually went wrong, not a generic wrapper.
- Fail-fast is the default for invariant violations and programmer errors; graceful degradation is a deliberate choice for expected, recoverable conditions.
- Expected failure outcomes (not found, validation failure, conflict) are part of the method's signature — a checked exception, a sealed result type, or a documented specific exception — not a generic `RuntimeException`.
- Partial failure in a multi-step operation leaves the system in a known, recoverable state, not an undefined one.

## Violation signatures

- `catch (Exception e) { log.error(...); }` with no rethrow, no fallback, no state cleanup.
- An empty catch block, with or without a comment like `// ignore`.
- Catching `Exception` or `Throwable` broadly when only a specific exception type is expected.
- An exception message that's just the exception's class name or a generic "something went wrong."
- A method that can fail in three different ways, all surfaced as the same generic exception type.
- Retry logic with no bound (infinite retry) or no backoff, applied uniformly regardless of failure type.
- A multi-step operation where an exception midway leaves some steps applied and others not, with no compensation.
- Logging an exception's message but discarding its stack trace or cause chain.

## Code: violation → fix

```java
// Violation: swallowed exception, caller has no idea the write failed
void updateInventory(String sku, int delta) {
    try {
        inventoryRepository.adjust(sku, delta);
    } catch (Exception e) {
        log.error("inventory update failed", e); // and then... nothing
    }
}
```

```java
// Fix: specific exception type, propagated with context, caller decides
void updateInventory(String sku, int delta) throws InventoryUpdateException {
    try {
        inventoryRepository.adjust(sku, delta);
    } catch (DataAccessException e) {
        throw new InventoryUpdateException(
            "failed to adjust inventory for sku=%s delta=%d".formatted(sku, delta), e);
    }
}
```

The fix removes the illusion of success: the caller now knows the write didn't happen and must decide whether to retry, alert, or abort the larger operation — instead of proceeding on inventory state that's silently wrong.

## Review checklist

1. Does any `catch` block do nothing but log, with no rethrow, fallback, or compensating action?
2. Is `Exception` or `Throwable` caught where a more specific type is actually expected?
3. For an operation with multiple expected failure outcomes, are they distinguishable to the caller (type, field, code) — not collapsed into one generic error?
4. Is the fail-fast/degrade choice made deliberately per failure type, or applied uniformly regardless of what failed?
5. Does retry logic have a bound and a backoff, and is it applied only to failures that are actually safe to retry (see [P-06](06-idempotency-retry-safety.md))?
6. If this operation fails partway through a multi-step process, is the resulting state defined and recoverable?
7. Is the original cause/stack trace preserved when an exception is wrapped or logged?

## How AI-generated code violates this

`catch (Exception e) { log.error(...); }` is one of the most reliable tells of generated code: it satisfies the surface requirement ("handle the exception") without requiring the model to reason about what the caller should do next, which is the actually hard part of the task and the part most likely to need information the model doesn't have (does this system prefer to fail fast or degrade here?). Because the model has no operational stake in the outcome, it defaults to the version that compiles and looks responsible rather than the version that's operationally correct. This compounds with [missing observability](../cross-cutting/ai-code-failure-modes.md#missing-observability-because-nothing-forced-the-agent-to-add-it): a caught-and-logged exception often gets a log line and nothing else, which looks like handling but provides no metric or alert that would let anyone notice the failure is happening at all. Generated retry logic is a related pattern: it's common to see a retry loop added because "handle transient failures" was in the prompt, wrapped around an operation without checking whether that operation is actually idempotent — see [P-06](06-idempotency-retry-safety.md).

## Guardrail snippet

```
Never write a catch block that only logs — every caught exception must
either be rethrown (possibly wrapped with more context), trigger a defined
fallback, or be part of an explicit, justified decision to suppress it
(with a comment explaining why suppression is safe). Distinguish expected,
recoverable failures (not found, validation, conflict) — which belong in
the method's return type or a specific checked exception — from
programmer errors and invariant violations, which should fail fast.
Always preserve the original exception as the cause when wrapping.
```

## Scoring

- **0 — Violated:** a caught exception is discarded with only a log statement on a path that affects state or a user-visible result.
- **1 — Partial:** exceptions are surfaced but generically (broad catch types, undifferentiated error responses); the caller can't distinguish failure modes.
- **2 — Met:** failures are typed, propagated or handled deliberately, and the fail-fast/degrade choice is explicit per failure type.
- **3 — Exemplary:** the type system enforces handling (sealed result types, checked exceptions for expected failures) so a failure mode can't be silently ignored at compile time.

## Related

- [P-06 Idempotency & Retry Safety](06-idempotency-retry-safety.md) — retry is a specific error-handling strategy that's only safe when the retried operation is idempotent.
- [P-08 Transaction & Data Integrity Boundaries](08-transaction-data-integrity.md) — partial failure inside a transaction boundary is where swallowed exceptions do the most damage.
- [P-17 Observability](../high/17-observability.md) — a handled-but-unobserved failure is still invisible in production; handling and visibility are separate requirements.

## Going deeper

- Nygard, *Release It!*, 2nd ed., ch. 5 ("Stability Patterns") — fail-fast vs. degrade as an explicit architectural choice.
- Bloch, *Effective Java*, 3rd ed., ch. 10 (Exceptions) — checked vs. unchecked, preserving cause chains.
- Nygard, *Release It!*, 2nd ed., ch. 4 ("Stability Antipatterns") — the integration-point and cascading-failure antipatterns that swallowed errors produce.
