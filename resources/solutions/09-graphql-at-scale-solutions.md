# Chapter 9: GraphQL at Scale — Solutions

*Corresponds to: [part3-graphql/09a-graphql-at-scale-exercises.md](../../part3-graphql/09a-graphql-at-scale-exercises.md)*

## Concept questions — Model answers

1. A query that stays within 8 levels of nesting but requests a broad, expensive field at every level — for example, `customers { orders { lineItems { sku } } }` where `customers` alone returns 10,000 rows and each has dozens of orders — is only 3 levels deep, well under a depth limit of 8, but its total resolved-field count (and therefore its cost) could be enormous; depth limiting alone would let it through. Conversely, a query that recurses through the `Order`/`Customer` cycle exactly 9 levels deep but touches only `id` at each level (`customer { orders { customer { orders { ... { id } } } } }`) might have a modest total field count that a complexity budget would accept, but a depth limit of 8 would reject it outright purely on nesting depth, regardless of how cheap each individual level is.
2. "Subset of traffic" means client populations you control end-to-end — the chapter specifies "client apps you control (mobile, first-party web)" — because persisted queries require the server to have a pre-approved, hashed set of query documents, which only works if you (or a coordinated client team) can ship query documents ahead of time and keep the server's approved set in sync with what the client actually sends. This fails outright for the external, third-party integrator population described in Chapter 1's design scenario (the "200+ third-party sellers") and analogous partner/public API consumers elsewhere in the book — you have no ability to pre-register their query shapes, and rejecting any hash you don't recognize would simply break their integration rather than protect your server.
3. `@key(fields: "id")` declares that `Order.id` is the field federation uses to identify and stitch together the *same* logical `Order` entity across subgraphs — it's how the gateway knows that the `Order` returned by `orders-service` and the `Order` referenced by `customer-service` are one object, not two. `@external` on `customer-service`'s copy of `id` marks that field as owned by a different subgraph (`orders-service`) — `customer-service` is declaring "I need this field to resolve my own additions, but I am not the authority on it, don't ask me to resolve it." This is how `customer-service` extends `Order` with a new field (`customer: Customer!`) without redefining `id`, `status`, or any of the fields `orders-service` already owns — it attaches only the field it's actually adding, referencing the shared key to tell the gateway how to join its contribution back onto the object the other subgraph produced.
4. The chapter's own example uses 20 customers to illustrate the *mechanism* (1 query + 20 additional queries = 21 total) — at that scale, 21 sequential queries against a local or lightly-loaded dev database each complete in low single-digit milliseconds, so the whole request still feels instantaneous to a developer testing it by hand; nothing about the response *time* signals a problem at 20 rows. The chapter is explicit that this pattern "scales linearly with result set size," so the qualitative failure only becomes visible once the multiplier N is large enough that N sequential round-trips (each carrying real network and connection-pool overhead in production, not local-loopback overhead) add up to something a human or an SLO notices — the chapter's own example of "a customer list page might render 200 rows" is an order of magnitude past a typical dev fixture, and production result sets can be another order of magnitude past that.

## Design question — Model answer

**Diagnostic plan — what to look at first, and why:** Start with the gateway's query plan for this specific screen's query, not individual subgraph logs — federation means the client sends one query, but the gateway decomposes it into a plan of sub-requests across `orders-service`, `inventory-service`, and `customer-service`, and the chapter is explicit that "a naive federated resolver can turn one client query into a request storm across your entire service mesh." The query plan will show whether the fan-out is proportional to the number of line items being displayed (a strong N+1 signal at the federation layer) or whether it's a small, fixed number of cross-service calls with one of them individually slow (a different problem — a single slow subgraph, not fan-out). Given the timing (latency regressed specifically after federating in two more subgraphs) and the screen's shape (order history × live inventory status *per line item*), the leading hypothesis should be: `inventoryStatus` is resolved once per line item, and that per-item resolution is unbatched both within `inventory-service` and, at the gateway level, across the federated call boundary.

**Fix 1 — gateway/query-planning level:** Ensure the gateway batches its calls into each subgraph rather than issuing one sub-request per entity per field. Apollo-style gateways do this via the `_entities` batch resolution mechanism when subgraphs implement `@key`-based entity resolution correctly — the gateway should be sending inventory-service a single batched `_entities` call carrying all the SKUs needed for the whole order-history screen, not one call per line item. If the query plan shows N individual round-trips into `inventory-service` instead of one batched round-trip, that's a gateway/query-planning-level bug (or a subgraph implementation gap in how it handles entity resolution requests) independent of anything happening inside `inventory-service`'s own resolvers.

**Fix 2 — subgraph resolver level:** Inside `inventory-service`, the `LineItem.inventoryStatus` resolver needs to go through a DataLoader keyed on SKU, exactly per the chapter's N+1 remedy — even if the gateway hands `inventory-service` a batched `_entities` request, if the *subgraph's own resolver* for `inventoryStatus` fetches inventory one SKU at a time inside that batch's resolution, you've just moved the N+1 down one level rather than eliminating it.

**Why fixing only one leaves the problem partially unsolved:** These are two independent multiplicative layers. If you fix only the gateway's batching but `inventory-service`'s internal resolver is still unbatched, the gateway now sends one clean batched request into `inventory-service`, but that service's own resolver still turns it into N internal database calls — the round-trip count across the network drops, but the database load and per-request latency inside `inventory-service` do not. If you fix only the subgraph's internal DataLoader but the gateway is still issuing one federated sub-request per entity instead of one batched call, you've made each individual sub-request cheap, but you still pay N times the network/serialization/scheduling overhead of federation itself. The chapter's warning that federation needs "its own N+1 awareness at the cross-service level" is precisely about this: cross-service fan-out and intra-service N+1 are two separate mechanisms that compound, and a fix at only one layer removes only one multiplier, not both.

## Coding exercise — Model answer

```python
from strawberry.dataloader import DataLoader

async def batch_load_inventory_status(skus: list[str]) -> list[str]:
    rows = await db.fetch_inventory(sku__in=skus)  # one query, all SKUs
    by_sku = {row["sku"]: row["status"] for row in rows}
    return [by_sku.get(sku, "UNKNOWN") for sku in skus]

inventory_status_loader = DataLoader(load_fn=batch_load_inventory_status)

# Without batching: 1 + N queries
async def resolve_inventory_status(self) -> str:
    row = await db.fetch_inventory(sku=self.sku)  # runs once per line item
    return row["status"]

# With batching: 2 queries total, regardless of N
async def resolve_inventory_status(self) -> str:
    return await inventory_status_loader.load(self.sku)
```

(Per Chapter 8's guidance, `inventory_status_loader` must actually be constructed per-request via the subgraph's context getter, not at module scope as shown for brevity here — otherwise this reintroduces the cross-request cache-leak failure mode from Chapter 8, not the N+1 problem this exercise is fixing.)

```python
from strawberry.extensions import QueryDepthLimiter

schema = strawberry.Schema(
    query=Query,
    extensions=[QueryDepthLimiter(max_depth=8)],
)
```

Depth limiting alone would not have fixed this N+1: the query calling `resolve_inventory_status` once per line item in an order is not necessarily deep — `order { lineItems { inventoryStatus } }` is only 3 levels of nesting, comfortably under a `max_depth=8` limit — but it is *wide*, fanning out across however many line items the order contains. Depth limiting bounds nesting, not breadth, so it does nothing to stop a shallow-but-wide query from triggering one unbatched database call per line item; only batching (via DataLoader) or a width-aware complexity/cost budget addresses that.

## Quiz (self-check) — Answers

1. **False.** Federation splits *ownership* of the schema across services, but it does not automatically batch anything — a federated query can still fan out into per-entity resolver calls both within a single subgraph and across the gateway's calls into multiple subgraphs. The chapter is explicit that "a naive federated resolver can turn one client query into a request storm across your entire service mesh," and lists "federation without cross-service N+1 awareness" as its own distinct failure mode.
2. Per-field cost annotations, kept up to date as the schema grows — the chapter calls this "a real ongoing maintenance cost, not a set-and-forget defense."
3. A hash identifying the pre-approved query document, rather than the query's full text.
4. Persisted queries combined with response caching keyed on the query hash plus variables.
