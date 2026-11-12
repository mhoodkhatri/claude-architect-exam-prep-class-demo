# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-13

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a task suits a workflow, you still need to shape it. Three patterns cover most production pipelines: chaining steps in sequence, routing inputs to specialised handlers, and running independent work in parallel. Knowing which pattern fits which problem lets you trade latency, cost and accuracy deliberately, and the exam tests exactly that matching.

## Core concepts

### Prompt chaining

Break a task into a **fixed sequence** of calls where each step's output feeds the next. Between steps you can insert programmatic **gates**: code that checks the intermediate result and stops, retries or reroutes if it is wrong.

- Use it when the task decomposes cleanly into subtasks and you would rather make each call easier than make one call do everything.
- It trades **latency for accuracy**: more calls, but each is simpler, focused and independently testable.
- Typical chains: outline → check outline against criteria → write document; draft → translate; extract → validate → summarise.
- The gate is the key design element. A cheap deterministic check (schema valid? totals match? length within limit?) between steps stops errors from compounding downstream.

### Routing

A **classifier step** examines the input and sends it to one of several specialised paths, each with its own prompt, tools or even model.

- Use it when inputs fall into **distinct categories** that are better handled separately, and classification can be done accurately (by an LLM or a traditional classifier).
- Benefits: focused prompts (no giant prompt covering every case), and **cost control**. Easy, common queries go to a small, fast model; hard or unusual ones go to a more capable model.
- Without routing, optimising for one input type often hurts performance on others.
- Always include a **fallback route** (a general handler or human escalation) for inputs the classifier cannot place confidently.
- The router's accuracy caps the whole system: evaluate it as its own stage.

### Parallelisation

Run multiple calls at the same time and combine results with code. Two variants:

1. **Sectioning**: split a task into **independent subtasks** that run concurrently. Example: one call screens a user message for policy violations while another answers it; or each section of a long report is analysed separately.
2. **Voting**: run the **same task several times** (possibly with different prompts) and aggregate for higher confidence. Example: three independent reviews of code for vulnerabilities, flagging if any (or a majority) finds an issue.

Use parallelisation when subtasks are truly independent and you want **speed**, or when multiple perspectives raise **confidence**. It does not reduce total tokens; it reduces wall-clock time (sectioning) or improves reliability at extra cost (voting). If step B needs the output of step A, it is a chain, not a parallel job.

### Choosing between them

| Situation | Pattern |
|---|---|
| Steps depend on previous outputs, each simpler than the whole | Chaining |
| Input types differ and need different handling | Routing |
| Independent subtasks, latency matters | Parallelisation (sectioning) |
| Need higher confidence on a judgment call | Parallelisation (voting) |

They compose: a router can send a request into one of several chains, and a chain step can fan out into parallel calls. In all three, **your code** controls the flow, so these remain workflows (see the previous lesson).

### Implementation notes

- Keep intermediate outputs **structured** (JSON) so gates and routers can parse them reliably.
- Fan-out needs error handling: decide what happens if one of five parallel calls fails (retry, proceed with partial results, or fail the whole job).
- Log every stage's input and output; pipelines are debugged stage by stage.

## Example

A support-ticket system combining all three patterns:

```python
import asyncio

async def handle_ticket(ticket: str) -> str:
    # Routing: classify, with a fallback route
    label = await classify(ticket)  # "billing" | "technical" | "account" | "other"
    if label not in HANDLERS:
        return await escalate_to_human(ticket)

    # Parallelisation (sectioning): independent checks run concurrently
    safety, sentiment, context = await asyncio.gather(
        check_policy(ticket),
        score_sentiment(ticket),
        fetch_customer_context(ticket),
    )
    if not safety.ok:
        return await escalate_to_human(ticket)

    # Prompt chaining: draft -> gate -> polish
    draft = await HANDLERS[label](ticket, context, sentiment)
    if not passes_style_gate(draft):          # deterministic check
        draft = await revise(draft, GATE_FEEDBACK)
    return await polish_tone(draft, sentiment)
```

Billing tickets use a billing-specific prompt and tools; simple "other" tickets could route to a smaller model. The three safety, sentiment and context calls do not depend on each other, so they run together.

## Common mistakes / exam traps

- **Parallelising dependent steps.** If one call needs another's output, it must be a chain. Parallelisation is only for independent work.
- **Chaining without gates.** A sequence with no validation between steps lets an early error flow through every later step.
- **Routing with no fallback.** Misclassified or unusual inputs get a wrong specialised answer instead of a safe default or human review.
- **Confusing sectioning with voting.** Sectioning splits *different* subtasks for speed; voting repeats the *same* task for confidence.
- **Assuming these patterns make the system an agent.** Fixed code paths mean it is still a workflow.

## Check yourself

1. Which pattern trades extra latency for higher accuracy by simplifying each call?
2. Why should a routing system have a fallback route?
3. What is the difference between sectioning and voting?
4. A task needs step B to use step A's output. Can you parallelise them?
5. How can routing reduce cost?

<details><summary>Answers</summary>

1. Prompt chaining: several simpler calls in sequence, with optional gates between them.
2. The classifier can be uncertain or meet inputs outside the known categories; a fallback (general handler or human escalation) avoids confidently wrong specialised answers.
3. Sectioning runs different independent subtasks concurrently to save time; voting runs the same task multiple times and aggregates results for confidence.
4. No. The dependency makes it a chain; parallelisation requires independent subtasks.
5. Easy, common inputs go to a smaller, cheaper, faster model, and only hard inputs go to a more capable one.

</details>

## Key takeaways

- Chaining: sequential steps with gates; accuracy over latency.
- Routing: classify, then dispatch to specialised handlers; always provide a fallback.
- Parallelisation: sectioning for independent subtasks and speed, voting for confidence.
- The patterns compose, and because code controls the flow they are all workflows.
