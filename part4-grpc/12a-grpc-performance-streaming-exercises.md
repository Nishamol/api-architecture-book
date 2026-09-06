# Chapter 12: gRPC Performance and Streaming — Exercises

*Corresponds to: [12-grpc-performance-streaming.md](12-grpc-performance-streaming.md)*

## Concept questions

1. The chapter names three factors that compound to produce gRPC's performance advantage. List them, and explain why the chapter says the performance gap "is typically large" against REST-over-HTTP/1.1-JSON but "much smaller" against a REST API that has already adopted HTTP/2 and a binary format.
2. In the `StreamOrderUpdates` example, explain what `await context.write(update)` actually blocks on, and why the chapter says `grpc.aio`'s streaming writes "naturally apply backpressure" without any additional code from you.
3. What does `if context.cancelled(): break` protect against in a server-streaming servicer, and what specifically happens to server-side resources (the chapter names two kinds) if that check is missing?
4. The chapter says a plain Kubernetes `ClusterIP` service is "a known trap for gRPC that teams migrating from REST reliably hit once." Using the Layer 4 vs. Layer 7 distinction from Chapter 2, explain what `ClusterIP` load balancing actually does to a small number of long-lived gRPC connections between services.

## Design question (interview-style)

Your team runs a telemetry-ingestion service: thousands of edge devices open long-lived client-streaming RPCs (`Report(stream Metric) returns (Ack)`) to push sensor readings continuously. It's deployed on Kubernetes behind a plain `ClusterIP` service. Over the past month, three problems have shown up together: (a) three of twelve pods consistently run hot while the rest idle; (b) p99 latency on `Report` calls spikes during traffic bursts, and profiling shows CPU time going into gzip compression on every message even though individual `Metric` messages are only a few hundred bytes; (c) during a recent rolling deploy, several pods kept processing device streams for almost a minute after Kubernetes sent them a termination signal, delaying the rollout.

For each problem, name the chapter concept that explains it and propose a concrete fix. Then explain why fixing only (a) without also addressing (b) could make the load imbalance in (a) look worse, not better, immediately after the fix ships.

## Coding exercise

This `StreamOrderUpdates` implementation is deployed and, under load testing, server memory grows without bound whenever a load-testing client disconnects mid-stream without a clean close:

```python
async def StreamOrderUpdates(self, request, context):
    updates = []
    async for update in order_update_source(request.order_id):
        updates.append(update)
        await context.write(update)
    return
```

Identify two distinct problems with this code relative to the chapter's `StreamOrderUpdates` example and its discussion of backpressure and cancellation, and rewrite it correctly.

## Quiz (self-check)

1. True or false: enabling gzip compression on a gRPC channel is a strict win whenever bandwidth matters. *(Explain your answer.)*
2. Name the two standard fixes the chapter gives for Layer 4 load balancers mishandling long-lived gRPC connections.
3. What HTTP/2-layer mechanism does the chapter say a receiver uses to tell a sender how much unacknowledged data it's willing to buffer, and what channel option would you tune to adjust it for a high-throughput streaming service?
4. According to the chapter, what's the methodological mistake in a benchmark that concludes "gRPC is 10x faster than REST" by comparing gRPC against a REST API still running HTTP/1.1 with cold connections per request?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
