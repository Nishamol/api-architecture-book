# Chapter 17: Observability — Exercises

*Corresponds to: [17-observability.md](17-observability.md)*

## Concept questions

1. The chapter says a correlation ID is what "ties together" structured logs, metrics, and traces. Using the example of a request that crosses a REST gateway, a GraphQL BFF, and two gRPC services, explain concretely what you're left with during an incident if that correlation ID isn't propagated across all four hops.
2. Why does GraphQL need per-resolver spans rather than a single span per query, when REST is fine with one span per call? Use the chapter's own example (a query taking 800ms) to explain what you can and can't diagnose without them.
3. The chapter says naive auto-instrumentation of gRPC streaming RPCs often "only wraps the RPC's opening handshake." What specifically do you lose visibility into as a result, and why is a unary-call span model insufficient for a long-lived stream?
4. Using the chapter's p99-vs-average example, explain why alerting on symptoms (elevated p99 latency, error rate) rather than causes (a specific function's call count) is more durable as a system evolves. What happens to a cause-based alert when the underlying implementation is refactored?
5. The chapter adds SLIs, SLOs, and error budgets, using a `GetOrder` 99.9%-under-300ms example. Explain what "burn rate" means for an error budget, and why alerting on the raw SLI value alone can't distinguish a slow, weeks-long budget leak from a sharp, minutes-long spike — even though both eventually exhaust the same budget.

## Design question (interview-style)

You're on-call for a platform where a single logical request can pass through a REST gateway, a GraphQL BFF, and two internal gRPC services (mirroring the chapter's own example). A customer reports that some requests are very slow, but your metrics dashboards — built entirely on RED-method aggregates per service — show every individual service's p50/p95/p99 within normal bounds.

Explain why service-level RED metrics alone can fail to surface this problem even when every service "looks healthy" in isolation. Then design the instrumentation you'd add, referencing specifically: what correlation key propagates across the four hops, what OpenTelemetry is doing at each protocol boundary (HTTP header vs. gRPC metadata), and what you'd want a trace view to show that a per-service metrics dashboard structurally cannot.

## Coding exercise

The structured-logging example in this chapter only logs two events — `order_lookup_started` and `order_not_found` — and never logs a successful lookup or records how long it took:

```python
import structlog

logger = structlog.get_logger()

async def get_order(order_id: str, trace_id: str):
    logger.info("order_lookup_started", order_id=order_id, trace_id=trace_id)
    try:
        order = await fetch_order(order_id)
    except OrderNotFound:
        logger.warning("order_not_found", order_id=order_id, trace_id=trace_id)
        raise
    return order
```

Extend this function so that it: (a) logs an `order_lookup_succeeded` event on success, including `order_id`, `trace_id`, and the resulting order's `status`; and (b) records the call's duration using a `prometheus_client.Histogram`, following the RED-method pattern shown later in the chapter (labeled by an operation name, not by `order_id` — explain in a comment why per-`order_id` labels would be a mistake).

## Quiz (self-check)

1. True or false: because OpenTelemetry is "the standard instrumentation layer across REST, GraphQL, and gRPC," you write instrumentation code once and it behaves identically regardless of protocol, with no protocol-specific gaps to worry about. *(Explain your answer.)*
2. What do the three letters in "RED method" stand for?
3. Per the chapter, what specific class of bug does averaging latency instead of looking at p99 tend to hide?
4. Name the two different wire mechanisms OpenTelemetry uses to propagate trace context, one for HTTP-based calls and one for gRPC calls.
5. In one sentence, what is the relationship between an SLO and an error budget?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
