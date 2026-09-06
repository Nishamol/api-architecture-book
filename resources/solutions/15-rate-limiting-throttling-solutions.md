# Chapter 15: Rate Limiting and Throttling — Solutions

*Corresponds to: [part5-cross-cutting/15a-rate-limiting-throttling-exercises.md](../../part5-cross-cutting/15a-rate-limiting-throttling-exercises.md)*

## Concept questions — Model answers

1. REST's request-per-URL model means each endpoint typically does a roughly bounded, predictable amount of work — the chapter says this "maps naturally onto a fixed-window or token-bucket limiter counting HTTP requests," because one request is a reasonable proxy for one unit of server-side cost. GraphQL breaks this because "a single request can be cheap... or extremely expensive... a deeply nested query touching thousands of underlying records" — the same "1 request" can represent wildly different amounts of backend work, so counting requests either "under-protects against expensive queries" (a client sends few requests but each is ruinous) "or over-throttles clients making many cheap ones" (a client sends lots of trivial requests and gets capped despite negligible load). The chapter's proposed alternative is rate-limiting by **cumulative query cost**, derived from the same complexity analysis used to bound individual queries in Chapter 9 — a client's budget is consumed proportionally to each query's complexity score, not a flat 1 token per call.

2. A token bucket held in a single process's memory only tracks the requests that process itself has seen. If a service runs, say, 10 replicas behind a load balancer and each replica independently enforces "100 req/min" using its own in-memory bucket, a client whose traffic is spread across all 10 replicas can send up to 100 requests to *each* replica per minute — 1,000 req/min in aggregate — because no replica has visibility into what the others allowed. The chapter's fix is to hold the limiter state in Redis (via `INCR` + `EXPIRE`, or a Lua script) so all replicas share one view of a given client's consumption. Atomicity matters because the check-then-decrement operation (read current tokens, verify enough remain, decrement) has to happen as one indivisible step — if two replicas concurrently read the same token count, both could pass the check and both decrement, over-admitting requests past the intended limit in exactly the same way the per-pod problem does, just at the Redis layer instead of the process layer. A Lua script (or Redis's atomic `INCR`) closes that race.

3. Fixed window: "100 requests per minute, resetting on the minute." A client sends 100 requests at 0:59 (the tail end of the first window) and another 100 requests at 1:00 (the instant the next window opens) — 200 requests inside roughly a two-second span, even though the long-run average never exceeds the stated 100/minute rate. A sliding window avoids this because it weights the previous window's count proportionally into the current calculation rather than resetting to zero at a hard boundary, so a burst straddling the boundary is still counted against a rolling 60-second span instead of being split across two independently-reset buckets.

4. The two alternatives are **concurrent open streams per client** and **messages per second within a stream**. A per-call token bucket assumes each call is a discrete, short-lived unit of work that consumes a token and finishes — the bucket's whole model is "count admissions, not duration." A `StreamOrderUpdates` call that "might stay open for hours" breaks that assumption because it's a single call (consuming, at most, one token at admission time) that then sends an effectively unbounded number of logical messages over its lifetime — the token bucket has no further visibility or control once the stream is open, so it can't protect against a client that opens few streams but pushes messages through them at an excessive rate, or against a client that opens many long-lived streams and holds server resources indefinitely.

## Design question — Model answer

1. **Diagnosis**: the flat 1-token-per-request limiter is protecting against a client making *too many requests* — it enforces a ceiling on request volume. It is blind to the *cost per request* — exactly the gap the chapter names: "a client staying well under a per-request quota while sending expensive queries that saturate the database — the limiter is technically working and the service is still degraded." The customer's small number of large, deeply nested queries is precisely this scenario: they're nowhere near 1000 tokens/minute, so the limiter reports them as compliant, while their actual backend cost per query is what's saturating the database and degrading service for others.

2. **Redesign**: replace the flat per-request cost with a per-query cost derived from Chapter 9's complexity analysis — the same static/structural cost function used there to bound individual queries (field count, nesting depth, list multipliers, etc.) should also be the unit the rate limiter charges against the customer's budget, rather than "1." Concretely: compute the query's complexity score at parse/validation time, *before* execution — the chapter is explicit that GraphQL servers must "defend themselves against expensive queries at request time" (Chapter 1) rather than after the fact, so the cost has to be assessed before the resolvers actually run and hit the database, not after the query has already done its damage. The token bucket then debits tokens proportional to that score rather than 1 per call — a cheap `{ order(id) { status } }` costs little, a deeply nested query costs proportionally more, and the 1000-token/minute budget now reflects actual backend load rather than raw call count.

3. Yes — there is an analogous distinction. A pure per-query cost budget (charge cost, refill continuously) still doesn't distinguish between a client spending its entire budget on one maximally expensive query at a single instant versus spreading equivalent total cost evenly across the minute. This chapter's sliding-window material is about smoothing *admission* over time to avoid boundary bursts; the same principle applies once cost, not request count, is the unit being smoothed — a token bucket with a large capacity and slow refill still allows a client to burn its whole budget in one enormous query at the start of every window, hitting the database with one massive spike rather than steady load, even though the same client would be "compliant" under a naive fixed-window-per-cost scheme. The fix is the same shape as the sliding-window fix described for request counts: track cost consumption on a rolling basis (or cap the bucket's burst capacity relative to typical query cost) rather than only capping total cost-per-fixed-window, so a single request's cost can't consume an entire window's budget in one spike.

4. Per the chapter's "Communicating limits to clients" section: since this is a GraphQL API riding on HTTP, use the REST convention headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`) but populate them with the *cost-based* budget rather than a raw request count — `X-RateLimit-Remaining` should reflect remaining cost tokens, not remaining request slots, so the customer's client can see its actual consumption rate change once the new scheme ships. If a query is rejected for exceeding the cost budget, return a `429` with a `Retry-After` header rather than a bare rejection — the chapter is explicit that omitting this "pushes clients toward either aggressive immediate retries... or overly conservative backoff," and this customer's team needs an explicit, actionable signal (their current dashboard shows plenty of "requests" remaining, so without a cost-aware readout they'll have no way to correlate the new throttling with their actual query pattern).

## Coding exercise — Model answer

**Bug 1 — wrong status code, no retry guidance.** The interceptor aborts with `grpc.StatusCode.PERMISSION_DENIED`, but the chapter is explicit: "gRPC's `RESOURCE_EXHAUSTED` is the correct signal for rate limiting, analogous to REST's `429 Too Many Requests` — using a generic error code instead makes it impossible for well-behaved clients to distinguish 'back off and retry' from 'this request is permanently invalid.'" `PERMISSION_DENIED` reads as a permanent authorization failure, not a transient, retry-after-backoff condition, so well-behaved clients correctly treat it as non-retryable... or, per the bug report, misbehave by retrying immediately anyway since they have no `Retry-After`-equivalent metadata telling them how long to wait. The fix uses `RESOURCE_EXHAUSTED` and attaches trailing metadata with retry guidance, per the chapter's "Communicating limits to clients" section.

**Bug 2 — deny path assumes unary.** `grpc.unary_unary_rpc_method_handler(deny)` always builds a unary-unary handler, regardless of whether the actual RPC being intercepted is unary or (server-)streaming. For a streaming RPC, gRPC's wire protocol expects a stream of messages back, not a single unary response — substituting a unary handler produces a response the client's streaming call can't correctly parse, which matches the reported "malformed responses to streaming clients."

Fixed version:

```python
import grpc

async def _deny(context):
    await context.abort_with_status(
        grpc.Status(
            code=grpc.StatusCode.RESOURCE_EXHAUSTED,
            details="rate limit exceeded",
            trailing_metadata=(("retry-after-ms", "1000"),),
        )
    )

class RateLimitInterceptor(grpc.aio.ServerInterceptor):
    def __init__(self, limiter):
        self.limiter = limiter

    async def intercept_service(self, continuation, handler_call_details):
        client_id = extract_client_id(handler_call_details)
        handler = await continuation(handler_call_details)

        if await self.limiter.allow(client_id):
            return handler

        # Build a deny handler of the SAME shape as the real one, so a
        # streaming call still gets a streaming-shaped handler back rather
        # than a unary response it can't parse.
        if handler.response_streaming:
            async def deny_stream(request, context):
                await _deny(context)
                return
                yield  # unreachable; marks this as an async generator

            return grpc.unary_stream_rpc_method_handler(
                deny_stream,
                request_deserializer=handler.request_deserializer,
                response_serializer=handler.response_serializer,
            )

        async def deny_unary(request, context):
            await _deny(context)

        return grpc.unary_unary_rpc_method_handler(
            deny_unary,
            request_deserializer=handler.request_deserializer,
            response_serializer=handler.response_serializer,
        )
```

The key fix is calling `continuation(handler_call_details)` first to obtain the real `handler`, checking `handler.response_streaming` to determine whether the actual RPC is unary or server-streaming (the only two shapes this service uses, per the prompt), and constructing a deny handler of the matching shape rather than always assuming unary-unary — plus switching the status code to `RESOURCE_EXHAUSTED` with `retry-after-ms` trailing metadata, addressing both reported bugs. A full production version would also handle client-streaming and bidi-streaming shapes (`request_streaming`), but this service doesn't use them.

## Quiz (self-check) — Answers

1. **False.** The chapter frames Redis (or another shared store) as a correctness requirement, not just a performance choice: "a service running multiple replicas needs a shared view of each client's consumption — an in-process limiter per pod effectively multiplies the real limit by the pod count." Without it, the enforced limit is wrong (too permissive), not merely slower to compute.
2. **`429 Too Many Requests`.** The chapter says using a generic error code instead of the specific rate-limit signal (`429` for REST, `RESOURCE_EXHAUSTED` for gRPC) "makes it impossible for well-behaved clients to distinguish 'back off and retry' from 'this request is permanently invalid,'" pushing them toward either aggressive immediate retries or overly conservative backoff.
3. **`X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`.**
4. The client stays "well under a per-request quota while sending expensive queries that saturate the database" — the service degrades even though the limiter, from its own (request-count) point of view, is functioning correctly.
5. Leaky bucket guarantees a strictly constant output rate — requests queue and are drained at that fixed rate no matter how bursty their arrival, so whatever sits downstream never sees anything faster than that rate. It gives up burst tolerance to provide this: a legitimate short burst that a token bucket would let through immediately (up to its capacity) instead gets smoothed into a steady drip under leaky bucket, adding latency to that burst even when the system momentarily has spare capacity to have served it faster.
