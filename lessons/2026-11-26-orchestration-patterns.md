# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-26

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you decide a fixed workflow fits your task, you still have to shape it. Three building-block patterns cover most production pipelines: chaining calls in sequence, routing inputs to specialised handlers, and running independent work in parallel. Knowing which one fits a requirement (accuracy, cost, latency, robustness) is a core exam skill and a daily design decision.

## Core concepts

### Prompt chaining

A task is decomposed into a **sequence of steps**, where each LLM call processes the output of the previous one. Between steps you can insert programmatic **gates**: code checks that validate intermediate output and stop or redirect the pipeline if something is wrong.

- **Use when** the task splits cleanly into fixed subtasks, and you want to trade some latency for higher accuracy because each call is easier.
- **Why it works**: a focused prompt on a narrow subtask is more reliable than one giant prompt doing everything.
- **Typical examples**: write an outline, check it against criteria, then write the document from the outline; draft marketing copy, then translate it.
- **Gates** matter: validate JSON schema, check length, or run a cheap classifier before passing output to the next stage. This catches errors early instead of compounding them.

### Routing

An initial step **classifies the input** and directs it to a specialised downstream prompt, tool set or model. This lets you optimise each path separately without one prompt trying to serve every case.

- **Use when** there are distinct categories that are better handled separately and classification can be done accurately (by an LLM or a traditional classifier).
- **Typical examples**: triage support tickets into billing, technical and refund flows; send easy questions to a small, fast model (such as Haiku) and hard ones to a larger model.
- **Benefits**: better quality per category, lower cost through model selection, and easier testing per route.
- **Risk**: a misrouted input gets the wrong specialist. Include a fallback or "other" route, and evaluate the router itself as its own component.

### Parallelisation

Independent work runs **simultaneously**, and the results are aggregated by code. It has two main variations:

- **Sectioning**: split a task into independent subtasks and run them at once. Example: one call extracts entities while another checks tone and a third checks policy compliance; or process each chunk of a long document separately.
- **Voting**: run the **same task several times** and compare outputs to raise confidence. Example: three independent reviews of a code change for vulnerabilities, flagging if any (or a majority) object.

- **Use when** subtasks are independent and you need speed, or when multiple perspectives raise confidence.
- **Benefit**: latency is roughly that of the slowest branch rather than the sum.
- **Cost**: total tokens and spend are the same or higher (voting multiplies them), and you must handle partial failures and rate limits.

### Choosing between them

| Need | Pattern |
|---|---|
| Steps depend on previous output | Chaining |
| Different input types need different handling | Routing |
| Independent subtasks, lower latency | Parallelisation (sectioning) |
| Higher confidence on a risky judgement | Parallelisation (voting) |

### Combining patterns

These compose. A realistic system routes a request, runs a chain for the chosen route, and parallelises a validation step inside it. All three remain **workflows**: your code controls the flow, which keeps them testable and predictable. Dynamic task decomposition by the model is a different pattern (orchestrator-workers, a later lesson).

## Example

A support pipeline that routes, then chains, with a parallel safety check:

```python
import asyncio

async def handle_ticket(ticket: str) -> str:
    # Routing: cheap model classifies the ticket
    route = await call_claude(ROUTER_PROMPT, ticket, model="claude-haiku-5-5")
    if route not in {"billing", "technical", "refund"}:
        route = "other"                       # fallback route

    # Chaining: draft -> gate -> polish
    draft = await call_claude(PROMPTS[route], ticket)
    if len(draft) > 2000 or "TODO" in draft:  # programmatic gate
        draft = await call_claude(SHORTEN_PROMPT, draft)

    # Parallelisation (sectioning): independent checks at once
    tone, policy = await asyncio.gather(
        call_claude(TONE_CHECK_PROMPT, draft),
        call_claude(POLICY_CHECK_PROMPT, draft),
    )
    if "FAIL" in (tone, policy):
        return escalate_to_human(ticket, draft)
    return draft
```

Each stage can be evaluated on its own, and the two checks add the latency of one call, not two.

## Common mistakes / exam traps

- **Parallelising dependent steps.** If step B needs step A's output, it is a chain, not a parallel job.
- **Chaining without gates.** An early error flows through every later stage and gets amplified; validate between steps.
- **Treating routing as free.** A weak classifier degrades the whole system; measure router accuracy separately and provide a fallback route.
- **Assuming parallel means cheaper.** It reduces latency, not cost; voting increases cost by the number of runs.
- **Confusing these with orchestrator-workers.** In these three patterns the code fixes the structure; if the model decides the subtasks at runtime, that is a different pattern.

## Check yourself

1. What is a "gate" in a prompt chain and why use one?
2. Name two benefits of routing.
3. What is the difference between sectioning and voting?
4. Does parallelisation reduce total token cost? Explain.
5. A task has subtasks whose number depends on the input, discovered at runtime. Which family of pattern is this, and is it covered by these three?

<details><summary>Answers</summary>

1. A programmatic check between steps (for example schema validation or a length check) that stops or redirects the pipeline on bad intermediate output, preventing errors from compounding.
2. Any two of: specialised prompts per category give better quality; cheaper models can handle easy inputs; each route can be tested and tuned independently.
3. Sectioning splits a task into different independent subtasks run at once; voting runs the same task multiple times to compare results and raise confidence.
4. No. It reduces wall-clock latency (roughly the slowest branch). Total tokens are the same, and voting multiplies them.
5. Orchestrator-workers, where the model decomposes work dynamically. It is not covered by chaining, routing or parallelisation, which have code-defined structure.

</details>

## Key takeaways

- Chaining trades latency for accuracy; add gates between steps.
- Routing specialises handling per input type and can save cost; evaluate the router and keep a fallback.
- Parallelisation (sectioning or voting) cuts latency or raises confidence, but not cost.
- The patterns compose, and all remain code-controlled workflows.
