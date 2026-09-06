# Chapter 21: Choosing the Right Protocol — Exercises

*Corresponds to: [21-choosing-the-right-protocol.md](21-choosing-the-right-protocol.md)*

## Concept questions

1. The chapter organizes the synchronous decision into four axes. Name all four, and for the "who is the client, and how many client teams exist?" axis, explain — using the chapter's own contrast — why a public API and a small set of first-party mobile teams lead to different protocol conclusions even though both are "just clients."
2. The quick-reference table lists gRPC's over/under-fetching row as "N/A (defined by proto contract)," distinct from REST's "Common problem" and GraphQL's "Solved by design." Explain why gRPC sidesteps this axis entirely rather than "solving" it the way GraphQL does.
3. The chapter says the mistake to avoid "is not 'using the wrong protocol' in isolation." What more specific mistake does it identify instead, and how do the chapter's two examples (raw gRPC to third-party integrators, a public GraphQL API without cost-limiting discipline) both illustrate it?
4. Using the chapter's caching-story axis, explain why REST's caching is described as "close to free at every layer" while GraphQL's requires "deliberate normalized client-side caching or persisted-query-keyed server caching." What structural property of each protocol's contract causes this difference?
5. The chapter says "the first question is not 'which protocol.'" What question does it put first, what are the three answers it lists, and why does it claim "no amount of REST-vs-gRPC deliberation" fixes the case it has in mind?

## Design question (interview-style)

A mid-sized company currently runs a single REST API that serves everything: their own web app, their own mobile apps, and roughly 40 third-party integration partners. Engineering leadership wants to introduce gRPC and GraphQL and asks you, the newly hired staff engineer, to propose a target architecture.

Using the chapter's four axes, make a specific recommendation for each of the three consumer populations (don't just say "hybrid" — say which protocol goes where and why, grounded in the axes). Then identify which one of the two mistakes the chapter warns against (raw internal-protocol exposure to third parties, or an under-defended public GraphQL surface) this company would be at highest risk of making during the migration, and explain what about their current situation (40 partners, existing REST surface) makes that specific mistake likely if the migration is done carelessly.

## Applied exercise

The chapter's quick-reference table has seven rows (Best client fit, Over/under-fetching, Caching, Type safety, Streaming, Browser-native, Operational complexity). Do two things:

1. **Add a new row** to the table for a concern the chapter discusses but doesn't put in the table: **"Coordinating a breaking schema/contract change with consumers."** Fill in the REST, GraphQL, and gRPC cells, grounding each cell in language from this chapter and Chapter 19 (which axis — client count, coordination ability — determines how costly a breaking change is for each protocol).
2. **Challenge one existing cell.** Pick the "Streaming" row's GraphQL cell ("Subscriptions (heavier-weight)") and argue the nuance that single cell hides: under what circumstances would a team reasonably choose gRPC streaming over GraphQL subscriptions even for a first-party client population that otherwise fits GraphQL well, per this chapter's own axes?

## Quiz (self-check)

1. True or false: the chapter's framework concludes that a mature engineering organization should standardize on a single protocol to minimize operational complexity. *(Explain your answer.)*
2. Which of the chapter's four axes determines whether you can "actually coordinate changes" with your clients?
3. Per the table, which protocol has the lowest operational complexity, and what phrase does the chapter use to describe why?
4. What does the chapter say most real production systems do instead of picking exactly one protocol for everything?

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
