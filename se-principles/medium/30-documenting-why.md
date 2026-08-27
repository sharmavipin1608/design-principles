# P-30 · Documenting the "Why"

> A comment earns its place by explaining a decision the code itself can't — never by restating what the code already says.

**Tier:** Medium · **Scope:** comments and documentation attached to non-obvious decisions, workarounds, and constraints

## What it means

Code can tell a reader *what* it does — that's what reading the code is for. It can't tell a reader *why* a particular approach was chosen over an equally readable alternative, why a magic-looking number is what it is, or what external constraint forced an otherwise-odd decision. A comment is justified when it captures one of those: a workaround for a specific bug in a specific dependency version, a business rule that isn't derivable from the code alone ("regulation X requires this to round up, not to nearest"), a non-obvious performance or correctness reason for choosing a less-obvious implementation. A comment that describes what the next three lines do is, at best, redundant with the code and, at worst, a lie waiting to happen — it has to be manually kept in sync with the code forever, and nothing enforces that it will be.

## Why it matters

A comment describing *what* code does goes stale the moment the code changes and nobody updates the comment, because nothing connects them mechanically — a compiler doesn't check that a comment still matches the logic below it. A stale comment is worse than no comment, because a reader trusts it by default and it actively misleads them into a wrong understanding of what the code does, right up until they trace through the logic themselves and discover the discrepancy — at which point they've lost time and, more insidiously, they start distrusting every other comment in the file too. A comment describing *why*, by contrast, doesn't go stale in the same way: even if the code around it changes, "this rounds up because of regulation X" stays true and useful as long as the regulation does, independent of the code's exact shape.

## What good looks like

- Comments explain a non-obvious constraint, workaround, or business rule that the code's structure can't convey by itself.
- A workaround for a specific bug names the bug (a ticket number, a library version, a linked issue) so a future reader knows when it's safe to remove.
- Magic numbers with a non-obvious source (a regulatory threshold, a third-party API's undocumented limit) are commented with that source, not just labeled as a constant.
- Comments are updated in the same change that changes the code they describe, or removed if they no longer apply.
- The absence of a comment on straightforward code is treated as normal, not as a coverage gap to fill.

## Violation signatures

- A comment that restates the line below it in English (`// increment count` above `count++`).
- A comment describing behavior that no longer matches the code beneath it (a stale comment).
- A magic number or non-obvious threshold with no comment explaining where it came from.
- A workaround with no reference to what it's working around — a future reader can't tell if it's still needed.
- Javadoc that repeats the method signature in prose with no additional information (covered further in [P-33 Trivial Documentation](../low/33-trivial-documentation.md), the low-tier automatable version of this same failure).
- Commented-out code left in place with no explanation of why it's preserved instead of deleted.

## Code: violation → fix

```java
// Violation: comments restate the code; the actually important context
// (why 1.15, why this order) is missing entirely
// multiply amount by 1.15
BigDecimal total = amount.multiply(new BigDecimal("1.15"));
// check if user is eligible
if (user.getTier() >= 3) {
    applyDiscount(total);
}
```

```java
// Fix: no comment needed for the obvious steps; the comment that remains
// explains the one thing the code can't say for itself
// 15% VAT rate per EU Directive 2006/112/EC, Art. 96 — do not change without
// checking the current applicable rate for the customer's jurisdiction
BigDecimal total = amount.multiply(VAT_MULTIPLIER);
if (user.getTier() >= 3) {
    applyDiscount(total);
}
```

The fix removes comments that added nothing beyond what the code already said, and keeps the one comment that captures information genuinely invisible in the code — a legal citation explaining why `1.15` is the multiplier and a warning about what changing it requires checking.

## Review checklist

1. Does this comment restate what the code already says, or does it add information the code can't convey?
2. Does this comment still accurately describe the code beneath it?
3. Is there a magic number or threshold with a non-obvious source and no comment explaining it?
4. Does a workaround comment name what it's working around, so a future reader can tell when it's safe to remove?
5. Is commented-out code left in place with no explanation for why it wasn't deleted?

## How AI-generated code violates this

Models generate line-by-line explanatory comments readily, because that pattern is heavily represented in tutorial and documentation-style training data, and it satisfies a surface-level notion of "well-documented code" without requiring the model to distinguish between information the code already conveys and information it doesn't — restating is easier to generate reliably than identifying a genuine non-obvious rationale, which requires actually knowing *why* a decision was made, something the model often doesn't know any better than a reader would from the code alone. This connects directly to [confidently wrong comments and docstrings](../cross-cutting/ai-code-failure-modes.md#confidently-wrong-comments-and-docstrings): a model will sometimes generate a comment describing intended behavior with total confidence even when it doesn't match what the code actually does, because the comment is produced as a plausible accompanying text for the code's shape, not verified against its actual control flow.

## Guardrail snippet

```
Do not write a comment that restates what the code below it already says.
Write a comment only to capture a non-obvious constraint, business rule,
workaround, or the source of a magic number — something the code's
structure can't convey by itself. When referencing a workaround, name
what it's working around (a bug, a library limitation) so a future reader
can tell when it's safe to remove. Verify a comment matches the code's
actual behavior before writing it — never describe intended behavior
without confirming the implementation does what the comment claims.
```

## Scoring

- **0 — Violated:** a comment is stale (contradicts the code) or is pure restatement with zero added information, on a non-trivial decision.
- **1 — Partial:** most comments are useful, but at least one restates the obvious or one non-obvious decision is left uncommented.
- **2 — Met:** comments consistently capture rationale, not mechanics, and none are stale.
- **3 — Exemplary:** non-obvious decisions are documented with enough context (a linked ticket, a citation, a dated rationale) that a reader years later can judge whether the reasoning still holds.

## Related

- [P-20 Naming & Readability](../high/20-naming-and-readability.md) — a comment compensating for an unclear name is treating a symptom; fixing the name is treating the cause.
- [P-33 Trivial Documentation](../low/33-trivial-documentation.md) — the same failure applied to formal docs (Javadoc) rather than inline comments; that page treats it as automatable, this one as a judgment call.
- [P-09 Separation of Concerns / SRP](../crucial/09-separation-of-concerns-srp.md) — a method that needs many comments to explain its steps is often a sign it's doing more than one thing and should be split instead.

## Going deeper

- Martin, *Clean Code*, ch. 4 (Comments) — the case for treating most comments as a failure to express intent in code.
- McConnell, *Code Complete*, 2nd ed., ch. 32 (Self-Documenting Code) — commenting rationale vs. mechanics.
- Ousterhout, *A Philosophy of Software Design*, ch. 13 — comments as a way to document design decisions the code structure can't capture.
