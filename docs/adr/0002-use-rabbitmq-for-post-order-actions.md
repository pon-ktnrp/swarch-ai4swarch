Agent: Codex; model: GPT-6

# ADR-0002: Use RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Pon Kittinaraporn

## Context and problem

R-12 requires order confirmation through MailFast, loyalty points through a new `loyaltyservice`, and an order-placed analytics event. Marketing expects SMS, fraud scoring and recommendations updates next year (`docs/scenario.md:9–14`).

| Evidence | Facts relevant to this decision |
|---|---|
| `system/online-boutique/src/checkoutservice/main.go:230–279` (`PlaceOrder`), `:369–393` | Checkout charges the card, ships the order, attempts to empty the cart, constructs `OrderResult`, then waits for the confirmation call before returning. Email errors only produce a warning. The helper calls use gRPC clients. |
| `docs/scenario.md:24–26` | Checkout p95 must stay ≤ 500 ms; today it is approximately 300 ms. Peak load is 200 orders/s. MailFast has p95 1.5 s, p99 8 s, approximately 0.5% call failures, and a 50 emails/s limit. |
| `docs/scenario.md:27–28` | Loyalty points must be awarded within 1 minute, exactly once. Orders must never fail because email, loyalty or analytics is down. |
| `docs/scenario.md:29–30` | Four developers are strong in Go and gRPC; two have used RabbitMQ. Kubernetes is established; open-source, self-hosted components are preferred. |
| `docs/adr/0001-use-redis-for-cart-storage.md:17–20,38–45` | The reference decision uses Redis for carts and treats cart durability as noncritical. It does not establish durable order or event storage. |

No durable order save, outbox write or event publication appears in `PlaceOrder` (`system/online-boutique/src/checkoutservice/main.go:230–279`). An existing order-storage/recovery contract is **Not evidenced in the provided files**.

**[Inferred]** Keeping MailFast in the checkout request path conflicts with the latency target: its p95 alone exceeds 500 ms. At one email per order, sustained peak traffic creates at least 150 queued emails/s (200 − 50), or 9,000/minute, before retries. An asynchronous design absorbs a finite burst; it cannot remove this capacity mismatch (`docs/scenario.md:10,24–26`).

## Decision drivers

1. **Availability:** isolate order success from post-order consumer outages (`docs/scenario.md:28`).
2. **Performance:** preserve checkout p95 ≤ 500 ms at 200 orders/s (`docs/scenario.md:24–25`).
3. **Correctness and timeliness:** award loyalty points once per order within 1 minute (`docs/scenario.md:27`).
4. **Modifiability:** add the expected post-order actions with limited checkout changes (`docs/scenario.md:14`).
5. **Operability and cost:** fit the four-person team's experience and self-hosted Kubernetes preference (`docs/scenario.md:29–30`).

## Considered options

These are proposed designs, not descriptions of deployed components. Their expected trade-offs are **[Inferred]** from the cited drivers and current call sequence.

1. Extend synchronous checkout orchestration: call MailFast, loyalty and analytics before returning, with timeouts and nonfatal post-order errors.
2. Use a durable job table and central dispatcher: persist separate email, loyalty and analytics jobs; workers execute and retry them outside checkout.
3. Use RabbitMQ publish/subscribe with a transactional outbox and independent consumers.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| Synchronous orchestration | Closest to the existing Go/gRPC request flow; no broker to operate. | Waiting for MailFast threatens the 500 ms target; timeouts can abandon required work without durable retry; each new action changes checkout. |
| Durable job dispatcher | Removes external calls from checkout; persistent jobs support recovery and individual retry without a broker. | Requires a durable store plus polling, claiming and retry logic; a central task registry must evolve as actions grow; polling adds scheduling delay. |
| RabbitMQ plus outbox | Independent queues isolate slow actions; new subscribers can consume the same event; uses existing RabbitMQ experience. | Adds broker and durable outbox operations; duplicates, schema compatibility, queue capacity and recovery need explicit handling. |

## Decision

Choose **option 3**. **[Inferred]** Independent consumers best address the expected growth in actions and isolate MailFast's latency and rate limit, while RabbitMQ fits the stated team and deployment constraints (`docs/scenario.md:14,24–30`). The following mechanisms are proposed additions:

1. **Durable publication:** after successful payment and shipping, atomically persist an order-completion record and an `OrderPlaced` outbox entry before returning success. A separate relay publishes persistent messages to RabbitMQ and marks entries sent only after publisher confirmation. Broker outages leave entries pending rather than failing checkout. This requires new durable storage; the cart-storage ADR is not an order durability guarantee. Recovery from a crash between remote payment/shipping success and the local commit remains an implementation prerequisite (`system/online-boutique/src/checkoutservice/main.go:252–279`; `docs/adr/0001-use-redis-for-cart-storage.md:17–20`).
2. **Independent delivery:** use a durable topic exchange with separate durable, replicated queues for email, loyalty and analytics. Configure at-least-once delivery; acknowledge only after the consumer's durable effect succeeds. Use bounded retries with backoff, dead-letter queues, alerts and controlled replay. Email retries must not delay loyalty processing. Future actions get their own subscriptions.
3. **MailFast adapter:** replace the mock confirmation behavior with an email consumer calling MailFast; remove the synchronous confirmation call from checkout. Apply a shared rate limiter across all email workers and retries, capped at 50 calls/s. MailFast's API protocol and idempotency support are **Not evidenced in the provided files**; do not assume them (`docs/scenario.md:10,26`; `system/online-boutique/src/checkoutservice/main.go:273–277,379–383`).
4. **Exactly-once loyalty effect:** have `loyaltyservice` consume the event. Atomically record a unique `(order_id, action)` key and apply the points update in its durable store; concurrent duplicates become no-ops. Acknowledge after commit. Redelivery after a crash must not award points again. This provides one business effect per order, conditional on that transaction, rather than assuming exactly-once message transport (requirement: `docs/scenario.md:27`).
5. **Event contract:** propose a stable event ID, order ID, user ID, occurrence time, schema version and only the order data each action needs. Order/user IDs and order details are available in `PlaceOrder` (`system/online-boutique/src/checkoutservice/main.go:230–270`); the event schema is new. Exclude credit-card details. The analytics consumer durably records one logical `order placed` event per order using an idempotent 
sink before acknowledging; the sink is undecided.
6. **Checkout retry safety:** Introduce a stable idempotency key for each logical purchase. Durably associate it with one order ID and processing state before payment or shipping. Concurrent or repeated requests with the same key must reuse that order; conflicting purchase data must be rejected. Recovery must resolve uncertain payment/shipping outcomes before repeating those effects. The existing handler generates a new UUID per invocation (`system/online-boutique/src/checkoutservice/main.go:233–236`). Durable storage and downstream idempotency or reconciliation mechanisms remain implementation prerequisites.
7. **Event compatibility:** Preserve existing field names, types and meanings within an event version. New fields must be optional, and consumers must tolerate unknown optional fields. Incompatible changes require a new event version, compatibility tests and a migration plan before the previous version is retired. This proposed policy supports the additional post-order actions anticipated in `docs/scenario.md:14`.
8. **Recovery ownership:** Assign the checkout maintainer responsibility for order/outbox recovery and publication failures, the broker maintainer responsibility for RabbitMQ health and queue alerts, and each consumer maintainer responsibility for failed actions and dead-letter investigation. The responsible maintainer authorizes replay for their component. Replays must preserve original event/order identifiers, record the operator and reason, and use the normal deduplication and rate-limiting mechanisms. Named individuals must be assigned to these roles before deployment.

## Consequences

- **Positive:** post-order provider time is removed from checkout, and independent queues prevent email backlog from blocking loyalty (**performance, availability**; `docs/scenario.md:24–28`). The 500 ms target still requires measurement of the new durable write.
- **Positive:** additional actions can subscribe without adding checkout calls when the event contains their required data (**modifiability**; `docs/scenario.md:14`).
- **Positive:** stable request identity and verified recovery reduce the risk of duplicate logical orders and repeated business effects (**correctness, reliability**; proposed checkout retry policy).
- **Positive:** explicit event compatibility rules allow consumers to evolve with less coordination (**modifiability**; growth requirement: `docs/scenario.md:14`).
- **Negative:** durable request tracking, compatibility testing, temporary support for older event versions and assigned recovery duties add implementation and operational work (**performance, cost, operability**). Measure the added persistence latency against the checkout target in `docs/scenario.md:24`.
- **Negative:** the broker, durable stores, relay, retries and monitoring increase work for four developers; the order/outbox store becomes a checkout dependency (**cost, operability, availability**; `docs/scenario.md:29–30`; proposed design).
- **Negative:** effects become eventually consistent. At sustained peak, email backlog grows by at least 150/s; retries reduce effective capacity further (**timeliness, scalability**; **[Inferred]** from `docs/scenario.md:25–26`). Mitigate with capacity monitoring and an agreed peak duration, delivery target or provider quota increase.
- **Risks and mitigations:** crashes can lose events at an unresolved order-completion boundary; require a durable recovery/reconciliation design before implementation approval. Outbox replay can duplicate delivery; use atomic loyalty deduplication and idempotent analytics writes (**reliability, correctness**; proposed design; current boundary: `system/online-boutique/src/checkoutservice/main.go:252–279`).
- **Risks and mitigations:** MailFast may accept a send before a timeout; retries may duplicate email unless the provider supports idempotency or status reconciliation. Consumer or broker outages longer than 1 minute prevent the loyalty deadline even though orders remain successful. Alert on event age and obtain an explicit outage policy (**correctness, timeliness, availability**; `docs/scenario.md:26–28`).

## Validation

These are proposed acceptance checks; none has been executed for this ADR.

- Load-test at **200 orders/s** with MailFast's stated latency, failure rate and rate cap; verify checkout **p95 ≤ 500 ms**, every healthy-path loyalty award occurs **within 1 minute**, and aggregate MailFast attempts stay **≤ 50/s** (`docs/scenario.md:24–27`). Use the peak duration agreed under Open questions.
- Stop email, loyalty and analytics individually, then stop RabbitMQ. Verify orders still succeed, completed orders retain pending work, independent queues remain isolated, and work recovers after restart. Measure and report loyalty deadline misses rather than treating eventual recovery as compliance (`docs/scenario.md:27–28`).
- Inject duplicate/concurrent events and crashes after points commit but before acknowledgement; verify exactly one points update per order. Exercise relay crashes after broker confirmation but before outbox marking, and analytics replay (loyalty requirement: `docs/scenario.md:27`; proposed delivery design).
- Exercise crashes before/after the order/outbox commit and after remote payment/shipping success; reconcile each completed order with its event and all required effects. Reject readiness if completed orders can disappear at the boundary (`system/online-boutique/src/checkoutservice/main.go:252–279`; `docs/scenario.md:9–12`).
- Add a trial subscriber and review event compatibility; monitor outbox age, queue age/depth, loyalty completion age, retries, dead letters and email drain time (growth requirement: `docs/scenario.md:14`; proposed design).
- Send repeated and concurrent checkout requests using the same idempotency key, including a retry after a lost response. Verify that they refer to the same order and do not duplicate charges, shipments or loyalty awards. Exercise crashes after payment or shipping and verify that recovery resolves uncertain outcomes before retrying those effects. Treat unresolved recovery behavior as a deployment blocker.
- Test an older consumer against events containing new optional fields. Verify that existing behavior remains correct. For incompatible changes, demonstrate the new event version and consumer migration before retiring the previous version.
- Simulate outbox publication failures and consumer failures. Verify that the assigned maintainers can identify affected events and perform an authorized, recorded replay. Confirm that replay preserves identifiers, prevents duplicate loyalty awards and respects the shared MailFast limiter.

## Open questions

- Who formally approves this ADR, and which named individuals will fill the checkout, broker and consumer maintenance roles? What escalation arrangements apply when an owner is unavailable? These assignments are not supplied by the scenario.
- Which durable stores and concurrency controls will implement request-key tracking, order/outbox atomicity and atomic loyalty updates? How will request keys be issued and retained? Which payment/shipping idempotency or reconciliation mechanisms will resolve uncertain outcomes after a crash? These implementation capabilities are **Not evidenced in the provided files**.
- How long does the 200 orders/s peak last, what email delay is acceptable, and can MailFast's quota increase? These determine buffer capacity and drain time (`docs/scenario.md:25–26`); durations, storage sizes and email deadlines are **Not evidenced in the provided files**.
- Does the 1-minute loyalty deadline include outages? An outage exceeding that deadline conflicts with continuing successful orders; agree on an explicit requirement interpretation (`docs/scenario.md:27–28`).
- What are the loyalty formula, analytics destination/schema, MailFast API/idempotency contract, event retention/privacy rules, and retry/replay limits? **Not evidenced in the provided files**.

