# Chapter 20: Asynchronous and Event-Driven APIs — Exercises

*Corresponds to: [20-asynchronous-event-driven-apis.md](20-asynchronous-event-driven-apis.md)*

## Concept questions

1. The chapter gives six defensible shapes for an operation that isn't instant. For a 30-second credit check triggered by a browser-based form (no server on the client side to receive a callback), walk through the chapter's decision flowchart and name the shape it lands on. Then explain why the "202 + webhook" branch is specifically ruled out here.
2. The chapter says "exactly-once" delivery "is a property of *your consumer*, not a checkbox on the broker." Unpack this: what three things does the chapter say combine to produce effectively-once processing, and why can none of them be provided by the broker alone?
3. Explain the dual-write problem in your own words, then explain precisely how the transactional outbox pattern eliminates it — including what makes the `UPDATE` and the `INSERT INTO outbox` safe together when the original `UPDATE` + `publish` was not.
4. The chapter claims a webhook receiver "should do almost nothing synchronously." Describe the specific failure loop that occurs when a receiver does its real processing before returning `200` and the backend is slow — name each step and why it compounds.

## Design question (interview-style)

You own the `payments` service. When a payment settles, four things must happen: the order is marked paid, the customer gets a receipt email, the analytics pipeline records revenue, and the fraud service re-scores the customer. Today all four are synchronous calls made from the payment-settlement handler, and settlement latency has crept to 1.2s with occasional failures when the email provider is down (which fails the whole settlement).

Redesign this as an event-driven flow. Specify: what event `payments` publishes and what's in its envelope; how you solve the dual-write between the payments database and the broker; which of the four consumers (if any) should stay synchronous and why; and how the receipt-email consumer avoids sending two emails when the broker redelivers. Then identify one business operation in this list that would need saga-style compensation if it were part of the settlement rather than a downstream reaction, and explain what the compensating action would be.

## Applied exercise

The chapter's webhook receiver snippet verifies the signature, dedupes on `event_id`, enqueues, and returns `200`.

1. **Extend the signature check** to also reject replays: given the `Webhook-Timestamp` header and a 5-minute tolerance, write the additional guard and explain why the timestamp must be inside the HMAC rather than just alongside it.
2. **Challenge the dedupe store.** The snippet uses an in-memory `set` with a note to "back with Redis/DB, TTL'd." Explain what breaks if (a) the set is per-process and the receiver runs three replicas, and (b) the TTL is shorter than the sender's total retry window. For each, describe the concrete duplicate-processing scenario it allows.

## Quiz (self-check)

1. True or false: returning `200 OK` from a `POST` that kicks off a 30-second background job is the correct status code. *(Explain your answer.)*
2. In a log-based stream like Kafka, ordering is guaranteed at what scope — global, per-topic, or per-partition — and what is the practical consequence for how you choose message keys?
3. What does the chapter say a producer should do with a webhook subscription whose endpoint has failed every delivery for an extended period, and why is "keep retrying" the wrong answer?
4. Name the consumer-side counterpart to the transactional outbox pattern, and state what it records and in which transaction.
5. Per the chapter, what has to be propagated through the broker for a distributed trace to span the async boundary instead of fragmenting at the queue?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
