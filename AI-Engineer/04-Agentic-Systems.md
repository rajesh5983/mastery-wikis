# Module 4 — Agentic Systems

---

## ReAct Pattern

**Status:** ⬜ Not Started

**Definition:** ReAct (Reasoning + Acting) is an agent pattern where the model alternates between Thought (reasoning about what to do next), Action (calling a tool), and Observation (processing the tool result), repeating until the task is complete. This interleaving of reasoning and action outperforms either alone.

**Mental Model:** ReAct is how a detective works — think about what you know (Thought), call a witness (Action), hear their answer (Observation), update your theory, repeat until the case is solved.

**Common Misconceptions:**
- ReAct agents always complete tasks successfully — agents can get stuck in loops, misuse tools, or give up; robust systems need loop detection, max iteration limits, and graceful fallbacks.
- The Thought step is optional overhead — the Thought step is what makes agents reliable; skipping it causes impulsive tool calls and poor error recovery.

**Interview Skeleton:**
- What it is: a structured agent loop of alternating reasoning and tool use that enables complex multi-step task completion
- Why it matters: the foundational pattern underlying most production agents; understanding it enables debugging and improvement
- Example: trace a ReAct agent solving a multi-step data analysis task, showing Thought/Action/Observation at each step

**Free Resources:** https://langchain-ai.github.io/langgraph — LangGraph documentation covering ReAct agents and advanced agent patterns

---

## Tool Use

**Status:** ⬜ Not Started

**Definition:** Tool use (function calling) allows LLMs to request external capabilities during generation — database queries, API calls, code execution, file operations — by generating structured tool call objects that the application layer executes and returns as observations. This grounds the agent in real-time, accurate information.

**Mental Model:** Tool use is giving the model a phone — it can look things up, send messages, and run calculations in the real world rather than relying purely on memorised knowledge.

**Common Misconceptions:**
- Models always know when to use a tool vs answer from memory — models can call tools unnecessarily or miss tool calls when they should use them; tool descriptions and few-shot examples guide this behaviour.
- Tool schemas don't need careful design — poorly named tools or vague descriptions lead to incorrect tool selection and wrong parameter usage; treat tool schemas as a critical interface design.

**Interview Skeleton:**
- What it is: the mechanism by which LLMs request and receive results from external functions, APIs, or databases during generation
- Why it matters: tool use is what makes agents capable of taking actions in the world, not just generating text
- Example: design a set of tools for a data analysis agent and explain how you'd write tool descriptions to avoid ambiguous tool selection

**Free Resources:** https://langchain-ai.github.io/langgraph — LangGraph documentation covering tool definition, binding, and agent tool use patterns

---

## Multi-Agent Patterns

**Status:** ⬜ Not Started

**Definition:** Multi-agent systems decompose complex tasks across specialised agents that communicate through a shared state, message passing, or an orchestrating supervisor agent. Common patterns include the Supervisor (one agent routes subtasks to specialists), Hierarchical (nested orchestrators), and Collaborative (agents debate or verify each other's output).

**Mental Model:** Multi-agent is like a project team — the project manager (supervisor) breaks down work, assigns it to specialists (subagents), and integrates the results. No single person does everything.

**Common Misconceptions:**
- More agents always solve harder problems — additional agents add coordination overhead, error propagation risk, and cost; use multi-agent only when decomposition genuinely helps.
- Agents can always communicate reliably — agent-to-agent communication must be explicitly designed; agents passing ambiguous or incomplete context to each other is a common failure mode.

**Interview Skeleton:**
- What it is: architectures that decompose complex tasks across multiple specialised LLM agents with defined communication patterns
- Why it matters: enables parallelism and specialisation for tasks too complex for a single agent loop
- Example: design a multi-agent pipeline for automated code review with separate agents for security, style, and correctness

**Free Resources:** https://langchain-ai.github.io/langgraph — LangGraph documentation covering supervisor, hierarchical, and collaborative multi-agent patterns

---

## Memory Systems

**Status:** ⬜ Not Started

**Definition:** Agent memory spans four types: in-context (current conversation window), external/episodic (retrieved from a vector store), procedural (instructions and workflows the agent follows), and semantic (factual knowledge about the world or user). Long-running agents need explicit memory management to stay effective beyond a single conversation.

**Mental Model:** Agent memory mirrors human memory types — working memory (in-context), long-term episodic memory (vector store), skills and habits (procedural), and general knowledge (semantic). Managing them well is what makes agents feel coherent over time.

**Common Misconceptions:**
- The context window is enough memory for production agents — context windows fill quickly in long tasks; without external memory, agents lose important context and repeat work.
- All memory should be stored and retrieved — indiscriminate storage creates noise; memory systems need selection logic about what is worth storing and how to retrieve it relevantly.

**Interview Skeleton:**
- What it is: the set of mechanisms an agent uses to persist, retrieve, and update information across steps and sessions
- Why it matters: without memory management, agents are stateless and cannot reason across long tasks or sessions
- Example: design the memory architecture for a coding assistant that must remember user preferences, past bugs, and code style across sessions

**Free Resources:** https://langchain-ai.github.io/langgraph — LangGraph documentation covering agent state, memory stores, and persistence patterns

---

## MCP (Model Context Protocol)

**Status:** ⬜ Not Started

**Definition:** MCP (Model Context Protocol) is an open protocol developed by Anthropic that standardises how AI models connect to external data sources, tools, and services. MCP servers expose resources, tools, and prompts through a standard interface; MCP clients (LLM hosts) consume them, enabling a plug-and-play ecosystem for agent capabilities.

**Mental Model:** MCP is like USB for AI agents — a standard connector that lets any agent plug into any tool or data source without custom integration code for every combination.

**Common Misconceptions:**
- MCP is only for Anthropic's Claude — MCP is an open protocol; any LLM host that implements the client can use any MCP server, making it provider-agnostic.
- MCP replaces function calling — MCP is a transport and discovery layer; function calling is the mechanism models use to invoke tools; they are complementary.

**Interview Skeleton:**
- What it is: a standardised protocol for connecting AI agents to external tools, data sources, and services
- Why it matters: enables composable agent capabilities without custom integration code for every tool/model combination
- Example: describe how you'd expose a database query tool as an MCP server and consume it from an agent

**Free Resources:** https://langchain-ai.github.io/langgraph — LangGraph documentation covering MCP integration and agent tool ecosystems

---

## Computer Use

**Status:** ⬜ Not Started

**Definition:** Computer use agents can interact with a computer's graphical interface — clicking, typing, scrolling, taking screenshots — rather than just calling APIs. This enables automation of tasks in applications that have no API, making agents capable of operating any software a human can.

**Mental Model:** Computer use is giving the agent eyes and hands in front of a screen. Instead of calling an API, it looks at a screenshot, clicks the button it sees, and reads the result — like a remote-control human operator.

**Common Misconceptions:**
- Computer use is a reliable production-ready capability for all workflows — computer use is powerful but slower, more expensive, and less reliable than API-based tool use; use it only when no API alternative exists.
- Computer use agents are fully autonomous and need no guardrails — agents interacting with UIs can accidentally trigger destructive actions; human-in-the-loop checkpoints are essential for high-stakes workflows.

**Interview Skeleton:**
- What it is: agent capability to interact with graphical interfaces by interpreting screenshots and generating mouse/keyboard actions
- Why it matters: expands the scope of automation to any software, not just those with APIs
- Example: describe a workflow where computer use is the right tool vs where an API integration would be preferable

**Free Resources:** https://langchain-ai.github.io/langgraph — LangGraph documentation on agent capabilities and computer use patterns

---

## Coding Agents

**Status:** ⬜ Not Started

**Definition:** Coding agents are LLM-based systems that can write, execute, test, and iterate on code autonomously. They typically have access to a code interpreter, file system, terminal, and often a web search tool. They follow a generate-execute-observe-fix loop until the task is complete.

**Mental Model:** A coding agent is a junior developer who has their own computer — they write code, run it, see what breaks, read the error, fix it, and repeat until the tests pass, without needing to ask for help on every step.

**Common Misconceptions:**
- Coding agents can replace human developers for complex software engineering tasks — current coding agents excel at well-scoped tasks with clear acceptance criteria; open-ended system design and architecture still require human judgement.
- Generated code that executes successfully is correct code — code can run without errors but still produce wrong outputs, have security vulnerabilities, or fail on edge cases; review and testing remain essential.

**Interview Skeleton:**
- What it is: agents that autonomously write, execute, debug, and iterate on code using a code interpreter and tool access
- Why it matters: dramatically accelerates scripting, data analysis, and boilerplate generation; understanding their limits prevents over-reliance
- Example: describe the system design for a coding agent that can complete a data analysis task end-to-end, including how you'd sandbox execution

**Free Resources:** https://langchain-ai.github.io/langgraph — LangGraph documentation covering code execution agents and safe sandboxing patterns
