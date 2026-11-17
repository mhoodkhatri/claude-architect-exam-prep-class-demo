# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-18

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you decide a workflow fits the task, you still have to choose its shape. Three patterns cover most production pipelines: chaining steps in sequence, routing inputs to specialised handlers, and running independent work in parallel. Knowing which one matches a given requirement (accuracy, cost, latency, coverage) is a staple of exam scenarios and of real system design.

## Core concepts

### Prompt chaining

A task is decomposed into a **fixed sequence of LLM calls**, where each call processes the output of the previous one. Code between steps can add **gates**: deterministic checks that stop, retry or divert the flow.

- **Use when** the task splits cleanly into subtasks and you want higher accuracy by making each call easier (for example, write an outline, check it against criteria, then write the document).
- **Trade-off**: you accept extra latency (calls are sequential) in return for focus and checkability at every stage.
- **Gates** are the key feature: validate JSON, check a length limit, confirm required fields exist. A failed gate can trigger a retry or an escalation instead of passing bad data downstream.
- Each link can have its own prompt, its own model and its own evaluation set.

### Routing

An **initial classification step** sends the input to one of several specialised downstream paths (prompts, tools, or models). It enables separation of concerns: each handler is tuned for one category without hurting the others.

- **Use when** inputs fall into distinct categories that are better handled separately, and classification can be done accurately (by an LLM or a traditional classifier).
- Typical uses: sending refund, technical and billing questions to different prompts and tool sets; sending easy questions to a small, fast model and hard ones to a larger model to control cost.
- Design details: include a **fallback or "other" route**, keep the router's output constrained (an enum, not free text), and log routing decisions so misroutes can be measured.
- Routing quality is the ceiling for the whole system. Test the router on its own.

### Parallelisation

Independent LLM calls run **at the same time**, and code aggregates the results. It has two main variations:

- **Sectioning**: split a task into independent subtasks that run concurrently (for example, review a document's security, style and performance aspects with separate prompts, or process each chunk of a long file). Benefit: lower latency and focused attention per call.
- **Voting**: run the **same task several times** and compare or combine outputs to get higher confidence (for example, several reviews of code for vulnerabilities, with a flag raised if any or a majority finds an issue). Benefit: robustness on high-stakes judgements, at multiplied cost.

Requirements: the subtasks must be **independent** (no call needs another's output), and you need an aggregation rule and a plan for partial failure (if one branch times out, do you proceed or fail?).

### Choosing and combining

| Need | Pattern |
|---|---|
| Steps depend on each other; accuracy over speed | Chaining |
| Different input types need different handling | Routing |
| Independent subtasks; reduce wall-clock time | Parallelisation (sectioning) |
| Higher confidence on a risky judgement | Parallelisation (voting) |

Patterns compose. A common shape is route → chain → parallel fan-out → aggregate. All three remain **workflows**: your code, not the model, controls the flow.

### Practical considerations

- Pass **structured data** between stages, not loose prose, so gates and aggregators can parse it reliably.
- Keep each stage's prompt narrow. If a stage keeps failing, split it further.
- Set timeouts and concurrency limits on parallel calls to respect rate limits.
- Measure each pattern against the single-call baseline; use it only if it wins.

## Example

A support-ticket system that combines all three patterns:

```python
import asyncio

async def handle_ticket(ticket: str) -> dict:
    # Routing: constrained classification
    route = await call_claude(ROUTER_PROMPT, ticket, model="small")  # "billing" | "technical" | "other"
    if route not in HANDLERS:
        return escalate_to_human(ticket, reason="unroutable")

    # Chaining with a gate
    draft = await call_claude(HANDLERS[route], ticket)
    if not passes_policy_checks(draft):          # deterministic gate
        draft = await call_claude(REVISE_PROMPT, draft)

    # Parallelisation (sectioning): independent checks at once
    tone, accuracy, pii = await asyncio.gather(
        call_claude(TONE_CHECK, draft),
        call_claude(FACT_CHECK, draft, context=ticket),
        call_claude(PII_CHECK, draft),
    )
    if any(r["fail"] for r in (tone, accuracy, pii)):
        return escalate_to_human(ticket, reason="review failed")
    return {"reply": draft}
```

The router is cheap and constrained, the chain has a code gate, and the three reviews are independent so they run concurrently. Total latency is about three sequential steps rather than five.

## Common mistakes / exam traps

- **Parallelising dependent steps**. If call B needs A's output, it is a chain, not a fan-out.
- **Confusing sectioning with voting**. Sectioning splits different subtasks; voting repeats the same task for confidence.
- **Free-text routing output** with no fallback route, so unexpected labels crash the pipeline or silently misroute.
- **Chaining without gates**. Errors then propagate and compound through every later stage.
- **Calling these patterns "agents"**. They are workflows because predefined code controls the flow.

## Check yourself

1. Which pattern would you use to send simple questions to a small model and complex ones to a large model?
2. What is the purpose of a gate in a prompt chain?
3. What is the difference between sectioning and voting?
4. What condition must hold for subtasks to run in parallel?
5. Why is the router's accuracy so important?

<details><summary>Answers</summary>

1. Routing: a classification step selects the downstream model or handler.
2. A gate is a programmatic check between steps that verifies intermediate output and stops, retries or diverts before errors propagate.
3. Sectioning runs different, independent subtasks concurrently to save time and focus each call; voting runs the same task multiple times and combines results for confidence.
4. They must be independent: no subtask may depend on another's output.
5. Every input passes through it, so a misroute sends the input to a handler tuned for the wrong category and caps overall quality. It should be tested on its own.

</details>

## Key takeaways

- Chaining: sequential steps with gates, trading latency for accuracy and checkability.
- Routing: classify first, then specialise, with constrained output and a fallback route.
- Parallelisation: sectioning for speed on independent subtasks, voting for confidence on risky judgements.
- Patterns combine freely and remain workflows because your code controls the flow.
- Justify each pattern against a single-call baseline.
