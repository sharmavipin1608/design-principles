# P-26 · Law of Demeter

> Talk to your immediate collaborators — not to what they expose, and not to what those expose in turn.

**Tier:** Medium · **Scope:** any chained method call that reaches through more than one object to get to the data or behavior it actually wants

## What it means

The Law of Demeter (a "unit talks only to its immediate friends") restricts a method to calling methods on: itself, its parameters, objects it creates, and its own direct fields — not the objects those return. A chain like `order.getCustomer().getAddress().getCity()` — a "train wreck" — means the calling code implicitly knows about `Order`'s internal structure, `Customer`'s internal structure, and `Address`'s internal structure all at once, when all it actually needed was a city name. The fix is usually to ask the immediate collaborator to do the work and return the answer (`order.getShippingCity()`), pushing the chain-walking into the object that actually owns the relevant structure, where a change to that structure only requires updating one method instead of every caller that happened to chain through it.

## Why it matters

A train-wreck call site is coupled to the entire shape of every object in the chain, not just the one it's nominally talking to — if `Customer` ever changes how it stores an address (a value object instead of a raw field, a different nesting), every caller that chained through `getCustomer().getAddress()` breaks, even though none of them cared about `Customer` or `Address` specifically, only about a city string. This is Medium-tier because a single chain rarely causes an incident, but a codebase full of them means an internal refactor of any object ripples out to every distant call site that happened to reach through it — the shotgun-surgery symptom from [P-11 Coupling & Cohesion](../high/11-coupling-and-cohesion.md), caused specifically by chains instead of shared state.

## What good looks like

- Call chains are one level deep in typical business logic — `a.getB()` is normal; `a.getB().getC().getD()` is a flag.
- An object exposes methods that answer the question a caller actually has (`getShippingCity()`), rather than exposing its internal structure for callers to walk themselves.
- A change to an intermediate object's internal structure doesn't require updating distant call sites that never should have known about that structure.
- Fluent builder APIs (which chain by design, on the same object) are distinguished from train wrecks (which chain across different objects' accessors) — the rule targets the latter, not the former.

## Violation signatures

- A chain of three or more accessor calls reaching through different objects: `a.getB().getC().getD()`.
- A method that reaches deep into a parameter's structure to extract a single value it could have asked for directly.
- The same multi-step chain repeated across several call sites, meaning the knowledge of that structure is duplicated everywhere it's used.
- A null-check pyramid guarding each step of a long chain (`if (a.getB() != null && a.getB().getC() != null ...)`), a sign the chain is both too deep and fragile.
- Test setup code that has to construct several nested objects just to satisfy one chained call three layers down.

## Code: violation → fix

```java
// Violation: reaches through Order -> Customer -> Address to get one field;
// every caller needs a defensive null chain and knows Customer's internals
String getShippingCity(Order order) {
    if (order.getCustomer() != null
            && order.getCustomer().getAddress() != null) {
        return order.getCustomer().getAddress().getCity();
    }
    return "Unknown";
}
```

```java
// Fix: each object answers for its own structure; the caller asks one question
class Order {
    String getShippingCity() {
        return customer != null ? customer.getShippingCity() : "Unknown"; // same guard, one hop
    }
}
class Customer {
    String getShippingCity() { return address != null ? address.getCity() : "Unknown"; }
}

// call site:
String city = order.getShippingCity();
```

The fix moves the null-handling and structural knowledge to the objects that actually own that structure — `Order` no longer needs to know `Customer` has an `Address`, and a future change to how `Customer` stores its address touches one method instead of every caller across the codebase. Each class still guards exactly the one reference it owns, so the composed behavior is identical to the original chain's; a refactor that quietly drops a null check while "cleaning up" a chain has traded a [P-26](26-law-of-demeter.md) smell for a [P-01](../crucial/01-correctness-and-edge-cases.md) bug, which is a bad trade in every tier ordering this guide uses.

## Review checklist

1. Is there a chain of three or more accessor calls reaching through different objects?
2. Does a null-check pyramid exist to guard a long chain?
3. Is the same multi-step chain repeated across more than one call site?
4. Could the immediate collaborator answer the actual question directly instead of exposing the structure to walk?
5. Is this a fluent chain on the same object (fine) or a train wreck across different objects' accessors (a flag)?

## How AI-generated code violates this

Given a task like "get the shipping city for this order," the shortest path from a model's perspective is often to walk whatever chain of getters is already available on the objects in scope — `order.getCustomer().getAddress().getCity()` compiles immediately and answers the question, whereas the properly encapsulated version requires adding a new method to `Customer` and possibly `Order`, which is a larger, more invasive change for a task the model perceives as "just reading a field." This is a case where the generated code is entirely correct on the happy path and only reveals its cost the next time `Customer` or `Address`'s internal structure changes — a cost that lands on whoever (or whatever) touches that structure next, not on the code that created the chain.

## Guardrail snippet

```
Never chain more than one accessor call across different objects
(a.getB().getC()) to reach a value — add a method to the immediate
collaborator that answers the actual question instead, and let it walk
its own internal structure. Distinguish fluent builder chains on the same
object (fine) from train wrecks reaching through unrelated objects
(refactor). If a null-check pyramid guards a chain, that's the signal to
push the logic down into the owning object.
```

## Scoring

- **0 — Violated:** a train-wreck chain three or more objects deep exists and is duplicated across multiple call sites.
- **1 — Partial:** most access is properly encapsulated, but a single chain reaches through more than one object.
- **2 — Met:** call sites talk only to immediate collaborators; deeper structure is hidden behind methods on the objects that own it.
- **3 — Exemplary:** the object model is designed so the questions callers actually need to ask map directly onto single-hop methods, with no legitimate need for a deeper chain anywhere in the codebase.

## Related

- [P-11 Coupling & Cohesion](../high/11-coupling-and-cohesion.md) — a train wreck is a specific, call-site-visible symptom of the feature-envy pattern this High-tier principle names more broadly.
- [P-13 Encapsulation & Information Hiding](../high/13-encapsulation-information-hiding.md) — a chain reaching through an object's structure is only possible because that structure wasn't hidden behind behavior.
- [P-23 DRY (Rule of Three)](23-dry-rule-of-three.md) — a repeated chain across several call sites is duplicated structural knowledge, not just a style issue.

## Going deeper

- Lieberherr & Holland, *"Assuring Good Style for Object-Oriented Programs"* — the original paper introducing the Law of Demeter.
- Hunt & Thomas, *The Pragmatic Programmer*, ch. 5 — "Tell, Don't Ask" as the practical framing of this principle.
- Fowler, *Refactoring*, 2nd ed. — "Message Chains" as a named smell with a corresponding "Hide Delegate" refactoring.
