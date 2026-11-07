# What an agent is: the agentic loop (model → tool call → result → model)

Date: 2026-11-07
Syllabus section: 1. Agentic architecture and orchestration

## Why it matters

Almost every Claude-based system that does real work, such as a coding assistant, a support bot that looks up orders, or a research tool, is built on the agentic loop. If you understand how the loop starts, continues and stops, you can reason about cost, latency, safety and failure modes. The exam treats this loop as the foundation for everything else in the architecture domain.

## Core concepts

### What an agent is

An agent is a system where the model decides, step by step, which actions to take to reach a goal, using tools and feeding the results of those actions back into its own context. Anthropic's useful distinction: a *workflow* follows paths predefined in code, while an *agent* directs its own process and tool usage. The agentic loop is the mechanism that makes this possible.

### The loop

1. Your application sends the model a request: system prompt, conversation messages and a list of tool definitions.
2. The model responds. Either it produces a final answer, or it asks to call one or more tools.
3. If it asks for a tool, the response has `stop_reason: "tool_use"` and contains `tool_use` content blocks, each with an `id`, a tool `name` and JSON `input`.
4. **Your code** executes the tool (the model never runs your function itself for client-side tools) and appends a `tool_result` block, referencing the matching `tool_use_id`, to the conversation as a user message.
5. You call the model again with the extended history. Repeat until the stop reason is `end_turn` (or another terminal reason).

The loop is therefore just a `while` loop in your code. The model supplies the decisions; your code supplies execution, state and guardrails.

### The conversation is the state

The API is stateless. Every iteration resends the full message history, including all earlier tool calls and results. Consequences:

- Context grows with each iteration, so long loops hit token limits and cost more (see the context management section later in the syllabus).
- Large tool outputs are expensive because they are re-read on every later turn. Return concise, relevant results.
- The model "remembers" only what is in the messages you send.

### Stop reasons and termination

- `end_turn`: the model considers the task done. Exit the loop.
- `tool_use`: execute tools and continue.
- `max_tokens`: the output was truncated. This may leave a half-formed tool call, so handle it explicitly.
- `pause_turn` / `refusal`: other conditions your loop should handle rather than ignore.

Never rely on the model to stop on its own forever. Add a **maximum iteration count**, a token or cost budget and a timeout. These are the hard stopping conditions that protect you from runaway loops.

### Parallel tool calls and errors

A single response can contain several `tool_use` blocks. You must return a `tool_result` for **every** one, in the next user message. If a tool fails, return the error text with `is_error: true` instead of crashing. The model can often read the message, adjust its arguments and retry, which is a basic form of self-correction.

### Where the intelligence and the control live

| Concern | Owner |
|---|---|
| Choosing which tool to call and with what arguments | Model |
| Executing the tool, validating input, enforcing permissions | Your code |
| Deciding when the task is complete | Model, bounded by your limits |
| Persisting state, logging, budgets | Your code |

## Example

A minimal loop in Python using the Anthropic SDK:

```python
import anthropic

client = anthropic.Anthropic()
MAX_STEPS = 10

tools = [{
    "name": "get_order_status",
    "description": "Look up the shipping status of an order by its ID.",
    "input_schema": {
        "type": "object",
        "properties": {"order_id": {"type": "string"}},
        "required": ["order_id"],
    },
}]

def run_tool(name, args):
    if name == "get_order_status":
        return {"order_id": args["order_id"], "status": "shipped"}
    raise ValueError(f"Unknown tool {name}")

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
        break  # end_turn: final answer is in response.content

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
    print("Stopped: hit MAX_STEPS without a final answer")
```

Note the three key habits: append the assistant turn unchanged, answer every `tool_use` id, and cap the iterations.

## Common mistakes / exam traps

- **Thinking the model executes tools.** For client-side tools, the model only *requests* a call; your code runs it and returns the result.
- **Forgetting a `tool_result` for each `tool_use` block**, or not matching `tool_use_id`. The API will reject the next request.
- **No iteration or budget limit.** An unbounded loop can burn tokens indefinitely if the model keeps calling tools.
- **Parsing the text to decide whether to stop.** Use `stop_reason`, not phrases like "I'm done".
- **Dropping prior turns or tool results from the history.** The API is stateless; missing history means the model loses track of what it already did.

## Check yourself

1. What `stop_reason` tells your code to run a tool and call the model again?
2. In which message role do you send `tool_result` blocks back?
3. The model returns three `tool_use` blocks in one response. What must your next request contain?
4. Why does a long agentic loop get progressively more expensive per step?
5. Name two hard stopping conditions you should add besides the model's own `end_turn`.

<details><summary>Answers</summary>

1. `tool_use`.
2. The `user` role, as content blocks of type `tool_result`.
3. A user message containing a `tool_result` for each of the three, each with the matching `tool_use_id`.
4. The whole history, including all earlier tool calls and results, is resent every iteration, so input tokens grow with each step.
5. Any two of: a maximum iteration count, a token or cost budget, a wall-clock timeout.

</details>

## Key takeaways

- The agentic loop is model → tool call → your code runs it → tool result → model, repeated until `end_turn`.
- The model decides; your code executes, validates and enforces limits.
- The conversation history is the only state, so manage its growth.
- Always bound the loop and return tool errors as `is_error` results so the model can recover.
