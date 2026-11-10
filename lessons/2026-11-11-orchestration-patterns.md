# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-11

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a fixed workflow fits your task, you still have to shape it. Three building blocks cover most production pipelines: chaining steps in sequence, routing inputs to specialised handlers, and running independent work in parallel. Knowing which one fits, and what each costs in latency, accuracy and complexity, is a core architect skill and a frequent exam scenario.

## Core concepts

### Prompt chaining

Break a task into a **sequence of LLM calls**, where each call's output feeds the next. Between steps you can add programmatic **gates**: code checks that validate the intermediate result before continuing.

- **Use when** the task decomposes cleanly into fixed subtasks and you want higher accuracy by making each call simpler.
- **Trade-off**: you accept extra latency (several calls in series) in exchange for focus and accuracy per step.
- **Typical shapes**: outline then write the document; draft then translate; extract then validate then summarise.
- **Gates matter**: a cheap deterministic check (valid JSON, required fields present, length limit) after a step stops errors from compounding down the chain.

### Routing

A **classifier step** examines the input and directs it to one of several specialised downstream paths, each with its own prompt, tools or even model.

- **Use when** inputs fall into distinct categories that are better handled separately, and classification can be done accurately (by an LLM or a traditional classifier).
- **Benefits**: separation of concerns, shorter and sharper prompts per path, and **cost control** by sending easy queries to a smaller, cheaper model and hard ones to a larger model.
- **Typical shapes**: support triage (billing vs technical vs refund), question difficulty routing, language or document-type routing.
- **Risk**: a misrouted input gets the wrong specialist. Include a fallback or "other" route and evaluate the router on its own.

### Parallelisation

Run multiple LLM calls **at the same time** and aggregate the results programmatically. Two variants:

- **Sectioning**: split a task into **independent subtasks** that run concurrently (for example, one call checks a document for policy violations while another answers the user's question; or review each file of a change separately).
- **Voting**: run the **same task several times** to get diverse outputs or higher confidence (for example, several reviewers each flag vulnerabilities, and you act on agreement or a threshold).

- **Use when** subtasks are independent and you need speed, or when multiple perspectives raise confidence.
- **Trade-off**: lower wall-clock latency than a sequential chain but the same or higher total token cost. Each parallel call needs its own focused context.
- **Requirement**: the pieces must truly be independent. If step B needs A's output, that is a chain, not parallel work.

### Choosing between them

| Situation | Pattern |
|---|---|
| Steps depend on previous output, fixed order | Prompt chaining |
| Different input types need different handling | Routing |
| Independent subtasks, want lower latency | Parallelisation (sectioning) |
| Want higher confidence on a judgment call | Parallelisation (voting) |

### Combining patterns

These compose. A common design: **route** a ticket to a category, run a **chain** for that category, and inside one step **parallelise** several independent checks. Keep each stage testable on its own with its own evaluation set.

### What they have in common

All three are **workflows**: your code decides the control flow. The model never chooses the next step. When the subtasks cannot be known in advance, you need the orchestrator-worker pattern (next lesson) or an agent.

## Example

A contract review service: classify the document, check clauses in parallel, then produce a summary.

```python
import asyncio

async def review_contract(text: str) -> dict:
    # Routing: choose the handler (and model) by document type
    doc_type = await call_claude(CLASSIFY_PROMPT, text, model="small")
    handler = HANDLERS.get(doc_type, HANDLERS["other"])   # fallback route

    # Parallelisation (sectioning): independent checks run concurrently
    liability, termination, payment = await asyncio.gather(
        call_claude(handler.liability_prompt, text),
        call_claude(handler.termination_prompt, text),
        call_claude(handler.payment_prompt, text),
    )

    # Gate: deterministic check before continuing the chain
    findings = [liability, termination, payment]
    if not all(valid_schema(f) for f in findings):
        return route_to_human(text, reason="malformed findings")

    # Prompt chaining: final step consumes earlier outputs
    summary = await call_claude(SUMMARY_PROMPT, findings, model="large")
    return {"type": doc_type, "findings": findings, "summary": summary}
```

Routing picks the path, parallel calls cut latency, the gate catches bad intermediate output, and the final call is a chained step that depends on everything before it.

## Common mistakes / exam traps

- **Parallelising dependent steps.** If one call needs another's output, running them together is impossible or yields wrong results. That is a chain.
- **Assuming parallel is cheaper.** It reduces latency, not token cost. Voting multiplies cost.
- **Routing without a fallback or router evaluation.** A weak classifier silently degrades every downstream path.
- **Chaining without gates.** Errors in an early step propagate and compound. Validate between steps.
- **Using a chain or router when the steps cannot be known up front.** These are fixed-path patterns; open-ended work needs dynamic orchestration or an agent.

## Check yourself

1. What is the main trade-off of prompt chaining?
2. Why might a router send some queries to a smaller model?
3. Distinguish sectioning from voting.
4. Does parallelisation reduce token cost? Explain.
5. In these three patterns, who controls the flow, the code or the model?

<details><summary>Answers</summary>

1. Higher accuracy per step (simpler, focused calls) in exchange for added latency from running calls in series.
2. Easy queries do not need a large model, so routing them to a smaller, cheaper, faster model reduces cost and latency without hurting quality; hard queries go to a larger model.
3. Sectioning splits a task into different independent subtasks run concurrently; voting runs the same task multiple times to get diverse outputs or higher confidence.
4. No. It reduces wall-clock latency, but total tokens are the same or higher (voting multiplies calls).
5. The code. All three are workflows with predefined control flow; the model only fills in each step.

</details>

## Key takeaways

- Chaining trades latency for accuracy; add programmatic gates between steps.
- Routing specialises handling and controls cost, but needs a good, separately evaluated classifier and a fallback.
- Parallelisation (sectioning or voting) cuts latency or raises confidence, only for independent work, and does not reduce cost.
- The patterns are workflows and compose freely; move to orchestrator-worker or agents when subtasks cannot be predetermined.
