# P-20 · Naming & Readability

> A name tells you what a thing is or does without needing to open it.

**Tier:** High · **Scope:** every identifier — variables, methods, classes, parameters — but especially public API surfaces and anything a reviewer sees without full context

## What it means

A name is a compression of intent — it's the interface a reader interacts with before they read a single line of implementation, and a good one makes reading the implementation optional for understanding what a thing does. `processData(Object data)` compresses nothing; it tells you exactly as much as the code's structure already did. `applyLoyaltyDiscount(Order order)` tells you the business intent, which is the part that isn't otherwise visible from the type signature alone. Boolean parameters are a specific, common readability failure: `schedule(true, false)` at a call site conveys nothing to a reader without navigating to the method definition, whereas a named parameter or enum makes the call site self-explanatory. Comments used to compensate for a bad name (`// this actually validates the order too` above a method called `getOrder`) are a symptom, not a fix — the name is still lying, and the comment is one refactor away from also being wrong.

## Why it matters

Poor naming is a tax paid by every future reader, not a one-time cost — every person who touches this code, human or AI, spends extra time and extra risk of misunderstanding reconstructing intent that a better name would have conveyed for free. This compounds specifically in AI-assisted workflows: a model editing a codebase relies heavily on names to infer intent when it can't hold the whole system in context, so a misleading name (`validate()` that actually mutates state, `getX()` with a side effect) doesn't just slow a human down, it actively misleads an agent making an edit elsewhere into an incorrect assumption about what's safe to change or reuse.

## What good looks like

- Method names describe what they do, including side effects — a `get`-prefixed method doesn't mutate state; a `validate`-prefixed method doesn't also persist.
- Boolean parameters that aren't self-evident at the call site are replaced with a named type, enum, or a static factory method name that conveys the choice.
- Names reflect domain vocabulary the business actually uses, not generic technical nouns (`Manager`, `Handler`, `Processor`, `Data`, `Info`) that could mean anything.
- A variable's name reflects what it holds well enough that a type-inferred (`var`) declaration is still readable without the type visible.
- Names are consistent for the same concept across the codebase — the same thing isn't called `userId` in one file and `accountId` in another when they mean the same thing.

## Violation signatures

- Generic suffixes/names with no qualifying context: `DataManager`, `processHelper()`, `handleStuff()`, `info`, `temp`, `obj`.
- A boolean parameter at a call site with no clue what it means (`save(true, false)`).
- A `get`/`is`/`has`-prefixed method with a side effect, or a method whose name implies read-only behavior but that mutates state.
- A comment directly above a method or variable that exists only to explain what its name should have conveyed.
- The same concept named differently across files (`userId` vs. `accountId` vs. `uid` for the same underlying identifier).
- Abbreviations that aren't domain-standard and require the reader to guess (`calcTtlAmt`, `procOrdReq`).
- A class or method whose name doesn't match what it actually does after a refactor — the name is now stale relative to the behavior.

## Code: violation → fix

```java
// Violation: generic name, unclear booleans, a "get" that has a side effect
class DataManager {
    Object getData(boolean flag1, boolean flag2) {
        Object result = fetch();
        if (flag1) cache.put(result); // a "get" that mutates the cache — surprising
        return flag2 ? transform(result) : result;
    }
}
```

```java
// Fix: intent-revealing name, no mystery booleans, side effects are explicit
class OrderRepository {
    Order findOrder(String orderId, FetchOptions options) {
        Order order = fetchFromDatabase(orderId);
        if (options.cacheResult()) {
            cacheOrder(order); // the mutation is named, not hidden inside "getData"
        }
        return options.applyPricingRules() ? withComputedPricing(order) : order;
    }
}

record FetchOptions(boolean cacheResult, boolean applyPricingRules) {
    static FetchOptions defaults() { return new FetchOptions(false, false); }
}
```

The fix makes the call site self-explanatory (`findOrder(id, FetchOptions.defaults())` vs. a bare `getData(true, false)`), and separates the read (`findOrder`) from its side effect (`cacheOrder`) so a reader isn't surprised that a "getter" changed state.

## Review checklist

1. Does a method's name accurately describe everything it does, including side effects?
2. Are there boolean parameters that aren't self-explanatory at the call site?
3. Is a generic name (`Manager`, `Handler`, `Data`, `Processor`, `Info`) used where a domain-specific one would convey actual intent?
4. Is there a comment that exists only to explain what the name should have conveyed?
5. Is the same underlying concept named inconsistently across files in this diff?

## How AI-generated code violates this

Generic names are one of the most reliable tells of AI-generated code — `processRequest`, `handleData`, `manager`, `helper` — because these names are statistically extremely common across the training distribution as placeholder-shaped identifiers for "some function that does the task," and producing a domain-specific name requires the model to have internalized the actual business vocabulary of *this* codebase rather than defaulting to generic software vocabulary that fits any codebase. Boolean-parameter sprawl compounds this specific to naming: when a generated method needs a second mode, appending a boolean parameter is the smallest diff, and it's a naming problem as much as an API-design one — see [P-14 Contract & API Design](14-contract-and-api-design.md) for the signature-level version of the same issue. Confidently generic names are also a case where nothing *looks* wrong — the code compiles, runs, and reads as plausible engineering — which is exactly the [plausible-but-wrong](../cross-cutting/ai-code-failure-modes.md#plausible-but-wrong-logic-that-reads-correctly) pattern applied to naming instead of logic.

## Guardrail snippet

```
Never name a class, method, or variable with a generic term (Manager,
Handler, Processor, Data, Info, Helper, Util) unless it's genuinely
generic infrastructure — use the actual domain vocabulary for this
codebase instead. A get/is/has-prefixed method must not have a side
effect; name the side effect explicitly if one exists. Replace
non-obvious boolean parameters with a named options type or enum. Use
the same name for the same concept consistently across the codebase.
```

## Scoring

- **0 — Violated:** a name actively misleads about behavior (a "get" with a side effect, a generic name hiding a specific responsibility).
- **1 — Partial:** most names are clear, but at least one generic name, mystery boolean, or inconsistent naming exists in this diff.
- **2 — Met:** names accurately and specifically describe behavior including side effects; boolean parameters are self-explanatory.
- **3 — Exemplary:** naming consistently reflects a shared domain vocabulary (a ubiquitous language) that a new reader could learn from the names alone, without a glossary.

## Related

- [P-14 Contract & API Design](14-contract-and-api-design.md) — a signature's names are part of its contract; a misleading name is a contract violation as much as a style issue.
- [P-30 Documenting the "Why"](../medium/30-documenting-why.md) — a comment compensating for a bad name is treating a symptom; fixing the name is treating the cause.
- [P-09 Separation of Concerns / SRP](../crucial/09-separation-of-concerns-srp.md) — a name that's hard to write clearly and specifically is often a sign the thing being named has more than one responsibility.

## Going deeper

- Martin, *Clean Code*, ch. 2 (Meaningful Names).
- McConnell, *Code Complete*, 2nd ed., ch. 11 (The Power of Variable Names).
- Evans, *Domain-Driven Design*, ch. 2 — Ubiquitous Language as the source names should be drawn from.
