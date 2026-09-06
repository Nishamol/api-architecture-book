# Chapter 9: GraphQL at Scale — Exercises

*Corresponds to: [09-graphql-at-scale.md](09-graphql-at-scale.md)*

## Concept questions

1. The chapter presents depth limiting and query complexity/cost analysis as two distinct defenses rather than one defense with two names. Give a concrete example of a query that a depth limit of 8 would allow through but that a complexity budget would reject, and vice versa — a query that complexity analysis would allow but a depth limit would reject.
2. Persisted queries are described as "effectively converting GraphQL's open query surface back into something closer to REST's fixed-endpoint model, for the subset of traffic where that trade-off makes sense." Explain what "subset of traffic" means here — for which category of client does this technique work, and for which category of client (named elsewhere in this book) would it fail outright?
3. Using the `@key(fields: "id")` and `@external` directives in the chapter's federation example, explain what problem these directives solve — specifically, how does the customer-service subgraph "extend" a type it doesn't own without redefining it from scratch?
4. The chapter says N+1 "is invisible in development... and catastrophic in production." Connect this directly to a specific number from the chapter's own N+1 example (20 customers) — why does a result-set size chosen for a dev/test fixture typically fail to reveal the problem at all, even qualitatively?

## Design question (interview-style)

You lead the platform team responsible for a federated GraphQL gateway sitting in front of three subgraphs: `orders-service`, `inventory-service`, and `customer-service`, each owned by a different team, composed the way the chapter's federation example shows (`Order @key(fields: "id")` in `orders-service`, extended by `customer-service`). Client teams report that a single screen — "customer order history with live inventory status per line item" — has a p99 latency of 4.2 seconds, up from 300ms before the last two subgraphs were federated in.

Using the chapter's explanation of federation's tradeoffs and its N+1 material, propose a diagnostic plan: what would you look at first, and why. Then propose fixes at two different levels — one specific to the gateway's query planning across subgraphs, and one specific to a single subgraph's own resolver layer — and explain why fixing only one of the two is likely to leave the problem partially unsolved.

## Coding exercise

The resolver below lives in `inventory-service` and backs a `LineItem.inventoryStatus` field that a federated query (like the one in the design question above) calls once per line item in an order:

```python
async def resolve_inventory_status(self) -> str:
    row = await db.fetch_inventory(sku=self.sku)  # runs once per line item
    return row["status"]
```

There is no `QueryDepthLimiter` or cost-analysis extension configured anywhere in `inventory-service`'s schema, and this resolver is unbatched.

Do two things. First, rewrite `resolve_inventory_status` using the DataLoader batching pattern from the chapter's `orders` example (`# Without batching: 1 + N queries` / `# With batching: 2 queries total, regardless of N`), including the batch-loading function. Second, add a `QueryDepthLimiter` extension to this subgraph's schema construction, using the chapter's own code example as your template, and state in one sentence why depth limiting alone would *not* have fixed the N+1 problem your batched resolver just solved.

## Quiz (self-check)

1. True or false: because each subgraph in a federated schema is a separate service, federation eliminates the N+1 problem — each service only has to worry about its own queries. *(Explain your answer.)*
2. What does query complexity/cost analysis require the schema owner to maintain on an ongoing basis that depth limiting does not?
3. In the persisted-queries model, what does the client send on each request instead of the full query document text?
4. Name the caching approach the chapter calls "the closest analog to REST's URL-based caching" on the server side.

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
