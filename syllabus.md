# Syllabus

The nightly **Daily Lesson** workflow (`.github/workflows/daily-lesson.yml`) reads this file every night at 9 PM Pakistan time. It picks the **first unchecked topic**, writes a lesson for it in `lessons/`, and ticks it off here.

Edit freely: reorder, add or remove topics. Keep the `- [ ]` / `- [x]` format so the workflow can track progress. Check the list against the official exam guide when you have it.

## 1. Agentic architecture and orchestration

- [x] What an agent is: the agentic loop (model → tool call → result → model) → [lesson](lessons/2026-11-07-the-agentic-loop.md)
- [x] Workflows vs agents: when a fixed pipeline beats an autonomous agent → [lesson](lessons/2026-11-07-workflows-vs-agents.md)
- [x] Orchestration patterns: prompt chaining, routing, parallelisation → [lesson](lessons/2026-11-13-orchestration-patterns.md)
- [ ] Orchestrator–worker and multi-agent designs (subagents, context isolation)
- [ ] Evaluator–optimiser loops and self-correction
- [ ] Human-in-the-loop: approvals, checkpoints and stopping conditions

## 2. Tool design and MCP integration

- [ ] Tool use basics: tool definitions, input schemas, tool_use / tool_result blocks
- [ ] Designing good tools: naming, descriptions, granularity, error messages
- [ ] Model Context Protocol: hosts, clients, servers; tools, resources, prompts
- [ ] Connecting MCP servers to Claude Code and the API (stdio vs remote)
- [ ] Tool security: least privilege, permission prompts, untrusted tool output

## 3. Claude Code configuration and workflows

- [ ] CLAUDE.md files: project, user and directory scopes
- [ ] settings.json: permissions (allow / deny), environment, model choice
- [ ] Slash commands, skills and subagents
- [ ] Hooks: running scripts on tool events
- [ ] Headless mode and Claude Code in CI (`claude -p`, claude-code-action)
- [ ] Scheduled and event-driven automation with GitHub Actions

## 4. Prompt engineering and structured output

- [ ] Clear instructions: role, context, examples, success criteria
- [ ] XML tags and prompt structure for long inputs
- [ ] Few-shot examples and chain-of-thought / extended thinking
- [ ] Structured output: JSON schemas, tool-forced output, validation and retries
- [ ] Reducing hallucinations: grounding, quoting sources, permission to say "I don't know"

## 5. Context management and reliability

- [ ] Context windows and token budgeting
- [ ] Prompt caching: how it works and what it costs
- [ ] Long-running sessions: compaction, summarisation, memory files
- [ ] Retrieval (RAG) and choosing what goes into context
- [ ] Evaluations: building test sets and measuring prompt changes
- [ ] Reliability in production: rate limits, retries, fallbacks, monitoring
