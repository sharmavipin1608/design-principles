# P-23 · DRY (Rule of Three)

> Deduplicate knowledge, not coincidentally similar-looking code — and don't extract an abstraction until a real third case shows the shape it should take.

**Tier:** Medium · **Scope:** any duplicated logic, especially business rules, but explicitly not surface-level code shape

## What it means

DRY targets duplicated *knowledge* — the same business rule, the same validation logic, the same calculation — expressed in more than one place, such that a change to the rule requires remembering to update every copy, and inevitably one gets missed. It does not target code that merely looks similar by coincidence: two functions that happen to both loop over a list and sum a field are not a DRY violation if they represent two unrelated concepts that just happen to share shape today and might diverge tomorrow for entirely different reasons. The "Rule of Three" is the practical heuristic for telling these apart: don't extract a shared abstraction on the second occurrence, wait for the third — by then you have enough real examples to see what actually varies and what's genuinely constant, instead of guessing from two data points and building the wrong abstraction.

## Why it matters

Duplicated knowledge is a maintenance trap that looks harmless until the rule changes: a tax calculation copied into three places gets updated in two of them, and the discrepancy isn't caught by any test because each copy has its own tests asserting its own (now-inconsistent) behavior. This is Medium- rather than Crucial-tier because a single missed update usually doesn't fail immediately — it produces a quiet correctness drift that surfaces later as "why do these two reports disagree," which is exactly the kind of bug that erodes trust in a system's numbers without ever showing up as an exception. The opposite failure — extracting an abstraction from two superficially similar pieces of code that don't actually share underlying knowledge — has its own cost: the shared abstraction now has to satisfy two different, coincidentally-similar callers, and the abstraction accretes conditional branches to handle their divergence, which is often worse than the duplication it replaced.

## What good looks like

- A business rule (a calculation, a validation, an eligibility check) exists in exactly one place, called from everywhere it's needed.
- Similar-looking code is examined for whether it represents the *same concept* before being merged — coincidental similarity is left alone.
- An abstraction is extracted once a third real occurrence appears, using all three examples to determine what's actually shared versus what only looked shared with two data points.
- When two pieces of duplicated logic are found to represent the same rule, the fix moves the rule to a single named location (a method, a policy class) that both call.
- A change to a business rule is a single-file (or single-method) change, not a search-and-replace across the codebase.

## Violation signatures

- The same numeric threshold, formula, or business condition appearing as a literal in more than one file.
- Two methods that clearly implement the same validation or calculation with slightly different (and likely drifted) logic.
- A bug fix applied in one location that should have been applied in a near-identical block elsewhere but wasn't, because nobody knew the duplicate existed.
- An abstraction extracted from exactly two call sites, with a parameter or conditional inside it that exists purely to handle the one way those two callers differ (a sign the "shared" logic wasn't actually shared).
- Copy-pasted code with only variable names changed, representing the same underlying rule.

## Code: violation → fix

```java
// Violation: the same "premium eligibility" rule, expressed independently twice
class SubscriptionService {
    boolean isPremiumEligible(User user) {
        return user.getAccountAgeInDays() > 30 && user.getOrderCount() >= 5;
    }
}

class ReportingService {
    boolean qualifiesForPremiumReport(User user) {
        return user.getAccountAgeInDays() > 30 && user.getOrderCount() >= 5; // drifted risk
    }
}
```

```java
// Fix: one rule, one place, both callers use it — a future change to the
// threshold can't be applied to only one of them by mistake
class PremiumEligibilityPolicy {
    boolean isEligible(User user) {
        return user.getAccountAgeInDays() > 30 && user.getOrderCount() >= 5;
    }
}
```

The fix isn't just fewer lines — it's that a future change to the eligibility rule (say, raising the order count to 10) now has exactly one place to change, and it's structurally impossible for the two call sites to silently disagree afterward.

## Review checklist

1. Does the same business rule, threshold, or formula appear as separate logic in more than one place?
2. If two blocks look similar, do they represent the same underlying concept, or are they coincidentally shaped alike?
3. Is an abstraction being extracted from only two occurrences, and does it already need a branch to handle their one difference?
4. If this rule changes, is there exactly one place to change it, or several that would need to be found and updated together?

## How AI-generated code violates this

Because each generation pass tends to solve the task directly in front of it without a broad search for existing logic elsewhere in the codebase, a model implementing "add a premium eligibility check to the reporting service" is likely to write the check fresh rather than discovering and reusing an existing one in the subscription service — this is the same root cause as [inconsistency across turns](../cross-cutting/ai-code-failure-modes.md#inconsistency-across-turns--the-same-concept-implemented-two-different-ways-in-one-session), applied specifically to business rules instead of stylistic choices, and it's a particularly damaging instance because the two copies can drift silently. The opposite failure also shows up: a model asked to "clean up duplication" between two coincidentally similar blocks will sometimes merge them into one function with an added conditional, satisfying the *letter* of DRY while producing exactly the premature-abstraction problem this page's Rule of Three heuristic exists to prevent.

## Guardrail snippet

```
Before implementing a business rule, calculation, or validation, search
the codebase for an existing implementation of the same concept and reuse
or extend it rather than writing a new copy. When you find similar-looking
code, extract a shared abstraction only if it represents the same
underlying concept — not merely similar shape — and prefer waiting for a
third real occurrence before extracting, per the Rule of Three.
```

## Scoring

- **0 — Violated:** the same business rule is duplicated across files with no shared source, and a fix note in one location would be applied inconsistently.
- **1 — Partial:** duplication is limited to two occurrences and hasn't drifted yet, but there's no shared source.
- **2 — Met:** business rules exist in exactly one place; incidental structural similarity is correctly left unmerged.
- **3 — Exemplary:** a mechanism (a shared policy module, a single source of business constants) makes it structurally hard to reintroduce the duplication even under time pressure.

## Related

- [P-21 YAGNI](21-yagni.md) — the direct tension: DRY pushes toward abstraction, YAGNI pushes against building it before it's earned; the Rule of Three is the practical resolution.
- [P-9 Separation of Concerns / SRP](../crucial/09-separation-of-concerns-srp.md) — duplicated business rules are often a symptom of a rule with no clear single owner across layers.
- [P-28 Codebase Convention Consistency](28-convention-consistency.md) — coincidentally similar code merged into a bad shared abstraction often violates the surrounding codebase's established patterns too.

## Going deeper

- Hunt & Thomas, *The Pragmatic Programmer*, ch. 2 — the original formulation of "Don't Repeat Yourself" and the distinction from surface duplication.
- Fowler, *Refactoring*, 2nd ed. — "Duplicated Code" as a smell, and the Extract Method/Extract Class refactorings used to resolve it.
- Sandi Metz, *"The Wrong Abstraction"* (sandimetz.com) — the canonical argument for preferring duplication over the wrong shared abstraction.
