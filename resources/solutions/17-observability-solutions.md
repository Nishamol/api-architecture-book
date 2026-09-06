# Chapter 17: Observability — Solutions

*Corresponds to: [part5-cross-cutting/17a-observability-exercises.md](../../part5-cross-cutting/17a-observability-exercises.md)*

## Concept questions — Model answers

1. Without a correlation ID propagated across the REST gateway, GraphQL BFF, and two gRPC services, an incident produces four disconnected sets of logs and spans — each service knows what happened inside itself, but nothing ties those four views together into a single request's story. The chapter is explicit that reconstructing the path by manually correlating timestamps across four systems "is close to impossible to do reliably under real production traffic," because under real traffic there are many overlapping requests at similar timestamps, not one isolated event you can eyeball. With a shared correlation ID (the trace ID) attached to every log line, span, and metric label for that request, your tracing backend can assemble the four hops into one trace automatically — the ID is the only thing that makes "this log line in service A" and "this span in service C" provably the same request.

2. REST's request/response model maps cleanly onto a single span per call because a REST endpoint typically does one bounded unit of work per request. GraphQL breaks that assumption: a single query can invoke dozens of independent resolvers, each potentially doing its own downstream work. If you only have a query-level span, you can see that the query took 800ms total, but you have no way to attribute that time to a specific resolver — you know the *aggregate* is slow but not the *cause*. Per-resolver spans (via extensions/middleware hooks most GraphQL server libraries support) let you see that, say, 750 of those 800ms came from one specific field's resolver, turning "the query is slow" into "this resolver is slow," which is the difference between a symptom and an actionable diagnosis.

3. Naive auto-instrumentation that only wraps a stream's opening handshake produces a span that covers connection setup but goes dark for the actual lifetime of the stream — you'd see that the stream opened successfully and get no further signal until it closes (if it closes cleanly at all). What's lost is per-message latency visibility within the stream: if message 40 of 200 arrives late, or the stream stalls partway through, a handshake-only span can't show you where in the stream's life that happened. A unary-call span model is insufficient because a unary call has one meaningful duration (request to response), while a stream has an entire timeline of individual message events that need their own visibility — treating the stream like a single call collapses that timeline into one opaque interval.

4. In the p99-vs-average example, a latency regression that affects 1% of users can leave the average largely unchanged because it's diluted by the 99% of requests that are still fast — the average is dominated by the common case, not the tail. A symptom-based alert on p99 latency directly reflects what that 1% of users are actually experiencing, regardless of what caused it. A cause-based alert (e.g., "alert if function X's call count exceeds N") is tied to a specific implementation detail; when the implementation is refactored — the function is renamed, split, replaced, or the code path changes — the alert has to be manually rewritten or it silently stops meaning anything, even though the user-facing behavior it was meant to protect hasn't changed. Symptom-based alerts don't need this upkeep because they're defined in terms of outcomes (latency, error rate) that stay meaningful independent of how the internals are implemented.

5. Burn rate is how fast the error budget is being consumed relative to the time remaining in its window — the same 0.1% budget could be spent gradually and evenly across the full 30-day period (a slow leak) or almost entirely within a few minutes (a sharp spike), and both end in the same place: budget exhausted. The raw SLI's current value — "99.85% success right now" — looks identical in either scenario at any given instant, since it's a snapshot, not a trend; it can't tell you whether the current shortfall is part of a routine, slowly-accumulating pattern or an active incident that will blow through the entire month's budget in the next hour. Only tracking the rate of consumption against the time left in the window distinguishes "no page needed, this is fine over 30 days" from "page someone now, this is about to exhaust the budget."

## Design question — Model answer

Service-level RED metrics are aggregates *per service*, computed independently at each hop. A request that is individually slow across multiple hops — say, 150ms slower than typical at the gateway, 150ms slower at the BFF, and 150ms slower at each gRPC service — can add up to a very noticeably slow end-to-end request while each individual service's added latency is well within that service's normal p95/p99 variance and never trips a per-service alert. Aggregate RED metrics also average across many requests; they can't show you that *this specific request's* path through all four hops, taken together, was pathological, because that's a cross-service property, not a per-service one. Each service "looking healthy" in isolation is consistent with the request as a whole being slow — RED metrics per service simply don't have the shape needed to detect that.

The instrumentation to add:

- **Correlation key**: a single trace ID generated at the first hop (the REST gateway) that is propagated through every subsequent hop and attached to every log line, span, and metric label for that request — exactly the mechanism the chapter describes as what ties logs, metrics, and traces together.
- **OpenTelemetry propagation per boundary**: at the REST gateway and the GraphQL BFF (both HTTP-based), the trace context propagates via the `traceparent` HTTP header; when the BFF calls into the two internal gRPC services, the same trace context propagates via gRPC metadata instead — different wire mechanism, same logical trace ID, handled automatically by OpenTelemetry's auto-instrumentation for FastAPI/HTTP and for gRPC servers/clients.
- **What a trace view shows that per-service metrics can't**: a single assembled trace showing all four spans (gateway, BFF, gRPC service 1, gRPC service 2) nested or sequenced under one trace ID, with each span's individual duration visible side by side. This lets you see the request's actual critical path — which hop or hops accounted for the added latency — even when no single service's own aggregate metrics would have flagged anything, because the trace is scoped to one request's journey rather than averaged across many.

## Coding exercise — Model answer

```python
import structlog
from prometheus_client import Histogram

logger = structlog.get_logger()

order_lookup_duration = Histogram(
    "order_lookup_duration_seconds",
    "Duration of order lookups",
    ["operation"],  # labeled by operation name, not order_id
)

async def get_order(order_id: str, trace_id: str):
    logger.info("order_lookup_started", order_id=order_id, trace_id=trace_id)
    with order_lookup_duration.labels(operation="get_order").time():
        try:
            order = await fetch_order(order_id)
        except OrderNotFound:
            logger.warning("order_not_found", order_id=order_id, trace_id=trace_id)
            raise
    logger.info(
        "order_lookup_succeeded",
        order_id=order_id,
        trace_id=trace_id,
        status=order.status,
    )
    return order
```

Labeling the histogram by `order_id` would be a mistake because Prometheus label values create a new time series per unique combination of label values — `order_id` is high-cardinality (potentially millions of distinct values), and using it as a metric label would explode the number of stored time series, degrading Prometheus's storage and query performance. `order_id` belongs on the structured log lines, where high-cardinality fields are fine (and expected, since logs are queried by exact match, not aggregated into time series), not on metric labels, which should stay low-cardinality (`operation`, `method`, `status`) since they're meant to be aggregated and grouped over.

## Quiz (self-check) — Answers

1. **False.** OpenTelemetry provides a shared SDK and API surface, but the chapter explicitly describes "protocol-specific instrumentation gaps": REST maps cleanly to one span per call, GraphQL needs additional per-resolver instrumentation to get useful granularity (not automatic by default), and gRPC streaming needs spans that cover the whole stream lifetime rather than just the handshake, which naive auto-instrumentation commonly misses. A shared SDK does not mean zero protocol-specific work.
2. **Rate, Errors, Duration.**
3. **A p99 latency regression affecting a small fraction of users** (the chapter's example: 1%) — averaging hides it because it's diluted by the much larger population of fast, unaffected requests, so the average barely moves even though a real subset of users has a materially worse experience.
4. **HTTP-based calls propagate trace context via the `traceparent` header; gRPC calls propagate it via gRPC metadata.**
5. An SLO is a target for an SLI over a rolling window (e.g., 99.9% of requests succeed under 300ms, trailing 30 days), and the error budget is the gap between 100% and that target — the amount of unreliability the team has explicitly agreed is acceptable before it's a problem worth acting on.

