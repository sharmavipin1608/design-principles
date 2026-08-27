# P-19 · Configuration Externalization

> Environment-specific values live outside the code, and the application refuses to start if they're missing or invalid.

**Tier:** High · **Scope:** connection strings, endpoints, credentials, feature flags, timeouts, and any value that legitimately differs between dev, staging, and production

## What it means

Anything that changes between environments — a database URL, an API key, a timeout, a feature flag — belongs in configuration (environment variables, a config service, a properties file loaded per environment), not in source code as a literal. The externalization half is necessary but not sufficient: the other half is validating configuration at startup, so a missing or malformed value fails immediately and loudly when the application boots, rather than surfacing as a confusing runtime error the first time that specific code path executes, possibly hours or days into production operation. "Fail fast at startup" versus "fail confusingly at first use" is the difference between a five-second deploy-time error and a 2 a.m. page.

This also covers magic numbers with environment-shaped meaning even when they're not literally per-environment — a hardcoded retry count, batch size, or rate limit that will eventually need to differ by environment or be tuned without a code change should live in configuration from the start, not be discovered as a hardcoded constant during an incident.

## Why it matters

A hardcoded endpoint or credential doesn't just create an inconvenience when environments diverge — it's a direct path to shipping a test/staging value to production (or vice versa) because nothing forced anyone to notice the divergence, and depending on what the hardcoded value is, that mistake ranges from "the feature is broken" to "we just pointed production traffic at a test database" or "we committed a real credential to source control." Missing startup validation compounds the cost of any such mistake: instead of a deploy failing immediately with a clear "missing required config: DATABASE_URL," the application starts successfully, looks healthy, and only fails deep inside a request handler the first time that path is exercised — often with a much less informative error, at a much less convenient time to debug it.

## What good looks like

- Environment-specific values are read from environment variables, a config service, or environment-specific property files — never as string/numeric literals in source.
- All required configuration is validated at application startup — missing or malformed values cause an immediate, clear startup failure, not a later runtime exception.
- Configuration objects are typed and constructed once at startup, not read ad hoc via scattered `System.getenv()`/`getProperty()` calls throughout the codebase.
- Secrets specifically come from a secrets manager or injected environment variable, never a config file committed to source control.
- Defaults, where they exist, are safe and documented — a missing feature flag defaults to "off," not to an unpredictable behavior.

## Violation signatures

- A URL, hostname, port, or connection string as a literal string in source code.
- `System.getenv("SOME_KEY")` scattered across multiple files instead of read once into a typed config object at startup.
- No validation step at startup — the first `NullPointerException` or `NumberFormatException` from bad config happens deep in a request handler.
- A config value with an undocumented, surprising default (e.g., a missing timeout silently becoming zero or infinite instead of failing loudly).
- Different hardcoded values for the same concept scattered across files (a retry count of `3` in one place, `5` in another) that should be one configurable value.
- A secrets file checked into source control, even if `.gitignore`d in later commits (it's still in history).

## Code: violation → fix

```java
// Violation: hardcoded, unvalidated, discovered as broken only when this method runs
class PaymentGatewayClient {
    private final String endpoint = "https://api-staging.paymentco.com"; // wrong in prod
    private final String apiKey = System.getenv("PAYMENT_API_KEY"); // null-checked nowhere
}
```

```java
// Fix: typed, externalized, validated once at startup
record PaymentGatewayConfig(String endpoint, String apiKey, Duration timeout) {
    static PaymentGatewayConfig fromEnv() {
        String endpoint = requireEnv("PAYMENT_GATEWAY_ENDPOINT");
        String apiKey = requireEnv("PAYMENT_API_KEY");
        return new PaymentGatewayConfig(endpoint, apiKey, Duration.ofSeconds(5));
    }
    private static String requireEnv(String key) {
        String value = System.getenv(key);
        if (value == null || value.isBlank()) {
            throw new IllegalStateException("missing required config: " + key); // fails at boot
        }
        return value;
    }
}
```

The fix moves the environment-specific value out of source entirely, and makes a missing value fail the moment the application starts — at deploy time, with a clear message — instead of the first time a payment is attempted in production.

## Review checklist

1. Is any URL, credential, or environment-specific value a literal in source code?
2. Is required configuration validated at startup, or does missing config surface later as a runtime error?
3. Are config values read once into a typed object, or scattered as ad hoc `getenv`/`getProperty` calls?
4. Do secrets come from a secrets manager/environment variable rather than a committed file?
5. Does a missing config value have a safe, documented default, or does it fail unpredictably?

## How AI-generated code violates this

A model demonstrating a working integration will very often hardcode a plausible-looking endpoint or a placeholder credential directly into the example, because that's the fastest way to produce code that "runs" for the immediate task, and environment differentiation is invisible in a single-environment prompt — there's no signal in "write a client for the payments API" that a staging and production environment need different endpoints unless that's stated explicitly. Startup validation is even less likely to appear spontaneously: it's an operational concern that doesn't affect whether the code "works" in the sense the model is optimizing for (does it run against the example input), so the model has no functional reason to add a check whose only payoff is a clearer failure mode for an input the immediate task never presents.

## Guardrail snippet

```
Never hardcode a URL, hostname, port, connection string, or credential in
source code — read it from environment variables or a config service.
Validate all required configuration at application startup and fail
immediately with a clear message if anything required is missing or
malformed — never let a missing config value surface as a runtime error
deep in a request handler. Read configuration once into a typed object,
not via scattered getenv/getProperty calls.
```

## Scoring

- **0 — Violated:** a URL, credential, or environment-specific value is hardcoded in source.
- **1 — Partial:** values are externalized, but startup validation is missing or a value is read ad hoc instead of through a typed config object.
- **2 — Met:** all environment-specific values are externalized, typed, and validated at startup with clear failure messages.
- **3 — Exemplary:** configuration schema is enforced by tooling (a validated schema, a config-linting step in CI) so an invalid deploy configuration is caught before it ever reaches a running instance.

## Related

- [P-03 Security Fundamentals](../crucial/03-security-fundamentals.md) — hardcoded credentials are a security violation and a configuration violation simultaneously.
- [P-04 Error Handling & Failure Semantics](../crucial/04-error-handling-failure-semantics.md) — fail-fast at startup is the same philosophy this principle applies specifically to configuration.
- [P-18 Backward Compatibility & Versioning](18-backward-compatibility-versioning.md) — a config schema change (renaming a required env var) is a breaking change to every deployment pipeline that sets the old one.

## Going deeper

- Wiggins, *The Twelve-Factor App*, Factor III ("Config") — the canonical short treatment of this exact principle.
- Nygard, *Release It!*, 2nd ed., ch. 9 (Configuration) — fail-fast validation as a stability pattern.
- Humble & Farley, *Continuous Delivery*, ch. 12 — managing configuration across environments as part of a deployment pipeline.
