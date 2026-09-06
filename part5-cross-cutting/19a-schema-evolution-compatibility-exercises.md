# Chapter 19: Schema Evolution and Backward Compatibility — Exercises

*Corresponds to: [19-schema-evolution-compatibility.md](19-schema-evolution-compatibility.md)*

## Concept questions

1. The chapter says renaming a Protobuf field is "safe for the wire format itself... but breaking for any code... that depends on the field name." Explain why the wire format doesn't care about the rename, and use the chapter's `grpc-gateway` example to explain what does break.
2. Proto3 defaults unrecognized enum values to the zero value. The chapter calls this "worth being deliberate about." Describe a concrete scenario where this default behavior could cause a client to silently misinterpret data rather than fail loudly.
3. The chapter distinguishes "technically breaking" from "breaking for a field nobody actually queries anymore," and says schema-diffing tools that check against real production query logs are "meaningfully stronger" than purely structural compatibility checks. Explain this distinction and why a structural-only check (like `buf breaking`, applied to GraphQL) would give a noisier or less actionable signal.
4. The chapter warns that applying REST's "just add a version" mental model to GraphQL or Protobuf causes problems. Describe the two distinct failure modes it names, and explain why they're different from each other even though both stem from the same mistaken analogy.
5. The chapter adds a REST/OpenAPI compatibility section to parallel the Protobuf and GraphQL sections. Compare the enforcement mechanism for each of the three protocols' compatibility checks — what artifact gets diffed, and what tool does the diffing — and explain why the underlying goal is described as "identical across all three" despite the different mechanisms.

## Design question (interview-style)

A team maintaining an internal `Order` gRPC service and its federated GraphQL equivalent wants to ship two changes in the same sprint: (1) remove the now-unused `int32 line_item_count = 4` field from the `.proto` message and reuse tag `4` for a new `int64 total_amount_cents` field, and (2) change the GraphQL `Order.discountCode` field from nullable to non-nullable, because "every order has one now."

For each change, explain specifically what breaks, for which consumers, and why the engineer's justification ("it's unused" / "it's always populated now") doesn't make the change safe per this chapter's rules. Then propose a safe migration path for each — the specific mechanism (not just "add a version") that lets the change ship without breaking an existing consumer, and what you'd want CI to enforce so this pair of changes couldn't ship this way again.

## Coding exercise

Using the pattern from this chapter's `Order` message example, write the resulting `.proto` message for the migration in the design question above: removing `line_item_count` (tag 4) safely and adding `total_amount_cents` as a new field. Follow the `reserved` pattern shown in the chapter exactly (both the reserved tag number and the reserved field name). Then, in one or two sentences, explain what would go wrong for an old client still sending or reading tag `4` as `line_item_count` if you had instead just deleted the field and assigned tag `4` to `total_amount_cents` directly.

## Quiz (self-check)

1. True or false: renaming a Protobuf field breaks the wire format for clients still using the old generated stubs. *(Explain your answer.)*
2. What GraphQL directive lets a field be marked for future removal without immediately breaking clients still using it?
3. Name the CI tool the chapter recommends running against every `.proto` change to catch breaking changes against the previous committed schema.
4. Per the chapter, what organizational signal (in terms of team/consumer count) indicates it's time to introduce a schema registry, even before it "feels urgent"?
5. Per the chapter's new REST/OpenAPI section, is "making a previously-optional request field required" classified as safe or breaking, and why?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
