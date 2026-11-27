# Orchestration patterns: prompt chaining, routing, parallelisation

Date: 2026-11-28

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Once you have decided a task suits a workflow, you still have to shape it. Three building-block patterns (prompt chaining, routing and parallelisation) cover most production pipelines built on Claude. Knowing which one fits a task's structure lets you raise accuracy, cut cost and keep latency bounded without handing control to an autonomous agent. The exam tests whether you can match a scenario to the right pattern and spot when a pattern is misapplied.

## Core concepts

### Prompt chaining

A task is decomposed into a **fixed sequence of steps**, where each Claude call processes the output of the previous one. Your code can insert **gates** between steps: programmatic checks that validate intermediate output and stop, retry or reroute if it is wrong.

- **Use when** the task splits cleanly into subtasks and you want to trade some latency for higher accuracy, because each call is easier than the whole.
- **Why it works**: smaller, focused prompts are easier to write, test and debug, and each stage can have its own evaluation set.
- **Typical shapes**: write an outline, check it against criteria, then write the document; extract, validate, then summarise; draft in one language, then translate.
- **Cost**: latency and tokens grow roughly linearly with the number of steps.

### Routing

An initial step **classifies the input** and sends it to a specialised downstream path, each with its own prompt, tools or even model.

- **Use when** inputs fall into distinct categories that are better handled separately, and classification can be done accurately (by an LLM or a traditional classifier).
- **Why it works**: one prompt trying to serve every category gets cluttered and optimises for none. Separate prompts let you tune each path without hurting the others.
- **Cost optimisation**: route easy, common requests to a smaller, faster model and hard ones to a more capable model.
- **Typical shapes**: customer support (refund, technical, billing, general), query triage by difficulty, sending code questions and prose questions to different prompts.
- **Design details**: include a sensible **fallback route** for unclear or low-confidence classifications (often a human or a general handler), and evaluate the router separately, since a misroute poisons everything after it.

### Parallelisation

Several LLM calls run **at the same time** and their outputs are aggregated by code. There are two variants:

- **Sectioning**: split a task into **independent subtasks** run in parallel. Example: one call answers the user while another screens the request for policy violations; or each document in a batch is analysed separately.
- **Voting**: run the **same task several times** and combine results for confidence. Example: three independent reviews of code for vulnerabilities, flagging if any (or a majority) report a problem.

Use it when subtasks are independent and you want speed, or when multiple perspectives raise confidence. It is also useful when a call with a single focused concern performs better than one call juggling many (for example, separate calls for each evaluation criterion).

Key constraint: the subtasks **must not depend on each other's output**. If step B needs step A's result, that is a chain, not parallelisation. Your code must also handle partial failure: decide what happens if one of five parallel calls errors or times out.

### Choosing between them

| Task shape | Pattern |
|---|---|
| Steps depend on each other in order | Chaining |
| Input type determines the right handling | Routing |
| Independent pieces, or need several opinions | Parallelisation |

### Combining patterns

They compose. A support system might **route** by ticket type, run a **chain** (draft, then policy-check gate, then polish) on the chosen path, and use **parallel** calls to check tone and factual accuracy at the end. All of this remains a workflow: your code decides the path.

## Example

A support pipeline that routes, then chains, with a parallel guardrail:

```python
import asyncio

async def handle_ticket(ticket: str) -> str:
    # ROUTING: classify, with a safe fallback
    label = await call_claude(ROUTER_PROMPT, ticket, model="small-fast-model")
    if label not in {"billing", "technical", "refund"}:
        return escalate_to_human(ticket, reason="unclear category")

    # PARALLELISATION (sectioning): draft and safety screen are independent
    draft, safe = await asyncio.gather(
        call_claude(PROMPTS[label], ticket, model="capable-model"),
        call_claude(SCREEN_PROMPT, ticket, model="small-fast-model"),
    )
    if safe.strip() != "OK":
        return escalate_to_human(ticket, reason="flagged by screen")

    # PROMPT CHAINING with a gate: check the draft before polishing it
    check = await call_claude(POLICY_CHECK_PROMPT, draft)
    if "VIOLATION" in check:
        draft = await call_claude(REVISE_PROMPT, draft + "\n" + check)

    return await call_claude(POLISH_PROMPT, draft)
```

The code, not the model, decides every transition, and each stage can be tested on its own.

## Common mistakes / exam traps

- **Parallelising dependent steps.** If one call needs another's output, it is a chain. Running them "in parallel" yields wrong or empty inputs.
- **Routing without a fallback.** Unclear or low-confidence inputs need a default or human path, or they get mishandled confidently.
- **Chaining with no gates.** An error in step one is passed silently through every later step; validate intermediate output in code.
- **Using one giant prompt where chaining or routing fits.** Overloaded prompts degrade; splitting by step or category is usually the fix.
- **Confusing voting with sectioning.** Sectioning splits *different* subtasks; voting repeats the *same* task for confidence.

## Check yourself

1. What is the main trade-off of prompt chaining, and what is a gate?
2. Which pattern lets you send simple queries to a cheaper model and hard ones to a stronger model?
3. Name the two variants of parallelisation and give one use for each.
4. A task has steps where step 3 needs the output of step 2. Can you parallelise them?
5. Why should the router in a routing system be evaluated separately?

<details><summary>Answers</summary>

1. You trade extra latency (and tokens) for higher accuracy because each call is simpler. A gate is a programmatic check between steps that validates intermediate output and can stop, retry or reroute.
2. Routing: classify by difficulty or type, then send each class to the appropriate model or prompt.
3. Sectioning (independent subtasks run concurrently, e.g. answering plus a safety screen) and voting (the same task run several times for confidence, e.g. multiple vulnerability reviews).
4. No. Dependent steps form a chain and must run in sequence.
5. A misclassification sends the input down the wrong path, so errors there cascade. Measuring router accuracy separately shows where failures originate and lets you fix them independently.

</details>

## Key takeaways

- Chaining: fixed sequential steps with optional gates; trades latency for accuracy.
- Routing: classify, then specialise; add a fallback and optionally use cheaper models for easy cases.
- Parallelisation: sectioning for independent subtasks, voting for confidence; subtasks must be independent.
- The patterns compose, and all remain workflows because your code controls the flow.
