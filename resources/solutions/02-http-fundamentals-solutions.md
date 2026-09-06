# Chapter 2: HTTP Transport Internals — Solutions

*Corresponds to: [part1-foundations/02a-http-fundamentals-exercises.md](../../part1-foundations/02a-http-fundamentals-exercises.md)*

## Concept questions — Model answers

1. At the **HTTP/1.1 layer**, head-of-line blocking means a single connection can only have one request in flight at a time — even though pipelining exists in the spec, it's effectively unusable, so a slow request blocks every request queued behind it on that connection. HTTP/2 fixes this specific problem with stream multiplexing: many logical request/response streams share one TCP connection, interleaved at the frame level, so a slow response no longer blocks unrelated ones. But HTTP/2 doesn't fix head-of-line blocking that exists **one layer down, at TCP**: because all of HTTP/2's streams still share a single TCP connection, a single dropped TCP packet blocks every stream until it's retransmitted, regardless of which logical stream that packet belonged to. HTTP/3 is what fixes this remaining layer — by replacing TCP with QUIC, which does per-stream loss recovery over UDP, so a dropped packet only blocks the one stream it belongs to.

2. The 6-connections-per-host workaround exists because under HTTP/1.1, one connection = one request in flight, so the only way for a browser to fetch multiple resources from the same host concurrently is to open multiple separate TCP (and TLS) connections. This is a workaround for a limitation that HTTP/2 removes at the protocol level: HTTP/2 gives you many concurrent streams over a *single* connection, so opening 6 connections to the same host no longer buys any concurrency you didn't already have — it just adds 6x the TCP/TLS handshake overhead for no benefit, and can even hurt by fragmenting the congestion-control state across connections instead of one well-utilized one.

3. TCP guarantees strictly ordered, reliable delivery for the *entire connection* — it has no concept of "streams" at all, so when a packet is lost, TCP holds back every byte behind it, across every HTTP/2 stream multiplexed onto that connection, until the retransmission arrives. QUIC avoids this because it implements loss recovery itself, per-stream, at the transport layer it owns — a lost packet only stalls the one QUIC stream whose data it carried, and every other stream on the same QUIC "connection" keeps delivering. This directly avoids the failure mode named in the chapter's "Failure modes" section: intermittent latency spikes that get misdiagnosed as application-code problems when they're actually TCP-level retransmission stalls on a shared connection.

4. Every new TLS connection pays a handshake cost — one to three round trips depending on TLS version and session-resumption support — before a single byte of application data moves. If a REST client opens a fresh connection per request, it pays that handshake cost on *every single request*, which dominates latency for anything but large payloads; this isn't something you can optimize away without changing the behavior, which is why the chapter frames reuse as a requirement rather than a nice-to-have. gRPC channels take this to its logical conclusion: they're designed to be created once and reused for the life of the process specifically so the TLS (and TCP) handshake cost is paid once and amortized to near zero across every RPC made over that channel afterward, rather than once per RPC.

5. Under `max-age=60`, the client can reuse its cached copy for the full 60 seconds without sending any request at all — the resource is treated as fresh and no round trip happens. Under `no-cache`, the client must send a request every time, but that request can be a cheap conditional one (`If-None-Match` with the cached `ETag`) rather than a full fetch — the server either confirms nothing changed (`304`) or returns a fresh body. A `304 Not Modified` still saves real cost even with no body returned because the expensive part of many responses is the payload itself, not the round trip; skipping the body transfer (and, upstream, potentially skipping the work of regenerating it) is the actual savings, while the round trip's fixed cost was going to be paid either way once caching wasn't in play.

## Design question — Model answer

**Diagnosis:** The Layer 4 load balancer distributes TCP *connections*, not individual requests — it has no visibility into HTTP/2 framing. Under gRPC, a client typically opens one long-lived HTTP/2 connection and multiplexes many RPCs over it. The L4 balancer picked a backend pod for that connection once, at connect time, and every RPC multiplexed onto it since has ridden to that same pod — which is exactly the "gRPC load imbalance" failure mode the chapter names: long-lived HTTP/2 connections pinned to one backend by a Layer 4 load balancer, causing uneven CPU load across a service's pods. Total request volume didn't change; what changed is that HTTP/1.1's one-request-per-connection model spread load naturally across many short-lived connections, while gRPC's one-connection-many-streams model concentrates it onto whichever pod each client happened to connect to.

**Fix 1 (infrastructure layer):** Replace the Layer 4 load balancer with a Layer 7, HTTP/2-aware load balancer that parses up into the application layer and can route individual streams/RPCs independently rather than routing the connection as a whole. This is the fix the chapter calls out as the typical requirement for gRPC deployments.

**Fix 2 (client layer):** Use client-side load balancing instead of (or in addition to) an infrastructure LB — configure the gRPC client to resolve multiple backend addresses and distribute RPCs across several channels/connections itself, rather than relying on a single long-lived connection to one pod.

**Tradeoff:** Fix 1 is centralized and requires no client changes, but adds an infrastructure dependency (an L7/HTTP-2-aware proxy) that must itself scale and stay healthy. Fix 2 avoids that extra hop and its latency/operational cost, but pushes load-balancing logic and connection-management complexity into every client, which must be kept consistent across every service that calls this one.

## Coding exercise — Model answer

**The bug:** a new `grpc.insecure_channel(...)` is created on *every call* to `get_order`, which means a fresh TCP connection and TLS-equivalent setup cost is paid on every single request — directly contradicting the chapter's guidance that "a gRPC channel is meant to be created once and reused for the life of the process, not per-call." This is the same class of problem as a REST client opening a fresh connection per request: the handshake/setup cost that should be amortized to near zero gets paid in full, repeatedly, on the hot path.

**Fix** — create the channel and stub once, outside the per-request function, and reuse them:

```python
import grpc

_channel = grpc.insecure_channel("service.internal:50051")
_stub = OrderServiceStub(_channel)

def get_order(order_id):
    return _stub.GetOrder(GetOrderRequest(order_id=order_id))
```

This holds under concurrent requests because a gRPC channel is safe to share across concurrent calls — it's designed for exactly this usage pattern, multiplexing many concurrent RPCs over the same long-lived HTTP/2 connection, which is the whole reason gRPC channels are built the way they are.

## Quiz (self-check) — Answers

1. **False.** HTTP/2 eliminates head-of-line blocking *at the HTTP layer* via stream multiplexing, but head-of-line blocking still exists one layer down, at TCP: a single dropped packet blocks every multiplexed stream sharing that TCP connection until it's retransmitted. Only HTTP/3 (via QUIC) removes this remaining layer.
2. A Layer 4 load balancer operates at the **transport layer (TCP)**. It distributes connections without any understanding of HTTP semantics, so when balancing HTTP/2 traffic it is blind to the fact that a single connection may carry many independent multiplexed streams/requests — it can only route the connection as a whole, not the individual requests inside it.
3. HTTP/3 runs over **QUIC** (itself over UDP). The structural change is per-stream loss recovery: QUIC implements its own reliability per logical stream rather than relying on one ordered byte stream the way TCP does, so a lost packet only blocks the stream it belongs to instead of blocking every multiplexed stream on the connection.
4. Because `httpx.Client()` defaults to HTTP/1.1 unless `http2=True` is explicitly passed when constructing the client — the client's own configuration determines the protocol version used, independent of what the server is capable of supporting.
5. **False.** `private` means only the requesting client itself may cache the response — a shared cache like a CDN may not. `no-store` is stricter: nobody, including the client, may cache it at all. `private` is the right choice for personalized-but-cacheable data (cache it on the user's own device, just not on a shared proxy); `no-store` is for data that shouldn't be persisted anywhere, cached or not.
