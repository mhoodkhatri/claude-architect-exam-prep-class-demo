# Workflows vs agents: when a fixed pipeline beats an autonomous agent

Date: 2026-11-07

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

The most common architecture mistake with Claude is reaching for a fully autonomous agent when a fixed sequence of LLM calls would be cheaper, faster and more predictable. Choosing correctly decides your latency, cost, testability and failure surface. The guiding principle is to use the simplest design that solves the problem, and add autonomy only when the task needs it.

## Core concepts

### Two kinds of agentic system

- **Workflow**: LLMs and tools are orchestrated through **predefined code paths**. Your code decides the steps and their order; Claude does the work inside each step.
- **Agent**: the LLM **dynamically directs its own process** and tool use, deciding the steps, the order and when to stop (the loop from the previous lesson).

Both are "agentic systems". The difference is who controls the flow: your code (workflow) or the model (agent).

### A spectrum, not a binary

Think of a ladder of increasing autonomy:

1. A single LLM call with good prompting, retrieval and examples.
2. A workflow of several fixed LLM calls (chain, router, parallel branches).
3. An agent that loops with tools.
4. Multiple agents coordinating.

Start at the bottom. Move up only when you can show, ideally with an evaluation, that the simpler level falls short.

### What workflows give you

- **Predictability**: the same input follows the same path, so behaviour is easy to reason about and audit.
- **Lower cost and latency**: no open-ended loop, no growing context from repeated exploration.
- **Easier testing**: each step can be evaluated on its own with fixed inputs and expected outputs.
- **Bounded failure**: you know the maximum number of calls and where an error can occur.
- **Easier compliance**: required steps (validation, redaction, approval) are enforced by code rather than requested in a prompt.

### What agents give you

- **Flexibility**: they handle open-ended problems where you cannot predict the number or order of steps (for example, fixing a bug across an unknown set of files).
- **Adaptation**: the model reacts to what each tool result reveals and changes plan.
- **Less hard-coding**: you describe the goal and tools instead of every branch.

The price is higher and more variable cost and latency, compounding errors over many steps, harder debugging and a need for guardrails (iteration caps, permissions, sandboxing).

### Decision heuristics

Prefer a **workflow** when:
- the steps are known in advance and rarely change,
- the task decomposes cleanly into fixed subtasks,
- you need consistency, strict ordering, or auditability,
- latency and cost matter, or the volume is high.

Prefer an **agent** when:
- the path depends on intermediate results and cannot be enumerated,
- the task is open-ended (research, coding, troubleshooting),
- you can trust the model's judgment in a sandboxed or reversible environment,
- the value per task justifies higher cost.

### Hybrids are normal

Real systems mix both. A deterministic workflow handles the predictable frame (classify the ticket, route it, validate the output, require approval), and a bounded agent handles the one open-ended step inside it. Putting a hard rule in code (for example "refunds over $500 need human approval") is more reliable than asking the model to remember it.

### Frameworks

Frameworks can speed up starting out, but they add layers that hide prompts and responses and make debugging harder. Understand the underlying API calls first, and only adopt an abstraction you can see through.

## Example

A support system that triages email. The stages are known, so the outer flow is a workflow. Only the investigation step is open-ended, so it is a bounded agent.

```python
def handle_ticket(ticket):
    # 1. Fixed step: classify (single cheap call, structured output)
    category = classify(ticket.text)          # "billing" | "bug" | "other"

    # 2. Fixed routing in code, not decided by the model
    if category == "billing":
        draft = billing_workflow(ticket)       # fixed 2-call chain
    elif category == "bug":
        # 3. Open-ended step: bounded agent with read-only tools
        draft = run_agent(
            task=ticket.text,
            tools=[search_logs, read_docs],    # least privilege
            max_steps=8,
        )
    else:
        draft = escalate_to_human(ticket)

    # 4. Fixed guardrails in code
    draft = redact_pii(draft)
    if needs_approval(draft):
        return queue_for_human(draft)
    return send(draft)
```

Billing is a workflow because the steps never vary. The bug path is an agent because the investigation depends on what the logs reveal. PII redaction and approval are in code, so they always run.

## Common mistakes / exam traps

- **Defaulting to an agent for everything.** If you can write the steps down in advance, a workflow is usually the better answer on cost, latency and reliability.
- **Treating "workflow" and "agent" as the same thing.** The distinguishing feature is who controls the flow (code versus model), not whether tools or multiple LLM calls are involved.
- **Enforcing critical rules only in the prompt.** Mandatory steps such as approvals, validation and limits belong in code, not in instructions the model might skip.
- **Adding complexity without measuring.** Move to a more autonomous design only when evaluations show the simpler one is insufficient.
- **Giving an agent broad tools and no limits** for a task that is really a fixed pipeline.

## Check yourself

1. In one sentence, what separates a workflow from an agent?
2. A nightly job always extracts fields from an invoice, validates them, then writes to a database. Workflow or agent? Why?
3. Name three advantages of a workflow over an agent.
4. Why is "orders over $500 require human approval" better implemented in code than in the system prompt?
5. What is the recommended starting point when designing a new Claude-based feature?

<details><summary>Answers</summary>

1. In a workflow, predefined code paths control the flow; in an agent, the LLM dynamically directs its own process and tool use.
2. Workflow. The steps and their order are known and fixed, so a pipeline is cheaper, faster and more predictable.
3. Any three of: predictability, lower cost, lower latency, easier testing, bounded failure, easier auditing and compliance.
4. Code enforces the rule every time, whereas a prompt instruction can be missed or overridden by the model. Hard requirements should be deterministic.
5. Start with the simplest option (a single well-prompted LLM call) and add workflow structure or agent autonomy only when evaluations show it is needed.

</details>

## Key takeaways

- Workflows follow code-defined paths; agents let the model decide the path.
- Use the simplest design that works, and increase autonomy only when measured results justify it.
- Workflows win on predictability, cost, latency and testability; agents win on open-ended, unpredictable tasks.
- Hybrid designs are common: a deterministic frame with a bounded agent for the open-ended step.
- Enforce non-negotiable rules in code, not in prompts.
