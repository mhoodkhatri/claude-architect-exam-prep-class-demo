# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-17

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you decide a fixed workflow suits your task, you still have to shape it. Three building-block patterns cover most production pipelines: prompt chaining, routing and parallelisation. Knowing which one fits a task lets you cut cost and latency and make each stage testable. The exam expects you to match the pattern to the task's structure.

## Core concepts

### Prompt chaining

A task is decomposed into a **sequence of steps**. Each Claude call processes the output of the previous one. Your code can insert **gates** between steps: programmatic checks that validate intermediate output before continuing.

- Use it when the task splits cleanly into fixed subtasks.
- It trades latency for accuracy, because each call is an easier, more focused task.
- Typical uses: draft marketing copy, then translate it; write an outline, check it against criteria, then write the document.
- Gates catch errors early. A failed gate can retry the step, stop, or route to a human.

### Routing

A **classification step** sends the input to one of several specialised downstream paths. Each path has its own prompt, tools, or even model.

- Use it when inputs fall into distinct categories that are better handled separately and classification can be done accurately.
- It allows separation of concerns: a refund prompt does not need to carry the instructions for technical support.
- It supports cost control: send easy queries to a smaller, cheaper model and hard ones to a more capable model.
- Without routing, optimising a prompt for one input type often hurts the others.
- Include a sensible **fallback route** for unclassifiable or low-confidence inputs, such as a general handler or a human.

### Parallelisation

Several LLM calls run **at the same time**, and code aggregates the results. It has two main variations:

- **Sectioning**: break a task into **independent subtasks** that run concurrently. Example: one call checks a document for policy violations while another summarises it; or each of ten files is reviewed separately.
- **Voting**: run the **same task several times** to get diverse outputs or higher confidence. Example: three reviewers independently flag vulnerable code, and you require agreement, or take the union for high-recall screening.

Use parallelisation when subtasks are independent and you want speed, or when multiple perspectives raise confidence. Each aspect gets a focused call, which usually beats one call juggling many considerations.

### Choosing between them

| Task shape | Pattern |
|---|---|
| Ordered steps, each depends on the previous output | Chaining |
| Different input types need different handling | Routing |
| Independent subtasks, or need several opinions | Parallelisation |

### Combining patterns

They compose. A support system might **route** a ticket by category, then run a **chain** (draft, policy gate, polish) for the chosen path, with **parallel** guardrail checks running alongside the main response. All are still workflows: your code controls the flow.

### Practical considerations

- **Gates in code, not prompts**: validate schemas, totals and permissions deterministically.
- **Latency**: chaining adds latency per step; parallelisation reduces wall-clock time but not total token cost (voting increases cost).
- **Error propagation**: in a chain, an early mistake flows downstream, so validate early.
- **Aggregation**: define up front how parallel results merge (concatenate, majority vote, take the strictest).
- **Evaluation**: test each stage and the router's classification accuracy separately.

## Example

An insurance claims intake pipeline combining all three patterns:

```python
import asyncio

async def handle_claim(text: str) -> dict:
    # Routing: classify, with a fallback route
    kind = await call_claude(ROUTER_PROMPT, text)          # "auto" | "property" | "other"
    handler = HANDLERS.get(kind, escalate_to_human)

    # Parallelisation (sectioning): independent checks run concurrently
    fields, fraud_flags = await asyncio.gather(
        call_claude(EXTRACT_PROMPT, text),
        call_claude(FRAUD_SCREEN_PROMPT, text),
    )

    # Prompt chaining with a gate
    if not schema_valid(fields):                           # deterministic gate
        return escalate_to_human(text, reason="bad extraction")
    draft = await call_claude(DRAFT_PROMPT, fields)        # step 2
    return await handler(draft, fraud_flags)               # step 3, path chosen by router
```

The router keeps each handler's prompt small, the two independent calls run concurrently, and the gate stops bad data before it reaches the draft step.

## Common mistakes / exam traps

- **Using a chain when steps are independent**. If subtasks do not depend on each other, run them in parallel to cut latency.
- **Routing without a fallback**. Misclassified or ambiguous inputs need a default path or human escalation.
- **Parallelising dependent steps**. If step B needs step A's output, it is a chain, not parallel work.
- **Assuming parallelisation saves cost**. It saves wall-clock time; voting actually multiplies token spend.
- **Skipping gates between chain steps**. Errors compound downstream; validate intermediate output in code.

## Check yourself

1. What is the key benefit of adding a gate between chain steps?
2. A system gets billing questions, bug reports and sales enquiries, each needing different tools. Which pattern, and why?
3. Distinguish sectioning from voting.
4. Which patterns reduce latency, and which increase it?
5. What should a router do with an input it cannot classify confidently?

<details><summary>Answers</summary>

1. It validates intermediate output in code so errors are caught early instead of propagating to later steps, and a failed gate can retry, stop or escalate.
2. Routing: classify each input and send it to a specialised path with its own prompt and tools, so each is optimised without hurting the others.
3. Sectioning splits a task into different independent subtasks run concurrently; voting runs the same task multiple times to gain confidence or diversity of output.
4. Parallelisation reduces wall-clock latency (sectioning); chaining adds latency per step in exchange for accuracy. Voting also raises cost.
5. Send it to a fallback route, such as a general handler or a human, rather than forcing a guess.

</details>

## Key takeaways

- Chaining: ordered steps with code gates. Routing: classify, then specialise. Parallelisation: independent subtasks (sectioning) or repeated runs (voting).
- Match the pattern to the task's structure; they compose within a single workflow.
- Put validation in deterministic gates, and give routers a fallback.
- Parallelisation cuts wall-clock time, not token cost.
