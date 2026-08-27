# Software Engineering Principles

A reference guide for reviewing **AI-generated code** — 35 principles organized into four tiers by blast radius, plus a cross-cutting lens for the failure modes specific to LLM-authored code.

**→ [Start here: the guide index](se-principles/README.md)**

It's built to serve three uses at once:

1. **Manual PR review checklist** — read a diff, check it against the tier.
2. **Agent guardrails** — every page ends with a copy-pasteable block for `CLAUDE.md` or an agent system prompt.
3. **Study guide** — refresh the reasoning behind a principle you already know.

Reasoning is language-agnostic; examples are modern Java.

## Layout

| Tier | Principles | When to review |
|---|---|---|
| [Crucial](se-principles/README.md#crucial--violations-ship-incidents) | P-01 – P-10 | Every diff, no exceptions |
| [High](se-principles/README.md#high--design-integrity) | P-11 – P-20 | Anything touching module boundaries or public contracts |
| [Medium](se-principles/README.md#medium--craftsmanship) | P-21 – P-30 | Refactors |
| [Low](se-principles/README.md#low--automate-dont-review) | P-31 – P-35 | Never manually — automate it |
| [Cross-cutting](se-principles/cross-cutting/ai-code-failure-modes.md) | X-01 | A second pass, after the tier pass |

Every principle page follows the same template: definition, why it matters, positive signals, greppable violation signatures, an annotated Java violation→fix, a review checklist, how AI-generated code specifically violates it, a guardrail snippet, 0–3 scoring anchors, related principles, and references.

A [condensed one-question-per-principle checklist](se-principles/README.md#condensed-checklist) at the bottom of the index is ready to paste into a PR template.

---

`BUILD_SPEC.md` in this repo is the specification the guide was generated from.
