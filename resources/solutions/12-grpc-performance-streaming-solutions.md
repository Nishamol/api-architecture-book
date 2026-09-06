# Chapter 12: gRPC Performance and Streaming — Solutions

*Corresponds to: [part4-grpc/12a-grpc-performance-streaming-exercises.md](../../part4-grpc/12a-grpc-performance-streaming-exercises.md)*

## Concept questions — Model answers

1. The three factors: (1) Protocol Buffers' binary encoding, which is smaller and faster to (de)serialize than JSON because there's no string parsing and no key names on the wire; (2) HTTP/2 multiplexing, which lets many concurrent RPCs share one connection without application-layer head-of-line blocking; and (3) connection reuse, which amortizes TLS handshake cost across the service's entire lifetime instead of paying it per request. The gap against REST-over-HTTP/1.1-JSON is typically large because that baseline is missing all three — it pays connection/TLS setup cost repeatedly, suffers HTTP/1.1's head-of-line blocking, and parses JSON. The gap against a REST API that's already adopted HTTP/2 and a binary format is much smaller because that comparison has already captured factors (2) and (3) through HTTP/2 adoption and presumably improved on (1) via binary encoding — so what's left to measure is the genuinely protocol-specific difference, not the difference in transport-layer discipline, which the chapter says is where most of a raw "gRPC is N times faster" number actually comes from.

2. `await context.write(update)` blocks on the write actually being accepted by the transport — practically, on the client being ready to receive more data (i.e., on room existing in the underlying HTTP/2 flow-control window). Because it's a genuine `await` on an I/O operation rather than a synchronous append to an in-memory buffer, the coroutine driving the loop is suspended exactly when the transport can't move data any faster than the consumer is reading it. That suspension *is* backpressure — the producer can't outrun the consumer because the language-level `await` won't return control back to the loop until the transport says it's ready, with no separate rate-limiting or buffering logic required.

3. It protects against continuing to do work — pulling from `order_update_source`, running whatever query or subscription backs it — for a client that has already disconnected or exceeded its deadline. Without the check, the chapter says the server "keeps consuming resources (**DB connections, CPU**) for a stream nobody is reading," which it calls "a very common resource leak in poorly-implemented streaming servicers."

4. Per Chapter 2, a Layer 4 load balancer "picks a backend once, when the TCP connection opens" and "has no visibility into the HTTP/2 frames flowing through it." `ClusterIP` performs exactly this kind of connection-level distribution: it assigns a backend pod when a TCP connection is established and has no awareness of the individual RPCs multiplexed inside it afterward. For gRPC's small number of long-lived connections between services, this means every RPC riding a given connection — potentially for the connection's entire lifetime — lands on whichever single pod was picked at connection time, so a handful of persistent connections can concentrate all their traffic on a handful of pods while the rest of the fleet sits idle, rather than the load spreading evenly at the RPC level the way Layer 7-aware balancing would achieve.

## Design question — Model answer

**(a) Hot pods.** This is the chapter's Layer 4/`ClusterIP` load-balancing trap: long-lived connections pinned to individual pods at connection time, so traffic concentrates rather than spreading evenly. Fix: either client-side load balancing (edge devices resolve multiple backend addresses via a headless service/DNS and distribute their RPCs themselves) or a Layer 7 proxy such as Envoy in front of the service that distributes at the RPC level rather than the connection level.

**(b) Compression CPU cost on small messages.** This is the "Compression applied indiscriminately" failure mode — compression "trades CPU for bandwidth," and is "largely wasted overhead for small, low-latency internal service calls where the CPU cost of compression exceeds the bytes saved." A few-hundred-byte `Metric` message is exactly this case. Fix: disable gzip compression on this channel — it's appropriate for large payloads over constrained networks (the chapter's example is a mobile client through a gRPC-Web gateway), not high-frequency small telemetry messages between internal services.

**(c) Pods ignoring termination during rollout.** This generalizes the chapter's cancellation discussion: a streaming servicer that doesn't notice it should stop is the same underlying failure as ignoring `context.cancelled()`, just triggered by server shutdown instead of client disconnect. Fix: ensure the server's graceful-shutdown path cancels in-flight streaming contexts (`grpc.aio` server graceful stop with a bounded grace period) and that the `Report` servicer's ingestion loop checks `context.cancelled()` so it exits promptly on shutdown instead of running until each device's stream naturally ends on its own.

**Why fixing (a) alone first could make things look worse:** right now the wasted compression CPU from (b) is concentrated on the same three pods that are already hot — everyone else is nearly idle. If you rebalance the *connections* (fixing (a)) without removing the unnecessary compression cost (b), you spread that same total wasted CPU work evenly across all twelve pods instead of three. Average per-pod CPU rises fleet-wide even though total traffic hasn't changed, because pods that previously had near-zero load are now carrying their share of a cost that was always unnecessary. The visible, easy-to-alert-on symptom (three hot pods) goes away, but it's replaced by a less visible, harder-to-notice reduction in headroom across the entire fleet — which is worse, not better, the next time a traffic burst hits, because there's no longer a set of idle pods with slack to absorb it.

## Coding exercise — Model answer

Two problems, both directly contradicting the chapter's guidance:

1. **No cancellation check.** There's no `if context.cancelled(): break`, so when a client disconnects mid-stream, the loop keeps iterating `order_update_source` and keeps attempting writes — exactly the "very common resource leak" the chapter warns about, where the server "keeps consuming resources (DB connections, CPU) for a stream nobody is reading."
2. **Unbounded buffering.** The `updates.append(update)` line accumulates every update ever sent into a list that lives for the entire duration of the stream. Nothing in the chapter's own `StreamOrderUpdates` example buffers past updates at all — each one is written and then eligible for garbage collection. Combined with problem (1), a disconnected load-test client leaves behind a coroutine that keeps growing this list indefinitely (or at least until the underlying source is exhausted), which is the direct cause of the unbounded memory growth observed under load testing.

Corrected version:

```python
async def StreamOrderUpdates(self, request, context):
    async for update in order_update_source(request.order_id):
        await context.write(update)
        if context.cancelled():
            break
```

This drops the buffering list entirely and restores the cancellation check, so the loop exits as soon as the client disconnects or the deadline passes, releasing whatever `order_update_source` holds (a DB cursor, a subscription, etc.) instead of continuing to pull and silently discard updates for an abandoned stream.

## Quiz (self-check) — Answers

1. **False.** Compression "trades CPU for bandwidth" — the chapter is explicit that it's "worthwhile for large payloads over constrained networks" but "largely wasted overhead for small, low-latency internal service calls where the CPU cost of compression exceeds the bytes saved." Whether it's a win depends on message size and where the actual bottleneck is (network vs. CPU), not on bandwidth mattering in the abstract.
2. Client-side load balancing (the client resolves multiple backend addresses and distributes RPCs itself) and a Layer 7 proxy such as Envoy that understands HTTP/2 streams and distributes at the RPC level.
3. HTTP/2 **flow control windows** (per-stream and per-connection) — a receiver advertises how much unacknowledged data it will buffer, and the sender must respect that window. Tune via `grpc.http2.max_frame_size` and related channel options.
4. It conflates three separate, independently variable contributions — Protobuf vs. JSON encoding, HTTP/2 vs. HTTP/1.1 multiplexing, and warm/reused vs. cold per-request connections — into one number, without holding the transport layer constant or stating which layer's contribution is being measured. Most of a "10x" result under those conditions comes from HTTP/2 adoption and connection-management discipline, not from gRPC or Protobuf specifically.
