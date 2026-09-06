# API Architecture in Python

### REST, GraphQL, and gRPC in Depth — for Senior and Lead Engineers

Most API books teach you how to draw a route decorator and call it a day. This book assumes you already know how to do that. It's about the decisions that show up later: why a REST resource model falls apart under a mobile client's query patterns, why a GraphQL schema turns into an N+1 minefield in production, why gRPC deadlines don't propagate the way you assumed, and how to choose between all three when a platform team is staring at a blank whiteboard.

The book is organized in seven parts:

**Part I — Foundations.** The landscape of API styles, the communication-model choice (synchronous / asynchronous / streaming), and the HTTP/network layer everything else sits on.

**Part II — REST APIs.** Resource design, building REST services in Python with FastAPI and Django REST Framework, serialization, and versioning.

**Part III — GraphQL APIs.** Schema design, building resolvers in Python with Strawberry and Graphene, and the scaling problems (N+1, complexity limits, federation) that only show up once you have real traffic.

**Part IV — gRPC Services.** Protocol Buffers, streaming RPC types, and building performant gRPC services in Python.

**Part V — Cross-Cutting Concerns.** The concerns that apply regardless of protocol: authentication, gateways and service mesh, rate limiting, error handling, observability, testing, and schema evolution.

**Part VI — Asynchronous and Event-Driven APIs.** When the answer to "which protocol" is "none — this shouldn't be a synchronous call": `202` and job resources, webhooks, message queues and event streams, delivery semantics, the transactional outbox, sagas, and AsyncAPI.

**Part VII — Putting It All Together.** A decision framework for choosing (or combining) protocols and communication models, and a case study of a production platform that runs all of them side by side.

Each chapter includes working Python code, the trade-offs behind the decisions, and the failure modes you only learn about after an incident review. Every chapter ends with exercises (concept questions, a design scenario, a coding exercise, and a self-check quiz); model answers live in [resources/solutions/](resources/solutions/).

---

## Table of Contents

### Part I — Foundations

1. [The API Architecture Landscape](part1-foundations/01-introduction.md) · [exercises](part1-foundations/01a-introduction-exercises.md)
2. [HTTP Transport Internals](part1-foundations/02-http-fundamentals.md) · [exercises](part1-foundations/02a-http-fundamentals-exercises.md)

### Part II — REST APIs

3. [REST API Design Principles](part2-rest/03-rest-design-principles.md) · [exercises](part2-rest/03a-rest-design-principles-exercises.md)
4. [Building REST APIs in Python](part2-rest/04-building-rest-apis-python.md) · [exercises](part2-rest/04a-building-rest-apis-python-exercises.md)
5. [Serialization and Validation](part2-rest/05-serialization-validation.md) · [exercises](part2-rest/05a-serialization-validation-exercises.md)
6. [Versioning REST APIs](part2-rest/06-versioning-rest-apis.md) · [exercises](part2-rest/06a-versioning-rest-apis-exercises.md)

### Part III — GraphQL APIs

7. [GraphQL Schema Design and Execution Model](part3-graphql/07-graphql-fundamentals.md) · [exercises](part3-graphql/07a-graphql-fundamentals-exercises.md)
8. [Building GraphQL APIs in Python](part3-graphql/08-building-graphql-apis-python.md) · [exercises](part3-graphql/08a-building-graphql-apis-python-exercises.md)
9. [GraphQL at Scale](part3-graphql/09-graphql-at-scale.md) · [exercises](part3-graphql/09a-graphql-at-scale-exercises.md)

### Part IV — gRPC Services

10. [gRPC Contracts and Protocol Buffers](part4-grpc/10-grpc-fundamentals.md) · [exercises](part4-grpc/10a-grpc-fundamentals-exercises.md)
11. [Building gRPC Services in Python](part4-grpc/11-building-grpc-services-python.md) · [exercises](part4-grpc/11a-building-grpc-services-python-exercises.md)
12. [gRPC Performance and Streaming](part4-grpc/12-grpc-performance-streaming.md) · [exercises](part4-grpc/12a-grpc-performance-streaming-exercises.md)

### Part V — Cross-Cutting Concerns

13. [Authentication and Authorization](part5-cross-cutting/13-authentication-authorization.md) · [exercises](part5-cross-cutting/13a-authentication-authorization-exercises.md)
14. [API Gateways and Service Mesh](part5-cross-cutting/14-api-gateway-service-mesh.md) · [exercises](part5-cross-cutting/14a-api-gateway-service-mesh-exercises.md)
15. [Rate Limiting and Throttling](part5-cross-cutting/15-rate-limiting-throttling.md) · [exercises](part5-cross-cutting/15a-rate-limiting-throttling-exercises.md)
16. [Error Handling and Resilience](part5-cross-cutting/16-error-handling-resilience.md) · [exercises](part5-cross-cutting/16a-error-handling-resilience-exercises.md)
17. [Observability](part5-cross-cutting/17-observability.md) · [exercises](part5-cross-cutting/17a-observability-exercises.md)
18. [Testing Strategies](part5-cross-cutting/18-testing-strategies.md) · [exercises](part5-cross-cutting/18a-testing-strategies-exercises.md)
19. [Schema Evolution and Backward Compatibility](part5-cross-cutting/19-schema-evolution-compatibility.md) · [exercises](part5-cross-cutting/19a-schema-evolution-compatibility-exercises.md)

### Part VI — Asynchronous and Event-Driven APIs

20. [Asynchronous and Event-Driven APIs](part6-async-apis/20-asynchronous-event-driven-apis.md) · [exercises](part6-async-apis/20a-asynchronous-event-driven-apis-exercises.md)

### Part VII — Putting It All Together

21. [Choosing the Right Protocol](part7-synthesis/21-choosing-the-right-protocol.md) · [exercises](part7-synthesis/21a-choosing-the-right-protocol-exercises.md)
22. [Production Case Study — A Polyglot API Platform](part7-synthesis/22-production-case-study.md) · [exercises](part7-synthesis/22a-production-case-study-exercises.md)

---

This same structure is also available as [SUMMARY.md](SUMMARY.md), used for navigation when the book is built with GitBook.
