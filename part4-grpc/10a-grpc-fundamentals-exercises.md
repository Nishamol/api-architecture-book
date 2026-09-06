# Chapter 10: gRPC Contracts and Protocol Buffers — Exercises

*Corresponds to: [10-grpc-fundamentals.md](10-grpc-fundamentals.md)*

## Concept questions

1. The chapter says field tags (the `= 1`, `= 2` numbers in a `.proto` message) are "not default values" and are "permanent once shipped." Explain what's actually encoded on the wire instead of field names, and why reusing a tag number for a different field in a later version corrupts data rather than just breaking a build.
2. Using the `TelemetryService` example (`Report`, `Subscribe`, `Sync`), match each RPC to one of the four RPC types and explain, for each, which side controls when the stream ends.
3. The chapter distinguishes a gRPC deadline from a client-side timeout by describing propagation through a call chain (A calls B calls C). Explain concretely what goes wrong in a system without deadline propagation, and why the chapter calls fixing this "one of the highest-leverage reliability investments" that's also "one of the most commonly skipped."
4. gRPC defines its own status code enum instead of reusing HTTP status codes even though it rides on HTTP/2. Give the specific example the chapter uses to justify this (a distinction HTTP has no native concept of), and explain why forcing that distinction into HTTP status codes would lose information.
5. The chapter adds `oneof`, `map<K,V>`, and `google.protobuf.Timestamp` to the `Order` message. Explain what `oneof` guarantees about `credit_card_token` and `bank_account_token` that two independent optional fields would not, and why `google.protobuf.Timestamp` is preferable to a hand-rolled string or epoch-integer field for `created_at` in a system with services written in multiple languages.

## Design question (interview-style)

You're designing a `.proto` contract for a new `InventoryService` that three types of callers will use: (1) a warehouse-floor scanner app that checks stock levels for a single SKU and needs an immediate answer; (2) a nightly batch job that uploads a stream of ~200,000 stock-count corrections from a physical inventory audit and only needs a single confirmation once everything is ingested; (3) a live dashboard that should update stock-level widgets in real time as inventory changes anywhere in the warehouse, for as long as the dashboard is open.

Sketch the `service` block of the `.proto` (RPC signatures only — you don't need full message bodies) choosing the correct RPC type from the chapter's four for each caller, and justify each choice. Then explain what would go wrong, concretely, if you implemented caller (2) as a unary RPC that accepted a `repeated StockCorrection` field instead of a client-streaming RPC.

## Coding exercise

A teammate is evolving the `Order` message from this chapter for a v2 release. The original:

```protobuf
message Order {
  string id = 1;
  string status = 2;
  repeated LineItem line_items = 3;
}
```

Their v2 diff removes `status` (the team decided order status should live in a separate `OrderStatus` message going forward) and adds a new field, `tracking_number`:

```protobuf
message Order {
  string id = 1;
  repeated LineItem line_items = 3;
  string tracking_number = 2;
}
```

Identify what's wrong with this diff relative to the chapter's explanation of field tags, explain what happens on the wire for a client still running the old `Order` definition, and rewrite the message so the change is safe.

## Quiz (self-check)

1. True or false: two different fields can safely share the same field tag number as long as they're never sent in the same message. *(Explain your answer.)*
2. Which of the four RPC types has no request/response round-trip boundary at all — both sides can send at any time?
3. Name the two files `protoc`/`grpc_tools.protoc` generates from a `.proto` file, and what each one contains.
4. What gRPC status code would a service return for a request it refuses to process because a required field was missing, versus one it refuses because the caller lacks permission?
5. True or false: a `oneof` field group is Protobuf's mechanism for a field that can hold one of several different message types, similar to a GraphQL union. *(Explain your answer.)*

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
