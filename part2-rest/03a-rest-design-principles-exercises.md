# Chapter 3: REST API Design Principles — Exercises

*Corresponds to: [03-rest-design-principles.md](03-rest-design-principles.md)*

## Concept questions

1. The chapter allows `POST /orders/42/cancel` as an "acceptable escape hatch" but treats `PATCH /orders/42` with a status field as the wrong way to model the same cancel operation. Using the chapter's framing of the URL as noun and the HTTP method as verb, explain what specifically goes wrong when you bury an action inside a `PATCH` payload instead.
2. The chapter says Level 2 of the Richardson Maturity Model is "where most production 'REST' APIs actually sit, and it's a perfectly reasonable place to stop." Explain what distinguishes Level 2 from Level 1, and name the kind of API where Level 3 (HATEOAS) is actually worth its complexity.
3. `PUT` and `DELETE` are idempotent by HTTP contract, but the chapter's `POST /orders` example still needs an explicit idempotency key. Explain why `POST`'s lack of default idempotency creates a real production risk under mobile network conditions specifically, and how the idempotency-key pattern shown in the chapter closes that gap.
4. Why does offset-based pagination (`?offset=100&limit=20`) silently skip or duplicate rows under concurrent writes, while cursor-based pagination (`?cursor=eyJpZCI6NDJ9`) doesn't have the same failure mode? Ground your answer in what each strategy actually uses to determine "where the next page starts."
5. The chapter's filtering/sorting addition requires both to be checked against an explicit allowlist (`ALLOWED_FILTER_FIELDS`, `ALLOWED_SORT_FIELDS`). Explain the two distinct failure modes this allowlist prevents at once, and why "the client asked for this exact field name" isn't sufficient justification to accept it unchecked.

## Design question (interview-style)

You're brought in to review an internal order-management REST API before it opens up to third-party integrators. The current design:

- Cancelling an order is done via `POST /api?action=cancelOrder&id=42`.
- Updating an order is done via `PUT /orders/42`, and the handler increments an internal `version` counter by 1 on every call, regardless of whether the payload actually changed anything.
- Every response — success or failure — returns HTTP `200`, with a JSON body like `{"success": false, "error": "insufficient inventory"}` on failure.
- `GET /orders` returns the entire orders table, unpaginated, sorted by insertion order.

Using this chapter's principles, identify each violation, state which Richardson Maturity Model level this API is currently stuck at, and give the corrected design (URLs, HTTP methods, status codes, pagination strategy) for each problem. Then explain why these fixes need to ship *before* third-party integrators are let in, rather than being deferred as "we'll clean it up later."

## Coding exercise

Here is a teammate's implementation of the idempotency-key pattern from this chapter, wired up to a real database instead of the chapter's in-memory dict:

```python
@app.post("/orders")
async def create_order(payload: dict, idempotency_key: str = Header(...), db=Depends(get_db)):
    order = await db.insert_order({"id": generate_id(), **payload})
    if idempotency_key in _seen_keys:
        return _seen_keys[idempotency_key]
    _seen_keys[idempotency_key] = order
    return order
```

This passes a naive test (calling it twice with the same key returns the same response body both times). Identify the bug, explain why it's worse than it looks even though the *response* the client sees is correctly deduplicated, and rewrite the handler so the database write itself only happens once per idempotency key.

## Quiz (self-check)

1. True or false: `POST` is idempotent by default under HTTP semantics, the same way `PUT` and `DELETE` are. *(Explain your answer.)*
2. Name the three distinct status codes this chapter uses to distinguish a malformed request, a validation failure, and an optimistic-concurrency conflict.
3. Between cursor-based and offset-based pagination, which does the chapter recommend for collections with high write volume, and why?
4. What is the "escape hatch" this chapter describes for operations that don't map cleanly to CRUD? Give the URL pattern it uses as an example.
5. Per the chapter's filtering/sorting example, what does checking `sort_field` against `ALLOWED_SORT_FIELDS` protect against, beyond just rejecting malformed input with a `422`?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
