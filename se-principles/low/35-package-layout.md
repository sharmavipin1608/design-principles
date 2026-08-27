# P-35 · Package Layout & Aesthetics

> Package structure being consistent matters far more than which convention was chosen.

**Tier:** Low · **Scope:** package/folder organization — layer-based (`controller/`, `service/`, `repository/`) vs. feature-based (`orders/`, `users/`)

## What it means

Layer-based and feature-based packaging are both legitimate; each has tradeoffs (layer-based groups by technical role, feature-based groups by business capability and scales better as a codebase grows). Neither is objectively correct. What actually matters is that the codebase picks one and applies it consistently — a mixed codebase where some features are packaged by layer and others by feature is worse than either pure approach.

## Why it matters

Inconsistent package layout costs a few seconds of "where does this go" confusion per file, repeated over the life of the codebase — real but small, which is exactly why this is Low-tier: it's never worth a review comment mid-PR, only worth a one-time team decision applied going forward.

## What good looks like

- One packaging convention is documented and followed throughout the codebase.
- A new feature's files land in the structure an existing, similar feature already established.

## Violation signatures

- A new feature packaged by layer when every other feature in the codebase is packaged by feature capability, or vice versa.
- Package names that don't reflect either convention clearly (a grab-bag `misc/` or `common/` package with unrelated classes).

## Code: violation → fix

Not applicable — this is a file/folder organization concern, not a code diff.

## Review checklist

1. Does this change's file placement match the codebase's existing, established packaging convention?
2. Is there a documented convention at all, or is placement ad hoc per contributor?

## How AI-generated code violates this

A model unfamiliar with which convention a codebase has settled on may default to whatever's more common in its training distribution (often layer-based, for smaller example projects) even when the actual codebase has standardized on feature-based packaging — another instance of [convention drift](../medium/28-convention-consistency.md) applied specifically to file placement.

## Guardrail snippet

```
Place new files according to the packaging convention already established
in this codebase (check two or three existing, similar features first) —
do not introduce a different convention even if it's also valid in
isolation.
```

## Scoring

- **0 — Violated:** new code's placement contradicts the codebase's established convention.
- **2 — Met:** placement matches the established convention consistently.
- **3 — Exemplary:** the convention is documented (a CONTRIBUTING.md or architecture doc) so it doesn't rely on tribal knowledge.

## Related

- [P-28 Codebase Convention Consistency](../medium/28-convention-consistency.md) — this page is that principle applied specifically to file/folder placement.

## Going deeper

- Fowler, *"PresentationDomainDataLayering"* and related essays (martinfowler.com) — a balanced take on layer-based organization's tradeoffs.
