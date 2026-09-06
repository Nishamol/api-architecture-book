# Chapter 11: Building gRPC Services in Python — Solutions

*Corresponds to: [part4-grpc/11a-building-grpc-services-python-exercises.md](../../part4-grpc/11a-building-grpc-services-python-exercises.md)*

## Concept questions — Model answers

1. Per the chapter, "an unhandled Python exception propagating out of a servicer method gets caught by the gRPC runtime and surfaced to the client as an opaque `UNKNOWN` status." This is a debugging dead-end for the caller because `UNKNOWN` carries no information about *why* the call failed — the caller can't distinguish "the resource doesn't exist" from "a downstream dependency is down" from "the input was invalid," and therefore can't make a correct programmatic decision (retry vs. fail fast vs. surface a specific error to an end user). Whatever detail exists (a Python traceback, an exception message) lives on the server and typically doesn't cross the wire in a structured, callable form, so debugging becomes "go find the matching server log by request ID" instead of "branch on the status code," which is exactly the deliberate error-mapping the chapter argues for.

2. `grpc.aio` runs on Python's asyncio event loop: when `GetOrder` does `await fetch_order(...)`, the coroutine yields control back to the event loop instead of blocking a thread, letting thousands of other in-flight calls make progress on the same OS thread while this one waits on I/O. The older synchronous `grpc.server` model instead dispatches each call to a worker thread from a thread pool executor; a call that's blocked waiting on a database round-trip still occupies a full OS thread for the duration of that wait. Scaling concurrency under the thread-pool model means scaling thread count, which brings real per-thread overhead (stack memory, context-switch cost, and GIL contention when many threads are runnable) that asyncio's single-threaded, cooperative model avoids for I/O-bound handlers — which is most of what a servicer like `GetOrder` actually does.

3. If `authorization` metadata is missing or `is_valid_token` returns false, `intercept_service` returns a handler (`grpc.unary_unary_rpc_method_handler(deny)`) that, when invoked, immediately calls `context.abort(grpc.StatusCode.UNAUTHENTICATED, "invalid token")` — critically, it does this *instead of* calling `await continuation(handler_call_details)`, which is the call that would forward the request into `OrderService.GetOrder`. So the request never reaches the servicer at all. This differs from letting it reach `GetOrder` and fail there in three ways: the auth check doesn't need to be duplicated in every servicer method; none of `GetOrder`'s business logic or downstream calls (e.g., `fetch_order`) execute for a request that was always going to be rejected, saving real work; and the check is applied uniformly to every RPC the interceptor is registered for, rather than depending on each method's author remembering to add it.

4. Chapter 2 establishes that "every new TLS connection costs a handshake — one to three round trips depending on TLS version and session resumption support," and that "connection pooling is not an optimization, it's a requirement" for exactly this reason — a client that opens a fresh connection per request pays that handshake cost every single time. A gRPC channel wraps this same connection-establishment cost. Creating a channel per call (as the `async with` client pattern does when used by a high-frequency caller) means re-paying the TLS handshake — and general connection setup — on every call instead of amortizing it to near zero across the life of a long-lived channel, which is precisely the mechanism Chapter 2 says gRPC channels are designed around.

5. In `GetOrder`, the servicer method's second-to-last argument is a single `request` object — one message, available in full the moment the method is called. In `Report` (client streaming), the servicer instead receives `request_iterator`, an async iterator it must loop over (`async for metric in request_iterator`) — there's no single fixed request object, because the client controls how many messages it sends and when it's done, and the servicer only knows the stream is complete once the iterator is exhausted. That's also why the response is returned only after the loop ends, not per-message. A bidirectional method's directions are called "independent" rather than turn-based because nothing requires the servicer to consume one incoming message before producing one outgoing message, or vice versa — `Sync` both iterates `request_iterator` and yields replies, and the two happen concurrently as messages arrive and results become ready, unlike a strict request-then-response pattern where one side must wait for the other.

## Design question — Model answer

**Complaint 1 — opaque `UNKNOWN` on failed `GetOrder` calls.** This matches the chapter's "Failure modes" entry directly: "servicer methods that don't map domain errors to gRPC status codes explicitly, forcing callers to parse error message strings instead of branching on status codes." The fix is to replace implicit failures (raised exceptions, or any code path that doesn't call `context.abort` with a specific code) with explicit status mapping inside the servicer: `NOT_FOUND` when the order genuinely doesn't exist, and something like `UNAVAILABLE` (caught explicitly around the database call) when the failure is a downstream outage rather than a business-logic "no"). This lives inside `GetOrder` itself (or, if the same handful of exception types recur across many methods, could be centralized as a pattern in a wrapping interceptor that catches known exception types and maps them consistently) — either way, the point is that client retry logic can now branch on `context.code()` instead of inspecting message strings.

**Complaint 2 — pods serving before their cache is warm.** This matches the "Missing health checks" failure mode: "a Kubernetes deployment routing traffic to pods that are up but not actually ready to serve (e.g., still warming a cache), because readiness was inferred from process liveness instead of an actual health check." The fix is the standard health checking protocol described in the chapter: register a `health.HealthServicer()`, and don't call `health_servicer.set("orders.v1.OrderService", health_pb2.HealthCheckResponse.SERVING)` until the SKU cache has actually finished warming — leave it at `NOT_SERVING` (or unset) until that point. The Kubernetes readiness probe then needs to be wired against this gRPC health check (e.g., via `grpc_health_probe`) rather than a bare TCP or process-liveness check, so the orchestrator only sends traffic once the health servicer itself reports `SERVING`. This lives outside any individual RPC path — it's a separate registered service the orchestrator polls.

## Coding exercise — Model answer

As written, both `ValueError` branches are unhandled Python exceptions that propagate out of the coroutine. Per the chapter, the gRPC runtime catches both and surfaces `UNKNOWN` to the client in either case — a caller has no way to tell "no such order" apart from "bad status value" apart from any other unexpected failure; the only distinguishing information is whatever text happens to be in the exception, which isn't something a caller should have to parse to decide how to react.

```python
class OrderService(orders_pb2_grpc.OrderServiceServicer):
    async def UpdateOrderStatus(self, request, context):
        record = await fetch_order(request.order_id)
        if record is None:
            await context.abort(
                grpc.StatusCode.NOT_FOUND, f"no such order: {request.order_id}"
            )
        if request.new_status not in VALID_STATUSES:
            await context.abort(
                grpc.StatusCode.INVALID_ARGUMENT,
                f"invalid status: {request.new_status}",
            )
        updated = await apply_status_update(record, request.new_status)
        return orders_pb2.Order(
            id=updated["id"],
            status=updated["status"],
            line_items=[
                orders_pb2.LineItem(sku=li["sku"], quantity=li["quantity"])
                for li in updated["line_items"]
            ],
        )
```

`context.abort` unwinds the coroutine itself (matching the pattern shown in the chapter's own `GetOrder` example), so no explicit `return` is needed after either call. Now a client can branch meaningfully: `NOT_FOUND` means the order genuinely doesn't exist — stop retrying, surface a not-found state; `INVALID_ARGUMENT` means the caller sent bad data — a client-side bug that retrying won't fix. That leaves only a genuinely unexpected failure (e.g., `apply_status_update` itself raising) still falling through to implicit `UNKNOWN`, which is worth flagging as a follow-up: wrapping that call in a try/except that maps to `UNAVAILABLE` or `INTERNAL` would close the same gap the interceptor/complaint-1 discussion above addresses more generally.

## Quiz (self-check) — Answers

1. **True.** The interceptor's `intercept_service` can return a handler that aborts the call directly (as `AuthInterceptor` does via `deny`) without ever invoking `continuation(handler_call_details)` — and `continuation` is what forwards the request to the target servicer method, so skipping it means the servicer never runs.
2. The `context` object — the second positional argument passed to every servicer method (e.g., `context.abort(grpc.StatusCode.NOT_FOUND, "...")`).
3. `orders_pb2.py` (the generated message classes, e.g., `Order`, `GetOrderRequest`) and `orders_pb2_grpc.py` (the generated servicer base class the implementation subclasses, and the client stub) — both produced by the `protoc` invocation in Chapter 10 and used directly by the servicer and client code in this chapter.
4. **Server reflection** (`grpc_reflection`), which lets tools like `grpcurl` introspect a running service's schema without the `.proto` file. The chapter's caveat: "like GraphQL introspection, worth disabling on externally-exposed gRPC endpoints" — it's a debugging convenience for internal environments, not something you want an untrusted external caller to have.
5. It receives an async iterator of `Metric` messages (`request_iterator`), not a single request object. It returns its single `Ack` response only after the client's stream ends — i.e., once the `async for metric in request_iterator` loop is exhausted and `count` reflects every metric received.
