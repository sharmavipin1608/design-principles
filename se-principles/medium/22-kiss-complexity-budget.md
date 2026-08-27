# P-22 · KISS / Complexity Budget

> The simplest design that actually satisfies the requirement wins — every unit of complexity has to earn its place.

**Tier:** Medium · **Scope:** every method and class, but especially ones with deep nesting, long parameter lists, or clever one-liners

## What it means

Complexity isn't free even when it's correct — every conditional branch, every level of nesting, every layer of indirection is something a reader has to hold in their head to understand what the code does, and complexity compounds combinatorially, not additively: a method with four independent boolean conditions has sixteen possible paths through it, whether or not the author was thinking about all sixteen. KISS means treating simplicity as a budget you spend deliberately, not a nice-to-have you get around to — preferring the boring, obvious implementation over the clever one, flattening nested conditionals with early returns, and splitting a long method along its natural seams rather than trying to hold the whole thing in one block. "Clever" code that requires a comment explaining what it's doing is usually complexity that should have been simplicity instead.

## Why it matters

Complexity is where bugs hide, specifically because untested paths are more likely to exist in complex code and more likely to go unnoticed — a method with cyclomatic complexity in the double digits has a combinatorial number of paths, and a typical test suite covers a linear number of them, so the ratio of untested-to-tested paths grows with complexity, not despite it. It's also where the cost of *changing* the code lives: a simple method can be modified with local reasoning, but a deeply nested one requires the modifier to hold the entire branching structure in mind to be confident a change in one branch doesn't have unintended interaction with another — which is exactly the condition under which changes introduce regressions.

## What good looks like

- Guard clauses and early returns replace deep nesting — a method reads top-to-bottom without needing to track five levels of indentation.
- A method does one thing at one level of abstraction; if it mixes "what" and "how" at different granularities, it's split.
- The obvious, boring implementation is chosen over a clever one unless the clever one has a measured, necessary performance reason (see [P-34 Micro-optimizations](../low/34-micro-optimizations.md)).
- Cyclomatic complexity and nesting depth are kept low enough that the method's logic can be described in a sentence, not a paragraph.
- Complexity that is genuinely necessary (a real business rule with many cases) is organized so each case is locally simple, even if the aggregate isn't.

## Violation signatures

- Nesting depth of four or more levels in a single method.
- A method whose cyclomatic complexity (independent path count) would take real effort to enumerate by eye.
- A one-liner that requires a comment to explain what it does — the comment is compensating for chosen cleverness.
- Boolean flags combined with `&&`/`||` chains spanning multiple conditions with no named intermediate variable explaining what the combination means.
- A method over roughly 40–50 lines doing several visibly different things in sequence with no natural single name.
- Reimplementing something the standard library already provides, in a less-tested, more clever way.

## Code: violation → fix

```java
// Violation: deep nesting, several concerns interleaved, hard to trace
void processOrder(Order order) {
    if (order != null) {
        if (order.getItems() != null && !order.getItems().isEmpty()) {
            if (order.getCustomer().isActive()) {
                if (inventoryService.hasStock(order)) {
                    // ... actual processing, four levels deep ...
                }
            }
        }
    }
}
```

```java
// Fix: guard clauses flatten the structure; each precondition is a readable line
void processOrder(Order order) {
    if (order == null || order.getItems() == null || order.getItems().isEmpty()) return;
    if (!order.getCustomer().isActive()) return;
    if (!inventoryService.hasStock(order)) return;

    // ... actual processing, at the top level, one thing at a time ...
}
```

The fix doesn't change what's checked — it changes how much a reader has to hold in their head at once, since each guard clause fully resolves one precondition instead of nesting all of them together.

## Review checklist

1. Is there nesting four or more levels deep that could be flattened with guard clauses?
2. Would you need more than a sentence to describe what this method does?
3. Does a one-liner need a comment to explain what it's doing?
4. Are there boolean expressions combining multiple conditions with no named intermediate explaining the combination's meaning?
5. Is there a simpler, more boring way to write this that a new team member would immediately understand?

## How AI-generated code violates this

Models are capable of producing genuinely clever, dense code — comprehension-heavy stream chains, nested ternaries, single-expression solutions to multi-step problems — because that pattern is well-represented in training data as a signal of skill, and it satisfies the immediate correctness bar while looking impressive. The cost of that density (a future reader, human or another agent, having to unpack it) isn't visible to the model in the moment it's generated, since nothing in a single-turn correctness check penalizes cleverness. This connects to [inconsistency across turns](../cross-cutting/ai-code-failure-modes.md#inconsistency-across-turns--the-same-concept-implemented-two-different-ways-in-one-session): complexity that would be caught and simplified by a human re-reading their own code the next day often survives in agent-generated code because there's no equivalent "sleep on it and reread" pass by default.

## Guardrail snippet

```
Prefer the simplest implementation that satisfies the requirement over a
clever one. Use guard clauses and early returns instead of nesting more
than two or three levels deep. If a one-liner needs a comment to explain
what it does, rewrite it as multiple named steps instead. Split a method
along its natural seams once it's doing more than one clearly nameable
thing.
```

## Scoring

- **0 — Violated:** deeply nested or densely clever code that's hard to verify by inspection, with no simpler equivalent attempted.
- **1 — Partial:** the logic is mostly clear, but one section is denser or more nested than the requirement justifies.
- **2 — Met:** the implementation is the simplest reasonable one for the requirement; structure matches the reader's mental model.
- **3 — Exemplary:** genuinely complex business logic is organized so each individual piece stays simple, even though the aggregate handles real complexity.

## Related

- [P-21 YAGNI](21-yagni.md) — the two overlap significantly: unnecessary abstraction is both speculative and needlessly complex.
- [P-16 Open/Closed](../high/16-open-closed.md) — in tension: an extension point adds a layer of indirection that's more complex locally in exchange for being simpler to extend later; KISS should win until that extensibility is actually needed.
- [P-27 Algorithmic Complexity & Access Patterns](27-algorithmic-complexity-access-patterns.md) — a different sense of "complexity" (runtime cost) that sometimes trades against this one (readability).

## Going deeper

- McConnell, *Code Complete*, 2nd ed., ch. 19 (General Control Issues) and its treatment of cyclomatic complexity.
- Martin, *Clean Code*, ch. 3 (Functions) — guard clauses, single level of abstraction per function.
- Ousterhout, *A Philosophy of Software Design*, ch. 2–4 — complexity as the central problem in software design, and "deep modules" as its remedy.
