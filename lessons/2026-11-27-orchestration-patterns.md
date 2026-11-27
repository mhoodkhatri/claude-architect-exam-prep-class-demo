# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-27

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a task suits a workflow, you still need to choose its shape. Three building blocks cover most production pipelines: chaining calls in sequence, routing inputs to specialised handlers, and running independent calls in parallel. Knowing which one fits, and what each costs in latency, accuracy and complexity, is a core architect skill and a frequent exam scenario.

## Core concepts

### Prompt chaining

A task is decomposed into a **fixed sequence of steps**, where each Claude call processes the output of the previous one. Your code can insert **gates** between steps: programmatic checks that verify intermediate output before continuing.

- **Use when** the task splits cleanly into subtasks and you are willing to trade latency for accuracy, because each call is simpler and easier to get right.
- **Typical examples**: write marketing copy, then translate it; draft an outline, check it against criteria, then write the document from the outline.
- **Benefits**: each prompt is short and focused, each stage can be evaluated on its own, and a gate can stop bad output early instead of propagating it.
- **Cost**: total latency is the sum of all steps, and an early error can contaminate later steps if there is no gate.

### Routing

A first step **classifies the input** and directs it to a specialised downstream path: a different prompt, tool set, or even a different model.

- **Use when** there are distinct categories that are better handled separately, and classification can be done accurately (by an LLM or by a traditional classifier or rules).
- **Typical examples**: customer support triage (refund, technical, billing, general); sending easy questions to a small, fast model such as Haiku and hard ones to a larger model.
- **Benefits**: specialised prompts do not compete with each other, so optimising for one category does not hurt another. Routing also controls cost by matching model size to difficulty.
- **Cost**: a misclassification sends the input down the wrong path. Include a sensible fallback or "other" route, and evaluate the router itself as a component.

### Parallelisation

Multiple calls run **at the same time**, and code aggregates the results. It has two variants:

- **Sectioning**: split a task into **independent subtasks** that run concurrently. Example: one call answers the user while another screens the input for policy problems; or review a document by section, each section handled separately.
- **Voting**: run the **same task several times** to get diverse outputs, then aggregate (majority vote, or flag on any positive). Example: several independent passes reviewing code for vulnerabilities, flagging if any finds an issue.

Use it when subtasks are truly independent and you want speed, or when you need higher confidence from multiple attempts. Separate calls per concern also let the model focus on one aspect at a time, which often beats one long prompt that asks for everything.

- **Cost**: voting multiplies token spend. Sectioning requires that subtasks do not depend on each other; if one needs another's output, it is a chain, not a parallel step. Watch rate limits when fanning out.

### Choosing and combining

| Pattern | Flow | Optimises for | Key risk |
|---|---|---|---|
| Chaining | A → B → C | Accuracy via simpler steps | Latency, error propagation |
| Routing | classify → one of several paths | Specialisation and cost | Misclassification |
| Parallelisation | A, B, C together → aggregate | Speed or confidence | Cost, hidden dependencies |

These are composable. A real system might route a ticket, run a chain on the chosen path, and parallelise two checks inside one step. In all three, **your code controls the flow**, which is why they are workflows rather than agents.

### Design rules

- Put **deterministic gates** (schema validation, totals checks) between chain steps; do not ask the model to verify what code can verify.
- Pass **structured output** between steps so the next stage parses reliably.
- Evaluate **each stage separately**, then the end-to-end result.
- Keep a **fallback** for every branch: retry, default route, or human escalation.

## Example

A support system for a software company. It routes first, then chains, with a parallel safety check.

```python
import asyncio

async def handle_ticket(ticket: str) -> str:
    # Routing: cheap model classifies the ticket
    category = await call_claude(ROUTER_PROMPT, ticket, model="claude-haiku-5-5")
    if category not in HANDLERS:
        return escalate_to_human(ticket, reason="unknown category")

    # Parallelisation (sectioning): draft and policy screen run concurrently
    draft_task = asyncio.create_task(chain_for(category, ticket))
    screen_task = asyncio.create_task(
        call_claude(POLICY_SCREEN_PROMPT, ticket, model="claude-haiku-5-5")
    )
    draft, screen = await asyncio.gather(draft_task, screen_task)

    if screen.strip() != "OK":
        return escalate_to_human(ticket, reason=screen)
    return draft

async def chain_for(category: str, ticket: str) -> str:
    # Prompt chaining with a gate
    facts = await call_claude(HANDLERS[category].extract, ticket)
    if not valid_json_with_required_fields(facts):          # deterministic gate
        raise ValueError("extraction failed validation")
    return await call_claude(HANDLERS[category].reply, facts, model="claude-sonnet-5-5")
```

Routing keeps prompts specialised and cheap, the chain makes each step simple with a validation gate, and the parallel screen adds no latency to the main path.

## Common mistakes / exam traps

- **Parallelising dependent steps.** If step B needs A's output, it is a chain. Running them "in parallel" gives wrong or empty inputs.
- **No gate in a chain.** A bad early output flows through every later step, wasting calls and producing confident but wrong results.
- **Routing without a fallback or router evaluation.** Misclassified inputs fail silently; measure router accuracy and provide an "other"/human route.
- **Using voting when sectioning is what is needed (or the reverse).** Sectioning splits different work; voting repeats the same work for confidence.
- **Reaching for an agent when one of these patterns suffices.** If the steps and categories are known, these fixed patterns are cheaper and more predictable.

## Check yourself

1. Which pattern would you use to send simple FAQ questions to a small model and complex ones to a larger model?
2. What is a "gate" in a prompt chain, and why add one?
3. Distinguish sectioning from voting.
4. A pipeline runs step 2 concurrently with step 1, but step 2 uses step 1's summary. What is wrong?
5. Name one cost of parallelisation and one cost of prompt chaining.

<details><summary>Answers</summary>

1. Routing: a classifier step directs each input to the appropriate model or prompt.
2. A programmatic check between steps that validates intermediate output before continuing. It stops errors early rather than letting them propagate.
3. Sectioning splits a task into different independent subtasks run concurrently; voting runs the same task multiple times and aggregates the outputs for confidence.
4. The steps are dependent, so they must run sequentially as a chain; concurrent execution gives step 2 missing or stale input.
5. Parallelisation: higher token and rate-limit usage (especially voting) or the need for truly independent subtasks. Chaining: added end-to-end latency, and error propagation without gates.

</details>

## Key takeaways

- Chaining trades latency for accuracy by making each step simpler, with gates between steps.
- Routing specialises handling per category and controls cost, but needs an accurate, evaluated classifier and a fallback.
- Parallelisation has two forms: sectioning for independent subtasks and voting for confidence.
- The patterns compose, and in all of them code, not the model, controls the flow.
