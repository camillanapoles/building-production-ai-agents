# Building an AI Agent Harness from Scratch: The Architecture Between LLM and Agent

> Artigo #21 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/building-an-ai-agent-harness-from-scratch-the-architecture-between-llm-and-agent-3a1o)

---

Everyone talks about the model. Nobody talks about the harness.

Give Claude Sonnet or GPT-4o a chat interface and you get a conversational AI. Wrap it in a loop that can call external tools, maintain state across turns, enforce budget limits, and validate its own outputs — and you get an agent. The difference isn't the LLM. It's everything around the LLM.

The AWS team published a guide on \"agent harnesses\" this week, and it got me thinking: most tutorials show you how to call an LLM or how to register a tool. Almost none show you the orchestration layer that makes those individual pieces behave as a coherent system.

I've built agents that run autonomously on production infrastructure 24/7. The mistakes I made early on weren't about picking the wrong model. They were about skipping the harness — assuming the model would \"just figure it out.\" It won't. The harness is what makes an agent reliable, and reliability is the only metric that matters once you move past the demo phase.

Here's how to build one from scratch.

What Is an Agent Harness, Really?

An agent harness is the execution environment that sits between the user and the LLM. It's not the prompt. It's not the model. It's the infrastructure that:

Manages the conversation loop — receiving input, calling the model, routing tool calls, feeding results back, repeating until termination
Registers and dispatches tools — maintaining a catalog of callable functions, validating arguments, executing them safely, and returning structured results
Maintains memory — storing conversation history, injecting relevant context, compressing old messages to stay within context limits
Enforces guardrails — limiting token budgets, capping tool call counts, preventing infinite loops, blocking dangerous actions
Handles failures — retrying on transient errors, degrading gracefully when a tool is unavailable, escalating to human review when confidence is low

Without a harness, you have a stateless API call. With a harness, you have a system.

The Minimal Agent Harness

Let's start with the smallest useful version. A harness needs three things: a model interface, a tool registry, and a loop.


import json
from typing import Callable, Any
from dataclasses import dataclass, field

@dataclass
class Tool:
name: str
description: str
parameters: dict  # JSON Schema
fn: Callable

class AgentHarness:
def __init__(self, model, system_prompt: str = \"\"):
self.model = model
self.system_prompt = system_prompt
self.tools: dict[str, Tool] = {}
self.max_iterations = 10

def register_tool(self, tool: Tool):
self.tools[tool.name] = tool

def tool_list(self) -> list[dict]:
return [
{\"type\": \"function\", \"function\": {
\"name\": t.name, \"description\": t.description,
\"parameters\": t.parameters,
}}
for t in self.tools.values()
]

def run(self, user_input: str) -> str:
messages = [
{\"role\": \"system\", \"content\": self.system_prompt},
{\"role\": \"user\", \"content\": user_input},
]
for i in range(self.max_iterations):
response = self.model.chat(
messages=messages, tools=self.tool_list() if self.tools else None,
)
if not response.tool_calls:
return response.content
messages.append(response.message)
for call in response.tool_calls:
tool = self.tools.get(call.function.name)
if not tool:
result = f\"Error: Unknown tool '{call.function.name}'\"
else:
try:
args = json.loads(call.function.arguments)
result = tool.fn(**args)
except Exception as e:
result = f\"Error: {type(e).__name__}: {e}\"
messages.append({\"role\": \"tool\", \"content\": str(result), \"tool_call_id\": call.id})
return \"Max iterations reached.\"


That's the skeleton. It loops: call model, check for tool calls, execute, feed back. Seven lines of core logic. It works for demos. It breaks in production. Let's see why.

Problem 1: The Tool Registry Lies

You register a tool, the agent calls it, and it crashes because input validation is wrong. The tool description promised certain parameters, the model complied, but the underlying function has tighter requirements. This isn't the model's fault — it's a harness problem: the tool registry should validate before dispatch.


class ToolRegistry:
def __init__(self):
self.tools: dict[str, Tool] = {}
self.call_counts: dict[str, int] = {}

def register(self, tool: Tool):
self.tools[tool.name] = tool
self.call_counts[tool.name] = 0

def validate_call(self, tool_name: str, arguments: dict) -> tuple[bool, str]:
if tool_name not in self.tools:
return False, f\"Unknown tool: {tool_name}\"
schema = self.tools[tool_name].parameters
for field in schema.get(\"required\", []):
if field not in arguments:
return False, f\"Missing required parameter: {field}\"
for arg_name, arg_value in arguments.items():
if arg_name not in schema.get(\"properties\", {}):
return False, f\"Unexpected parameter: {arg_name}\"
return True, \"OK\"

def execute(self, tool_name: str, arguments: dict) -> Any:
self.call_counts[tool_name] += 1
return self.tools[tool_name].fn(**arguments)


The registry acts as a gatekeeper, not just a dispatcher. Before any tool fires, the harness validates existence, required fields, type correctness, and hallucinated parameters. This catches 60-70% of tool-call errors before they reach application code.

Problem 2: Memory Bloat Kills Context

Ten turns in, the conversation contains the original prompt, four tool call/response pairs, and a partial draft. The context window is filling up. By turn 20, the model starts forgetting the system prompt. The solution is intelligent context management: compress what you don't need, preserve what you do.


import tiktoken
from dataclasses import dataclass

@dataclass
class MemoryConfig:
max_context_tokens: int = 64_000
keep_recent_messages: int = 8
always_preserve_system: bool = True

class AgentMemory:
def __init__(self, config: MemoryConfig):
self.config = config
self.messages: list[dict] = []
self.encoder = tiktoken.encoding_for_model(\"gpt-4o\")

def add(self, role: str, content: str, **kwargs):
self.messages.append({\"role\": role, \"content\": content, **kwargs})

def get_messages(self) -> list[dict]:
total = sum(len(self.encoder.encode(m.get(\"content\", \"\"))) + 4 for m in self.messages)
if total \u003C= self.config.max_context_tokens:
return self.messages
return self._compress()

def _compress(self) -> list[dict]:
keep = self.config.keep_recent_messages
system_msg = None
if self.config.always_preserve_system:
system_msgs = [m for m in self.messages if m[\"role\"] == \"system\"]
if system_msgs:
system_msg = system_msgs[0]
recent = self.messages[-keep:]
old = self.messages[:-keep]
if not old:
return [system_msg] + recent if system_msg else recent
# Summarize old messages (in production, call a cheap model like Haiku)
old_text = \"\
\".join(f\"[{m['role']}]: {m.get('content', '')[:200]}\" for m in old)
summary = \" | \".join([line[:100] for line in old_text.split(\"\
\") if any(kw in line.lower() for kw in [\"tool:\", \"result:\", \"error:\"])][:10])
compressed = [{\"role\": \"system\", \"content\": f\"[EARLIER CONTEXT: {summary}]\"}]
if system_msg:
compressed = [system_msg] + compressed
compressed.extend(recent)
return compressed


Treat the context window like OS memory: recent messages are your hot cache, old messages are swap space, and the system prompt is kernel memory — never page it out.

Problem 3: The Loop Runs Forever

The model enters a reasoning spiral. It calls search_database, gets a result, calls it again with slightly different parameters, repeats indefinitely. Tokens pile up. Budget enforcement is the most critical guardrail, and it belongs in the harness, not the prompt.


from dataclasses import dataclass
import time

@dataclass
class BudgetConfig:
max_tokens: int = 30_000
max_tool_calls: int = 25
max_time_seconds: float = 300.0
max_per_tool_calls: int = 5

class BudgetEnforcer:
def __init__(self, config: BudgetConfig):
self.config = config
self.tokens_used = 0
self.tool_calls_total = 0
self.tool_calls_per_tool: dict[str, int] = {}
self.start_time = time.time()

def record_tokens(self, input_tokens: int, output_tokens: int):
self.tokens_used += input_tokens + output_tokens

def record_tool_call(self, tool_name: str):
self.tool_calls_total += 1
self.tool_calls_per_tool[tool_name] = self.tool_calls_per_tool.get(tool_name, 0) + 1

def check(self) -> str | None:
if self.tokens_used >= self.config.max_tokens:
return f\"Token budget exceeded: {self.tokens_used} (limit {self.config.max_tokens})\"
if self.tool_calls_total >= self.config.max_tool_calls:
return f\"Tool call budget exceeded: {self.tool_calls_total}\"
if time.time() - self.start_time >= self.config.max_time_seconds:
return \"Time budget exceeded\"
for tool, count in self.tool_calls_per_tool.items():
if count >= self.config.max_per_tool_calls:
return f\"Per-tool limit: '{tool}' called {count} times\"
return None


Four budgets, any of which stops the agent before costs spiral: token budget, tool call budget, time budget, and per-tool budget.

Problem 4: Errors Swallowed, Not Handled

A tool call raises ConnectionError. The harness catches it, returns \"Error: ConnectionError\", and the model gets confused. It doesn't know if it should retry, try a different tool, or give up. Error formatting is an agent design problem. The model needs structured error messages that tell it what went wrong and what to do.


from enum import Enum
from dataclasses import dataclass

class ErrorType(Enum):
TRANSIENT = \"transient\"
PERMANENT = \"permanent\"
UNAVAILABLE = \"unavailable\"

@dataclass
class ToolError:
error_type: ErrorType
message: str
suggestion: str

def format_tool_error(error: ToolError) -> str:
parts = [f\"[TOOL ERROR: {error.error_type.value.upper()}]\"]
parts.append(error.message)
if error.suggestion:
parts.append(f\"Suggested action: {error.suggestion}\")
return \"\
\".join(parts)


Examples:

Transient: Rate limit hit → \"Retry with different parameters or try an alternative tool.\"
Permanent: DELETE query rejected → \"Use SELECT queries to read data instead.\"
Unavailable: Weather service down → \"Inform the user data is unavailable.\"

A bare exception traceback tells the model nothing. A structured error with a suggested action gives it a decision tree.

Problem 5: The Harness Has No State

The minimal harness is stateless between runs. For cross-session persistence, you need a state layer:


import json
import sqlite3
from datetime import datetime, UTC

class AgentState:
def __init__(self, db_path: str = \"agent_state.db\"):
self.db = sqlite3.connect(db_path)
self.db.execute(\"\"\"CREATE TABLE IF NOT EXISTS sessions (
session_id TEXT PRIMARY KEY, created_at TEXT,
last_active TEXT, user_id TEXT)\"\"\")
self.db.execute(\"\"\"CREATE TABLE IF NOT EXISTS tool_invocations (
id INTEGER PRIMARY KEY AUTOINCREMENT,
session_id TEXT, turn_number INTEGER,
tool_name TEXT, arguments TEXT, result TEXT,
success INTEGER, duration_ms INTEGER, timestamp TEXT)\"\"\")
self.db.commit()

def create_session(self, session_id: str, user_id: str):
self.db.execute(
\"INSERT INTO sessions VALUES (?, ?, ?, ?)\",
(session_id, datetime.now(UTC).isoformat(), datetime.now(UTC).isoformat(), user_id))
self.db.commit()

def record_tool_invocation(self, session_id: str, turn: int,
tool: str, args: dict, result: str,
success: bool, duration_ms: int):
self.db.execute(
\"INSERT INTO tool_invocations VALUES (NULL, ?, ?, ?, ?, ?, ?, ?, ?)\",
(session_id, turn, tool, json.dumps(args), result,
int(success), duration_ms, datetime.now(UTC).isoformat()))
self.db.commit()

def get_analytics(self, session_id: str) -> dict:
total = self.db.execute(\"SELECT COUNT(*) FROM tool_invocations WHERE session_id = ?\", (session_id,)).fetchone()[0]
rate = self.db.execute(\"SELECT AVG(success) FROM tool_invocations WHERE session_id = ?\", (session_id,)).fetchone()[0] or 0
return {\"total_invocations\": total, \"success_rate\": round(rate * 100, 1)}


The state layer gives you session persistence, tool invocation audit logs, and built-in analytics — essential for debugging failed sessions.

The Complete Architecture

All five pieces fit together:


User Input
▼
┌───────────────────────────────┐
│         Budget Enforcer        │  ← Checks before every iteration
├───────────────────────────────┤
│         Agent Memory           │  ← Compresses old context
├───────────────────────────────┤
│         LLM Call               │  ← With tool definitions
├─────────────────┬─────────────┤
│   tool calls?   │   no → return
├─────────────────┤
│  Tool Registry   │  ← Schema + type validation
├───────────────────────────────┤
│  Safe Execute    │  ← Structured errors with suggestions
├───────────────────────────────┤
│  Agent State     │  ← Log turn + tool invocation
└───────────────────────────────┘
loop back


Each component has a single responsibility. The harness coordinates them. The model is just one node in the graph.

Where Managed Platforms Fit In

Building this harness from scratch teaches you exactly what's involved. But the five components — tool registry, memory management, budget enforcement, error handling, and state persistence — are infrastructure, not business logic. They're identical whether you're building a GitHub agent, a content agent, or a customer support agent.

Platforms like Nebula abstract exactly this layer. You define the tools (automatically MCP-exposed), the system prompt, and constraints like max iterations and token budgets. The platform handles the harness: tool validation, context compression, budget tracking, error formatting, and session persistence. Every agent execution is traced end-to-end with cost attribution, and the observability dashboard shows tool call distributions, success rates, and budget consumption in real time.

You focus on what the agent does. The platform ensures you can see when it goes wrong.

Actionable Takeaways

Start with the loop, not the model. The call-observe-decide-repeat pattern is fundamental. Pick any capable LLM and focus on getting the harness right.

Validate tool calls before dispatch. Schema validation catches 60-70% of errors before they hit application code.

Compress context aggressively. Use a hot-cache pattern: keep recent messages, summarize old ones, preserve the system prompt.

Enforce budgets in code, not prompts. A max_iterations field in your prompt is a suggestion. A BudgetEnforcer that halts execution is a guarantee.

Structure your errors. Classify errors as transient (retry), permanent (redirect), or unavailable (graceful degradation), always with a suggested action.

Log everything. Tool invocations with arguments, results, durations, and success status. When a session goes wrong, logs are the only way to reconstruct what happened.

Build the harness first, optimize the model second. A well-harnessed GPT-3.5 outperforms an unharnessed GPT-4o every time.

The agent harness isn't glamorous. But it's the difference between an agent that works once in a notebook and one that works at 2 AM on a Tuesday when nobody's watching. Build it right, and the model becomes the least interesting part of your system.

This article is part of the Building Production AI Agents series on Dev.to.

Building Production AI Agents (26 Part Series)
The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
Building an AI Agent Harness from Scratch: The Architecture Between LLM and Agent
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools
DEV Community

Build Apps with Google AI Studio 🧱

This track will guide you through Google AI Studio's new \"Build apps with Gemini\" feature, where you can turn a simple text prompt into a fully functional, deployed web application in minutes.

Read more →

Read More