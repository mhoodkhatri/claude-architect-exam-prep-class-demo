# What an agent is: the agentic loop (model → tool call → result → model)

Date: 2026-11-07

Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Almost every Claude-powered system that does real work, from a coding assistant to a support bot that looks up orders, is built on the agentic loop. If you understand exactly who does what in the loop (Claude decides, your code executes), you can reason about cost, latency, safety and failure modes. This is the base concept for every later topic: workflows, orchestration, subagents and human-in-the-loop.

## Core concepts

### What is an agent?

An agent is a system where the model **directs its own process**: it decides which tools to call, in what order, and when the task is done. This differs from a plain LLM call (one prompt in, one answer out) and from a fixed workflow (your code decides the steps). The simplest definition: *an LLM using tools in a loop until it reaches a stopping point.*

### The loop

1. Your code sends the model a request: a system prompt, the conversation so far, and a list of tool definitions.
2. Claude responds. If it wants to act, the response contains one or more `tool_use` blocks and `stop_reason: "tool_use"`.
3. **Your code** runs the tool (Claude never executes anything itself for client-side tools). You append Claude's response as an assistant message, then send the output back in a user message as `tool_result` blocks, each matched to its `tool_use_id`.
4. Claude sees the result and either calls another tool or produces a final answer with `stop_reason: "end_turn"`.
5. Repeat from step 2 until the model stops asking for tools.

The key point is that the model's decision, not your code, determines the number of iterations.

### Stop reasons drive control flow

- `tool_use`: run the tools and continue the loop.
- `end_turn`: Claude is finished; return the answer.
- `max_tokens`: output was truncated; the response may contain an incomplete tool call, so don't just execute it blindly.
- `pause_turn` (server-side tools): the turn was paused; send the conversation back to continue.
- `refusal`: the model declined; handle it explicitly.

Loop on `stop_reason`, not on parsing the text for phrases like "I'm done".

### State lives in the messages

The API is stateless. Each iteration you resend the full history, including every earlier tool call and result. This means context grows with every step, so long loops cost more and eventually hit the context window. Prompt caching, compaction and tool-result trimming (later syllabus topics) exist to manage this.

### Parallel tool calls

Claude may return several `tool_use` blocks in one response when the calls are independent. You should run them all and return all the `tool_result` blocks together in a single user message.

### Guardrails around the loop

Because the model controls iteration, you must add limits in your code:
- a **maximum iteration count** or token/cost budget,
- timeouts on tool execution,
- error handling that returns failures to Claude as `tool_result` with `is_error: true` so it can recover or try something else,
- validation and permission checks before executing sensitive tools.

### Ground truth each step

Agents work best when each step gives them real feedback from the environment (test output, query results, file contents). Feedback lets the model correct itself rather than guess.

## Example

A minimal loop in Python using the Anthropic SDK:

```python
import anthropic

client = anthropic.Anthropic()
MAX_STEPS = 10

tools = [{
    "name": "get_order_status",
    "description": "Look up the status of an order by its ID.",
    "input_schema": {
        "type": "object",
        "properties": {"order_id": {"type": "string"}},
        "required": ["order_id"],
    },
}]

def run_tool(name, args):
    if name == "get_order_status":
        return {"order_id": args["order_id"], "status": "shipped"}
    raise ValueError(f"unknown tool {name}")

messages = [{"role": "user", "content": "Where is order A-1042?"}]

for step in range(MAX_STEPS):
    response = client.messages.create(
        model="claude-sonnet-5-5",
        max_tokens=1024,
        tools=tools,
        messages=messages,
    )
    messages.append({"role": "assistant", "content": response.content})

    if response.stop_reason != "tool_use":
        break  # end_turn (or another terminal reason)

    results = []
    for block in response.content:
        if block.type == "tool_use":
            try:
                out = run_tool(block.name, block.input)
                results.append({"type": "tool_result",
                                "tool_use_id": block.id,
                                "content": str(out)})
            except Exception as e:
                results.append({"type": "tool_result",
                                "tool_use_id": block.id,
                                "content": f"Error: {e}",
                                "is_error": True})
    messages.append({"role": "user", "content": results})
else:
    raise RuntimeError("Agent exceeded step budget")
```

Note the assistant message is appended in full (including `tool_use` blocks) before the results are sent, and every `tool_use_id` gets a matching result.

## Common mistakes / exam traps

- **Thinking Claude runs the tool.** For client-side tools, your application executes them; Claude only requests the call.
- **Ending the loop by text matching** instead of checking `stop_reason`.
- **Forgetting the assistant turn or mismatching `tool_use_id`s**, which causes API errors because every `tool_use` must be followed by a corresponding `tool_result`.
- **No iteration or cost cap**, allowing runaway loops.
- **Raising an exception on tool failure** instead of returning the error as a `tool_result` so the model can adapt.

## Check yourself

1. What `stop_reason` tells you to execute tools and continue the loop?
2. Who executes a client-side tool: Claude or your application?
3. Why does cost per iteration tend to rise as an agent runs longer?
4. Claude returns three independent `tool_use` blocks in one response. How should you reply?
5. Name two safeguards you should put around an agentic loop.

<details><summary>Answers</summary>

1. `tool_use`.
2. Your application. Claude only emits the request.
3. The API is stateless, so the whole history (all prior tool calls and results) is resent each iteration and the context grows.
4. Run all three and return all three `tool_result` blocks, matched by `tool_use_id`, in a single user message.
5. Any two of: max iterations/token budget, tool timeouts, returning errors with `is_error: true`, permission/validation checks on sensitive tools.

</details>

## Key takeaways

- An agent is an LLM using tools in a loop, with the model deciding the path and when to stop.
- Control flow is driven by `stop_reason`: continue on `tool_use`, finish on `end_turn`.
- Your code executes tools and owns state, since the API is stateless and context grows each step.
- Always add budgets, timeouts and error handling; return tool failures to the model as errors.
