# Chapter 21: Production Case Study — A Polyglot API Platform — Exercises

*Corresponds to: [21-production-case-study.md](21-production-case-study.md)*

## Concept questions

1. The DataLoader lifetime bug described in this chapter had DataLoaders "created once per gateway process instead of once per request." Explain what this caused (in terms of what leaked, and to whom) and why it connects to what the chapter calls "Chapter 8's exact warning."
2. In the deadline-propagation incident, the orders-to-inventory call had a deadline, but the chapter says inventory "kept working on requests orders had already timed out and abandoned." Explain the specific mechanical cause (what was missing when the downstream channel was created) and why this wastes capacity specifically "during exactly the traffic spike that caused the original slowness."
3. The gateway forwards "a service-to-service token (not the original user JWT)" to each subgraph, yet the chapter insists per-field user-level authorization is still enforced at the subgraph level. Explain why skipping that per-field check — on the theory that "the gateway already checked the session" — would be a mistake, given what the service-to-service token does and doesn't represent.
4. The chapter says the REST API for third-party sellers "cannot drift apart silently" from the internal gRPC contract. What specific mechanism (named in the chapter) guarantees this, and how does it differ from a REST API that's hand-maintained alongside a separate gRPC service?

## Design question (interview-style)

Picture yourself as the engineer assigned to add a new internal gRPC call from the customer-profile service to the billing service (for a new "billing history on customer profile" feature), six months after the deadline-propagation postmortem described in this chapter. You create the channel the same way the original, buggy orders-to-inventory-to-warehouse chain was built, without explicitly wiring deadline propagation through.

Using the chapter's description of what actually went wrong in that postmortem, predict specifically what will happen the next time there's a traffic spike that causes the customer-profile-to-billing call to run long. Then describe the checklist item the chapter says this incident produced, and explain concretely what code-review or CI check would need to exist to catch your channel before it ships, given that the bug is invisible under normal (non-spiky) load.

## Applied exercise

The platform's leadership wants to add a fourth consumer population: an internal analytics/BI system that needs bulk, near-real-time read access across orders, inventory, customer-profile, and billing data — for example, "give me every order and its line items updated in the last 5 minutes, joined with customer tier."

Using the three existing layers as a model (gRPC internal, federated GraphQL for first-party apps, REST-via-transcoding for third parties), decide where this new consumer fits: does it reuse an existing layer, or does it need a new one? Justify your answer using the chapter's own reasoning about why each existing layer was chosen for its consumer population (client count, coordination ability, latency needs, access pattern shape). Then, by analogy to the two postmortems in this chapter, identify one concrete new failure mode this bulk/cross-service access pattern could introduce that the existing three layers weren't designed to guard against, and propose the specific guardrail (tracing, deadline, rate-limit, or auth mechanism) you'd put in place before launch.

## Quiz (self-check)

1. True or false: the postmortems in this chapter show that gRPC-internal, GraphQL-for-first-party, and REST-for-third-party was the wrong architecture for this platform. *(Explain your answer.)*
2. What load-balancing pattern do the four internal services use to avoid the Layer-4 connection-pinning trap mentioned elsewhere in the book?
3. What technology automatically handles mTLS between the four internal services, so no team had to build certificate rotation themselves?
4. What is the REST API for third-party sellers generated from, rather than being hand-maintained as a separate artifact?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
