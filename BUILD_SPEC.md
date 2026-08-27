# Software Engineering Principles — Build Spec

**This file is instructions for Claude Code, not the guide itself.** Read it fully, then generate the guide described below.

---

## 1. Purpose

Build a reference guide of software engineering principles that serves **three uses at once**:

1. **Manual PR review checklist** — I read a diff and check it against the principle.
2. **Agent guardrails** — I paste sections into `CLAUDE.md` / agent system prompts so generated code follows them by default.
3. **Personal study guide** — I brush up on the principle itself.

The code under review is **AI-agent-generated**, across mixed stacks. So the guide is **language-agnostic in its reasoning** but uses **Java as the primary example language** (my home stack). Reach for another language only when Java genuinely can't illustrate the point.

Every page must be written with the assumption that **the code being judged was written by an LLM**, not a junior human. That changes the failure modes and it must be reflected throughout, not bolted on.

---

## 2. Output structure

Generate a directory:

```
se-principles/
├── README.md                    # the main index page
├── crucial/
│   ├── 01-correctness-and-edge-cases.md
│   ├── 02-input-validation-trust-boundaries.md
│   └── ... (10 files)
├── high/
│   └── ... (10 files)
├── medium/
│   └── ... (10 files)
├── low/
│   └── ... (5 files)
└── cross-cutting/
    └── ai-code-failure-modes.md
```

**One file per principle. No exceptions, including the Low tier.**

Filenames: `NN-slug.md` using the exact numbers and slugs in Section 6.

---

## 3. `README.md` (main index page)

Structure it in this order:

1. **Title + a 3–5 sentence statement of what this guide is for** — reviewing AI-generated code, three use cases above.
2. **How to use it** — a short section per use case (review pass, agent guardrail, study). Include the recommended review order: Crucial tier on every diff, High tier on anything touching module boundaries or public contracts, Medium on refactors, Low never manually (automate it).
3. **The tier model** — one paragraph explaining that tier = *(blast radius if violated) × (cost to fix later)*, not "importance."
4. **Four tables, one per tier.** Columns: `#` | `Principle` (linked to its page) | `One-line definition` | `Fastest signal to check`. The "fastest signal" column is the single highest-yield thing to grep for or eyeball in a diff — keep it under 12 words.
5. **Cross-cutting lens** — a short section pointing to `cross-cutting/ai-code-failure-modes.md` explaining it's a second pass applied *after* the tier pass.
6. **Scoring model** — the 0–3 anchors from Section 5, defined once here and referenced (not repeated in full) on each page.
7. **A one-page condensed checklist** at the bottom — every principle collapsed to a single yes/no question, grouped by tier, ready to copy into a PR template.

---

## 4. Per-principle page template

Use these exact section headings, in this order. Omit a section only if it is genuinely non-applicable, and say so in one line rather than padding it.

```markdown
# P-NN · <Principle Name>

> One-sentence definition. Sharp enough to quote.

**Tier:** <Crucial/High/Medium/Low> · **Scope:** <where it applies — e.g. "any code crossing a process boundary">

## What it means
The principle itself. Assume a competent engineer who may not have thought
about this in a while — no textbook throat-clearing, no history lesson.

## Why it matters
Concrete consequence of violating it. Name the blast radius and the
cost-to-fix curve. Real failure modes, not "reduces maintainability."

## What good looks like
Bulleted positive signals. Things you'd see in a diff that satisfy it.

## Violation signatures
The most useful section on the page. Concrete, greppable, diff-visible
smells. Prefer "a `catch (Exception e)` with only a log statement inside"
over "poor error handling." Aim for 6–10 signatures.

## Code: violation → fix
Java. Short and annotated — 15–30 lines per side, not a full class listing.
Show the violation, then the fix, then one sentence on what changed and why.
Add a second example only if the principle has a genuinely different second
shape (e.g. sync vs async).

## Review checklist
5–8 yes/no questions to ask of a diff. Answerable in seconds. Ordered by
yield: highest-signal question first.

## How AI-generated code violates this
Specific to LLM output, not humans. Examples of the genre: plausible APIs
that don't exist, happy-path-only logic, tests that assert the
implementation back at itself, abstractions invented for a single caller,
confident comments describing behavior the code doesn't have. Be specific
to *this* principle — no generic filler.

## Guardrail snippet
A copy-pasteable block for `CLAUDE.md` or an agent system prompt that makes
the agent comply preemptively. Imperative voice, 3–8 lines, self-contained
(readable without the rest of this page).

## Scoring
The 0–3 anchors from README, but *instantiated for this principle* — what
a 0, 1, 2, and 3 concretely look like here. Four short lines.

## Related
Links to sibling pages, with a clause on *how* they relate — reinforcing,
in tension with, or a prerequisite for. Cross-tier links encouraged.

## Going deeper
2–4 references (book + chapter, paper, or canonical article). No URLs
needed. Skip anything you can't name confidently — do not invent sources.
```

---

## 5. Scoring anchors (define in README, instantiate per page)

- **0 — Violated.** Ship-blocker for Crucial/High. Fix before merge.
- **1 — Partial.** Principle acknowledged but inconsistently applied; gaps are load-bearing.
- **2 — Met.** Satisfies the principle. Nothing to flag.
- **3 — Exemplary.** Sets the pattern; worth pointing other code at.

---

## 6. The principle list

Use these IDs, names, and slugs verbatim. The scope note tells you what the page covers — expand it, don't just restate it.

### Crucial — violations ship incidents

| ID | Name | Slug | Scope note |
|---|---|---|---|
| P-01 | Correctness & Edge Cases | `correctness-and-edge-cases` | Null/empty/boundary handling, off-by-one, unhandled branches, silent truncation |
| P-02 | Input Validation & Trust Boundaries | `input-validation-trust-boundaries` | Validate at the edge, define what's trusted, no revalidation theatre inside the core |
| P-03 | Security Fundamentals | `security-fundamentals` | Injection, authz on every path (not just the UI path), secrets in code/logs, unsafe deserialization |
| P-04 | Error Handling & Failure Semantics | `error-handling-failure-semantics` | Swallowed exceptions, fail-fast vs degrade, error types as contract, partial failure |
| P-05 | Concurrency & Thread Safety | `concurrency-thread-safety` | Shared mutable state, races, visibility, deadlock, unbounded parallelism |
| P-06 | Idempotency & Retry Safety | `idempotency-retry-safety` | At-least-once delivery, duplicate side effects, idempotency keys, safe retry windows |
| P-07 | Resource Lifecycle | `resource-lifecycle` | Connections, streams, thread pools, executors; leaks, unbounded queues, missing timeouts |
| P-08 | Transaction & Data Integrity Boundaries | `transaction-data-integrity` | Atomicity scope, partial writes, dual-write problem, compensations, isolation assumptions |
| P-09 | Separation of Concerns / SRP | `separation-of-concerns-srp` | One reason to change; business logic bleeding into controllers, persistence, or handlers |
| P-10 | Meaningful Test Coverage | `meaningful-test-coverage` | Tests assert behavior not implementation; coverage % as a lie; missing failure-path tests |

### High — design integrity

| ID | Name | Slug | Scope note |
|---|---|---|---|
| P-11 | Coupling & Cohesion | `coupling-and-cohesion` | Module boundaries, afferent/efferent coupling, shotgun surgery, feature envy |
| P-12 | Dependency Inversion & Injection | `dependency-inversion` | Depend on abstractions, no `new` in business logic, testability as the tell |
| P-13 | Encapsulation & Information Hiding | `encapsulation-information-hiding` | Leaky internals, exposed mutable collections, public fields, over-broad visibility |
| P-14 | Contract & API Design | `contract-and-api-design` | Interface segregation, explicit pre/postconditions, nullability in signatures, error contracts |
| P-15 | Immutability by Default | `immutability-by-default` | Value objects, defensive copies, records, mutable statics, aliasing bugs |
| P-16 | Open/Closed | `open-closed` | Extend without editing; switch-on-type sprawl; strategy/plugin seams |
| P-17 | Observability | `observability` | Structured logs, metrics, traces, correlation IDs, log levels, PII in logs |
| P-18 | Backward Compatibility & Versioning | `backward-compatibility-versioning` | API and schema evolution, additive-only change, expand/contract migrations, consumer breakage |
| P-19 | Configuration Externalization | `configuration-externalization` | Hardcoded endpoints, magic numbers, env-specific branching, config validation at startup |
| P-20 | Naming & Readability | `naming-and-readability` | Intent-revealing names, `processData()`, boolean params, comment-as-crutch |

### Medium — craftsmanship

| ID | Name | Slug | Scope note |
|---|---|---|---|
| P-21 | YAGNI | `yagni` | Speculative generality, config flags nobody sets, interfaces with one implementation |
| P-22 | KISS / Complexity Budget | `kiss-complexity-budget` | Cyclomatic complexity, nesting depth, method length, clever one-liners |
| P-23 | DRY (Rule of Three) | `dry-rule-of-three` | Dedupe knowledge not coincidence; premature extraction as its own smell |
| P-24 | Composition over Inheritance | `composition-over-inheritance` | Deep hierarchies, inheritance for reuse, fragile base class |
| P-25 | Liskov Substitution | `liskov-substitution` | Subtypes honoring the contract, strengthened preconditions, `UnsupportedOperationException` |
| P-26 | Law of Demeter | `law-of-demeter` | Train wrecks, reaching through objects, brittle chains |
| P-27 | Algorithmic Complexity & Access Patterns | `algorithmic-complexity-access-patterns` | N+1 queries, nested loops over collections, hot-loop allocation, unnecessary sorting |
| P-28 | Codebase Convention Consistency | `convention-consistency` | Matching local idioms, mixed paradigms, imported patterns that don't fit |
| P-29 | Caching Correctness | `caching-correctness` | Invalidation, staleness windows, stampedes, cache-as-source-of-truth, key collisions |
| P-30 | Documenting the "Why" | `documenting-why` | Comments explaining rationale not mechanics; comments that will go stale and lie |

### Low — automate, don't review

Keep these pages **short and blunt**. The main content of each is *"here's the tool that should be catching this instead of you."* Name the specific tooling.

| ID | Name | Slug | Scope note |
|---|---|---|---|
| P-31 | Formatting & Style | `formatting-and-style` | Delegate to formatter; enforce in CI, never in review comments |
| P-32 | Dead Code & Import Hygiene | `dead-code-import-hygiene` | Linters and static analysis; unreachable branches, unused deps |
| P-33 | Trivial Documentation | `trivial-documentation` | Javadoc on getters; docs that restate the signature |
| P-34 | Micro-optimizations | `micro-optimizations` | Measure first; when it *does* matter and how you'd know |
| P-35 | Package Layout & Aesthetics | `package-layout` | Layer-vs-feature packaging; matters far less than agreement on one |

### Cross-cutting

| ID | Name | Slug |
|---|---|---|
| X-01 | AI Code Failure Modes | `ai-code-failure-modes` |

`cross-cutting/ai-code-failure-modes.md` is a **second-pass lens**, applied after the tier pass. It gets a different structure — not the standard template. Cover each failure mode with: what it is, why LLMs produce it, how to detect it, and which principles it usually violates (linked). Include at minimum:

- Hallucinated APIs, methods, and config keys
- Plausible-but-wrong logic that reads correctly
- Tests written against the implementation rather than the requirement
- Silent scope creep into files that weren't part of the task
- Speculative abstraction — layers invented for a single caller
- Missing observability, because nothing forced the agent to add it
- Confidently wrong comments and docstrings
- Version drift — patterns from an older library version applied to a newer one
- Inconsistency across turns — the same concept implemented two different ways in one session

---

## 7. Depth guidance

Target lengths per page body (excluding code):

- **Crucial:** 700–1,000 words. These earn the space.
- **High:** 500–750 words.
- **Medium:** 350–550 words.
- **Low:** 150–250 words. Terse by design.
- **Cross-cutting page:** 1,200–1,800 words.

Density beats length. If a section can be a table or a list, make it one. Cut anything that could appear verbatim in a generic blog post about clean code.

---

## 8. Writing rules

- **No filler.** No "in today's fast-paced software landscape." Open each page on the substance.
- **Specific over abstract.** Every claim should survive the question "what would that actually look like in a diff?"
- **Java for code.** Modern Java (records, sealed types, `Optional`, virtual threads where relevant). Short annotated snippets, not full classes.
- **Prose over bullets** for explanation; **bullets over prose** for signatures, checklists, and signals.
- **Cross-link generously** using relative paths. Principles that pull against each other (DRY vs YAGNI, Open/Closed vs KISS) must acknowledge the tension explicitly rather than pretending it away.
- **Don't invent references.** Only name a book, paper, or article you're confident exists, and name the chapter or section if you can.
- **Consistent voice** across all 36 files. Direct, second person for checklists, declarative for explanation.

---

## 9. Build order

1. `README.md` skeleton with all links in place (links may be dead initially).
2. `cross-cutting/ai-code-failure-modes.md` — write this early; the tier pages reference its vocabulary.
3. Crucial tier, P-01 → P-10.
4. High tier, P-11 → P-20.
5. Medium tier, P-21 → P-30.
6. Low tier, P-31 → P-35.
7. Final pass: verify every link resolves, every page has all template sections, and the README's condensed checklist covers all 35.

Stop and confirm with me after step 3 so I can react to the format before you build the remaining 25 pages.
