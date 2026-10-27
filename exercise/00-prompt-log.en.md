**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:**
Pon Kittinaraporn / 6630237121
**Agent(s) used:** e.g. GitHub Copilot (Agent mode), Gemini Code Assist, Gemini CLI, Claude Code, Codex, Cursor, Cline: …
Codex
**Model(s):** …
GPT-6.1 Sol
**Plan / tier:** e.g. Copilot Student, Copilot Free, Gemini free tier: …
Codex Chulalongkorn Access

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Codex — GPT-6 | Follow `prompts/1-recover-architecture.md`; run A; evidence: `system/online-boutique/protos/demo.proto`; record the run in this log. | `prompts/1-recover-architecture.md`<br>`system/online-boutique/protos/demo.proto`<br> |yes |3 |Not complete recover |
| 2 | 1-B | Codex / GPT-6 | Follow prompts/1-recover-architecture.md; run B; evidence: demo.proto and kubernetes-manifests/; update this log with all files read; no terminal commands. | `prompts/1-recover-architecture.md`<br>`system/online-boutique/protos/demo.proto`<br>`system/online-boutique/kubernetes-manifests/adservice.yaml`<br>`system/online-boutique/kubernetes-manifests/cartservice.yaml`<br>`system/online-boutique/kubernetes-manifests/checkoutservice.yaml`<br>`system/online-boutique/kubernetes-manifests/currencyservice.yaml`<br>`system/online-boutique/kubernetes-manifests/emailservice.yaml`<br>`system/online-boutique/kubernetes-manifests/frontend.yaml`<br>`system/online-boutique/kubernetes-manifests/kustomization.yaml`<br>`system/online-boutique/kubernetes-manifests/loadgenerator.yaml`<br>`system/online-boutique/kubernetes-manifests/paymentservice.yaml`<br>`system/online-boutique/kubernetes-manifests/productcatalogservice.yaml`<br>`system/online-boutique/kubernetes-manifests/README.md`<br>`system/online-boutique/kubernetes-manifests/recommendationservice.yaml`<br>`system/online-boutique/kubernetes-manifests/shippingservice.yaml`<br> |yes |5 |AI maybe assume that end user-> frontend |
| 3 | 2-a | Codex / GPT-6 | Follow `prompts/2-write-adr.md`; run a; context: none; draft an ADR for MailFast confirmations, loyalty points, analytics, and future post-order actions; update the prompt log with all files read. | `prompts/2-write-adr.md`<br>`docs/adr/template.md`<br> |yes |3 |Not complete |
| 4 | 2-b | Codex / GPT-6 | Follow `prompts/2-write-adr.md`; run b; use the scenario, Redis ADR and checkout code to draft an ADR for MailFast confirmations, loyalty points, analytics and future post-order actions; update this log with all files read; no terminal commands. | `prompts/2-write-adr.md`<br>`docs/adr/template.md`<br>`docs/scenario.md`<br>`docs/adr/0001-use-redis-for-cart-storage.md`<br>`system/online-boutique/src/checkoutservice/main.go`<br>|yes |4 |Almost complete adr |
| 5 | 3 | Codex / GPT-6 | Follow `prompts/3-review-adr.md`; review `docs/adr/0002-use-rabbitmq-for-post-order-actions.md`; update this log with agent/model, message summary and all files read; leave assessment fields blank; no terminal commands. | `prompts/3-review-adr.md`<br>`docs/adr/0002-use-rabbitmq-for-post-order-actions.md`<br>`docs/scenario.md` |yes |4 |good second check |
| 6 |bonus |Codex / GPT-6 | | | | | |
| 7 | | | | | | | |
