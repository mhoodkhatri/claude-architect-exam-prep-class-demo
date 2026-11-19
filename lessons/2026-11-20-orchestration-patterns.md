# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-20

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a task suits a workflow, you still have to shape it. Three patterns cover most production pipelines: chain steps in sequence, route inputs to specialised handlers, or run independent work in parallel. Picking the right one controls accuracy, latency and cost, and the exam often describes a scenario and asks which pattern fits.

## Core concepts

### Prompt chaining

A task is decomposed into a **fixed sequence** of LLM calls, where each call processes the output of the previous one. Between steps you can insert programmatic **gates**: code checks that validate the intermediate result before continuing.

Use it when:
- The task cleanly splits into subtasks that must happen in order (draft, then critique, then translate).
- You trade latency for accuracy: each call is an easier problem than the whole.
- Intermediate output benefits from validation (schema check, length limit, a "does this meet the criteria?" gate).

Typical examples: write an outline, check it against requirements, then write the document from the outline; extract data, validate it in code, then summarise it.

Failure mode: an error early in the chain propagates. Gates exist to stop bad output as early as possible.

### Routing

A **classification step** looks at the input and sends it to one of several specialised downstream paths, each with its own prompt, tools or even model. Routing enables **separation of concerns**: you can tune each handler without degrading the others.

Use it when:
- Inputs fall into distinct categories that need different handling (billing question vs technical bug vs refund request).
- One generic prompt would be bloated or would trade off between categories.
- Cost matters: send easy queries to a smaller, faster model and hard ones to a more capable model.

The classifier can be an LLM call, a traditional classifier or plain rules. Its accuracy is the ceiling for the whole system, so evaluate it separately, and define a sensible **fallback route** for low-confidence or unrecognised inputs (often a general handler or a human).

### Parallelisation

Multiple LLM calls run **at the same time** and their outputs are combined by code. It has two main variants:

- **Sectioning**: split a task into **independent subtasks** and run them concurrently. Example: one call screens a user message for policy violations while another answers it; or review a document's security, performance and style in three parallel calls.
- **Voting**: run the **same task several times** to get diverse outputs, then aggregate (majority vote, or flag if any call objects). Example: several independent reviews of a code change for vulnerabilities, where a finding is raised if any reviewer, or a threshold of them, flags it.

Use it when subtasks are truly independent (speed) or when you need multiple perspectives or higher confidence (reliability). Each focused call also tends to perform better than one call juggling many considerations.

Cost note: parallelisation reduces **latency**, not token spend. Voting increases spend.

### How the patterns differ

| Pattern | Control flow | Dependency between calls | Main benefit |
|---|---|---|---|
| Chaining | Sequential | Each needs the previous output | Accuracy via simpler steps and gates |
| Routing | Branch | One classifier, then one handler | Specialisation, cost control |
| Parallelisation | Fan-out and merge | Independent | Speed or confidence |

### Combining them

These are building blocks. A support system might route a ticket, run a chain inside the chosen branch, and fan out several checks in parallel inside one chain step. They are all still workflows: your code owns the control flow. Build each pattern in plain code first; measure before adding layers.

### Choosing quickly

1. Does step B need step A's output? Chain.
2. Do different input types need different handling? Route.
3. Are there independent pieces, or do you need several opinions? Parallelise.
4. Is the number of steps unknowable ahead of time? None of these; consider an orchestrator-worker design or an agent (later lessons).

## Example

A support assistant for a software product:

```python
import asyncio

async def handle_ticket(ticket: str) -> str:
    # ROUTING: classify first, with a fallback
    category = await classify(ticket)          # "billing" | "bug" | "how_to" | "other"
    handler = HANDLERS.get(category, handle_other)
    return await handler(ticket)

async def handle_bug(ticket: str) -> str:
    # PARALLELISATION (sectioning): independent lookups at once
    logs, docs, dupes = await asyncio.gather(
        search_logs(ticket),
        search_docs(ticket),
        find_duplicate_issues(ticket),
    )
    # PROMPT CHAINING: draft -> gate -> polish
    draft = await call_claude(DRAFT_PROMPT, ticket, logs, docs, dupes)
    if not passes_policy_check(draft):         # programmatic gate
        return escalate_to_human(ticket, draft)
    return await call_claude(TONE_PROMPT, draft)
```

Routing picks the path, the three searches run concurrently because none depends on another, and the reply is built in a gated chain. Every step is fixed code, so each can be tested and monitored on its own.

## Common mistakes / exam traps

- **Parallelising dependent steps.** If call B needs call A's output, it is a chain; forcing concurrency yields wrong or empty inputs.
- **Assuming parallelisation saves money.** It cuts wall-clock time; total tokens are the same, or higher with voting.
- **Routing with no fallback.** An unrecognised category must go somewhere safe, not crash or be mis-handled.
- **Skipping gates in a chain.** Without intermediate validation, one bad step silently corrupts everything downstream.
- **Confusing these patterns with agents.** All three are workflows with code-controlled flow; the model does not decide the sequence.

## Check yourself

1. Which pattern fits "summarise a contract, then translate the summary, then check the translation against a glossary"?
2. What is the difference between sectioning and voting?
3. Why evaluate a router's classifier separately?
4. Does parallelisation reduce latency, token cost, or both?
5. A pipeline sends easy FAQ questions to a small model and complex ones to a larger model. Which pattern is this, and what must it include for unknown inputs?

<details><summary>Answers</summary>

1. Prompt chaining: each step depends on the previous output, and the glossary check can be a gate.
2. Sectioning splits a task into different independent subtasks run concurrently; voting runs the same task multiple times and aggregates the results for confidence.
3. Its accuracy caps the whole system: a misrouted input gets the wrong specialised handler, so errors there cannot be fixed downstream.
4. Latency only. Total token cost stays the same, and voting increases it.
5. Routing (with model selection by difficulty). It needs a fallback route for low-confidence or unrecognised inputs, such as a general handler or a human.

</details>

## Key takeaways

- Chaining = ordered steps with gates; routing = classify then specialise; parallelisation = independent calls (sectioning) or repeated calls (voting) merged by code.
- Parallelisation improves speed or confidence, not cost.
- Always give a router a fallback, and always validate between chain steps.
- These patterns are composable workflow building blocks; the code, not the model, controls the flow.
