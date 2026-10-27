**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture (approximately 15 min)

> **Lecture link:** T7 architecture recovery, AICH2 (traceability), AICH3 (context), and the JSS finding that 93% of studies never validate AI output.
> **Question:** how much of an architecture can AI recover, and how much does it depend on the evidence it reads?

**Don't look at the original Online Boutique repository during this task:** its README shows the architecture diagram you are recovering.

For each run, start a **new chat or session** in your agent and send the message below. The agent follows [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) and writes its result to `exercise/outputs/1<run>-architecture.md`.

**Run A: interfaces only**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interfaces + deployment config**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

While the agent works, watch which files it opens. If it reads files outside the evidence you allowed, write that down in your prompt log.

## Results

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

Open each file with **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) to see the diagram.

## Verify Run B yourself (the important part)

Check **at least five** dependencies in Run B's table against the files. Open the cited file and find the evidence. You may also check the service's source code, e.g. `src/checkoutservice/main.go`.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What you found |
|---|---|---|---|
|End user → frontend |system/online-boutique/kubernetes-manifests/frontend.yaml (frontend-external LoadBalancer; app: frontend selector; http port 80 to 8080, lines 123–136); the human caller is [Inferred] |❓ |External access to frontend is configured, but a human end user is not directly evidenced in this manifest. |
|frontend → productcatalogservice |system/online-boutique/kubernetes-manifests/frontend.yaml (PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550, lines 69–70); system/online-boutique/kubernetes-manifests/productcatalogservice.yaml (Service port named grpc, lines 79–81); system/online-boutique/protos/demo.proto (ProductCatalogService) |✅ |productcatalogservice	I found PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550 in frontend.yaml lines 69–70. This matches the Service port named grpc on 3550 in productcatalogservice.yaml lines 79–81, supporting the configured dependency. |
|frontend → currencyservice |system/online-boutique/kubernetes-manifests/frontend.yaml (CURRENCY_SERVICE_ADDR=currencyservice:7000, lines 71–72); system/online-boutique/kubernetes-manifests/currencyservice.yaml (Service port named grpc, lines 80–82); system/online-boutique/protos/demo.proto (CurrencyService) |✅ |currencyservice	I found CURRENCY_SERVICE_ADDR=currencyservice:7000 in frontend.yaml lines 71–72. This matches the Service port named grpc on 7000 in currencyservice.yaml lines 80–82, supporting the configured dependency. |
|frontend → cartservice |system/online-boutique/kubernetes-manifests/frontend.yaml (CART_SERVICE_ADDR=cartservice:7070, lines 73–74); system/online-boutique/kubernetes-manifests/cartservice.yaml (Service port named grpc, lines 80–82); system/online-boutique/protos/demo.proto (CartService) |✅ |I found CART_SERVICE_ADDR=cartservice:7070 in frontend.yaml lines 73–74. This matches the Service port named grpc on 7070 in cartservice.yaml lines 80–82, supporting the configured dependency. |
|frontend → recommendationservice |system/online-boutique/kubernetes-manifests/frontend.yaml (RECOMMENDATION_SERVICE_ADDR=recommendationservice:8080, lines 75–76); system/online-boutique/kubernetes-manifests/recommendationservice.yaml (Service port named grpc, lines 83–85); system/online-boutique/protos/demo.proto (RecommendationService) |✅ |recommendationservice	I found RECOMMENDATION_SERVICE_ADDR=recommendationservice:8080 in frontend.yaml lines 75–76. This matches the Service port named grpc on 8080 in recommendationservice.yaml lines 83–85, supporting the configured dependency. |

- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question): I found suspected wrong external access to frontend is configured, but a human end user is not directly evidenced in this manifest.

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?
Run B provides a more complete architecture because it includes deployment configuration as well as API contracts. From system/online-boutique/protos/demo.proto alone,
2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?
I did not find an incorrect configured service dependency among the rows I checked. A potentially misleading mistake would be presenting the End user → frontend relationship as directly verified, because system/online-boutique/kubernetes-manifests/frontend.yaml lines 123–136 establishes external HTTP exposure but does not identify a human caller.
