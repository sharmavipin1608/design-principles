# P-29 · Caching Correctness

> A cache never becomes a second, disagreeing source of truth.

**Tier:** Medium · **Scope:** any read path backed by a cache — in-memory, distributed (Redis/Memcached), or HTTP-layer caching

## What it means

A cache is a bet that a stale answer is an acceptable trade for speed, and caching correctness means that bet is made deliberately, with an explicit staleness window and a real invalidation path — not implicitly, by adding a cache and hoping. Every write path that affects cached data needs a corresponding invalidation or update to the cache, or the cache silently starts returning wrong answers with no error, no exception, and no obvious symptom beyond "the data seems old." The other failure mode at scale is the cache stampede: when a popular cache entry expires, many concurrent requests can all simultaneously find it missing and all hit the underlying data source at once, turning a cache meant to reduce load into a synchronized spike of load at exactly the wrong moment. Key design matters too — a cache key that's insufficiently specific (missing a tenant ID, a locale, a permission level) can return one user's or one context's cached data to another.

## Why it matters

A stale cache doesn't fail loudly — it returns a plausible, well-formed, wrong answer, and the caller has no way to tell the difference between a fresh and a stale result without checking the source of truth directly, which defeats the purpose of caching in the first place. This makes cache bugs some of the hardest in this guide to trace: a customer sees the wrong balance, the wrong price, or someone else's data, and the investigation has to establish that a cache is even involved before it can find which write path failed to invalidate it. A cache key collision across tenants or users is the sharpest version of this — it's not just staleness, it's one party seeing another party's data, which is a security-adjacent failure wearing a performance-optimization costume.

## What good looks like

- Every write path that changes data covered by a cache also invalidates or updates the corresponding cache entry, as part of the same logical operation.
- Cache keys include every dimension that affects the value (tenant, user, locale, permission level) — never a key that's correct for one context and wrong for another that happens to share it.
- TTLs are chosen deliberately based on how stale the data is acceptable to be for this specific use case, not copied from an unrelated cache's default.
- High-traffic cache entries use a stampede-prevention mechanism (a lock/single-flight on cache miss, staggered TTLs, background refresh) rather than letting every concurrent miss hit the source simultaneously.
- The cache is treated as a derived, disposable optimization — the system remains correct (just slower) if the cache were cleared entirely.

## Violation signatures

- A write/update method with no corresponding cache invalidation call for data it just changed.
- A cache key built from a subset of the dimensions that actually affect the cached value (missing tenant ID, locale, or similar).
- A hardcoded or copy-pasted TTL with no comment or reasoning tying it to this specific data's actual staleness tolerance.
- A cache-aside pattern with no protection against a stampede on a hot key's expiry.
- Application logic that would behave incorrectly (not just slower) if the cache were disabled — a sign the cache has become a de facto source of truth rather than an optimization.
- A distributed cache shared across environments (dev, staging, prod) with no key namespacing, risking cross-environment data leakage.

## Code: violation → fix

```java
// Violation: update path has no cache invalidation — reads keep returning
// the old balance until the TTL happens to expire
void updateBalance(String accountId, BigDecimal newBalance) {
    accountRepo.updateBalance(accountId, newBalance); // cache still has the old value
}

BigDecimal getBalance(String accountId) {
    return cache.get("balance:" + accountId, () -> accountRepo.findBalance(accountId));
}
```

```java
// Fix: the write path invalidates the cache as part of the same operation
void updateBalance(String accountId, BigDecimal newBalance) {
    accountRepo.updateBalance(accountId, newBalance);
    cache.evict("balance:" + accountId); // next read repopulates from the source of truth
}

BigDecimal getBalance(String accountId) {
    return cache.get("balance:" + accountId, () -> accountRepo.findBalance(accountId));
}
```

The fix ties the invalidation to the write, so a read immediately after an update reflects the new value rather than serving a stale cached one until an unrelated TTL happens to expire.

## Review checklist

1. Does every write path to cached data include a corresponding cache invalidation or update?
2. Does the cache key include every dimension (tenant, user, locale) that actually affects the cached value?
3. Was the TTL chosen deliberately for this data's staleness tolerance, or copied from elsewhere?
4. Is there stampede protection on high-traffic keys, or can many concurrent misses all hit the source at once?
5. Would the system still be *correct* (just slower) if this cache were disabled entirely?

## How AI-generated code violates this

A model adding a cache to speed up a read path will reliably implement the read-through logic correctly — that's the explicit, stated part of the task — and just as reliably miss updating every existing write path that affects the same data, because those write paths live in different methods or files that weren't in scope for "add caching to this endpoint," and nothing forces the model to search for all of them. This is a specific case of [silent scope creep's inverse](../cross-cutting/ai-code-failure-modes.md#silent-scope-creep-into-files-that-werent-part-of-the-task): here the problem is scope that *should* have expanded (to every write path touching this data) but didn't, because the task as stated only mentioned the read side. Cache-key dimensionality bugs show up for a related reason — a key built from the arguments visible to the function being cached, without considering context (the current tenant, the current user) that lives elsewhere and wasn't passed in explicitly.

## Guardrail snippet

```
When adding a cache to a read path, find every existing write path that
can change the cached data and add invalidation to each of them as part
of the same change — do not scope the change to the read side only.
Include every dimension that affects the cached value (tenant, user,
locale, permission) in the cache key. Choose the TTL deliberately for
this data's actual staleness tolerance. Add stampede protection for any
cache key expected to see concurrent traffic on miss.
```

## Scoring

- **0 — Violated:** a write path exists with no corresponding cache invalidation, or a cache key omits a dimension that causes cross-context data leakage.
- **1 — Partial:** invalidation exists for the primary write path, but a secondary one (batch update, admin action) was missed.
- **2 — Met:** every write path invalidates correctly; keys are fully dimensioned; TTLs are deliberate.
- **3 — Exemplary:** invalidation is structurally coupled to writes (a single repository method that updates and invalidates together) so a future write path can't be added without it.

## Related

- [P-08 Transaction & Data Integrity Boundaries](../crucial/08-transaction-data-integrity.md) — a cache invalidation that isn't atomic with its write can leave a window where the cache disagrees with the source of truth.
- [P-27 Algorithmic Complexity & Access Patterns](27-algorithmic-complexity-access-patterns.md) — caching is sometimes reached for to mask an N+1 or expensive-query problem instead of fixing the access pattern itself.
- [P-06 Idempotency & Retry Safety](../crucial/06-idempotency-retry-safety.md) — a stampede-prevention lock on cache miss has the same coordination shape as an idempotency check.

## Going deeper

- Kleppmann, *Designing Data-Intensive Applications*, ch. 3 (on derived data and caching as materialized views of a source of truth).
- Karumanchi or standard distributed-systems references on the "cache stampede"/"thundering herd" problem and single-flight mitigation.
- AWS/Redis vendor documentation on cache invalidation strategies (write-through, cache-aside, TTL-based) — practical patterns for this exact tradeoff.
