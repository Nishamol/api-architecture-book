# Chapter 7: GraphQL Schema Design and Execution Model — Solutions

*Corresponds to: [part3-graphql/07a-graphql-fundamentals-exercises.md](../../part3-graphql/07a-graphql-fundamentals-exercises.md)*

## Concept questions — Model answers

1. Bidirectional relationships are normal in GraphQL because the schema is meant to model the domain as a graph, not as a set of one-way lookups the way a REST resource tree implicitly is — if `Order` can point to its `Customer`, it's natural for `Customer` to point back to its `Order`s, and clients legitimately need to traverse in both directions depending on which object they start from. The danger is structural: because each relationship field is resolved independently (per the chapter's "Resolvers" section), a query that lists customers and asks for each one's `orders` will, without batching, issue one database call per customer — the N+1 pattern. The bidirectionality is what makes this reachable from *either* side of the relationship and, combined with the cyclical shape (`Order → Customer → Order → ...`), is also what makes unbounded-depth queries possible, since a client can alternate directions indefinitely without ever leaving the schema's declared types.

2. In REST, the server's endpoint already decides what data it fetches and serializes; adding a field to the response usually just means exposing a column or attribute that's already sitting in memory as part of the object the endpoint fetched anyway. In GraphQL, every field is a distinct point of execution — the schema *is* the query interface, so a new field is a new thing a client can independently request, and the runtime must have a resolver capable of producing it regardless of what else was requested in the same query. The specific risk with a *relationship* field, as opposed to a scalar one, is that a scalar field is usually just an attribute read off an object you already have in hand, while a relationship field implies fetching a *different* object or collection — and if that fetch isn't batched, adding one relationship field to a type can silently turn every list query that includes that type into a query whose cost scales with the length of the list, exactly the "query-cost discipline a DBA would apply" the chapter invokes.

3. It implies you cannot infer purity from the schema alone — a field named `order` under `Query` gives you no compiler- or runtime-enforced guarantee that resolving it won't, say, log an audit event, touch a cache, or (in a poorly designed schema) mutate something. You have to read the resolver implementation, not just the type name, to know whether a "query" is actually safe to run repeatedly or in parallel. This contrasts with Mutations, which the chapter says "by convention, return the post-mutation state of the affected object" — a weaker but still useful convention, because at least the *contract of intent* (this is where writes happen) is signaled by which root type the field lives under, even though, like the Query purity assumption, it's still convention rather than something the type system enforces.

4. The concrete benefit is developer experience: introspection (`__schema`, `__type`) is what powers tools like GraphiQL and Apollo Studio's schema explorer, letting developers browse and autocomplete against the live schema without separate documentation. The concrete risk is that the same mechanism, left enabled on a public-facing production endpoint, "hands an attacker a complete map of your data model, including fields you never intended to expose broadly" — meaning reconnaissance that would otherwise require guessing endpoint shapes or reading leaked documentation becomes a single query away, with zero authentication required in a naive setup.

5. The rule: use an interface when the possible types genuinely share an overlapping field set the client can query without knowing the concrete type — `Node`'s `id: ID!` is present on both `Order` and `Customer`, so a client can ask for `id` on a `Node`-typed field regardless of which concrete type comes back. Use a union when the possible types have no shared fields at all — `SearchResult = Order | Customer` doesn't imply `Order` and `Customer` have anything in common, so there's nothing generic to ask for. Mechanically, this means a client querying an interface field can select the shared fields directly, while a client querying a union field must use inline fragments (`... on Order { status }`, `... on Customer { name }`) to pick fields specific to whichever concrete type is actually returned — there's no field a union guarantees exists on every member, so the client has to branch by type in the query itself.

## Design question — Model answer

**(1) What a partner can learn via introspection alone:** With introspection enabled, a partner can send a `__schema` query and receive the complete type graph — every type (`Order`, `Customer`, `LineItem`, `Money`, `OrderStatus`), every field on each type with its exact name and type signature, and both root operations (`order`, `customer` under `Query`). This includes the full bidirectional relationship shape (`Order.customer` and `Customer.orders`) even though nobody documented it for them, and it happens without the partner ever having to send a single "real" query against your data — it's pure schema reconnaissance, no authorization check beyond whatever protects the `/graphql` endpoint itself.

**(2) What happens mechanically with unbounded nesting:** Nothing in the schema as given stops the cyclic traversal `customer { orders { customer { orders { ... } } } }`, because `Order.customer` and `Customer.orders` are both declared as non-null, always-resolvable fields with no depth ceiling anywhere in the type definitions. Each level of nesting is a legal GraphQL selection against a legal field, so the server will attempt to resolve it: with no batching, this is compounding N+1 (each `orders` level triggers per-customer queries feeding the next level's per-order lookups), and with no depth limit (Chapter 9's `QueryDepthLimiter`) or complexity/cost budget, there is nothing in this schema or server configuration that rejects the query before execution starts. In the worst case this is a single client request that fans out into an exponentially growing number of resolver calls and database round-trips.

**(3) Which failure modes are live, and why introspection compounds them:** All three of the chapter's listed failure modes are live here. **Unbounded relationship depth** is live because the `Order`/`Customer` cycle exists and nothing limits traversal. **Resolver-level N+1** is live because the schema gives no indication any of `Order.customer`, `Customer.orders`, or `Order.lineItems` is batched — the chapter's default assumption is that they aren't unless you've deliberately added DataLoader (Chapter 8). **Introspection left on in production** is live by the scenario's own premise. Introspection isn't merely a fourth independent risk sitting next to the other two — it's a force multiplier for them: an attacker doesn't have to guess that `Order` and `Customer` cycle back into each other to build a depth-based denial-of-service query, or guess which relationship fields are unbatched and therefore expensive; introspection hands them the exact field names and exact cyclic shape needed to construct the worst-case query on the first try, turning what would otherwise require blind trial-and-error into a schema lookup.

## Coding exercise — Model answer

```graphql
type Order {
  id: ID!
  status: OrderStatus!
  customer: Customer!
  lineItems: [LineItem!]!
  total: Money!
  reviews: [Review!]!
}

type Review {
  id: ID!
  rating: Int!
  body: String!
  author: Customer!
}
```

Field-by-field resolution:

- `Order.id`, `Order.status`, `Order.total` — resolved automatically; these are scalar-shaped attributes the framework reads directly off the fetched `Order` object.
- `Order.customer` — requires an explicit resolver; it's a relationship field that must fetch a different object (the owning `Customer`).
- `Order.lineItems` — requires an explicit resolver; same reasoning, a relationship to a collection.
- `Order.reviews` — requires an explicit resolver; this is a new relationship field, and per the chapter's warning, adding it means writing a resolver that "can silently introduce a new database round-trip per item in every list that includes it" — i.e., every place `Order` is queried in a list context now risks a new N+1 unless this resolver is batched.
- `Customer.id`, `Customer.name` — resolved automatically; scalar attributes.
- `Customer.orders` — requires an explicit resolver; relationship to a collection (already discussed in the chapter).
- `Review.id`, `Review.rating`, `Review.body` — resolved automatically; scalar attributes.
- `Review.author` — requires an explicit resolver. Name-checked against the chapter: this is precisely a "relationship field (`Order.customer`)"-shaped case — it points at a *different* object (`Customer`), not an attribute already sitting on the `Review` record, so the framework cannot "read the attribute by name" the way it can for `rating` or `body`.

**New cyclic path:** adding `Review.author: Customer!` alongside the existing `Customer.orders: [Order!]!` and the new `Order.reviews: [Review!]!` opens `order → reviews → author → orders → reviews → author → ...` — a second cycle through the graph, structurally identical in kind to the `customer → orders → customer → orders` cycle already named in the chapter, just one hop longer per iteration. This falls under the **unbounded relationship depth** failure mode: nothing in the SDL as written stops a client from traversing this new cycle indefinitely, and because `reviews` and `orders` are both relationship fields on list-context types, an unbatched traversal through this cycle is also a fresh surface for **resolver-level N+1**.

## Quiz (self-check) — Answers

1. **False, with a caveat.** As written and used idiomatically, a scalar field like `status` is read directly off the object the framework already has in hand and needs no resolver function. The "never" breaks down only if you deliberately write a custom resolver for a scalar field anyway (e.g., to reformat or compute a derived value) — nothing in GraphQL prevents that, it's just unnecessary for a plain pass-through attribute like the ones in this schema.
2. `__schema` (introspection query).
3. Unbounded relationship depth.
4. It returns the post-mutation `Order` object (its `id` and `status`, as queried), following the convention that mutations "return the post-mutation state of the affected object so the client doesn't need a follow-up query."
5. **False.** A custom scalar needs its own serialization rule — for `DateTime`, that's typically an ISO-8601 string on the wire mapped to and from a native `datetime` object in resolver code — and that mapping is supplied by the server framework, not the GraphQL spec itself. The five built-in scalars (`ID`, `String`, `Int`, `Float`, `Boolean`) don't cover this case at all, which is exactly why a custom scalar declaration exists in the first place.
