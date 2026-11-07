# Workflows vs agents: when a fixed pipeline beats an autonomous agent

Date: 2026-11-07

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

The most common design error in Claude systems is reaching for a fully autonomous agent when a simple pipeline would do. Agents trade predictability, cost and latency for flexibility, so choosing between the two is an architectural decision you should make deliberately. The exam expects you to justify the choice from the shape of the task, not from what sounds more advanced.

## Core concepts

### The two definitions

- **Workflow**: LLM calls and tools are orchestrated through **predefined code paths**. Your code decides the sequence, the branches and when to stop. Claude fills in each step.
- **Agent**: the LLM **dynamically directs its own process** and tool usage, looping until it decides the task is done (the loop from the previous lesson).

Both are "agentic systems". The difference is who controls the flow: your code (workflow) or the model (agent).

### Start with the simplest thing that works

The recommended ladder, in order of increasing complexity:

1. A single LLM call with a good prompt (plus retrieval and a few examples).
2. A workflow of several fixed calls.
3. An autonomous agent.

Only move up when measurements show the simpler option is not good enough. Every step up adds latency, cost and more ways to fail.

### When a workflow wins

Choose a fixed pipeline when:
- The steps are **known in advance** and rarely change (extract fields, validate, summarise, file the result).
- You need **predictable behaviour**, auditability or compliance: the same input should follow the same path.
- **Latency and cost** must be bounded. A pipeline of three calls costs roughly three calls; an agent's cost varies with how long it wanders.
- You can **test each stage** independently with its own evaluation set.
- The failure of a step must be handled in a **specific, scripted way** (retry, route to a human, fall back).

### When an agent wins

Choose an agent when:
- The number and order of steps **cannot be predicted** (debugging a failing test, researching an open-ended question, navigating a codebase).
- The task needs **branching based on what is discovered** in the environment.
- You can trust the model within **guardrails**: sandboxed tools, budgets, permission checks.
- Each step gives **real feedback** (test output, search results) so the model can self-correct.
- The value of the task justifies higher cost and the tolerance for variability.

### The deciding questions

| Question | Points to workflow | Points to agent |
|---|---|---|
| Can I write the steps down ahead of time? | Yes | No |
| Is the path the same for most inputs? | Yes | No, it varies a lot |
| Is the cost of an error high and must it be predictable? | Yes | Lower stakes or strong guardrails |
| Is latency tightly bounded? | Yes | Flexible |
| Is there environment feedback to steer by? | Not needed | Available |

### Hybrids are normal

Real systems mix both. A workflow can have one agentic step (a pipeline whose "investigate" stage is a bounded agent), and an agent can call deterministic workflows as tools. Put determinism where the steps are known and autonomy only where the path is genuinely open.

### Frameworks are optional

Frameworks can speed up setup, but they add abstraction layers that hide prompts and responses and make debugging harder. Understand the underlying API calls first; many patterns take only a few lines of code.

## Example

Task A: process incoming invoices. Every invoice needs the same treatment: extract fields, validate totals, classify the expense, write to the ledger. This is a workflow.

```python
def process_invoice(pdf_text: str) -> dict:
    fields = call_claude(EXTRACT_PROMPT, pdf_text)          # step 1: structured extraction
    if not totals_match(fields):                            # step 2: deterministic check in code
        return route_to_human(fields, reason="totals mismatch")
    category = call_claude(CLASSIFY_PROMPT, fields)         # step 3: classification
    return write_to_ledger(fields, category)                # step 4: fixed action
```

The path is fixed, each stage is testable, and cost is about two model calls per invoice.

Task B: "Find why the checkout service started returning 500s since yesterday." You cannot know in advance whether to read logs, inspect a deploy diff, query metrics or run a test. This needs an agent with read-only tools, a step cap and a requirement to cite evidence, and likely a human approving any remediation.

Same company, same model, different architecture, because the shape of the task differs.

## Common mistakes / exam traps

- **Defaulting to an agent** for a task with a known sequence. This raises cost and variance for no gain.
- **Building a rigid pipeline for an open-ended task**, then piling on special-case branches until it is a poor imitation of an agent.
- **Treating "agentic" as meaning "autonomous"**. Workflows are also agentic systems; the exam distinction is who controls the flow.
- **Letting the model make decisions that code can make reliably** (arithmetic, schema validation, permission checks). Do these deterministically.
- **Skipping the single-call baseline**. A well-prompted single call often beats a complex system and should be your starting benchmark.

## Check yourself

1. In one sentence, what separates a workflow from an agent?
2. A nightly job extracts fields from receipts, validates them and stores them. Workflow or agent, and why?
3. Name two properties of a task that favour an agent.
4. What should you try before building either a workflow or an agent?
5. Can a system be both? Give an example.

<details><summary>Answers</summary>

1. In a workflow, predefined code paths control the flow; in an agent, the model dynamically decides the steps and when to stop.
2. Workflow: the steps are known and identical for every input, so it is cheaper, predictable and each stage can be tested.
3. Any two of: steps cannot be predicted in advance; branching depends on discoveries; environment feedback is available; guardrails make autonomy safe; the task's value justifies variable cost.
4. A single, well-prompted LLM call (with retrieval or examples as needed), used as a baseline and only escalated if measurements show it is insufficient.
5. Yes. For example, a fixed triage pipeline whose "investigate" stage is a budget-limited agent, or an agent that calls deterministic workflows as tools.

</details>

## Key takeaways

- Workflows: code controls the flow. Agents: the model controls the flow.
- Start with a single call, then a workflow, and only then an agent, justified by measured need.
- Use workflows for known, repeatable, auditable steps; use agents for open-ended tasks with environment feedback and guardrails.
- Hybrids are common: keep determinism wherever the steps are known.
