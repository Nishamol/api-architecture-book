# Chapter 4: Building REST APIs in Python — Exercises

*Corresponds to: [04-building-rest-apis-python.md](04-building-rest-apis-python.md)*

## Concept questions

1. Under what circumstance does this chapter say Django REST Framework is the right choice over FastAPI, even though FastAPI is described as "the default choice for greenfield Python REST APIs since roughly 2020"? Be specific about what DRF gives you that FastAPI doesn't provide out of the box.
2. In the `create_order` example, `response_model=OrderOut` does more than shape the JSON response — the chapter calls it a security control. Explain what specific class of bug it prevents, and connect it to the "Missing `response_model`" failure mode named later in the chapter.
3. The chapter says FastAPI's `Depends` mechanism "exists as much for testability as for convenience." Using the `get_db` / `get_test_db` example, explain concretely what breaks in a test suite built on module-level singleton database connections instead, and how `dependency_overrides` avoids that problem.
4. Using the `get_report` example, explain why a single synchronous blocking call inside an `async def` handler can degrade latency for requests that have nothing to do with that handler. What is the underlying execution model (be specific about what "the event loop" is doing) that makes this true, and what is the one-line fix shown in the chapter?

## Design question (interview-style)

Your team runs a FastAPI order service behind Gunicorn with 8 `UvicornWorker` processes, each configured to handle up to 100 concurrent async requests, backed by a database connection pool sized at 15 connections per worker. Two weeks ago, someone added a call into a synchronous PDF-generation library inside the `async def generate_invoice` handler, without wrapping it in `run_in_executor`. This week, two things are happening in production:

1. p99 latency has tripled across *every* endpoint on this service — not just `/invoices`.
2. The database's connection limit is periodically exhausted during traffic spikes, even though `8 workers × 15 connections = 120` looks like it should be comfortably within the database's configured max of 200.

Using this chapter's explanation of the FastAPI concurrency model and connection pool sizing, diagnose both incidents. Propose the fix for each, and explain why fixing only the PDF call while leaving the pool size unchanged would still leave the service unsafe under load.

## Coding exercise

The handler below is live in production:

```python
@app.get("/users/{user_id}")
async def get_user(user_id: str, db=Depends(get_db)):
    user = await db.get_user(user_id)
    return user
    # user carries: id, email, password_hash, is_admin, feature_flags, created_at
```

Using the `response_model` allowlist pattern from this chapter's `create_order` example, write a `UserOut` Pydantic model and the corrected route signature so that `password_hash`, `is_admin`, and `feature_flags` can never leak into the API response — including if the underlying ORM model gains new sensitive fields in the future that nobody remembers to explicitly exclude.

## Quiz (self-check)

1. True or false: an `async def` handler in FastAPI is automatically safe from blocking other concurrent requests, regardless of what code runs inside it. *(Explain your answer.)*
2. Which ASGI server does this chapter say FastAPI apps typically run under, and what role does Gunicorn play alongside it?
3. In the testing example, what does `app.dependency_overrides[get_db] = get_test_db` replace at test time, and why does this let `test_create_order_rejects_empty_line_items` run without a real database?
4. What formula does the chapter say a database connection pool must be sized against, beyond just "however many connections seem reasonable"?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
