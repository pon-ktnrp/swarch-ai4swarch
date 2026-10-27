Agent: Codex; Model: GPT-6.

# ADR-0002: Use durable events for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Not provided; to be confirmed.

## Context and problem

Evidence is limited to the requirement in the student's message (`context: none`). No implementation files were read.

| Reference | Evidence from the supplied requirement |
|---|---|
| R1 | Clause 1: send confirmation through external provider MailFast, replacing mock `emailservice` logic. |
| R2 | Clause 2: award loyalty points through a new `loyaltyservice`. |
| R3 | Clause 3: record an “order placed” analytics event for marketing. |
| R4 | Final paragraph: expect more actions next year, including SMS, fraud scoring, and a recommendations update. |

We must decide how order placement triggers these actions and accommodates additions. Existing calls, protocols, order storage, and messaging infrastructure: **Not evidenced in the provided files**. R1 identifies mock email logic but does not establish its call path.

**[Inferred]** The actions can complete independently and asynchronously after order placement. The requirement does not specify completion deadlines, ordering, or whether customers must wait; this proposal requires confirmation of those semantics.

## Decision drivers

The following priorities are **[Inferred]** from R1–R4; no ranking or numerical targets were supplied.

1. **Modifiability:** accommodate additional actions without repeatedly changing the order-placement flow (R4).
2. **Reliability:** recover required actions after transient failures (R1–R3).
3. **Availability and performance:** isolate downstream failures and delays from order completion; this is a proposed quality goal.
4. **Cost and operability:** keep infrastructure and recovery manageable; budget and team skills are unknown.

## Considered options

1. **Direct synchronous calls:** the order-placement component invokes an email adapter for MailFast, `loyaltyservice`, and an analytics endpoint.
2. **Durable event fan-out:** publish an `OrderPlaced` event to a durable broker with an independent subscription for each action.

Both are proposed designs, not verified descriptions of the current system.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| Direct synchronous calls | Fewer new infrastructure components; immediate action results available to the caller. | Each addition changes the calling flow; downstream latency and failures affect completion; partial success still requires recovery and duplicate protection. |
| Durable event fan-out | Consumers can be added independently when the event contract suffices; separate subscriptions isolate retries and allow concurrent processing. | Adds messaging infrastructure, publication recovery, duplicate handling, schema management, and monitoring; completion becomes eventually consistent. |

## Decision

**Propose durable event fan-out**, subject to confirming asynchronous completion. It supports the expected growth in actions (R4). Broker product selection remains open.

Proposed behavior:

- Secure a durable, recoverable event handoff for every successfully placed order before reporting success. Use a transactional outbox if compatible transactional order storage exists; otherwise establish an equivalent recoverable handoff before implementation. Existing persistence is unknown. A best-effort publish after saving an order would leave a failure window.
- Provide independent durable subscriptions for replacement email logic calling MailFast (R1), a handler awarding points through `loyaltyservice` (R2), and an analytics handler (R3). Their APIs and deployment boundaries remain unspecified.
- Use stable event/order identifiers, a versioned event contract, and at-least-once delivery. Each handler must protect against duplicate effects. Loyalty awarding and analytics recording must atomically deduplicate their respective effects using an idempotency key.
- Retry transient failures with backoff; retain exhausted failures for investigation and controlled replay. Acknowledge an action only after its outcome is safely recorded.
- Confirm MailFast's idempotency or delivery-status lookup support. A crash after provider acceptance but before local acknowledgement creates an ambiguous outcome; retries alone cannot guarantee one email. Agree on reconciliation and duplicate-email policy before implementation.
- Under the proposed semantics, order responses do not wait for action completion after durable handoff, and actions have no cross-consumer execution order. A future fraud check that must gate acceptance would require revisiting this decision.
- Limit event data to what each action needs, exclude payment credentials, and correlate processing with event/order identifiers.

## Consequences

- **Positive:** new consumers can extend behavior without publisher changes when existing event fields suffice (quality attribute: **modifiability**).
- **Positive:** one action's failure need not stop other consumers; downstream processing leaves the response path after durable handoff (quality attributes: **availability, performance**).
- **Negative:** confirmations, loyalty balances, and analytics may lag order placement; acceptable delays remain unresolved (quality attributes: **consistency, usability**).
- **Negative:** messaging, recovery, and monitoring add infrastructure expense and operational work (quality attributes: **cost, operability**).
- **Risks and mitigations:** lost handoffs and duplicate effects could violate R1–R3. Validate publication recovery and effect-level idempotency; resolve MailFast ambiguity explicitly (quality attribute: **reliability**).
- **Risks and mitigations:** contract changes and shared personal data can affect consumers. Review schema compatibility and minimize payload/access scope (quality attributes: **modifiability, security**).

## Validation

Proposed checks; none were executed.

- Complete an order and verify its MailFast confirmation, one loyalty award, and one analytics record using the order identifier.
- Interrupt processing around order persistence, event handoff, and publication. Verify every successful order retains recoverable work and unsuccessful orders trigger no actions.
- Redeliver events and crash handlers around effects. Verify duplicate protection for points and analytics, and MailFast behavior against the agreed email policy.
- Simulate broker/downstream outages and replay retained failures. Verify independent consumer progress and the agreed customer response when durable handoff fails.
- Measure response latency, queue age, action completion delay, and unresolved failures under an agreed workload; compare with targets set before acceptance.
- Add a sample subscriber supported by the existing contract and verify no publisher change is required.

## Open questions

- Who approves this decision, and which quality goals have priority?
- Is asynchronous completion acceptable? What deadlines, ordering, and customer-visible status are required?
- What defines a durably placed order? Which persistence/transaction capabilities support recoverable publication, and how should handoff failures and order retries behave?
- What workloads, service-level targets, recovery objectives, retention periods, budget, team skills, and existing infrastructure apply?
- What MailFast API, authentication, rate limits, idempotency, and status-lookup capabilities apply? What duplicate-email policy is acceptable?
- What loyalty rules, customer identity, cancellation/reversal behavior, and analytics schema/destination are required?
- What privacy/access constraints apply, and who owns each consumer, failure investigation, and replay?
- Will future actions require sequencing or gate order acceptance?
