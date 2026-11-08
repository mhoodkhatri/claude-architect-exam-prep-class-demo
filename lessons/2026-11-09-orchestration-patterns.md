# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-09

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a workflow fits the task, you still have to shape it. Three building-block patterns cover most production pipelines: chaining steps in sequence, routing inputs to specialised handlers, and running work in parallel. Knowing which pattern matches which problem lets you improve accuracy, cost and latency without jumping to a full agent.

## Core concepts

### Prompt chaining

A task is decomposed into a **fixed sequence of LLM calls**, where each call consumes the previous call's output. Between steps you can insert **gates**: deterministic code checks (schema validation, length limits, a required field) that decide whether to continue, retry or stop.

Use it when:
- The task splits cleanly into subtasks (outline, then draft, then translate).
- You trade a little latency for **higher accuracy**, because each call has an easier job and a focused prompt.
- You want to inspect or test intermediate outputs.

Typical examples: generate marketing copy, then check it against brand rules, then translate; write an outline, validate it, then write the document from the outline.

Key design point: gates make chains reliable. A chain with no validation between steps simply passes errors downstream.

### Routing

A **classification step directs the input to one of several specialised paths**. The router can be an LLM call, a small model, a traditional classifier or plain rules. Each downstream path has its own prompt, tools or even model.

Use it when:
- Inputs fall into **distinct categories** that need different handling (billing question, technical fault, refund request).
- One generic prompt performs worse than several focused ones, and optimising for one category would hurt another.
- You want **cost control**: send easy queries to a small, fast model and hard ones to a larger model.

Design points:
- The router must be accurate; a misroute sends the input down the wrong path. Evaluate the classifier separately.
- Include a **fallback or "other" route** (often to a human or a general handler) for low-confidence or unrecognised inputs.
- Keep the categories mutually exclusive and clearly defined.

### Parallelisation

Independent LLM calls run **at the same time**, and your code aggregates the results. There are two variants:

- **Sectioning**: split a task into independent subtasks run concurrently. Example: one call screens a user message for policy violations while another answers it; or separate calls review a document for security, style and performance.
- **Voting**: run the **same task multiple times** and combine outputs for confidence. Example: several independent reviews of a code change, flagging it if any (or a majority) find a problem; useful where false negatives are costly.

Use it when:
- Subtasks are **independent** (no call needs another's output), so running them concurrently cuts latency.
- Several perspectives or repeated attempts raise confidence.
- Each aspect is best handled by its own focused prompt.

Trade-off: parallelism reduces wall-clock time but **not total cost**; token usage is the sum of all calls (voting multiplies it). You also need an aggregation rule and a plan for partial failures (what if one branch times out?).

### Choosing between them

| Situation | Pattern |
|---|---|
| Steps depend on each other, in order | Chaining |
| Input type determines which handling is right | Routing |
| Independent subtasks, or need for consensus | Parallelisation |

### Combining patterns

They compose. A support system might **route** a ticket by category, run a **chain** (extract details, look up account, draft reply, check tone) on the chosen path, and use **parallel** guardrail checks alongside the draft. All three keep code in control of the flow, so they remain workflows rather than agents.

## Example

A support pipeline that routes, then chains, with a parallel safety check:

```python
import asyncio

async def handle_ticket(text: str) -> str:
    # Routing: classify into a known category
    route = await call_claude(ROUTER_PROMPT, text)          # "billing" | "technical" | "other"
    if route not in HANDLERS:
        return escalate_to_human(text, reason="unrouted")

    # Parallelisation (sectioning): draft and safety screen run concurrently
    draft_task = asyncio.create_task(run_chain(route, text))
    safety_task = asyncio.create_task(call_claude(SAFETY_PROMPT, text))
    draft, safety = await asyncio.gather(draft_task, safety_task)

    if safety.strip() != "SAFE":
        return escalate_to_human(text, reason="safety flag")
    return draft

async def run_chain(route: str, text: str) -> str:
    # Prompt chaining with a gate between steps
    details = await call_claude(HANDLERS[route].extract, text)
    if not valid_json_with_fields(details, HANDLERS[route].required):   # gate in code
        return escalate_to_human(text, reason="extraction failed")
    return await call_claude(HANDLERS[route].reply, details)
```

Each pattern appears once and each is independently testable: router accuracy, extraction validity, and reply quality.

## Common mistakes / exam traps

- **Chaining without gates**: an early bad output flows through every later step. Validate between steps.
- **Parallelising dependent steps**: if step B needs A's output, it is a chain, not parallel work.
- **Assuming parallelism saves money**: it saves latency; total token cost stays the same or rises (voting).
- **No fallback route**: a router with no "other" or low-confidence path forces bad guesses into a wrong handler.
- **Using an agent where one of these patterns fits**: if the structure is known, code-controlled patterns are cheaper and more predictable.

## Check yourself

1. What is a "gate" in a prompt chain, and why add one?
2. Which pattern lets you send easy queries to a cheaper model?
3. Distinguish sectioning from voting.
4. Does parallelisation reduce total token cost? Explain.
5. A task has three independent checks on one document plus a final summary that uses all three results. Which patterns apply?

<details><summary>Answers</summary>

1. A deterministic programmatic check between steps (for example schema validation) that decides to continue, retry or stop, preventing errors from propagating down the chain.
2. Routing: a classifier assesses difficulty or category and directs the input to a small or large model accordingly.
3. Sectioning splits a task into different independent subtasks run concurrently; voting runs the same task several times and aggregates the outputs for confidence.
4. No. It reduces wall-clock latency, but total tokens are the sum of all calls, and voting increases cost.
5. Parallelisation (sectioning) for the three checks, then chaining for the summary step that depends on their combined outputs.

</details>

## Key takeaways

- Chaining: fixed sequence with gates; trade latency for accuracy.
- Routing: classify, then dispatch to specialised paths; always include a fallback.
- Parallelisation: sectioning for independent subtasks, voting for confidence; it saves time, not tokens.
- The patterns compose, and all keep your code in control of the flow.
