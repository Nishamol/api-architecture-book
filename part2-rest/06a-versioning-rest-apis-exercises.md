# Chapter 6: Versioning REST APIs — Exercises

*Corresponds to: [06-versioning-rest-apis.md](06-versioning-rest-apis.md)*

## Concept questions

1. Compare URI versioning and header versioning on exactly one axis: HTTP caching. Explain why URI versioning is "compatible with HTTP caching with zero extra configuration" while header versioning "breaks naive HTTP caching."
2. The chapter says the "no versioning, additive-only" strategy "scales poorly once you have external integrators you can't schedule migrations with." Explain, using the chapter's own reasoning, exactly what breaks down about additive-only changes once your consumer base is no longer coordinated.
3. The chapter says "deprecation is a project, not a flag." List the four concrete elements it says a real deprecation needs, and explain why a changelog entry alone fails to move client traffic.
4. Consumer-driven contract testing (via Pact) is described as "the single most effective technique for versioning discipline on internal APIs where you can require consumers to participate." Why does the phrase "where you can require consumers to participate" matter — what kind of API consumer does this technique *not* work for, and why?

## Design question (interview-style)

Your public `/v1/orders/{order_id}` endpoint returns a field named `total`. Telemetry and support tickets reveal the field is ambiguous in practice — some third-party integrators treat it as pre-tax, others as post-tax, and both readings are currently "correct" for different callers depending on when they integrated. Product wants to split it into `subtotal` and `total_with_tax`, and stop returning `total` entirely, within 6 months. You have telemetry showing this endpoint is called by 340 distinct third-party API keys, 12 of which account for 90% of total call volume.

Using this chapter's versioning strategies, backward-compatible change categories, and deprecation requirements, design the full rollout: what (if anything) can ship as an additive change to `/v1` first, what specifically requires a `/v2` and why, what headers and dates you'd attach to the deprecated path, and how you'd sequence outreach between the 12 high-volume callers and the remaining 328.

## Coding exercise

This chapter's deprecation example marks `/v1/orders/{order_id}` deprecated using `Deprecation`, `Sunset`, and `Link` headers, with a sunset date of `Wed, 01 Apr 2026 00:00:00 GMT`. A teammate has written the following `/v2` handler to implement the `total` → `subtotal` / `total_with_tax` split described above:

```python
@app.get("/v2/orders/{order_id}")
async def get_order_v2(order_id: str):
    order = await fetch_order_v1_shape(order_id)
    order["subtotal"] = order["total"] - order["tax"]
    order["total_with_tax"] = order["total"]
    del order["total"]
    return order
```

Using this chapter's backward-compatible change categories, explain why this change could not simply have shipped as an additive update to `/v1` instead of requiring a new `/v2` endpoint. Then write the corrected `/v1` handler — reusing the deprecation pattern and sunset date already established in the chapter — so that it's fully wired up as a real deprecation project rather than just a stale comment.

## Quiz (self-check)

1. True or false: adding a new enum value to an existing field is always safe to ship without a version bump. *(Explain your answer, using the chapter's own caveat.)*
2. Which RFC defines the `Sunset` header used in this chapter's deprecation example?
3. Name one downside of URI versioning and one downside of header versioning, each specific to a different concern raised in the chapter.
4. What tool does the chapter name as an example of consumer-driven contract testing?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
