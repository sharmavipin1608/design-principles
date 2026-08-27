# P-17 · Observability

> You can tell what happened in production without attaching a debugger.

**Tier:** High · **Scope:** every new code path, especially failure paths, external calls, and anything a future on-call engineer will need to reconstruct after the fact

## What it means

Observability is the ability to answer questions about production behavior you didn't anticipate at the time you wrote the code — which requires structured logs, metrics, and traces that carry enough context (correlation IDs, relevant identifiers) to reconstruct what happened, not just that something happened. A log line that says `"Error occurred"` with no request ID, no relevant entity ID, and no structured fields is barely better than nothing — it tells you the code path executed without telling you which invocation, for which user, with what input. Correlation IDs are the connective tissue that make this work across a distributed system: without one propagated through a request's full path, "what happened to this specific request" requires manually correlating timestamps across services, which doesn't scale past a handful of log lines.

Log level discipline and PII handling are part of the same principle: logs that are too verbose (everything at `INFO`) or contain sensitive data are actively harmful, either because they drown the signal or because they turn your log aggregator into a compliance liability.

## Why it matters

Missing observability doesn't cause an incident by itself — it makes every other incident take longer to diagnose and, in the worst case, makes some incidents undiagnosable at all, closed as "resolved itself" with no understanding of root cause and no protection against recurrence. This is High-tier rather than Crucial because the absence is silent: the code works, ships, and runs fine for months, right up until the moment something goes wrong and there's nothing to look at — at which point the cost isn't a code review comment, it's hours of an incident bridge trying to reconstruct behavior after the fact instead of reading it directly off a dashboard.

## What good looks like

- Every new failure path (a caught exception, a fallback branch, a retry) has a corresponding log line, metric increment, or trace span next to it.
- Logs are structured (key-value or JSON fields), not free-text string interpolation, so they're queryable, not just readable.
- A correlation/request ID is generated or propagated at the entry point and included in every log line for that request's full path, across services.
- Log levels are used deliberately: `ERROR` for things that need attention, `WARN` for degraded-but-handled situations, `INFO` for significant business events, `DEBUG` for detail not needed in normal operation.
- No PII, credentials, or full sensitive payloads appear in log output — fields that could contain them are redacted or omitted.

## Violation signatures

- A `catch` block, fallback, or retry with no log line, metric, or trace span at all.
- Free-text string-concatenated log messages (`log.info("processed order " + orderId)`) instead of structured fields.
- A log statement with no correlation/request ID, in a system where a request touches more than one service or thread.
- Everything logged at `INFO` (or everything at `ERROR`), with no differentiation by actual severity.
- A log line containing a full request/response body, a password, a token, or an unredacted PII field.
- A metric or trace span added only for the success path, with no equivalent on the failure/timeout path — the case you'd most want visibility into.
- Debug-level logging left enabled by default in a way that will flood production log volume.

## Code: violation → fix

```java
// Violation: no correlation ID, unstructured message, and total silence on failure
void processPayment(String orderId, BigDecimal amount) {
    try {
        gateway.charge(orderId, amount);
    } catch (GatewayException e) {
        log.error("payment failed: " + e.getMessage()); // no orderId, no context, no metric
        throw e;
    }
}
```

```java
// Fix: structured, correlated, and the failure path is now visible in aggregate
void processPayment(String orderId, BigDecimal amount) {
    try {
        gateway.charge(orderId, amount);
        metrics.increment("payment.success");
    } catch (GatewayException e) {
        log.error("payment charge failed",
            kv("orderId", orderId), kv("amount", amount), kv("correlationId", MDC.get("requestId")),
            e);
        metrics.increment("payment.failure", "reason", e.getErrorCode());
        throw e;
    }
}
```

The fix makes the failure findable by `orderId` or `correlationId` in a log aggregator, and makes the failure *rate* visible on a dashboard via the metric — the difference between "we might have a problem" and "we can see exactly how big the problem is and for which orders."

## Review checklist

1. Does every new failure path (catch, fallback, retry) have a log line, metric, or trace span?
2. Are log messages structured with relevant fields, not free-text string concatenation?
3. Is a correlation/request ID present and propagated across the request's full path?
4. Is the log level appropriate to actual severity, not uniformly `INFO` or `ERROR`?
5. Does any log statement include PII, credentials, or a full unredacted payload?
6. Is there a metric or trace span for the failure path specifically, not just the success path?

## How AI-generated code violates this

Observability is close to a textbook example of a requirement that's invisible in a ticket description — "implement retry logic for the payment gateway" doesn't mention logging or metrics, so a model satisfying that literal request has no signal telling it to add them, and it doesn't spontaneously add the operational visibility an experienced engineer would want by default; see [missing observability, because nothing forced the agent to add it](../cross-cutting/ai-code-failure-modes.md#missing-observability-because-nothing-forced-the-agent-to-add-it) for the general pattern. When a model does add logging, it's often skewed toward what's easy to generate locally — a message string built from whatever variables are in scope — rather than structured fields and correlation propagation, which require knowing the surrounding system's logging conventions, something a single-function generation pass typically doesn't have visibility into.

## Guardrail snippet

```
Every new failure path (catch block, fallback, retry, timeout) must emit
a structured log line or metric — never fail silently. Use structured
key-value logging, not string-concatenated messages. Include the
prevailing correlation/request ID on every log line for a multi-step or
cross-service operation. Never log a password, token, full PII field, or
full request/response body. Match log level to actual severity.
```

## Scoring

- **0 — Violated:** a new failure path has no logging, metric, or trace at all.
- **1 — Partial:** failure paths are logged, but unstructured, without correlation IDs, or with inconsistent log levels.
- **2 — Met:** structured, correlated logging and metrics exist on both success and failure paths, with no PII exposure.
- **3 — Exemplary:** observability is built into a shared middleware/aspect (automatic correlation propagation, standard metric emission) so new code inherits it without hand-writing it each time.

## Related

- [P-04 Error Handling & Failure Semantics](../crucial/04-error-handling-failure-semantics.md) — a handled failure that isn't observed is still invisible in production; the two are separate requirements that are often conflated.
- [P-03 Security Fundamentals](../crucial/03-security-fundamentals.md) — logging discipline and PII redaction sit directly at the intersection of these two principles.
- [P-06 Idempotency & Retry Safety](../crucial/06-idempotency-retry-safety.md) — retry attempts specifically need visibility (attempt count, outcome) or silent retries can mask a systemic failure until it's severe.

## Going deeper

- Majors, Fong-Jones, Miranda, *Observability Engineering*, ch. 1–3 — the distinction between monitoring and observability, and structured events as the unit of telemetry.
- Nygard, *Release It!*, 2nd ed., ch. 4 (Stabilizing patterns) and its treatment of production visibility.
- Google SRE Book, ch. 6 (Monitoring Distributed Systems) — the four golden signals as a baseline for what to instrument.
