# port-feature — GitHub Copilot agents for IntelliJ

Port a feature from one codebase (e.g. a Kotlin service) to another (e.g. a Java 21 Spring Boot API)
through a functional spec instead of translating code.

```
SOURCE project window                 TARGET project window
─────────────────────                 ─────────────────────────────────────────────────────────
functional-analyst                    port-feature (orchestrator)
  └─► porting/<f>/functional-spec.md    ├─ conventions-profiler ─► porting/target-conventions.md
        (you review: Status REVIEWED)   ├─ target-implementer PLAN ─► implementation-plan.md  ◄ you approve
              │ copy folder             ├─ target-implementer BUILD ─► code + tests + traceability.md
              └───────────────────────► └─ parity-reviewer ─► review.md
```

The source and target live in **separate IntelliJ windows**. The implementer and reviewer run in
the target project, so the original code isn't in their workspace: they can only reproduce what
the spec says, the way this codebase does it.

## Agents

| File | Runs in | Writes |
|---|---|---|
| `functional-analyst.agent.md` | source project | `porting/<feature>/functional-spec.md` |
| `port-feature.agent.md` | target project | nothing; delegates to the three below as subagents |
| `conventions-profiler.agent.md` | target project | `porting/target-conventions.md` |
| `target-implementer.agent.md` | target project | plan, code, tests, `traceability.md` |
| `parity-reviewer.agent.md` | target project | `review.md` |

## Install

1. Update the GitHub Copilot plugin in IntelliJ. On a Copilot Business/Enterprise licence, preview
   agent features need the *Editor preview features* policy enabled by an admin; if the agent
   dropdown has no **Configure Agents...**, that is the likely cause.
2. **Target repo:** copy `..` into the repo root. Commit it if the team wants to share it.
3. **Source repo:** the analyst must be available there too. Either copy
   `functional-analyst.agent.md` into the source repo's `..` (no need to commit), or
   create it once at *Personal* scope: Copilot Chat → settings icon → **Customizations** → Personal,
   and paste the file. Personal agents are available in every project.
4. Optional: add a `model:` line to an agent (IntelliJ offers autocomplete). Use your strongest
   model for `functional-analyst` and `target-implementer`.

## Run

**1. Spec (source window).** Open Copilot Chat, select `functional-analyst`:

```
Feature: order-cancellation
Entry point: src/main/kotlin/com/acme/order/CancelOrderController.kt
```

It writes the spec, then asks you about each suspected bug (port as-is / fix / drop) and sets
`Status: REVIEWED`. Don't commit `porting/` in the source repo.

**2. Copy** `porting/order-cancellation/` to the same path in the target repo.

**3. Port (target window).** New chat, select `port-feature`:

```
Port order-cancellation. Base branch: main.
```

It stops once for plan approval, then builds, reviews (max 2 fix rounds) and reports. It does not commit.

**Open questions** found during the build go to `porting/<feature>/open-questions.md`. Copy that file
to the source window, ask `functional-analyst` to answer it, copy the updated spec back, and tell
`port-feature` to continue.

## Manual mode

If subagent delegation misbehaves, run the target-side agents yourself from the dropdown.
Start a **new chat** for each so every phase has a clean context, and the reviewer hasn't seen the build:

1. `conventions-profiler`: "Profile this repo for porting a POST endpoint with validation, persistence and a Kafka event."
2. `target-implementer`: "Mode PLAN for order-cancellation." → review the plan.
3. `target-implementer`: "Mode BUILD for order-cancellation."
4. `parity-reviewer`: "Review order-cancellation against base branch main."

## Notes

- **Review `target-conventions.md` by hand once.** It has the biggest effect on output quality and
  is reused for every later feature. Once it's right, consider referencing it from
  `..` so everyday Copilot use follows it too.
- **The firewall is strong but not absolute.** The source isn't in the target workspace, but an
  agent with terminal access could still `cat ../kotlin-service/...`. The prompts forbid it; keep an
  eye on the terminal approvals.
- **Expect approval prompts** for terminal commands; the BUILD phase runs the test command repeatedly.
- **Start with one small vertical slice** (one endpoint) before porting a larger feature.
- **Differential check.** If the source service runs locally, replay the EX-xx examples as HTTP calls
  against both services and diff the responses.
