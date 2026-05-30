# Module 4 — Agentic Systems

---

## ReAct Pattern

**Status:** ⬜ Not Started

**Definition:** ReAct (Reasoning + Acting) is an agent pattern where the model alternates between Thought (reasoning about what to do next), Action (calling a tool), and Observation (processing the tool result), repeating until the task is complete. This interleaving of reasoning and action outperforms either alone.

**Key Mental Model:** ReAct is how a detective works — think about what you know (Thought), call a witness (Action), hear their answer (Observation), update your theory, repeat until the case is solved.

**How It Works:**
- The agent loop starts with the user query and the available tool definitions injected into the context. The model generates a Thought token sequence ("I need to find the current price..."), then an Action token sequence specifying the tool name and parameters in structured format, then halts and yields control to the application.
- The application layer executes the requested tool call, captures the result, and appends it to the conversation history as an Observation message. The full context (original query + all prior Thought/Action/Observation turns) is then sent back to the model for the next iteration.
- The model decides at each step whether to call another tool or to generate a final answer. This is a conditional branch in the generation: if the model produces a final answer token sequence rather than a tool call, the loop terminates. Frameworks like LangGraph encode this as a graph edge condition.
- State accumulates across loop iterations in the conversation history. Each new context includes all prior reasoning steps, making the agent's decisions traceable. This also means context window consumption grows linearly with the number of loop iterations — long-running agents can exhaust their context budget.
- Loop termination requires explicit safeguards: a maximum iteration count (to prevent infinite loops), a cost or token budget limit, and a graceful fallback response when limits are hit. Without these, a poorly-scoped task can run indefinitely.

**Common Misconceptions:**
- ReAct agents always complete tasks successfully — agents can get stuck in loops, misuse tools, or give up; robust systems need loop detection, max iteration limits, and graceful fallbacks.
- The Thought step is optional overhead — the Thought step is what makes agents reliable; skipping it causes impulsive tool calls and poor error recovery.

**Interview Answer Skeleton:**
- **What it is:** An agent loop pattern that interleaves LLM-generated reasoning (Thought), structured tool calls (Action), and tool result processing (Observation) in repeated cycles until the task terminates with a final answer.
- **Why it matters / trade-offs:** ReAct is the foundational pattern for most production agents. Understanding how the loop state accumulates in context explains why agents drift, get confused, or run up costs. Proper loop termination logic and iteration limits are non-negotiable for production deployment.
- **Example or context:** Trace a ReAct agent on "what was AAPL's stock price change yesterday?": Thought: need current price and yesterday's price. Action: call price_lookup("AAPL", "today") → $182. Action: call price_lookup("AAPL", "yesterday") → $179. Thought: change is +$3 or +1.7%. Final Answer: AAPL rose $3 (1.7%) yesterday.

**Free Resources:**
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph) — Framework implementation of ReAct agents with graph-based state management and loop control
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Production ReAct agent examples with tool use, error handling, and termination patterns

---

## Tool Use

**Status:** ⬜ Not Started

**Definition:** Tool use (function calling) allows LLMs to request external capabilities during generation — database queries, API calls, code execution, file operations — by generating structured tool call objects that the application layer executes and returns as observations. This grounds the agent in real-time, accurate information.

**Key Mental Model:** Tool use is giving the model a phone — it can look things up, send messages, and run calculations in the real world rather than relying purely on memorised knowledge.

**How It Works:**
- Tool definitions are passed to the model as a list of JSON Schema objects, each specifying the tool's name, description, and parameters schema. The model's training includes examples of choosing the right tool and generating valid parameter arguments given these definitions.
- When the model decides to call a tool, it generates a structured `tool_use` content block (Anthropic) or `function_call` object (OpenAI) rather than plain text. The generation stops at this point — the model yields control back to the application to execute the call.
- The application layer executes the function with the model-provided parameters, captures the return value, and sends it back to the model as a `tool_result` message in the next API call. The model then has both its reasoning and the concrete result available for the next step.
- Tool selection behaviour is determined by the tool descriptions. The model selects tools based on the name and description matching the current task context. A description of "search the knowledge base for relevant documents" is unambiguous; "get information" is too vague and leads to incorrect selection. Treat tool schemas as a critical API design problem.
- Parallel tool calling allows the model to request multiple tool calls in a single response when the calls are independent (e.g., looking up two different data sources simultaneously). The application executes them concurrently and returns all results before the model continues. This reduces round-trip latency for agents with independent subtasks.

**Common Misconceptions:**
- Models always know when to use a tool vs answer from memory — models can call tools unnecessarily or miss tool calls when they should use them; tool descriptions and few-shot examples guide this behaviour.
- Tool schemas don't need careful design — poorly named tools or vague descriptions lead to incorrect tool selection and wrong parameter usage; treat tool schemas as a critical interface design.

**Interview Answer Skeleton:**
- **What it is:** The mechanism by which the model generates structured tool call objects during inference, which the application layer executes and returns as observations — enabling agents to take actions and access real-time information beyond their training data.
- **Why it matters / trade-offs:** Tool use is what transforms a text generator into an agent capable of acting in the world. Schema quality directly determines tool selection accuracy. Parallel tool calling, error handling on tool failures, and safe execution sandboxing are all production engineering concerns.
- **Example or context:** Design a data analysis agent's tool set: `execute_sql(query: str, database: str)`, `read_csv(path: str)`, `plot_chart(data: dict, chart_type: str)`. The schema descriptions should specify what each parameter expects — "SQL query string, SELECT only, no DDL statements" is better than "a SQL query." Add a tool-call validation layer that rejects DDL before execution.

**Free Resources:**
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph) — Covers tool definition, binding, parallel tool calling, and agent tool use patterns
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering) — Tool use API mechanics, schema design best practices, and tool result handling

---

## Multi-Agent Patterns

**Status:** ⬜ Not Started

**Definition:** Multi-agent systems decompose complex tasks across specialised agents that communicate through a shared state, message passing, or an orchestrating supervisor agent. Common patterns include the Supervisor (one agent routes subtasks to specialists), Hierarchical (nested orchestrators), and Collaborative (agents debate or verify each other's output).

**Key Mental Model:** Multi-agent is like a project team — the project manager (supervisor) breaks down work, assigns it to specialists (subagents), and integrates the results. No single person does everything.

**How It Works:**
- In the Supervisor pattern, a central orchestrator agent receives the task and determines which specialised subagent to invoke next. The supervisor generates a routing decision ("call the SQL_agent with this query"), the selected subagent executes and returns a result, and the supervisor integrates results and decides the next action. The supervisor's context grows with each subagent result.
- Subagents can run in parallel when their tasks are independent. The orchestrator dispatches multiple subagents simultaneously (e.g., a research agent and a data retrieval agent operating concurrently), waits for all results, then passes the merged output to the next stage. LangGraph implements this via fan-out/fan-in graph nodes.
- Shared state in a multi-agent system is a structured object (a TypedDict or Pydantic model in LangGraph) that all agents can read from and write to. Each agent appends to the state rather than overwriting it, creating an append-only audit trail of the full execution. This state must be carefully designed to avoid agents conflicting on shared fields.
- Inter-agent communication reliability is a production concern. When one agent passes a result to another, the handoff message must be explicit about format and completeness. Vague handoffs ("here are some results") cause the receiving agent to make incorrect assumptions. Structured handoff schemas (same principle as tool schemas) dramatically improve reliability.
- Reflexion and debate patterns use multiple agents to improve output quality: one agent generates an answer, a second agent critiques it, the first agent revises based on the critique. This self-play loop converges on higher-quality outputs for tasks where correctness can be evaluated, but adds latency proportional to the number of debate rounds.

**Common Misconceptions:**
- More agents always solve harder problems — additional agents add coordination overhead, error propagation risk, and cost; use multi-agent only when decomposition genuinely helps.
- Agents can always communicate reliably — agent-to-agent communication must be explicitly designed; agents passing ambiguous or incomplete context to each other is a common failure mode.

**Interview Answer Skeleton:**
- **What it is:** Architectures that route sub-tasks to specialised LLM agents through supervisor orchestration, parallel fan-out, or debate patterns — enabling parallelism and specialisation beyond what a single agent context window can handle.
- **Why it matters / trade-offs:** Multi-agent unlocks tasks that require true parallelism or distinct expert knowledge. The costs are coordination complexity, compounding error rates (a mistake in agent A propagates to agent B), and debugging difficulty. Start with single-agent; add multi-agent when single-agent demonstrably cannot handle the complexity.
- **Example or context:** Automated code review: Supervisor decomposes the PR into file groups, dispatches a security-focused agent and a logic-focused agent in parallel on different file groups, collects their findings, passes them to a synthesis agent that produces a unified review. Structured handoff schemas ensure each sub-agent outputs findings in an identical format the synthesiser can reliably parse.

**Free Resources:**
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph) — Multi-agent graph patterns including supervisor, hierarchical, and parallel agent execution
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Multi-agent orchestration examples with state management and subagent communication patterns

---

## Memory Systems

**Status:** ⬜ Not Started

**Definition:** Agent memory spans four types: in-context (current conversation window), external/episodic (retrieved from a vector store), procedural (instructions and workflows the agent follows), and semantic (factual knowledge about the world or user). Long-running agents need explicit memory management to stay effective beyond a single conversation.

**Key Mental Model:** Agent memory mirrors human memory types — working memory (in-context), long-term episodic memory (vector store), skills and habits (procedural), and general knowledge (semantic). Managing them well is what makes agents feel coherent over time.

**How It Works:**
- In-context memory is simply everything currently in the active conversation window. It is the fastest and highest-fidelity memory, but it is bounded by the context window size and is lost when the session ends. For short tasks, it is the only memory needed.
- External episodic memory uses a vector store to persist past interactions. At session end (or periodically during long sessions), key events and outcomes are embedded and stored. At the start of new sessions, the agent retrieves relevant past interactions by embedding the current task and querying the store — effectively injecting relevant history without filling the whole context window with every past event.
- Memory consolidation reduces storage size and retrieval noise. Instead of storing every raw message, a summarisation step periodically compresses recent history into a shorter episodic summary. The summary is stored and the raw messages are discarded (or archived). This mirrors how human working memory consolidates into long-term memory.
- Procedural memory is encoded in the system prompt — instructions, standard operating procedures, and decision frameworks the agent should follow. This is static across sessions and is the cheapest form of memory (zero retrieval cost). Updating procedural memory means updating the system prompt and redeploying.
- Write-vs-read selectivity is critical. Not everything that happens in a session is worth storing. Memory write logic should filter for high-value events: user corrections, important facts revealed by the user, successful task completions, and confirmed user preferences. Storing everything creates retrieval noise that degrades memory quality over time.

**Common Misconceptions:**
- The context window is enough memory for production agents — context windows fill quickly in long tasks; without external memory, agents lose important context and repeat work.
- All memory should be stored and retrieved — indiscriminate storage creates noise; memory systems need selection logic about what is worth storing and how to retrieve it relevantly.

**Interview Answer Skeleton:**
- **What it is:** The four-layer memory architecture for agents: in-context (working memory), external episodic (vector store retrieval), procedural (system prompt instructions), and semantic (factual knowledge) — each with distinct persistence, retrieval cost, and update mechanics.
- **Why it matters / trade-offs:** Without memory management, agents are stateless and restart from scratch every session. With poorly designed memory, agents retrieve noisy or irrelevant past context that degrades quality. The engineering challenge is selectivity: what to store, when to consolidate, and how to retrieve relevantly.
- **Example or context:** A coding assistant needs to remember user-specific preferences (tabs vs spaces, preferred libraries), past bugs in their codebase, and their code style. Store these as embeddings in a user-scoped vector store. At session start, retrieve the top-5 most relevant memories based on the current task description. Procedural memory (coding standards) lives in the system prompt and applies universally.

**Free Resources:**
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph) — Agent state persistence, memory stores, checkpointing, and cross-session memory management
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Memory architecture examples for long-running and multi-session agent applications

---

## MCP (Model Context Protocol)

**Status:** ⬜ Not Started

**Definition:** MCP (Model Context Protocol) is an open protocol developed by Anthropic that standardises how AI models connect to external data sources, tools, and services. MCP servers expose resources, tools, and prompts through a standard interface; MCP clients (LLM hosts) consume them, enabling a plug-and-play ecosystem for agent capabilities.

**Key Mental Model:** MCP is like USB for AI agents — a standard connector that lets any agent plug into any tool or data source without custom integration code for every combination.

**How It Works:**
- An MCP server is a lightweight process that exposes capabilities through three primitives: tools (callable functions the model can invoke), resources (data sources the model can read, like files or database tables), and prompts (reusable prompt templates with parameters). The server exposes these over a JSON-RPC transport (stdio or HTTP/SSE).
- An MCP client (embedded in the LLM host application — Claude Desktop, an IDE plugin, a custom agent) discovers the server's capabilities at startup via a `list_tools`, `list_resources`, and `list_prompts` handshake. The client then injects the discovered tool definitions into the model's context as available tools.
- When the model calls an MCP tool, the client forwards the tool call to the server over the JSON-RPC transport, waits for the result, and returns it as a tool observation. This is identical to standard function calling from the model's perspective — MCP is purely a standardisation of the application-layer execution and discovery mechanism.
- MCP decouples tool implementation from LLM host implementation. A database MCP server can be written once and used by Claude Desktop, a custom Python agent, and a VS Code extension without any code changes. This is the ecosystem value — community-contributed MCP servers become drop-in capabilities for any MCP-compatible agent.
- Security boundaries matter in MCP deployments. MCP servers with write access to file systems, databases, or external APIs can cause significant damage if the agent misuses them. Access scoping (read-only vs read-write), authentication tokens, and audit logging of all MCP tool invocations are production requirements.

**Common Misconceptions:**
- MCP is only for Anthropic's Claude — MCP is an open protocol; any LLM host that implements the client can use any MCP server, making it provider-agnostic.
- MCP replaces function calling — MCP is a transport and discovery layer; function calling is the mechanism models use to invoke tools; they are complementary.

**Interview Answer Skeleton:**
- **What it is:** An open JSON-RPC protocol that standardises how agent hosts discover and invoke external tools, data sources, and prompt templates through a server/client architecture — enabling plug-and-play tool ecosystems across different LLM platforms.
- **Why it matters / trade-offs:** MCP reduces integration work for tool-rich agent systems. Instead of writing custom function-calling wrappers for every tool/model combination, you write one MCP server per tool and one MCP client per agent host. The trade-offs are added network latency (for HTTP transport) and the need to manage server process lifecycle.
- **Example or context:** Expose a PostgreSQL database as an MCP server with three tools: `execute_read_query`, `list_tables`, and `describe_table`. The server enforces read-only access at the SQL level. Any MCP-compatible agent can now query the database with zero additional integration code — the client discovers the tools automatically and the model uses them through standard function calling.

**Free Resources:**
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph) — MCP integration patterns and tool ecosystem composition in agent systems
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — MCP server and client implementation examples with security and access control patterns

---

## Computer Use

**Status:** ⬜ Not Started

**Definition:** Computer use agents can interact with a computer's graphical interface — clicking, typing, scrolling, taking screenshots — rather than just calling APIs. This enables automation of tasks in applications that have no API, making agents capable of operating any software a human can.

**Key Mental Model:** Computer use is giving the agent eyes and hands in front of a screen. Instead of calling an API, it looks at a screenshot, clicks the button it sees, and reads the result — like a remote-control human operator.

**How It Works:**
- Computer use is implemented through a set of low-level action tools: `screenshot` (capture the current screen state), `click` (x, y coordinates), `type` (keyboard input), `scroll` (mouse wheel), and `key` (special key presses like Enter or Escape). The model receives a screenshot as a visual observation and decides which action to take next.
- The vision encoder processes the screenshot as image tokens and maps UI elements (buttons, text fields, menus) to spatial coordinates. The model generates action tool calls with specific pixel coordinates — e.g., `click(x=342, y=158)`. This requires the vision encoder to accurately map semantic UI elements to screen positions.
- The execution loop is: take screenshot → model observes and reasons → model outputs an action → application executes the action → take new screenshot → repeat. Each iteration is slow (1–5 seconds per step) because of screenshot capture, image tokenisation, and model inference. Complex browser tasks may require 20–50 iterations.
- Computer use agents need sandboxed environments to be safe. Running a computer use agent against a real production machine with access to email, file system, and browser history is extremely high-risk. Best practice is to run computer use agents in ephemeral VMs or containerised desktop environments with network access tightly scoped to required services only.
- Reliability degrades with UI complexity. Desktop GUIs with consistent layouts are more reliable than web pages that change structure dynamically. Agents should fall back to API calls whenever an API equivalent exists — computer use is a last resort for automation, not a first choice.

**Common Misconceptions:**
- Computer use is a reliable production-ready capability for all workflows — computer use is powerful but slower, more expensive, and less reliable than API-based tool use; use it only when no API alternative exists.
- Computer use agents are fully autonomous and need no guardrails — agents interacting with UIs can accidentally trigger destructive actions; human-in-the-loop checkpoints are essential for high-stakes workflows.

**Interview Answer Skeleton:**
- **What it is:** An agent capability that controls a computer through screenshot observation and low-level UI actions (click, type, scroll), enabling automation of any graphical application — not just those with APIs.
- **Why it matters / trade-offs:** Computer use expands agent scope to legacy systems, third-party SaaS tools without APIs, and any task requiring GUI interaction. The costs are high latency (seconds per step), reliability issues with dynamic UIs, and significant security risk requiring sandbox isolation.
- **Example or context:** Automating data entry into a legacy ERP system with no API is a valid computer use case — API alternatives do not exist. For a workflow where Salesforce does have an API, use the API instead. When deploying computer use, run in an ephemeral VM, scope network access to only the target application's domain, and add a human confirmation step before any data submission action.

**Free Resources:**
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph) — Agent capability patterns including computer use integration and sandbox orchestration
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Computer use implementation examples with sandboxing, safety boundaries, and human-in-the-loop patterns

---

## Coding Agents

**Status:** ⬜ Not Started

**Definition:** Coding agents are LLM-based systems that can write, execute, test, and iterate on code autonomously. They typically have access to a code interpreter, file system, terminal, and often a web search tool. They follow a generate-execute-observe-fix loop until the task is complete.

**Key Mental Model:** A coding agent is a junior developer who has their own computer — they write code, run it, see what breaks, read the error, fix it, and repeat until the tests pass, without needing to ask for help on every step.

**How It Works:**
- The coding agent loop begins with a task specification. The agent generates a plan (if using ReAct or extended thinking), writes code to a file using a file-write tool, then executes the file in a sandboxed interpreter via an `execute_code` tool. The interpreter returns stdout, stderr, and exit code as the observation.
- Error recovery is a key capability. When execution fails, the agent reads the error output, reasons about the cause (often a missing import, a type error, or an off-by-one), edits the file, and re-executes. This generate-observe-fix loop is what separates coding agents from one-shot code generation.
- Test-driven iteration gives coding agents a clear termination condition. The task is considered complete when all provided unit tests pass. The agent knows whether it succeeded because the test runner exit code is 0 — a verifiable ground truth that prevents the agent from hallucinating success.
- Sandbox isolation is non-negotiable. Code execution must happen in a container or VM with no access to production data, credentials, or network resources beyond what is explicitly needed. Agents that can write and execute arbitrary code can exfiltrate data, install malware, or consume unbounded compute. E2B, Docker containers, and Pyodide (browser WASM) are common sandbox implementations.
- Context management in long coding tasks requires careful engineering. A large codebase cannot fit in a single context window. Agents must use file-reading tools to load only relevant files, maintain a mental map of the project structure, and avoid loading the entire codebase at once. See [[AI-Engineer/03-RAG-Systems]] for retrieval patterns applicable to code search.

**Common Misconceptions:**
- Coding agents can replace human developers for complex software engineering tasks — current coding agents excel at well-scoped tasks with clear acceptance criteria; open-ended system design and architecture still require human judgement.
- Generated code that executes successfully is correct code — code can run without errors but still produce wrong outputs, have security vulnerabilities, or fail on edge cases; review and testing remain essential.

**Interview Answer Skeleton:**
- **What it is:** LLM agents that autonomously generate, execute, debug, and iterate on code using a sandboxed interpreter and file system tools, following a generate-observe-fix loop until tests pass or a step limit is reached.
- **Why it matters / trade-offs:** Coding agents dramatically accelerate well-scoped scripting and data analysis tasks. The engineering constraints are sandbox security, context window management for large codebases, and defining clear termination conditions (passing tests) rather than leaving agents to judge their own success.
- **Example or context:** A data analysis coding agent receives a CSV path and a task description. It uses `read_file` to inspect the CSV schema, writes a pandas analysis script, executes it, reads the error when a column name is wrong, corrects the script, re-executes, and produces a final chart — all in a Docker container with network access disabled. The entire loop is observable via a trace in [[AI-Engineer/07-Observability-Evals]].

**Free Resources:**
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph) — Coding agent patterns with code execution tools, sandboxing setup, and ReAct loop implementation
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Coding agent examples including test-driven iteration, error recovery, and sandbox integration
