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
| 1 | One proto file defines all services | Evidenced | Evidenced | `demo.proto` has the service blocks, all in package `hipstershop` |
| 2 | Calls are traced with OpenTelemetry | Evidenced | Evidenced | `otelgrpc.NewClientHandler()` in `frontend/main.go` line 230 and `checkoutservice/main.go` line 216 |
| 3 | Email failure does not fail the order | Evidenced | Evidenced | `checkoutservice/main.go` `PlaceOrder`: `log.Warnf` then returns the response |
| 4 | Services are in several languages | Inferred | Inferred | Both files read are Go; the claim needs other `src/` folders (e.g. `.py`, `.js` files exist there) |
| 5 | gRPC chosen for cross-language typed contracts | Inferred | Inferred | No rationale text anywhere in the three files; a guess |

**Count:** of all the agent's claims, how many are *Evidenced* according to the agent? 9 According to you? 8 (claim 9 is only true for the files read)

## Reflect

1. Did the agent ever label a guess as *Evidenced*? Give an example, or explain how you checked that it didn't.
2. A new developer reads only this reconstructed ADR. What wrong belief could they form?
3. What can AI recover from code, and what can only the original architects know? Use one example from your ADR.

**Answers**

1. No claim was mislabelled as *Evidenced*, except claim 9 ("no message broker"), which only holds for the two `main.go` files and the proto; I rated it as limited to those files. I checked each Evidenced row against the cited line.
2. They might believe the team deliberately weighed REST and messaging and picked gRPC for performance. Neither is in the code; options and drivers are reconstructed.
3. Code shows what is connected and how (gRPC, plaintext, sequential `PlaceOrder`). Only the original architects know why, what was rejected, and when. Example: the ADR cannot say whether not refunding after a shipping failure was a conscious trade-off.
