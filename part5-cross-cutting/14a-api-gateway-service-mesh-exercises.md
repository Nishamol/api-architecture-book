# Chapter 14: API Gateways and Service Mesh — Exercises

*Corresponds to: [14-api-gateway-service-mesh.md](14-api-gateway-service-mesh.md)*

## Concept questions

1. The chapter defines a gateway as handling "north-south traffic" and a mesh as handling "east-west traffic," and warns that "teams sometimes reach for a mesh to solve an edge problem or a gateway to solve an internal routing problem." Give a concrete example of each mismatch and describe what "shows up" as a result (the chapter names two possible symptoms).
2. Explain how `grpc-gateway`-style transcoding, driven by `google.api.http` annotations in a `.proto` file, lets a team "build internal services gRPC-first... while still serving REST-consuming external clients, without maintaining two separate implementations of the same business logic." What would the team have to build instead if they *didn't* use this pattern?
3. The chapter distinguishes a BFF GraphQL layer from GraphQL federation (Chapter 9) by ownership and purpose. Explain the distinction in the chapter's own terms, and describe the specific failure mode that occurs when a BFF drifts toward acting like the thing it isn't.
4. A service mesh sidecar gives you "automatic mTLS between services (Chapter 13)... circuit breaking (Chapter 16), and distributed tracing propagation (Chapter 17)... all configured centrally rather than reimplemented per service in every language your organization uses." Pick any two of these three cross-referenced capabilities and explain, in the chapter's terms, what specifically the mesh is centralizing relative to the per-service, per-language alternative it replaces.

## Design question (interview-style)

Your organization runs 5 backend services in Python, all written and operated by a single 8-person platform team, serving one internal web app and one internal admin tool — both known, stable callers. A newly hired staff engineer proposes adopting Istio "to get mTLS, retries, and circuit breaking for free, since we'll need it eventually as we grow." Separately, they propose putting an API gateway in front of the two internal callers "for consistency with how we'll eventually expose things publicly."

Using the chapter's "When to skip both" section and its stated costs of each technology, evaluate both proposals for the *current* state of this organization (not the hypothetical future one). For each proposal, either endorse it or argue against it using specific costs or non-needs the chapter identifies, and describe the concrete trigger condition(s) that should change your answer later.

## Coding exercise

A team is exposing an internal gRPC `OrderService` externally via Envoy's gRPC-JSON transcoding, following the pattern in this chapter. Their current `.proto` only has the RPC definition, with no HTTP annotation:

```protobuf
service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc CancelOrder(CancelOrderRequest) returns (Order);
}

message GetOrderRequest {
  string order_id = 1;
}

message CancelOrderRequest {
  string order_id = 1;
  string reason = 2;
}
```

Add `google.api.http` annotations so that `GetOrder` is reachable as `GET /v1/orders/{order_id}` and `CancelOrder` is reachable as `POST /v1/orders/{order_id}/cancel` with `reason` supplied in the JSON request body, matching the transcoding pattern shown in the chapter. Include the necessary import.

## Quiz (self-check)

1. True or false: a service mesh and an API gateway solve the same problem at different scales, so a large enough deployment should replace its gateway with a mesh. *(Explain your answer.)*
2. Name the two traffic directions the chapter uses to distinguish gateway from mesh responsibilities.
3. What sidecar proxy does the chapter name as the typical implementation underlying Istio, Linkerd, and Consul Connect?
4. According to the chapter's failure modes, what happens when a gateway cluster has "no capacity headroom or independent failure domain from the services behind it"?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
