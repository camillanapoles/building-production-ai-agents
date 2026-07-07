# How to Stop AI Agent Cost Spirals Before They Start

> Artigo #4 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/how-to-stop-ai-agent-cost-spirals-before-they-start-2k1a)

---

You wake up to a $500 OpenAI bill. Your agent ran overnight, looping through a research task that should have taken two minutes. Each iteration re-sent the full conversation history, retried failed tool calls three times each, and used GPT-4 for every step — including the ones that just formatted JSON.

This is the AI agent cost spiral, and it hits nearly every team that ships agents to production. The pattern is predictable: context windows bloat, retry storms compound, and unbounded tool chains drain budgets while you sleep.

Most guides tell you how to cut costs after the damage. This article takes the opposite approach. Here are five AI agent cost optimization patterns that prevent spirals from starting — each with production Python code you can drop into your stack today.

Pattern 1: Token Budgets Per Task

The simplest cost control is a hard ceiling. Before any agent run starts, set a token budget. When the budget is exhausted, the run stops — no exceptions.

Most teams skip this because they assume costs are unpredictable. They're not. After a few runs, you can estimate expected token usage for each task type. Set the budget at 3x your expected usage to handle variance, and you'll catch runaways without blocking legitimate work.


```python
class TokenBudget:
def __init__(self, max_tokens: int):
self.max_tokens = max_tokens
self.used = 0

def track(self, input_tokens: int, output_tokens: int):
self.used += input_tokens + output_tokens
if self.used >= self.max_tokens:
raise BudgetExceeded(
f"Token budget exhausted: {self.used}/{self.max_tokens}"
)

@property
def remaining(self) -> int:
return max(0, self.max_tokens - self.used)


class BudgetExceeded(Exception):
pass


# Usage: wrap your LLM calls
budget = TokenBudget(max_tokens=50_000)  # ~$0.50 at GPT-4 pricing

for step in agent.run(task):
response = llm.chat(step.messages)
budget.track(
input_tokens=response.usage.prompt_tokens,
output_tokens=response.usage.completion_tokens,
)
# BudgetExceeded fires automatically if limit is hit

```

The key insight: token budgets should be per-task, not global. A research task might need 100K tokens. A formatting task needs 5K. A single global budget masks the expensive outliers. Set budgets by task type, and you'll spot waste immediately.

Rule of thumb: Estimate expected tokens, multiply by 3, use that as the ceiling. Tighten over time as you collect data.

Pattern 2: Tiered Model Routing

Not every agent step needs your most expensive model. Most workflows are 60-70% routine tasks — classification, formatting, simple extraction — that a cheap model handles perfectly.

The fix is a router that classifies each step and picks the cheapest model that can handle it reliably:


```python
import openai

MODEL_TIERS = {
"flash": "gpt-4o-mini",       # $0.15/$0.60 per 1M tokens
"standard": "gpt-4o",          # $2.50/$10 per 1M tokens
"complex": "o3-mini",          # $1.10/$4.40 per 1M tokens (reasoning)
}

def route_model(task_description: str, requires_reasoning: bool = False) -> str:
"""Pick the cheapest model that can handle the task."""
if requires_reasoning:
return MODEL_TIERS["complex"]

low_complexity_signals = [
"format", "extract", "classify", "summarize",
"parse", "convert", "list", "filter",
]
task_lower = task_description.lower()
if any(signal in task_lower for signal in low_complexity_signals):
return MODEL_TIERS["flash"]

return MODEL_TIERS["standard"]


# Example: route based on what the agent is doing
model = route_model("extract email addresses from this text")
# Returns: gpt-4o-mini (flash tier — 17x cheaper than standard)

model = route_model("debug this race condition", requires_reasoning=True)
# Returns: o3-mini (complex tier — reasoning needed)

```

The savings are dramatic. If 60% of your agent's steps are flash-tier tasks, you cut those costs by 17x. On a $1,000/month agent bill, that's roughly $500 saved by changing a few lines of routing logic.

For production systems, consider a lightweight classifier that examines the prompt and routes automatically. The classifier itself runs on the flash tier — costing fractions of a cent per classification while saving dollars per routed call.

Pattern 3: Context Window Pruning

Context window bloat is the silent budget killer. Every agent call sends the system prompt, full conversation history, tool schemas, and retrieved documents. A single turn can hit 30,000 input tokens — and your agent makes dozens of turns per task.

Three pruning strategies, ordered by implementation effort:

Sliding window — keep only the last N messages. Simple and effective for tasks where recent context matters more than full history.

Summary compression — every K turns, compress the conversation into a summary. This is the sweet spot for most agent workloads:


```python
def compress_history(
messages: list[dict],
llm_client,
keep_recent: int = 4,
max_summary_tokens: int = 300,
) -> list[dict]:
"""Compress old messages into a summary, keep recent ones intact."""
if len(messages) <= keep_recent:
return messages

old_messages = messages[:-keep_recent]
recent_messages = messages[-keep_recent:]

summary_response = llm_client.chat.completions.create(
model="gpt-4o-mini",  # Use cheap model for summarization
messages=[
{
"role": "system",
"content": (
"Summarize this conversation in under "
f"{max_summary_tokens} tokens. "
"Preserve key decisions, results, and context."
),
},
*old_messages,
],
max_tokens=max_summary_tokens,
)

summary = summary_response.choices[0].message.content
return [
{"role": "system", "content": f"Previous context: {summary}"},
*recent_messages,
]


Relevant-only retrieval — instead of injecting all tool results into context, store them in a scratchpad and retrieve only what's relevant to the current step. This works best for agents with many tool calls.
```

The numbers: a 20-turn conversation with full history carries ~50K tokens of context. After summary compression, that drops to ~5K — a 90% reduction in input tokens. At GPT-4o pricing, that saves roughly $0.11 per compression cycle. Across hundreds of daily agent runs, it adds up fast.

Pattern 4: Circuit Breakers for Agent Loops

The most dangerous cost pattern is the infinite loop. An agent hits an error, retries, hits the same error, retries again — each time sending the full context. Without a circuit breaker, a single stuck task can burn through your entire daily budget.

Circuit breakers detect runaway behavior and kill the loop before it drains your wallet:


```python
import time
from dataclasses import dataclass, field


@dataclass
class CircuitBreaker:
max_steps: int = 25
max_cost_usd: float = 2.00
max_consecutive_errors: int = 3
_step_count: int = field(default=0, init=False)
_total_cost: float = field(default=0.0, init=False)
_consecutive_errors: int = field(default=0, init=False)

def record_step(self, cost_usd: float, success: bool):
self._step_count += 1
self._total_cost += cost_usd

if success:
self._consecutive_errors = 0
else:
self._consecutive_errors += 1

self._check_breakers()

def _check_breakers(self):
if self._step_count >= self.max_steps:
raise CircuitOpen(
f"Step limit reached: {self._step_count}/{self.max_steps}"
)
if self._total_cost >= self.max_cost_usd:
raise CircuitOpen(
f"Cost limit reached: ${self._total_cost:.2f}/${self.max_cost_usd:.2f}"
)
if self._consecutive_errors >= self.max_consecutive_errors:
raise CircuitOpen(
f"Error streak: {self._consecutive_errors} consecutive failures"
)


class CircuitOpen(Exception):
pass


# Usage
breaker = CircuitBreaker(max_steps=25, max_cost_usd=2.00)

for step in agent.run(task):
try:
result = execute_step(step)
cost = calculate_step_cost(result)
breaker.record_step(cost_usd=cost, success=True)
except StepError as e:
breaker.record_step(cost_usd=cost, success=False)
# CircuitOpen fires after 3 consecutive failures

```

The three breaker conditions — step count, cost ceiling, and error streaks — catch different failure modes. Step limits catch infinite loops. Cost ceilings catch expensive-but-technically-succeeding runs. Error streaks catch agents that are stuck but keep retrying.

When the circuit opens, don't just fail silently. Log the task state, notify the team, and queue the task for human review. The $2 you spent hitting the breaker is nothing compared to the $200 you'd spend without one.

Pattern 5: Cache Deterministic Tool Results

Many agent tool calls return the same result every time. File reads, API lookups for static data, configuration checks — these don't change between calls, but agents re-execute them on every run.

A simple time-aware cache eliminates the redundant calls:


```python
import hashlib
import json
import time
from functools import wraps


def cached_tool(ttl_seconds: int = 3600):
"""Cache tool results based on input arguments."""
cache: dict[str, tuple[float, any]] = {}

def decorator(func):
@wraps(func)
def wrapper(*args, **kwargs):
key = hashlib.sha256(
json.dumps({"args": args, "kwargs": kwargs}, sort_keys=True, default=str).encode()
).hexdigest()

if key in cache:
timestamp, result = cache[key]
if time.time() - timestamp < ttl_seconds:
return result  # Cache hit — zero API cost

result = func(*args, **kwargs)
cache[key] = (time.time(), result)
return result

return wrapper
return decorator


# Apply to your agent's tools
@cached_tool(ttl_seconds=3600)  # Cache for 1 hour
def lookup_user(user_id: str) -> dict:
return database.query(f"SELECT * FROM users WHERE id = '{user_id}'")


@cached_tool(ttl_seconds=86400)  # Cache for 24 hours
def get_config(key: str) -> str:
return config_service.get(key)

```

# First call: hits the database. Second call: returns cached result.
lookup_user("usr_123")  # DB query
lookup_user("usr_123")  # Cache hit — instant, free


Set TTL based on data freshness requirements: 5 minutes for user-facing data, 1 hour for reference data, 24 hours for configuration. Even a conservative caching strategy typically eliminates 30-50% of redundant tool calls in agent workflows.

What not to cache: Anything time-sensitive (stock prices, live status), user-specific mutations (write operations), or results that depend on external state that changes frequently.

Putting It All Together: The Cost-Aware Agent Stack

These five patterns layer together into a defense-in-depth cost architecture:

Layer	Pattern	Effort	Typical Savings	Catches
1	Token Budgets	30 min	10-20%	Runaway tasks
2	Model Routing	1-2 hrs	30-50%	Model overspend
3	Context Pruning	2-4 hrs	15-30%	Context bloat
4	Circuit Breakers	1-2 hrs	5-15%	Infinite loops
5	Result Caching	1-2 hrs	10-20%	Redundant calls

Start with model routing and token budgets. They cover 80% of cost problems with the least implementation effort. Add context pruning when your agents handle multi-turn conversations. Add circuit breakers before any agent runs unattended. Add caching last — it's the most situational.

The order matters. Token budgets set the ceiling so nothing can spiral. Model routing reduces the baseline cost of every call. Context pruning shrinks the payload. Circuit breakers catch the edge cases. Caching eliminates the repeat work.

Platforms like Nebula take this further by building cost controls into the agent architecture itself — step budgets per task, automatic model tier routing, and multi-agent delegation that isolates costs per sub-agent so one runaway task can't drain the shared budget.

TL;DR

Five patterns to stop AI agent cost spirals before they start:

Token budgets per task — set a hard ceiling before the run starts. 3x expected usage.
Tiered model routing — use the cheapest model that can handle each step. 60-70% of steps don't need your best model.
Context window pruning — compress old conversation history into summaries. 90% reduction in context tokens.
Circuit breakers — kill runaway loops after N steps, $X cost, or K consecutive errors.
Cache deterministic results — don't re-execute tool calls that return the same data.

Implement them in order. Model routing and token budgets alone will cut your agent bill in half.

This article is part of the Building Production AI Agents series. Previous: Your AI Agent Is One Prompt Away From Disaster. See also: 5 AI Agent Failures in Production and Single Agent vs Multi-Agent: Why Monoliths Fail.

The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
How to Stop AI Agent Cost Spirals Before They Start
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools