# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-24

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a task suits a workflow, you still have to shape it. Three patterns cover most production pipelines: chaining steps in sequence, routing inputs to specialised handlers, and running independent work in parallel. Knowing which one fits a given requirement (accuracy, cost, latency) is a staple of architecture questions on the exam.

## Core concepts

### Prompt chaining

Break a task into a **fixed sequence of LLM calls**, where each call's output feeds the next. Between steps you can insert **programmatic gates**: code that checks the intermediate result and stops, retries or reroutes if it fails.

- Trades a little latency for **higher accuracy**, because each call has an easier, narrower job.
- Use when the task decomposes cleanly into subtasks: draft then translate, outline then check the outline then write the document, extract then validate then summarise.
- Each step can be prompted, tested and evaluated on its own.
- Gates catch errors early, so a bad step 1 does not silently corrupt step 4.

### Routing

A **classifier step** examines the input and sends it to one of several specialised downstream paths (prompts, models, tools or whole sub-workflows).

- Use when inputs fall into **distinct categories** that need different handling: refund vs technical issue vs general question.
- Lets you **optimise per category** with focused prompts, instead of one bloated prompt that handles everything and does each case worse.
- Enables **cost control**: send easy, high-volume queries to a small fast model (such as Haiku) and hard ones to a more capable model.
- The router can be an LLM call, a cheap traditional classifier or plain rules. The router's accuracy caps the whole system, so evaluate it separately and include a sensible fallback or "unknown" route.

### Parallelisation

Run multiple LLM calls at the same time and combine results in code. Two variants:

- **Sectioning**: split a task into **independent subtasks** that run concurrently, for example one call checks a document for policy violations while another drafts the answer, or several calls each review a different file.
- **Voting**: run the **same task several times** and aggregate (majority vote, any-flag, threshold) to get higher confidence, for example multiple prompts reviewing code for vulnerabilities, or several judgements of whether content is inappropriate.

Use it when subtasks are truly independent (so latency drops to the slowest branch) or when you need multiple perspectives for confidence. It costs more tokens, so justify it with speed or reliability gains. Do not parallelise steps where one needs the other's output; that is a chain.

### Choosing between them

| Situation | Pattern |
|---|---|
| Steps depend on each other, accuracy matters more than speed | Chaining |
| Different input types need different handling | Routing |
| Independent subtasks, latency matters | Parallelisation (sectioning) |
| High-stakes judgement needing confidence | Parallelisation (voting) |

### Combining patterns

They compose. A support system might route a ticket to a category, run a chain within that category, and in one step run guardrail checks in parallel with response drafting. All of this is still a workflow: code owns the control flow.

### Implementation notes

- Pass **structured output** (JSON validated against a schema) between steps so gates are deterministic.
- Keep each step's prompt focused on one job.
- Use async calls or a thread pool for parallel branches, and decide up front how to handle a branch that fails or times out.
- Log every intermediate result; it is what makes debugging a pipeline easy.

## Example

A support pipeline combining all three patterns:

```python
import asyncio

async def handle_ticket(ticket: str) -> str:
    # ROUTING: cheap model classifies the ticket
    route = await classify(ticket, model="claude-haiku-5-5")  # "billing" | "technical" | "other"
    if route not in HANDLERS:
        return escalate_to_human(ticket)                       # fallback route

    # PARALLELISATION (sectioning): independent work runs concurrently
    draft, safety = await asyncio.gather(
        HANDLERS[route](ticket),          # specialised prompt per category
        check_policy(ticket),             # guardrail check
    )
    if not safety.ok:
        return escalate_to_human(ticket)

    # PROMPT CHAINING with a gate: draft -> review -> final
    review = await critique(draft)
    if review.needs_fix:
        draft = await revise(draft, review.notes)
    return draft
```

Routing keeps easy tickets cheap, parallel calls cut latency, and the chain with a gate improves quality before anything is sent.

## Common mistakes / exam traps

- **Parallelising dependent steps.** If step B needs A's output it is a chain; forcing it parallel gives wrong results.
- **No fallback route.** A router that must pick a category will misroute unusual input; add an "other/unknown" path or human escalation.
- **Confusing sectioning with voting.** Sectioning splits different subtasks; voting repeats the same task for confidence.
- **Chaining without gates.** Without checks between steps, errors compound and you lose the main benefit of chaining.
- **Over-engineering.** If a single well-prompted call meets the quality bar, adding patterns only adds cost and latency.

## Check yourself

1. Which pattern would you use to send simple FAQs to a small model and complex disputes to a larger one?
2. What is the purpose of a gate in a prompt chain?
3. Distinguish sectioning from voting.
4. Why is parallelisation inappropriate when step 2 uses step 1's output?
5. A router misclassifies 15% of inputs. What does that imply for the overall system, and what should you do?

<details><summary>Answers</summary>

1. Routing: a classifier step directs each input to the appropriate model or handler.
2. A programmatic check between steps that validates the intermediate output and stops, retries or reroutes on failure, preventing errors from propagating.
3. Sectioning runs different, independent subtasks concurrently; voting runs the same task multiple times and aggregates the results for confidence.
4. The steps are dependent, so they must run sequentially. That is prompt chaining, not parallelisation.
5. The router caps overall accuracy, so those errors flow downstream. Evaluate the router on its own, improve its prompt or examples, and add a fallback or escalation route for low-confidence cases.

</details>

## Key takeaways

- Chaining: sequential decomposition with gates, trading latency for accuracy.
- Routing: classify, then specialise, which improves quality and controls cost.
- Parallelisation: sectioning for independent subtasks and speed, voting for confidence.
- Patterns compose, and all of them remain workflows where code controls the flow.
- Evaluate each stage (especially the router) independently.
