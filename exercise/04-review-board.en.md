**Language:** [ภาษาไทย](04-review-board.md) | English

# Part B: Architecture review board (approximately 25 min, groups of 4)

> **Lecture link:** AICH4 (reviews need domain expertise; *plausible ≠ appropriate*), Pillar 5 (AI as Chief Analyst, humans as Chief Strategist), JSS (85% of studies keep a human in the loop).
> **Question:** what does a human review board catch that the AI's review missed, and why?

The company's architecture review board meets to approve the R-12 design before the 11.11 sale. One ADR goes to the board: the **architect** presents it, the **AI analyst** presents the AI's review, and two stakeholders bring knowledge the AI never saw.

## 1. Form the board (2 min)

In groups of 4, choose **one** member's ADR to review (`docs/adr/0002-*.md`, plus their `exercise/outputs/3-review.md`). Then assign roles:

| Role | Who | Brings |
|---|---|---|
| **Architect** | Pitchanan Thaiphayak | defends the ADR |
| **AI analyst** | Chayutphong | reads out the AI's review only, and may not add their own opinions |
| **Product owner** | Kotchapan | role card P (business side) |
| **Site reliability engineer (SRE)** | Pon Kittinaraporn | role card S (operations side) |
| **Security & privacy officer** | Piyapanchanok | role card X (security and privacy) |

Groups of 5 add a **security & privacy officer** (role card X). In groups of 3, the architect also plays the AI analyst.

## 3. Record the decision (8 min, together)

**ADR reviewed:** `docs/adr/0002-publish-order-events-via-rabbitmq.md` by Pitchanan Thaiphayak — Publish an OrderPlaced event to RabbitMQ for post-order actions.
**Board members and roles:** Pitchanan Thaiphayak — Architect; Chayutphong — AI analyst; Kotchapan — Product owner; Pon Kittinaraporn — SRE; Piyapanchanok — Security & privacy officer. This record uses AI-assisted wording based on our group discussion notes.

**Decision:** accept with changes (vote: 5–0)

**Required changes:**

1. Clarify the boundaries between payment, order persistence, outbox recording and event publication. Define relay operation, retries, duplicate handling, dead-letter processing, monitoring and recovery, including reconciliation when payment succeeds before an event is durably recorded.
2. Agree on acceptable email delays, backlog recovery and the interpretation of the one-minute loyalty requirement during outages. Confirm the expected peak duration, label the 30-minute example as an assumption, and support broker comparisons with evidence.
3. Define security and privacy controls for event contents, authentication, authorization, connection protection, retention and access to queues, dead-letter records and logs. Validate peak throughput, broker outages, consumer failures, duplicate delivery and recovery before production.

**Who raised what?** List every concern the board discussed.

| # | Concern | Raised by (AI review / architect / PO / SRE / security) | Was it in the AI's review? | Could the AI have known? Why or why not? |
|---|---|---|---|---|
| 1 | Payment may succeed without a durable event. Relay operation, retries, duplicate handling, monitoring and recovery also need clearer definitions. | AI review / architect / SRE | Yes. | AI can identify the consistency risk and missing recovery details from the documented design. The recovery policy and operational ownership require human agreement. |
| 2 | Loyalty may miss its one-minute deadline during an outage. Workload and failure testing are needed to validate the proposed guarantees. | AI review / PO / SRE | Yes for the deadline risk; the SRE also emphasized realistic validation. | AI can identify the conflict between the deadline and continued order availability. Business priorities require stakeholder decisions, while actual performance requires measured results. |
| 3 | Email backlog can grow at peak load, and the 30-minute peak illustration must not be treated as a confirmed requirement. | AI review / PO | Yes. | AI can calculate approximately 9,000 additional queued emails per minute from 200 orders/s and 50 emails/s, assuming one email per order and sustained rates. Actual peak duration and acceptable delay need confirmation. |
| 4 | Some RabbitMQ-versus-Kafka comparison claims lack supporting evidence. | AI review / architect | Yes. | AI can flag unsupported claims and request evidence. The technology decision still needs justification against the actual constraints. |
| 5 | Personal data in events and dead-letter records needs explicit controls for access, authentication, connection protection and retention. | Security | Not a main finding in the supplied review summary; the ADR listed privacy as an open question. | AI could flag the unresolved question and suggest controls. Organization-specific security and privacy requirements must come from the responsible stakeholders. |

**The AI's best finding** (one the humans had missed): The AI first brought the payment-to-outbox consistency gap to our attention. We had not raised this risk before reading the AI review. This finding prompted us to require explicit recovery and reconciliation when payment succeeds before an order event is durably recorded.
**The AI's weakest finding** (generic, wrong or irrelevant): The 30-minute peak illustration was the least dependable basis for a decision because the duration was not established by the scenario. The backlog calculation was useful, but the duration must remain labelled as an assumption.

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.
