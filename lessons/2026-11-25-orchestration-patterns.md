# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-25

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a task suits a workflow, you still have to choose its shape. Three building blocks cover most production pipelines: prompt chaining, routing and parallelisation. Picking the right one controls accuracy, latency and cost, and the exam will describe a scenario and ask which pattern fits.

## Core concepts

### Prompt chaining

A task is decomposed into a **sequence of steps**, where each LLM call processes the output of the previous one. Your code can insert **gates** between steps: programmatic checks that validate intermediate output before continuing.

- **Use when** the task splits cleanly into fixed subtasks and you want higher accuracy by making each call simpler, trading some latency for it.
- **Typical shapes**: write an outline, check it against criteria, then write the document; generate copy, then translate it; extract, validate, then summarise.
- **Benefits**: each step has a focused prompt, intermediate results are inspectable, and a failed gate stops bad data early instead of letting errors compound.
- **Cost**: latency adds up because steps run one after another.

### Routing

An input is **classified**, then sent to a specialised downstream prompt, tool set or model. This is a separation of concerns: each branch can be tuned without hurting the others.

- **Use when** inputs fall into distinct categories that need different handling, and classification can be done accurately (by an LLM or a traditional classifier).
- **Typical shapes**: customer support triage (billing, technical, refund), sending easy questions to a small, fast model and hard ones to a larger model, choosing which knowledge base to search.
- **Benefits**: optimising for one category does not degrade another, and you can control cost by matching model size to difficulty.
- **Risks**: a misclassification sends the input down the wrong path, so give the router a fallback or "other/unclear" route and evaluate the classifier on its own.

### Parallelisation

Independent work runs **at the same time**, and the results are combined by code. There are two variants:

- **Sectioning**: split a task into independent subtasks and run them concurrently. Example: one call reviews code for security, another for performance, another for style. Also used for guardrails, where one call answers the user while another screens the request.
- **Voting**: run the **same task several times** and aggregate (majority vote, or any-flag-wins) for higher confidence. Example: three independent reviews of whether a snippet is vulnerable, flagged if two agree.

- **Use when** subtasks are independent and speed matters, or when several perspectives or attempts raise confidence.
- **Cost**: more total tokens (voting multiplies them) in exchange for lower wall-clock time or higher reliability.
- **Requirement**: the parts must not depend on each other's output. If step B needs A's result, that is a chain, not parallelisation.

### Choosing between them

| Situation | Pattern |
|---|---|
| Fixed sequence, each step needs the previous output | Chaining |
| Different input types need different handling | Routing |
| Independent subtasks, want lower latency | Parallelisation (sectioning) |
| One high-stakes judgement, want confidence | Parallelisation (voting) |

### Combining patterns

They compose. A common design is a router that picks a branch, and each branch is a chain whose middle step fans out in parallel. All of these remain **workflows**: your code, not the model, decides the control flow. Only when the number of subtasks cannot be known in advance do you need the orchestrator-worker design (next lesson).

### Implementation notes

- Keep gates in **code** where possible (schema checks, length limits, arithmetic).
- Use structured output at routing points so the branch decision is machine-readable.
- Run parallel calls with async concurrency, and decide ahead of time how to handle one branch failing (retry, skip or fail the whole run).
- Evaluate each stage separately, so you know which part to improve.

## Example

A support system for a software company: route, then chain, with a parallel guardrail.

```python
import asyncio

async def handle_ticket(ticket: str) -> str:
    # Parallel sectioning: guardrail screens while the router classifies
    safe, category = await asyncio.gather(
        screen_for_abuse(ticket),           # call 1: yes/no
        classify(ticket),                   # call 2: "billing" | "bug" | "how_to" | "unclear"
    )
    if not safe:
        return escalate_to_human(ticket, reason="flagged")

    # Routing: each branch has its own prompt and model
    if category == "unclear":
        return ask_clarifying_question(ticket)
    handler = {"billing": billing_chain, "bug": bug_chain, "how_to": howto_chain}[category]
    return await handler(ticket)

async def bug_chain(ticket: str) -> str:
    # Prompt chaining with a gate
    repro = await call_claude(EXTRACT_REPRO_PROMPT, ticket)
    if not has_required_fields(repro):      # programmatic gate
        return ask_for_missing_details(repro)
    draft = await call_claude(DRAFT_REPLY_PROMPT, repro)
    return await call_claude(TONE_CHECK_PROMPT, draft)
```

All three patterns appear, and the control flow is entirely in code.

## Common mistakes / exam traps

- **Parallelising dependent steps.** If one call needs another's output, they must be chained.
- **Routing without a fallback.** Inputs that fit no category need an "unclear" path or human escalation.
- **Confusing sectioning with voting.** Sectioning runs different subtasks; voting runs the same task repeatedly.
- **Chaining when one call is enough.** Extra steps add latency and cost; chain only if it measurably improves quality.
- **Calling a dynamic-subtask design "parallelisation".** If the model decides at runtime how many subtasks exist, that is orchestrator-worker, not a fixed parallel workflow.

## Check yourself

1. What is a "gate" in prompt chaining?
2. A product sends simple FAQs to a small model and complex complaints to a large one. Which pattern is this?
3. What is the difference between sectioning and voting?
4. Name one cost of parallelisation and one cost of chaining.
5. Why is a router with no "unclear" category risky?

<details><summary>Answers</summary>

1. A programmatic check between steps that validates intermediate output and stops or redirects the pipeline if it fails.
2. Routing: the input is classified and sent to a specialised handler (here, differing by model size).
3. Sectioning splits a task into different independent subtasks run concurrently; voting runs the same task several times and aggregates the results for confidence.
4. Parallelisation uses more total tokens/cost (especially voting); chaining adds latency because steps run sequentially.
5. Ambiguous or out-of-scope inputs are forced into the wrong category and handled incorrectly, with no way to ask for clarification or escalate.

</details>

## Key takeaways

- Chaining: sequential steps with gates, trading latency for accuracy.
- Routing: classify, then specialise, with a fallback route.
- Parallelisation: sectioning for independent subtasks, voting for confidence; parts must be independent.
- The patterns compose, and all are workflows where code controls the flow.
