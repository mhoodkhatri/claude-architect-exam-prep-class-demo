# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-23

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a workflow beats an agent for your task, you still have to shape it. Three building-block patterns, prompt chaining, routing and parallelisation, cover most production pipelines built on Claude. Knowing which one fits a task's structure lets you raise quality, cut cost and bound latency without handing control to an autonomous loop.

## Core concepts

All three patterns are **workflows**: your code controls the flow, and each step is a focused LLM call. They are the simple building blocks that larger systems (including the orchestrator–worker designs in the next lesson) are assembled from.

### Prompt chaining

Decompose a task into a **fixed sequence of steps**, where each call consumes the previous call's output. Between steps you can insert programmatic **gates**: code checks that verify the intermediate result before continuing.

- **Use when** the task splits cleanly into ordered subtasks and you want higher accuracy by making each call easier.
- **Trade-off**: you spend extra latency (calls run serially) to gain accuracy and debuggability.
- **Typical examples**: write an outline, check it against criteria, then write the document; draft marketing copy, then translate it.
- **Gates** are the key feature. If the outline fails validation, stop or retry that step instead of passing bad input downstream.

### Routing

A **classifier step** inspects the input and directs it to one of several specialised handlers, each with its own prompt, tools or even model.

- **Use when** inputs fall into distinct categories that are better handled separately, and classification can be done accurately (by an LLM or a traditional classifier).
- **Why it helps**: optimising one prompt for all input types hurts the others. Separate prompts let each be sharp and independently testable.
- **Cost control**: route easy queries to a small, cheap model (such as Haiku) and hard ones to a more capable model.
- **Typical examples**: customer-support triage (refund, technical, billing, general), or sending simple versus complex questions to different models.
- Always include a **fallback route** for low-confidence or unrecognised inputs, often a human or a general handler.

### Parallelisation

Run multiple LLM calls **at the same time** and combine the results in code. It has two variations:

- **Sectioning**: split a task into **independent subtasks** that run concurrently. Example: one call answers the user while another screens the input for policy violations; or several calls each review a different part of a document.
- **Voting**: run the **same task several times** (possibly with different prompts) to get diverse outputs, then aggregate by majority or threshold. Example: three independent reviews of code for vulnerabilities, flagging if any or most agree.

Use parallelisation when subtasks are truly independent, so speed improves, or when you need higher confidence from multiple attempts. Cost rises with the number of calls, but latency stays near that of the slowest call.

### Choosing between them

| Task shape | Pattern |
|---|---|
| Ordered steps, each depends on the last | Chaining |
| Different input types need different handling | Routing |
| Independent pieces, or need several opinions | Parallelisation |

### Combining patterns

Patterns compose. A support system might **route** a ticket to a category, then run a **chain** (draft, policy-check, polish) for that category, with **parallel** guardrail checks running beside the draft step.

### Engineering notes

- Keep each step's prompt narrow. The reason to decompose is that simpler calls are more reliable.
- Pass structured data (JSON) between steps so gates in code can validate it.
- Handle failure per step: retry, fall back, or escalate to a human.
- Evaluate each stage separately, and measure end-to-end too. Adding steps only helps if measurements show it.

## Example

A support pipeline combining all three patterns:

```python
import asyncio

async def handle_ticket(ticket: str) -> str:
    # Routing: cheap model classifies the ticket
    route = await call_claude(CLASSIFY_PROMPT, ticket, model="haiku")
    if route not in HANDLERS:
        return escalate_to_human(ticket, reason="unrecognised category")

    # Parallelisation (sectioning): draft and safety screen run concurrently
    draft_task = call_claude(HANDLERS[route], ticket)
    screen_task = call_claude(PII_SCREEN_PROMPT, ticket)
    draft, screen = await asyncio.gather(draft_task, screen_task)

    # Gate: deterministic check in code
    if screen.get("contains_pii"):
        return escalate_to_human(ticket, reason="PII detected")

    # Prompt chaining: second step refines the first step's output
    return await call_claude(TONE_POLISH_PROMPT, draft)
```

Routing keeps each handler prompt specialised and lets simple tickets use a cheap model. Sectioning hides the screening latency behind the drafting call. The chain's gate stops bad input before the polish step.

## Common mistakes / exam traps

- **Chaining when steps are independent**: running independent subtasks serially wastes latency. Parallelise them.
- **Parallelising dependent steps**: if step B needs step A's output, it is a chain, not a parallel job.
- **Routing without a fallback**: unrecognised or low-confidence inputs need a default or human path.
- **Skipping gates in a chain**: errors in early steps compound silently through later calls.
- **Confusing sectioning with voting**: sectioning splits *different* subtasks; voting repeats the *same* task for confidence.

## Check yourself

1. What is the purpose of a gate in a prompt chain?
2. A support inbox receives refund requests, bug reports and sales questions, each needing different handling. Which pattern fits?
3. Name the two variations of parallelisation and how they differ.
4. How does routing help control cost?
5. Why might you prefer chaining over one large prompt for a document-writing task?

<details><summary>Answers</summary>

1. A gate is a programmatic check between steps that verifies the intermediate output, so bad results are stopped or retried instead of being passed downstream.
2. Routing: classify each message and send it to a specialised handler with its own prompt (and tools or model).
3. Sectioning runs different independent subtasks concurrently; voting runs the same task multiple times and aggregates the outputs for confidence.
4. Easy inputs can be sent to a small, cheap model and only hard ones to a larger, costlier model, while each handler's prompt stays focused.
5. Each call is simpler and so more accurate, intermediate results (such as an outline) can be validated, and failures are easier to locate and debug, at the cost of extra latency.

</details>

## Key takeaways

- Chaining = ordered steps with gates; routing = classify then specialise; parallelisation = concurrent calls (sectioning or voting).
- All three are workflows: code controls the flow, which keeps them predictable and testable.
- Match the pattern to the task's shape, and add steps only when measurements show a benefit.
- Patterns compose, and routing should always have a fallback path.
