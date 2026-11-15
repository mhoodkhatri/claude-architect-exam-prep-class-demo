# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-16

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a task suits a workflow, you still have to shape it. Three patterns cover most production pipelines: chaining steps in sequence, routing inputs to specialised handlers, and running independent work in parallel. Knowing which pattern fits a task lets you cut latency, raise quality and keep cost predictable without needing an autonomous agent.

## Core concepts

### Prompt chaining

Decompose a task into a **fixed sequence of steps**, where each LLM call processes the output of the previous one. Between steps you can add programmatic **gates**: code checks that validate intermediate output before continuing.

- Use it when the task splits cleanly into subtasks and you want to trade a little latency for **higher accuracy**, because each call has an easier, narrower job.
- Typical shapes: outline → check outline → write document; extract → validate → translate; draft copy → localise.
- Gates are the key feature. A schema check, a length check or a lightweight LLM check after step N can stop bad output early, retry the step or route to a human.
- Each step is independently testable, and a failure points to a specific stage.

### Routing

A **classifier step** examines the input and sends it to one of several specialised downstream paths, each with its own prompt, tools or even model.

- Use it when inputs fall into **distinct categories** that are better handled separately, and one generic prompt would be a compromise for all of them.
- Typical shapes: support tickets (billing / technical / refund), query difficulty (easy questions to a small, cheap model; hard ones to a larger model), document type (invoice / contract / receipt).
- The classifier can be an LLM call, a traditional classifier or plain rules. Use code when the rule is deterministic.
- Always include a **fallback route** (a default handler or human queue) for inputs that fit no category or have low classification confidence.
- Routing's benefit is **separation of concerns**: optimising the prompt for one category does not degrade another.

### Parallelisation

Run multiple LLM calls **at the same time** and combine the outputs programmatically. There are two variants:

- **Sectioning**: split a task into **independent subtasks** that run concurrently. Example: one call screens a user message for policy violations while another answers it; or each section of a long report is analysed separately.
- **Voting**: run the **same task several times** (possibly with different prompts or models) and aggregate the results, for example by majority vote or by requiring all to agree. Example: several reviewers each judge whether a code change is vulnerable, and you flag it if any one says yes.

Use parallelisation when subtasks are truly independent, so speed improves, or when you need **multiple perspectives or higher confidence**. Cost rises with the number of calls; latency is that of the slowest branch, not the sum.

### Choosing between them

| Task shape | Pattern |
|---|---|
| Ordered steps where each depends on the last | Chaining |
| Different input types need different handling | Routing |
| Independent subtasks, or need several opinions | Parallelisation |

### Combining patterns

The patterns compose. A common design: **route** a ticket to a category, run a **chain** inside that category, and use **parallel** calls for independent checks within a step. All of these are still workflows: your code controls the flow.

### Engineering notes

- Make the dependencies explicit. If step B needs A's output, it cannot be parallelised with A.
- Handle partial failure in parallel branches: decide whether one failed branch fails the whole job, is retried or is skipped.
- Pass **structured output** between steps (JSON validated against a schema) so gates and aggregation are reliable.
- Measure each pattern against a single well-prompted call. If the simple call is good enough, do not add orchestration.

## Example

A support system for a software company:

```python
import asyncio

async def handle_ticket(ticket: str) -> str:
    # Routing: classify first (cheap model)
    category = await classify(ticket)            # "billing" | "technical" | "other"
    if category not in HANDLERS:
        return await escalate_to_human(ticket)  # fallback route

    # Parallelisation (sectioning): independent work at the same time
    draft, sentiment, pii_found = await asyncio.gather(
        HANDLERS[category](ticket),              # specialised prompt for this category
        score_sentiment(ticket),
        detect_pii(ticket),
    )

    # Prompt chaining with a gate: check the draft, then polish it
    if pii_found or not passes_policy_check(draft):
        return await escalate_to_human(ticket)
    return await polish_tone(draft, sentiment)   # final step in the chain
```

Routing picks the handler, `asyncio.gather` runs three independent calls concurrently, and a gate sits between drafting and polishing. Everything is fixed code, so each stage can be tested and the cost per ticket is bounded.

## Common mistakes / exam traps

- **Parallelising dependent steps.** If one call needs another's output, it is chaining, not parallelisation.
- **Routing with no fallback.** Misclassified or unknown inputs silently go to the wrong handler. Always define a default or human route.
- **Chaining without gates.** An error in step 1 flows through every later step. Validate between steps.
- **Confusing sectioning with voting.** Sectioning divides *different* subtasks; voting repeats the *same* task for confidence.
- **Using an LLM to route when a rule would do.** Deterministic conditions (file type, a field value) belong in code.

## Check yourself

1. What is a "gate" in a prompt chain, and why is it useful?
2. A system sends easy questions to a small model and hard ones to a large model. Which pattern is this?
3. What is the difference between sectioning and voting?
4. Does parallelisation reduce latency, cost, or both? Explain.
5. Why should every router have a fallback route?

<details><summary>Answers</summary>

1. A programmatic check between steps that validates intermediate output (schema, policy, length, a light LLM check). It stops errors propagating and allows retry or escalation.
2. Routing: a classifier step sends each input to a handler suited to its difficulty or category.
3. Sectioning splits a task into different independent subtasks run concurrently; voting runs the same task multiple times and aggregates the results for confidence.
4. It can reduce latency, since total time is roughly that of the slowest branch. It does not reduce cost; more calls usually cost more.
5. Inputs may fit no category or be misclassified. A default handler or human queue prevents wrong or failed handling.

</details>

## Key takeaways

- Chaining: fixed sequence with gates. Routing: classify, then specialise. Parallelisation: sectioning for independent subtasks, voting for confidence.
- All three are workflows; your code controls the flow, which keeps them testable and predictable.
- Only parallelise independent work, and always add gates and fallbacks.
- Compose the patterns, and benchmark against a single well-prompted call before adding complexity.
