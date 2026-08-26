# P-07 · Resource Lifecycle

> Every acquired resource — connection, stream, thread, lock — has a deterministic, guaranteed release, including on the error path.

**Tier:** Crucial · **Scope:** any code that acquires a connection, file handle, stream, thread pool, executor, or lock

## What it means

A resource, in this sense, is anything finite that has to be explicitly given back: a database connection from a pool, a file handle, a socket, a thread, a lock. Acquiring one creates an obligation to release it, and that obligation has to hold on every exit path from the code that acquired it — the normal return, every exception path, and early returns in between. "I closed it at the end of the method" isn't good enough if an exception three lines earlier skips that line entirely. The language mechanism for this in Java is try-with-resources (or a `finally` block for cases that predate `Closeable`), and the discipline is using it every time, not just when you remember.

This extends past individual resources to pools and bounds: an executor with no queue limit, a connection pool with no max size, a cache with no eviction policy — these aren't leaks in the classic sense, but they're the same failure mode at the aggregate level: something unbounded, growing until it exhausts memory, connections, or threads under load that a fixed-size test never simulated.

## Why it matters

A resource leak doesn't fail the request that caused it — it fails some *other* request later, once the pool is exhausted, which makes it one of the hardest classes of bug to connect back to its source. A connection leaked in a rarely-hit error path can run fine for weeks until that path executes often enough to exhaust the pool, at which point the symptom is "the whole service is timing out" with no obvious link to the three-week-old code change that caused it. The cost-to-fix curve is steep because diagnosis, not the fix itself, dominates: the fix is usually one `try-with-resources` wrapper; finding which of hundreds of resource acquisitions is the leaking one, under production load, is the expensive part.

## What good looks like

- Every `Closeable`/`AutoCloseable` resource is acquired inside a try-with-resources statement, not manually closed at the end of a method body.
- Resource acquisition and release are visibly paired in the same scope — a reviewer shouldn't have to trace call graphs to find the matching close.
- Executors, thread pools, and connection pools are explicitly bounded (max size, queue capacity) and shut down on application/component shutdown.
- Timeouts are set on anything that waits on an external resource (socket reads, connection acquisition, lock waits) — nothing blocks indefinitely by default.
- Streams composed from other streams (wrapping a `FileInputStream` in a `BufferedInputStream`) close correctly as a unit, without leaking the inner one if the outer wrapper's construction fails.

## Violation signatures

- `new FileInputStream(...)`, `DriverManager.getConnection(...)`, or similar acquired outside a try-with-resources block.
- A manual `.close()` call at the end of a method body with no surrounding `try`/`finally`.
- An executor (`Executors.newFixedThreadPool`, `newCachedThreadPool`) created per-request instead of once and reused, or never shut down.
- A connection pool, cache, or queue with no configured maximum size.
- A blocking call (`Future.get()`, socket read, lock `acquire()`) with no timeout argument.
- A resource wrapped in another resource where the outer constructor can throw after the inner one is already open (leaks the inner one).
- `finally` blocks that themselves throw, silently swallowing the original exception and skipping cleanup.
- A `while(true)` polling loop with no interrupt/cancellation path to release what it's holding.

## Code: violation → fix

```java
// Violation: connection leaks if query() throws; no timeout on the query itself
Connection conn = dataSource.getConnection();
ResultSet rs = conn.createStatement().executeQuery(sql);
List<Row> rows = mapRows(rs);
conn.close(); // never reached if executeQuery or mapRows throws
return rows;
```

```java
// Fix: try-with-resources guarantees release on every exit path
try (Connection conn = dataSource.getConnection();
     Statement stmt = conn.createStatement()) {
    stmt.setQueryTimeout(5); // seconds — no unbounded wait
    try (ResultSet rs = stmt.executeQuery(sql)) {
        return mapRows(rs);
    }
} // conn, stmt, and rs all close here, even if mapRows throws
```

The fix guarantees release regardless of where an exception occurs, and adds a query timeout so a slow or hanging query can't hold the connection indefinitely under load.

## Review checklist

1. Is every `Closeable`/`AutoCloseable` resource acquired inside a try-with-resources block?
2. If a resource is wrapped in another resource, does construction failure of the outer one still release the inner one?
3. Are executors, thread pools, and connection pools bounded, and shut down somewhere on component/application shutdown?
4. Does every blocking call (I/O, lock, future) have a timeout?
5. Is there a `finally` block that could itself throw and skip the cleanup it was meant to guarantee?
6. Is any resource created per-request that should be created once and reused (a new thread pool per call is a leak in slow motion)?

## How AI-generated code violates this

Generated code that demonstrates a database or file operation frequently gets the *acquire* and the *use* right — because that's the part directly implied by the task — and skips or gets wrong the *release on every exit path* part, because that requires reasoning about exception flow that isn't visible in the happy-path example the model is pattern-matching against. It's common to see a manual `.close()` placed correctly for the normal return but with no `finally` or try-with-resources, which passes every test that doesn't force an exception mid-method — exactly the blind spot described in [tests written against the implementation](../cross-cutting/ai-code-failure-modes.md#tests-written-against-the-implementation-rather-than-the-requirement). Unbounded executors and connection pools show up for a related reason: `Executors.newCachedThreadPool()` or a default pool size is the shortest snippet that "handles concurrency," and bounding it correctly requires knowing the expected load, which isn't in scope for a model generating one function in isolation.

## Guardrail snippet

```
Acquire every Closeable/AutoCloseable resource inside a try-with-resources
statement — never call .close() manually at the end of a method body.
When wrapping one resource in another, ensure the inner one is still
released if the outer wrapper's construction fails. Set an explicit
timeout on every blocking call (I/O, lock acquisition, future.get()).
Bound every executor, thread pool, and connection pool explicitly — never
use an unbounded pool or queue on a path whose load can grow.
```

## Scoring

- **0 — Violated:** a resource is acquired without a guaranteed release on an exception path, or a pool/executor is unbounded on a load-scaling path.
- **1 — Partial:** most resources use try-with-resources, but at least one manual close exists, or a blocking call has no timeout.
- **2 — Met:** all resources are released deterministically on every exit path; pools and executors are bounded and have timeouts.
- **3 — Exemplary:** resource lifecycle is enforced structurally (a wrapper type that can't be constructed without a matching release, static analysis in CI) rather than by convention alone.

## Related

- [P-05 Concurrency & Thread Safety](05-concurrency-thread-safety.md) — unbounded thread pools are a resource-lifecycle failure with a concurrency-shaped symptom (resource exhaustion under load).
- [P-04 Error Handling & Failure Semantics](04-error-handling-failure-semantics.md) — a `finally` block that swallows the original exception while attempting cleanup is a specific, damaging instance of this principle.
- [P-08 Transaction & Data Integrity Boundaries](08-transaction-data-integrity.md) — an unreleased connection can hold a transaction open longer than intended, turning a resource leak into a data-integrity risk.

## Going deeper

- Bloch, *Effective Java*, 3rd ed., Item 9 ("Prefer try-with-resources to try-finally").
- Goetz et al., *Java Concurrency in Practice*, ch. 7 (Cancellation and Shutdown) — bounded executors and graceful shutdown.
- Nygard, *Release It!*, 2nd ed., ch. 5 — timeouts and bulkheads as resource-exhaustion defenses.
