Agent: Claude Code, claude-sonnet-5-5

# ADR-0003: Use synchronous gRPC with one shared `demo.proto` for all service communication

- **Status:** Reconstructed
- **Date:** 2026-10-26 (reconstruction date; original decision date not evidenced)
- **Deciders:** Not evidenced in the provided files

## Context and problem

Online Boutique is split into services (`frontend`, `checkoutservice`, `cartservice`, `productcatalogservice`, `currencyservice`, `shippingservice`, `paymentservice`, `emailservice`, `recommendationservice`, `adservice`) that must call each other. The files show a single contract, `demo.proto` (package `hipstershop`), defining every service. The services are written in different languages; the evidence shows Go in `checkoutservice` and `frontend`. The original problem statement is not in the evidence files.

## Decision drivers

- One typed contract usable from several languages [Inferred]
- Simple request/response flow for a user waiting on a page or an order [Inferred]
- Tracing across calls (both services wire OpenTelemetry into gRPC)

## Considered options

The files do not record any alternatives. These are reconstructed, not found:

1. Synchronous gRPC with one shared proto (what the code does)
2. REST/JSON per service with separate API specs
3. Asynchronous messaging for part of the flow (e.g. a broker after payment)

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. gRPC + shared proto | One generated client/server contract; typed messages; OTel hooks exist | All callers need the proto; a slow or failed callee blocks the caller; one file couples every service |
| 2. REST/JSON | Easy to test by hand; no codegen | No single schema; weaker typing |
| 3. Async messaging | Caller does not wait; absorbs bursts | Extra infrastructure; eventual consistency; none is used in the evidence |

## Decision

All service-to-service calls are unary gRPC calls defined in the shared `protos/demo.proto`. The only non-gRPC entry point in the evidence is the `frontend` HTTP server for browsers. Why this option was chosen over the others is not evidenced.

## Consequences

- **Positive:** a single contract for all services (modifiability); calls are traced end to end (observability).
- **Negative:** `PlaceOrder` runs charge, ship and email in sequence, so total latency is the sum of the calls and a failure in `shipOrder` fails the order after the card was charged (availability, consistency); the email step only logs a warning on failure.
- **Risks and mitigations:** any change to `demo.proto` can affect every service [Inferred]. Mitigation is not evidenced.

## Validation

Not evidenced in the provided files.

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | One proto file defines all services | Evidenced | `protos/demo.proto`: `CartService`, `RecommendationService`, `ProductCatalogService`, `ShippingService`, `CurrencyService`, `PaymentService`, `EmailService`, `CheckoutService`, `AdService`, all in package `hipstershop` |
| 2 | `frontend` calls its backends over gRPC | Evidenced | `frontend/main.go`: `mustConnGRPC` for currency, productcatalog, cart, recommendation, shipping, checkout, ad |
| 3 | `checkoutservice` calls other services over gRPC | Evidenced | `checkoutservice/main.go`: `pb.NewPaymentServiceClient(...).Charge`, `NewEmailServiceClient`, `NewShippingServiceClient`, `NewCartServiceClient` |
| 4 | Connections are plaintext | Evidenced | `grpc.WithTransportCredentials(insecure.NewCredentials())` in both `main.go` files |
| 5 | Calls are traced with OpenTelemetry | Evidenced | `otelgrpc.NewClientHandler()` / `NewServerHandler()` in both `main.go` files |
| 6 | `PlaceOrder` is a synchronous chain: charge, then ship, then empty cart, then email | Evidenced | `checkoutservice/main.go` `PlaceOrder` |
| 7 | Email failure does not fail the order | Evidenced | `PlaceOrder`: `log.Warnf("failed to send order confirmation ...")`, response still returned |
| 8 | The browser talks to `frontend` over HTTP, not gRPC | Evidenced | `frontend/main.go`: `r.HandleFunc(... )`, `http.ListenAndServe` |
| 9 | No message broker is used | Evidenced (in these files only) | No broker client in either `main.go`; other services not read |
| 10 | Services are in several languages | Inferred | Only Go files were read; `demo.proto` has `go_package` only |
| 11 | gRPC was chosen for cross-language typed contracts | Inferred | Not stated in the evidence |
| 12 | gRPC was chosen for performance | Assumption | General knowledge about gRPC; no benchmark in the evidence |
| 13 | REST/JSON and async messaging were considered | Assumption | No record of alternatives in the evidence |
| 14 | A shared proto makes every service depend on one file | Inferred | Follows from claim 1; build setup not read |
| 15 | Total latency of `PlaceOrder` is the sum of the downstream calls | Inferred | Calls are sequential in the code; no timings in the evidence |
| 16 | A shipping failure after payment leaves a charged customer | Evidenced | `PlaceOrder` returns `Unavailable` after `chargeCard` succeeded; no refund call is present |
| 17 | The decision was made for a demo/teaching purpose | Assumption | Nothing in the evidence says so |
| 18 | Deciders and decision date | Not evidenced | Listed as such in the header |
