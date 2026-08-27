# P-05 · Concurrency & Thread Safety

> Any state shared across threads is protected by synchronization, made immutable, or eliminated — never left to chance.

**Tier:** Crucial · **Scope:** any code where two or more threads (or coroutines, or async callbacks) can access the same mutable state — including virtual threads, thread pools, and reactive pipelines

## What it means

Concurrency correctness means the program's outcome doesn't depend on the interleaving of operations across threads. That requires identifying, for every piece of mutable state, who else can touch it and when — and either giving it explicit protection (a lock, an atomic, a concurrent collection), removing the mutability (make it immutable, confine it to one thread), or removing the sharing (each thread gets its own copy). "It hasn't broken yet" is not evidence of correctness here — race conditions are famously load-dependent and can pass thousands of test runs before appearing in production under real concurrency.

This also covers the failure modes at the coordination level, not just the data level: deadlock from inconsistent lock ordering, unbounded thread creation that exhausts resources under load, and visibility bugs where one thread's write is never guaranteed to be seen by another (the class of bug `volatile` and the Java Memory Model exist to prevent).

## Why it matters

Concurrency bugs are the worst combination of rare, non-reproducible, and catastrophic: a race condition might occur once in ten thousand requests, evade every test run on a developer machine, and then corrupt a shared counter or double-process a payment under real production load. Because the bug depends on timing, it's often impossible to reproduce in isolation once found, which pushes the cost-to-fix curve to its steepest point in this entire guide — you're debugging a bug that won't recur on demand, often from a stack trace that shows the *symptom* thread, not the *cause* thread. Deadlocks and unbounded thread growth fail differently but just as expensively: the system doesn't corrupt data, it just stops, usually during peak load, when the coordination pattern that worked at low concurrency finally collides.

## What good looks like

- Shared mutable state is protected by a lock, an atomic type, or a concurrent collection — and the protection covers the *entire* operation (check-then-act sequences included), not just the individual reads/writes.
- Where possible, mutable shared state is avoided entirely — immutable objects, thread confinement, or message-passing instead of shared memory.
- Lock acquisition order is consistent across the codebase to prevent deadlock.
- Thread pools and executors have bounded size and bounded queues, sized deliberately for the workload.
- Any use of `volatile`, `synchronized`, `Atomic*`, or `java.util.concurrent` types reflects an understanding of what guarantee is actually needed (visibility vs. atomicity vs. ordering), not a reflexive addition.

## Violation signatures

- A shared field read and written from multiple threads with no `synchronized`, lock, `volatile`, or atomic type.
- Check-then-act on shared state (`if (map.containsKey(k)) map.put(k, ...)`) instead of an atomic equivalent (`computeIfAbsent`, `putIfAbsent`).
- A `HashMap`, `ArrayList`, or other non-thread-safe collection shared across threads without external synchronization.
- Nested lock acquisition where the order isn't consistent across all call sites (classic deadlock setup).
- `new Thread(...)` or an unbounded `Executors.newCachedThreadPool()` on a path that scales with request volume.
- A singleton or static field that's lazily initialized without synchronization (double-checked locking done wrong, or omitted).
- Synchronizing on a mutable or non-final field, or on a boxed primitive/interned `String` (accidental shared lock).
- A `synchronized` block that only covers part of a compound operation, leaving a window between the protected pieces.

## Code: violation → fix

```java
// Violation: check-then-act race on a shared cache
private final Map<String, User> cache = new HashMap<>();

User getUser(String id) {
    if (!cache.containsKey(id)) {          // two threads can both see "absent"
        cache.put(id, loadFromDb(id));     // both call loadFromDb, one write is lost/wasted
    }
    return cache.get(id);
}
```

```java
// Fix: atomic compute, thread-safe map, no window for a lost update
private final ConcurrentMap<String, User> cache = new ConcurrentHashMap<>();

User getUser(String id) {
    return cache.computeIfAbsent(id, this::loadFromDb); // check-and-set is atomic
}
```

The fix collapses the check and the write into a single atomic operation on a map designed for concurrent access, eliminating the window where two threads could both decide the entry is missing.

One caveat worth knowing before copying this pattern: `ConcurrentHashMap.computeIfAbsent` holds a lock on the map's bin for the duration of the mapping function, so a slow `loadFromDb` blocks other threads whose keys hash to that bin, and a mapping function that recursively touches the same map throws `IllegalStateException`. For a cheap, non-blocking computation this is the right tool; for genuine I/O-backed caching, prefer a purpose-built cache (Caffeine's `LoadingCache`, which bounds and evicts as well) over hand-rolling one on top of a map.

## Review checklist

1. For every field that can be read and written from more than one thread, is it protected (lock, atomic, concurrent collection) or immutable?
2. Are there any check-then-act sequences on shared state that aren't wrapped in a single atomic or synchronized operation?
3. Is any non-thread-safe collection (`HashMap`, `ArrayList`, `SimpleDateFormat`) shared across threads?
4. If multiple locks are acquired anywhere in this change, is the acquisition order consistent with the rest of the codebase?
5. Is any thread pool or executor unbounded, on a path whose concurrency scales with load?
6. Does a `synchronized` block or lock cover the *whole* compound operation it's meant to protect, not just part of it?

## Code: a second shape — async/reactive instead of shared-memory threading

```java
// Violation: shared mutable accumulator across async callbacks, no coordination
int[] total = {0};
futures.forEach(f -> f.thenAccept(v -> total[0] += v)); // racy, non-atomic increment
```

```java
// Fix: let the concurrency framework own the aggregation
CompletableFuture<Integer> totalFuture = CompletableFuture
    .allOf(futures.toArray(new CompletableFuture[0]))
    .thenApply(v -> futures.stream()
        .mapToInt(CompletableFuture::join)
        .sum()); // no shared mutable state at all
```

Async code hides shared-state races behind callback syntax — the fix here is the same principle applied differently: remove the shared mutable variable instead of trying to synchronize access to it across callbacks that may run on different threads.

## How AI-generated code violates this

Generated concurrent code tends to be syntactically fluent and semantically unaware of interleaving — a model can produce a perfectly formatted `synchronized` block that protects the wrong scope, or reach for `ConcurrentHashMap` as a reflexive "thread safety" signal while still performing a non-atomic check-then-act sequence on it, because the fix looks safe without actually being safe. This is a case where [plausible-but-wrong logic that reads correctly](../cross-cutting/ai-code-failure-modes.md#plausible-but-wrong-logic-that-reads-correctly) is especially dangerous, because concurrency bugs specifically don't show up on the inputs a quick test run would exercise — a single-threaded test suite, which is what a generated test suite almost always is, will pass every time regardless of whether the underlying code is race-free. Generated code also tends to under-bound thread pools and retry/backoff loops, since "handle concurrent requests" is often satisfied with `newCachedThreadPool()` or an unbounded `Executors` call that works fine in the small example and fails only under production load the model never simulated.

## Guardrail snippet

```
Before writing to any field that could be accessed from more than one
thread, identify every reader and writer and either protect the entire
operation with a lock/atomic/concurrent collection, make the state
immutable, or eliminate the sharing. Never leave a check-then-act sequence
on shared state unguarded — use computeIfAbsent, putIfAbsent, or an
explicit lock around the whole sequence, not just part of it. Never use
an unbounded thread pool or executor on a path whose load scales with
traffic. Do not assume a single passing test run proves thread safety.
```

## Scoring

- **0 — Violated:** shared mutable state exists with no protection, or a check-then-act race exists on shared state.
- **1 — Partial:** most shared state is protected, but at least one compound operation has a window, or a thread pool is unbounded.
- **2 — Met:** all shared mutable state is protected or eliminated, lock ordering is consistent, pools are bounded.
- **3 — Exemplary:** shared mutability is designed out entirely (immutable data, message-passing, thread confinement) rather than managed with locks.

## Related

- [P-06 Idempotency & Retry Safety](06-idempotency-retry-safety.md) — concurrency and retries compound: a race condition combined with a retry can double an effect that would've been merely wrong once.
- [P-15 Immutability by Default](../high/15-immutability-by-default.md) — the strongest fix for a concurrency bug is usually removing the mutability, not adding a lock.
- [P-07 Resource Lifecycle](07-resource-lifecycle.md) — thread pools and executors are resources with the same lifecycle-leak risks as connections.

## Going deeper

- Goetz et al., *Java Concurrency in Practice* — the canonical reference for this entire principle, especially ch. 2–4 on shared state and safe publication.
- Lea, *Concurrent Programming in Java*, 2nd ed. — deeper treatment of lock ordering and deadlock avoidance.
- The Java Language Specification, ch. 17 (Threads and Locks) — the formal memory model underlying `volatile` and `synchronized`.
