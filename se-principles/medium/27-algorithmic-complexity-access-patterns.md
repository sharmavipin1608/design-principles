# P-27 · Algorithmic Complexity & Access Patterns

> Data access patterns don't scale worse than the data itself does.

**Tier:** Medium · **Scope:** loops over collections, database access inside loops, and any code whose cost grows with input size

## What it means

This principle is about the shape of an algorithm's cost as input grows, not about micro-level speed (that's [P-34 Micro-optimizations](../low/34-micro-optimizations.md), and it's deliberately Low-tier — this one is Medium because the *pattern* itself, not the constant factor, is often visibly wrong in a diff). The single most common instance in real codebases is the N+1 query problem: a loop over N records that issues one database query per iteration instead of one query for all of them, turning what should be a constant number of round trips into a number that grows linearly with the data. The same shape shows up as nested loops over collections where a lookup structure (a `Map`) would turn an O(n²) scan into O(n), or as repeated sorting of the same data inside a loop instead of once outside it. None of this requires a formal complexity analysis in review — it requires asking, for any loop, "does the number of database calls, network calls, or nested iterations grow with the size of the input," and if so, whether that's actually necessary.

## Why it matters

An N+1 query or an O(n²) scan is invisible at small scale — it passes every test written against a handful of fixture rows, runs fine in development, and only becomes visible once production data grows past the point where the pattern's cost curve diverges from what a reviewer or the original author assumed. By then it's not a code review comment, it's a production incident: a page that used to load in 200ms now takes 8 seconds because the "get orders for this customer" endpoint issues one query per order line item, and the fix requires the same code change it would have taken in review, except now it's happening under a latency alert instead of before merge.

## What good looks like

- Data needed for a batch of items is fetched in one query (a single `WHERE id IN (...)`, a join, a batch-loader), not one query per item in a loop.
- Repeated lookups inside a loop use a `Map`/`Set` built once outside the loop, not a linear scan of a list on every iteration.
- Sorting, deduplication, or other O(n log n)/O(n) operations happen once, outside any loop that would otherwise repeat them per iteration.
- Pagination or streaming is used for unbounded collections instead of loading everything into memory to process it.
- Where a nested loop is genuinely necessary, the inner collection is small and bounded, not something that grows with the same data set as the outer loop.

## Violation signatures

- A database/repository call inside a `for`/`forEach` loop over a collection fetched just before it (the canonical N+1 pattern).
- `list.contains(x)` or a linear search inside a loop, where a `Set`/`Map` built once would make each check O(1).
- Sorting or building an index/map freshly inside a loop that repeats for every outer iteration.
- Loading an entire table/collection into memory to filter it in application code instead of filtering at the query level.
- A nested loop where the inner loop's size grows with the same data set as the outer loop, with no early-exit or indexing.
- An API endpoint whose response time is documented or known to grow with the number of related records, with no pagination or batching in place.

## Code: violation → fix

```java
// Violation: N+1 — one query per order to fetch its line items
List<Order> orders = orderRepo.findByCustomerId(customerId);
for (Order order : orders) {
    order.setLineItems(lineItemRepo.findByOrderId(order.getId())); // N extra queries
}
```

```java
// Fix: one batch query, grouped in memory — a constant number of round trips
List<Order> orders = orderRepo.findByCustomerId(customerId);
List<String> orderIds = orders.stream().map(Order::getId).toList();
Map<String, List<LineItem>> itemsByOrderId = lineItemRepo.findByOrderIdIn(orderIds)
    .stream().collect(Collectors.groupingBy(LineItem::getOrderId));
orders.forEach(order -> order.setLineItems(itemsByOrderId.getOrDefault(order.getId(), List.of())));
```

The fix replaces N queries with 2, regardless of how many orders the customer has — the response time no longer grows with the customer's order history, which is exactly the property that N+1 code silently lacks until it's tested against realistic data volume.

## Review checklist

1. Is there a database, HTTP, or other I/O call inside a loop whose iteration count depends on earlier query results?
2. Does a loop perform a linear search (`contains`, indexOf) on a list that could be a `Map`/`Set` instead?
3. Is sorting, deduplication, or index-building happening repeatedly inside a loop instead of once outside it?
4. Is an entire collection loaded into memory where a paginated or filtered query would do?
5. Does a nested loop's inner size grow with the same data set as the outer loop?

## How AI-generated code violates this

The N+1 pattern is an easy default for a model to produce because it's the most locally obvious way to satisfy "for each order, get its line items" — looping and calling the natural-sounding per-item method reads as correct and passes any test built from a small fixture, which is virtually all generated tests. Recognizing that the loop needs to become a batch operation requires reasoning about a data volume the model isn't shown and has no reason to assume — the same blind spot as [correctness only against the given example](../cross-cutting/ai-code-failure-modes.md#plausible-but-wrong-logic-that-reads-correctly) applied to performance instead of correctness: the code is right for three test rows and silently wrong in its scaling behavior for three hundred thousand.

## Guardrail snippet

```
Never issue a database, HTTP, or other I/O call inside a loop whose
iteration count depends on runtime data size — batch the operation
instead (a single query with an IN clause, a bulk API call, a join).
Replace a linear search inside a loop with a Map/Set built once outside
it. Check any new loop: does its cost grow with input size in a way that
could be avoided, and would it still perform acceptably at realistic
production data volume, not just the size of the test fixture.
```

## Scoring

- **0 — Violated:** an N+1 query pattern or O(n²) scan exists on a path with unbounded or realistically large input.
- **1 — Partial:** the pattern is present but bounded to a small, known-fixed input size that won't grow.
- **2 — Met:** data access is batched appropriately; loops don't hide I/O or repeated linear scans.
- **3 — Exemplary:** access patterns are verified against realistic data volume (a load test, a query-count assertion in CI) so a regression is caught automatically, not just by inspection.

## Related

- [P-22 KISS / Complexity Budget](22-kiss-complexity-budget.md) — occasionally in tension: a batched, indexed solution can read as less obviously simple than the naive loop, even though it's the correct choice at scale.
- [P-29 Caching Correctness](29-caching-correctness.md) — caching is sometimes reached for to paper over an N+1 pattern instead of fixing the access pattern itself, which trades one problem for another (staleness).
- [P-08 Transaction & Data Integrity Boundaries](../crucial/08-transaction-data-integrity.md) — batching queries can change transaction/locking behavior; the fix needs to preserve whatever atomicity the original loop had.

## Going deeper

- Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, ch. 1–4 — the formal basis for reasoning about growth rate.
- Karwin, *SQL Antipatterns*, ch. 1 ("Jaywalking") and the N+1 selects antipattern specifically.
- High Scalability / vendor ORM documentation (e.g., Hibernate's "N+1 Selects Problem") — practical, tool-specific detection and fixes.
