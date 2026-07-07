# Structured Outputs vs Tool Calling: When Your Agent Actually Needs Which

> Artigo #17 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/structured-outputs-vs-tool-calling-when-your-agent-actually-needs-which-kgk)

---

If you're building an AI agent, you've probably asked this: should I use structured outputs or tool calling to get data from the model?

The answer matters more than most guides suggest. The wrong choice means brittle parsing, wasted tokens on unnecessary steps, or an agent that confidently hallucinates a JSON schema it never actually followed.

I've run both patterns in production across dozens of agent workflows. This is the decision framework I use.

The Two Patterns
Structured Outputs

You tell the model: "Respond in this exact format." You supply a schema (Pydantic, JSON Schema, or a system prompt describing fields), and the LLM produces a single response that matches.


```python
from openai import OpenAI
from pydantic import BaseModel

class WeatherResponse(BaseModel):
city: str
temperature_c: float
condition: str
wind_kmh: float
summary: str

client = OpenAI()
response = client.beta.chat.completions.parse(
model="gpt-4o-2026-04-20",
messages=[{"role": "user", "content": "What's the weather in Tokyo?"}],
response_format=WeatherResponse,
)
result: WeatherResponse = response.choices[0].message.parsed
print(result.temperature_c)

```

The model returns one JSON object that satisfies the schema. No intermediate steps, no decisions.

Tool Calling (Function Calling)

You register callable functions with the model. The model decides which tool to call, with what arguments, and returns a tool call request instead of a text answer.


```python
from openai import OpenAI

client = OpenAI()
response = client.chat.completions.create(
model="gpt-4o-2026-04-20",
messages=[{"role": "user", "content": "What's the weather in Tokyo?"}],
tools=[{
"type": "function",
"function": {
"name": "get_weather",
"description": "Get current weather for a city. Returns temperature, condition, and wind speed.",
"parameters": {
"type": "object",
"properties": {
"city": {"type": "string", "description": "City name"},
"units": {"type": "string", "enum": ["metric", "imperial"]},
},
"required": ["city"],
},
},
}],
)

tool_call = response.choices[0].message.tool_calls[0]
print(tool_call.function.arguments)
# '{"city": "Tokyo", "units": "metric"}'

```

The model returns a decision to call a function. You execute it, feed results back, and the model produces a final answer.

The Core Difference

Structured outputs say: "Give me data in this shape, now."

Tool calling says: "Here are actions you can take. Decide which ones to use."

That distinction drives every architectural decision that follows.

When to Use Structured Output
1. Single-Step Extraction

Your input is a blob of text. Your output is a structured extraction. No intermediate actions needed.


```python
class IssueExtraction(BaseModel):
severity: str  # LOW, MEDIUM, HIGH, CRITICAL
affected_service: str
root_cause_summary: str
suggested_fix: str
estimated_hours: int

response = client.beta.chat.completions.parse(
model="gpt-4o-2026-04-20",
messages=[
{"role": "system", "content": "Extract structured incident data from the following report."},
{"role": "user", "content": open('incident-report.txt').read()},
],
response_format=IssueExtraction,
)

```

The model reads the report and produces a structured IssueExtraction. One API call. No function execution. Use this when the transformation is read-only — the model analyzes and extracts, it doesn't act.

Signal: input → extraction → done.

2. Validation-Heavy Pipelines

OpenAI's response_format with structured outputs does schema enforcement at the API level. If the model produces invalid JSON, the API retries internally. You get a valid Pydantic object or an error — never malformed output you need to parse defensively.

Anthropic's tool_use blocks offer similar enforcement, Anthropic's beta.tools with response models enforce schema. But for pure extraction, structured output APIs have tighter integration.


```python
# This always returns a valid IssueExtraction or raises an API error
# You don't need try/except around json.loads()
extraction = response.choices[0].message.parsed
assert isinstance(extraction, IssueExtraction)  # Always true
```

3. Parallel Extraction Across Multiple Inputs

You want to extract the same schema from 500 documents. Structured output means you send 500 independent API calls, no state sharing between them.

Tool calling would require you to track which tool was called for which document, feed the results back into conversation, manage a growing context window. For batch extraction, that's unnecessary complexity.

4. Cost-Sensitive Workflows

Structured output is one API call. Tool calling is at minimum two (model decides to call tool, you execute, model receives result and generates final answer). For high-throughput extraction pipelines, the 2x call difference matters.

When to Use Tool Calling
1. Multi-Step Workflows

The user asks: "Find the most expensive error in Sentry this week, check what deployment triggered it, and create a GitHub issue."

A structured output can't do this. The model needs to:

```python
Query Sentry (tool call)
Read the result (context in next message)
Query deployment logs (tool call)
Cross-reference the data
Create a GitHub issue (tool call)
Summarize the finding (final text response)
```

This is the agent loop: decide → call → observe → decide again.


```python
tools = [
{"type": "function", "function": {
"name": "query_sentry",
"description": "Query Sentry for issues. Returns issue details including count and stack trace.",
"parameters": {"type": "object", "properties": {
"filter": {"type": "string"},
"sort_by": {"type": "string", "enum": ["count", "date"]},
"limit": {"type": "integer"},
}},
}},
{"type": "function", "function": {
"name": "query_deployments",
"description": "Query recent deployments. Returns deployment hashes and timestamps.",
"parameters": {"type": "object", "properties": {
"environment": {"type": "string"},
"since": {"type": "string"},
}},
}},
{"type": "function", "function": {
"name": "create_github_issue",
"description": "Create a GitHub issue with title, body, and labels.",
"parameters": {"type": "object", "properties": {
"title": {"type": "string"},
"body": {"type": "string"},
"labels": {"type": "array", "items": {"type": "string"}},
}},
}},
]

# The agent loop
messages = [{"role": "user", "content": "Find the most expensive error in Sentry..."}]

for _ in range(5):  # Max iterations to prevent infinite loops
response = client.chat.completions.create(
model="gpt-4o-2026-04-20",
messages=messages,
tools=tools,
)

if response.choices[0].message.tool_calls:
tool_call = response.choices[0].message.tool_calls[0]
result = execute_tool(tool_call)
messages.append(response.choices[0].message)
messages.append({"role": "tool", "content": result, "tool_call_id": tool_call.id})
else:
# Final text response
print(response.choices[0].message.content)
break

```

The model orchestrates three different systems. Structured outputs can express the final result but can't get you there.

2. Conditional Execution

The agent needs to decide if it should call a tool at all. Structured outputs always produce output. Tool calling allows the model to say "I don't need any tools for this."


# User asks: "What's 2 + 2?"
# The model should NOT call query_sentry for this.
# With tool calling, the model responds with text directly.
# With structured output, the model is forced to fill the schema — even if the question doesn't warrant it.


This matters a lot in production. Users ask things that don't match your tools, and an agent that always hallucinates tool calls (because the schema forces it) is worse than one that sometimes says "I can't help with that."

3. Parallel Tool Calls

Modern models support calling multiple tools in a single response. If the model needs to check the weather in three cities simultaneously:


# Model returns:
{
```python
"tool_calls": [
{"function": {"name": "get_weather", "arguments": '{"city": "Tokyo"}'}, "id": "call_1"},
{"function": {"name": "get_weather", "arguments": '{"city": "London"}'}, "id": "call_2"},
{"function": {"name": "get_weather", "arguments": '{"city": "NYC"}'}, "id": "call_3"},
]
}
```

# You execute all three in parallel, then feed results back


This is the performance killer for agent systems. Instead of five sequential API calls, you get one call that triggers three parallel lookups. Structured output can't express "call this function three times with different arguments."

4. MCP Server Exposure

If you're building an MCP server, tool calling is the protocol. MCP is, at its core, a standardization of function calling over a transport layer (stdio or Streamable HTTP). The MCP spec defines tools, resources, and prompts — tools being the function-calling primitives.

When your agent connects to an MCP server, it's using tool calling under the hood. The MCP server advertises its capabilities, the agent picks which tools to invoke, and sends JSON-RPC requests.


```python
# MCP client side — this is structured tool calling over a transport
from mcp import ClientSession

async with client_session(server) as session:
tools = await session.list_tools()  # Discover available tools
result = await session.call_tool("get_weather", {"city": "Tokyo"})

```

If your agent needs to connect to external systems via MCP, tool calling isn't optional — it's the interface.

Decision Matrix
Scenario	Best Pattern	Why
Extract entities from text	Structured output	Single call, schema-enforced
Answering questions that need external data	Tool calling	Model decides which tool to call
Multi-step reasoning (check A, then decide B)	Tool calling	Agent loop enables conditional chaining
Batch extraction across 1000 documents	Structured output	Independent, parallelizable, cheaper
Connect to systems via MCP	Tool calling	MCP = function calling protocol
Validate and return a single response format	Structured output	API-level schema enforcement
Handle ambiguous requests gracefully	Tool calling	Model can skip tools and answer directly
Parallel API calls in one round	Tool calling	Multiple tool calls per response
The Hybrid Pattern That Works Best in Practice

Most real agent workflows use both. The pattern looks like this:

Tool calling for the orchestration layer — the model decides which systems to query and in what order.
Structured output for the final result — the parsed, validated data you persist or act on.
```python
# Layer 1: Tool calling to gather data
tools = [query_database, search_docs, check_status]
# ... agent loop ...

# Layer 2: Structured output for the final synthesis
analysis = client.beta.chat.completions.parse(
model="gpt-4o-2026-04-20",
messages=[system_msg] + conversation_history,
response_format=IncidentReport,
)

# analysis is a validated IncidentReport object, not raw text
save_to_database(analysis.model_dump())

```

This gives you flexible orchestration (the model picks the right tools in the right order) and reliable output (the final response is schema-validated, ready for downstream processing).

Common Anti-Patterns
Tool Calling for Simple Extraction

Building a five-tool pipeline when you really just need to extract a JSON object from text. Every extra tool call adds latency, cost, and an opportunity for the model to pick the wrong tool. If the task is read-only, use structured output.

Structured Output for Agent Behavior

Forcing a model into a fixed schema when the task requires decisions. "Extract a ticket" where one field is next_action: ToolChoice — and then you execute based on that — is a fragile workaround. Proper tool calling gives the model the agency to make those decisions natively.

Missing the Max-Iteration Guard

Every tool-calling loop must have a hard cap. Models can enter reflection loops where they call the same tool three times with slightly different arguments, convinced the first result wasn't right.


for iteration in range(5):  # ALWAYS enforce a max
```python
response = call_model(messages, tools)
if not response.tool_calls:
break
messages += execute_and_append(response.tool_calls)
else:
messages.append({"role": "system", "content": "Max iterations reached. Summarize what you found so far."})

No Fallback on Tool Failure
```

When a tool call raises an exception, feed the error back to the model as a tool response. The model can retry, pick a different tool, or gracefully degrade:


messages.append({
```python
"role": "tool",
"content": f"Error: Database connection timeout. Retry or try a different query.",
"tool_call_id": call_id,
})

```

Silent tool failures mean the agent loop stalls. Explicit error messages mean the agent adapts.

Where Platforms Like Nebula Fit In

The structured output vs tool calling question gets significantly harder when your agent needs both patterns, state management, parallel tool execution, and guardrails. Running the full agent loop — with max iterations, error handling, tool budget enforcement, and structured output parsing — is non-trivial infrastructure.

Platforms like Nebula abstract this: you define the agent's tools (which become MCP-exposed), set constraints (max calls, budgets), and the platform manages the orchestration loop, structured output parsing, and state persistence across tool calls.

The architecture looks like this:


User Message
│
▼
┌─────────────────────────┐
│   Orchestrator (Nebula)  │
│  - Max iteration guard   │
│  - Tool budget tracking  │
│  - Context compression   │
│  - Structured parsing    │
└─────────────────────────┘
│
```python
├──→ Tool: database query (MCP)
├──→ Tool: web search (MCP)
├──→ Tool: GitHub API (MCP)
│
▼
┌─────────────────────────┐
│  Structured Output Layer │
│  - Validate final result  │
│  - Persist to store       │
│  - Deliver to user        │
└─────────────────────────┘

```

If you're building a single agent with a handful of tools, the raw chat.completions API works fine. If your agents span multiple MCP servers, need tool budget enforcement, or require structured output validation at scale, a platform that handles the orchestration loop is worth the overhead.

Takeaways

Structured output for extraction, tool calling for action. If the model needs to read and return data — use structured output. If it needs to decide, act, observe, and decide again — use tool calling.

Hybrid is the production pattern. Tool calling for orchestration, structured output for the final validated result. Don't force one to do both jobs.

Always cap iterations. A tool-calling loop without a max iteration count is a production incident waiting to happen.

Feed errors back to the model. Failed tool calls should return structured error messages, not silent exceptions. The model can retry or adapt if it knows what went wrong.

MCP = standardized tool calling. If your agent connects to external systems, MCP is the protocol to expose your functions. Every MCP server is a tool-calling registry.

The choice between structured outputs and tool calling isn't just an API preference — it's a design decision about how much agency your agent actually needs.

This article is part of the Building Production AI Agents series on Dev.to, covering the real engineering challenges of running autonomous AI agents — from MCP server architecture to agent guardrails.

The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
Structured Outputs vs Tool Calling: When Your Agent Actually Needs Which
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools