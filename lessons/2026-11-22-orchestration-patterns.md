# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-22

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a task suits a workflow, you still have to shape it. Three building blocks cover most production pipelines: chaining steps in sequence, routing inputs to specialised handlers, and running independent work in parallel. Knowing which pattern fits which task shape is a core exam skill, and it determines the accuracy, latency and cost of the system you ship.

## Core concepts

### Prompt chaining

A task is decomposed into a **fixed sequence of steps**, where each Claude call processes the output of the previous one. Code between the steps can add **gates**: programmatic checks that validate intermediate output before continuing.

- **Use when** the task cleanly splits into subtasks and you want higher accuracy by making each call simpler, trading some latency for it.
- **Typical examples**: write marketing copy, then translate it; draft an outline, check it against criteria, then write the document from the approved outline.
- **Why it works**: each call has one narrow job, so prompts are shorter and clearer, and each stage can be tested and debugged on its own.
- **Gates** catch errors early. If the outline fails validation, stop or retry before spending tokens on the full draft.

### Routing

A first step **classifies the input** and sends it to one of several specialised follow-up paths, each with its own prompt, tools or even model.

- **Use when** inputs fall into distinct categories that are better handled separately, and classification can be done accurately (by an LLM or a traditional classifier).
- **Typical examples**: customer support triage (refund, technical issue, general question); sending easy questions to a small, cheap model and hard ones to a larger model.
- **Why it works**: it is separation of concerns. Specialised prompts do not have to compromise to cover every case, and optimising for one category does not hurt another.
- **Key risk**: a misrouted input gets the wrong handler. Include a fallback or "other" route, and evaluate the router on its own test set.

### Parallelisation

Independent subtasks run **at the same time**, and code aggregates the results. It has two main variations:

- **Sectioning**: split a task into different independent subtasks run in parallel. Example: one call answers the user while another screens the request for policy violations; or several calls each review a different aspect of a document.
- **Voting**: run the **same task several times** to get diverse outputs, then aggregate (majority vote, or a stricter threshold). Example: multiple independent reviews of code for vulnerabilities, flagging if any or most flag it.

Use it when subtasks are truly independent (speed benefit) or when you need multiple perspectives or higher confidence on a high-stakes judgment. Each focused call often performs better than one call asked to juggle every consideration.

### Choosing between them

| Task shape | Pattern |
|---|---|
| Steps depend on each other in order | Chaining |
| Different input types need different handling | Routing |
| Independent pieces, or need several opinions | Parallelisation |

### Composing patterns

These are building blocks, not exclusive options. A support system might route a ticket, run a chain for the refund path, and use parallel sectioning for a guardrail check alongside the main answer. Keep each pattern visible in code so it can be tested and traced.

### What they have in common

All three are **workflows**: your code defines the paths. Claude supplies judgment within each step (or in the router, which picks the branch), but it does not decide the overall structure. That is why they are predictable and cheap to evaluate. Dependencies decide the latency: chaining adds latencies, while parallel branches cost roughly the slowest branch, though token cost still adds up.

## Example

A support pipeline combining all three:

```python
import asyncio

async def handle_ticket(ticket: str) -> str:
    # Routing: classify first (cheap model)
    category = await call_claude(ROUTER_PROMPT, ticket, model="small")
    if category not in HANDLERS:
        return escalate_to_human(ticket)              # fallback route

    # Parallelisation (sectioning): draft and safety check run together
    draft, safety = await asyncio.gather(
        HANDLERS[category](ticket),                   # specialised handler
        call_claude(SAFETY_PROMPT, ticket),
    )
    if safety.strip() != "OK":
        return escalate_to_human(ticket)

    # Prompt chaining with a gate: polish the draft, then verify
    reply = await call_claude(TONE_PROMPT, draft)
    if not passes_policy_checks(reply):               # deterministic gate in code
        return escalate_to_human(ticket)
    return reply
```

Routing picks the handler, sectioning overlaps the draft and the safety screen, and the final steps are a chain with a code gate. Every path is explicit and individually testable.

## Common mistakes / exam traps

- **Chaining independent steps** that could run in parallel, adding latency for no benefit.
- **Parallelising dependent steps**: if step B needs step A's output, it is a chain, not a fan-out.
- **A router with no fallback**: ambiguous or unexpected inputs need an "other" or human-escalation route, and the router needs its own evaluation.
- **Confusing sectioning with voting**: sectioning runs *different* subtasks; voting runs the *same* task multiple times for confidence.
- **Using an agent where a pattern suffices**: if the structure is known, these workflow patterns are simpler and cheaper than letting the model plan.

## Check yourself

1. What is a "gate" in a prompt chain, and why use one?
2. A support system sends simple FAQs to a small model and complex billing disputes to a larger one. Which pattern is this?
3. What is the difference between sectioning and voting?
4. Three code-review checks (security, style, performance) are independent. Chain or parallelise, and why?
5. Name one risk specific to routing and one mitigation.

<details><summary>Answers</summary>

1. A programmatic check between steps that validates intermediate output before continuing, so errors are caught early and later calls are not wasted.
2. Routing: classify the input, then send it to a handler (and model) suited to its category.
3. Sectioning splits a task into different independent subtasks run in parallel; voting runs the same task several times and aggregates the outputs for confidence.
4. Parallelise: the checks do not depend on each other, so running them concurrently cuts latency and each call stays focused.
5. Misclassification sends input to the wrong handler. Mitigate with a fallback or "other" route, human escalation, and a dedicated evaluation set for the router.

</details>

## Key takeaways

- Chaining = sequential steps with optional gates; routing = classify then specialise; parallelisation = sectioning or voting.
- Pick the pattern from the task's dependency structure: dependent steps chain, distinct input types route, independent pieces parallelise.
- All three are workflows: your code owns the flow, which keeps them predictable and testable.
- Patterns compose, but keep each one explicit, with gates and fallbacks.
