Agent: Codex; model: GPT-6

# ADR review: RabbitMQ for post-order actions

Evidence: **ADR** = `docs/adr/0002-use-rabbitmq-for-post-order-actions.md`; **S** = `docs/scenario.md`. Numbers below are file line numbers. Assessments marked **[Inferred]** are reviewer judgments, not measured results.

## Quality-attribute scenarios

1. **Source:** sale customers · **Stimulus:** 200 orders/s · **Environment:** MailFast has p95 1.5 s, p99 8 s, approximately 0.5% failures and a 50 emails/s limit · **Response:** complete checkout and retain post-order work independently · **Response measure:** `PlaceOrder` p95 ≤ 500 ms; no order failures caused by email, loyalty or analytics outages (S:24–28).
2. **Source:** relay/loyalty worker · **Stimulus:** duplicate delivery or a crash after points commit, before acknowledgement · **Environment:** ongoing orders and recovery · **Response:** redeliver without another points update · **Response measure:** exactly one award per order within 60 s. The deadline has no outage exception in the requirement (S:27–28; ADR:56,81,93).
3. **Source:** email traffic · **Stimulus:** orders arrive at 200/s · **Environment:** the peak lasts `T` seconds, with MailFast limited to 50/s · **Response:** buffer email while loyalty continues · **Response measure:** shared attempts ≤ 50/s; loyalty ≤ 60 s. **[Inferred]** At one email/order, backlog grows by at least `150T` messages; peak duration and acceptable email delay are **Not evidenced in the provided files** (S:10,25–27; ADR:55,71,92).

## Findings table

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | Atomic loyalty deduplication and points updates, followed by acknowledgement, address duplicate delivery explicitly (ADR:56,81). | **[Inferred] Correctness:** sound conditional design for one business effect; validation remains unexecuted (ADR:77). |
| Risk | Recovery between successful payment/shipping and the order/outbox commit remains a prerequisite, despite the request-key policy (ADR:53,59,91). | **[Inferred] Reliability:** unresolved outcomes could leave completed orders without events or cause repeated remote effects; a key alone does not establish recovery. |
| Risk | The ADR acknowledges that outages exceeding one minute prevent the loyalty deadline (ADR:73,93). | **Timeliness/availability:** continuing successful orders cannot satisfy the unconditional deadline when loyalty remains unavailable beyond it (S:27–28). |
| Sensitivity point | Email peak duration, retry volume and effective send capacity determine backlog growth and drain time (ADR:55,71,92). | **[Inferred] Scalability:** the 50/s limiter protects MailFast but does not bound stored work during a sustained 200/s peak (S:25–26). |
| Trade-off point | Durable request tracking and order/outbox commits protect recovery while adding synchronous persistence and a checkout dependency (ADR:53,59,69–70). | **Reliability versus performance/availability:** the proposed writes must fit p95 ≤ 500 ms; store failure behavior is unresolved (S:24; ADR:91). |

## Missing or weak

- **Options:** option 2's central-registry coupling is asserted without comparing per-action workers with independent claiming. **[Inferred]** That coupling depends on its design; compare recovery effort and operability against RabbitMQ for four developers (ADR:38,46–47; S:29).
- **Consequence/claim:** “independent queues prevent email backlog from blocking loyalty” is stronger than the stated design establishes. **[Inferred]** Resource contention in the shared broker needs capacity limits and a combined backlog test; separate queues alone provide no demonstrated latency guarantee (ADR:54,65,79–80).
- **Evidence:** checks are proposals, not results (ADR:77). Implementation assertions about checkout and cart storage are reported by the ADR; their cited source files were outside this review's permitted evidence (ADR:15,19,21,59).

## Verdict

**Accept with changes** to the proposed direction; deployment readiness is unproven. The three most important changes are:

1. Specify the durable stores, transaction boundaries, uncertain-payment/shipping recovery and store-failure behavior; demonstrate crash recovery without lost events or repeated effects (ADR:53,59,82,84,91).
2. Obtain an explicit agreement on the loyalty deadline during outages; define detection and recovery bounds consistent with it (S:27–28; ADR:73,93).
3. Agree peak duration and email delivery targets; size buffers, define saturation behavior and test checkout/loyalty targets with email backlog, retries and broker recovery (ADR:79–80,92).
