# ADR-0002: Use RabbitMQ events for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-22
- **Deciders:** Architect (author), to be reviewed by the architecture review board

> Drafted from `exercise/outputs/2b-adr.md` and then edited by the author. See the change table in `exercise/02-write-adr.en.md`.

## Context and problem

Requirement R-12 adds three post-order actions: a confirmation email through MailFast, loyalty points through a new `loyaltyservice`, and an "order placed" analytics event. Marketing expects more actions next year (SMS, fraud scoring, recommendations update).

Today `checkoutservice.PlaceOrder` calls payment and shipping, then `emailservice`, synchronously over gRPC. An email failure is only logged (`log.Warnf` in `main.go`), so the order still succeeds.

Facts (from `docs/scenario.md`):
- `PlaceOrder` p95 must stay ≤ 500 ms; today it is about 300 ms.
- Peak is 200 orders/s (11.11 sale).
- MailFast: p95 1.5 s, p99 8 s, about 0.5 % failures, rate limit 50 emails/s.
- Loyalty points: within 1 minute, exactly once.
- An order must never fail because email, loyalty or analytics is down.
- Team of 4, strong in Go and gRPC, two know RabbitMQ. Kubernetes already in use; open-source, self-hosted preferred.

A synchronous MailFast call (p95 1.5 s) cannot fit in a 500 ms budget. At peak, orders arrive at four times MailFast's rate limit, so some buffering is unavoidable.

## Decision drivers

1. **Availability:** a post-order failure never fails an order.
2. **Performance:** `PlaceOrder` p95 ≤ 500 ms at 200 orders/s.
3. **Correctness:** loyalty points exactly once, within 1 minute.
4. **Modifiability:** new post-order actions without changing checkout.
5. **Operability and cost:** open source, self-hosted, within the team's skills.

## Considered options

1. Synchronous gRPC calls from checkout to each consumer
2. Checkout publishes an `OrderPlaced` event to RabbitMQ; consumers handle email, loyalty and analytics
3. In-process goroutines in `checkoutservice`
4. Apache Kafka as the event log

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Sync gRPC | Matches today's code; simplest | MailFast p95 1.5 s breaks the 500 ms budget; every new action changes checkout and adds latency |
| 2. RabbitMQ | Takes slow calls out of the request path; one durable queue per consumer; absorbs the 200 vs 50/s gap; retry and dead-letter built in; team knowledge; self-hosted | New stateful component; at-least-once delivery means duplicates; email backlog at peak; event lost if publish fails |
| 3. Goroutines | No new infrastructure | Events lost on pod restart; no cross-replica rate limit; cannot guarantee points |
| 4. Kafka | Replayable log; high throughput | Heavier to run than 200 events/s needs; no team experience stated; per-message retry and rate-limited consumption are awkward |

## Decision

Use **Option 2: RabbitMQ**. After payment and shipping succeed, `checkoutservice` publishes one `OrderPlaced` event (order ID, user ID, email, items) to a topic exchange using publisher confirms and a short timeout (about 50 ms, to be tuned in the load test). A publish failure is logged and counted, and does not fail the order.

Three durable queues, each with a dead-letter queue:
- **email:** a worker replaces `emailservice`'s logic and calls MailFast, rate-limited to 40 emails/s (below the 50/s limit), retrying with backoff.
- **loyalty:** `loyaltyservice` credits points with a unique key on `order_id`, so redelivery cannot double-award.
- **analytics:** a consumer stores the event.

RabbitMQ runs in Kubernetes as a 3-node cluster with quorum queues. New actions become new queues.

## Consequences

- **Positive:** `PlaceOrder` no longer waits for MailFast, loyalty or analytics, so p95 stays close to 300 ms (performance).
- **Positive:** a consumer outage only grows its queue; orders continue (availability).
- **Positive:** the queue absorbs the peak against MailFast's rate limit (performance, cost).
- **Positive:** new post-order actions are new queues, with no checkout change (modifiability).
- **Negative:** at 200 orders/s against 40 emails/s the email backlog grows by about 160/s. A 10-minute peak would delay the last emails by about 40 minutes (usability).
- **Negative:** RabbitMQ is a new stateful component to run, monitor and upgrade (operability).
- **Negative:** duplicates are possible, so all consumers must be idempotent (correctness).
- **Risks and mitigations:**
  - *Event lost if RabbitMQ is unreachable at publish time:* retry once, alert on publish failures, and run a reconciliation job that finds orders without a loyalty record. Consider a transactional outbox if checkout gets a database.
  - *Email backlog during 11.11:* confirm with the product owner what delay is acceptable. Prioritise email over analytics, and consider a second MailFast plan or a higher rate limit.
  - *Broker single point of failure:* clustered quorum queues, plus a memory/disk alarm on queue depth.

## Validation

- Load test at 200 orders/s with MailFast simulated at p95 1.5 s / p99 8 s: `PlaceOrder` p95 ≤ 500 ms.
- Stop RabbitMQ, then each consumer, in turn: orders still succeed.
- Deliver the same `OrderPlaced` event twice: points are credited once.
- At peak: loyalty points appear within 1 minute, and the MailFast send rate never exceeds 50/s.
- Review after the 11.11 sale: peak queue depth and oldest email age.
