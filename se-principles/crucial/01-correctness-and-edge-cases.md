# P-01 · Correctness & Edge Cases

> The code produces the right output for every input it can actually receive, not just the one the author had in mind.

**Tier:** Crucial · **Scope:** any function or method with a non-trivial input domain — collections, numeric ranges, optional values, external responses

## What it means

Correctness isn't "it worked when I ran it." It's that the implementation's behavior matches the specification across the whole input domain, including the parts of the domain that are rare, boundary, or degenerate: the empty list, the single-element list, the maximum value, the value one past the maximum, the null that a type system didn't rule out, the response that came back malformed. A function that's right on the common case and wrong on the edges isn't half-correct — it's a bug with a delay timer on it, waiting for the input distribution to shift.

The discipline here is enumerating the domain before writing the body: what are all the shapes this input can take, and does the code have a deliberate answer for each one? "Deliberate" is the key word — an edge case that happens to work because of how a loop terminates is not the same as one that was considered.

## Why it matters

Edge-case bugs are disproportionately expensive because they're rare enough to escape testing and code review, common enough to eventually occur in production, and by the time they occur, they've often already corrupted state or shipped a wrong answer to a customer. An off-by-one in a billing loop doesn't crash — it silently overcharges or undercharges until someone notices the discrepancy weeks later, at which point you're reconciling ledgers, not reading a stack trace. The cost-to-fix curve here is brutal: catching it in review is a comment; catching it in production after data has been written is a data migration, a customer apology, and possibly a compliance conversation.

## What good looks like

- Every collection access is preceded by a size/emptiness check or uses an API that can't go out of bounds (`Optional`, `stream().findFirst()`).
- Boundary values (0, 1, max, max+1, negative where relevant) are visibly considered, in code or in tests.
- `null` is either impossible by construction (`Optional`, non-null parameter contracts) or explicitly handled at first use.
- Loop termination conditions are checked against the empty and single-element case, not just the "normal" case.
- Numeric operations that can overflow, divide by zero, or lose precision are guarded or use a type that can't.

## Violation signatures

- Direct indexing (`list.get(0)`, `array[0]`) with no preceding size check.
- `Optional.get()` without `isPresent()`/`isEmpty()` nearby.
- A loop that assumes at least one iteration will happen.
- String parsing (`split`, `substring`, `parseInt`) with no bounds or format check on the result.
- A `for` loop bound computed as `size() - 1` with no comment or test proving it can't go negative.
- Integer arithmetic on values that could be `Integer.MAX_VALUE`-adjacent (pagination offsets, counters, timestamps in millis vs. seconds).
- A conditional that handles exactly two of three actual states (e.g., checks `true`/`false` on a field that can also be `null`).
- Comparison logic that only tests the middle of a range, never the endpoints.

## Code: violation → fix

```java
// Violation: assumes the list is non-empty and every price is non-null
double averagePrice(List<Product> products) {
    double sum = 0;
    for (Product p : products) {
        sum += p.getPrice(); // NPE if getPrice() can return null
    }
    return sum / products.size(); // empty list -> 0.0/0 -> NaN, silently returned
}
```

```java
// Fix: handles empty input and null prices explicitly
OptionalDouble averagePrice(List<Product> products) {
    return products.stream()
        .map(Product::getPrice)
        .filter(Objects::nonNull)
        .mapToDouble(Double::doubleValue)
        .average(); // empty stream -> OptionalDouble.empty(), not a crash or a lie
}
```

The fix makes "no valid price" a representable, checkable result instead of two different silent wrong answers. Note the violation's empty-list case doesn't crash: `sum` is a `double`, so `0.0 / 0` is floating-point division yielding `NaN`, not an `ArithmeticException` — only *integer* division by zero throws. That `NaN` propagates quietly through every downstream calculation until something formats it for a user, which is strictly worse than a crash. `OptionalDouble.empty()` forces the caller to decide what "no prices" means.

## Review checklist

1. For every collection access, is there a check for empty/size, or is an API used that can't go out of bounds?
2. For every `Optional` or nullable reference, is the absent case handled at first use, not assumed away?
3. Are the boundary values (0, 1, max) visibly considered somewhere — code or test?
4. Does any loop assume at least one iteration, and is that assumption actually guaranteed?
5. Can any arithmetic here overflow, underflow, or divide by zero given realistic input ranges — and if it's floating-point, does a zero divisor produce a silent `NaN`/`Infinity` rather than an exception?
6. If this parses external input (string, JSON, CSV), does malformed input get a defined outcome instead of an uncaught exception?

## How AI-generated code violates this

LLM-generated implementations are frequently correct for the example in the prompt and untested against anything else — the model pattern-matches to the common shape of the problem and stops once that shape works. A generated pagination function will handle "page 2 of 5" fluently and mishandle the last partial page or a page number beyond the end, because the prompt's implicit example never exercised it. Generated string-parsing code especially tends to assume well-formed input, because the training examples it's drawing from are usually demonstrating the parser working, not defending against malformed input. The [tests the same generation pass writes](../cross-cutting/ai-code-failure-modes.md#tests-written-against-the-implementation-rather-than-the-requirement) typically mirror this blind spot — they exercise the same happy path the implementation was built against, so green tests provide false confidence here specifically.

## Guardrail snippet

```
Before implementing any function that takes a collection, string, or numeric
input, enumerate its edge cases: empty, single-element, maximum size, null,
and malformed. Write the handling for each explicitly — do not rely on
incidental loop behavior to "handle" empty input. Prefer Optional and
stream terminal operations that have defined empty-input behavior over
manual indexing. Never call .get(0) or Optional.get() without a preceding
presence check.
```

## Scoring

- **0 — Violated:** an unguarded index/`.get()`, or an arithmetic path that yields `NaN`/`Infinity`/an overflow on realistic input, is reachable.
- **1 — Partial:** common edge cases (empty, null) handled; boundary cases (max, off-by-one) are not.
- **2 — Met:** empty, null, and boundary cases are all explicitly handled or provably impossible.
- **3 — Exemplary:** edge cases are encoded as types (sealed results, `Optional`) so the compiler enforces handling, not just convention.

## Related

- [P-02 Input Validation & Trust Boundaries](02-input-validation-trust-boundaries.md) — validation decides what's *allowed in*; correctness decides what happens to it *once it's in*. Both are needed; neither substitutes for the other.
- [P-10 Meaningful Test Coverage](10-meaningful-test-coverage.md) — the enforcement mechanism for this principle; edge cases you didn't test are edge cases you're hoping about.
- [P-14 Contract & API Design](../high/14-contract-and-api-design.md) — a signature that makes illegal states unrepresentable (e.g., returning `Optional` instead of nullable) prevents whole classes of edge-case bugs at the call site.

## Going deeper

- Hoare, C.A.R., *"An Axiomatic Basis for Computer Programming"* — the origin of reasoning about pre/postconditions, the formal ancestor of "enumerate the domain."
- Myers, Sandler, Badgett, *The Art of Software Testing*, the Test-Case Design chapter (ch. 4 in the 3rd ed.) — boundary value analysis and equivalence partitioning.
- Bloch, *Effective Java*, 3rd ed., Item 49 ("Check parameters for validity") and Item 55 (using `Optional` judiciously).
