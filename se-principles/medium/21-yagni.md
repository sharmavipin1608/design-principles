# P-21 · YAGNI

> Don't build capability the current requirement doesn't need.

**Tier:** Medium · **Scope:** new interfaces, configuration options, extension points, and generalized parameters added ahead of an actual second use case

## What it means

"You Aren't Gonna Need It" is a discipline against building for imagined future requirements instead of the one in front of you: an interface with one implementation "in case we need another provider later," a configuration flag nobody has asked to set, a generic parameter that accepts five modes when only one is used. The future need might be real, but building for it now costs certainty (you're guessing at a shape you don't have real requirements for yet) and costs the reader (more to understand, more surface to keep consistent) for a payoff that may never arrive, or that would look different once the actual second requirement shows up. The counter-argument — "we'll need it eventually" — is usually wrong about the shape even when it's right about the eventuality, which means the speculative version gets rewritten anyway when the real second case appears.

## Why it matters

Speculative generality doesn't cause incidents, which is why it's Medium- not Crucial-tier, but it has a real, compounding cost: every unnecessary abstraction is something a reader has to understand and reason about being *possibly* used differently, every unused config flag is a thing that could theoretically be set wrong, and every interface with one implementation is indirection a debugger has to step through for no payoff. In an AI-assisted workflow specifically, this cost multiplies, because a model generating new code in a file full of speculative abstraction has to decide whether that abstraction is load-bearing or decorative before it can safely extend it — extra unresolved ambiguity in every future change to that area.

## What good looks like

- New interfaces are introduced when there are at least two concrete implementations or callers, not preemptively for one.
- Configuration options exist because someone has actually asked to vary that value, not because it "might need to be configurable."
- Generic/parameterized code handles the actual variation the codebase has today, not variation it might have someday.
- A method's parameter list matches what its current callers need — no unused parameters carried "for future flexibility."
- When a real second use case does appear, refactoring toward the abstraction it actually needs is treated as a normal, cheap step — not something to avoid by having over-built early.

## Violation signatures

- An interface with exactly one implementation and no second implementation planned in a concrete, dated way.
- A configuration flag with no code path that reads a non-default value, or that nobody has ever set to non-default in any environment.
- A generic type parameter or strategy pattern used by exactly one caller.
- A method parameter that's accepted but never used, or always passed the same literal value at every call site.
- A "pluggable" architecture (a registry, a factory) for a concept that has exactly one real plugin.
- Comments like `// for future use` or `// in case we need this later` attached to unused code.

## Code: violation → fix

```java
// Violation: an abstraction for a hypothetical second notification channel
interface NotificationChannel {
    void send(String recipient, String message);
}

class EmailNotificationChannel implements NotificationChannel { // the only implementation
    public void send(String recipient, String message) { /* ... */ }
}

class NotificationService {
    private final NotificationChannel channel; // injected, but only ever EmailNotificationChannel
    NotificationService(NotificationChannel channel) { this.channel = channel; }
}
```

```java
// Fix: one real implementation, no interface until a second one actually exists
class NotificationService {
    void sendEmailNotification(String recipient, String message) { /* ... */ }
}
```

The fix removes an abstraction with no second implementation to justify it; if an SMS channel becomes a real requirement later, extracting an interface at that point takes minutes and is guided by what the *actual* second implementation needs, not a guess made before it existed.

## Review checklist

1. Does this interface have more than one real implementation, or is it speculative?
2. Is this configuration option actually set to a non-default value anywhere, by anyone?
3. Is this parameter used differently by more than one caller, or always passed the same value?
4. Would removing this abstraction change any current behavior?
5. Is there a comment justifying this as being for hypothetical future use?

## How AI-generated code violates this

Models are trained on a large amount of code that demonstrates "good architecture" — interfaces, dependency injection, configurable strategies — and that pattern is recognizable and reproducible independent of whether the current task actually needs it; generating the abstraction pattern-matches to what confident, senior-looking code looks like, which makes it a common default even for a single, concrete task. This is one half of a matched pair with [P-16 Open/Closed](../high/16-open-closed.md): a model will sometimes speculatively abstract a case that doesn't need it yet, and in a different file in the same session, under-refactor a conditional that's genuinely outgrown its shape — the instinct isn't reliably calibrated to where extensibility is actually earned versus merely decorative. See [speculative abstraction](../cross-cutting/ai-code-failure-modes.md#speculative-abstraction--layers-invented-for-a-single-caller) for the general cross-cutting pattern this principle names directly.

## Guardrail snippet

```
Do not introduce an interface, strategy pattern, or configuration option
for a capability with only one current implementation or use case. Build
the concrete thing the current requirement needs; extract an abstraction
only once a second real caller or implementation exists. Remove unused
parameters and flags rather than keeping them "for future flexibility."
```

## Scoring

- **0 — Violated:** an interface, config flag, or generic parameter exists with no current second use, adding pure indirection.
- **1 — Partial:** most code is need-driven, but one speculative abstraction or unused flag exists in this diff.
- **2 — Met:** every abstraction present serves a real, current need with more than one concrete use.
- **3 — Exemplary:** the codebase visibly favors concrete-first design, with abstractions introduced in later, smaller commits exactly when a second real case appears.

## Related

- [P-16 Open/Closed](../high/16-open-closed.md) — the direct tension: this principle says don't build the extension point yet; P-16 says build it once variation is real and recurring.
- [P-22 KISS / Complexity Budget](22-kiss-complexity-budget.md) — speculative generality is one of the most common sources of unnecessary complexity.
- [P-23 DRY (Rule of Three)](23-dry-rule-of-three.md) — premature abstraction to avoid duplication is the DRY-side version of the same overreach this principle warns against.

## Going deeper

- Fowler, *"Yagni"* (martinfowler.com) — the canonical short statement of the principle and its origin in Extreme Programming.
- Hunt & Thomas, *The Pragmatic Programmer*, ch. "Prototypes and Post-it Notes" / the tracer bullets discussion on building only what's needed to learn.
- Beck, *Extreme Programming Explained*, 2nd ed. — YAGNI as one of the core XP practices.
