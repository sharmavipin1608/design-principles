# P-34 · Micro-optimizations

> Performance work is driven by a measurement, not a hunch — and most code doesn't need it at all.

**Tier:** Low · **Scope:** local performance tweaks (loop unrolling, avoiding boxing, string-concatenation choices) below the algorithmic level

**Related but distinct:** algorithmic/access-pattern cost (N+1 queries, O(n²) scans) is [P-27](../medium/27-algorithmic-complexity-access-patterns.md), a Medium-tier concern — that's about the *shape* of the cost curve. This page is about micro-level constant-factor tweaks, which almost never matter until proven otherwise.

## What it means

A claimed performance improvement below the algorithmic level (using a `StringBuilder` instead of concatenation in a cold path, avoiding a boxed `Integer`, manually inlining a call) needs a profiler result or benchmark showing it matters for this specific code path, under a realistic load. Absent that, it's a readability cost paid for a benefit nobody has demonstrated exists.

## Why it matters

Micro-optimizing a cold path (code that runs rarely, or where the actual bottleneck is elsewhere — a network call, a database query) trades real readability for imaginary speed. Knuth's line about premature optimization exists precisely because engineers are notoriously bad at guessing where the actual bottleneck is without measuring.

## What good looks like

- Performance changes are backed by a profiler flame graph, a benchmark (JMH or equivalent), or a production metric showing the specific path is actually hot.
- Hot paths identified by measurement get targeted, documented optimization; everything else stays as readable as possible.

## Violation signatures

- A performance-motivated change with no attached measurement.
- Micro-optimization applied to a path that isn't called frequently or isn't the actual bottleneck (e.g., optimizing a startup-only code path).
- A comment claiming a performance benefit ("faster this way") with no evidence.

## Code: violation → fix

Not applicable in the usual sense — the "fix" is a measurement, not a code change. If a PR claims a performance win, ask for the before/after benchmark; if none exists, the claim (and any resulting readability cost) should be reverted until one does.

## Review checklist

1. Is there a profiler/benchmark result attached to this performance change, or just a claim?
2. Is the "optimized" path actually one that runs frequently enough to matter?
3. Would reverting this to the more readable version measurably regress anything?

## How AI-generated code violates this

Models sometimes add micro-optimizations reflexively when a prompt mentions "performance" or "efficiency," reaching for patterns that look fast (manual loops instead of streams, primitive arrays instead of collections) without any actual measurement establishing that the path in question is hot — the same [plausible-but-wrong](../cross-cutting/ai-code-failure-modes.md#plausible-but-wrong-logic-that-reads-correctly) pattern, applied to a performance claim instead of a correctness one.

## Guardrail snippet

```
Do not micro-optimize (avoid boxing, hand-roll loops, avoid string
concatenation) without a profiler or benchmark result showing the
specific path is measurably hot. Prefer the more readable implementation
by default; optimize only a path proven to matter, and say what measurement
justified it.
```

## Scoring

- **0 — Violated:** a readability-costly micro-optimization ships with no measurement behind it.
- **2 — Met:** any non-trivial optimization is backed by a profiler/benchmark result.
- **3 — Exemplary:** hot paths are tracked by an ongoing benchmark suite in CI, so regressions and improvements are both measured automatically.

## Related

- [P-27 Algorithmic Complexity & Access Patterns](../medium/27-algorithmic-complexity-access-patterns.md) — the Medium-tier sibling: algorithmic shape matters at any scale; micro-level constant factors almost never do until measured.
- [P-22 KISS / Complexity Budget](../medium/22-kiss-complexity-budget.md) — an unmeasured micro-optimization is usually a straightforward KISS violation with no offsetting benefit.

## Going deeper

- Knuth, *"Structured Programming with go to Statements"* — origin of "premature optimization is the root of all evil."
- Shipilev / JMH documentation — the standard tool for producing a real Java micro-benchmark instead of a guess.
