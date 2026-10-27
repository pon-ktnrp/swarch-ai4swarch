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
| Uses the scenario's numbers (latency, peak, MailFast limits)? |No. The supplied R-12 requirement did not include these numbers. The draft leaves workload, latency targets and MailFast limits as open questions. |Yes. It uses checkout p95 ≤ 500 ms, approximately 300 ms today, peak load of 200 orders/s, and MailFast’s p95 1.5 s, p99 8 s, approximately 0.5% failures and 50 emails/s limit. These match docs/scenario.md, Quality goals and constraints. |
| Options considered (list them) |1. Direct synchronous calls. 2. Durable event fan-out with independent subscriptions. |1. Synchronous checkout orchestration. 2. Durable job table with a central dispatcher. 3. RabbitMQ publish/subscribe with a transactional outbox and independent consumers. |
| Handles "exactly once" loyalty points and MailFast's rate limit? How? |It proposes stable identifiers and atomic deduplication of loyalty updates to prevent duplicate points. However, it does not specify the one-minute deadline or a MailFast rate limiter because these constraints were not supplied. |It proposes atomically recording a unique (order_id, action) key with the points update, making duplicate deliveries no-ops. It also proposes a shared limiter capped at 50 MailFast attempts/s and separate queues so email delays do not block loyalty. It acknowledges that long outages can prevent meeting the one-minute deadline. |
| Anything invented (a number, tool or fact not in the scenario)? |No unsupported numerical targets were introduced. The broker, event contract and outbox are proposed design additions. They are not presented as existing infrastructure. |RabbitMQ, the outbox, new durable storage and consumer mechanisms are proposed additions.  |

## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

| # | What you changed from the AI's draft | Why |
|---|---|---|
| 1 | Added a checkout retry policy requiring a stable idempotency key, one order identity, durable processing state and recovery validation. | The existing handler generates a new UUID per invocation (`system/online-boutique/src/checkoutservice/main.go:233–236`). Request-level retry safety needs explicit treatment to improve correctness and reliability. |
| 2 | Added event compatibility rules covering optional fields, incompatible versions, migration and compatibility testing. | The scenario anticipates more post-order actions (`docs/scenario.md:14`). These rules improve modifiability as consumers evolve. |
| 3 | Assigned recovery responsibilities by role and added requirements for authorized, recorded replay. Named individuals remain an open question. | The draft proposes outbox recovery and dead-letter handling but leaves ownership unresolved. Clear responsibilities improve operability and controlled recovery. |