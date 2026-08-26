# X-01 · AI Code Failure Modes

> A second-pass lens: after you've reviewed a diff tier by tier, run it again looking specifically for how an LLM, not a human, produces bugs.

**Scope:** any diff generated substantially by an AI agent, applied as a pass independent of and after the [tier review](../README.md#the-tier-model).

The tier pages assume a competent author who sometimes cuts corners under time pressure. That's not what you're reviewing. An LLM doesn't get tired, doesn't skip the boring parts out of laziness, and doesn't cut a corner because a deadline is close — it produces a different distribution of mistakes, ones that come from how it generates text: locally coherent, globally unaware of what it can't see, and unable to feel the absence of something it never generated. This page names that distribution so you know what to look for beyond "is this code correct."

## Hallucinated APIs, methods, and config keys

**What it is:** code that calls a method, constructor overload, or configuration key that doesn't exist in the version of the library actually in the dependency tree — or that exists but with a different signature.

**Why LLMs produce it:** the model is generating the *statistically likely* next token given a library's name and a task, not looking up the actual installed version. It has seen thousands of similar APIs across versions and languages and blends them into something plausible but synthetic. This gets worse with less popular libraries and with libraries that changed their API surface across major versions.

**How to detect it:** the code won't compile or the call will throw at runtime — this is often caught by the build, which is the good case. The bad case is a hallucinated *config key* or *silently ignored parameter*, which compiles fine and fails only in behavior. Check any unfamiliar method call or config key against the actual dependency version in the lockfile, not against memory or a general web search.

**Principles it usually violates:** [P-01 Correctness & Edge Cases](../crucial/01-correctness-and-edge-cases.md), [P-14 Contract & API Design](../high/14-contract-and-api-design.md).

## Plausible-but-wrong logic that reads correctly

**What it is:** code that is well-formatted, well-named, and reads as obviously correct on skim — but implements the wrong algorithm, an off-by-one boundary, or a subtly incorrect formula.

**Why LLMs produce it:** LLMs optimize for fluent, idiomatic-looking output, and fluency is not correctness. The model has seen thousands of correct examples of the *shape* of this code and reproduces the shape confidently, without executing the logic to check the arithmetic or boundary condition actually holds for this specific case.

**How to detect it:** you cannot catch this by reading — you have to trace it against concrete inputs, ideally boundary ones (empty, one element, the max, the min, one past the end). If you find yourself thinking "this looks right" without having traced an example, you haven't reviewed it yet.

**Principles it usually violates:** [P-01 Correctness & Edge Cases](../crucial/01-correctness-and-edge-cases.md), [P-10 Meaningful Test Coverage](../crucial/10-meaningful-test-coverage.md) — because the tests it wrote for itself have the same blind spot.

## Tests written against the implementation rather than the requirement

**What it is:** a test suite that passes with 100% confidence but would also pass against almost any implementation, because it asserts on internal structure, mocks out the actual logic being tested, or was written by reading the implementation rather than the spec.

**Why LLMs produce it:** when the same generation pass writes the code and the test, the model has the implementation in its context window and the natural move is to describe what that implementation does, not what it *should* do. There's no independent party forcing the test to encode the requirement instead of the code.

**How to detect it:** revert the actual bug fix or feature logic locally and rerun the test suite. If nothing fails, the tests aren't testing the thing they claim to test. Also look for mocks that stub out the exact behavior under test, and assertions that check a mock was called rather than checking an output.

**Principles it usually violates:** [P-10 Meaningful Test Coverage](../crucial/10-meaningful-test-coverage.md) directly; often co-occurs with plausible-but-wrong logic because nothing caught it.

## Silent scope creep into files that weren't part of the task

**What it is:** a diff that touches files unrelated to the stated task — reformatting a neighboring function, "improving" an unrelated import, renaming a variable in a file it happened to open while searching for something else.

**Why LLMs produce it:** agentic coding tools read more context than they change, and a model with edit access sometimes acts on things it notices along the way rather than staying scoped to the request. There's no innate sense of "this file is not mine to touch right now" unless the task or the harness enforces it.

**How to detect it:** diff the full file list against the stated task before reading line-by-line. Any file that isn't obviously required by the task description is a flag — ask why it's there before reviewing what changed in it.

**Principles it usually violates:** [P-09 Separation of Concerns / SRP](../crucial/09-separation-of-concerns-srp.md) at the PR level; increases blast radius on every other principle by widening what the diff touches.

## Speculative abstraction — layers invented for a single caller

**What it is:** an interface with one implementation, a strategy pattern with one strategy, a generic configuration object built for flexibility no caller uses.

**Why LLMs produce it:** models are trained on a huge amount of code that demonstrates "good practice" patterns — interfaces, dependency injection, extensible design — and reproduce the pattern because it's recognizable as high-quality-looking code, independent of whether this specific call site needs it. Abstraction is a mode the model defaults to when uncertain, because it looks more thorough.

**How to detect it:** for any new interface, abstract class, or configuration surface, count the concrete callers and implementations. One of each, with no second one on the near-term roadmap, is speculative.

**Principles it usually violates:** [P-21 YAGNI](../medium/21-yagni.md) directly; often in tension with [P-16 Open/Closed](../high/16-open-closed.md), which this failure mode mimics without earning.

## Missing observability, because nothing forced the agent to add it

**What it is:** new code paths, especially new failure paths, with no log statement, metric, or trace span — the code will fail silently in production with nothing to grep.

**Why LLMs produce it:** observability is invisible in the task description most of the time. If a ticket says "handle the retry case," the model implements the retry and stops; it doesn't spontaneously add the operational visibility a human on-call engineer would want, because nothing in the prompt or the visible code made that need salient.

**How to detect it:** for every new `catch`, new branch, or new external call, check for a log line, metric increment, or trace span next to it. Its absence is the default state to expect, not the exception.

**Principles it usually violates:** [P-17 Observability](../high/17-observability.md) directly; compounds [P-04 Error Handling & Failure Semantics](../crucial/04-error-handling-failure-semantics.md) when the missing observability is on a swallowed exception.

## Confidently wrong comments and docstrings

**What it is:** a comment or docstring that describes behavior the code doesn't actually have — a stale parameter description, a claimed thread-safety guarantee that isn't implemented, a "returns null if not found" on a method that throws instead.

**Why LLMs produce it:** the model generates comments the same way it generates code — as the statistically likely accompanying text for this shape of function — rather than by verifying the comment against the actual control flow it just wrote. There's no cross-check step by default.

**How to detect it:** treat every comment claim as a hypothesis and verify it against the code below it, especially claims about nullability, thread-safety, error behavior, and complexity. A confident tone is not evidence.

**Principles it usually violates:** [P-30 Documenting the "Why"](../medium/30-documenting-why.md); a wrong comment about a contract also undermines [P-14 Contract & API Design](../high/14-contract-and-api-design.md).

## Version drift — patterns from an older library version applied to a newer one

**What it is:** code that uses a deprecated API, an old idiom, or a workaround for a bug that was fixed versions ago — correct for some version of the library, wrong or unnecessary for the one actually pinned.

**Why LLMs produce it:** training data spans years of a library's history, weighted toward whatever was most common in that data, which often lags the current major version. The model doesn't know which version is in your `pom.xml` or `package.json` unless it's told and checks.

**How to detect it:** cross-reference any non-trivial library usage against the actual pinned version, especially usages that look like they're working around a limitation — that limitation may no longer exist.

**Principles it usually violates:** [P-18 Backward Compatibility & Versioning](../high/18-backward-compatibility-versioning.md); can also manifest as unnecessary complexity under [P-22 KISS / Complexity Budget](../medium/22-kiss-complexity-budget.md) when the workaround is dead weight.

## Inconsistency across turns — the same concept implemented two different ways in one session

**What it is:** within a single PR, the same kind of thing — error handling, pagination, a null check, a naming convention — done two different ways because it was generated in two different turns or two different files without cross-checking.

**Why LLMs produce it:** each generation pass has a limited view of prior output and no persistent style memory across a long session; without an explicit instruction to match an established local pattern, the model re-derives its own default each time, and its default isn't perfectly deterministic.

**How to detect it:** for any repeated concern (error handling shape, validation shape, response shape), diff the instances against each other, not just against the file they sit in. Two correct-looking but different solutions to the same problem in one PR is itself the finding.

**Principles it usually violates:** [P-28 Codebase Convention Consistency](../medium/28-convention-consistency.md) directly; can produce accidental [P-23 DRY](../medium/23-dry-rule-of-three.md) violations when the two implementations should have been one.
