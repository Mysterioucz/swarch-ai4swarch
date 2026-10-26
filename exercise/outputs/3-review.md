Agent: Claude Code, model claude-sonnet-5-5 (read only `docs/adr/0002-use-rabbitmq-for-post-order-events.md` and `docs/scenario.md`)

# ATAM-style review of ADR-0002

## 1. Quality-attribute scenarios
| # | Source · Stimulus · Environment | Response | Measure |
|---|---|---|---|
| Q1 Performance | Customer · `PlaceOrder` · 200 orders/s, MailFast at p99 8 s | Order completes without waiting for email, loyalty or analytics | p95 ≤ 500 ms |
| Q2 Availability | Operations · RabbitMQ or a consumer is down · normal or peak load | Orders still succeed; no post-order action fails an order | 0 failed orders caused by post-order actions |
| Q3 Correctness | Broker · same `OrderPlaced` delivered twice · any load | Points credited once | Exactly once, within 1 minute of the order |

## 2. Findings
| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | MailFast is removed from the request path | Its p95 of 1.5 s alone exceeds the 500 ms budget |
| Non-risk | Unique key on `order_id` in `loyaltyservice` | Turns at-least-once delivery into an exactly-once effect |
| Sensitivity point | Email send rate (40/s) against 200 orders/s | The email backlog grows about 160/s, so delay depends directly on the length of the peak |
| Sensitivity point | Publish timeout (about 50 ms) | Too long erodes the 500 ms budget; too short drops events |
| Trade-off point | Dropping the event when publish fails | Protects "never fail an order" (availability) but breaks "points within 1 minute, exactly once" (correctness) |
| Trade-off point | Self-hosted RabbitMQ | Fits the company's preference and the team's skill, but adds a stateful component to operate |
| Risk | Publish happens after payment and shipping succeed, and nothing durable backs it | A broker blip at that moment means a paid order with no points and no email, found only by a reconciliation job that is not designed yet |
| Risk | Email backlog at 11.11 | Customers may wait tens of minutes for confirmation |
| Risk | Loyalty consumer throughput is not sized | "Within 1 minute" holds only if the loyalty consumer keeps up at 200 events/s |

## 3. Missing or weak
- **Reconciliation job** is mentioned but has no owner, schedule or source of truth for finding missing orders.
- **Transactional outbox** is only "considered". `checkoutservice` has no database today, so the real fix for lost events is not decided.
- **Email priority and backlog limits** (what to do if the queue is hours long) are not specified.
- **"Replaces `emailservice`"** is stated, but the migration of the existing gRPC `EmailService` contract and its callers is not.
- **Privacy:** the event carries the customer's email address into the broker and into analytics; no retention or access rule is given.
- **The "about 40 minutes" delay** is an estimate for a 10-minute peak. The scenario does not state how long the peak lasts.
- **RabbitMQ sizing and failure modes** (disk, memory alarms, queue-length limits) are missing.

## 4. Verdict
**Accept with changes.** The decision is sound for the stated constraints. The three most important changes:
1. Decide how events survive a broker outage (outbox or durable local buffer) and design the reconciliation job, so "exactly once within 1 minute" is not best-effort.
2. Get an agreed email-delay limit for 11.11, and size or raise MailFast's rate limit accordingly.
3. Size the loyalty consumer for 200 events/s and add queue-depth and age alerts.
