Agent: Codex; model: GPT-6.

# ADR-BONUS: Use synchronous gRPC with a shared demo.proto contract

- **Status:** Reconstructed
- **Date:** 2026-10-27 (reconstruction date; original decision date: Not evidenced in the provided files)
- **Deciders:** Not evidenced in the provided files

## Context and problem

- **[C01, Evidenced]** `demo.proto` contains nine unary business-service interfaces and shared messages such as `Money`, `CartItem`, and `OrderResult`. Unary signatures describe request/response interfaces; observed caller behavior is addressed separately below.
- **[C02, Evidenced]** `checkoutservice` imports its `genproto` package, configures gRPC clients, and registers a gRPC server.
- **[C03, Evidenced]** `PlaceOrder` waits for preparation, payment, shipping, cart cleanup, and the email attempt before returning an order response. Its helpers make direct request/response RPC calls.
- **[C04, Evidenced]** `frontend/main.go` configures seven business-service gRPC clients and exposes HTTP routes.
- **[C05, Assumption — unverified]** The question’s “all services” premise is broader than this evidence. Universal synchronous gRPC use and exclusive generation from this one proto: **Not evidenced in the provided files**. Frontend handler bodies, other service implementations, and generation rules are outside the permitted evidence. Original motives and historically considered alternatives are also **Not evidenced in the provided files**.

The problem reconstructed here is how to coordinate the observed request/response interactions and maintain their contracts. Evidence references for every numbered claim appear in **Claim check**.

## Decision drivers

The following priorities are **[Inferred]**, not recorded author intent. Their ordering is also reconstructed.

1. **[C06] Understandability:** keep result-dependent checkout orchestration straightforward.
2. **[C07] Modifiability:** reuse consistent domain contracts across service boundaries.
3. **[C08] Development cost:** use a common RPC integration approach.

## Considered options

These are reconstructed comparison options; they are not a record of a historical evaluation.

1. **[C09, Inferred] A — Synchronous gRPC with a shared proto:** reconstruct the observed checkout interactions and configured clients around the business contracts in `demo.proto`.
2. **[C10, Inferred] B — Synchronous HTTP/JSON with separate service contracts:** replace the observed gRPC integrations and give each service its own contract.
3. **[C11, Inferred] C — Hybrid RPC and messaging with separate service contracts:** retain synchronous result-dependent RPCs, enqueue email confirmation for asynchronous delivery, and split contracts by service. A broker is part of this proposed alternative; its existence in this system is not established.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A | **[C12, Assumption]** Shared typed messages can reduce duplicate contract definitions. | **[C13, Inferred]** Requests wait on downstream RPCs. |
| B | **[C14, Assumption]** HTTP/JSON payloads are directly inspectable with common HTTP tooling. | **[C15, Inferred]** Replacing the observed gRPC integrations adds migration work. |
| C | **[C16, Assumption]** With successful durable enqueueing, checkout need not wait for `emailservice`. | **[C17, Assumption]** Delivery-state handling and broker operations add complexity. |

## Decision

Reconstruct **Option A for the observed integrations**, subject to the scope limit in C05. The synchronous checkout behavior is evidenced (C02–C03); the shared schema is evidenced (C01), while its precise generation and adoption across every service remain unverified (C05, C09).

**[Inferred]** The likely rationale is understandable result-dependent orchestration, reusable contracts, and a common integration approach (C06–C08). These are plausible explanations, not the authors’ documented reasons. The evidence does not establish a historical choice against B or C.

## Consequences

- **Positive — modifiability:** shared typed definitions can reduce duplicated contract definitions (**C12, Assumption**).
- **Positive — understandability:** a direct request/response flow is a plausible benefit for result-dependent checkout (**C06, Inferred**).
- **Negative — performance [C18, Inferred]:** serial product lookup and currency conversion for each cart item can increase checkout latency as carts grow; email latency also remains on the response path. No measured latency benefit or cost is established.
- **Negative — availability [C19, Evidenced]:** preparation, payment, and shipping failures abort checkout. Cart-cleanup errors are ignored and email errors only generate a warning; these dependencies therefore have different failure effects.
- **Negative — modifiability [C20, Inferred]:** breaking shared-message changes can require coordination among affected consumers. This does not establish that every change requires all services to deploy together.
- **Risks and mitigations — consistency [C21, Inferred; C24, Inferred proposal]:** charging precedes shipping, so shipping failure after a successful charge can produce a failed checkout with a charge. No compensation is visible in `PlaceOrder`. Define idempotent payment/retry behavior and recovery or compensation; downstream guarantees are unknown.
- **Risks and mitigations — availability and performance [C22, Evidenced; C23, Inferred proposal]:** both connection helpers discard the context returned by their three-second timeout, and `convertCurrency` calls the RPC with `context.TODO()`. These statements do not establish a three-second RPC deadline. Propagate caller cancellation and apply explicit RPC deadlines.
- **Risks and mitigations — modifiability [C25, Inferred proposal]:** check shared-contract compatibility against affected consumers before rollout; existing checks are not evidenced.

## Validation

These are proposed checks, not tests performed during this reconstruction.

- **[C26, Inferred proposal] Performance:** agree a latency target, then measure checkout p95/p99 across cart sizes and slow dependency responses, including email. Targets and results: **Not evidenced in the provided files**.
- **[C27, Inferred proposal] Reliability:** inject preparation, payment, shipping, cart, and email failures; verify the different response outcomes in C19. Test cancellation during currency conversion and inspect payment/shipping side effects after partial failure.
- **[C28, Inferred proposal] Compatibility:** exercise supported old/new consumer and provider combinations after shared-message changes. Establish actual code-generation and rollout rules before extending the conclusion to all services; that requires additional permitted evidence.

## Claim check

“Assumption” identifies general design knowledge or the explicitly unverified universal premise. “Inferred” includes reconstructed motives, alternatives, predicted effects, and proposed checks; none is evidence of historical intent.

| # | Claim | Type | Evidence |
|---|---|---|---|
| C01 | One proto3 file declares nine unary business-service interfaces and shared message types. | Evidenced | system/online-boutique/protos/demo.proto:15–19, CartService:23–27, RecommendationService:56–58, ProductCatalogService:71–75, ShippingService:107–110, CurrencyService:140–143, PaymentService:177–179, EmailService:199–201, CheckoutService:224–226, AdService:243–245; Money:146–161, CartItem:29–32, OrderResult:208–214. |
| C02 | checkoutservice uses a genproto package, gRPC clients, and a registered gRPC server. | Evidenced | system/online-boutique/src/checkoutservice/main.go:34 (pb import), main:118–123, 138–146; mustConnGRPC:210–220. |
| C03 | Checkout waits for successive downstream RPC results before returning its order response, including cart cleanup and the email attempt. | Evidenced | system/online-boutique/src/checkoutservice/main.go:PlaceOrder:238–279; prepareOrderItemsAndShippingQuoteFromCart:288–310; prepOrderItems:339–356; RPC helpers:313–394. |
| C04 | frontend configures seven business-service gRPC clients and serves HTTP routes. | Evidenced | system/online-boutique/src/frontend/main.go:main:132–147 (client configuration), 149–171 (HTTP routes and ListenAndServe). |
| C05 | The question’s universal premise, that all service communication is synchronous gRPC and all bindings come exclusively from this proto, is unverified. | Assumption | Not evidenced in the provided files. system/online-boutique/protos/demo.proto declares unary interfaces; system/online-boutique/src/checkoutservice/main.go:PlaceOrder shows synchronous calls; system/online-boutique/src/frontend/main.go:main shows connection configuration, but not handler bodies or code-generation provenance. Original rationale and historical options: Not evidenced in the provided files. |
| C06 | A likely driver is understandable checkout orchestration that obtains results before constructing the response. | Inferred | system/online-boutique/src/checkoutservice/main.go:PlaceOrder:238–279; system/online-boutique/protos/demo.proto:PlaceOrderResponse:237–239 and OrderResult:208–214. These show result dependencies, not an author-stated priority. |
| C07 | A likely driver is consistency through reuse of shared domain contracts. | Inferred | system/online-boutique/protos/demo.proto:Money:146–161, Product.price_usd:82, ChargeRequest.amount:189, OrderResult:208–214, SendOrderConfirmationRequest.order:218; no explicit rationale. |
| C08 | A likely driver is lower integration effort through a common RPC approach. | Inferred | system/online-boutique/src/checkoutservice/main.go:main:118–123 and 138–146; system/online-boutique/src/frontend/main.go:main:141–147. Repeated gRPC wiring suggests this motive but does not establish it. |
| C09 | Option A reconstructs synchronous gRPC with one shared business-contract proto for the observed integrations. | Inferred | system/online-boutique/protos/demo.proto:service declarations and shared messages; system/online-boutique/src/checkoutservice/main.go:34 and PlaceOrder:238–279; system/online-boutique/src/frontend/main.go:main:141–147. The exact generation pipeline and system-wide adoption are not evidenced. |
| C10 | Option B is a reconstructed alternative: synchronous HTTP/JSON with independently owned service contracts. | Inferred | Proposed comparison against gRPC wiring in system/online-boutique/src/checkoutservice/main.go:main:118–146 and system/online-boutique/src/frontend/main.go:main:141–147. Historical consideration: Not evidenced in the provided files. |
| C11 | Option C is a reconstructed alternative: retain synchronous result-dependent RPCs, queue email confirmation asynchronously, and split contracts by service. | Inferred | system/online-boutique/src/checkoutservice/main.go:PlaceOrder:273–279 provides a candidate asynchronous boundary; system/online-boutique/protos/demo.proto:EmailService:199–201 and SendOrderConfirmationRequest:216–219 define its data. Broker deployment and historical consideration: Not evidenced in the provided files. |
| C12 | Option A can reduce duplicate contract definitions through reuse of typed request, response, and domain messages. | Assumption | General contract-first design assumption; system-specific reuse examples: system/online-boutique/protos/demo.proto:Money:146–161, ChargeRequest:188–191, OrderResult:208–214. Measured development savings: Not evidenced in the provided files. |
| C13 | Option A incurs runtime waiting on downstream RPCs. | Inferred | system/online-boutique/src/checkoutservice/main.go:PlaceOrder:238–279 and prepOrderItems:339–356 show the synchronous waits; latency impact is inferred, not measured. |
| C14 | Option B makes payloads directly inspectable as HTTP/JSON text with common HTTP tooling. | Assumption | General HTTP/JSON property, not a finding about a deployed alternative. No implementation of Option B is evidenced in the provided files. |
| C15 | Option B would incur migration work replacing the observed gRPC integrations. | Inferred | system/online-boutique/src/checkoutservice/main.go:main:118–146 and RPC helpers:313–394; system/online-boutique/src/frontend/main.go:main:141–147 show integrations that would need adaptation; effort is not measured. |
| C16 | Option C could let checkout finish without waiting for emailservice if a durable queue accepts the notification. | Assumption | General asynchronous messaging assumption, conditional on queue durability and successful enqueueing; current blocking boundary: system/online-boutique/src/checkoutservice/main.go:PlaceOrder:273–279. No such queue is evidenced. |
| C17 | Option C adds delivery-state handling and broker operations, including retry and duplicate-message handling. | Assumption | General asynchronous messaging trade-off; implementation and operational costs for this system: Not evidenced in the provided files. |
| C18 | Serial per-item product and currency calls, followed by an email wait, can increase checkout latency as cart size or dependency latency grows. | Inferred | system/online-boutique/src/checkoutservice/main.go:prepOrderItems:343–350, convertCurrency:359–366, PlaceOrder:273–279. No benchmark or latency target is evidenced. |
| C19 | Checkout aborts on preparation, payment, or shipping errors, while ignoring cart-cleanup errors and only logging email errors. | Evidenced | system/online-boutique/src/checkoutservice/main.go:PlaceOrder:238–240, 252–263, 273–279; prepareOrderItemsAndShippingQuoteFromCart:290–304. |
| C20 | Breaking changes to shared message definitions can require coordination among affected consumers. | Inferred | system/online-boutique/protos/demo.proto:Money reused by Product.price_usd:82, ChargeRequest.amount:189, and OrderResult.shipping_cost:211; OrderResult reused by SendOrderConfirmationRequest:218 and PlaceOrderResponse:238. Lockstep deployment is not established. |
| C21 | A successful charge followed by shipping failure can leave a charged customer with a failed checkout response. | Inferred | system/online-boutique/src/checkoutservice/main.go:PlaceOrder:252–260. No compensation appears in this method; actual external side effects, refunds elsewhere, and retry outcomes are not evidenced. |
| C22 | The shown code does not attach the helpers’ three-second contexts to RPCs, and currency conversion uses context.TODO instead of the caller context. | Evidenced | system/online-boutique/src/checkoutservice/main.go:mustConnGRPC:210–216 and convertCurrency:359–362; system/online-boutique/src/frontend/main.go:mustConnGRPC:225–231. Both helpers discard the context returned by context.WithTimeout. |
| C23 | Proposed mitigation: propagate caller cancellation and apply explicit RPC deadlines to bound waiting. | Inferred | Recommendation prompted by system/online-boutique/src/checkoutservice/main.go:convertCurrency:359–362 and mustConnGRPC:210–216; system/online-boutique/src/frontend/main.go:mustConnGRPC:225–231. Effectiveness has not been validated. |
| C24 | Proposed mitigation: define idempotent payment/retry behavior and recovery or compensation after shipping failure. | Inferred | Recommendation prompted by system/online-boutique/src/checkoutservice/main.go:PlaceOrder:252–260. Existing downstream idempotency or recovery guarantees: Not evidenced in the provided files. |
| C25 | Proposed mitigation: check backward compatibility of shared contracts against affected consumers. | Inferred | Recommendation prompted by shared types in system/online-boutique/protos/demo.proto:Money:146–161 and OrderResult:208–214. Existing compatibility checks: Not evidenced in the provided files. |
| C26 | Proposed performance validation: agree a latency target and measure checkout percentiles across cart sizes and slow downstream responses. | Inferred | Test proposal derived from system/online-boutique/src/checkoutservice/main.go:prepOrderItems:343–350 and PlaceOrder:273–279. Targets and test results: Not evidenced in the provided files. |
| C27 | Proposed reliability validation: inject dependency errors and cancellation, then inspect outcomes and payment/shipping side effects. | Inferred | Test proposal derived from system/online-boutique/src/checkoutservice/main.go:PlaceOrder:238–279 and convertCurrency:359–362. No test execution or results are claimed. |
| C28 | Proposed compatibility validation: test supported old/new consumer and provider versions after shared-message changes. | Inferred | Test proposal derived from shared messages and RPC signatures in system/online-boutique/protos/demo.proto:Money:146–161, OrderResult:208–214, CheckoutService:224–239. Generation and rollout verification require evidence beyond the permitted files. |
