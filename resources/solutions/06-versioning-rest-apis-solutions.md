# Chapter 6: Versioning REST APIs — Solutions

*Corresponds to: [part2-rest/06a-versioning-rest-apis-exercises.md](../../part2-rest/06a-versioning-rest-apis-exercises.md)*

## Concept questions — Model answers

1. URI versioning puts the version directly in the URL (`/v1/orders`, `/v2/orders`), and CDNs and browsers key their caches on the URL by default — so a versioned URL is automatically a distinct, independently cacheable cache key with no extra configuration needed anywhere in the request path. Header versioning keeps the URL identical across versions (`/orders` for both `v1` and `v2`, distinguished by an `Accept` or custom header), which means the cache key most CDNs compute by default — based on the URL alone — is the same for both versions, so a cached `v1` response can get served to a `v2` request unless the CDN is explicitly configured to vary its cache key on that header, which the chapter notes "most CDNs don't ... by default." The caching behavior isn't a minor detail here — it's the direct, mechanical consequence of what each strategy puts into the URL versus what it hides in a header.
2. Additive-only, no-versioning works by discipline: you promise never to remove or repurpose a field, and you treat any genuinely breaking change as an entirely new resource. That discipline is sustainable when you have "a small, coordinated set of consumers" — you can literally talk to every team that calls your API and agree on migration timing together. Once you have external integrators you don't control and can't schedule with, two things break down: first, you lose the ability to *ever* clean up a field or fix a bad early design decision, because "breaking change → new resource" without any version marker means every historical design mistake has to live forever as its own permanently-maintained resource; second, you have no mechanism at all for the case where a change is unavoidably breaking (fixing a genuine correctness bug in a field's meaning, for instance) — additive-only simply doesn't have an answer for that case once "just tell the other team" isn't a realistic option, which is exactly the situation URI versioning is designed to handle.
3. The four elements: a **`Sunset` HTTP header** (RFC 8594) so automated tooling can detect the deprecation programmatically; a **`Deprecation` header** pointing at migration docs; **telemetry** on which specific clients (by API key or user agent) are still calling the old version; and a **hard sunset date communicated well in advance with actual outreach to the top callers by volume**. A changelog entry fails to move traffic because it's passive — it requires someone on the client side to be reading changelogs and to correctly connect "this note" to "this specific endpoint my service calls," which doesn't happen reliably at scale. The four elements above replace hope with mechanism: automated header detection means client tooling *can* flag it without a human reading anything, telemetry means you know exactly who's still on the old version rather than guessing, and direct outreach to top-volume callers means the highest-impact migrations get a human push rather than waiting for the changelog to be noticed.
4. Consumer-driven contract testing works by having each consuming service publish, in a machine-checkable form, the exact subset of the contract it actually depends on — and having the provider's CI run against every published contract before any deploy. That mechanism inherently requires consumers to be participants in your development process: they have to actually write and publish a Pact contract, and your CI has to have access to run against it. Internal consumers can be required to do this, since they're part of the same organization and can be mandated to adopt the tooling. Third-party or external integrators can't be required to do anything — you have no lever to force a random third-party seller to publish a Pact contract before you're allowed to change your API — so this technique simply has no consumer contracts to check against for that population, and versioning discipline for external consumers has to fall back on the URI-versioning-plus-deprecation-project approach instead.

## Design question — Model answer

**What ships as additive-only to `/v1` first:** add the new `subtotal` and `total_with_tax` fields to the `/v1/orders/{order_id}` response *alongside* the existing `total` field, without touching or removing `total`. Per the chapter's backward-compatible change categories, "adding a new optional field" is explicitly safe to ship without a version bump — this lets every client start reading the unambiguous fields immediately, on their own schedule, without anything breaking for clients still reading `total`.

**What requires `/v2`, and why:** actually **removing** `total` requires a version bump, because "removing or renaming a field" is explicitly listed under changes that require a version bump or a new endpoint — this is a hard breaking change for any client still reading `total`, and there's no way to make field removal additive. So the sequence is: `/v1` gets the additive fields now; `/v2/orders/{order_id}` is introduced as the endpoint that omits `total` entirely (or `/v1` eventually drops `total` only after its own deprecation window closes — functionally the same commitment, framed either as a resource without the field or as a version bump, but the chapter's guidance to combine URI versioning at the major-version level with additive-only changes within a version points toward treating this as the trigger for a `/v2` cut).

**Headers and dates on the deprecated path:** once `total` is slated for removal, the `/v1` (or whichever path still serves `total`) handler needs the full deprecation project from the chapter — a `Deprecation: true` header, a `Sunset` header (RFC 8594) carrying the actual sunset date set relative to the 6-month window, and a `Link` header pointing at migration documentation explaining the `subtotal`/`total_with_tax` split, mirroring the chapter's own `/v1/orders/{order_id}` example.

**Sequencing outreach:** the 12 callers responsible for 90% of volume get direct, individual outreach — account manager or developer-relations contact, explicit confirmation they've migrated before the sunset date is treated as final — because a single one of them failing to migrate breaks the overwhelming majority of real traffic on sunset day. The remaining 328 get the automated path: `Sunset`/`Deprecation` headers so their own tooling can flag it, migration docs linked via the `Link` header, and telemetry (by API key) tracking who's still calling the old field so any long-tail caller still active close to the sunset date can get an escalated, targeted reminder rather than being silently cut off.

## Coding exercise — Model answer

**Why this can't ship as additive to `/v1`:** the change removes the `total` field (`del order["total"]`) and repurposes its meaning into two differently-named fields. Per the chapter's categories, "removing or renaming a field" and "changing the meaning of an existing field" are both explicitly listed as requiring a version bump or a new endpoint — this isn't an "add something new, leave the rest alone" change, it eliminates something existing clients are actively reading. Any `/v1` client still parsing `order["total"]` would break outright the moment this shipped in place of the existing `/v1` handler, which is exactly the class of change the chapter says additive-only strategies can't accommodate.

**Corrected `/v1` handler**, implementing the actual deprecation project rather than a stale comment:

```python
from fastapi import FastAPI, Response
from datetime import datetime, timezone

app = FastAPI()

@app.get("/v1/orders/{order_id}", deprecated=True)
async def get_order_v1(order_id: str, response: Response):
    response.headers["Deprecation"] = "true"
    response.headers["Sunset"] = "Wed, 01 Apr 2026 00:00:00 GMT"
    response.headers["Link"] = '</docs/migration/v1-to-v2-total-split>; rel="deprecation"'
    return await fetch_order_v1_shape(order_id)


@app.get("/v2/orders/{order_id}")
async def get_order_v2(order_id: str):
    order = await fetch_order_v1_shape(order_id)
    return {
        **{k: v for k, v in order.items() if k != "total"},
        "subtotal": order["total"] - order["tax"],
        "total_with_tax": order["total"],
    }
```

`/v1` keeps returning the original shape (including `total`) unchanged, but now carries the `Deprecation`, `Sunset`, and `Link` headers so automated client tooling and human integrators alike have a concrete signal and deadline — reusing the exact sunset date from the chapter's own example. `/v2` is the new endpoint that implements the breaking `total` → `subtotal`/`total_with_tax` split cleanly, with no field ambiguity, as its own versioned contract.

## Quiz (self-check) — Answers

1. **False.** It's only safe "*if* clients are contractually required to handle unknown values gracefully" — and the chapter immediately notes that's "a big 'if' — most clients don't, in practice, which is why even this is risky without an explicit contract." Without that explicit contractual guarantee, a new enum value can break clients with a hardcoded `switch`/`match` over the previously-known values.
2. **RFC 8594.**
3. URI versioning's downside: it "implies the entire API version lockstep, when in practice most changes only affect one resource." Header versioning's downside: it "breaks naive HTTP caching" since "most CDNs don't vary cache keys on arbitrary headers by default."
4. **Pact.**
