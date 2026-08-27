# P-03 · Security Fundamentals

> Untrusted input never reaches a query, shell, or deserializer unescaped, and authorization is checked on every path that reaches protected data — not just the one the UI uses.

**Tier:** Crucial · **Scope:** any code that builds a query/command from external input, handles secrets, deserializes external data, or gates access to data/actions

## What it means

Security fundamentals here are narrower than "security" in general — this is the small set of mistakes that turn a normal bug into a breach: injection (SQL, command, LDAP, template), broken or missing authorization, secrets in code or logs, and unsafe deserialization. Each has the same shape: untrusted data is allowed to control something it shouldn't — a query's structure, a shell command's arguments, which record gets returned, which class gets instantiated. The fix is almost always structural, not clever: parameterize instead of concatenate, check authorization at the data layer instead of the UI layer, keep secrets out of the codebase entirely, deserialize into fixed, known types.

The authorization half of this is the one most often missed by omission rather than by mistake: a UI hides a button, so the reviewer assumes access is controlled, without checking whether the underlying endpoint enforces the same rule for a caller who skips the UI entirely.

## Why it matters

A security bug's blast radius isn't bounded by the feature it lives in — a SQL injection in an obscure admin report can expose the entire database, and a missing object-level authorization check can let one authenticated user read every other user's data through the exact same endpoint they're allowed to use for their own. The cost-to-fix curve is the steepest of any principle in this guide: caught in review, it's a one-line change to a parameterized query; caught in production, it's an incident response, a forensic investigation into what was actually accessed, mandatory disclosure in many jurisdictions, and reputational cost that outlives the fix itself.

## What good looks like

- All queries use parameterized statements or an ORM's query builder — string concatenation never touches SQL, LDAP, or shell commands with untrusted input in it.
- Authorization is checked at the data-access layer (can *this* caller access *this* record), not only inferred from which endpoint or menu item was used.
- Secrets (API keys, credentials, tokens) come from a secrets manager or environment configuration, never a literal in source, a comment, or a log line.
- Deserialization targets a specific, fixed class — never `readObject` on arbitrary streams or a generic `Object`/`Class.forName` from external input.
- Object references from a client (an ID in a URL or payload) are checked for ownership before use, not just checked for existence.

## Violation signatures

- String concatenation or `String.format` building a SQL, LDAP, or shell command from a variable.
- `Runtime.exec()` or `ProcessBuilder` with an argument built from user input without an allowlist.
- An endpoint that checks authentication (`isLoggedIn()`) but not authorization (`canAccessThisResource()`), especially on an ID passed directly in the URL/body.
- Java native deserialization (`ObjectInputStream.readObject`) on any externally-sourced byte stream.
- A hardcoded credential, API key, or connection string literal in source, even in a test or example file.
- A log statement that includes a password, token, full credit card number, or full SSN.
- Authorization logic duplicated per-controller with drift, instead of centralized in a policy/guard layer.
- A newly added admin/internal endpoint with weaker auth than the equivalent user-facing one.

## Code: violation → fix

```java
// Violation: injectable query, and no ownership check on the ID
Order getOrder(String orderId, Connection conn) throws SQLException {
    String sql = "SELECT * FROM orders WHERE id = " + orderId; // injection
    ResultSet rs = conn.createStatement().executeQuery(sql);
    return mapOrder(rs); // any caller can fetch any order's ID
}
```

```java
// Fix: parameterized query, plus explicit ownership check
Order getOrder(String orderId, String requestingUserId, Connection conn) throws SQLException {
    String sql = "SELECT * FROM orders WHERE id = ? AND user_id = ?";
    try (PreparedStatement ps = conn.prepareStatement(sql)) {
        ps.setString(1, orderId);
        ps.setString(2, requestingUserId); // authorization enforced in the query itself
        ResultSet rs = ps.executeQuery();
        if (!rs.next()) throw new NotFoundException(orderId);
        return mapOrder(rs);
    }
}
```

The fix closes two independent holes at once: parameterization removes the injection vector, and folding the ownership check into the query removes the class of bug where existence and access are conflated — a caller can no longer fetch an order by guessing or incrementing an ID.

## Review checklist

1. Does any query, command, or template concatenate untrusted input directly into its structure?
2. For any endpoint that takes a resource ID, is ownership/authorization checked against the *caller*, not just resource existence?
3. Does an internal, admin, or batch path enforce the same authorization as the user-facing path that reaches the same data?
4. Are there any secrets, tokens, or credentials as literals in source, config committed to the repo, or log statements?
5. Does any deserialization target arbitrary types instead of a fixed, known class?
6. Would this code behave differently for a malicious caller who bypasses the UI and calls the endpoint directly?

## How AI-generated code violates this

Generated data-access code frequently reaches for string concatenation for queries because it's the simplest pattern that "looks like it should work" for the described task, and the model has no innate risk model flagging it — parameterization requires knowing *why* concatenation is dangerous, not just how to fetch a row. Authorization gaps are even more common: a prompt like "add an endpoint to fetch an order by ID" gets implemented literally — fetch the order by ID — without the unstated requirement that the caller must own it, because that requirement lives in institutional knowledge, not in the ticket text. Secrets leakage shows up when a model generates a working example with a plausible-looking placeholder credential that gets copy-pasted as-is, or when it adds verbose debug logging that includes a full request/response body during development and nothing removes it before merge — see [missing observability discipline](../cross-cutting/ai-code-failure-modes.md#missing-observability-because-nothing-forced-the-agent-to-add-it) for the inverse failure of the same root cause: nobody told it what belongs in a log and what doesn't.

## Guardrail snippet

```
Never build a SQL, shell, or template string by concatenating untrusted
input — always use parameterized queries, prepared statements, or an
allowlist-validated argument list. For any endpoint or method that accepts
a resource ID from a caller, always check that the caller is authorized
for that specific resource, not just that the resource exists. Never
hardcode a credential, API key, or token in source, tests, or examples —
use configuration/secrets management. Never log a password, token, or full
PII field. Deserialize only into fixed, specific types.
```

## Scoring

- **0 — Violated:** an injection vector, missing authorization check, or hardcoded secret exists on a reachable path.
- **1 — Partial:** authentication and injection defenses are present, but object-level authorization is inconsistent across equivalent paths.
- **2 — Met:** all queries are parameterized, authorization is checked per-resource on every path, no secrets in source or logs.
- **3 — Exemplary:** authorization is centralized in a policy layer that's impossible to bypass by adding a new endpoint, and this is enforced by a test.

## Related

- [P-02 Input Validation & Trust Boundaries](02-input-validation-trust-boundaries.md) — prerequisite: a well-defined trust boundary is where injection defenses actually get applied.
- [P-17 Observability](../high/17-observability.md) — in tension at the margin: logging enough to debug production issues without logging secrets or PII requires deliberate field-level redaction.
- [P-13 Encapsulation & Information Hiding](../high/13-encapsulation-information-hiding.md) — authorization is a form of information hiding enforced across a network boundary instead of a class boundary.

## Going deeper

- OWASP Top 10 (current revision) — injection and broken access control are consistently the top two categories.
- OWASP, *Cheat Sheet Series* — SQL Injection Prevention, Authorization Testing Automation.
- McGraw, *Software Security: Building Security In*, ch. 3 (Introduction to Software Security Touchpoints) — code review and architectural risk analysis as the two highest-yield touchpoints.
