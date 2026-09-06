# Chapter 2: HTTP Transport Internals — Exercises

*Corresponds to: [02-http-fundamentals.md](02-http-fundamentals.md)*

## Concept questions

1. Explain what "head-of-line blocking" means at the HTTP/1.1 layer versus at the TCP layer. Why does HTTP/2 fix one but not the other, and what protocol fixes the one HTTP/2 leaves behind?
2. Browsers historically open up to 6 parallel connections per host under HTTP/1.1. Why is this workaround specific to HTTP/1.1's limitations, and why does the chapter call it "a hack that HTTP/2 makes obsolete"?
3. QUIC runs over UDP and re-implements loss recovery itself, per-stream, instead of relying on TCP. Using the chapter's explanation, describe the specific failure mode this design avoids that HTTP/2-over-TCP cannot avoid.
4. The chapter says TLS connection reuse "is not an optimization, it's a requirement." Explain why, and connect this directly to why gRPC channels are designed to be long-lived.
5. The chapter's "HTTP caching" section distinguishes `Cache-Control: max-age=60` from `Cache-Control: no-cache`. Explain what a client does differently on its *next* request to the same URL under each directive, and why a `304 Not Modified` response still saves real cost even though it returns no body.

## Design question (scenario-style)

A team migrates an internal service-to-service REST API — previously HTTP/1.1, sitting behind a standard cloud Layer 4 (TCP) load balancer — to gRPC. After the migration, monitoring shows one pod pinned at 90% CPU while five others sit nearly idle, even though total request volume is unchanged.

Using the chapter's explanation of L4 vs L7 load balancing and HTTP/2 multiplexing, diagnose what's happening. Then propose two different fixes that operate at two different layers of the stack, and explain the tradeoff between them.

## Coding exercise

The function below is called on every incoming request to fetch order data from a downstream gRPC service:

```python
import grpc

def get_order(order_id):
    channel = grpc.insecure_channel("service.internal:50051")
    stub = OrderServiceStub(channel)
    return stub.GetOrder(GetOrderRequest(order_id=order_id))
```

Identify the bug relative to the guidance in the chapter's "TLS overhead and connection reuse" section, explain what cost this pays on every single call that shouldn't be paid repeatedly, and rewrite the code so the fix holds even under concurrent requests.

## Quiz (self-check)

1. True or false: HTTP/2 eliminates head-of-line blocking entirely. *(Explain your answer.)*
2. What layer does a Layer 4 load balancer operate at, and what is it structurally blind to when balancing HTTP/2 traffic?
3. Name the transport protocol HTTP/3 runs over, and the one structural change it makes that avoids HTTP/2's TCP-level head-of-line blocking.
4. In the chapter's first code example, `response.http_version` prints `"HTTP/1.1"` even though the server may support HTTP/2. Why?
5. True or false: setting `Cache-Control: private` on a response has the same effect as `no-store`. *(Explain your answer.)*

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
