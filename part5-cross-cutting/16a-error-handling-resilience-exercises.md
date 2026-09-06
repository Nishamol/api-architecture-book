# Chapter 16: Error Handling and Resilience — Exercises

*Corresponds to: [16-error-handling-resilience.md](16-error-handling-resilience.md)*

## Concept questions

1. The chapter argues that machine-readable `code` values matter more than human-readable `message` strings for error responses. Explain why, using the specific failure mode the chapter warns about when client code doesn't follow this guidance.
2. The chapter says retrying is "actively harmful for non-idempotent operations without an idempotency key" and "pointless for permanent failures." For each of these two categories, give an example error/status code from the chapter's own material and explain concretely what goes wrong if a client retries anyway.
3. Explain what the `random.uniform(0, base_delay)` jitter term in the chapter's `call_with_retry` function is defending against, using the chapter's own description of the scenario it prevents. What specifically would happen without it?
4. A circuit breaker's `HALF_OPEN` state is described as "testing recovery." Walk through, using the `CircuitBreaker` class in the chapter, exactly what happens on the next call after the cooldown period elapses — what state is the breaker in, what happens if that call succeeds, and what happens if it fails.
5. The chapter adds bulkheads alongside circuit breakers, using separate `asyncio.Semaphore` instances per downstream dependency. Explain what specific resource-exhaustion scenario a bulkhead prevents that a circuit breaker alone does not, and why the chapter says the two patterns are "usually deployed together."

## Design question (interview-style)

Your `orders` service calls `billing`, which calls `payment-gateway` (three hops total: client → orders → billing → payment-gateway). Each service independently sets a 5-second timeout on its outbound call to the next hop, chosen because "5 seconds felt safe" — no service in the chain is aware of how much time upstream callers have already spent waiting.

1. Using the chapter's "Timeouts at every layer" section, explain precisely how this configuration can let the end-to-end request take up to 15 seconds even though the original client-facing timeout budget was intended to be 5 seconds. Be specific about which hop's delay compounds with which.
2. The chapter names deadline propagation (cross-referencing Chapter 10) as "the more correct model." Redesign the timeout strategy for this three-hop chain using deadline propagation instead of independent per-hop timeouts, and explain what data has to flow from `orders` to `billing` to `payment-gateway` for this to work.
3. Deadline propagation alone doesn't protect `orders` from a `billing` dependency that is failing outright rather than just slow. What additional mechanism from this chapter should sit in front of the `billing` call, and what specific harm does it prevent that a deadline alone does not?
4. Suppose `payment-gateway`'s failure only affects the "charge card" step, and `orders` also independently calls a `recommendations` service to populate a "you might also like" section on the same response. Using the chapter's material on graceful degradation, explain why these two downstream failures should be handled with different strategies in the response to the client, and describe what each strategy looks like concretely.

## Coding exercise

The following endpoint handler charges a customer's card via a downstream `payment-gateway` call and is wrapped in the chapter's retry helper:

```python
async def charge_customer(order_id: str, amount: int):
    async def attempt():
        return await payment_gateway.charge(order_id=order_id, amount=amount)

    return await call_with_retry(attempt, max_attempts=3, base_delay=0.1)
```

Assume `payment_gateway.charge` raises `TransientError` on a network blip or a `503` from the gateway, exactly the case `call_with_retry` is designed to retry. Using the chapter's failure modes section, identify why this code is dangerous as written despite correctly using the chapter's own retry helper, and rewrite `charge_customer` (and, if needed, the call signature into `payment_gateway.charge`) so that retries are safe. You do not need to implement the idempotency-key storage/lookup itself — assume a function `generate_idempotency_key(order_id: str) -> str` exists — but show exactly where it's generated, where it's passed, and why it must be generated once per logical charge attempt rather than once per retry.

## Quiz (self-check)

1. True or false: a circuit breaker and a timeout solve the same problem, so a well-configured timeout on every outbound call makes a circuit breaker redundant. *(Explain your answer.)*
2. In the chapter's error envelope example (`ORDER_NOT_FOUND`), what does the `"retryable": false` field communicate to a client, and why does putting this in the envelope matter more than a client trying to infer retryability from the HTTP status code alone?
3. Name the three `CircuitState` values in the chapter's `CircuitBreaker` implementation and, in a few words each, what each one means for whether a call is allowed through.
4. According to the chapter's failure modes, what specific resource-exhaustion chain results from having "no circuit breaker on a critical dependency"?
5. True or false: a bulkhead and a circuit breaker protect against the same underlying failure — a slow or failing downstream dependency — so implementing one makes the other redundant. *(Explain your answer.)*

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
