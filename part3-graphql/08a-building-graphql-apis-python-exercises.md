# Chapter 8: Building GraphQL APIs in Python — Exercises

*Corresponds to: [08-building-graphql-apis-python.md](08-building-graphql-apis-python.md)*

## Concept questions

1. The chapter positions Strawberry, Graphene, and Ariadne as three different philosophies (type-hint-driven, class-based, schema-first) rather than three interchangeable options. Describe a concrete situation — team composition, existing codebase, or organizational preference — under which you'd choose Ariadne over Strawberry, even though the chapter says Strawberry "reflects its current adoption trajectory for greenfield services."
2. The chapter states DataLoaders "must be created fresh per request, not shared globally, or their batching cache will leak data across requests." Explain mechanically what a DataLoader's internal cache is keyed on and why a module-level (shared) instance causes one user's cached result to be returned to a different user.
3. "A GraphQL response can be partially successful... and the response still returns `200 OK` with an `errors` array alongside whatever `data` did resolve." Explain why this is "a genuine paradigm shift from REST's all-or-nothing status-code model," and describe specifically what a client written with REST-era assumptions would get wrong if it only checked the HTTP status code.
4. In the chapter's `Order` type, `id` and `status` are plain dataclass-style attributes, but `line_items` is written as an `async def` method decorated with `@strawberry.field`. Why does Strawberry require this distinction, and what would go wrong if `line_items` were also declared as a plain attribute?

## Design question (scenario-style)

Your team's `Context` class currently looks like the chapter's example — a fresh `DataLoader` instance created per request via `get_context`. A new engineer, trying to "simplify" the code and reduce object allocation, moves `line_items_loader = DataLoader(load_fn=batch_load_line_items)` out of `Context.__init__` and up to module scope, next to the other top-level definitions, and updates every resolver to reference the module-level loader directly instead of `self.context.line_items_loader`. Local tests pass — a single test process only ever has one in-flight request at a time.

Explain, referencing the chapter's failure modes, exactly what will go wrong once this ships to a server handling concurrent requests, why it will pass code review if the reviewer only reads the resolver code and not `get_context`, and why it will pass local single-request testing but not production traffic. Then describe the fix and what property it must have to actually prevent recurrence (hint: think about what makes this bug easy to reintroduce even after it's fixed once).

## Coding exercise

The resolver below powers a `Customer.orders` field mounted through a schema like the chapter's:

```python
import strawberry

@strawberry.type
class Order:
    id: strawberry.ID
    status: str

@strawberry.type
class Customer:
    id: strawberry.ID
    name: str

    @strawberry.field
    async def orders(self) -> list[Order]:
        rows = await db.fetch_orders(customer_id=self.id)
        return [Order(id=r["id"], status=r["status"]) for r in rows]
```

This resolver is deployed and a list query like `{ customers { id orders { id status } } }` is now the single most common query against this API. Using the chapter's DataLoader pattern (`batch_load_line_items` / `line_items_loader` example) as your template, rewrite `Customer.orders` to be batched. Write both the batch function and the `DataLoader` instantiation, and show where in the request lifecycle (per the chapter's "Context and per-request state" section) the loader instance needs to be created so it doesn't reproduce the failure mode from the design question above.

## Quiz (self-check)

1. True or false: calling `line_items_loader.load(order_id)` issues a database query immediately, at the moment it's called. *(Explain your answer.)*
2. What HTTP status code does a GraphQL server return for a response that includes both partial `data` and a non-empty `errors` array?
3. Which of the three frameworks discussed in this chapter is described as schema-first, where the SDL is authored by hand separately from the Python resolver code?
4. Name the two pieces of request-scoped state the chapter's `Context` example carries besides the DataLoader instance itself.
5. In the chapter's `create_order` mutation example, what Strawberry decorator marks `CreateOrderInput` as an input type, and why can't `@strawberry.type` be reused for both input and output types?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
