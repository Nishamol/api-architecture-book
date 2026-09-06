# Chapter 13: Authentication and Authorization — Solutions

*Corresponds to: [part5-cross-cutting/13a-authentication-authorization-exercises.md](../../part5-cross-cutting/13a-authentication-authorization-exercises.md)*

## Concept questions — Model answers

1. A concrete scenario: an `orders` service checks `Authorization: Bearer <token>`, verifies the JWT signature, and — because the token is valid — lets the request through to the handler for `POST /orders/{order_id}/cancel`. The handler cancels whatever order ID is in the URL, without checking whether the authenticated caller (`sub` claim, say `user_42`) actually owns `order_id`. Any authenticated user can now cancel any other user's order simply by changing the URL, because the only check performed was "does this caller have a valid token," which answers authentication, not "is *this* caller allowed to cancel *this* order," which is authorization. The token being valid says nothing about the relationship between the caller and the specific resource — that check has to be implemented separately, and the chapter is explicit that skipping it is a recurring source of exactly this bug.

2. A per-user JWT answers "which end user, somewhere upstream, originally initiated this chain of calls." A service-to-service mTLS certificate answers "which service is making this specific network connection to me right now." For an internal call from `billing` to `inventory`, the question that actually matters for most authorization decisions at that hop is the second one — is this connection really coming from the `billing` service, not an attacker who's landed inside the network perimeter — not which end user's request triggered it (though that information may still need to be *propagated*, just not as the primary credential). mTLS is the right primary credential because both sides present and verify certificates as part of establishing the connection itself, authenticating the service identity independent of any per-request token; a per-user JWT would be the wrong primary credential here because it identifies a person, not the calling service, and services don't inherently have "a user" the way an external API caller does.

3. RBAC checks a role against an action and is described as "simple to reason about, but coarse-grained and prone to role sprawl" — it has no natural way to express "edit access flows from the document owner through team membership," short of minting a combinatorial explosion of roles per document/team pairing. ABAC evaluates a policy against attributes of caller, resource, and context — closer, since you could encode "caller.team == resource.grantedTeam," but the chapter notes it's "harder to audit and reason about at a glance," and it still doesn't model the graph structure (team membership, ownership, delegated grants) directly — it evaluates attributes, not relationships. ReBAC is the fit: it "models authorization as relationships in a graph," which is exactly the shape of the requirement — user X is related to team Z via membership, team Z is related to document Y via a grant from the owner, and the permission check is a graph traversal across those relationship edges. RBAC and ABAC strain because the requirement is fundamentally about chained relationships (membership → grant → permission), not a static role assignment or an attribute match, which is precisely the case the chapter says ReBAC (and tools like OpenFGA/SpiceDB) are suited for, at the cost of standing up a dedicated authorization service.

4. If authorization is checked once at the top — "does this caller have permission to run this query at all" — before `order(id: "42")` resolves, that check has no visibility into which fields the query will eventually touch. A query requesting `order(id: "42") { status internalNotes }` would pass that single top-level check (the caller is allowed to query orders in general) and then the `internalNotes` resolver would return its value to a caller who was never authorized to see admin-only notes, because nothing re-checked authorization at the point where that specific field — annotated `@auth(requires: ADMIN)` in the chapter's example — was resolved. This is exactly the "Field-level authorization skipped in GraphQL" failure mode the chapter names: a query-level check doesn't account for a field deep in the response exposing data the caller shouldn't see, because GraphQL's single endpoint means the sensitive boundary is the field, not the query.

5. The raw key is hashed before storage so that a database leak hands out useless hashes rather than working credentials — the same principle as password storage: anyone with read access to the database (an attacker, an over-privileged internal query, a backup snapshot) can't reconstruct a usable key from a hash, whereas a stored plaintext key would let them impersonate every seller immediately. API keys are described as simpler than a JWT for a third-party seller integration because there's no interactive login flow to build or maintain, no token refresh cycle the integration has to implement, and no per-end-user identity to assert at all — the seller's account itself is the caller, so a single static, long-lived credential exchanged once at integration time is sufficient, unlike OIDC's flow which is designed around authenticating an individual end user.

## Design question — Model answer

1. **Vulnerability class**: the chapter names this exactly — Broken Object Level Authorization (BOLA), which OWASP ranks as "consistently the #1 API security risk in their API Security Top 10." A valid JWT only proves authentication (this caller is who they claim to be, per the identity provider) — it says nothing about the relationship between that caller and the specific `order_id` in the URL. Without an explicit ownership check, "has a valid token" and "is allowed to see this order" are being treated as equivalent, which is precisely the conflation the chapter opens by warning against.

2. **REST fix**: the dependency/middleware layer currently checks "is there a valid, signed, non-expired token for `orders-api`" and stops there. It needs an additional check, after token validation and before the handler runs: load the order's owning seller ID (from the orders table/service) and compare it against the `sub`/`seller_id` claim in the verified token, rejecting with `403` if they don't match. That ownership data has to come from the resource itself — a lookup against the order record's `seller_id` column, not from anything in the token, since the token only proves identity, not the current ownership relationship, which can change independently of the token's contents.

3. **GraphQL enforcement point**: the chapter's recommended point is per-field authorization directives (`@auth(...)`), evaluated at resolution time. The GraphQL path has the *same* gap as REST if the `order(id: ID!)` resolver fetches and returns the order based solely on "caller is authenticated" without the resolver itself checking `order.sellerId == caller.sellerId` — i.e., if the field-level directive pattern is used for coarse checks like "is this field admin-only" but nobody wired an equivalent object-ownership check into the `order` resolver specifically. Directives protect field *visibility* by role/claim; they don't automatically protect object *ownership* unless someone writes that check into the resolver. **gRPC enforcement point**: the chapter's recommended point is interceptors, running before the servicer method executes. The gRPC path has the same gap if the interceptor only validates the JWT (as in the coding exercise) and populates caller identity, but the `GetOrder` servicer method itself never compares the caller's identity against the order's owning seller — the interceptor authenticates, the servicer has to authorize, and if that second step is missing, gRPC has exactly REST's bug wearing a different transport.

4. **ReBAC**, using an ownership relationship (`seller X owns order Y`) as the authorization graph edge, checked at every enforcement point (REST middleware, GraphQL resolver, gRPC interceptor/servicer) via a shared authorization service call rather than three independently-implemented ownership checks. This is preferable to RBAC, which would require a role explosion ("seller-of-order-42") to express per-resource ownership and is explicitly called out as prone to this kind of sprawl. It's also preferable to a hand-rolled ABAC policy for this specific case, because the relationship being modeled — direct ownership, potentially later extended to "seller's employee" or "delegated access" — is naturally graph-shaped, and centralizing it in one authorization service (OpenFGA/SpiceDB, per the chapter) guarantees the REST, GraphQL, and gRPC paths all consult the *same* ownership check instead of three teams independently re-implementing (and potentially getting wrong) the same logic.

## Coding exercise — Model answer

The interceptor violates the chapter's JWT validation guidance in two distinct ways:

1. **`verify_signature: False`** — this is the exact mistake the chapter calls out: "a common vulnerability is validating the signature but skipping expiry or audience checks," except this code goes further and skips signature verification entirely. Any caller can forge a token with an arbitrary `sub` claim (e.g., `sub: "admin"` or another user's ID) and the interceptor will accept it and propagate that forged identity downstream as `caller_sub` — full authentication bypass.
2. **No `exp`, `iss`, or `aud` checks** — even if signature verification were turned on, the chapter is explicit that "the token's `exp`, `iss`, and `aud` claims must be explicitly checked," warning specifically that skipping audience checks "allow[s] a token issued for a different service to be replayed against this one." As written, a token issued for an entirely unrelated service (or an expired token) would decode successfully and be trusted.

Fixed version, using the chapter's `PyJWKClient` pattern:

```python
import grpc
from jwt import PyJWKClient, PyJWTError

jwks_client = PyJWKClient("https://auth.example.com/.well-known/jwks.json")

class AuthInterceptor(grpc.aio.ServerInterceptor):
    async def intercept_service(self, continuation, handler_call_details):
        metadata = dict(handler_call_details.invocation_metadata)
        token = metadata.get("authorization", "").removeprefix("Bearer ")

        try:
            signing_key = jwks_client.get_signing_key_from_jwt(token)
            claims = jwt.decode(
                token,
                signing_key.key,
                algorithms=["RS256"],
                audience="orders-api",
                issuer="https://auth.example.com/",
            )
        except PyJWTError:
            async def deny(request, context):
                await context.abort(grpc.StatusCode.UNAUTHENTICATED, "invalid token")
            return grpc.unary_unary_rpc_method_handler(deny)

        handler_call_details.invocation_metadata = metadata | {"caller_sub": claims["sub"]}
        return await continuation(handler_call_details)
```

This restores signature verification against the JWKS-fetched key (rather than a hardcoded key — the chapter notes identity providers rotate signing keys, so `PyJWKClient` fetching and caching the JWKS handles rotation automatically) and adds explicit `audience`/`issuer` checks so a token minted for a different service is rejected rather than replayed successfully against `orders-api`. `jwt.decode` also validates `exp` by default when a real signing key and algorithm are supplied, closing the third gap. Note this interceptor still only authenticates — per the design question above, the servicer method still needs its own object-level ownership check; authentication and authorization remain separate layers even after this fix.

## Quiz (self-check) — Answers

1. **False.** The chapter is explicit that claims "should never be trusted until the signature is verified against the issuer's public key," and even after signature verification, `exp`, `iss`, and `aud` must be explicitly checked — a validly-signed token can still be expired or issued for a different audience, and trusting it anyway (e.g., replaying a token meant for another service) is the specific vulnerability the chapter warns about.
2. **Broken Object Level Authorization (BOLA)** — the chapter states OWASP ranks it as "consistently the #1 API security risk in their API Security Top 10."
3. Identity providers **rotate signing keys periodically**, and a hardcoded key becomes stale the moment rotation happens, turning what looked like a one-time setup step into **an outage** once the old key is no longer valid — which is why the chapter recommends fetching and caching the JWKS instead.
4. **gRPC** — authentication rides on metadata (`metadata = [("authorization", f"Bearer {access_token}")]`) rather than an HTTP header, but the chapter treats it as the same mechanism because metadata is "the RPC equivalent of headers" — same bearer-token pattern, different transport-level container.
5. **False.** The chapter explicitly calls for treating rotation as a first-class feature — issuing a new key with an overlapping grace period on the old one — rather than a single permanent credential nobody can ever safely change. Treating an API key as non-rotatable is exactly the anti-pattern the chapter warns against: without a rotation mechanism, a seller (or the platform) can never safely respond to a compromised key without an uncoordinated, breaking cutover.
