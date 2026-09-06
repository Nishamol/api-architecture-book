# Chapter 7: GraphQL Schema Design and Execution Model — Exercises

*Corresponds to: [07-graphql-fundamentals.md](07-graphql-fundamentals.md)*

## Concept questions

1. The chapter's `Order`/`Customer` schema defines the relationship bidirectionally — `Order.customer` and `Customer.orders` both exist. Explain why this bidirectionality is "normal in GraphQL" and, specifically, why it is described as "exactly the kind of relationship that creates the N+1 problem."
2. "In REST, adding a field to a response is close to free... In GraphQL, adding a field to the schema means writing a resolver." Explain what changes structurally between the two protocols that makes this true, and describe the specific risk of adding a *relationship* field (as opposed to a scalar field) to an existing type.
3. Queries are described as "conceptually side-effect-free (though nothing stops a resolver from having side effects — that discipline is convention, not enforcement)." What does this parenthetical imply about how much you should trust a `Query` field's name when reviewing someone else's schema, and how does this contrast with the guarantee Mutations give you by convention?
4. Introspection is called "a double-edged capability." Name the concrete tool/workflow benefit it provides and the concrete production risk it creates, using the chapter's own framing of what an attacker gains from it.
5. The chapter adds interfaces, unions, input types, and custom scalars using the `Node` interface and `SearchResult` union as examples. State the rule for choosing between an interface and a union, and describe what a client has to do differently when querying a field typed as a union compared to one typed as an interface.

## Design question (interview-style)

You've inherited the schema below, currently deployed with introspection enabled and no depth or complexity limiting, serving both your company's mobile app and a small number of external partners over the same public `/graphql` endpoint:

```graphql
type Order {
  id: ID!
  status: OrderStatus!
  customer: Customer!
  lineItems: [LineItem!]!
  total: Money!
}

type Customer {
  id: ID!
  name: String!
  orders: [Order!]!
}

type Query {
  order(id: ID!): Order
  customer(id: ID!): Customer
}
```

Walk through, in order: (1) what an external partner can learn about your entire data model without you telling them anything, using only capabilities this chapter describes; (2) what happens on the server, mechanically, if that partner sends a query that nests `customer { orders { customer { orders { customer { orders { ... } } } } } }` many levels deep; (3) which of this chapter's three listed failure modes are already live risks in this exact schema, and why introspection being on for partners makes the other two easier to trigger, not just correlated with them. You do not need to write code — this is a reasoning exercise about the schema as given.

## Coding exercise

The chapter's `Resolvers: the actual implementation` section distinguishes fields that "often need no explicit resolver (the framework reads the attribute by name)" from relationship fields that "require a resolver that knows how to fetch the related data." Extend the schema above with a new type and field:

```graphql
type Review {
  id: ID!
  rating: Int!
  body: String!
  author: Customer!
}

# Add this field to the existing Order type:
#   reviews: [Review!]!
```

Write out the full updated `Order` type SDL with `reviews: [Review!]!` added, plus the `Review` type. Then, for each field across `Order`, `Customer`, and your new `Review` type, state in one line whether it can be resolved automatically or requires an explicit resolver, and justify `Review.author` specifically by name-checking it against the chapter's distinction. Finally, name the new cyclic path this addition opens up (comparable to the `customer → orders → customer → orders` path already named in the chapter) and which failure mode it falls under.

## Quiz (self-check)

1. True or false: a scalar field like `Order.status` never needs a resolver function written for it. *(Explain your answer — note what "never" would require ignoring.)*
2. What single GraphQL query name lets a client fetch the entire schema's types and fields programmatically?
3. Name the chapter's failure mode that occurs when a query is allowed to traverse a relationship cycle with no maximum nesting enforced.
4. In the `cancelOrder` mutation example, what does the mutation return, and what convention does this follow that saves the client a follow-up request?
5. True or false: a custom scalar like `DateTime` requires no special serialization logic beyond what the five built-in scalars already provide. *(Explain your answer.)*

---

Solutions to these exercises are available separately in [resources/solutions/](../resources/solutions/).
