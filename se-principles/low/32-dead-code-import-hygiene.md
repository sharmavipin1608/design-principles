# P-32 · Dead Code & Import Hygiene

> Nothing unreachable or unused ships — and a linter, not a reviewer, is what should catch it.

**Tier:** Low · **Scope:** unused imports, unreachable branches, unused variables/parameters, dead private methods

## What it means

Static analysis tools reliably detect unused imports, unreachable code, and unused declarations at compile time or in a fast linting pass. This is a solved problem technically; the only open question is whether it's wired into your build.

## Why it matters

Dead code isn't dangerous by itself, but it's noise that makes a reader spend time understanding something that doesn't matter, and it occasionally hides a real bug (an unreachable branch that was meant to run). Neither reason justifies a human spending review time on it when a linter does it for free, every time, with no fatigue.

## What good looks like

- A linter (Checkstyle, PMD, SpotBugs, ESLint, etc.) runs in CI and fails on unused imports/dead code.
- No unreachable branches or unused private members survive to merge.

## Violation signatures

- Unused imports.
- A private method or field with no callers/readers anywhere in the codebase.
- An unreachable branch (e.g., code after an unconditional `return`/`throw`).
- An unused method parameter with no documented reason (e.g., interface conformance) for its presence.

## Code: violation → fix

Not applicable — run the configured static analyzer and let it flag/remove these automatically; don't hand-audit for them.

## Review checklist

1. Is a linter/static analyzer wired into CI and blocking on these findings?
2. Is this PR's feedback about dead code that a linter should have already caught?

## How AI-generated code violates this

Agentic edits across multiple files sometimes leave behind an import or a helper method that a since-reverted approach no longer needs — the model's working set doesn't always get swept for what's now orphaned once the final approach is settled.

## Guardrail snippet

```
Run the project's linter/static analyzer before finishing any change and
remove anything it flags as unused or unreachable. Do not leave an import,
variable, or method that a prior approach needed but the final code
doesn't.
```

## Scoring

- **0 — Violated:** no linter in CI; unused imports or dead code present.
- **1 — Partial:** a linter exists but isn't enforced (warnings only, not blocking).
- **2 — Met:** linter runs in CI and blocks on dead code/unused imports.
- **3 — Exemplary:** dead-code detection runs on every save locally (IDE-integrated), not just in CI.

## Related

- [P-31 Formatting & Style](31-formatting-and-style.md) — the same "delegate to tooling" posture applied to layout instead of content.
- [P-21 YAGNI](../medium/21-yagni.md) — speculative code that's never called is a design-level version of the same waste this tooling catches mechanically.

## Going deeper

- SonarSource, SonarQube documentation on dead-code and unused-import rules — representative of what any mainstream static analyzer already covers.
