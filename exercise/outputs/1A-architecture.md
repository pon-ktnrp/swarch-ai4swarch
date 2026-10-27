# Architecture recovery — Run A

## Agent

Codex — GPT-6.

## Files read

- `prompts/1-recover-architecture.md` — task instructions.
- `system/online-boutique/protos/demo.proto` — sole architecture evidence.

The repository instructions in `AGENTS.md` were supplied in the conversation; the file was not opened.

## Containers table

The evidence declares nine service interfaces. **[Inferred]** Each interface is represented as a candidate container below; a one-to-one mapping to independently deployed containers is **Not evidenced in the provided files**. Responsibilities describe the declared APIs, not verified implementation behavior.

The technology column describes the contract only. Implementation languages, runtime frameworks and network transports are **Not evidenced in the provided files**. `syntax = "proto3"` identifies the schema syntax (`system/online-boutique/protos/demo.proto:15`); `go_package` is a code-generation option, not proof of any service's implementation language (`system/online-boutique/protos/demo.proto:19`).

| Name | Technology/language | Responsibility | Evidence (file path and symbol) |
| --- | --- | --- | --- |
| CartService | proto3 RPC contract; implementation unknown | Declare adding an item, retrieving a user's cart and emptying it. | `system/online-boutique/protos/demo.proto:23–27` — `CartService.AddItem`, `GetCart`, `EmptyCart`; user association in `AddItemRequest`, `GetCartRequest`, `EmptyCartRequest` (lines 34–45). |
| RecommendationService | proto3 RPC contract; implementation unknown | Declare listing recommended product IDs using a user ID and product IDs as inputs. | `system/online-boutique/protos/demo.proto:56–67` — `RecommendationService.ListRecommendations`, `ListRecommendationsRequest`, `ListRecommendationsResponse`. |
| ProductCatalogService | proto3 RPC contract; implementation unknown | Declare listing, retrieving and searching products. | `system/online-boutique/protos/demo.proto:71–75` — `ProductCatalogService.ListProducts`, `GetProduct`, `SearchProducts`. |
| ShippingService | proto3 RPC contract; implementation unknown | Declare quoting shipping cost and shipping an order, returning a tracking ID. | `system/online-boutique/protos/demo.proto:107–128` — `ShippingService.GetQuote`, `ShipOrder`, `GetQuoteResponse.cost_usd`, `ShipOrderResponse.tracking_id`. |
| CurrencyService | proto3 RPC contract; implementation unknown | Declare listing supported currencies and converting money to a requested currency. | `system/online-boutique/protos/demo.proto:140–143` — `CurrencyService.GetSupportedCurrencies`, `Convert`; `CurrencyConversionRequest.from/to_code` (lines 168–173). |
| PaymentService | proto3 RPC contract; implementation unknown | Declare charging an amount against supplied card information and returning a transaction ID. | `system/online-boutique/protos/demo.proto:177–195` — `PaymentService.Charge`, `ChargeRequest.amount/credit_card`, `ChargeResponse.transaction_id`. |
| EmailService | proto3 RPC contract; implementation unknown | Declare sending an order confirmation using an email address and order result. | `system/online-boutique/protos/demo.proto:199–201` — `EmailService.SendOrderConfirmation`; `SendOrderConfirmationRequest.email/order` (lines 216–219). |
| CheckoutService | proto3 RPC contract; implementation unknown | Declare placing an order using user, currency, address, email and card inputs, returning an order result. | `system/online-boutique/protos/demo.proto:224–239` — `CheckoutService.PlaceOrder`, `PlaceOrderRequest`, `PlaceOrderResponse.order`. |
| AdService | proto3 RPC contract; implementation unknown | Declare retrieving advertisements using page-context keywords. | `system/online-boutique/protos/demo.proto:243–261` — `AdService.GetAds`, `AdRequest.context_keys`, `AdResponse.ads`, `Ad`. |

Data stores, message brokers and external systems: **Not evidenced in the provided files**; no such containers are added.

## Dependencies table

Runtime caller → callee dependencies and their protocols: **Not evidenced in the provided files**. The table therefore has no dependency rows.

| Caller → callee | Protocol | Evidence (file path and symbol) |
| --- | --- | --- |

The `service` and `rpc` declarations identify offered operations, not their callers (`system/online-boutique/protos/demo.proto`: the nine service declarations listed above). Shared message types do not establish runtime calls: for example, `OrderResult` appears in `SendOrderConfirmationRequest.order` and `PlaceOrderResponse.order` (lines 216–219 and 237–239), but this does not prove that `CheckoutService` calls `EmailService`. gRPC as the actual runtime protocol is **Not evidenced in the provided files**.

## Mermaid diagram

This is a partial contract-based recovery. All nine service boxes carry the candidate-container interpretation stated above. **[Assumption]** The unconnected end-user symbol is included to satisfy the required diagram notation; the evidence establishes neither an actor nor a user-facing container or interaction. No runtime relationships are drawn because none are evidenced.

```mermaid
C4Container
  title Online Boutique — Run A: service contracts; container mapping unverified
  Person(user, "[Assumption] End user", "Required notation placeholder; interaction not evidenced")
  System_Boundary(boutique, "Online Boutique") {
    Container(cart, "CartService", "proto3 RPC contract; implementation unknown", "Add items, retrieve cart and empty cart")
    Container(recommendation, "RecommendationService", "proto3 RPC contract; implementation unknown", "List recommended product IDs")
    Container(catalog, "ProductCatalogService", "proto3 RPC contract; implementation unknown", "List, retrieve and search products")
    Container(shipping, "ShippingService", "proto3 RPC contract; implementation unknown", "Quote shipping and ship orders")
    Container(currency, "CurrencyService", "proto3 RPC contract; implementation unknown", "List currencies and convert money")
    Container(payment, "PaymentService", "proto3 RPC contract; implementation unknown", "Charge a card and return a transaction ID")
    Container(email, "EmailService", "proto3 RPC contract; implementation unknown", "Send order confirmation")
    Container(checkout, "CheckoutService", "proto3 RPC contract; implementation unknown", "Place an order and return its result")
    Container(ads, "AdService", "proto3 RPC contract; implementation unknown", "Retrieve ads using context keywords")
  }
  UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Gaps

Each item below is **Not evidenced in the provided files**. Suggested paths are requests for additional evidence only; these files were not opened and their existence has not been verified.

| Gap | Extra file that would help settle it |
| --- | --- |
| Mapping of declared interfaces to running containers, runtime names, endpoints and ports. | A deployment manifest for each service, e.g. `system/online-boutique/kubernetes-manifests/checkoutservice.yaml`, together with the server registration in `system/online-boutique/src/checkoutservice/main.go`. |
| Actual caller → callee edges, including whether order placement invokes payment, shipping, cart or email operations. | The caller implementation, e.g. `system/online-boutique/src/checkoutservice/main.go`; inspect client creation and RPC call sites. Shared request/response types alone do not settle this. |
| Runtime protocol, transport security and synchronous/asynchronous execution behavior. | Client and server transport configuration, e.g. `system/online-boutique/src/checkoutservice/main.go`; follow any referenced configuration in a later, explicitly permitted evidence set. |
| Implementation languages and frameworks for the nine services. | Each service's implementation and build file, e.g. `system/online-boutique/src/checkoutservice/main.go` and `system/online-boutique/src/checkoutservice/Dockerfile`; the `go_package` option alone is insufficient. |
| End-user role, UI/entrypoint container and user-facing protocol. The reference to a current page in `AdRequest.context_keys` (`system/online-boutique/protos/demo.proto:247–249`) does not identify a frontend. | A candidate entrypoint implementation and manifest, e.g. `system/online-boutique/src/frontend/main.go` and `system/online-boutique/kubernetes-manifests/frontend.yaml`. |
| Data stores, persistence mechanisms, message brokers and external integrations, if any. | Service storage/integration implementations and their manifests; for cart storage, a candidate starting file is `system/online-boutique/kubernetes-manifests/cartservice.yaml`. Configuration must be checked against implementation before asserting a dependency. |
| Rationale for service boundaries, protocol choices or persistence choices. | An architecture decision record or design document explaining the decision; no such file path is identified by the supplied evidence. |
