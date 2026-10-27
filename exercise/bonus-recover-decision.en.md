**Language:** [ภาษาไทย](bonus-recover-decision.md) | English

# Bonus (optional, take-home): Recover an unwritten decision (approximately 15 min)

> **Lecture link:** T5 (Dhar et al. generate decisions from code and diagrams to reduce *knowledge vaporization*), AICH4.
> **Question:** when AI explains *why* a system was built this way, how much of the explanation is evidence and how much is a confident guess?

## Run

Start a new chat or session in your agent and send:

```
Follow prompts/bonus-recover-decision.md
decision: Why do all services communicate synchronously over gRPC, using one shared demo.proto?
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go
```

Want the ADR in Thai? Add a line `language: Thai`.

The agent writes the reconstructed ADR, including its **Claim check** table, to [`outputs/bonus-recovered-adr.md`](outputs/bonus-recovered-adr.md).

## Audit the claim check

Choose **five** claims from the agent's table: at least two it marked *Evidenced* and two it marked *Inferred*. Check each one yourself.

| # | Claim (short) | AI says | You say | Why (file path or reasoning) |
|---|---|---|---|---|
| 1 | C01: demo.proto declares nine unary business-service interfaces and shared message types. | Evidenced | Evidenced | `system/online-boutique/protos/demo.proto` directly declares the nine services. Examples of shared messages include `CartItem:29–32`, `Money:146–161` and `OrderResult:208–214`. The RPC signatures contain no streaming parameters or responses. |
| 2 | C03: checkout waits for downstream RPC results before returning, including cart cleanup and the email attempt. | Evidenced | Evidenced | `system/online-boutique/src/checkoutservice/main.go:238–279` calls preparation, payment, shipping, cart cleanup and email before returning the response. The helpers at lines 313–394 make direct RPC calls. Ignoring an error does not make the call asynchronous. |
| 3 | C06: understandable result-dependent orchestration was a likely decision driver. | Inferred | Inferred | `system/online-boutique/src/checkoutservice/main.go:238–279` shows a result-dependent flow, but does not state that understandability motivated the original architecture. |
| 4 | C07: consistent domain contracts were a likely decision driver. | Inferred | Inferred | `system/online-boutique/protos/demo.proto` reuses `Money` in `Product.price_usd:82`, `ChargeRequest.amount:189` and `OrderResult.shipping_cost:211`. This demonstrates reuse, but the motivation behind it is not documented. |
| 5 | C05: all service communication is synchronous gRPC and every binding comes exclusively from this proto. | Assumption | Assumption | The permitted files show contracts, checkout calls and frontend connection configuration. They do not include every service implementation or the generation rules, so the universal premise remains unverified. See `system/online-boutique/protos/demo.proto`, `system/online-boutique/src/checkoutservice/main.go:238–279` and `system/online-boutique/src/frontend/main.go:132–171`. |


**Count:** of all the agent's claims, how many are *Evidenced* according to the agent? **6** According to you? **6**

## Reflect

1. Did the agent ever label a guess as *Evidenced*? Give an example, or explain how you checked that it didn't.
I did not identify a guess labelled Evidenced. I checked C01, C02, C03, C04, C19 and C22 against their cited declarations, configuration and control flow in the three permitted files. Claims about original motives, such as understandability in C06 and consistent contracts in C07, were labelled Inferred rather than presented as documented author intent.
2. A new developer reads only this reconstructed ADR. What wrong belief could they form?
A developer could mistake the reconstructed rationale for the original architects' documented reasoning. For example, C07 suggests that consistent domain contracts motivated the shared proto, but `system/online-boutique/protos/demo.proto` only demonstrates message reuse. They could also overgeneralize the observed integrations to all service communication, despite C05 explicitly identifying that premise as unverified.
3. What can AI recover from code, and what can only the original architects know? Use one example from your ADR.
AI can recover declared interfaces, shared messages, configured connections and observable call sequences. For example, reuse of `Money` in `system/online-boutique/protos/demo.proto:82,189,211` is directly evidenced, while the claim that consistency motivated this choice is inferred. The original priorities and historically considered alternatives require decision records or confirmation from the original architects.