# P-06 · Idempotency & Retry Safety

> Any operation that can be delivered or retried more than once produces the same end state as if it happened exactly once.

**Tier:** Crucial · **Scope:** any operation reachable via a network call, message queue, or scheduled retry — anything that doesn't have exactly-once delivery guaranteed underneath it (which is nearly everything)

## What it means

Most distributed systems give you at-least-once delivery, not exactly-once — a client retries after a timeout even though the server actually processed the request, a message queue redelivers after a consumer crash before acknowledging, a load balancer replays a request against a new instance. The system has to be designed so that processing the same logical operation twice is harmless: the second attempt either detects it's a duplicate and no-ops, or the operation is naturally idempotent (setting a value is idempotent; incrementing it is not). This is a design property, not a testing property — you can't test your way to idempotency by trying it twice in a test, you have to reason about it structurally: what makes this operation's *effect* the same regardless of how many times it's applied.

The common mechanism is an idempotency key: the caller supplies a unique identifier for the logical operation (not the HTTP request, the *business* operation — "charge this order once"), and the server records that it's seen that key and returns the original result on a repeat instead of repeating the side effect.

## Why it matters

Non-idempotent operations under retry produce duplicated side effects that are indistinguishable from legitimate ones after the fact: a payment charged twice, an email sent three times, an inventory decrement applied twice for one sale. These aren't crashes you get a stack trace for — they're financial and customer-facing discrepancies discovered later, usually by the customer, and reconciling them means proving which of the duplicate operations was the "real" one, which is often impossible from the data alone. The cost-to-fix curve is asymmetric in the same way as [error handling](04-error-handling-failure-semantics.md): an idempotency key at design time is a column and a check; a refund process for double-charged customers after the fact touches support, finance, and trust.

## What good looks like

- Any operation with a real-world side effect (charge, send, deduct, publish) that can be retried has an idempotency key tied to the logical operation, checked before the side effect runs.
- Retries are only applied to operations verified safe to repeat — reads and idempotent writes — not blindly wrapped around anything that failed.
- State-setting operations (`setStatus(PAID)`) are preferred over state-transitioning ones (`incrementRetryCount()`) wherever the business logic allows it, because sets are naturally idempotent and increments aren't.
- A message consumer checks whether it has already processed a given message ID before applying its effect, not just before acknowledging it.
- The idempotency key or dedup mechanism survives the failure it's meant to protect against — stored durably, not just in the retrying client's memory.

## Violation signatures

- A payment, email, or inventory-changing call wrapped in a retry loop with no idempotency key.
- `UPDATE ... SET count = count + 1` on an operation triggered by a webhook or message that could be redelivered.
- A message consumer that acknowledges *before* processing, or processes before checking a dedup table.
- An idempotency key derived from something that changes per retry (a timestamp, a random request ID generated fresh each call) instead of the logical operation.
- A "retry on any exception" policy applied uniformly, without checking whether the specific operation is safe to repeat.
- No unique constraint or dedup check at the data layer for an operation that's supposed to happen "once per X."
- A distributed job scheduler with no lock/lease, so a slow run and a retried run can both execute concurrently.

## Code: violation → fix

```java
// Violation: retry can double-charge — no idempotency key, no dedup
void chargeOrder(Order order) {
    for (int attempt = 0; attempt < 3; attempt++) {
        try {
            paymentGateway.charge(order.getCustomerId(), order.getAmount());
            return;
        } catch (TimeoutException e) {
            // timeout doesn't mean it failed — it might have succeeded server-side
        }
    }
}
```

```java
// Fix: idempotency key tied to the order; retries bounded, backed off, and never silent
void chargeOrder(Order order) throws ChargeFailedException {
    String idempotencyKey = "charge:" + order.getId(); // stable across retries
    TimeoutException lastFailure = null;
    for (int attempt = 0; attempt < 3; attempt++) {
        try {
            paymentGateway.charge(order.getCustomerId(), order.getAmount(), idempotencyKey);
            return; // gateway guarantees this key charges at most once
        } catch (TimeoutException e) {
            lastFailure = e; // safe to retry: the same key collapses a duplicate
            backoff(attempt); // exponential + jitter, so retries don't synchronize
        }
    }
    // exhausting retries is a failure, not a success — never fall out of the loop silently
    throw new ChargeFailedException("charge unresolved for order=" + order.getId(), lastFailure);
}
```

The fix doesn't prevent the retry — a timeout genuinely doesn't tell you whether the charge landed — it makes the retry safe by tying every attempt to a stable key the gateway uses to collapse duplicates into a single effect. Note the two things beyond the key itself: a backoff, so three retries from many callers don't converge into a synchronized burst against an already-struggling gateway, and a `throw` after the loop. Falling out of a retry loop and returning normally reports success for an operation whose outcome is genuinely unknown — a [P-04](04-error-handling-failure-semantics.md) violation that idempotency does nothing to excuse.

## Review checklist

1. Does this operation have a real side effect (money, email, inventory, external state), and can it be retried or redelivered?
2. If so, is there an idempotency key or dedup check tied to the logical operation, not the transport-level request?
3. Is the idempotency key stable across retries (derived from the business entity), not regenerated per attempt?
4. Is retry logic applied only to operations verified safe to repeat, not blanket-applied to any failure?
5. Does a message/event consumer check for duplicate processing before applying its effect, not just before acknowledging?
6. Would a slow request that times out but actually succeeded server-side cause a duplicate on retry?

## How AI-generated code violates this

Retry loops are one of the most common patterns a model adds reflexively when a prompt mentions reliability or "handle transient failures," because retry-with-backoff is a well-represented, easy-to-produce pattern in training data — and it gets wrapped around whatever operation is in scope without the model checking whether that specific operation is safe to repeat, since idempotency is a property of the *business* semantics, not something visible in the code being retried. This is a case where fluent, idiomatic-looking code is actively dangerous: the retry loop looks like exactly the kind of resilience pattern a senior engineer would praise, while quietly making the failure mode worse (a single-shot bug becomes an occasionally-double-executed one). It compounds with [P-05 Concurrency](05-concurrency-thread-safety.md): a generated system that retries under concurrent load without a dedup key can produce more than two executions of the same logical operation, not just two.

## Guardrail snippet

```
Before adding retry logic to any operation, determine whether that
operation is idempotent — check-then-act operations, increments, and
anything with an external side effect (payment, email, message publish)
are not idempotent by default. If it isn't, add an idempotency key tied to
the logical business operation (not the transport request) and have the
downstream system dedupe on it before retrying. Never assume a timeout
means an operation didn't happen server-side.
```

## Scoring

- **0 — Violated:** a retriable operation with a real side effect has no idempotency mechanism.
- **1 — Partial:** an idempotency key exists but is derived from something that changes per retry, or dedup is checked inconsistently (e.g., only in one consumer, not all).
- **2 — Met:** every retriable side-effecting operation has a stable idempotency key checked before the effect runs.
- **3 — Exemplary:** idempotency is enforced structurally (a unique constraint, a dedicated dedup table with a durable TTL) so a bug in application logic can't reintroduce a duplicate.

## Related

- [P-04 Error Handling & Failure Semantics](04-error-handling-failure-semantics.md) — retry is a specific failure-handling strategy; this principle is the precondition that makes it safe.
- [P-08 Transaction & Data Integrity Boundaries](08-transaction-data-integrity.md) — the idempotency check and the side effect it guards usually need to be atomic with each other.
- [P-05 Concurrency & Thread Safety](05-concurrency-thread-safety.md) — a race on the dedup check itself reintroduces the exact bug idempotency was meant to prevent.

## Going deeper

- Kleppmann, *Designing Data-Intensive Applications*, ch. 8–9 — distributed systems failure modes and exactly-once semantics.
- Stripe API docs, *Idempotent Requests* — a widely cited, concrete production implementation of idempotency keys.
- Newman, *Building Microservices*, 2nd ed., ch. 6 — retries and idempotency in service-to-service communication.
