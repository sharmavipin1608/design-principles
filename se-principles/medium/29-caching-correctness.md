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

It is not, however, race-free, and it's worth knowing exactly how far it gets you. Between `updateBalance` committing and `evict` running, a concurrent reader can miss the cache, read the *old* value from a replica or a not-yet-committed snapshot, and repopulate the entry after the evict lands — re-poisoning the cache with a stale value that now survives for a full TTL. Closing that window takes one of: evicting inside the same transaction (so the invalidation commits or rolls back with the write), writing through the cache under the same lock, or accepting the window deliberately and bounding it with a short TTL. Pick one on purpose. "Evict after write" is the right *shape* and the minimum bar — it is not, by itself, a correctness guarantee under concurrency.

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

- Kleppmann, *Designing Data-Intensive Applications*, Part III (Derived Data), esp. ch. 11 — caches and indexes as derived views that must be kept consistent with a system of record.
- Nygard, *Release It!*, 2nd ed., ch. 4 ("Stability Antipatterns") — the dogpile/thundering-herd failure mode when a hot cache entry expires under load.
- Caffeine and Redis documentation on cache-aside vs. write-through and on single-flight/`refreshAfterWrite` loading — the concrete mechanisms for the mitigations named above.
- AWS/Redis vendor documentation on cache invalidation strategies (write-through, cache-aside, TTL-based) — practical patterns for this exact tradeoff.
