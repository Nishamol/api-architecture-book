# Chapter 10: gRPC Contracts and Protocol Buffers — Solutions

*Corresponds to: [part4-grpc/10a-grpc-fundamentals-exercises.md](../../part4-grpc/10a-grpc-fundamentals-exercises.md)*

## Concept questions — Model answers

1. What actually gets encoded on the wire is the numeric field tag plus a wire-type indicator — not the field name string at all. A decoder reading a Protobuf message doesn't see `status`, it sees "tag 2, wire type X" and looks up what tag 2 means according to *its own* copy of the schema. If a later version reassigns tag 2 to a different field, a decoder still running the old schema will read the new field's bytes and interpret them as whatever tag 2 used to mean — it doesn't fail to compile, and it often doesn't even throw at runtime, because the wire format has no way to notice the mismatch. This is exactly why the chapter calls it "wire-format-breaking" rather than "build-breaking": the two schema versions can each compile and run fine in isolation, but any encoder/decoder pair that disagrees on what a tag means silently misinterprets data rather than erroring.

2. `Report(stream Metric) returns (Ack)` is **client streaming**: the client sends a stream of `Metric` messages and controls when the stream ends (by closing it), and the server sends back exactly one `Ack` once that happens. `Subscribe(SubscribeRequest) returns (stream Event)` is **server streaming**: the client sends one request, and the server controls the stream, producing `Event` messages for as long as it decides there's something to send (the client can also end things early by cancelling). `Sync(stream SyncMessage) returns (stream SyncMessage)` is **bidirectional streaming**: both sides stream independently over the same connection, and each side controls the end of its own stream — the chapter's example use case (a live collaborative editing backend) needs exactly this because either party can send at any time, with no fixed request/response turn-taking.

3. Without deadline propagation, a call chain A → B → C loses the caller's time budget at each hop: if A gives up on B after 2 seconds (client-side timeout on that one edge), B might still be waiting on C, and C has no idea a deadline exists at all — C keeps executing the request to completion even though nothing downstream will ever consume the result. That's wasted work: DB connections held, CPU spent, capacity consumed on a request whose answer is already worthless. The chapter calls fixing this high-leverage because deadline propagation protects the *entire* call graph with one mechanism, and calls it commonly skipped because nothing about a missing deadline shows up in normal testing or code review — a service without propagated deadlines behaves identically to one with them until the system is under load and cascading latency turns "C did unnecessary work" into "C is now also slow, which makes B slower, which makes A time out more, which sends more retries," which is precisely the kind of incident the chapter says you only discover once you're "already deep into" it.

4. The specific example is `DEADLINE_EXCEEDED` versus `CANCELLED` — a call that ran out of the time budget the caller gave it, versus a call the caller explicitly aborted, are operationally different events, but "a distinction HTTP has no native concept of," per the chapter. HTTP status codes describe the *outcome* of a request the server did process (2xx/4xx/5xx), not the *reason a caller stopped waiting* for one it may or may not have finished. Forcing both cases into, say, a generic HTTP timeout/499-style status would collapse two situations that need different responses — a deadline exceeded might warrant an alert about downstream slowness, while a client-initiated cancellation is expected, routine behavior (e.g., a user navigating away) — into one signal, losing exactly the information an operator or a piece of retry logic needs to react correctly.

5. `oneof` guarantees that only one of `credit_card_token` or `bank_account_token` is ever set at a time — setting one clears the other, giving a mutual-exclusivity guarantee enforced by the generated code itself. Two independent optional fields would give no such guarantee: both could be set simultaneously, and nothing in the message definition tells a reader which one is supposed to "win," pushing that invariant into application logic that has to remember to enforce it manually every place the message is constructed. `google.protobuf.Timestamp` is preferable to a hand-rolled string or epoch-int for `created_at` because every generated language binding gets a native, well-defined representation of it — in a polyglot system, a Python service, a Go service, and a Java service all decode the same `Timestamp` field into their own idiomatic date/time type automatically, whereas a raw string or int field requires every service to agree, out-of-band, on a specific format or epoch convention that nothing in the schema itself enforces or documents.

## Design question — Model answer

```protobuf
service InventoryService {
  rpc GetStock(GetStockRequest) returns (StockLevel);                          // unary
  rpc UploadStockCorrections(stream StockCorrection) returns (UploadSummary);  // client streaming
  rpc WatchStockLevels(WatchStockLevelsRequest) returns (stream StockUpdate);  // server streaming
}
```

- **(1) Scanner app → `GetStock`, unary.** It needs a single immediate answer for a single SKU — "one request, one response — the gRPC equivalent of a REST call," exactly the chapter's unary description. No streaming machinery is warranted for a point lookup.
- **(2) Nightly batch upload → `UploadStockCorrections`, client streaming.** This is a direct match to the chapter's own client-streaming example: "a stream of requests, one response — e.g., a client uploading a stream of telemetry events, with the server acknowledging once at the end." The audit job pushes corrections as it produces them and only needs one confirmation when it's done.
- **(3) Live dashboard → `WatchStockLevels`, server streaming.** One request establishes interest, and the server pushes updates for as long as the dashboard stays open — the same shape as the chapter's `StreamOrderUpdates`/`Subscribe` pattern: "pushing status changes as they happen, without the client polling."

**What goes wrong with a unary `repeated StockCorrection` for (2):** all ~200,000 corrections have to be marshalled into a single Protobuf message and sent as one HTTP/2 request body, which means both the client and the server must buffer the entire payload in memory before any processing can begin — none of the incremental, bounded-memory processing a stream gives you for free. It's also not resumable: if the connection drops at correction #180,000, the client has to resend the whole batch from scratch rather than picking up where a stream left off. And large single messages risk bumping into gRPC's default per-message size limits, which a client-streaming RPC — where each `StockCorrection` is its own small message — simply doesn't run into.

## Coding exercise — Model answer

The bug: tag `2` was reused. In v1, tag 2 belongs to `status` (a `string`). In v2's diff, tag 2 is reassigned to `tracking_number` (also a `string`). Per the chapter, "field tags are permanent once shipped: reusing a tag number for a different field in a later version is a wire-format-breaking change." Because both fields happen to be the `string` type here, this failure mode is especially dangerous: a client or service still running the v1 schema will decode tag 2 expecting `status`, and will silently get the `tracking_number` value in its place — no crash, no exception, just a corrupted `status` field wherever `tracking_number` was actually populated. This is worse than a loud failure precisely because nothing signals that anything went wrong.

Correct version — reserve the old tag and name, and give the new field an unused tag:

```protobuf
message Order {
  string id = 1;
  reserved 2;
  reserved "status";
  repeated LineItem line_items = 3;
  string tracking_number = 4;
}
```

`reserved 2;` and `reserved "status";` tell `protoc` to reject any future attempt to reuse either the tag number or the old field name, turning this exact mistake into a build-time error the next time someone makes it. `tracking_number` gets its own never-before-used tag (`4`), so old clients simply don't see it (safe — unknown/absent fields are the normal case in Protobuf evolution) and new clients read it correctly, while `status` moving to a separate `OrderStatus` message elsewhere is unaffected by this fix.

## Quiz (self-check) — Answers

1. **False.** Rolling deployments mean old and new binaries run simultaneously against the same wire format, and a Protobuf decoder identifies fields purely by tag number with no awareness of "intent" — there's no way to enforce "never in the same message" across independently built and deployed services. Reusing a tag is unsafe the moment any two schema versions that disagree about it can both be in production at once, which is effectively always.
2. **Bidirectional streaming** — both sides send and receive independently over the same connection with no fixed request/response turn order.
3. `orders_pb2.py` contains the generated message classes (e.g., `Order`, `GetOrderRequest`); `orders_pb2_grpc.py` contains the generated client stub and the server servicer base class.
4. Missing a required field → `INVALID_ARGUMENT` (explicitly listed among the chapter's status codes). Lacking permission falls under the chapter's "and others" — the standard gRPC codes for this are `PERMISSION_DENIED` (identity known, access denied) or `UNAUTHENTICATED` (identity not established at all); the chapter doesn't enumerate these by name but they're part of the same fixed enum it describes.
5. **False, with an important distinction.** Both `oneof` and a GraphQL union express "one of several possibilities," but they operate at different levels. A `oneof` groups multiple *named fields* within a single message, only one of which is ever populated at a time (mutual exclusivity within one message) — it doesn't mean the RPC returns one of several different message *types*. A GraphQL union, by contrast, is exactly about a field returning one of several distinct types entirely (`SearchResult = Order | Customer`). Protobuf's closer analog to a union — a field that's actually one of several different message types — would need to be modeled with a `oneof` of message-typed fields (`oneof result { Order order = 1; Customer customer = 2; }`), which is possible but is a specific *usage* of `oneof`, not what the construct itself fundamentally is.
