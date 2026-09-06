# Chapter 18: Testing Strategies — Solutions

*Corresponds to: [part5-cross-cutting/18a-testing-strategies-exercises.md](../../part5-cross-cutting/18a-testing-strategies-exercises.md)*

## Concept questions — Model answers

1. The in-process `FakeServicerContext` test calls the servicer's method directly in Python — no network, no serialization, no real gRPC channel involved. It's fast and hermetic, and it's excellent for verifying business logic and status-code behavior (e.g., that a missing order raises `grpc.StatusCode.NOT_FOUND`), but because it never actually serializes the request/response through Protobuf or sends bytes over HTTP/2, it cannot catch serialization bugs (a field that doesn't round-trip correctly through the wire format) or interceptor ordering issues (interceptors are wired into the real gRPC server/channel stack, not invoked by a direct method call). The real `grpc.aio.server()` integration test, bound to an ephemeral port with a real client stub connecting to it, exercises the actual wire protocol end to end — it's slower, but it's the only one of the two that can catch those two specific classes of bug, which is exactly why the chapter says both are worth having rather than picking one.

2. GraphQL's execution model allows a query to resolve some fields successfully while other fields fail — the response can contain both `data` (with the successful parts populated) and a non-null `errors` array (describing what failed and where). Because this is normal, expected behavior for GraphQL rather than an edge case, client code has to be written to handle it correctly in production — checking `errors` and rendering whatever `data` did resolve rather than treating any non-empty `errors` array as a total failure. A test suite that only exercises `test_order_query_returns_line_items` (asserting `result.errors is None`) only ever verifies the fully-successful path; it never proves that a partial failure produces the shape (specific fields in `data` still populated, `errors` containing entries with the correct `path`) that client code is relying on. Testing partial failure explicitly is testing the exact contract clients depend on, not just the easy case.

3. Per-layer tests, even at 100% coverage, each validate a layer against its own assumptions about what the adjacent layer sends or expects — the REST gateway's tests assume a certain shape from the gRPC backend, the gRPC backend's tests validate its own service logic, and neither test suite actually calls the other layer and checks that their shared assumption is true. Contract testing across protocol boundaries specifically verifies that the translation between layers is correct — that the gateway's REST response shape genuinely matches what the gRPC backend returns, not what the gateway's test author assumed it returns. This is the category of bug per-layer tests structurally cannot catch, because each layer's tests are, by construction, only testing that layer in isolation.

4. Generic HTTP/1.1 load generators don't understand HTTP/2 multiplexing — they'd typically open one connection per virtual user and serialize requests on it the way HTTP/1.1 clients do, which doesn't exercise how gRPC's real traffic pattern behaves (multiple concurrent streams multiplexed over one connection). This connects directly to Chapter 12's load-balancing pitfalls: if your load test doesn't multiplex requests the way a real gRPC client does, it won't reveal the connection-pinning and uneven-load-distribution problems that Layer 4 load balancers create for HTTP/2 traffic, because the test traffic pattern itself doesn't resemble production traffic closely enough to expose them. `ghz`, being purpose-built for gRPC, generates traffic that actually exercises HTTP/2 multiplexing and gRPC's call semantics.

## Design question — Model answer

Each team's test suite passed because each suite only validates its own layer against its own author's assumptions about the neighboring layer. The gateway team's tests presumably mock or stub what the gRPC backend returns, and the BFF team's tests presumably mock what the gateway expects from it — neither actually invoked the real gRPC backend and checked, end to end, that a field it returns survives all the way to the REST response. This is exactly the gap the chapter calls "the most valuable and most commonly neglected test category": tests exist for each layer, but nothing exists for the translation between them.

The fix is a genuine cross-layer contract test, sitting logically between the integration and end-to-end levels of the pyramid: it calls the real (or a realistic ephemeral) gRPC backend, takes its actual response, passes it through the real BFF resolver logic, and asserts that the specific field in question survives into the final REST response the gateway produces. Concretely, this could be built as a Pact-style consumer-driven contract (Chapter 6): the gateway (as consumer of the BFF/backend) records the exact fields and shapes it depends on, and that contract is verified against the real provider on every change to either side, so a field silently dropped by an intermediate layer breaks contract verification in CI rather than surfacing as a customer-facing bug.

## Coding exercise — Model answer

```python
import pytest

@pytest.mark.asyncio
async def test_order_query_partial_failure_on_customer_lookup():
    query = """
        query {
            order(id: "42") {
                id
                lineItems { sku quantity }
                customer { id }
            }
        }
    """
    context = fake_context(customer_lookup_should_fail=True)
    result = await schema.execute(query, context_value=context)

    # The order and its line items still resolved successfully.
    assert result.data["order"]["id"] == "42"
    assert result.data["order"]["lineItems"][0]["sku"] == "WIDGET-1"

    # But the customer resolver failed, and that's reflected precisely.
    assert result.errors is not None
    assert len(result.errors) == 1
    assert result.errors[0].path == ["order", "customer"]
```

This asserts the exact partial-failure contract the chapter describes: unrelated fields (`id`, `lineItems`) resolve correctly and are present in `data` even though a sibling field (`customer`) failed, and the failure is reported as a specific `errors` entry whose `path` pinpoints where in the query it happened — the shape client code has to branch on correctly in production.

## Quiz (self-check) — Answers

1. **False.** `FakeServicerContext` calls the servicer method directly in-process, bypassing the network entirely — it never serializes the request/response through Protobuf or sends anything over HTTP/2, so it cannot catch serialization bugs or interceptor-ordering issues. Only a real `grpc.aio.server()`-based integration test exercises the actual wire protocol.
2. **`httpx`.**
3. **`ghz`.**
4. **A representative mix of query shapes** — some cheap, some deliberately expensive — rather than a single repeated query, so the test exercises the same kind of complexity/N+1 exposure that real production traffic would (per Chapter 9), instead of only validating the happy path.

