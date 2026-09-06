# Chapter 18: Testing Strategies — Exercises

*Corresponds to: [18-testing-strategies.md](18-testing-strategies.md)*

## Concept questions

1. The chapter shows two different ways to test a gRPC service: an in-process test using `FakeServicerContext` that calls the servicer method directly, and a "genuine integration test" that spins up a real `grpc.aio.server()` on an ephemeral port. What does each style catch that the other one structurally cannot?
2. GraphQL responses can be partially successful. Explain why the chapter says "a meaningful chunk of GraphQL-specific test cases should specifically target partial-failure scenarios," and describe what such a test asserts that `test_order_query_returns_line_items` (which asserts `result.errors is None`) does not.
3. For the layered architecture referenced in the chapter — a REST gateway calling a gRPC backend queried through a GraphQL BFF — the chapter calls contract testing between layers "the most valuable and most commonly neglected test category." What specifically does this category verify that per-layer unit and integration tests, each passing on their own, do not?
4. Why does gRPC load testing need HTTP/2-aware tooling like `ghz` instead of a generic HTTP/1.1 load generator? Connect your answer to a concept from an earlier chapter that the text explicitly references.

## Design question (scenario-style)

Your team owns three services that together implement the chapter's referenced architecture: a REST gateway, a GraphQL BFF, and a gRPC backend. Each layer has its own test suite with high coverage, all green, all merged independently by three different sub-teams. A production incident occurs where the gateway's REST response is silently missing a field that the gRPC backend actually populated — the BFF's GraphQL resolver never mapped it through.

Using the chapter's discussion of contract testing and the test pyramid, explain why each team's own test suite passed despite this bug. Then design the specific test(s) you'd add — what layer(s) they sit at, what they assert, and how they'd have caught this before it reached production. Be specific about what tooling or pattern from the chapter you'd use.

## Coding exercise

The chapter's GraphQL test only covers the happy path:

```python
import pytest

@pytest.mark.asyncio
async def test_order_query_returns_line_items():
    query = """
        query {
            order(id: "42") {
                id
                lineItems { sku quantity }
            }
        }
    """
    result = await schema.execute(query, context_value=fake_context())
    assert result.errors is None
    assert result.data["order"]["lineItems"][0]["sku"] == "WIDGET-1"
```

Write a second test, `test_order_query_partial_failure_on_customer_lookup`, for a scenario where the order itself resolves successfully but the nested `customer` field's resolver raises (e.g., the customer service is down). Assert that `result.data["order"]["lineItems"]` still resolves correctly, that `result.errors` is not `None`, and that the error's `path` points at `["order", "customer"]`. You may assume a query that additionally requests `order { customer { id } }` and a `fake_context()` that can be configured to make the customer resolver raise.

## Quiz (self-check)

1. True or false: the `FakeServicerContext` in-process gRPC testing pattern exercises the real gRPC wire protocol, including serialization. *(Explain your answer.)*
2. What HTTP library underlies FastAPI's `TestClient`?
3. What is the name of the purpose-built gRPC load testing tool the chapter recommends over generic HTTP/1.1 load generators?
4. Per the chapter, what does a GraphQL load test need that a naive load test (a single repeated query) will miss?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
