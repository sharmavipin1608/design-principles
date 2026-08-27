# P-31 · Formatting & Style

> Code layout is consistent because a tool enforces it — never because a reviewer asked for it.

**Tier:** Low · **Scope:** indentation, brace placement, line length, import ordering, whitespace

## What it means

Formatting has exactly one correct process: a formatter runs automatically (pre-commit hook or CI gate) and reformats or fails the build. There is no manual formatting review step in a healthy pipeline.

## Why it matters

Every minute spent discussing brace style in a PR comment is a minute not spent on correctness or security. Formatting debates are also uniquely pointless because they're fully automatable — there's no judgment call a formatter can't make as well as a human.

## What good looks like

- A formatter (`google-java-format`, `Spotless`, `Prettier`, etc.) runs in CI and fails the build on violations.
- No formatting-only review comments appear in PR history.

## Violation signatures

- Inconsistent indentation, brace style, or import order within the same file or across files.
- A PR comment about spacing, line breaks, or brace placement.
- No formatter configured in the build pipeline at all.

## Code: violation → fix

Not applicable — the fix is tooling, not a code diff. Run `mvn spotless:apply` / `./gradlew spotlessApply` (or your stack's equivalent) and wire the check step into CI so it can't regress.

## Review checklist

1. Is a formatter wired into CI, gating the build?
2. Is this PR's only feedback about layout, not behavior?

## How AI-generated code violates this

Generated code is usually well-formatted on its own, but different generation sessions can drift toward slightly different stylistic defaults (spacing, import grouping) absent an enforced formatter — an inconsistency a tool would erase instantly and a human reviewer shouldn't have to.

## Guardrail snippet

```
Do not hand-format code or debate formatting in review — run the
project's configured formatter before committing. If no formatter is
configured, flag that as a tooling gap rather than fixing style by hand.
```

## Scoring

- **0 — Violated:** no formatter in CI; inconsistent style across files.
- **1 — Partial:** a formatter exists but isn't enforced in CI (opt-in only).
- **2 — Met:** formatter runs in CI and blocks non-conforming code.
- **3 — Exemplary:** formatting is auto-fixed at commit time (pre-commit hook), so it's never even visible in a diff.

## Related

- [P-32 Dead Code & Import Hygiene](32-dead-code-import-hygiene.md) — the other half of "let tooling handle it."

## Going deeper

- Google, *Google Java Style Guide* — a widely adopted, fully automatable formatting spec.
