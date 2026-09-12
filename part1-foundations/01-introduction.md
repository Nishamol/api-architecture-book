# Chapter 1: The API Architecture Landscape

## Why protocol choice is an architecture decision, not a style preference

Junior teams pick an API style because it's what the framework tutorial used. Senior and lead engineers pick one because they've mapped it against access patterns, client diversity, latency budgets, and the organizational boundaries the API has to survive. REST, GraphQL, and gRPC are not competing implementations of the same idea — they optimize for different failure modes, and choosing wrong shows up eighteen months later as a rewrite.

## Three protocols, three contracts

**REST** is a resource-oriented style built on HTTP semantics. A REST API exposes nouns (`/orders/42`) and lets HTTP verbs (`GET`, `POST`, `PATCH`, `DELETE`) express intent. Its contract is loose *at the protocol level* — HTTP itself enforces media types and status codes but not payload shape. Teams routinely impose a strong contract on top with OpenAPI, JSON Schema, and consumer-driven contract tests; the point is that this is a discipline the organization adds, not something the protocol checks for you the way a GraphQL schema or a `.proto` file does. That looseness is REST's biggest strength (any HTTP client can talk to it, caching works for free) and, left undisciplined, its biggest weakness (over-fetching, under-fetching, and undocumented payload drift).

**GraphQL** is a query-oriented style built on a single endpoint and a strongly typed schema. The client specifies the exact shape of the data it wants, and the server resolves it field by field. This solves REST's over-fetching problem at the cost of moving complexity into the resolver graph — a single query can now trigger dozens of downstream calls, and the server has to defend itself against expensive queries at request time rather than at design time.

**gRPC** is a contract-first RPC framework built on HTTP/2 and Protocol Buffers. The client calls what looks like a local function; the wire format is a compact binary encoding, and the contract is a `.proto` file that generates client and server stubs in whatever language you need. gRPC trades human-readability and browser-nativeness for performance, strong typing, and native support for streaming — this is why it dominates service-to-service communication inside a datacenter but rarely faces an external mobile client directly.

| Aspect | REST | GraphQL | gRPC |
|---|---|---|---|
| Contract | Loose at the protocol level — HTTP verbs and media types; strong payload contracts (OpenAPI/JSON Schema) are a convention layered on top | Strongly typed schema; client specifies the exact shape it wants | Contract-first `.proto` file; generates typed client/server stubs |
| Biggest strength | Any HTTP client can talk to it; caching works for free | Solves over-fetching and under-fetching | Compact binary wire format, strong typing, native streaming |
| Biggest weakness | Over-fetching, under-fetching, undocumented payload drift | Complexity moves into the resolver graph; server must defend against expensive queries | Not browser-native; trades human-readability for performance |

## Same operation, three shapes

Fetching order `42` and its line items looks different in each protocol. Seeing the same request side by side is the fastest way to internalize what each one actually optimizes for.

**REST**

```
GET /v1/orders/42 HTTP/1.1
Host: api.example.com
Authorization: Bearer <token>
```

```json
{
  "id": "42",
  "status": "SHIPPED",
  "customer_id": "cus_9",
  "line_items": [
    {"sku": "WIDGET-1", "quantity": 2},
    {"sku": "GADGET-7", "quantity": 1}
  ],
  "shipping_address": { "...": "..." },
  "billing_address": { "...": "..." }
}
```

The server decides the shape. If the client only needed `status`, it still paid for `shipping_address` and `billing_address` — classic over-fetching.

**GraphQL**

```graphql
query {
  order(id: "42") {
    status
    lineItems { sku quantity }
  }
}
```

```json
{
  "data": {
    "order": {
      "status": "SHIPPED",
      "lineItems": [
        {"sku": "WIDGET-1", "quantity": 2},
        {"sku": "GADGET-7", "quantity": 1}
      ]
    }
  }
}
```

The client asked for exactly `status` and `lineItems`. No addresses, nothing unused — but the server now had to resolve `lineItems` as an independent field, which is where Chapter 9's N+1 problem originates.

**gRPC**

```protobuf
rpc GetOrder(GetOrderRequest) returns (Order);
```

```python
response = await stub.GetOrder(GetOrderRequest(order_id="42"), timeout=2.0)
print(response.status, response.line_items)
```

No JSON parsing, no URL construction — the client calls a typed method and gets a typed object back. The contract (`Order`, `GetOrderRequest`) is generated from the `.proto` file, so a field typo is a build error, not a runtime surprise.

## First, a prior question: which communication model?

REST, GraphQL, and gRPC are all **synchronous request/response**: a client sends a request, waits, and gets a response on the same connection. That's the right model for most API interactions, and it's what Parts II through IV cover in depth. But it isn't the only model, and choosing it by default is itself an architecture decision worth making consciously.

- **Synchronous request/response** — the caller needs the answer now and will wait for it (fetch an order, validate a form, run a search). REST / GraphQL / gRPC.
- **Asynchronous** — the work takes longer than a caller should hold a connection for, or no single caller is waiting for "the answer" at all (a bulk import, a payment settlement, an event other systems react to). Webhooks, message queues, or an event stream — covered in Chapter 20.
- **Streaming** — a continuous flow of messages in one or both directions rather than a single response (live updates, telemetry ingestion, large result sets). gRPC streaming, Server-Sent Events, WebSockets, or a subscribed event stream.

```mermaid
flowchart TB
    Q["What communication model<br/>does this interaction need?"]
    Q --> RR["Request / response"]
    Q --> AS["Asynchronous"]
    Q --> ST["Streaming"]

    RR --> REST["REST"]
    RR --> GraphQL["GraphQL"]
    RR --> gRPC["gRPC"]

    AS --> WH["Webhook"]
    AS --> QU["Queue"]
    AS --> EV["Event / pub-sub"]

    ST --> GS["gRPC streaming"]
    ST --> SSE["SSE"]
    ST --> WS["WebSocket"]
    ST --> ES["Event streaming"]

    classDef neutral fill:#f3f4f6,stroke:#9ca3af,color:#374151
    classDef client fill:#dbeafe,stroke:#2563eb,color:#1e3a8a,stroke-width:2px
    classDef success fill:#dcfce7,stroke:#16a34a,color:#14532d

    class Q neutral
    class RR,AS,ST client
    class REST,GraphQL,gRPC,WH,QU,EV,GS,SSE,WS,ES success
```

A large share of "we chose the wrong API style" pain is really "we modeled an asynchronous interaction synchronously" — a 30-second operation forced into a blocking request. Settle the communication model first; the rest of this framework assumes you've landed on synchronous request/response.

## A decision framework, previewed

The full framework arrives in Chapter 21, once you've seen the mechanics of all three. Within synchronous request/response, protocol choice is the *result* of a set of constraints, not a lookup by client type:

- **Interaction model** — simple resource fetches, or client-driven queries over a variable graph of data?
- **Client diversity** — a bounded set of client teams you can coordinate with, or unknown third parties you can't?
- **Contract strength** — is a loose, self-describing contract acceptable, or do you need compile-time enforcement?
- **Latency** — is this on a p99-sensitive internal path, or a human-facing request where tens of milliseconds don't matter?
- **Cacheability** — does URL-keyed HTTP caching at the CDN/proxy layer materially help this workload?
- **Payload and query characteristics** — small and uniform, or large with per-client field selection needs?
- **Ownership** — do you control both ends, or only one?
- **Ecosystem and operational maturity** — is your team already fluent in Protobuf toolchains and L7 load balancing, or is HTTP/JSON the path of least resistance?
- **Streaming** — does any part of this need server-push or bidirectional streams?

As a starting heuristic, those constraints tend to cluster: unknown/diverse public consumers that benefit from caching lean **REST**; a few coordinated first-party UI teams that need to minimize over-fetching lean **GraphQL**; internal service-to-service calls where you own both ends and care about p99 latency and compile-time safety lean **gRPC**. But the clustering is a consequence of the constraints, not a rule — an internal path can rationally stay REST for debuggability and HTTP-centric infra, and a public API can rationally be gRPC-with-transcoding. Chapter 21 works through the constraint set in full.

```mermaid
flowchart TB
    C["Synchronous request/response —<br/>weigh the constraints"]
    C --> F1["Diverse/unknown consumers<br/>+ cacheability matters"]
    C --> F2["Few coordinated client teams<br/>+ variable per-screen data needs"]
    C --> F3["You own both ends<br/>+ p99 latency + compile-time contract"]

    F1 -->|typical fit| REST["REST"]
    F2 -->|typical fit| GraphQL["GraphQL"]
    F3 -->|typical fit| gRPC["gRPC"]

    C -.-> Note["A single constraint can override the cluster:<br/>internal REST for debuggability, public gRPC via transcoding,<br/>GraphQL BFF in front of gRPC services"]

    classDef neutral fill:#f3f4f6,stroke:#9ca3af,color:#374151
    classDef client fill:#dbeafe,stroke:#2563eb,color:#1e3a8a,stroke-width:2px
    classDef success fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef warn fill:#fef3c7,stroke:#d97706,color:#78350f

    class C neutral
    class F1,F2,F3 client
    class REST,GraphQL,gRPC success
    class Note warn
```

Most real systems don't pick one. A production platform commonly runs gRPC internally between services, exposes a GraphQL gateway to first-party client apps, maintains a REST API for third-party integrators, and publishes events to a broker that drives internal consumers and outbound webhooks. Chapter 22 walks through exactly this architecture end to end.

```mermaid
flowchart LR
    Mobile["Mobile / web apps<br/>(first-party clients)"] --> GW["GraphQL gateway"]
    ThirdParty["Third-party integrators<br/>+ webhook consumers"] --> REST["REST API"]

    GW --> SvcA["Internal service A"]
    REST --> SvcA
    SvcA <-->|gRPC| SvcB["Internal service B"]

    classDef client fill:#dbeafe,stroke:#2563eb,color:#1e3a8a,stroke-width:2px
    classDef neutral fill:#f3f4f6,stroke:#9ca3af,color:#374151
    classDef success fill:#dcfce7,stroke:#16a34a,color:#14532d

    class Mobile,ThirdParty client
    class GW,REST neutral
    class SvcA,SvcB success
```

## What "senior-level" means for this book

This book does not re-explain what JSON is or how to define a Flask route. Each protocol chapter assumes you can read Python comfortably and have shipped at least one production API. The value here is in the second-order concerns: what breaks under load, what breaks under schema change, what breaks when a client team you don't control ships a mobile release you can't force-update.

## What's next

Chapter 2 covers the HTTP and networking layer that REST, GraphQL, and (transport-wise) gRPC all sit on top of.

---

## Exercises

Exercises for this chapter live in [01a-introduction-exercises.md](01a-introduction-exercises.md). Solutions are available separately in [resources/solutions/](../resources/solutions/).
