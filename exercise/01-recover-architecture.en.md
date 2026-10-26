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
| checkoutservice → paymentservice | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR`) | ✅ | `main.go` line 370: `NewPaymentServiceClient(...).Charge(...)` |
| checkoutservice → emailservice | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR`) | ✅ | `main.go` line 380: `SendOrderConfirmation`; failure only logs a warning (line 274) |
| checkoutservice → cartservice | `checkoutservice.yaml` (`CART_SERVICE_ADDR`) | ✅ | `main.go` line 333: `EmptyCart`, plus `GetCart` |
| frontend → adservice | `frontend.yaml` (`AD_SERVICE_ADDR`) | ✅ | `frontend/main.go` line 138 reads `AD_SERVICE_ADDR` |
| recommendationservice → productcatalogservice | `recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`) | ✅ | `recommendation_server.py` line 131 reads it and builds `ProductCatalogServiceStub` |

- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question): the source reads `COLLECTOR_SERVICE_ADDR` (`checkoutservice/main.go` line 164, `frontend/main.go` line 178) but the manifests never set it, so a telemetry collector dependency is invisible in Run B. `frontend` also depends on `shoppingassistantservice` (`SHOPPING_ASSISTANT_SERVICE_ADDR`), which Run B correctly left out because no manifest deploys it. Run B drew `loadgenerator` although `kustomization.yaml` comments it out.

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?
   A `.proto` file says what each service offers, not who calls it or where it runs. Run A could therefore not find the `frontend`, the Redis store, the ports or any caller-to-callee link except three guesses from shared message types. Run B got these from environment variables and Service definitions in the manifests.
2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?
   Run A's diagram shows no frontend, no data store and only three links, so a newcomer would think `cartservice` is stateless and that the services barely interact. Run B's missing telemetry collector is a smaller error.
