# Chapter 5: Serialization and Validation — Exercises

*Corresponds to: [05-serialization-validation.md](05-serialization-validation.md)*

## Concept questions

1. The chapter distinguishes schema validation (`age_must_be_reasonable`) from business-rule validation (`check_email_not_taken`). Explain why `check_email_not_taken` must not be written as a Pydantic `field_validator`, even though it's technically possible to give a validator database access.
2. Pydantic v2's validation core was rewritten in Rust (`pydantic-core`). The chapter argues against hand-rolled `if`/`raise` validation "for performance" as a result. What two things do you actually lose by bypassing Pydantic models in favor of hand-rolled checks, beyond the performance argument itself?
3. The chapter calls `/orders.csv` an anti-pattern that "conflates resource identity with representation." Explain what that phrase means concretely, and describe how the chapter's `Accept`-header-based example avoids the same problem.
4. The chapter claims "sparse fieldsets solve 80% of the over-fetching pain in REST APIs with 5% of GraphQL's operational complexity." What specific REST mechanism is it referring to, and what does the chapter say is the common overcorrection engineers make instead?

## Design question (scenario-style)

Two incidents land on your desk in the same week, both traced back to patterns from this chapter:

**Incident A**: Your signup endpoint uses the `SignupRequest` model shown in the chapter. An audit turns up thousands of accounts where the `age` field was submitted as `"25 "` (a string with a trailing space) or as a JSON boolean, and Pydantic silently coerced it into a valid `int` rather than rejecting the request with a `422`. Nobody wrote a validator that's technically wrong — the coercion is Pydantic's default lenient-mode behavior.

**Incident B**: A hotfix to a different endpoint, shipped under time pressure, replaced the normal `response_model=OrderOut` handler with one that returns a hand-built `dict` directly. For about six hours, the response briefly included an internal `cost_basis` field that should never reach customers.

Using this chapter's "Silent coercion surprises" and "Response model drift" failure modes, explain the underlying cause of each incident. Then propose a concrete fix for each — including any Pydantic configuration relevant to Incident A, and any process or tooling safeguard (not just "be more careful") relevant to Incident B.

## Coding exercise

This handler merges the content-negotiation pattern and the sparse-fieldset pattern shown separately in this chapter:

```python
@app.get("/orders/{order_id}")
async def get_order(order_id: str, request: Request, fields: str | None = None):
    order = await fetch_order(order_id)
    if request.headers.get("accept") == "text/csv":
        buf = io.StringIO()
        writer = csv.writer(buf)
        writer.writerow(order.keys())
        writer.writerow(order.values())
        return PlainTextResponse(buf.getvalue(), media_type="text/csv")
    full = OrderOut.model_validate(order)
    if fields:
        requested = set(fields.split(","))
        return full.model_dump(include=requested)
    return full
```

It has a bug: a request with `Accept: text/csv` and `?fields=status,total` should return a CSV containing only those two columns, but it always returns every field in the CSV branch — sparse fieldsets are only honored on the JSON path. Fix the handler so field filtering applies to both branches, without duplicating the field-filtering logic.

## Quiz (self-check)

1. True or false: Pydantic v2 is slower than v1 at validation because it added stricter type-checking on top of v1's behavior. *(Explain your answer.)*
2. According to the chapter, where does business-rule validation like "is this email already registered" belong, and why does it need to live there rather than in the Pydantic model?
3. Which HTTP request header should a REST API inspect to decide response format, instead of baking the format into the URL path?
4. Name the two mitigation strategies the chapter gives for REST's over-fetching problem when a single response model has to serve both a mobile list view and an admin dashboard.

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
