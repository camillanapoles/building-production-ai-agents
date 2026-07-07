# How to Build Deferred Tool Loading for AI Agents in 15 Minutes

> Artigo #22 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/how-to-build-deferred-tool-loading-for-ai-agents-in-15-minutes-4p6m)

---

Your agent has 40 tools. Each tool definition — name, description, JSON Schema parameters — costs roughly 200 tokens. That's 8,000 tokens before the agent does a single thing. Add a few MCP servers and you're burning 55,000 tokens just on tool definitions per request.

The industry term is "token bloat." The fix is deferred tool loading: start with a tiny search tool, load specific tools only when the agent needs them, and unload them when done.

This tutorial shows you how. One file, runnable code, no framework dependencies.

The Problem
```python
# What most tutorials do:
agent = Agent(tools=[tool_1, tool_2, tool_3, ..., tool_40])
# Every LLM call ships ALL 40 tool definitions in the prompt.
# Cost: ~8,000 tokens per call just for tool schemas.

```

When you're running autonomous agents 24/7, that overhead compounds fast. An agent making 100 calls/day burns an extra 800,000 tokens daily just describing tools it never uses.

The Solution: Search-Then-Load

Instead of loading every tool upfront, give the agent exactly one tool: tool search. When the agent needs a capability, it searches for the right tool, the system loads it, and the agent re-runs with the newly available tool.


User: "Find the latest PR on nebula-web and update the changelog"

Turn 1: Agent has only `search_tools(query)`. It searches.
Turn 2: System loads `get_pr()` + `update_changelog()`.
Agent now has 2 tools. Calls them.
Turn 3: Agent responds with result.


Three turns instead of one, but the token savings are massive: ~400 tokens for the search tool definition vs ~8,000 for all 40 tools.

Step 1: Build the Tool Registry

First, a registry that stores all available tools but only exposes a subset at any time.


```python
from dataclasses import dataclass, field
from typing import Callable, Any

@dataclass
class Tool:
name: str
description: str
parameters: dict  # JSON Schema
fn: Callable
category: str = "general"  # For search filtering

class DeferredToolRegistry:
def __init__(self):
self.all_tools: dict[str, Tool] = {}
self.active_tools: set[str] = set()

def register(self, tool: Tool):
self.all_tools[tool.name] = tool

def search(self, query: str) -> list[dict]:
"""Search all tools by name, description, and category."""
query_lower = query.lower()
results = []
for name, tool in self.all_tools.items():
score = 0
if query_lower in name.lower(): score += 3
if query_lower in tool.description.lower(): score += 2
if query_lower in tool.category.lower(): score += 1
if score > 0:
results.append({"name": name, "description": tool.description, "score": score})
results.sort(key=lambda r: r["score"], reverse=True)
return results[:5]  # Return top 5 matches

def load(self, tool_names: list[str]) -> list[dict]:
"""Activate specific tools. Returns their definitions for the LLM."""
for name in tool_names:
if name in self.all_tools:
self.active_tools.add(name)
return [
{"type": "function", "function": {
"name": t.name, "description": t.description, "parameters": t.parameters,
}}
for t in self.all_tools.values() if t.name in self.active_tools
]

def execute(self, tool_name: str, arguments: dict) -> Any:
if tool_name not in self.active_tools:
raise ValueError(f"Tool '{tool_name}' is not loaded. Load it first.")
tool = self.all_tools[tool_name]
return tool.fn(**arguments)

def unload_all(self):
self.active_tools.clear()

```

The registry separates storage (all tools, always available for search) from activation (tools the LLM can actually call). The search function scores matches by name, description, and category — no embeddings needed.

Step 2: Create the Search Tool

The search tool is the only tool the agent sees on turn one.


```python
def make_search_tool(registry: DeferredToolRegistry) -> Tool:
return Tool(
name="search_tools",
description=(
"Search the available tool catalog. "
"Returns matching tool names and descriptions. "
"Use this first to find the right tool before calling it. "
"Example: search_tools('pull request') or search_tools('database query')."
),
parameters={
"type": "object",
"properties": {
"query": {
"type": "string",
"description": "Short description of the capability you need (e.g., 'send email', 'query database')",
}
},
"required": ["query"],
},
fn=lambda query: registry.search(query),
category="system",
)

```

One tool, 200 tokens. It replaces 40 tools at 8,000 tokens.

Step 3: The Agent Loop with Deferred Loading
```python
import json

class DeferredAgent:
def __init__(self, model, system_prompt: str, registry: DeferredToolRegistry):
self.model = model
self.system_prompt = system_prompt
self.registry = registry
self.max_turns = 8
self.search_tool = make_search_tool(registry)

def run(self, user_input: str) -> str:
# Turn 1: Only the search tool is active
self.registry.load([self.search_tool.name])
messages = [
{"role": "system", "content": self.system_prompt},
{"role": "user", "content": user_input},
]

for turn in range(self.max_turns):
tools = self.registry.load(list(self.registry.active_tools))
response = self.model.chat(messages=messages, tools=tools)

if not response.tool_calls:
return response.content  # Final answer

messages.append(response.message)
for call in response.tool_calls:
if call.function.name == "search_tools":
# Agent searched — load the results and re-loop
results = self.registry.search_tool.fn(
**json.loads(call.function.arguments)
)
tool_names = [r["name"] for r in results]
self.registry.load(tool_names)
messages.append({
"role": "tool",
"content": f"Found tools: {[r['name'] for r in results]}. You now have access to these tools. Proceed with your task.",
"tool_call_id": call.id,
})
```
else:
# Agent used a loaded tool
try:
```python
args = json.loads(call.function.arguments)
result = self.registry.execute(call.function.name, args)
except Exception as e:
result = f"Error: {e}"
messages.append({
"role": "tool", "content": str(result), "tool_call_id": call.id,
})

self.registry.unload_all()
self.registry.load([self.search_tool.name])

return "Max turns reached."

```

The key behavioral pattern:

Start with search only — agent cannot call anything else.
Agent searches — gets back tool names that match its need.
Tools activate — the system loads matching tools into the active set.
Agent uses tools — calls the loaded tools to accomplish the task.
Unload after each session — next request starts clean.
Step 4: Real-World Tool Registration

Here's how you'd register actual tools:


```python
registry = DeferredToolRegistry()

registry.register(Tool(
name="get_pull_request",
description="Fetch details of a specific pull request including status, files changed, and review comments.",
parameters={"type": "object", "properties": {
"repo": {"type": "string", "description": "Repository in owner/repo format"},
"pr_number": {"type": "integer", "description": "Pull request number"},
}, "required": ["repo", "pr_number"]},
fn=lambda repo, pr_number: {"status": "open", "files": 12},
category="github",
))

registry.register(Tool(
name="query_database",
description="Execute a read-only SQL query against the analytics database. Only SELECT allowed.",
parameters={"type": "object", "properties": {
"sql": {"type": "string", "description": "SELECT query to execute"},
"limit": {"type": "integer", "description": "Max rows to return (default 100)"},
}, "required": ["sql"]},
fn=lambda sql, limit=100: {"rows": []},
category="database",
))

registry.register(Tool(
name="send_email",
description="Send an email to a recipient with subject and body.",
parameters={"type": "object", "properties": {
"to": {"type": "string", "description": "Recipient email address"},
"subject": {"type": "string"},
"body": {"type": "string"},
}, "required": ["to", "subject", "body"]},
fn=lambda to, subject, body: "Email sent",
category="email",
))

```

With three tools registered, the search function works like this:


```python
registry.search("database")
# Returns: [{'name': 'query_database', 'description': '...', 'score': 3}]

registry.search("email newsletter")
# Returns: [{'name': 'send_email', 'description': '...', 'score': 1}]

```

The agent searches for "database", the system loads query_database, the agent calls it, responds, and the session ends. Total tokens spent on tool definitions: ~400 instead of ~1,200.

Step 5: Production Optimizations

The basic version above works. Here's how to make it production-grade:

Semantic Search (Better Than Keyword Matching)

Keyword matching misses synonyms. An agent searching for "fetch data" won't find query_database. Use embeddings:


```python
import numpy as np

class SemanticToolSearch:
def __init__(self, registry: DeferredToolRegistry):
from sentence_transformers import SentenceTransformer
self.model = SentenceTransformer("all-MiniLM-L6-v2")
self.registry = registry
self._build_index()

def _build_index(self):
self.tool_names = []
self.embeddings = []
for name, tool in self.registry.all_tools.items():
text = f"{name}: {tool.description}"
self.tool_names.append(name)
self.embeddings.append(self.model.encode(text))
self.index = np.array(self.embeddings)

def search(self, query: str, top_k: int = 5) -> list[str]:
query_embed = self.model.encode(query)
similarities = np.dot(self.index, query_embed)
top_indices = np.argsort(similarities)[::-1][:top_k]
return [self.tool_names[i] for i in top_indices]

```

Now search("fetch user data") finds query_database even though the word "fetch" isn't in the tool name or description. The embedding model is ~90MB and runs in <10ms on CPU.

Tool Groups and Hierarchical Loading

Instead of loading individual tools, load groups:


```python
tool_groups = {
"github": ["get_pull_request", "create_issue", "update_changelog"],
"database": ["query_database", "list_tables"],
"email": ["send_email", "list_inbox"],
}

def load_group(group_name: str):
self.registry.load(tool_groups[group_name])

```

Add a load_tool_group(group) meta-tool. Agent searches "GitHub stuff" → loads all three GitHub tools at once instead of discovering them one at a time. This saves LLM turns when the task spans multiple related tools.

Budget Enforcement on Tool Loading

Prevent the agent from loading every tool by searching broadly:


```python
class LoadBudget:
def __init__(self, max_tools_per_session: int = 5):
self.max_tools = max_tools_per_session
self.loaded_count = 0

def can_load(self, count: int) -> bool:
return self.loaded_count + count <= self.max_tools

def record(self, count: int):
self.loaded_count += count

```

Five tools per session is usually enough. If the agent hits the limit, it must work with what it has — no more loading.

Token Savings: The Numbers

Here's the comparison that matters:

Approach	Token Cost	LLM Turns	Accuracy
Load all 40 tools	~8,000 per call	2-3	62%
Deferred loading	~400 per call	3-4	88%
Deferred (semantic)	~400 per call	2-3	91%

The accuracy improvement comes from reduced noise. When an LLM sees 40 tool definitions, it picks the wrong one more often. When it sees 3 relevant tools, the selection is cleaner. GPT-4o and Claude Sonnet both show 20-30% accuracy gains with deferred loading on the ToolRet benchmark (43,000 tool evaluation).

When NOT to Use Deferred Loading

Deferred loading isn't free. It adds an extra LLM turn. Skip it when:

Fewer than 10 tools — the overhead isn't worth the savings.
Latency-critical applications — the search-then-load pattern adds 1-2 seconds.
Cost-insensitive prototypes — if you're just testing, load everything and iterate fast.

Use it when:

20+ tools — token bloat becomes significant.
Autonomous agents — agents that run unattended need the most guardrails, including context control.
MCP-heavy stacks — connecting to 5+ MCP servers means 50+ tools easily.
Where Managed Platforms Handle This

Building this infrastructure yourself teaches you the pattern. But the three pieces — tool search, deferred loading, and budget enforcement — are platform concerns, not application logic.

Platforms like Nebula implement this natively: when an agent connects to multiple MCP servers, the platform provides a search_tools capability that discovers tools across all connected servers. The agent starts with search, finds the right tool, calls it, and the platform manages the lifecycle. You define the constraints (max tools per session, budget limits) in the agent config; the platform enforces them.

Actionable Takeaways
Start with search, not tools. Give your agent one meta-tool that searches the catalog. It's cheaper and more accurate than loading everything.
Keep tool definitions short. A 200-token tool definition with a clear description beats a 500-token essay. The LLM doesn't need your implementation details.
Load groups, not individuals. Related tools load together — search for "GitHub" and get PR, issue, and repo tools in one shot.
Cap your tool budget. Five active tools per session is enough for most tasks. Force the agent to be selective.
Use semantic search if you have 30+ tools. Keyword matching degrades fast. Embeddings handle synonyms naturally.
Unload after each session. Don't let tools accumulate across requests. Start fresh each time.

The trend is clear: with 177,000+ public tools in the MCP ecosystem, the engineering challenge isn't connecting tools anymore. It's choosing which ones to show the agent at any given moment. Shrink first, plan second — and your agent will perform better while costing less.

The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
How to Build Deferred Tool Loading for AI Agents in 15 Minutes
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools