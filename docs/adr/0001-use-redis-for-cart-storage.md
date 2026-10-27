# ADR-0001: Use Redis for cart storage

- **Status:** Accepted
- **Date:** 2026-10-01
- **Deciders:** Course teaching team

> **Example ADR for this exercise.** It was written by the teaching team, using what the Online Boutique code shows. It is not a decision record from Online Boutique's original authors. Use it as an example of the expected style and depth.

## Context and problem

`cartservice` (C#) keeps each user's shopping cart between requests. A cart is small (a list of product IDs and quantities), keyed by the user's session ID, and read on almost every page view through `frontend` and again by `checkoutservice` at checkout. Carts are short-lived: losing one is annoying but not a financial loss, because no money has changed hands.

`cartservice` runs as several stateless replicas on Kubernetes, so a cart cannot live in a single replica's memory.

## Decision drivers

1. **Performance:** cart reads happen on almost every page; they must add only a few milliseconds.
2. **Scalability:** `cartservice` must scale horizontally, so its replicas must share cart state.
3. **Simplicity and cost:** the demo should run on any Kubernetes cluster without a managed cloud database.
4. **Durability:** nice to have, not critical (no money is involved until checkout).

## Considered options

1. In-memory store inside each `cartservice` replica
2. Redis (in-cluster), accessed through a cache abstraction
3. A managed relational/distributed database (Cloud Spanner or AlloyDB)

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| In-memory per replica | Fastest; no extra container | State lost on restart; replicas don't share carts, so it can't scale out |
| Redis | Sub-millisecond key lookups; shared by all replicas; runs anywhere as one container | Another component to operate; default setup has weak durability |
| Spanner / AlloyDB | Durable, strongly consistent, managed | Cloud-specific; higher cost and latency than needed for cart data |

## Decision

Use **Redis** as the default cart store (`REDIS_ADDR`, implemented by `RedisCartStore`), behind the `ICartStore` interface so the store can be swapped. Keep Spanner and AlloyDB implementations as optional alternatives for deployments that need durability.

## Consequences

- **Positive:** fast cart reads and writes (performance); stateless `cartservice` replicas can scale out (scalability); runs on any cluster (portability).
- **Positive:** `ICartStore` keeps the storage choice reversible (modifiability).
- **Negative:** one more stateful container to run; carts may be lost if Redis restarts without persistence (durability).
- **Risks and mitigations:** Redis is a single point of failure for the cart. Mitigate with Redis replication in production, or switch to the Spanner/AlloyDB store.

## Validation

Load-test the cart endpoints (e.g. with the Locust `loadgenerator`) and check p95 cart latency stays under 10 ms at expected peak traffic.
