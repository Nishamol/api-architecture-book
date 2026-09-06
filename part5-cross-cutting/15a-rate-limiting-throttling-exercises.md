# Chapter 15: Rate Limiting and Throttling — Exercises

*Corresponds to: [15-rate-limiting-throttling.md](15-rate-limiting-throttling.md)*

## Concept questions

1. The chapter opens by arguing "requests per second" isn't the right unit for every protocol. Walk through why request-count-based limiting is a reasonable default for REST but breaks down for GraphQL, and describe the alternative unit the chapter proposes for GraphQL.
2. Explain why the chapter says an in-process token bucket "effectively multiplies the real limit by the pod count" for a service running multiple replicas, and describe the fix it recommends, including why atomicity matters for that fix.
3. Using the chapter's fixed-window example ("100 requests per minute, resetting on the minute"), construct the specific request timing that allows 200 requests in a much shorter span than a minute, and explain in one sentence how a sliding window avoids it.
4. For gRPC streaming RPCs, the chapter says "calls per second" is the wrong metric and proposes two alternatives. Name both, and explain why a `StreamOrderUpdates` call that "might stay open for hours" breaks the assumption baked into a per-call token bucket.

## Design question (scenario-style)

Your platform exposes a public GraphQL API rate-limited today with a flat token bucket: 1 token per incoming GraphQL request, 1000 tokens/minute per API key, refilled continuously, state held in Redis. A key customer complains that their integration — which sends a small number of large, deeply nested queries per minute, well under 1000 requests — is causing your database to show sustained high load and slow query times for *other* customers, even though the offending customer's dashboard shows they're nowhere near their rate limit.

1. Diagnose the mismatch using the chapter's own terms — what is the flat-request-count limiter actually protecting against, and what is it blind to?
2. Redesign the limiting scheme for this API using material from this chapter (referencing Chapter 9's complexity analysis, which this chapter explicitly builds on). Be specific about what a request's "cost" should be computed from and when in the request lifecycle that cost should be assessed.
3. The chapter also covers rate limiting for gRPC streams via concurrent-streams-plus-messages-per-second rather than calls/second. Is there an analogous distinction for GraphQL between "cost of one query" and some other unit that a pure per-query cost budget would still miss (hint: think about what a client can do across many small, cheap requests versus one expensive one, and what the chapter's sliding-window material implies about smoothing consumption over time)? Justify your answer.
4. Whatever scheme you land on, describe how you'd communicate the new budget to the affected customer's client using the chapter's guidance on communicating limits, so their integration can adapt instead of hitting a wall with no explanation.

## Coding exercise

The following is a first-pass implementation of a `RateLimitInterceptor` for a gRPC service that mixes unary and server-streaming RPCs, loosely adapted from the chapter's interceptor pattern:

```python
class RateLimitInterceptor(grpc.aio.ServerInterceptor):
    def __init__(self, limiter):
        self.limiter = limiter

    async def intercept_service(self, continuation, handler_call_details):
        client_id = extract_client_id(handler_call_details)
        if not await self.limiter.allow(client_id):
            async def deny(request, context):
                await context.abort(grpc.StatusCode.PERMISSION_DENIED, "slow down")
            return grpc.unary_unary_rpc_method_handler(deny)
        return await continuation(handler_call_details)
```

This interceptor is now failing in two ways in production: (1) well-behaved clients that get rate-limited report they can't tell when to retry and start hammering the service with immediate retries, making things worse; (2) it silently breaks every streaming RPC on the service, returning malformed responses to streaming clients even when the limiter allows the call. Identify both bugs using the chapter's guidance, and fix them — for bug (2), you don't need to implement full streaming rate-limit logic, but the deny path must not respond to a streaming call as if it were unary.

## Quiz (self-check)

1. True or false: using Redis instead of an in-process counter for a token bucket is only a performance optimization, not a correctness requirement. *(Explain your answer.)*
2. What HTTP status code does the chapter identify as REST's analog to gRPC's `RESOURCE_EXHAUSTED`, and why does the chapter say returning a generic error code instead of this specific one is harmful to well-behaved clients?
3. Name the three `X-RateLimit-*` headers the chapter lists as the REST convention for communicating budget to clients.
4. According to the chapter's failure modes, what specific outcome results from "request-count limiting on GraphQL" even when the limiter is "technically working"?
5. The chapter adds leaky bucket as a fourth algorithm alongside token bucket, fixed window, and sliding window. Unlike token bucket, what guarantee does leaky bucket provide, and what does it give up to provide it?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
