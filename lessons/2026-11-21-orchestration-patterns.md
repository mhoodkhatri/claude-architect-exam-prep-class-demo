# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-21

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you decide a fixed workflow fits your task, you still have to shape it. Three patterns cover most production pipelines: chaining steps, routing inputs to specialised handlers, and running independent work in parallel. Knowing which one fits a scenario is a core exam skill, and it is how you get predictable cost, latency and quality out of Claude.

## Core concepts

### Prompt chaining

Break a task into a **sequence of steps**, where each LLM call processes the output of the previous one. Between steps you can insert **gates**: programmatic checks that validate intermediate output before continuing.

- Use it when the task decomposes cleanly into fixed subtasks.
- It trades **latency for accuracy**: each call is simpler and easier to get right than one giant prompt.
- Typical shapes: write an outline, check it against criteria, then write the document; generate copy, then translate it; extract data, validate it, then summarise it.
- Gates are the key feature. A gate can be plain code (schema check, arithmetic, length limit) or another model call. If a gate fails, you stop, retry the step or route to a human.

### Routing

**Classify the input, then send it to a specialised path.** The router is one call (or even a rule or a small classifier) that picks among downstream prompts, tools or models.

- Use it when inputs fall into **distinct categories** that need different handling, and a single prompt would be a compromise across all of them.
- Benefits: separation of concerns, shorter focused prompts, and the ability to optimise each branch independently.
- It also enables **cost control**: send easy, high-volume queries to a smaller, cheaper model and hard ones to a more capable model.
- Examples: customer support (refund vs technical vs billing), triage of code review requests, query difficulty tiers.
- Routing only works if classification is **reasonably accurate**. Include a fallback or "other / unsure" route, and evaluate the router on its own test set.

### Parallelisation

Run multiple LLM calls **at the same time** and aggregate the results programmatically. It has two main variants:

- **Sectioning**: split a task into **independent subtasks** that run concurrently (for example, analyse each file of a pull request separately, or run a guardrail check alongside the main answer).
- **Voting**: run the **same task several times** (possibly with different prompts) to get diverse outputs, then take a majority or union. Useful when you need higher confidence, such as flagging vulnerable code or content moderation with tuned thresholds.

Parallelisation reduces **wall-clock latency** (sectioning) or raises **confidence** (voting). It costs more total tokens, and it requires that subtasks truly do not depend on each other. Watch out for rate limits when fanning out widely.

### Choosing between them

| Situation | Pattern |
|---|---|
| Fixed ordered steps, each building on the last | Chaining |
| Different input types need different handling | Routing |
| Independent subtasks, or need multiple opinions | Parallelisation |

### Combining patterns

They compose. A support system might route a ticket by category, run a chain inside the chosen branch (draft, policy check, polish), and run a parallel safety check on the final draft. All of this is still a **workflow**: your code owns the control flow. None of these patterns need the model to decide the next step, which is what separates them from the orchestrator-worker and agent designs in later lessons.

### Practical design notes

- Keep each step's **prompt narrow** and its output format explicit (often structured JSON) so the next step or gate can parse it.
- Put **deterministic checks in code**, not in a model call.
- **Evaluate per stage**: you can measure the router, each chain step and the aggregator separately.
- Decide upfront what a **failed gate** does: retry with feedback, fall back, or escalate.

## Example

A pull request review pipeline combining all three patterns:

```python
import asyncio

async def review_pr(pr):
    # Routing: cheap classification picks the review track
    track = await classify(pr.diff)            # "docs" | "code" | "config"

    if track == "docs":
        return await quick_docs_review(pr)     # small model, short prompt

    # Parallelisation (sectioning): one call per changed file
    file_reviews = await asyncio.gather(
        *(review_file(f) for f in pr.files)
    )

    # Parallelisation (voting): 3 independent security passes
    sec_votes = await asyncio.gather(
        *(security_scan(pr.diff) for _ in range(3))
    )
    security_flagged = sum(v.flagged for v in sec_votes) >= 2

    # Prompt chaining with a gate: summarise, then validate
    summary = await summarise(file_reviews, security_flagged)
    if not valid_schema(summary) or len(summary.text) > 2000:   # gate in code
        summary = await summarise(file_reviews, security_flagged, retry_hint=True)
    return summary
```

The routing step skips unnecessary work, the file reviews run concurrently, the security vote raises confidence, and the final chain step is gated by deterministic code.

## Common mistakes / exam traps

- **Parallelising dependent steps.** If step B needs step A's output, it is a chain, not a parallel job.
- **Routing without a fallback.** Misclassified or ambiguous inputs need a default path or human escalation.
- **Skipping gates in a chain.** Without validation between steps, an early error propagates and compounds through every later call.
- **Calling a fixed chain or router "an agent".** If your code decides the flow, it is a workflow, however many calls it makes.
- **Using voting when sectioning is what you need** (or vice versa). Sectioning splits different work for speed; voting repeats the same work for confidence.

## Check yourself

1. What does a gate do in a prompt chain?
2. Which pattern lets you send simple queries to a cheaper model and hard ones to a stronger model?
3. Distinguish sectioning from voting.
4. A task has three steps where each needs the previous step's result. Which pattern, and why not parallelisation?
5. Who controls the flow in all three patterns, and what does that make them?

<details><summary>Answers</summary>

1. It is a programmatic check (code or a model call) on an intermediate output that decides whether to continue, retry or escalate before the next step runs.
2. Routing: a classifier step picks the downstream path and model based on input type or difficulty.
3. Sectioning runs different, independent subtasks concurrently for speed; voting runs the same task multiple times and aggregates for confidence.
4. Prompt chaining, because the steps depend on each other's output, so they cannot run concurrently.
5. Your code controls the flow, so all three are workflows (predefined code paths), not autonomous agents.

</details>

## Key takeaways

- Chaining: fixed sequential steps with gates, trading latency for accuracy.
- Routing: classify, then specialise; add a fallback and evaluate the router itself.
- Parallelisation: sectioning for speed on independent subtasks, voting for confidence.
- The patterns compose, and all are workflows because code owns the control flow.
