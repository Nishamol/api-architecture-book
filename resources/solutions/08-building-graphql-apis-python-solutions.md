# Chapter 8: Building GraphQL APIs in Python — Solutions

*Corresponds to: [part3-graphql/08a-building-graphql-apis-python-exercises.md](../../part3-graphql/08a-building-graphql-apis-python-exercises.md)*

## Concept questions — Model answers

1. Ariadne's schema-first approach is attractive precisely when the schema needs to be reviewable as its own artifact, independent of how the Python resolver code is organized — the chapter notes "some teams prefer" this "for keeping the schema as a reviewable artifact independent of Python code structure." A concrete situation: a platform team with a dedicated API-design or schema-governance function (common in larger organizations, or anywhere multiple backend teams contribute resolvers to one federated-looking domain) wants schema changes to go through review as `.graphql` diffs that non-Python stakeholders (frontend leads, API governance, even partner-facing API docs generators) can read without touching Python at all. Strawberry's type-hint-driven approach couples the schema definition to Python class structure, which is more ergonomic for a single team owning both the schema and the implementation, but less ergonomic when the schema itself is the primary reviewed contract and Python is "just" the implementation detail underneath it.

2. A DataLoader's internal cache is keyed by the argument passed to `.load()` — in the chapter's example, an order ID or customer ID — and it exists to deduplicate and batch calls made with the same key during a single execution tick. If the DataLoader instance is created once at module scope and shared across every request the process handles, its cache persists across requests rather than being scoped to one: once `.load("cus_9")` resolves for one user's request, that same instance's cache holds the result keyed by `"cus_9"`, and if a *different* request (a different user, potentially with different authorization) calls `.load("cus_9")` on the same shared instance before the cache is invalidated, it receives the cached value rather than a freshly authorized fetch — which is exactly how "one user's data" can be "return[ed] to another" under concurrent load, per the chapter's failure-modes list.

3. It's a paradigm shift because REST trains both developers and tooling to treat the HTTP status code as the primary success/failure signal — `200` means "trust the body," `4xx`/`5xx` means "don't." GraphQL breaks that assumption: a response can be `200 OK` and still contain an `errors` array describing a resolver that failed, alongside a `data` object where the failed field is `null` and everything else resolved normally. A client written with REST-era assumptions — checking only `response.status == 200` and then trusting the entire body — will render the response as if it were fully successful, silently treating a `null` (from a failed resolver, per the chapter's exact JSON example where `lineItems` is `null` due to a load failure) as if it were legitimately empty data, rather than surfacing the error to the user or retrying.

4. Strawberry needs the distinction because a plain dataclass-style attribute (`id`, `status`) is expected to already exist as a value on the object at the time the type is instantiated — the framework just reads it off the instance. `line_items`, by contrast, isn't data sitting on the `Order` instance; it has to be *fetched* (via `line_items_loader.load(self.id)`), which requires executing code — specifically async code that awaits a DataLoader — at resolution time. If `line_items` were declared as a plain attribute instead of an `@strawberry.field async def` method, there would be no mechanism to run that fetch: either the schema construction would fail because there's no value to assign to a bare `list[LineItem]` attribute at instantiation time, or (if you tried to precompute it eagerly before constructing `Order`) you'd lose the entire point of resolving fields independently and lazily, fetching line items even for queries that never asked for them.

## Design question — Model answer

**What goes wrong under concurrent requests:** Once `line_items_loader` is module-scoped and shared by every resolver invocation regardless of which request it's servicing, it stops behaving as a per-request batching cache and starts behaving as a long-lived, cross-request cache — exactly the chapter's first listed failure mode, "DataLoader created at module scope." Under concurrent traffic, two different requests (potentially from two different users, or the same user across two different orders) can interleave calls to `.load()` on the *same* instance; the loader's internal batching logic, designed to coalesce calls made within one request's execution tick, now has no way to distinguish "these calls belong to the same batch because they're the same request" from "these calls happen to be concurrent because the server is handling multiple requests at once." A cached result computed for one request's key can be handed back to a different request that happens to load the same key, which is a data leak across requests, not just a stale-cache correctness bug.

**Why it survives code review:** A reviewer reading only the resolver code sees `line_items_loader.load(self.id)` — syntactically identical to the correct, per-request version, since the call site doesn't change; only where the loader is *instantiated* changes. Unless the reviewer specifically checks `get_context` (or wherever the loader is constructed) and confirms it's built fresh inside the per-request context-construction path rather than imported from module scope, the bug is invisible from the resolver alone. This is exactly why the chapter frames context-scoped construction as a rule about *where* the DataLoader is created, not how it's called.

**Why local single-request tests pass:** A test process that only ever has one request in flight at a time never exercises the interleaving that causes the leak — with no concurrent second request, the module-level loader behaves identically to a per-request one, because there's effectively only ever "one request's worth" of calls hitting it between test runs (and each test run typically restarts the process or reimports modules, incidentally resetting the shared state). The bug is a property of concurrent access, so a test suite that never runs two requests against the same live server process simultaneously will never observe it — this is precisely why the chapter calls out that it "can return one user's data to another... under concurrent load," not under any load.

**The fix and what makes it durable:** Move `line_items_loader` construction back inside `Context.__init__` (or wherever `get_context` builds the per-request context), so a new instance is created on every request, as in the chapter's example. To prevent recurrence, the fix needs to be structural, not just corrected-by-hand: for example, banning module-level `DataLoader(...)` instantiation via a lint rule or code-review checklist item, or — more robustly — making the loader's constructor require a request-scoped argument (like the authenticated user or a request ID) that literally cannot be supplied at module import time, so that instantiating one outside `get_context` becomes a type error or an obvious code smell rather than a subtle behavioral bug that only manifests under concurrency.

## Coding exercise — Model answer

```python
from strawberry.dataloader import DataLoader
import strawberry

async def batch_load_orders_by_customer(customer_ids: list[str]) -> list[list[Order]]:
    rows = await db.fetch_orders(customer_id__in=customer_ids)  # one query, all customers
    by_customer = group_by_customer_id(rows)
    return [
        [Order(id=r["id"], status=r["status"]) for r in by_customer.get(cid, [])]
        for cid in customer_ids
    ]

@strawberry.type
class Customer:
    id: strawberry.ID
    name: str

    @strawberry.field
    async def orders(self, info: strawberry.Info) -> list[Order]:
        return await info.context.orders_loader.load(self.id)
```

The batch function follows the chapter's `batch_load_line_items` template exactly: it takes a list of keys (`customer_ids`, plural), issues a single query covering all of them (`fetch_orders(customer_id__in=customer_ids)`), groups the results by key, and returns a list of per-key result lists in the same order as the input keys — the shape `DataLoader` requires so it can map batched results back to individual `.load()` callers.

The loader instance must **not** be created at module scope — per the design question above, that reproduces the leak. It has to be created inside the per-request `Context`, alongside `line_items_loader` in the chapter's example:

```python
class Context(BaseContext):
    def __init__(self, user, db):
        self.user = user
        self.db = db
        self.line_items_loader = DataLoader(load_fn=batch_load_line_items)
        self.orders_loader = DataLoader(load_fn=batch_load_orders_by_customer)
```

and the resolver reads it off `info.context.orders_loader` (or `self.context.orders_loader`, depending on how the resolver accesses context), never off a module-level name — so a query like `{ customers { id orders { id status } } }` against a list of 20 customers now issues one batched `fetch_orders` call instead of 20 individual ones, and each request gets its own loader instance with its own isolated cache.

## Quiz (self-check) — Answers

1. **False.** `DataLoader.load()` schedules the key for a batched fetch and returns a value that resolves once the batch executes at the end of the current execution tick — it doesn't hit the database on that individual call. The whole point, per the chapter, is that multiple `.load()` calls made within one tick get coalesced into a single batched query via the `load_fn`.
2. `200 OK`.
3. Ariadne.
4. The authenticated `user` and a `db` connection (`self.user = user`, `self.db = db` in the `Context.__init__` example).
5. `@strawberry.input`. `@strawberry.type` can't be reused for both because GraphQL's SDL treats input and output types as distinct kinds with different rules — an input type can only be composed of other input types and scalars (it can't carry resolver methods like `Order.line_items`), so Strawberry needs a separate decorator to know which set of rules to validate a given class against and to generate the correct SDL (`input CreateOrderInput { ... }` versus `type Order { ... }`).
