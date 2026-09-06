# Chapter 1: The API Architecture Landscape — Exercises

*Corresponds to: [01-introduction.md](01-introduction.md)*

## Concept questions

1. REST, GraphQL, and gRPC are described as optimizing for different failure modes rather than being interchangeable. For each protocol, name the specific failure mode it's optimized to avoid, and the cost it accepts in exchange.
2. A client fetching an order only needs `status`. Explain, in one sentence each, what happens on the wire for REST vs. GraphQL vs. gRPC in this scenario, and why the REST response is larger.
3. Why does gRPC's `.proto`-generated contract catch certain bugs at build time that REST and GraphQL can only catch at runtime? Give a concrete example of a change that would break each protocol differently.
4. "GraphQL solves over-fetching but moves complexity into the resolver graph." What does this mean concretely — what new class of problem does a GraphQL server have to defend against that a REST server doesn't?

## Design question (interview-style)

You're the lead engineer for a new order-management platform. Consumers are: (a) your company's iOS/Android apps, (b) 200+ third-party sellers integrating via API, (c) four internal microservices (inventory, billing, shipping, notifications) that need to talk to each other with sub-50ms latency.

Propose a protocol for each consumer group and justify it using the decision framework in this chapter. Then argue the strongest case *against* your own recommendation for one of the three — what would have to be true about this business for a different protocol to be the better choice?

## Coding exercise

Given this REST endpoint:

```python
@app.get("/orders/{order_id}")
async def get_order(order_id: str):
    order = await fetch_order(order_id)
    return order  # returns id, status, customer_id, line_items, shipping_address, billing_address
```

Write a GraphQL schema (`type Order { ... }` and a `Query` type) that would let a client request only `status` and `lineItems { sku quantity }` for the same underlying data, matching the example shown in this chapter. You don't need to implement resolvers — just the schema.

## Quiz (self-check)

1. True or false: GraphQL always uses less bandwidth than REST for the same underlying data. *(Explain your answer — this chapter gives you what you need.)*
2. Which protocol's contract is enforced by the compiler/build step rather than by convention or runtime validation?
3. Name one type of consumer for which gRPC would be a poor direct choice, and why.
4. In the three-protocol example in this chapter, which protocol's client code contains no manual URL or query-string construction at all?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
