# P-28 · Codebase Convention Consistency

> New code matches the idioms already established around it, even when a different idiom is equally valid in isolation.

**Tier:** Medium · **Scope:** any new code added to an existing codebase — error handling style, naming conventions, layering patterns, testing idioms

## What it means

A codebase accumulates conventions over time — how errors are surfaced, how DTOs are named, whether validation happens via annotations or manual checks, how tests are structured — and consistency means new code follows the conventions already in place, even when the new author (or agent) might have chosen differently starting from scratch. This isn't about one style being objectively superior; it's that a codebase with one consistent way of doing something is easier to read, search, and modify than one with three equally valid but different ways scattered throughout, because a reader has to relearn the local dialect every time they cross a boundary between styles. This principle explicitly yields to a higher-tier one when they conflict — a locally established pattern that violates a Crucial or High-tier principle should be fixed, not matched — but for genuinely stylistic choices, match what's there.

## Why it matters

Inconsistency compounds specifically because it removes the ability to pattern-match: a developer who has learned "this codebase always validates in the controller with `@Valid`" can review a hundred controllers quickly using that assumption, but the moment a handful of controllers use a different, equally valid validation style, every review of every controller now requires re-checking which style this particular one uses. This is a Medium-tier concern rather than a cosmetic one because it directly increases the surface area for the Crucial-tier mistakes to slip through — a reviewer scanning for "is validation missing" moves faster and more reliably across a codebase with one validation idiom than one with three.

## What good looks like

- New code's error-handling style (exception types, response shapes) matches what's already established for the same kind of operation elsewhere in the codebase.
- Naming conventions (for DTOs, test methods, package structure) match the surrounding code, even where a different convention is also defensible.
- A new class in a given layer follows that layer's established responsibilities and patterns, rather than importing a pattern from a different part of the codebase or a different codebase entirely.
- Where an existing convention is genuinely bad (violates a higher-tier principle), the fix is proposed as a deliberate, visible change — not silently diverged from in just the new code.
- Test structure (naming, setup/teardown style, assertion library) matches what the rest of the test suite already uses.

## Violation signatures

- A new controller using manual `if`-based validation when every other controller in the codebase uses bean validation annotations, or vice versa.
- A new module's error responses shaped differently from every existing endpoint's error response.
- A new test file using a different assertion library or naming convention than the rest of the test suite.
- A new class introducing a different logging framework/pattern than what's used everywhere else.
- Import of a pattern that looks like it was learned from a different, unrelated codebase — a naming scheme, folder layout, or idiom that doesn't match anything else nearby.
- A PR that silently reformats or restyles pre-existing code near the change to match a different convention, without that being the stated purpose of the PR.

## Code: violation → fix

```java
// Violation: every other controller in this codebase returns a shared
// ApiError DTO on failure; this new one invents its own shape
@PostMapping("/invoices")
ResponseEntity<?> createInvoice(@RequestBody InvoiceRequest req) {
    if (req.amount() == null) {
        return ResponseEntity.badRequest().body(Map.of("problem", "amount missing")); // new, one-off shape
    }
    // ...
}
```

```java
// Fix: matches the established shape used by every other endpoint
@PostMapping("/invoices")
ResponseEntity<?> createInvoice(@Valid @RequestBody InvoiceRequest req) {
    // validation happens via @Valid + bean validation annotations, same as elsewhere
    Invoice invoice = invoiceService.create(req);
    return ResponseEntity.ok(InvoiceView.from(invoice));
}
// on failure, the codebase's existing @ExceptionHandler maps validation
// errors to the shared ApiError shape automatically, same as every other endpoint
```

The fix isn't "more correct" in isolation — the manual check would have worked — it's that a client integrating with this API now gets the same error shape from every endpoint, and a developer reading this controller recognizes the pattern instantly instead of learning a one-off convention just for this file.

## Review checklist

1. Does this error-handling shape match what's already established elsewhere in the codebase for the same kind of operation?
2. Does naming (classes, tests, variables) match the surrounding convention, even where an alternative would also be valid?
3. Is a pattern being introduced here that doesn't appear anywhere else nearby, without a stated reason?
4. If an existing local convention is genuinely bad, is the fix a deliberate, visible change rather than a silent one-off divergence?
5. Does the test structure in this diff match the rest of the suite's style?

## How AI-generated code violates this

A model generating code for one file at a time often defaults to whatever pattern is most common across its training distribution in general, rather than the specific pattern already established in the surrounding codebase, unless it's explicitly grounded in the local context — this is the most direct cause of the [inconsistency-across-turns](../cross-cutting/ai-code-failure-modes.md#inconsistency-across-turns--the-same-concept-implemented-two-different-ways-in-one-session) failure mode: without a persistent memory of "this codebase does X," each generation re-derives its own default, and that default is shaped by the global training distribution, not this one repository's history. It shows up most visibly when a model has recently seen a different codebase or a different convention within the same session (a different project's error-handling pattern, a different test framework) and carries that pattern over into unrelated work, producing code that's individually well-formed but locally foreign.

## Guardrail snippet

```
Before implementing a new controller, service, test, or DTO, look at two
or three existing examples of the same kind of thing in this codebase and
match their conventions — error handling shape, naming, validation style,
test structure — even if a different approach would also be valid in
isolation. If an existing convention appears to violate a higher-tier
principle (security, correctness, error handling), flag it explicitly
rather than silently diverging from it in just the new code.
```

## Scoring

- **0 — Violated:** new code introduces a materially different pattern (error shape, validation style) than every comparable existing example, with no stated reason.
- **1 — Partial:** most conventions match, but one aspect (naming, test style) diverges from the local norm.
- **2 — Met:** new code matches established local conventions consistently.
- **3 — Exemplary:** the change also documents or codifies the convention (a shared base class, a lint rule, a style guide entry) so future contributors — human or AI — inherit it automatically.

## Related

- [P-20 Naming & Readability](../high/20-naming-and-readability.md) — naming consistency across the codebase is a specific instance of this broader convention-matching principle.
- [P-23 DRY (Rule of Three)](23-dry-rule-of-three.md) — an inconsistent reimplementation of an existing pattern is often also an undetected duplication.
- [P-14 Contract & API Design](../high/14-contract-and-api-design.md) — a consistent error-response shape across endpoints is itself part of the API's overall contract.

## Going deeper

- Martin, *Clean Code*, ch. 17 (Smells and Heuristics) — the general case for consistency as a readability property.
- Google, *"Google Java Style Guide"* — a concrete example of a documented, enforced convention set at scale.
- Ousterhout, *A Philosophy of Software Design*, ch. 4 — "deep modules" and consistent interfaces as a way of keeping a growing system learnable.
