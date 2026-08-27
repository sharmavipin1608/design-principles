# P-08 · Transaction & Data Integrity Boundaries

> Atomicity boundaries match business intent — there is no window where a partial write is visible or possible.

**Tier:** Crucial · **Scope:** any operation that writes to more than one place — multiple tables, a database plus a cache, a database plus a message publish, two independent services

## What it means

Every operation that should be "all or nothing" from the business's perspective needs an actual atomicity boundary that matches — a database transaction spanning exactly the writes that must succeed or fail together, no more and no less. The hard cases are the ones where the natural transaction boundary doesn't cover everything that needs to be atomic: writing to a database and publishing a message, writing to a database and calling an external API, writing to two different databases. This is the dual-write problem, and there's no free lunch — you either need a single transactional resource (outbox pattern, a single database), or you need to design for eventual consistency deliberately, with compensations that repair the cases where one side succeeds and the other doesn't.

The other half of this principle is isolation: knowing what concurrent transactions can see of each other's in-flight writes, and whether the isolation level in use actually prevents the anomalies the business logic assumes are prevented (dirty reads, non-repeatable reads, phantom reads, lost updates).

## Why it matters

A partial write is worse than a failed write, because a failed write is detectable — you get an error and can retry or alert — while a partial write leaves the system in a state that looks valid but isn't: an order marked paid with no corresponding ledger entry, a message published for a database write that then rolled back, a balance updated by two concurrent transactions that both read the same starting value and one update is silently lost. These bugs surface as data reconciliation problems — auditors or customers finding discrepancies — rather than as exceptions, which means the cost-to-fix curve includes not just a code fix but a data-repair effort to figure out how many records are affected and what the correct value should have been.

## What good looks like

- Multi-table writes that must succeed or fail together are wrapped in a single database transaction, not sequential independent writes.
- Where a write must be atomic with an external effect (message publish, API call) that can't share a transaction, an outbox or saga pattern makes the two eventually consistent with a defined repair path — not a hope that both calls happen to succeed.
- The isolation level is chosen deliberately based on what anomalies the business logic can tolerate, not left at a framework default without checking.
- Read-modify-write sequences on shared data use a mechanism (`SELECT ... FOR UPDATE`, optimistic locking with a version column, atomic database operations) that prevents lost updates under concurrency.
- Compensating actions exist for operations that can partially fail across a boundary that isn't atomic (a saga's rollback step, a reconciliation job).

## Violation signatures

- Two related writes to two different tables/data stores issued as independent calls with no shared transaction.
- A database write followed by a message publish (or vice versa) with no outbox pattern — if the second call fails, the first is already committed.
- A read-modify-write sequence (`read balance, compute new balance, write balance`) with no locking or version check, editable concurrently by two callers.
- `@Transactional` (or equivalent) applied to a method that also makes an external HTTP call inside it — the external call isn't part of the transaction, but a rollback can't undo it.
- `@Transactional` on a non-public method, or on a method called from *within* the same class — under Spring's default proxy-based AOP both are silently ignored, so the annotation is decorative and the transaction never starts.
- Two or more rows locked in an order derived from the arguments (`from` then `to`) rather than a deterministic global order — a deadlock as soon as two callers pass those arguments in opposite order.
- A batch job that processes N records without a per-record or per-batch transaction boundary, so a crash midway leaves an undefined subset processed.
- No unique constraint or version column on a table where concurrent updates are expected.
- A "transaction" that spans a network call to another service, blocking a database connection for the duration of an external round trip.

## Code: violation → fix

```java
// Violation: two independent writes; if the second fails, the first isn't undone,
// and there's no atomicity across the read-modify-write on balance either
void transferFunds(String fromId, String toId, BigDecimal amount) {
    Account from = accountRepo.findById(fromId);
    from.setBalance(from.getBalance().subtract(amount));
    accountRepo.save(from);           // committed independently

    Account to = accountRepo.findById(toId);
    to.setBalance(to.getBalance().add(amount));
    accountRepo.save(to);             // if this throws, money has vanished
}
```

```java
// Fix: one transaction; pessimistic row locks taken in a consistent order
@Transactional // MUST be public: Spring's proxy-based AOP silently ignores it otherwise
public void transferFunds(String fromId, String toId, BigDecimal amount) {
    // Lock in a deterministic order (by id), NOT in from/to order — otherwise a
    // concurrent A->B and B->A transfer acquire the two rows in opposite orders
    // and deadlock. Ordering the acquisition removes the cycle entirely.
    String firstId = fromId.compareTo(toId) <= 0 ? fromId : toId;
    String secondId = fromId.compareTo(toId) <= 0 ? toId : fromId;
    accountRepo.lockById(firstId);   // SELECT ... FOR UPDATE
    accountRepo.lockById(secondId);

    Account from = accountRepo.findById(fromId);
    Account to = accountRepo.findById(toId);
    if (from.getBalance().compareTo(amount) < 0) {
        throw new InsufficientFundsException(fromId);
    }
    from.setBalance(from.getBalance().subtract(amount));
    to.setBalance(to.getBalance().add(amount));
    accountRepo.save(from);
    accountRepo.save(to); // both commit together, or the whole method rolls back
}
```

The fix makes the transfer atomic at the database level (both writes commit or neither does) and takes a row lock so a concurrent transfer touching the same account can't read a stale balance and silently lose an update. Three details carry most of the weight:

- **`public` is load-bearing.** With Spring's default proxy-based AOP, `@Transactional` on a non-public method is silently ignored — no proxy interception, no transaction, no error. The method would look transactional and not be.
- **`SELECT ... FOR UPDATE` is *pessimistic* locking** — it blocks competing writers until commit. The alternative, optimistic locking, uses a `@Version` column and detects the conflict at commit time, failing the loser with an exception the caller must retry. Either prevents lost updates; they are not the same mechanism, and mixing up the terms leads to expecting retry semantics the code doesn't have.
- **Lock ordering prevents deadlock.** Acquiring in `from`/`to` order means simultaneous A→B and B→A transfers each hold what the other needs. Sorting the identifiers before acquiring imposes a global order, which is the standard structural fix (see [P-05](05-concurrency-thread-safety.md)).

## Review checklist

1. Do all writes that must succeed or fail together share one transaction, or are they issued as independent calls?
2. Is there a dual-write here (database + message, database + external API) with no outbox/saga to make it eventually consistent and repairable?
3. Does any read-modify-write sequence on shared data lack locking or a version check?
4. Is an external network call happening *inside* a database transaction, holding a connection for the round trip?
5. Is every `@Transactional` method public and called from outside its own class — or is the annotation silently doing nothing?
6. If more than one row/resource is locked, are they acquired in a deterministic global order rather than argument order?
7. Was the isolation level chosen deliberately, and does it actually prevent the anomalies this logic assumes are prevented?
8. If this operation fails partway, is the resulting state either fully rolled back or explicitly compensable?

## How AI-generated code violates this

Generated code for "transfer X from A to B" style operations frequently implements each write as its own repository call because that's how the individual CRUD operations were each demonstrated in isolation during training — the model composes two correct-looking single-entity saves without the transaction wrapper that makes them atomic together, because atomicity is a property of the combination, not visible in either save call on its own. Dual-write bugs are even more common in generated code that "sends a notification after saving," because the natural, readable order (save, then publish) is exactly the order that has no correctness guarantee, and only someone thinking specifically about the failure window between the two calls would catch it. This is a case where the code is entirely idiomatic and passes every functional test — see [plausible-but-wrong logic that reads correctly](../cross-cutting/ai-code-failure-modes.md#plausible-but-wrong-logic-that-reads-correctly) — because the bug only manifests during a failure of the second call, which normal test runs never inject.

## Guardrail snippet

```
Any set of writes that must succeed or fail together must share a single
transaction — never issue them as independent sequential calls. When a
write must be atomic with an effect that can't share a transaction
(message publish, external API call), use an outbox pattern or an
explicit compensation step — never assume both calls will simply succeed.
Any read-modify-write on data that can be updated concurrently needs a
lock or version check. Never make an external network call inside an open
database transaction.
```

## Scoring

- **0 — Violated:** related writes are issued independently with no shared transaction, or a dual-write has no compensation path.
- **1 — Partial:** the primary transaction boundary is correct, but a read-modify-write lacks locking, or isolation level wasn't deliberately chosen.
- **2 — Met:** atomicity boundaries match business intent; dual-writes have an outbox/saga; concurrent updates are protected.
- **3 — Exemplary:** the boundary is enforced structurally (a repository method that only exposes the atomic operation, not the individual steps) so it can't be decomposed incorrectly by a future caller.

## Related

- [P-06 Idempotency & Retry Safety](06-idempotency-retry-safety.md) — a saga's compensation steps and an outbox's redelivery both need to be idempotent, or the repair mechanism creates new duplicates.
- [P-05 Concurrency & Thread Safety](05-concurrency-thread-safety.md) — lost updates from unprotected read-modify-write are the database-level version of the same race-condition problem.
- [P-04 Error Handling & Failure Semantics](04-error-handling-failure-semantics.md) — how a partial failure across a non-atomic boundary is surfaced determines whether anyone finds out before a customer does.

## Going deeper

- Kleppmann, *Designing Data-Intensive Applications*, ch. 7 (Transactions) and ch. 9 (Consistency and Consensus).
- Richardson, *Microservices Patterns*, ch. 4 (Saga pattern) and the transactional outbox pattern.
- Joshi, *"Patterns of Distributed Systems"* (hosted on martinfowler.com) — write-ahead log, leader-follower, and related consistency patterns.
