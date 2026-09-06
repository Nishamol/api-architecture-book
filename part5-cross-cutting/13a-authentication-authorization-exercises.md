# Chapter 13: Authentication and Authorization — Exercises

*Corresponds to: [13-authentication-authorization.md](13-authentication-authorization.md)*

## Concept questions

1. The chapter distinguishes authentication ("who is this caller?") from authorization ("is this caller allowed to do this specific thing?") and calls conflating them "a recurring source of privilege-escalation bugs." Describe a concrete scenario where a service correctly authenticates a caller but still has a privilege-escalation bug because authorization was implemented as "has a valid token" rather than as a separate check.
2. The chapter says mTLS "authenticates the *service*, not a per-user token, which is the right model for service-to-service calls." Explain why a per-user JWT is the wrong primary credential for a gRPC call between two internal services, and why mTLS is the right one — what question does each credential actually answer?
3. Compare the three authorization models (RBAC, ABAC, ReBAC) using the chapter's own tradeoffs. If a document-sharing system needs to express "user X can edit document Y because X is on team Z, and team Z was granted edit access by the document's owner," which model fits, and why do the other two strain under this requirement?
4. The chapter states that GraphQL authorization "needs to happen *per field*, not just per query." Using the `Order` type's `internalNotes` field as the example, explain what goes wrong if a GraphQL server only checks authorization once, at the top of query execution, before resolving `order(id: "42")`.
5. The chapter adds API keys as an authentication mechanism alongside OAuth2/OIDC and mTLS. Using the `issue_api_key`/`authenticate_api_key` example, explain why the raw key is hashed before storage rather than stored as-is, and why API keys are described as simpler than a JWT specifically for a third-party seller integration.

## Design question (scenario-style)

You're the lead engineer on an orders platform exposing all three protocols: gRPC internally between `orders`, `billing`, and `shipping` services; a GraphQL BFF for the mobile app; and a REST API for third-party sellers. A security audit flags the following finding:

> "The REST `GET /v1/orders/{order_id}` endpoint checks that the caller has a valid access token, but any authenticated seller can fetch any other seller's order by guessing or incrementing `order_id`. We also could not confirm whether the GraphQL and gRPC paths to the same underlying order data have equivalent protection."

Using the chapter's material on JWT validation, authorization models, and the per-protocol authorization enforcement points, do the following:

1. Name the specific vulnerability class the audit identified (the chapter gives you the exact term) and explain, referencing what JWT validation alone provides, why "has a valid token" was insufficient to prevent it.
2. Design the fix for the REST endpoint — what does the middleware/dependency layer need to check that it currently doesn't, and where does the data it needs (which seller owns which order) come from?
3. The audit's second concern is about the GraphQL and gRPC paths to the same data. For each, identify the specific enforcement point the chapter recommends and explain what would have to be true for that path to have the *same* gap as the REST endpoint despite using a different mechanism.
4. Propose one authorization model (RBAC, ABAC, or ReBAC) for enforcing "seller can only access their own orders" going forward across all three protocols, and justify the choice against the other two using the chapter's tradeoffs.

## Coding exercise

The following gRPC interceptor is meant to authenticate calls to an internal `OrderService` using JWTs, following the chapter's JWT validation guidance:

```python
import jwt

class AuthInterceptor(grpc.aio.ServerInterceptor):
    async def intercept_service(self, continuation, handler_call_details):
        metadata = dict(handler_call_details.invocation_metadata)
        token = metadata.get("authorization", "").removeprefix("Bearer ")

        try:
            claims = jwt.decode(token, options={"verify_signature": False})
        except jwt.PyJWTError:
            async def deny(request, context):
                await context.abort(grpc.StatusCode.UNAUTHENTICATED, "invalid token")
            return grpc.unary_unary_rpc_method_handler(deny)

        handler_call_details.invocation_metadata = metadata | {"caller_sub": claims["sub"]}
        return await continuation(handler_call_details)
```

Identify every way this code violates the JWT validation guidance in this chapter (there is more than one), explain the concrete exploit each violation enables, and rewrite the interceptor to fix them using the `PyJWKClient` pattern shown in the chapter.

## Quiz (self-check)

1. True or false: once a JWT's signature is verified, it's safe to trust every claim inside it for authorization decisions. *(Explain your answer.)*
2. What does OWASP call the authorization gap where a caller is authenticated but not checked for ownership of the specific resource being accessed, and where does the chapter rank it among API security risks?
3. Name the two identity-provider-side behaviors that make hardcoding a JWT signing key a latent outage rather than a one-time setup step.
4. Which of the three protocols in this book has authentication expressed as HTTP metadata rather than an HTTP header, and why does the chapter treat this as functionally the same mechanism rather than a different one?
5. True or false: an API key should be a permanent, non-rotatable credential once issued, since rotating it would break the seller's integration. *(Explain your answer.)*

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
