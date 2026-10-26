Agent: Claude Code, model claude-sonnet-5-5 (Run a, zero-shot: only the template and the requirement were read)

# ADR-0002: Decouple post-order actions with an event-driven message broker

- **Status:** Proposed
- **Date:** 2026-10-22
- **Deciders:** Not given (see Open questions)

## Context and problem

After an order is placed, Online Boutique must send a confirmation email through a new external provider (MailFast), award loyalty points through a new `loyaltyservice`, and record an "order placed" analytics event. Marketing expects more post-order actions next year (SMS, fraud scoring, a recommendations update).

Calling each of these synchronously from checkout would couple order placement to every downstream system. A slow or failing provider would slow or fail checkout, and each new action would require changing checkout.

## Decision drivers

- Checkout must stay fast and must not fail because a post-order action fails.
- Adding new post-order actions should be easy.
- Loyalty points should not be awarded twice.
- Operational simplicity.

## Considered options

1. Synchronous calls from checkout to each service
2. Message broker with publish/subscribe (e.g. RabbitMQ or Kafka)
3. Background job or cron that polls the order database

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| Synchronous calls | Simple; easy to debug | Checkout latency is the sum of all calls; downstream failure affects checkout; every new action changes checkout |
| Message broker (pub/sub) | Decoupled; new consumers need no checkout change; buffers load spikes; retries | New infrastructure; eventual consistency; duplicates need idempotent consumers |
| Polling job | No new broker | Delay; load on the database; clumsy to extend |

## Decision

Publish an `OrderPlaced` event to a message broker after the order is saved. Email, loyalty and analytics each subscribe as independent consumers. Consumers must be idempotent. Failed deliveries are retried and sent to a dead-letter queue.

## Consequences

- **Positive:** checkout latency is independent of downstream systems (performance); downstream failures do not fail orders (availability); new consumers can be added without touching checkout (modifiability).
- **Negative:** a broker must be operated (operability, cost); eventual consistency; duplicate delivery must be handled (correctness).
- **Risks and mitigations:** the broker becomes a single point of failure. Run it clustered and use an outbox pattern so events are not lost.

## Validation

Load-test checkout with the broker and with downstream systems failing, and confirm orders still succeed.

## Open questions

- What are the checkout latency targets and the peak order rate?
- What are MailFast's latency, error rate and rate limit?
- What delivery guarantees and time limits apply to loyalty points?
- Which broker technologies does the team know, and are there hosting constraints (self-hosted vs managed)?
- Is there an order database to hold an outbox table?
