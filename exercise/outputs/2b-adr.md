Agent: Claude Code, model claude-sonnet-5-5 (Run b: template, `docs/scenario.md`, ADR-0001 and `checkoutservice/main.go` were read)

# ADR-0002: Publish an OrderPlaced event to RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-22
- **Deciders:** Course teaching team (to be confirmed)

## Context and problem

Requirement R-12 adds three post-order actions: confirmation email via MailFast, loyalty points via a new `loyaltyservice`, and an analytics event. More actions (SMS, fraud scoring, recommendation updates) are expected next year.

Today `checkoutservice.PlaceOrder` calls payment, shipping and then `emailservice` synchronously over gRPC. An email failure is only logged (`log.Warnf` in `main.go`), so the order still succeeds.

Facts from the scenario:
- p95 of `PlaceOrder` must stay ≤ 500 ms (today about 300 ms), leaving about 200 ms of headroom.
- Peak is 200 orders/s (11.11 sale).
- MailFast has p95 1.5 s, p99 8 s, about 0.5 % failures and a rate limit of 50 emails/s.
- Loyalty points must be awarded within 1 minute and exactly once.
- An order must never fail because email, loyalty or analytics is down.
- The team has 4 developers (strong in Go and gRPC, two have used RabbitMQ). Everything runs on Kubernetes and the company prefers open-source, self-hosted components.

Calling MailFast synchronously would put a 1.5 s p95 call inside a 500 ms budget, so it cannot stay in the request path. Peak orders (200/s) are four times MailFast's limit (50/s).

## Decision drivers

1. **Availability:** post-order failures never fail an order.
2. **Performance:** `PlaceOrder` p95 ≤ 500 ms.
3. **Correctness:** loyalty points exactly once, within 1 minute.
4. **Modifiability:** new post-order actions without changing checkout.
5. **Operability and cost:** self-hosted, open source, within the team's skills.

## Considered options

1. Keep synchronous gRPC calls from checkout to each consumer
2. Checkout publishes an `OrderPlaced` event to RabbitMQ; email, loyalty and analytics consume it independently
3. Checkout fires asynchronous goroutines (in-process) for each action
4. Apache Kafka as the event log

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Sync gRPC | Matches today's code; simple | MailFast p95 1.5 s breaks the 500 ms budget; checkout would need to ignore errors for every new service; each new action changes checkout |
| 2. RabbitMQ pub/sub | Removes slow calls from checkout; one queue per consumer; buffers the 200 → 50/s email gap; retries and dead-letter queues; two team members know it; open source | One more stateful component to run; at-least-once delivery means duplicates; the email backlog grows during the peak |
| 3. In-process goroutines | No new infrastructure | Events lost if the pod dies; no rate limiting or retry across replicas; no loyalty guarantee |
| 4. Kafka | Replayable log; high throughput | Heavier to operate than the load needs; no team experience stated; per-message retry and rate-limited consumption are harder |

## Decision

Option 2. After payment and shipping succeed, `checkoutservice` publishes one `OrderPlaced` event (order ID, user, email, items) to a RabbitMQ topic exchange, with a short publish timeout. Publish failure is logged and does not fail the order. Three durable queues are bound to the exchange:

- **email queue:** an email worker calls MailFast, limited to below 50 emails/s, retries failures with backoff, and dead-letters after the final retry. `emailservice` is replaced by this worker.
- **loyalty queue:** `loyaltyservice` awards points and de-duplicates on `order_id` (unique key), which gives exactly-once effect over at-least-once delivery.
- **analytics queue:** an analytics consumer writes the event.

New actions (SMS, fraud scoring) are added as new queues without changing checkout.

## Consequences

- **Positive:** `PlaceOrder` no longer waits for MailFast, loyalty or analytics, which keeps p95 near today's 300 ms (performance).
- **Positive:** an outage of any consumer only grows its queue; orders continue (availability).
- **Positive:** the broker absorbs the 200/s peak against MailFast's 50/s limit (performance, cost).
- **Positive:** new consumers need no checkout change (modifiability).
- **Negative:** emails and points are eventually consistent. During the peak the email backlog grows by about 150/s, so a sale of several minutes delays emails by minutes (usability).
- **Negative:** RabbitMQ is a new stateful component to run and monitor (operability).
- **Negative:** duplicates are possible, so every consumer must be idempotent (correctness).
- **Risks and mitigations:** if the broker is down when an order is placed, the event is lost. Mitigate by retrying the publish, with an alert; consider a transactional outbox if checkout gets a database. The broker is a single point of failure: run a clustered RabbitMQ with quorum queues.

## Validation

- Load test at 200 orders/s: `PlaceOrder` p95 ≤ 500 ms with MailFast simulated at p95 1.5 s / p99 8 s.
- Stop RabbitMQ and each consumer in turn: orders still succeed.
- Replay the same event twice: points are awarded once.
- Loyalty points appear within 1 minute at peak; email send rate stays ≤ 50/s.

## Open questions

- Is a sale lasting several minutes acceptable for email delay, or is there an email SLA?
- Which database does `loyaltyservice` use?
- What is the analytics sink?
- Is the order ID the right de-duplication key if checkout retries a request?
