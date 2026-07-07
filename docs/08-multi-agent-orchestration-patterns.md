# Multi-Agent Orchestration: A Guide to Patterns That Work

> Artigo #8 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/multi-agent-orchestration-a-guide-to-patterns-that-work-4881)

---

Every multi-agent article opens with the same pitch: your single agent is failing, you need five agents, here are the patterns. The internet is full of pattern catalogs. What it lacks is honesty about when multi-agent orchestration actually helps -- and when it makes everything worse.

This article is different. We will cover the patterns, but we will start with the question nobody asks: do you actually need multi-agent? Then we will walk through the four patterns that cover 90% of production use cases, show you framework-agnostic Python code, and do the cost math that most guides skip.

This is part of the Building Production AI Agents series. If you have not read Why Your AI Workflow Breaks at Scale yet, start there -- it covers the three walls (monolith prompts, tool explosion, state amnesia) that push teams toward multi-agent in the first place.

When You Actually Need Multi-Agent

Before picking a pattern, confirm you have a multi-agent problem. Most teams jump to multi-agent because it sounds sophisticated. Then they spend months debugging coordination overhead that a single well-structured agent would have avoided entirely.

You need multi-agent orchestration when you hit one or more of these ceilings:

1. Context window saturation. Your agent's conversation history, tool outputs, and intermediate reasoning consume 80%+ of the context window. The model spends more tokens reading history than thinking about the current step. A 20-step ReAct trace with tool outputs can easily hit 50,000 tokens of accumulated context. At that point, the model's attention is diluted across so much history that it starts missing instructions and hallucinating tool calls.

2. Tool count exceeds 12. Anthropic's research confirms that agent accuracy degrades significantly past 15 tools and collapses past 50. Every tool definition in the context window competes for the model's attention during selection. We covered this in depth in MCP Tool Overload. If your single agent has 20+ tools, decomposition is not optional -- it is the fix.

3. Error compounding in long chains. A mistake at step 4 of a 20-step chain propagates silently. By step 18, you have a confident, coherent, completely wrong result. Shorter chains with handoffs between specialized agents create natural checkpoints where errors surface instead of snowballing.

If none of these apply, stay single-agent. Add a router to pick the right system prompt per request. Group your tools into clusters loaded dynamically. These are cheaper, simpler fixes than introducing agent coordination.

The 4 Patterns That Cover 90% of Use Cases

The internet describes 6, 7, even 10 orchestration patterns. In production, four patterns handle almost everything. The rest -- swarm, mesh, group chat -- are either niche or actively harmful when over-applied. Start simple. Graduate to complexity only when the simple pattern demonstrably fails.

Pattern 1: Sequential Pipeline

Agents run in a fixed order. Each agent receives the output of the previous one. Think: research, then write, then review, then edit.


User Input --> [Researcher] --> [Writer] --> [Reviewer] --> Final Output


Each agent carries a focused prompt and a small tool set. The researcher has web search. The writer has formatting tools. The reviewer has style guidelines. No agent is overwhelmed.

When to use it: Your workflow has a defined order of operations. Each step depends on the previous output. You need an audit trail of what each agent contributed.

When to avoid it: Subtasks are independent and could run in parallel. You are wasting wall-clock time running them sequentially.

Cost profile: Total tokens = input + sum of all outputs passed forward. A 3-agent pipeline with 1,000-token input and 500-token outputs costs roughly 1K + 1.5K + 2K = 4,500 tokens. That is 4.5x a single-agent call.

Pattern 2: Fan-Out / Fan-In (Concurrent)

The same input goes to multiple agents simultaneously. Their outputs are collected and merged by an aggregator.


+--> [Security Reviewer] --+
User Input ----> +--> [Perf Reviewer]     --+--> [Aggregator] --> Final Output
+--> [Style Reviewer]    --+


When to use it: Subtasks are genuinely independent. Speed matters more than token cost. You can write a meaningful aggregation strategy.

When to avoid it: Subtasks depend on each other's output. Your aggregator is weak -- a bad synthesizer loses information or introduces contradictions.

```python
Cost profile: Total tokens = (N agents x input tokens) + aggregation. More predictable than sequential, but scales linearly with agent count.

Pattern 3: Router / Handoff

A lightweight classifier picks which specialist handles the request. Only one specialist runs per request. This is the most common production pattern because most workloads are not "do five things at once" -- they are "figure out which one thing to do, then do it well."


User Input --> [Router] --> [Email Agent]
--> [Calendar Agent]
--> [Code Review Agent]

```

When to use it: Requests fall into distinct categories. Each category needs different tools, different prompts, different behavior. You cannot predict the category upfront.

When to avoid it: Every request needs all specialists to contribute. That is fan-out, not routing.

Cost profile: Cheapest multi-agent pattern. One small routing call + one specialist call. Total overhead is just the router's classification cost.

```python
Pattern 4: Hierarchical (Manager + Teams)

A manager agent decomposes a complex task, delegates subtasks to team leads, and team leads coordinate workers. Each level adds abstraction: the manager reasons about strategy, team leads handle tactics, workers execute.


[Manager]
|-- [Research Lead] --> [Web Searcher] --> [Doc Analyzer]
|-- [Writing Lead]  --> [Drafter] --> [Editor]

```

When to use it: The task spans multiple domains. No single agent can hold the full context. You need 10+ agents organized into logical groups.

When to avoid it: Your task fits in one domain. Hierarchical coordination adds 2-4 seconds of latency per level. A two-level hierarchy on a simple task is like hiring a project manager for a one-person project.

Cost profile: Highest overhead. Every level adds a full LLM call for planning and synthesis. A 3-level hierarchy adds minimum 6 seconds of latency before any worker starts.

Pattern Selection: Match Symptoms to Solutions

Instead of a flowchart, use this diagnostic. Find your symptom, get your pattern.

Your Symptom	Pattern	Why
Agent forgets instructions mid-task, context window is full	Sequential Pipeline	Split the work into stages. Each agent gets a fresh, focused context.
Agent picks the wrong tool 30%+ of the time	Router / Handoff	Give each specialist only the tools it needs. The router picks the specialist, not the tool.
Tasks take too long because subtasks run sequentially but do not depend on each other	Fan-Out / Fan-In	Parallelize independent work. Wall-clock time = slowest agent, not sum of all agents.
Single agent cannot hold enough context across multiple domains	Hierarchical	Distribute context across levels. No agent needs the full picture.
None of the above -- agent works fine but could be better	Stay single-agent	Seriously. Optimize your prompt, group your tools, add structured output. Do not add coordination overhead you do not need.
The Code: A Framework-Agnostic Multi-Agent Router

Here is the router/handoff pattern in plain Python. No framework dependencies. Works with any LLM API that supports function calling.


```python
import json
from openai import OpenAI

client = OpenAI()

# --- Specialist definitions ---
SPECIALISTS = {
"email": {
"prompt": "You are an email specialist. Triage, draft, and manage emails.",
"tools": [send_email, search_inbox, archive_email],
},
"calendar": {
"prompt": "You are a calendar specialist. Check availability and schedule meetings.",
"tools": [check_calendar, create_event],
},
"research": {
"prompt": "You are a research specialist. Search the web and summarize findings.",
"tools": [web_search, scrape_page, summarize],
},
}

# --- Router: one cheap LLM call to classify ---
def route_request(user_input: str) -> str:
"""Classify the request into a specialist category."""
response = client.chat.completions.create(
model="gpt-4o-mini",  # cheap, fast model for routing
messages=[
{"role": "system", "content": (
"Classify the user request into exactly one category: "
f"{", ".join(SPECIALISTS.keys())}. "
"Return JSON: {\"category\": \"...\", \"confidence\": 0.0-1.0}"
)},
{"role": "user", "content": user_input},
],
response_format={"type": "json_object"},
)
result = json.loads(response.choices[0].message.content)
return result["category"] if result["confidence"] > 0.7 else "unknown"

# --- Execute: specialist runs with focused context ---
def run_specialist(category: str, user_input: str) -> str:
"""Run the specialist agent with its dedicated prompt and tools."""
spec = SPECIALISTS[category]
response = client.chat.completions.create(
model="gpt-4o",  # capable model for execution
messages=[
{"role": "system", "content": spec["prompt"]},
{"role": "user", "content": user_input},
],
tools=[to_openai_tool(t) for t in spec["tools"]],
)
return handle_tool_calls(response)

# --- Main loop ---
def handle_request(user_input: str) -> str:
category = route_request(user_input)
if category == "unknown":
return "I'm not sure what you need. Could you clarify?"
return run_specialist(category, user_input)

```

Notice three things:

The router uses a cheap model. gpt-4o-mini for classification, gpt-4o for execution. Routing is a simple classification task -- do not waste expensive tokens on it.

Each specialist sees only its tools. The email agent has 3 tools. The calendar agent has 2. Nobody has 10+. This is the tool explosion fix applied structurally.

No framework. This is 40 lines of Python. You can swap OpenAI for Anthropic, Google, or a local model. The pattern is the same regardless of provider.

The Costs Nobody Talks About

Every multi-agent article shows you the architecture diagram. None of them show you the bill.

Token Multiplication

Multi-agent systems do not just use more tokens -- they multiply them.

Sequential pipeline (3 agents, 1K input, 500-token outputs):

Agent 1: 1,000 input tokens
Agent 2: 1,000 + 500 = 1,500 input tokens
Agent 3: 1,000 + 500 + 500 = 2,000 input tokens
Total: 4,500 input tokens (4.5x a single-agent call)

Fan-out (3 agents, 1K input):

Each agent: 1,000 input tokens
```python
Aggregator: 1,000 + (3 x 500) = 2,500 input tokens
Total: 5,500 input tokens (5.5x a single-agent call)
```

Group chat (the pattern most articles recommend first but should be recommended last):

5 agents, 10 rounds, 300-token average messages
Each round: every agent reads ALL previous messages
Total context: 15,000+ tokens just in accumulated history -- before any actual reasoning

The router/handoff pattern? One routing call (~200 tokens) plus one specialist call (~1,000 tokens). 1,200 tokens total. That is why it is the most common pattern in production.

Latency Stacking

Every agent in the chain adds 1-3 seconds of LLM latency. A 3-agent sequential pipeline adds 3-9 seconds minimum. A hierarchical system with 3 levels adds 6+ seconds before any worker starts executing. For batch workflows, this is fine. For real-time chat, it is a dealbreaker.

Debugging Overhead

A bug in a single-agent system means reading one conversation trace. A bug in a multi-agent system means reconstructing message flows across 3-10 agents, figuring out which agent made the wrong decision, and understanding how that decision propagated through the pipeline. Invest in logging every routing decision, every agent input/output, and every tool call. Without observability, multi-agent systems are black boxes.

Anti-Patterns That Kill Multi-Agent Systems
The Premature Orchestrator

You have 3 tools and 1 use case, but you built a router with 4 specialist agents because the architecture diagram looked cool. Your system is now slower, more expensive, and harder to debug than a single agent with 3 tools would have been. Multi-agent is a scaling strategy, not a starting strategy.

The God Orchestrator

Your orchestrator agent has a 3,000-token system prompt that describes every specialist, every routing rule, and every edge case. You have recreated the monolith prompt problem -- just moved it one level up. Keep the orchestrator dumb: classify the request, pick the specialist, hand off. Nothing else.

The Chatty Mesh

You read about mesh patterns and connected 8 agents in a full mesh. Each agent can talk to every other agent. You now have 28 possible communication channels, none of them observable, and your token cost is through the roof. Full mesh works with 3-4 tightly coupled agents iterating on a shared artifact. Beyond that, decompose into smaller groups.

The Framework Trap

You adopted a multi-agent framework before understanding the pattern you need. Now your code is locked into the framework's abstractions, and when you need to change the orchestration logic, you are fighting the framework instead of solving the problem. Learn the patterns first. Adopt a framework only when the plain-Python version becomes genuinely hard to maintain.

When to Start Simple

If you are building AI agents that need to coordinate across multiple tools and services, the router/handoff pattern gets you 80% of the way with 20% of the complexity. Platforms like Nebula implement this pattern natively -- agents delegate to specialist sub-agents, each with their own tools and context, with state and auth handled automatically across runs. Whether you build it yourself or use a platform, the principle is the same: start with the simplest orchestration that solves your problem.

The progression that works in practice:

Single agent with grouped tools and structured output
Router + specialists when tool count exceeds 12 or request types diverge
Sequential pipeline when tasks have a natural stage order
Fan-out when independent subtasks need parallel speed
Hierarchical when the problem genuinely spans multiple domains at scale

Skip steps 3-5 until step 2 demonstrably fails. Most production agent systems never get past step 2 -- and that is a sign of good engineering, not a limitation.

This is part of the Building Production AI Agents series. Previous: Why Your AI Workflow Breaks at Scale. Next up: evaluation pipelines that catch agent regressions before your users do.

The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
Multi-Agent Orchestration: A Guide to Patterns That Work
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools