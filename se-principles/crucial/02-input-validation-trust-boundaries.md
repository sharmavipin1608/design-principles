# P-02 · Input Validation & Trust Boundaries

> Every input is validated exactly once, at the point where trust changes — and treated as trusted everywhere after.

**Tier:** Crucial · **Scope:** any code that receives data from outside its own process, or from a less-trusted layer within it (HTTP handlers, message consumers, file parsers, CLI args)

## What it means

A trust boundary is any point where data crosses from a context that can't be controlled (a user, another service, a file on disk) into one your code is responsible for. The discipline is: validate rigorously at that boundary, then treat the data as trusted for the rest of its life inside your system. That means defining, explicitly, what "valid" means for this input — type, range, format, cardinality — and rejecting anything outside it before it goes one line further.

The failure isn't just "missing validation." It's just as often *misplaced* validation: checks scattered three layers deep in business logic, re-validating the same field in five different places because nobody could say with confidence where the boundary actually was. That's revalidation theatre — it looks like diligence but it means nobody actually owns the guarantee, and it usually still has a gap.

## Why it matters

Everything downstream of the boundary is written assuming valid input, because rewriting every function to defend against garbage at every call is how you get an unmaintainable, un-auditable mess. If the boundary check has a gap, that assumption is now false everywhere at once, and the failure surfaces far from the actual defect — a null pointer three services and two days later, a corrupted record discovered in an audit, a security bypass discovered by someone other than you. The cost-to-fix curve is steep specifically because the fix has to happen at the boundary, but the *symptom* shows up deep in the system, so diagnosis time dominates the cost.

## What good looks like

- There is one identifiable place per input type where validation happens — a controller layer, a DTO constructor, a schema validator — not scattered checks of decreasing rigor as data flows deeper.
- Validation failures produce a specific, actionable error (which field, why) rather than a generic rejection or a crash.
- Once past the boundary, internal code trusts the shape of its input — no defensive re-checking of already-validated fields.
- The definition of "valid" is explicit and testable (a schema, a set of constraints), not implicit in whatever the first caller happened to send.
- Deserialization (JSON, form data) is bound to typed, constrained objects, not passed around as raw maps that get spot-checked ad hoc.

## Violation signatures

- The same field (`email`, `userId`, `amount`) is checked for validity in more than one layer with different logic each time.
- A REST controller passes a raw `Map<String, Object>` or unvalidated DTO straight into a service method.
- Validation exists on the API/UI path but not on an internal or batch/admin path that reaches the same logic.
- A regex or range check duplicated across files instead of centralized in one validator or constraint annotation.
- Business logic contains a null/format check for a field the boundary layer was supposed to guarantee — a sign nobody trusts the boundary, which usually means the boundary is incomplete.
- Deserialization straight into a domain object with no bean-validation annotations (`@NotNull`, `@Size`, `@Pattern`) or equivalent.
- File or CSV parsing that assumes well-formed rows.

## Code: violation → fix

```java
// Violation: no boundary — "validation" is scattered guesswork downstream
@PostMapping("/transfers")
ResponseEntity<?> transfer(@RequestBody Map<String, Object> body) {
    service.transfer(body); // service has to guess what's in here
    return ResponseEntity.ok().build();
}

void transfer(Map<String, Object> body) {
    Object amt = body.get("amount");
    if (amt != null) { // half-hearted check, wrong layer, easy to bypass
        doTransfer((Double) amt); // ClassCastException if amt is a String
    }
}
```

```java
// Fix: one typed boundary; everything past it is trusted
record TransferRequest(
    @NotNull String fromAccount,
    @NotNull String toAccount,
    @Positive BigDecimal amount
) {}

@PostMapping("/transfers")
ResponseEntity<?> transfer(@Valid @RequestBody TransferRequest req) {
    service.transfer(req.fromAccount(), req.toAccount(), req.amount());
    return ResponseEntity.ok().build();
}

void transfer(String from, String to, BigDecimal amount) {
    doTransfer(from, to, amount); // no re-checking — the boundary already guaranteed this
}
```

The fix moves validation to one place (`@Valid` on the boundary DTO), makes the constraint explicit and machine-checked (`@Positive`, `@NotNull`), and lets `transfer` trust its arguments completely instead of guessing at their shape.

## Review checklist

1. For this input, can you point to exactly one place where it's validated? If two or more, that's a flag, not a bonus.
2. Does every path that reaches this logic (API, batch job, admin tool, message consumer) go through the same validation?
3. Is the validation rule explicit and testable (annotation, schema, dedicated function) rather than an inline `if` buried in business logic?
4. Do failure messages identify which field failed and why, not just "bad request"?
5. Does code past the boundary re-check anything the boundary already guarantees? If so, why doesn't it trust the boundary?
6. Is deserialization bound to a typed, constrained object rather than a raw map or untyped JSON tree?

## How AI-generated code violates this

Generated code frequently validates on the path the prompt described (usually the primary REST endpoint) and skips equivalent paths it wasn't explicitly asked about — an admin endpoint, a batch import, a webhook consumer that reaches the same service method. Because each generation pass tends to solve the specific request in front of it, the same validation logic often gets *reinvented* per endpoint instead of centralized, producing the revalidation-theatre pattern by default rather than by mistake — this is the same [inconsistency-across-turns](../cross-cutting/ai-code-failure-modes.md#inconsistency-across-turns--the-same-concept-implemented-two-different-ways-in-one-session) failure mode applied specifically to validation logic. It's also common to see validation of the *type* (does this parse as a number) without validation of the *domain* (is this number a valid account ID), because the type check is what the framework surfaces first and the model treats it as sufficient.

## Guardrail snippet

```
Identify the trust boundary for every new input (HTTP request, message
payload, file, CLI arg) and put all validation for that input in exactly
one place at that boundary — a bean-validated DTO, a schema, or a single
validator function. Do not re-check already-validated fields deeper in the
call stack. Before adding validation logic, check whether equivalent logic
already exists elsewhere for the same field — reuse it, don't reimplement
it. Any additional code path that reaches the same business logic (batch,
admin, internal) must go through the same boundary validation.
```

## Scoring

- **0 — Violated:** an input reaches business logic through at least one path with no validation.
- **1 — Partial:** the primary path is validated; a secondary path (batch, admin, internal) is not, or validation is duplicated with drifting rules.
- **2 — Met:** one clear boundary validates every path consistently; downstream code trusts it.
- **3 — Exemplary:** validation is encoded in types (constrained constructors, sealed request objects) so an unvalidated instance is unconstructible.

## Related

- [P-03 Security Fundamentals](03-security-fundamentals.md) — trust boundaries are where injection and authorization bypasses are prevented; this principle is the general case, P-03 is the security-specific instance.
- [P-01 Correctness & Edge Cases](01-correctness-and-edge-cases.md) — validation defines what's *allowed*; correctness handles everything allowed, correctly.
- [P-14 Contract & API Design](../high/14-contract-and-api-design.md) — a well-designed contract makes the boundary self-documenting instead of relying on prose.

## Going deeper

- OWASP, *Input Validation Cheat Sheet* — canonical treatment of boundary validation strategy (allowlist vs. denylist, canonicalization).
- Evans, *Domain-Driven Design*, ch. 14 (Maintaining Model Integrity) — bounded contexts, whose edges map closely to "where trust changes."
- Bloch, *Effective Java*, 3rd ed., Item 49 ("Check parameters for validity").
