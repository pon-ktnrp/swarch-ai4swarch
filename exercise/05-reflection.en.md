**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:**
Pon Kittinaraporn / 6630237121
**My role on the review board:**
SRE
After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.
I would trust AI to identify failure scenarios, calculate throughput constraints, and suggest areas for monitoring and testing. For example, the review highlighted the need to clarify what happens when the broker is unavailable and loyalty processing cannot meet its one-minute target. AI can suggest retries, dead-letter queues, and recovery tests, but these mechanisms must be validated under realistic failure conditions. As the SRE, I would keep responsibility for setting operational requirements, checking observability and recovery procedures, and verifying that the system meets its service targets. A design that looks reliable on paper may still fail under load or during an outage. I would therefore use AI to broaden the set of risks we consider, while relying on testing, operational evidence, and human judgment before approving production readiness.
…
