# Chapter 14: API Gateways and Service Mesh — Solutions

*Corresponds to: [part5-cross-cutting/14a-api-gateway-service-mesh-exercises.md](../../part5-cross-cutting/14a-api-gateway-service-mesh-exercises.md)*

## Concept questions — Model answers

1. **Mesh solving an edge problem**: a team stands up Istio expecting it to handle authentication and rate limiting for external, third-party API consumers hitting their public REST API — but a mesh is built for east-west (service-to-service) traffic and doesn't natively address north-south concerns like terminating external client auth or enforcing per-external-client quotas the way a gateway does. The symptom here is "missing capabilities" — the mesh doesn't give them what an edge gateway would. **Gateway solving an internal routing problem**: a team routes all internal service-to-service calls through a central API gateway instead of using direct service discovery, hoping to get retries and mTLS between internal services this way — but gateways aren't designed to sit on every internal call path with mesh-level granularity (per-hop mTLS, sidecar-level retry policy). The symptom here is "unnecessary operational overhead" — every internal call now pays gateway hop latency and the gateway becomes a scaling bottleneck for traffic it was never designed to carry at that volume.

2. Transcoding lets an external REST/JSON caller hit `GET /v1/orders/{order_id}`, and Envoy's gRPC-JSON transcoding filter (driven by the `google.api.http` annotation in the `.proto`) translates that into the actual `GetOrder` gRPC call against the internal service, and translates the typed `Order` response back into JSON on the way out. The internal service itself only ever implements and exposes the gRPC method — it never sees or handles REST directly. Without this pattern, the team would have to build and maintain a second implementation of the same business logic: either a parallel REST API that independently re-implements `GetOrder`'s logic (now two places that can drift and disagree), or a custom REST-to-gRPC translation layer they write and maintain by hand instead of getting it from the annotation-driven proxy.

3. The chapter's distinction: a BFF is "typically owned by the client team it serves and treated as disposable, client-specific glue," with "few or no resolvers backed directly by a database" — its resolvers exist purely to call out to internal gRPC/REST services and compose their results into the shape one specific client needs. Federation (Chapter 9), by contrast, is a model where "multiple teams each own a piece of the graph" — a shared platform artifact, not disposable glue owned by one client team. The failure mode when a BFF drifts toward acting like federation (or like a real service) is named directly in the chapter's failure modes: "BFF resolvers becoming a second source of business logic" — the BFF starts embedding actual business rules instead of pure composition, which creates two places (the BFF and the underlying service) that can disagree about behavior, defeating the entire point of treating it as thin, disposable glue.

4. Two of the three: **mTLS (Chapter 13)** — without a mesh, each service in each language would need its own certificate issuance, rotation, and verification logic; the mesh's sidecar centralizes this so "adopting a mesh is common specifically to get this for free rather than building custom cert management per service," as Chapter 13 puts it. **Circuit breaking (Chapter 16)** — without a mesh, each service's client code, in whatever language it's written in, would need its own circuit-breaker implementation guarding every outbound call; the mesh's sidecar applies a circuit-breaking policy at the proxy layer uniformly, so the logic isn't reimplemented per service per language. **Distributed tracing propagation (Chapter 17)** — without a mesh, every service would need tracing instrumentation wired into its own request/response handling to propagate trace context to the next hop; the sidecar can inject and propagate this transparently at the network layer, centrally configured rather than requiring every team's codebase to carry the instrumentation correctly.

## Design question — Model answer

**Mesh proposal — argue against, for now.** The chapter's "When to skip both" section describes almost exactly this organization: "a small number of services (single digits) with a stable, known set of internal callers often doesn't need a mesh — plain load-balanced service discovery... plus disciplined per-client libraries for retries and timeouts covers the same ground with far less operational surface." Five services, one team, is squarely in "single digits... stable, known callers." The chapter is explicit that the mesh's cost is real and immediate — "a sidecar adds latency... and meaningful operational complexity — mesh upgrades, sidecar resource overhead multiplied across every pod, and a new class of 'why did this request fail' debugging that spans the mesh's control plane" — paid starting on day one, while the benefit ("we'll need it eventually") is speculative. This is the chapter's named failure mode directly: "Adopting a mesh before it's needed... for a handful of services that could be solved with a shared client library." A shared Python retry/timeout library plus Kubernetes Services for discovery covers the current need. **Trigger to revisit**: the number of services or, more specifically, the diversity of languages/teams grows to the point where "just implement retries consistently everywhere" becomes, in the chapter's words, "an organizational coordination problem rather than a technical one" — e.g., a second team starts writing services in Go and can't share the Python retry library.

**Gateway proposal — argue against, for now.** A gateway's stated purpose is centralizing concerns for north-south traffic — external clients talking to internal services — via auth termination, protocol translation, and per-client quota enforcement. Both current callers (the internal web app and the internal admin tool) are internal, known, stable callers, not the "external clients, diverse or third-party" case the gateway model addresses. Introducing a gateway here adds a hop and an operational component (per the chapter's gateway failure mode about capacity headroom and independent failure domains) to solve a problem — "consistency with how we'll eventually expose things publicly" — that doesn't exist yet for these two callers. **Trigger to revisit**: the moment the organization actually needs to expose an API to genuinely external, unknown, or third-party consumers (per Chapter 1's REST use case), or needs protocol translation (e.g., exposing gRPC services as REST/JSON to a browser client) — at that point the gateway is solving a real, current north-south problem rather than a hypothetical future one.

## Coding exercise — Model answer

```protobuf
import "google/api/annotations.proto";

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order) {
    option (google.api.http) = {
      get: "/v1/orders/{order_id}"
    };
  }

  rpc CancelOrder(CancelOrderRequest) returns (Order) {
    option (google.api.http) = {
      post: "/v1/orders/{order_id}/cancel"
      body: "*"
    };
  }
}

message GetOrderRequest {
  string order_id = 1;
}

message CancelOrderRequest {
  string order_id = 1;
  string reason = 2;
}
```

`GetOrder`'s annotation maps the `order_id` path parameter directly onto the `order_id` field of `GetOrderRequest`, matching the chapter's `GetOrder` example exactly. `CancelOrder`'s annotation maps `order_id` from the URL path the same way, and `body: "*"` tells the transcoding filter to populate the remaining request fields (here, `reason`) from the JSON request body — so a client can `POST /v1/orders/42/cancel` with `{"reason": "customer requested"}` and Envoy's gRPC-JSON transcoding filter translates it into a `CancelOrder(CancelOrderRequest(order_id="42", reason="customer requested"))` call against the internal gRPC service, exactly the pattern described in the chapter's "gRPC-to-REST/JSON translation" section — no second, hand-written REST implementation of the cancellation logic required.

## Quiz (self-check) — Answers

1. **False.** The chapter frames gateway and mesh as solving different problems by traffic direction (north-south vs. east-west), not the same problem at different scales — "teams sometimes reach for a mesh to solve an edge problem or a gateway to solve an internal routing problem," and the mismatch causes missing capabilities or unnecessary overhead. A large deployment typically needs *both*, not a mesh replacing a gateway: the gateway still owns external-facing concerns (auth termination, protocol translation for external clients) that a mesh doesn't address.
2. **North-south** (client-to-service, gateway's domain) and **east-west** (service-to-service, mesh's domain).
3. **Envoy** — the chapter states meshes "typically inject a sidecar proxy (usually Envoy) alongside every service instance."
4. Per the chapter's failure modes, it becomes **"an undocumented single point of failure"** — all north-south traffic depends on one gateway cluster that has no spare capacity and shares (rather than isolates) its failure domain from the services it fronts, so a gateway problem takes down access to otherwise-healthy backend services.
