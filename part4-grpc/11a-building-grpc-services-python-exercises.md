# Chapter 11: Building gRPC Services in Python — Exercises

*Corresponds to: [11-building-grpc-services-python.md](11-building-grpc-services-python.md)*

## Concept questions

1. In the `GetOrder` servicer example, the code calls `await context.abort(grpc.StatusCode.NOT_FOUND, "order not found")` instead of `raise OrderNotFoundError(...)`. Explain what the gRPC runtime does with an unhandled Python exception that propagates out of a servicer method, and why that's "a debugging dead-end" for the caller.
2. The chapter describes `grpc.aio` as "a natural choice for I/O-bound Python services that already use `asyncio`," contrasted with the older synchronous `grpc.server` with a thread pool executor. Explain the concurrency model difference between the two for an I/O-bound handler like `GetOrder`, which awaits a database call — and explain why the chapter doesn't call either one universally correct.
3. Walk through what `AuthInterceptor.intercept_service` does when a request arrives with no `authorization` metadata: what does it return, and how is that different from just letting the request reach `OrderService.GetOrder` and failing there?
4. The chapter says a gRPC channel is "expensive to set up and cheap to reuse," and recommends a long-lived channel (module-level singleton or connection pool) over the `async with` pattern shown in the client example for high-frequency callers. Connect this back to the specific mechanism from Chapter 2 that makes per-call channel creation expensive.
5. The chapter adds client-streaming and bidirectional servicer implementations (`Report`, `Sync`). Explain what changes about the servicer method's signature for client streaming compared to the unary `GetOrder` example, and why a bidirectional method's read and write directions are described as "independent" rather than turn-based.

## Design question (scenario-style)

You're on-call for `OrderService`. Two independent complaints land in the same week: (1) client teams say every failed `GetOrder` call surfaces as gRPC status `UNKNOWN` with no useful detail, so their retry logic can't tell a missing order apart from a downstream database outage; (2) the SRE team reports that during the last two deploys, a handful of pods received live traffic for 10-15 seconds before they were actually ready (their in-memory SKU cache hadn't finished warming), causing a burst of failed requests right after each rollout.

Using this chapter's coverage of exception handling, interceptors, and health checking, diagnose the likely root cause of each complaint and propose a specific fix for each — citing which mechanism from the chapter you'd use and where in the request path it belongs.

## Coding exercise

A junior engineer wrote this servicer method, modeled on the chapter's `GetOrder` example but for updating an order's status:

```python
class OrderService(orders_pb2_grpc.OrderServiceServicer):
    async def UpdateOrderStatus(self, request, context):
        record = await fetch_order(request.order_id)
        if record is None:
            raise ValueError(f"no such order: {request.order_id}")
        if request.new_status not in VALID_STATUSES:
            raise ValueError(f"invalid status: {request.new_status}")
        updated = await apply_status_update(record, request.new_status)
        return orders_pb2.Order(
            id=updated["id"],
            status=updated["status"],
            line_items=[...],
        )
```

Identify what a calling client sees today for each of the two `ValueError` cases, explain why that's a problem for a caller trying to branch on the outcome, and rewrite the method so each failure maps to an appropriate, distinguishable gRPC status code.

## Quiz (self-check)

1. True or false: an interceptor registered on a `grpc.aio.server()` can reject a request before it ever reaches the target servicer method. *(Explain your answer.)*
2. What Python object does a servicer method use to set a gRPC status code and detail message on an error response?
3. Name the two pieces of generated code, from this chapter and Chapter 10 combined, that a servicer implementation depends on and extends.
4. What tool does the chapter mention that can introspect a running gRPC service's schema without the `.proto` file on hand, and what's the chapter's caveat about using it on externally-exposed services?
5. In the chapter's new `Report` client-streaming example, what does the servicer receive instead of a single request message, and when does it return its one response?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
