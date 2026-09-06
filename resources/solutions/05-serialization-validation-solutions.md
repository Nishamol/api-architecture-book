# Chapter 5: Serialization and Validation — Solutions

*Corresponds to: [part2-rest/05a-serialization-validation-exercises.md](../../part2-rest/05a-serialization-validation-exercises.md)*

## Concept questions — Model answers

1. `check_email_not_taken` requires a database round-trip to answer — it can only be evaluated against the current state of the system, which by definition is not something the request payload alone can tell you. Putting it inside a Pydantic `field_validator` couples pure schema validation (a fast, synchronous, side-effect-free check of shape and type) to I/O latency and the availability of a live database connection at model-construction time, which the chapter explicitly calls out as a failure mode ("Business rules baked into Pydantic validators"). It also breaks testability: a `SignupRequest` model should be constructible and testable in complete isolation, with no database running at all, and a validator that silently reaches out to a DB destroys that property along with making the model's behavior depend on ordering and mocking concerns that have nothing to do with what the model is for. Business-rule validation belongs in the service layer specifically because that's where DB access is already expected and where the validation can be tested independently of the HTTP/schema layer.
2. First, correctness coverage: Pydantic's validation, including coercion rules, type checking, constrained types, and edge cases across nested models, has been exercised across a huge number of real-world schemas — hand-rolled `if`/`raise` checks are far more likely to miss an edge case (unicode handling, numeric boundary conditions, nested optional fields) than to beat a mature, widely-used library on both correctness and speed. Second, and just as costly: bypassing Pydantic models loses the automatic OpenAPI schema generation FastAPI derives directly from the model's type annotations. Hand-rolled validation means you either maintain the OpenAPI schema by hand (which drifts from actual behavior) or ship an API without accurate generated documentation — a real ongoing cost, not just a one-time inconvenience.
3. Baking the format into the URL means `/orders.csv` and `/orders/42` (or `/orders.json`) are treated as *different resources* by anything that reasons about URLs — caches, routers, links — even though they represent the exact same underlying order data in two different serializations. This conflates "what thing am I identifying" (a specific order) with "how do I want it represented" (JSON vs. CSV), which is precisely what `Accept`/`Content-Type` content negotiation exists to separate: the chapter's example uses a single URL, `/orders/{order_id}`, and branches purely on the `Accept` header to decide whether to return `JSONResponse` or a CSV via `PlainTextResponse`. The resource identity (`/orders/42`) stays constant regardless of representation, which is the correct REST relationship between a resource and its representations.
4. It's referring to **sparse fieldsets** — the `?fields=id,status,total` pattern applied against `OrderOut.model_dump(include=requested)`, letting different callers request different subsets of the same underlying resource shape without introducing a second protocol. The chapter names "reaching for GraphQL specifically to solve this one problem, without the accompanying scaling concerns in Chapter 9" as the common overcorrection — i.e., adopting an entire second query protocol, with its own resolver graph, N+1 defenses, and operational surface, purely to solve over-fetching, when sparse fieldsets solve the bulk of that same problem within REST at a fraction of the operational cost.

## Design question — Model answer

**Incident A (silent `age` coercion).** Pydantic's default validation mode is lenient: it will actively try to coerce compatible-looking input (a numeric string, in some configurations even certain booleans) into the declared type rather than rejecting anything that isn't already exactly that type. This is convenient for genuinely benign cases (a form field that naturally arrives as a string) but means malformed or unexpected client input — a stray trailing space, a boolean where an int belongs — is silently accepted instead of surfaced as a `422`, which is exactly the "Silent coercion surprises" failure mode: "Pydantic's lenient mode coercing `"123"` to `123` for an `int` field, masking a client-side bug that should have been rejected as a `422`." The concrete fix is to switch the model (or the specific field) into **strict mode** — Pydantic v2 supports `model_config = ConfigDict(strict=True)` on the model, or `Field(strict=True)` per-field — so that `age` must arrive as an actual JSON integer and anything else is rejected outright rather than coerced. This should be applied deliberately per-field or per-model rather than globally, since some endpoints may still want lenient coercion for genuinely interchangeable input; the point is that the choice should be explicit, not accidental.

**Incident B (response model drift leaking `cost_basis`).** This is the "Response model drift" failure mode named directly in the chapter: "hand-serialized dicts bypassing `response_model` entirely in a 'quick fix,' reintroducing the field-leakage risk from Chapter 4." The underlying cause isn't a one-off mistake so much as the absence of a guardrail that would have caught it before shipping — a hotfix under time pressure bypassed the declared `response_model=OrderOut` contract, and nothing in the pipeline flagged that the handler's return type no longer matched its declared output contract. The fix isn't "be more careful" — it's a structural safeguard: enforce a lint/CI check (or a code-review checklist item) that flags any handler using `response_model=...` where the return statement doesn't route through `.model_dump()`/`model_validate()` against that model, and/or add a contract test per endpoint that asserts the response body's keys are a subset of the declared `response_model`'s fields. That converts "did the reviewer happen to notice" into something CI catches automatically, which matters most exactly when a fix ships under time pressure and human review is most likely to be rushed.

## Coding exercise — Model answer

```python
@app.get("/orders/{order_id}")
async def get_order(order_id: str, request: Request, fields: str | None = None):
    order = await fetch_order(order_id)
    full = OrderOut.model_validate(order)

    if fields:
        requested = set(fields.split(","))
        data = full.model_dump(include=requested)
    else:
        data = full.model_dump()

    if request.headers.get("accept") == "text/csv":
        buf = io.StringIO()
        writer = csv.writer(buf)
        writer.writerow(data.keys())
        writer.writerow(data.values())
        return PlainTextResponse(buf.getvalue(), media_type="text/csv")

    return data
```

The fix moves field filtering (`model_dump(include=requested)`) to happen once, before branching on `Accept`, so both the JSON and CSV paths serialize from the same already-filtered `data` dict rather than the CSV branch reading directly from the unfiltered `order`. This removes the duplication risk entirely — there's exactly one place sparse fieldsets are applied, and both representations (JSON, CSV) are built from its output, so a future third representation (e.g. XML) gets the same filtering for free instead of needing to remember to reapply the `fields` logic yet again.

## Quiz (self-check) — Answers

1. **False.** Pydantic v2's validation core (`pydantic-core`) was rewritten in Rust, which made validation that used to show up meaningfully in profiler flame graphs under load "close to free" — v2 is faster than v1, not slower, despite doing at least as much validation work.
2. In the **service layer**, where it has access to the database and can be tested independently of the HTTP layer — not inside the Pydantic model, because business-rule checks require I/O and current system state, which schema validation isn't meant to depend on.
3. The **`Accept`** header (with `Content-Type` governing what the client sends back on write requests) — content negotiation should be driven by headers, not encoded into the URL path.
4. **Sparse fieldsets** (`?fields=id,status,total` applied against `.model_dump(include=...)`) and **splitting into purpose-built response models per endpoint** (e.g. `OrderSummaryOut` vs. `OrderDetailOut`).
