# Chapter 1: The API Architecture Landscape — Solutions

*Corresponds to: [part1-foundations/01a-introduction-exercises.md](../../part1-foundations/01a-introduction-exercises.md)*

## Concept questions — Model answers

1. **REST** is optimized to avoid client/server coupling to a specific query shape — any HTTP client can call it, and intermediaries (CDNs, caches, proxies) can reason about requests using standard HTTP semantics. The cost it accepts is over-fetching/under-fetching, since the server — not the client — decides the response shape. **GraphQL** is optimized to avoid over-fetching and under-fetching — the client gets exactly the fields it asks for. The cost it accepts is server-side complexity: the resolver graph must defend itself against expensive, client-constructed queries at request time. **gRPC** is optimized to avoid runtime contract drift and serialization overhead — the `.proto` contract is enforced at compile time and the wire format is compact binary. The cost it accepts is human-readability and browser-native accessibility; you need generated stubs and HTTP/2 support to talk to it at all.

2. **REST**: the server always returns the full resource representation it's configured to return, so the client receives `status` plus every other field (`customer_id`, `line_items`, both addresses) whether it asked for them or not. **GraphQL**: the client's query document specifies `status` only, and the resolver returns just that field, so the response contains nothing else. **gRPC**: the client calls a method that returns a typed `Order` message; unless the API is explicitly designed with field masks or a narrower RPC, it also returns the full message — so by default it behaves like REST here, not like GraphQL. The REST response is larger than GraphQL's in this scenario specifically because REST's contract is "the server decides the shape," while GraphQL's contract lets the client constrain the shape per request.

3. gRPC's contract catches bugs at build time because the `.proto` file is the single source of truth that generates client *and* server code in the target language — a field rename, type change, or removed field breaks compilation on both sides before the code ever runs. REST and GraphQL don't generate code from a shared, versioned artifact in the same way: a REST server can rename a JSON field and ship it, and the break only surfaces when a client parses the response and gets `null` or a missing key at runtime. A GraphQL server can change a field's type or remove it, and unless the client has codegen wired against the schema (an optional add-on, not inherent to the protocol), the break similarly surfaces at runtime as a query error. Concrete example: renaming `line_items` to `items`. In gRPC, every generated stub referencing `.line_items` fails to compile. In REST, the JSON key silently disappears and the client's `order["line_items"]` lookup returns `None`/`undefined` at runtime. In GraphQL, a query still asking for `lineItems` fails at query-validation time against the schema — closer to gRPC's build-time safety, but only if the client validates its queries against the schema before deploying, which is a separate tooling step, not something the protocol forces.

4. It means a GraphQL server can no longer treat "what data gets fetched" as something decided at design time, since the client's query shape is decided at request time and can nest arbitrarily deep across resolvers. A REST server's designer chooses which joins/lookups a given endpoint performs, so the worst-case cost per endpoint is known in advance. A GraphQL server has to defend against a client (or a malicious caller) constructing a query that fans out into dozens or hundreds of resolver calls in one request — the N+1 problem introduced here and detailed in Chapter 9 — which means the server needs runtime defenses like query depth limiting, complexity/cost analysis, and batching (DataLoader) that a REST server simply doesn't need, because REST's fixed endpoints already bound the work per request.

5. The three models are **synchronous request/response**, **asynchronous**, and **streaming**. A 30-second bulk import is asynchronous: no client should hold an HTTP connection open for 30 seconds (connection timeouts, load-balancer idle limits, and client retries all make it fragile), and the browser that triggered it doesn't need the result on that same connection. The right shapes are `202 Accepted` plus a job resource the browser polls, or an event/webhook when done (Chapter 20). Forcing it into a synchronous request is the exact mistake the chapter names — "we modeled an asynchronous interaction synchronously" — and no REST-vs-GraphQL-vs-gRPC decision fixes it, because the problem is the communication model, not the protocol.

## Design question — Model answer

- **(a) iOS/Android apps → GraphQL.** A small number of client teams you can coordinate schema changes with, strong incentive to minimize payload size and round trips (mobile battery/latency), and UI screens that commonly need differently-shaped subsets of the same underlying data — the canonical GraphQL fit described in this chapter's framework.
- **(b) 200+ third-party sellers → REST.** Unknown, diverse clients you can't coordinate tooling with; REST's HTTP-native contract means any seller's HTTP client can integrate without adopting Protobuf or GraphQL tooling, and standard HTTP caching/status codes give you a well-understood integration surface at that scale.
- **(c) Four internal microservices → gRPC.** You control both ends, sub-50ms latency is a hard requirement, and compile-time contract safety across four services matters more as the number of internal call paths grows — exactly the internal service-to-service case the chapter calls out.

**Arguing against my own recommendation — the mobile GraphQL choice:** if the two client teams (iOS/Android) rarely diverge in what data they need — say, both apps are thin wrappers around the same handful of fixed screens — then GraphQL's flexibility is solving a problem you don't have, and you're paying for it anyway: an extra resolver layer, N+1 defenses, query cost analysis, and a GraphQL gateway to operate. In that world, REST (or even a small set of purpose-built BFF endpoints) would be simpler to build, cache, and debug, with none of GraphQL's operational overhead. This would be the better choice if the business is a single-purpose app with a stable, small set of screens rather than a platform with fast-changing or highly variable client UIs.

## Coding exercise — Model answer

```graphql
type Order {
  id: ID!
  status: String!
  customerId: ID!
  lineItems: [LineItem!]!
  shippingAddress: Address!
  billingAddress: Address!
}

type LineItem {
  sku: String!
  quantity: Int!
}

type Address {
  street: String!
  city: String!
  postalCode: String!
  country: String!
}

type Query {
  order(id: ID!): Order
}
```

With this schema, the query shown in the chapter —

```graphql
query {
  order(id: "42") {
    status
    lineItems { sku quantity }
  }
}
```

— resolves against `Query.order`, and the client receives only `status` and `lineItems { sku quantity }`, matching the chapter's example. `customerId` and both addresses are defined on the type so other clients/queries can request them, but this particular query never touches them.

## Quiz (self-check) — Answers

1. **False.** For a single request fetching a narrow field set, GraphQL is smaller. But GraphQL has query overhead (the query document itself is sent on every request) and, per Chapter 9, poorly bounded queries can request far more than an equivalent REST call ever would. Bandwidth comparison depends on access pattern, not the protocol alone.
2. **gRPC** — the `.proto` file generates typed stubs in both client and server; a mismatched field type fails to compile rather than failing at runtime.
3. **Third-party/external integrators** — gRPC requires HTTP/2 and Protobuf tooling that's a poor fit for browser clients and partners you don't control the tooling stack for; it's typically exposed to them via REST/JSON transcoding instead (Chapter 14).
4. **gRPC** — `stub.GetOrder(GetOrderRequest(order_id="42"))` is a typed method call, not a hand-built URL or query string.
5. Any three of: interaction model, client diversity, contract strength, latency, cacheability, payload/query characteristics, ownership (do you control both ends), ecosystem/operational maturity, streaming needs.
