# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-19

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

The previous lesson said to prefer fixed workflows when the steps are known. This lesson covers the three core building blocks of those workflows: prompt chaining, routing and parallelisation. Most production Claude systems are compositions of these patterns, and the exam expects you to pick the right one from the shape of the task and explain the trade-off.

## Core concepts

### Prompt chaining

A task is split into a **sequence of steps**, where each LLM call processes the output of the previous one. Between steps you can insert programmatic **gates**: code checks that validate the intermediate result before continuing.

- **Use when** the task decomposes cleanly into fixed subtasks and you want higher accuracy by making each call simpler, trading some latency for it.
- **Typical examples**: write an outline, check it against criteria, then write the document; draft marketing copy, then translate it; extract data, validate it, then summarise it.
- **Why it works**: each call has one focused job, so prompts are shorter and clearer, and each stage can be tested and debugged on its own.
- **Gates matter**: a deterministic check (schema valid? totals match? length within limit?) catches errors early instead of letting them propagate down the chain.

### Routing

An input is **classified**, then sent to a specialised downstream path (a prompt, a tool set, or even a different model).

- **Use when** the input categories are distinct, are better handled separately, and can be classified accurately (by an LLM or by a traditional classifier or rules).
- **Typical examples**: customer support triage (refund, technical issue, general question each get their own prompt and tools); sending easy questions to a small, cheap model and hard ones to a larger model.
- **Why it works**: it lets you optimise each path without one generic prompt becoming a tangle of special cases, and it controls cost.
- **Key risk**: the router is a single point of failure. A misclassification sends the request down the wrong path, so evaluate the classifier on its own and include a sensible default or "unsure, ask a human" route.

### Parallelisation

Independent work runs **simultaneously** and the results are aggregated by code. It has two main variants:

- **Sectioning**: split a task into independent subtasks run in parallel. Example: one call checks a document for policy violations while another answers the user's question; or analyse each section of a long report separately.
- **Voting**: run the **same task several times** to get diverse outputs, then combine them (majority vote or a threshold). Example: several independent reviews of a code snippet for vulnerabilities, flagging if any or a quorum find an issue.

- **Use when** subtasks are independent (so speed improves), or when you need multiple perspectives or higher confidence on a high-stakes judgment.
- **Cost note**: parallel calls reduce latency, not total token cost. Voting multiplies cost.
- **Separate concerns**: models often do better when each consideration gets its own call with its own focused prompt, rather than one call juggling many.

### Choosing between them

| Task shape | Pattern |
|---|---|
| Fixed sequence, each step depends on the last | Prompt chaining |
| Distinct input types needing different handling | Routing |
| Independent subtasks, or need for consensus | Parallelisation |

They compose freely: route first, then run a chain on the chosen path, with a parallel guardrail check alongside.

### What they are not

All three are **workflows**: code, not the model, decides the structure. If the number of subtasks cannot be known in advance, that points toward the orchestrator-worker pattern (next lesson), not parallelisation.

## Example

A support system for a software company combines all three:

```python
import asyncio

async def handle_ticket(ticket: str) -> str:
    # ROUTING: classify into a known category
    category = await call_claude(ROUTER_PROMPT, ticket)  # "billing" | "bug" | "howto" | "unsure"
    if category not in HANDLERS:
        return escalate_to_human(ticket, reason="unsure")

    # PARALLELISATION (sectioning): answer and safety check run at the same time
    answer_task = asyncio.create_task(HANDLERS[category](ticket))
    safety_task = asyncio.create_task(call_claude(SAFETY_PROMPT, ticket))
    draft, safety = await asyncio.gather(answer_task, safety_task)
    if safety.strip() != "OK":
        return escalate_to_human(ticket, reason="safety")

    # PROMPT CHAINING with a gate: draft -> programmatic check -> polish
    if len(draft) > 2000 or "http" in draft and not links_allowed(draft):
        draft = await call_claude(SHORTEN_PROMPT, draft)
    return await call_claude(TONE_PROMPT, draft)
```

The router picks the path, the answer and safety check run concurrently, and the draft passes through a gate and a polishing step. Every stage is a separately testable call.

## Common mistakes / exam traps

- **Using an agent when a chain suffices**. If you can list the steps, chain them.
- **Unevaluated routers**. Accuracy of the classifier caps the quality of the whole system; no fallback route means misclassifications fail silently.
- **Parallelising dependent steps**. If step B needs step A's output, they cannot run concurrently; that is a chain.
- **Assuming parallelism saves money**. It cuts wall-clock time; total tokens stay the same, and voting increases them.
- **Chains without gates**. An early error flows through every later step; add cheap deterministic validation between stages.

## Check yourself

1. What is the difference between sectioning and voting?
2. Name the main benefit of prompt chaining and what a "gate" is.
3. A system sends simple FAQs to a small model and complex disputes to a larger one. Which pattern is this?
4. Why is the router a risk, and how do you mitigate it?
5. Does parallelisation reduce token cost? Explain.

<details><summary>Answers</summary>

1. Sectioning splits a task into different independent subtasks run in parallel; voting runs the same task multiple times and combines the outputs for confidence.
2. Each call handles a simpler, focused subtask, improving accuracy and testability. A gate is a programmatic check between steps that validates intermediate output before proceeding.
3. Routing: the input is classified and directed to a specialised path (here, different models).
4. A misclassification sends the request down the wrong path and caps overall quality. Mitigate by evaluating the classifier separately and adding a default or escalate-to-human route for uncertain cases.
5. No. It reduces latency by running work concurrently; total tokens are unchanged, and voting multiplies them.

</details>

## Key takeaways

- Chaining: fixed sequential steps, with gates between them to catch errors early.
- Routing: classify, then specialise; evaluate the router and provide a fallback.
- Parallelisation: sectioning for independent subtasks, voting for confidence; it saves time, not tokens.
- All three are workflows and compose together; unpredictable subtask counts call for orchestrator-worker instead.
