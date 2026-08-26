# Software Engineering Principles — Review Guide

This guide exists to judge code that an AI agent wrote, not code a junior engineer wrote — the failure modes are different, and pretending otherwise misses most of what actually goes wrong in these diffs. It works three ways: as a **manual PR review checklist** you run a diff against, as **agent guardrails** you paste into `CLAUDE.md` or a system prompt so the next generation avoids the problem instead of getting caught after the fact, and as a **personal study guide** for principles you know but haven't had to articulate recently. Thirty-five principles are organized into four tiers by how much damage a violation does and how expensive it is to unwind later, plus one cross-cutting lens specific to LLM-authored code. Every page follows the same template so you can scan it in seconds or read it in full.

## How to use it

**As a review pass.** Match your review depth to the diff, not to the guide's completeness:

- **Crucial tier — every diff, no exceptions.** These ten principles are where violations become incidents: data corruption, security holes, silent wrong answers. If you only have five minutes, spend them here.
- **High tier — anything touching module boundaries or public contracts.** New interfaces, new public methods, new API surfaces, changes to how modules talk to each other. Skip it for a change contained entirely inside one private method.
- **Medium tier — refactors and anything you're already rewriting.** These are craftsmanship principles; they matter most when code is already in motion and the cost of raising the bar is near zero.
- **Low tier — never manually.** If you're reading a diff and thinking about import order or Javadoc on a getter, stop — that's a tooling gap, not a review finding. Fix the tooling instead.

Then apply the [cross-cutting AI failure-mode lens](cross-cutting/ai-code-failure-modes.md) as a second pass, independent of tier, on any diff that smells like it was generated end-to-end from a prompt rather than incrementally by a human.

**As agent guardrails.** Each page ends with a **Guardrail snippet** — a self-contained, imperative block meant to be pasted directly into `CLAUDE.md` or a system prompt. Start with the Crucial-tier snippets; they cover the failure modes with the highest blast radius per token of prompt budget.

**As a study guide.** Read a tier top to bottom when you want to refresh the underlying reasoning, not just the checklist. The "What it means" and "Why it matters" sections are written to stand alone.

## The tier model

Tier is not a proxy for "importance" in the abstract — every principle here is worth knowing. Tier is **(blast radius if violated) × (cost to fix later)**. A missing null check (Crucial) can corrupt data in production and requires a hotfix, an incident review, and possibly a data-repair script. A one-implementation interface that violates YAGNI (Medium) costs you a few extra minutes of reading and a trivial deletion. Both are real violations. Only one of them should stop a merge.

## Crucial — violations ship incidents

| # | Principle | One-line definition | Fastest signal to check |
|---|---|---|---|
| P-01 | [Correctness & Edge Cases](crucial/01-correctness-and-edge-cases.md) | The code produces the right output for every input it can actually receive, not just the happy path. | Grep for unchecked `.get(0)`, unguarded array/list index access |
| P-02 | [Input Validation & Trust Boundaries](crucial/02-input-validation-trust-boundaries.md) | Every input is validated exactly once, at the boundary where trust changes. | Find the boundary; confirm validation happens there, not three layers deep |
| P-03 | [Security Fundamentals](crucial/03-security-fundamentals.md) | Untrusted input never reaches a query, shell, or deserializer unescaped, and every path checks authorization. | Grep for string-concatenated queries and missing authz checks on non-UI paths |
| P-04 | [Error Handling & Failure Semantics](crucial/04-error-handling-failure-semantics.md) | Failures are classified, surfaced, and handled deliberately — never swallowed. | Grep for `catch (Exception e)` with only a log line inside |
| P-05 | [Concurrency & Thread Safety](crucial/05-concurrency-thread-safety.md) | Shared mutable state is protected or eliminated; nothing races. | Find fields shared across threads with no `synchronized`, lock, or atomic |
| P-06 | [Idempotency & Retry Safety](crucial/06-idempotency-retry-safety.md) | Any operation that can be retried or delivered twice produces the same end state. | Check if a retried POST/write can duplicate a side effect |
| P-07 | [Resource Lifecycle](crucial/07-resource-lifecycle.md) | Every acquired resource has a deterministic, guaranteed release. | Grep for `new` on a `Closeable`/connection without try-with-resources |
| P-08 | [Transaction & Data Integrity Boundaries](crucial/08-transaction-data-integrity.md) | Atomicity boundaries match business intent; there is no window of partial write. | Check whether a multi-step write spans a transaction or two independent stores |
| P-09 | [Separation of Concerns / SRP](crucial/09-separation-of-concerns-srp.md) | Each unit has one reason to change. | Check if a controller/handler method contains business or persistence logic |
| P-10 | [Meaningful Test Coverage](crucial/10-meaningful-test-coverage.md) | Tests assert observable behavior against the requirement, not the implementation. | Check if a test would still fail after reverting the actual bug fix |

## High — design integrity

| # | Principle | One-line definition | Fastest signal to check |
|---|---|---|---|
| P-11 | [Coupling & Cohesion](high/11-coupling-and-cohesion.md) | Modules depend on little outside themselves, and everything inside belongs together. | Count how many unrelated modules a single change touches |
| P-12 | [Dependency Inversion & Injection](high/12-dependency-inversion.md) | Business logic depends on abstractions, supplied from outside, not concrete instances it constructs itself. | Grep for `new ConcreteService()` inside business logic |
| P-13 | [Encapsulation & Information Hiding](high/13-encapsulation-information-hiding.md) | Internal state and representation are hidden; only behavior is exposed. | Grep for public mutable fields or getters returning live collections |
| P-14 | [Contract & API Design](high/14-contract-and-api-design.md) | A caller can tell what a method requires and guarantees from its signature alone. | Check if nullability, thrown errors, and preconditions are explicit in the signature |
| P-15 | [Immutability by Default](high/15-immutability-by-default.md) | Objects don't change state after construction unless there's a specific reason they must. | Grep for setters on objects that only need to be constructed once |
| P-16 | [Open/Closed](high/16-open-closed.md) | New behavior is added by extension, not by editing code that already works. | Grep for `if (type == X) ... else if (type == Y)` sprawl |
| P-17 | [Observability](high/17-observability.md) | You can tell what happened in production without attaching a debugger. | Check if a new failure path has a log line, metric, or trace span |
| P-18 | [Backward Compatibility & Versioning](high/18-backward-compatibility-versioning.md) | Changes to a shared contract don't break existing consumers without a migration path. | Check if a field was removed or a type narrowed on a shared schema/API |
| P-19 | [Configuration Externalization](high/19-configuration-externalization.md) | Environment-specific values live outside the code, validated at startup. | Grep for hardcoded URLs, ports, or credentials in source |
| P-20 | [Naming & Readability](high/20-naming-and-readability.md) | A name tells you what a thing is or does without opening it. | Grep for `data`, `info`, `manager`, `helper`, `processX` as identifiers |

## Medium — craftsmanship

| # | Principle | One-line definition | Fastest signal to check |
|---|---|---|---|
| P-21 | [YAGNI](medium/21-yagni.md) | Don't build capability the current requirement doesn't need. | Find interfaces with exactly one implementation and no second consumer planned |
| P-22 | [KISS / Complexity Budget](medium/22-kiss-complexity-budget.md) | The simplest design that satisfies the requirement wins. | Check nesting depth and method length against the file's own norm |
| P-23 | [DRY (Rule of Three)](medium/23-dry-rule-of-three.md) | Duplicate *knowledge*, not coincidentally similar code, gets a single source of truth. | Check if the "duplication" is the same business rule or just similar shape |
| P-24 | [Composition over Inheritance](medium/24-composition-over-inheritance.md) | Prefer assembling behavior from parts over extending a base class. | Grep for inheritance hierarchies deeper than two levels |
| P-25 | [Liskov Substitution](medium/25-liskov-substitution.md) | A subtype must be usable anywhere its supertype is expected, honoring the same contract. | Grep for `throw new UnsupportedOperationException` in an override |
| P-26 | [Law of Demeter](medium/26-law-of-demeter.md) | Talk to your immediate collaborators, not to what they expose. | Grep for chained calls like `a.getB().getC().getD()` |
| P-27 | [Algorithmic Complexity & Access Patterns](medium/27-algorithmic-complexity-access-patterns.md) | Data access patterns don't scale worse than the data does. | Grep for a query or DB call inside a loop |
| P-28 | [Codebase Convention Consistency](medium/28-convention-consistency.md) | New code matches the idioms already established around it. | Compare error handling / naming style against the surrounding file |
| P-29 | [Caching Correctness](medium/29-caching-correctness.md) | A cache never becomes a second, disagreeing source of truth. | Check if there's an invalidation path for every write path |
| P-30 | [Documenting the "Why"](medium/30-documenting-why.md) | Comments explain a decision the code can't explain itself. | Check if a comment restates the code below it instead of explaining a constraint |

## Low — automate, don't review

| # | Principle | One-line definition | Fastest signal to check |
|---|---|---|---|
| P-31 | [Formatting & Style](low/31-formatting-and-style.md) | Code layout is consistent because a tool enforces it, not because reviewers do. | Is a formatter wired into CI? If not, that's the finding |
| P-32 | [Dead Code & Import Hygiene](low/32-dead-code-import-hygiene.md) | Nothing unreachable or unused ships. | Is a linter/static analyzer running in CI? If not, that's the finding |
| P-33 | [Trivial Documentation](low/33-trivial-documentation.md) | Docs earn their place by adding information the signature doesn't already give. | Grep for Javadoc on a getter that just restates the field name |
| P-34 | [Micro-optimizations](low/34-micro-optimizations.md) | Performance work is driven by a measurement, not a hunch. | Is there a profiler/benchmark result attached, or just a claim? |
| P-35 | [Package Layout & Aesthetics](low/35-package-layout.md) | Package structure is consistent, not necessarily "correct." | Is there one documented convention the repo actually follows? |

## Cross-cutting lens

Tier tells you *where* to look; it doesn't tell you *what an LLM specifically gets wrong*. After you've run the tier pass, run [`cross-cutting/ai-code-failure-modes.md`](cross-cutting/ai-code-failure-modes.md) as a second, independent pass over the same diff. It covers patterns like hallucinated APIs, tests written against the implementation instead of the requirement, and speculative abstractions invented for a single caller — failure modes that don't map cleanly to one principle but show up constantly in agent-generated code.

## Scoring model

Every page's **Scoring** section instantiates these four anchors for that specific principle. Defined once here:

| Score | Meaning |
|---|---|
| **0 — Violated** | Ship-blocker for Crucial/High. Fix before merge. |
| **1 — Partial** | Principle acknowledged but inconsistently applied; the gaps are load-bearing, not cosmetic. |
| **2 — Met** | Satisfies the principle. Nothing to flag. |
| **3 — Exemplary** | Sets the pattern; worth pointing other code at as the example to follow. |

## Condensed checklist

Copy this into a PR template. Each line is a single yes/no question; a "no" on Crucial or High is a blocking comment.

### Crucial
- [ ] **P-01** Does every branch handle null, empty, and boundary inputs explicitly?
- [ ] **P-02** Is every external input validated once, at the trust boundary, and trusted after?
- [ ] **P-03** Is untrusted input never concatenated into a query/command, and is authorization checked on every path?
- [ ] **P-04** Is every caught exception either handled meaningfully or rethrown — never swallowed with just a log?
- [ ] **P-05** Is all state shared across threads protected or made immutable?
- [ ] **P-06** Can this operation be retried or delivered twice without duplicating a side effect?
- [ ] **P-07** Is every acquired resource released deterministically, including on the error path?
- [ ] **P-08** Does the transaction boundary match the actual unit of business atomicity?
- [ ] **P-09** Does this unit have exactly one reason to change?
- [ ] **P-10** Would the new tests fail if the actual fix were reverted?

### High
- [ ] **P-11** Does this change stay contained to one module without rippling into unrelated ones?
- [ ] **P-12** Does business logic receive its dependencies instead of constructing them?
- [ ] **P-13** Is internal state hidden behind behavior instead of exposed as public/mutable?
- [ ] **P-14** Can a caller tell nullability, errors, and preconditions from the signature alone?
- [ ] **P-15** Is this type immutable unless it has a specific, stated reason not to be?
- [ ] **P-16** Can new behavior be added here without editing existing, working branches?
- [ ] **P-17** Does every new failure path emit a log, metric, or trace a human can find later?
- [ ] **P-18** Does this change to a shared contract avoid breaking existing consumers?
- [ ] **P-19** Is every environment-specific value externalized and validated at startup?
- [ ] **P-20** Does every new name tell you what it is without opening the file?

### Medium
- [ ] **P-21** Does every abstraction here serve a need that exists today?
- [ ] **P-22** Is this the simplest design that satisfies the actual requirement?
- [ ] **P-23** Is duplicated code duplicated *knowledge*, and if so, is it deduplicated?
- [ ] **P-24** Does this use composition rather than inheritance for reuse?
- [ ] **P-25** Does every subtype honor its supertype's contract without narrowing it?
- [ ] **P-26** Does this code talk only to its immediate collaborators?
- [ ] **P-27** Does this avoid a query or nested loop over data whose size can grow?
- [ ] **P-28** Does this match the conventions already established in the surrounding code?
- [ ] **P-29** Does every write path that affects a cache also invalidate it?
- [ ] **P-30** Do comments explain a decision the code itself can't, instead of restating it?

### Low (automate — don't spend review time here)
- [ ] **P-31** Formatter enforced in CI?
- [ ] **P-32** Linter/static analysis enforced in CI?
- [ ] **P-33** No documentation that only restates its own signature?
- [ ] **P-34** Any performance change backed by a measurement?
- [ ] **P-35** Package layout consistent with one documented convention?
