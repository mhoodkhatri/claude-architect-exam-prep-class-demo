# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-10

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a task suits a workflow, you still have to choose its shape. Three building blocks cover most production pipelines: chaining steps in sequence, routing inputs to specialised handlers, and running independent work in parallel. Picking the right one controls accuracy, latency and cost, and the exam expects you to match a scenario to the pattern.

## Core concepts

### Prompt chaining

A task is decomposed into a **fixed sequence of steps**, where each Claude call processes the output of the previous one. Between steps you can insert **programmatic gates**: code that checks the intermediate result and stops, retries or reroutes before continuing.

- Use it when the task cleanly splits into subtasks that are known in advance.
- It trades **latency for accuracy**: each call is simpler and easier to get right than one giant prompt.
- Typical examples: draft marketing copy, then translate it; write an outline, check it against criteria, then write the document.
- Gates are the key feature. A cheap check (valid JSON? length limit? required fields present?) catches errors early, before later steps waste tokens building on them.

### Routing

A **classification step** looks at the input and sends it to one of several specialised downstream paths, each with its own prompt, tools or even model.

- Use it when inputs fall into **distinct categories** that are better handled separately, and classification can be done accurately.
- It enables **separation of concerns**: optimising a prompt for refund questions does not degrade the prompt for technical support.
- It enables **cost control**: send easy, common queries to a small fast model (such as Haiku) and hard ones to a more capable model (such as Sonnet or Opus).
- The router can be an LLM call, a traditional classifier or plain rules. If rules work, use rules.
- Always include a **fallback route** (a default handler or human escalation) for inputs that fit no category or have low classification confidence.

### Parallelisation

Multiple LLM calls run **at the same time** and their outputs are aggregated by code. It comes in two variants:

- **Sectioning**: break a task into **independent subtasks** that run concurrently. Example: one call checks content for policy violations while another answers the user's question; or reviewing each file of a diff separately.
- **Voting**: run the **same task several times** to get diverse outputs or higher confidence. Example: three independent reviews of code for vulnerabilities, flagging if any (or a majority) report an issue.

Use it when subtasks are genuinely independent (so speed improves) or when you need multiple perspectives on a high-stakes decision. Because calls do not depend on each other, total latency is roughly the slowest call, not the sum. Cost is the sum of all calls, so parallelisation buys speed and confidence, not savings.

### Choosing between them

| Situation | Pattern |
|---|---|
| Steps depend on each other in a known order | Chaining |
| Inputs differ in kind and need different handling | Routing |
| Subtasks are independent and latency matters | Parallelisation (sectioning) |
| One decision needs higher confidence | Parallelisation (voting) |

### Combining patterns

These are composable. A realistic system might route a ticket by type, run a chain for each route, and fan out parallel checks inside one step. Keep each pattern simple and add complexity only when evaluations show a gain.

### Failure handling

- In a chain, a failed gate should trigger a **scripted response** (retry the step, return a fallback, escalate), not silently continue.
- In parallel runs, decide in advance what happens if **one branch fails or times out**: wait, proceed with partial results, or fail the whole job.
- In routing, log the chosen route so misclassifications can be found and fixed.

## Example

A support system for a software company:

```python
import asyncio

def handle_ticket(ticket: str) -> str:
    # ROUTING: cheap model classifies; rules for fallback
    category = call_claude(ROUTER_PROMPT, ticket, model="haiku").strip()
    if category not in {"billing", "bug", "how_to"}:
        return escalate_to_human(ticket)

    if category == "bug":
        # CHAINING with a gate
        summary = call_claude(SUMMARISE_BUG_PROMPT, ticket)
        if not has_required_fields(summary):          # programmatic gate
            return ask_customer_for_details(ticket)
        return call_claude(BUG_REPLY_PROMPT, summary)

    return call_claude(PROMPTS[category], ticket, model="sonnet")

async def review_pr(files: list[str]) -> list[str]:
    # PARALLELISATION (sectioning): one independent review per file
    results = await asyncio.gather(*(review_file(f) for f in files))
    return [r for r in results if r.has_issues]
```

Routing keeps each prompt specialised and sends easy work to a cheaper model. The bug path chains two steps with a gate. The PR review runs per-file reviews concurrently, so the wall-clock time is about one review rather than N.

## Common mistakes / exam traps

- **Using parallelisation for dependent steps.** If step B needs step A's output, it is a chain; running them concurrently cannot work.
- **Routing without a fallback.** Misclassified or unusual inputs then go to the wrong handler with no safety net.
- **Confusing sectioning with voting.** Sectioning splits *different* subtasks; voting repeats the *same* task for confidence.
- **Assuming parallel calls reduce cost.** They reduce latency; total token cost is the same or higher (voting multiplies it).
- **Chaining with no gates.** Without intermediate checks, an early error propagates and compounds through every later step.

## Check yourself

1. What is a "gate" in a prompt chain and why is it useful?
2. Name two benefits of routing beyond better answers.
3. What is the difference between sectioning and voting?
4. Your pipeline sends every request through a large model, but 80% are simple FAQs. Which pattern helps and how?
5. Three independent reviewers flag security issues in the same code and you act if any of them finds one. Which pattern is this, and what does it cost versus a single review?

<details><summary>Answers</summary>

1. A programmatic check between steps (for example validating format or required fields) that lets you stop, retry or reroute before errors propagate and waste later calls.
2. Separation of concerns (specialised prompts that do not interfere with each other) and cost control (cheap models for easy inputs, stronger models for hard ones). Also: easier testing per route.
3. Sectioning splits a task into different independent subtasks run concurrently; voting runs the same task multiple times to get diverse outputs or higher confidence.
4. Routing: classify with a cheap, fast model and send simple FAQs to a small model while only hard queries go to the large one, with a fallback route for unclear cases.
5. Parallelisation by voting. Latency is about one review (run concurrently), but token cost is roughly three times a single review.

</details>

## Key takeaways

- Chaining: fixed sequential steps with gates; trades latency for accuracy.
- Routing: classify, then dispatch to specialised handlers (and cheaper models), always with a fallback.
- Parallelisation: sectioning for independent subtasks, voting for confidence; saves time, not money.
- Patterns compose, but add each only when evaluation shows it helps.
