**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:** Chatrin Yoonchalard 6631304721
**Agent(s) used:** Claude Code
**Model(s):** claude-sonnet-5-5
**Plan / tier:** Claude Pro

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code, sonnet-5-5 | `prompts/1-…` run A | `demo.proto` | Yes | 5 | Only listed gRPC services; 3 dependencies, all inferred |
| 2 | 1-B | Claude Code, sonnet-5-5 | `prompts/1-…` run B | `demo.proto`, 12 files in `kubernetes-manifests/` | Yes | 5 | Found redis-cart and 17 dependencies; flagged shoppingassistantservice as a gap |
| 3 | 2-a | Claude Code, sonnet-5-5 | `prompts/2-…` run a | template only | Yes | 5 | No numbers; listed open questions |
| 4 | 2-b | Claude Code, sonnet-5-5 | `prompts/2-…` run b | template, `scenario.md`, ADR-0001, `checkoutservice/main.go` | Yes | 5 | Used scenario numbers; chose RabbitMQ |
| 5 | 3 | Claude Code, sonnet-5-5 | `prompts/3-…` | `docs/adr/0002-…`, `docs/scenario.md` | Yes | 5 | |
| 6 | Bonus | Claude Code, sonnet-5-5 | `prompts/bonus-…` | `demo.proto`, `checkoutservice/main.go`, `frontend/main.go`, template | Yes | 5 | Rationale mostly Inferred/Assumption |
| 7 | | | | | | | |
