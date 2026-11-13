# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-14

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a task suits a workflow, you still need to choose its shape. Prompt chaining, routing and parallelisation are the three building blocks behind most production Claude pipelines, and each solves a different problem: accuracy through decomposition, specialisation by input type, and speed or confidence through concurrency. The exam expects you to match the pattern to the task's structure and to explain the trade-offs.

## Core concepts

### Prompt chaining

A task is split into a **sequence of steps**, where each LLM call consumes the output of the previous one. Code between the steps can add **gates**: deterministic checks that stop, retry or reroute the flow.

- **Use when** the task decomposes cleanly into fixed subtasks and you want higher accuracy by making each call simpler.
- **Trade-off**: you pay extra latency (calls are sequential) for better focus per step.
- **Typical examples**: write an outline, check it against criteria, then write the document; extract data, validate, then translate.
- **Gates matter**: validating the output of step N before step N+1 catches errors early instead of compounding them.

### Routing

An initial step **classifies the input** and sends it to a specialised downstream path: a different prompt, tool set, or even a different model.

- **Use when** there are distinct categories that are better handled separately, and classification can be done accurately (by an LLM or a traditional classifier).
- **Benefits**: separation of concerns, prompts that stay short and focused, and cost control, for example sending easy queries to a small, fast model and hard ones to a larger model.
- **Typical examples**: customer support triage (billing, technical, refund), or model selection by difficulty.
- **Without routing**, optimising a single prompt for one category often hurts performance on the others.
- **Always include a fallback route** for low-confidence or unrecognised inputs, such as a general handler or a human.

### Parallelisation

Multiple LLM calls run **at the same time**, and code aggregates the results. There are two main variants:

- **Sectioning**: split a task into **independent subtasks** run concurrently. Example: one call screens a user message for policy violations while another answers it; or separate calls review a document for security, style and performance.
- **Voting**: run the **same task several times** (possibly with different prompts) and combine the outputs by majority or threshold. Example: several independent reviews of code for vulnerabilities, flagging if any or most agree.

Use parallelisation when subtasks are independent (so latency drops) or when you need multiple perspectives for higher confidence. It does not help when step B needs step A's output; that is a chain.

### Choosing between them

| Situation | Pattern |
|---|---|
| Fixed ordered steps, each depends on the last | Chaining |
| Inputs fall into distinct categories needing different handling | Routing |
| Independent subtasks, or need several opinions | Parallelisation |

### Combining patterns

The patterns compose. A common design: **route** the request, then run a **chain** on the selected path, with a **parallel** step for independent checks. Because they are all predefined code paths, the whole system is still a workflow and remains testable stage by stage.

### Practical engineering notes

- **Structured hand-offs**: pass JSON between steps, validated in code, rather than free text.
- **Per-stage evaluation**: each stage has its own test set, which makes regressions easy to localise.
- **Concurrency limits**: parallel calls consume rate limit faster; handle 429s with backoff.
- **Aggregation is code**: merging, voting and thresholds should be deterministic where possible.
- **Error handling**: decide per stage whether to retry, fall back or escalate to a human.

## Example

A support system that routes tickets, then chains a draft and a check, with a parallel safety screen.

```python
import asyncio

async def handle_ticket(ticket: str) -> dict:
    # Routing: cheap model classifies the ticket
    category = await call_claude(ROUTER_PROMPT, ticket, model="small")
    if category not in {"billing", "technical", "refund"}:
        return escalate_to_human(ticket, reason="unrecognised category")

    # Parallelisation (sectioning): independent calls at the same time
    draft_task = call_claude(PROMPTS[category], ticket, model="large")
    safety_task = call_claude(SAFETY_PROMPT, ticket, model="small")
    draft, safety = await asyncio.gather(draft_task, safety_task)

    if safety["flagged"]:
        return escalate_to_human(ticket, reason="safety screen")

    # Chaining with a gate: check the draft before sending
    review = await call_claude(REVIEW_PROMPT, {"ticket": ticket, "draft": draft})
    if not review["approved"]:
        draft = await call_claude(REVISE_PROMPT, {"draft": draft, "notes": review["notes"]})
    return {"category": category, "reply": draft}
```

Routing sends work to the right prompt and model, parallelisation hides the safety-check latency, and the chain adds a quality gate. Every path is fixed in code.

## Common mistakes / exam traps

- **Parallelising dependent steps**: if step B needs A's output, concurrency is impossible; it is a chain.
- **Routing without a fallback**: unrecognised or low-confidence inputs get silently mishandled.
- **Chaining with no gates**: errors from an early step flow through and compound.
- **Confusing sectioning with voting**: sectioning splits *different* subtasks; voting repeats the *same* task for confidence.
- **Calling a composed system an agent**: if code defines every path, it is still a workflow.

## Check yourself

1. What is the main trade-off of prompt chaining?
2. Why would you add a router in front of specialised prompts?
3. What is the difference between sectioning and voting?
4. A pipeline runs three independent document reviews one after another and is too slow. Which pattern fixes this?
5. Your router receives an input that fits no category. What should the design do?

<details><summary>Answers</summary>

1. It trades extra latency (sequential calls) for higher accuracy, since each call handles a simpler subtask and gates can validate intermediate output.
2. It separates concerns so each prompt stays focused, avoids one prompt being tuned at the expense of other categories, and lets you use cheaper models for easy inputs.
3. Sectioning runs different, independent subtasks concurrently; voting runs the same task multiple times and aggregates results for confidence.
4. Parallelisation (sectioning): run the independent reviews concurrently and aggregate the results.
5. Send it to a fallback route, such as a general handler or a human, rather than forcing it into a wrong category.

</details>

## Key takeaways

- Chaining: ordered steps with gates, trading latency for accuracy.
- Routing: classify, then specialise by prompt, tools or model, with a fallback.
- Parallelisation: sectioning for independent subtasks, voting for confidence.
- The patterns compose, and a system built only from predefined paths is still a workflow.
