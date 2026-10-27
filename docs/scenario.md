**Language:** [ภาษาไทย](scenario.th.md) | English

# Scenario: what happens after an order is placed

> A **fictional** business scenario for Task 2. The numbers are made up but internally consistent. Treat them as facts.

## The new requirement (R-12)

> After an order is placed, Online Boutique must:
> 1. send the order confirmation email through **MailFast**, a new external email provider, which replaces the mock `emailservice` logic;
> 2. award **loyalty points** through a new `loyaltyservice` (to be built);
> 3. record an **"order placed" analytics event** for the marketing team.
>
> Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).

## How it works today

`checkoutservice.PlaceOrder` calls `paymentservice`, `shippingservice` and then `emailservice` **synchronously over gRPC**, in sequence. If the email call fails, checkout only logs a warning, and the order still succeeds.

## Quality goals and constraints

| Topic | Fact |
|---|---|
| Checkout latency | p95 of `PlaceOrder` must stay **≤ 500 ms** (today approximately 300 ms) |
| Peak load | **200 orders/s** during the 11.11 sale |
| MailFast | p95 **1.5 s**, p99 **8 s**; fails approximately **0.5 %** of calls; rate limit **50 emails/s** |
| Loyalty points | must be awarded **within 1 minute** of the order, **exactly once** (no double points) |
| Availability | an order must **never fail** because email, loyalty or analytics is down |
| Team | 4 developers; strong in Go and gRPC; two have used **RabbitMQ** before |
| Operations | everything already runs on Kubernetes; the company prefers **open-source, self-hosted** components over cloud-only managed services |
