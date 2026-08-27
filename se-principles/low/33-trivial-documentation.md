# P-33 · Trivial Documentation

> A doc comment earns its place by adding information the signature doesn't already give — otherwise it's noise, not documentation.

**Tier:** Low · **Scope:** Javadoc/docstrings on trivial accessors and self-explanatory methods

## What it means

`/** Gets the name. */` above `getName()` tells a reader nothing they didn't already know from the method name. Trivial documentation isn't wrong so much as pointless — it inflates the file without informing anyone, and unlike a substantive comment (see [P-30](../medium/30-documenting-why.md)), there's no judgment call here: a getter restating its own name in prose adds zero information, full stop.

## Why it matters

Trivial docs cost reading time and, at scale, train reviewers to skim past *all* Javadoc, including the rare instance that actually says something important — the boy-who-cried-wolf effect on documentation quality.

## What good looks like

- Javadoc exists on public methods where behavior isn't obvious from the signature (thread-safety, ordering, side effects, exceptional cases).
- Trivial getters/setters have no Javadoc, or a project-wide suppression for "missing Javadoc on obvious accessors" is configured rather than papering over it with restated prose.

## Violation signatures

- `/** Returns the id. */` above `getId()`, or equivalent restatement patterns.
- A class-level Javadoc that just repeats the class name in sentence form ("This class represents an Order.").
- Boilerplate Javadoc generated for every method uniformly, regardless of whether the method needs it.

## Code: violation → fix

```java
// Violation
/** Gets the customer id. */
String getCustomerId() { return customerId; }
```

```java
// Fix: no doc needed — the signature already says everything there is to say
String getCustomerId() { return customerId; }
```

## Review checklist

1. Does this Javadoc say anything the signature doesn't already convey?
2. Is Javadoc present uniformly on every method regardless of whether it needs it?

## How AI-generated code violates this

Models trained on code that includes heavy Javadoc coverage tend to generate a doc comment for every public method uniformly, as a completeness habit, without discriminating between methods that need explanation and ones that don't — see [trivial documentation as a subset of confidently generated but valueless text](../cross-cutting/ai-code-failure-modes.md#confidently-wrong-comments-and-docstrings).

## Guardrail snippet

```
Do not add Javadoc to a method whose behavior is already fully conveyed
by its name and signature. Reserve doc comments for methods with
non-obvious behavior, side effects, or exceptions worth calling out.
```

## Scoring

- **0 — Violated:** Javadoc is added uniformly, purely restating obvious signatures.
- **2 — Met:** Javadoc appears only where it adds real information; trivial accessors are left undocumented.
- **3 — Exemplary:** a lint rule enforces "no Javadoc that duplicates the method name" alongside "Javadoc required on public non-trivial methods."

## Related

- [P-30 Documenting the "Why"](../medium/30-documenting-why.md) — the same principle applied to inline comments instead of formal doc comments.

## Going deeper

- Bloch, *Effective Java*, 3rd ed., Item 56 ("Write doc comments for all exposed API elements") — note its emphasis on *exposed, non-obvious* behavior, not blanket coverage.
