**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement (approximately 20 min)

> **Lecture link:** T3 (GPT-4 zero-shot was coherent but *less complete than humans*; few-shot closed much of the gap), AICH3 (context), W10 (message brokers).
> **Question:** how much does an AI-written ADR improve when you give it an example and the real context?

Read [`docs/scenario.md`](../docs/scenario.md) first: requirement **R-12**, what happens after an order is placed.

For each run, start a **new chat or session** and send the message below. Replace `<paste R-12 here>` with the quoted R-12 block from the scenario.

**Run a: zero-shot** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
```

**Run b: example + context** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
```

> Want the ADR in Thai? Add a line `language: Thai` to both messages (template headings and technical names stay in English). **Use the same language in Run a and Run b**, so the comparison is fair.

> In Run a, check that the agent really read nothing besides the template. If it opened `docs/scenario.md` anyway, note that in your prompt log, because it changes the comparison.

## Compare a and b

| Criterion | Run a | Run b |
|---|---|---|
| Uses the scenario's numbers (latency, peak, MailFast limits)? | No. It had no scenario and listed the numbers as open questions | Yes: 500 ms, 200 orders/s, MailFast p95/p99 and 50/s, 1 minute |
| Options considered (list them) | Sync calls; message broker (RabbitMQ or Kafka); polling job | Sync gRPC; RabbitMQ; in-process goroutines; Kafka |
| Handles "exactly once" loyalty points and MailFast's rate limit? How? | Only says "consumers must be idempotent"; no rate limit | Unique key on `order_id`; email worker rate-limited and buffered by the queue |
| Anything invented (a number, tool or fact not in the scenario)? | Suggested an outbox, but no numbers invented | Invented quorum queues and "clustered RabbitMQ" (not in the scenario); the 50 ms timeout, 3-node cluster and 40 emails/s were added in my edit |

## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

| # | What you changed from the AI's draft | Why |
|---|---|---|
| 1 | Lowered the email rate limit to 40/s (below MailFast's 50/s) and computed the backlog (about 160/s, about 40 min for a 10-minute peak) | The draft ignored that 200 orders/s is four times MailFast's limit |
| 2 | Added reconciliation job and outbox as mitigations for a lost event, instead of only "retry" | The draft's "never fail an order" conflicts with "points exactly once"; a lost event needs a recovery path |
| 3 | Gave the publish timeout a starting value (about 50 ms, "to be tuned in the load test") and added a post-sale review of queue depth and oldest email age to Validation | A bare "short timeout" cannot be tested; the 50 ms is my assumption, not from the scenario |
