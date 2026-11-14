# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-15

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a task suits a workflow, you must choose its shape. Three patterns cover most production pipelines: chaining, routing and parallelisation. Picking the right one controls accuracy, latency and cost, and the exam frequently describes a scenario and asks which pattern fits.

## Core concepts

### Prompt chaining

A task is decomposed into a **fixed sequence of steps**, where each LLM call processes the output of the previous one. Between steps you can insert programmatic **gates**: code checks that verify the intermediate result before continuing.

Use it when:
- The task cleanly decomposes into subtasks (outline, then write, then translate).
- You want to trade a little **latency for accuracy**, because each call has a simpler job than one giant prompt.
- Intermediate outputs benefit from validation (schema check, length check, policy check).

Strengths: easy to debug (you can inspect every hop), each stage can be tuned and evaluated separately. Weakness: errors propagate downstream, and total latency is the sum of all steps. Gates exist to catch bad output early and stop or retry instead of passing it on.

### Routing

An input is **classified**, then sent to a specialised downstream prompt, tool or model. The classifier can be an LLM call, a cheaper model, or ordinary code (a regex, a lookup).

Use it when:
- Inputs fall into **distinct categories** that need different handling (billing question vs technical bug vs refund request).
- A single prompt trying to serve every category performs worse than separate specialised prompts.
- You want **cost control**: send easy queries to a small, fast model (such as Haiku) and hard ones to a larger model.

Strengths: separation of concerns, specialised few-shot examples per route, per-route evaluation. Weaknesses: the router is a single point of failure, so misclassification sends the request down the wrong path. Always include a **fallback or "other" route**, and measure router accuracy on its own.

### Parallelisation

Several LLM calls run **at the same time** and their outputs are aggregated by code. Two variants:

- **Sectioning**: split a task into **independent subtasks** run concurrently (check a document for PII, for toxicity and for factual claims at once). The subtasks do not depend on each other.
- **Voting**: run the **same task multiple times** and combine results (majority vote, or any-flag-wins) for higher confidence, for example several reviewers checking code for a vulnerability.

Use it when subtasks are independent and you want **speed**, or when you need **multiple perspectives** for confidence. A common pattern is running a guardrail check in parallel with the main response so the guardrail adds no latency.

Strengths: latency is roughly that of the slowest call rather than the sum. Weakness: cost multiplies with the number of calls, and you need an aggregation rule.

### Choosing between them

| If the work is... | Use |
|---|---|
| Sequential, each step needs the previous result | Chaining |
| One input, several kinds of handling depending on its type | Routing |
| Independent pieces, or you want redundancy for confidence | Parallelisation |

### Composition

These patterns combine freely. A support system might route a ticket by category, run a chain inside the "refund" route (extract, verify policy, draft reply), and parallelise a safety check alongside the draft. All of these remain **workflows**: code owns the control flow.

### Practical details

- Keep the data passed between steps **structured** (JSON) so gates and routers can be deterministic code.
- Set **timeouts and partial-failure rules** for parallel calls: decide whether one failed branch fails the whole job.
- Evaluate each stage and the end-to-end result separately.

## Example

A support pipeline combining all three:

```python
import asyncio

async def handle_ticket(ticket: str) -> str:
    # Routing: classify (cheap model), with a fallback route
    route = await classify(ticket, model="haiku")   # billing | bug | refund | other
    if route not in HANDLERS:
        route = "other"

    # Parallelisation (sectioning): draft and safety check run concurrently
    draft_task = HANDLERS[route](ticket)
    safety_task = safety_check(ticket)
    draft, safety = await asyncio.gather(draft_task, safety_task)
    if not safety.ok:
        return escalate_to_human(ticket, safety.reason)
    return draft

async def refund_handler(ticket: str) -> str:
    # Prompt chaining with a gate
    facts = await extract_facts(ticket)             # step 1
    if not valid_order_id(facts.order_id):          # gate: deterministic code
        return ask_customer_for_order_id()
    decision = await apply_policy(facts)            # step 2
    return await write_reply(facts, decision)       # step 3
```

Routing picks the handler, the refund handler chains steps with a gate, and the safety check is parallelised so it adds no latency.

## Common mistakes / exam traps

- **Using chaining when steps are independent.** If subtasks do not depend on each other, run them in parallel and save latency.
- **Parallelising dependent steps.** If step B needs step A's output, they cannot be sectioned.
- **Routing without a fallback.** Unknown or ambiguous inputs need a default or human-escalation route.
- **Confusing sectioning with voting.** Sectioning splits different subtasks; voting repeats the same task for confidence.
- **Forgetting that these are workflows.** None of the three lets the model decide the control flow; if the model chooses the next step dynamically, that is an agent.

## Check yourself

1. Which pattern would you use to run PII, toxicity and factual checks on a document as fast as possible?
2. What is a "gate" in a prompt chain?
3. Why might routing reduce cost?
4. What is the difference between sectioning and voting?
5. A router occasionally misclassifies tickets. Name two mitigations.

<details><summary>Answers</summary>

1. Parallelisation (sectioning): the checks are independent, so run them concurrently.
2. A programmatic check between steps that validates the intermediate output before the chain continues, so bad output is stopped or retried early.
3. Easy inputs can be sent to a smaller, cheaper, faster model and only hard ones to a larger model; each route also gets a shorter, focused prompt.
4. Sectioning runs different independent subtasks at once; voting runs the same task several times and aggregates for confidence.
5. Any two of: add a fallback/"other" route or human escalation; evaluate the router separately with a labelled test set; add few-shot examples to the classifier; use a confidence threshold; use a stronger model or deterministic rules for the router.

</details>

## Key takeaways

- Chaining: fixed sequence with gates, trading latency for accuracy.
- Routing: classify, then specialise; always provide a fallback route.
- Parallelisation: sectioning for independent subtasks, voting for confidence; faster but costlier.
- The patterns compose, and all remain workflows because code controls the flow.
