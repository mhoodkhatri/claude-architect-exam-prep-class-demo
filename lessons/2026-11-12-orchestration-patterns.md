# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-12

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a task suits a workflow, you still have to shape it. Three building blocks cover most production pipelines: prompt chaining, routing and parallelisation. Knowing which one fits a task lets you cut latency, raise accuracy and keep each stage testable, and the exam often describes a scenario and asks you to name the right pattern.

## Core concepts

### Prompt chaining

A task is split into a **fixed sequence of steps**. Each Claude call processes the output of the previous one, and your code can insert **gates** (programmatic checks) between steps.

- Use it when the task decomposes cleanly into subtasks and you want to trade some latency for higher accuracy, because each call is easier than one giant call.
- Typical shapes: write an outline, check it against criteria, then write the document; draft copy, then translate it; extract, validate, then summarise.
- Gates are the key feature. A deterministic check after a step (schema validation, length, a required field) lets you stop, retry or route to a human early instead of passing bad data downstream.
- Each stage can have its own prompt, its own evaluation set and even its own model size.

### Routing

An **input is classified and sent to a specialised downstream path**. The classifier may be an LLM call or ordinary code (rules, keywords, metadata).

- Use it when there are distinct categories that are better handled separately and classification can be done accurately.
- It enables **separation of concerns**: a focused prompt per category instead of one prompt full of special cases that hurts every category.
- It enables **cost control**: send easy queries to a small, fast model and hard ones to a larger model.
- Typical shapes: support triage (refund, technical, billing), question difficulty routing, selecting a tool set or a prompt template.
- Include a sensible **fallback route** for low-confidence or unknown classes, often a human or a general handler.

### Parallelisation

Independent work runs **at the same time**, and the results are aggregated by code. It has two variants:

- **Sectioning**: split a task into independent subtasks run concurrently. Example: one call checks a document for policy violations while another answers the user's question; or analyse each section of a long report separately.
- **Voting**: run the **same task several times** and combine the answers (majority vote, or any-flag-wins) to get higher confidence. Example: several reviewers each check code for vulnerabilities, and a finding is accepted if enough agree.

Use parallelisation when subtasks are genuinely independent, so it cuts wall-clock latency, or when multiple perspectives raise confidence on high-stakes decisions. Remember it **does not reduce cost**: you pay for every call, and voting multiplies cost.

### Choosing between them

| Situation | Pattern |
|---|---|
| Fixed ordered steps, each depends on the last | Chaining |
| Different kinds of input need different handling | Routing |
| Independent subtasks, want lower latency | Parallelisation (sectioning) |
| Need higher confidence on one judgement | Parallelisation (voting) |

### Combining patterns

They compose. A realistic system routes a request, runs a chain on the chosen path, and fans out a parallel step inside the chain. Dependencies decide the structure: if step B needs A's output, it is a chain; if not, it can run in parallel.

### What these are not

All three are **workflows**: your code defines the paths. None needs the model to decide its own next step. When the number of subtasks cannot be known in advance, you need the orchestrator–worker pattern (next lesson).

## Example

A support assistant for a software company:

```python
import asyncio

async def handle_ticket(ticket: str) -> str:
    # Routing: cheap model classifies the ticket
    category = await classify(ticket, model="small")      # billing | bug | how_to | other
    if category == "other":
        return escalate_to_human(ticket)                  # fallback route

    # Parallelisation (sectioning): independent checks run together
    safety, context = await asyncio.gather(
        check_for_pii_and_abuse(ticket),                  # guardrail call
        retrieve_docs(ticket, category),                  # retrieval for the chosen path
    )
    if not safety.ok:
        return escalate_to_human(ticket)

    # Prompt chaining with a gate
    draft = await write_reply(ticket, context, prompt=PROMPTS[category])
    if not passes_tone_and_policy_check(draft):           # deterministic gate
        draft = await revise_reply(draft)
    return draft
```

Routing picks the path, sectioning cuts latency, and the chain with a gate protects quality.

## Common mistakes / exam traps

- **Parallelising dependent steps.** If step B needs A's output, they cannot run concurrently; that is a chain.
- **Assuming parallelisation saves money.** It saves wall-clock time (sectioning) or adds confidence (voting), but total token cost stays the same or rises.
- **Routing without a fallback.** Unknown or low-confidence inputs need a default or human path, or they get misrouted silently.
- **Skipping gates in a chain.** Without checks between steps, an early error propagates and is amplified by later calls.
- **Confusing these patterns with agents.** They are fixed code paths; if the model chooses the next step itself, it is no longer one of these workflows.

## Check yourself

1. Which pattern fits "summarise a contract, then check the summary against a checklist, then draft a cover email"?
2. Why might you route easy questions to a smaller model?
3. What is the difference between sectioning and voting?
4. What is a gate in a prompt chain, and why is it useful?
5. A teammate says parallelising three calls will cut your token bill by two thirds. Are they right?

<details><summary>Answers</summary>

1. Prompt chaining: ordered steps where each uses the previous output, ideally with a gate after the summary.
2. It lowers cost and latency for simple inputs while reserving the larger model for hard ones, and each route can use a focused prompt.
3. Sectioning runs different independent subtasks concurrently; voting runs the same task multiple times and aggregates for confidence.
4. A programmatic check between steps (schema, policy, required fields) that stops, retries or escalates early, preventing bad output from flowing downstream.
5. No. Parallelisation reduces wall-clock latency, not token cost; you still pay for every call.

</details>

## Key takeaways

- Chaining = fixed sequence with gates; routing = classify then specialise; parallelisation = sectioning or voting.
- Pick by dependency and variety: dependent steps chain, varied inputs route, independent steps parallelise.
- Parallelisation buys speed or confidence, not lower cost.
- Always give routing a fallback and chains gates; combine patterns freely, but keep them as fixed code paths.
