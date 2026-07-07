# AI Agent Observability: The 4 Pillars That Keep Your Agents from Burning $2,000 at 3 AM

> Artigo #20 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/ai-agent-observability-the-4-pillars-that-keep-your-agents-from-burning-2000-at-3-am-5882)

---

Last month, a production agent at a startup started hallucinating customer data. Every API call returned 200 OK. The agent's dashboard showed green across the board. Latency was normal. Error rate was zero.

Six hours later, the billing arrived: $2,847 in tokens for a single user query that entered a reasoning loop and never stopped.

This isn't a hypothetical. The Operator Collective documented $47,000 runaway agent invoices in 2025. In 2026, the numbers are only going up as agents get more autonomous and cheaper per token.

The problem is fundamental: traditional monitoring was built for deterministic software. Request comes in, code executes, response goes out. Same input always produces same output. You know what \"healthy\" looks like.

AI agents break that contract. The same prompt can spiral through 15 reasoning iterations, call 40 tools, and produce a confident but completely wrong answer — all returning HTTP 200. Your dashboards stay green. Your agent is on fire.

After running autonomous agents 24/7 on production infrastructure and watching them fail in ways I never anticipated, I've built the observability stack that actually catches these failures. It comes down to four pillars that traditional APM doesn't cover.

Pillar 1: Cost Observability — Token Tracking with Anomaly Detection

The most urgent failure mode isn't correctness — it's cost. An agent in a reasoning loop has no natural stopping point unless you enforce one at the observability layer.

Per-Run Cost Attribution

Every agent execution needs unique identifiers: trace ID, session ID, and agent ID. Each LLM call and tool invocation gets tagged with these identifiers so you can attribute cost at any granularity.


```python
import time
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class TokenLedger:
\"\"\"Track token usage per agent, per run, per call.\"\"\"
trace_id: str
session_id: str
agent_id: str

input_tokens: int = 0
output_tokens: int = 0
cost_usd: float = 0.0
```

# Pricing config — update per model
input_cost_per_1k: float = 0.003  # Sonnet 4 input
output_cost_per_1k: float = 0.015  # Sonnet 4 output

```python
def record(self, input_tok: int, output_tok: int) -> float:
\"\"\"Record a single LLM call. Returns the cost.\"\"\"
self.input_tokens += input_tok
self.output_tokens += output_tok
call_cost = (
(input_tok / 1_000) * self.input_cost_per_1k
+ (output_tok / 1_000) * self.output_cost_per_1k
)
self.cost_usd += call_cost
return call_cost

def running_cost(self) -> float:
return self.cost_usd

```

Integrated into an agent loop with a hard budget:


MAX_RUN_COST = 0.50  # Hard cap per agent session

```python
ledger = TokenLedger(
trace_id=\"trace_abc123\",
session_id=\"sess_xyz789\",
agent_id=\"support-agent-v2\",
)

while agent_running:
if ledger.running_cost() >= MAX_RUN_COST:
logger.error(
f\"Budget exceeded: ${ledger.running_cost():.3f} \"
f\"(limit ${MAX_RUN_COST:.2f}). Aborting run.\"
)
# Save partial results, escalate to human
break

response = call_llm(prompt, model=\"claude-sonnet-4\")
ledger.record(
input_tok=response.usage.input_tokens,
output_tok=response.usage.output_tokens,
)

# ... process response ...

Real-Time Anomaly Alerts
```

Budget caps prevent catastrophic runs. Anomaly detection catches creeping cost increases before they blow past the cap.


```python
from collections import deque
import time

class CostAnomalyDetector:
\"\"\"Alert when token burn rate exceeds 3x the rolling average.\"\"\"

def __init__(self, window_seconds: int = 300, threshold_multiplier: float = 3.0):
self.window = window_seconds
self.threshold = threshold_multiplier
self.burn_rates: deque[tuple[float, float]] = deque()  # (timestamp, rate)

def record_burn(self, tokens_per_second: float) -> Optional[str]:
\"\"\"Returns alert message if anomaly detected, else None.\"\"\"
now = time.time()
self.burn_rates.append((now, tokens_per_second))
```

# Prune old entries
cutoff = now - self.window
```python
self.burn_rates = deque(
(ts, rate) for ts, rate in self.burn_rates if ts >= cutoff
)

if len(self.burn_rates) \u003C 5:
return None  # Not enough data

avg_rate = sum(r for _, r in self.burn_rates) / len(self.burn_rates)
if tokens_per_second > avg_rate * self.threshold:
return (
f\"ANOMALY: burn rate {tokens_per_second:.0f} tps \"
f\"is {tokens_per_second/avg_rate:.1f}x above average \"
f\"({avg_rate:.0f} tps)\"
)
return None

```

The alert fires within seconds — not when the monthly bill arrives. I run this as a sidecar process that polls the agent's ledger every 10 seconds and fires a Slack webhook if the burn rate spikes.

Pillar 2: Quality Observability — Canary Evaluations in Production

Pre-deployment evals tell you how the agent performed before going live. Production quality monitoring tells you whether it's still performing correctly.

Canary Query Suite

Pick 10-20 queries with known-correct answers. Run them every 5 minutes. Alert when accuracy drops below 90%.


```python
from dataclasses import dataclass

@dataclass
class CanaryQuery:
question: str
expected_answer: str
evaluation_prompt: str  # LLM-as-judge prompt

CANARY_QUERIES = [
CanaryQuery(
question=\"What's the status of deployment d-4821?\",
expected_answer=\"completed\",
evaluation_prompt=(
\"Does the agent's response indicate that deployment d-4821 \"
\"was completed successfully? Answer YES or NO.\"
),
),
CanaryQuery(
question=\"Which region has the most errors today?\",
expected_answer=\"us-east-1\",
evaluation_prompt=(
\"Did the agent correctly identify us-east-1 as the region \"
\"with the most errors? Answer YES or NO.\"
),
),
]

def run_canary_suite(agent, queries: list[CanaryQuery]) -> dict:
results = {}
for q in queries:
response = agent.run(q.question)
# LLM-as-judge evaluation
judge_result = llm.ask(
f\"{q.evaluation_prompt}\
```
\
Agent response: {response}\"
```python
)
passed = \"yes\" in judge_result.text.lower()
results[q.question] = passed

pass_rate = sum(results.values()) / len(results)
return {\"pass_rate\": pass_rate, \"details\": results}

```

If pass_rate drops below 0.9, something changed: a model provider degraded, a tool API changed its response format, or a prompt update broke something. The canary suite catches it before customers notice.

Semantic Drift Detection

Beyond binary pass/fail, track whether the agent's responses are becoming less semantically similar to historical good answers. This catches the slow degradation that canary queries might miss.


```python
import numpy as np
from sentence_transformers import SentenceTransformer

class SemanticDriftDetector:
def __init__(self, baseline_responses: list[str]):
self.model = SentenceTransformer(\"all-MiniLM-L6-v2\")
self.baseline_embeddings = self.model.encode(baseline_responses)

def check_drift(self, current_responses: list[str]) -> float:
current_embeddings = self.model.encode(current_responses)
# Cosine similarity between baseline and current
similarities = np.mean(
np.dot(current_embeddings, self.baseline_embeddings.T), axis=1
)
return float(np.mean(similarities))  # 1.0 = identical, 0.0 = unrelated

```

Drift below 0.7 means the agent's output style or content has fundamentally shifted — time to roll back and investigate.

Pillar 3: Behavioral Observability — Tracing the Agent's Reasoning

This is where agent observability diverges most sharply from traditional APM. You need to see what the agent decided, not just what HTTP requests it made.

The Structured Agent Log

Every agent action produces structured telemetry. Not text logs that require regex parsing, but JSON objects with a consistent schema:


{
```python
\"timestamp\": \"2026-04-30T04:23:45Z\",
\"trace_id\": \"trace_abc123\",
\"span_id\": \"span_001\",
\"event_type\": \"tool_call\",
\"agent_name\": \"support-agent-v2\",
\"tool_name\": \"query_database\",
\"tool_input\": {\"query\": \"SELECT status FROM deployments WHERE id = 'd-4821'\"},
\"tool_output_summary\": \"status=completed\",
\"latency_ms\": 142,
\"input_tokens\": 245,
\"output_tokens\": 1023,
\"cost_usd\": 0.002,
\"reasoning_depth\": 3,
\"confidence_score\": 0.87,
\"parent_span_id\": null
}

```

The key fields beyond standard logging:

reasoning_depth — how many times the agent has looped. If this number climbs past 8, the agent is probably in a reasoning spiral.
confidence_score — the agent's self-assessed confidence (if your prompt requests it). Low confidence + high reasoning depth = almost certainly a stuck agent.
tool_output_summary — not the full output (too expensive to store), but a truncated preview plus a status indicator (success/error/partial).
Tool-Call Attribution

When your database monitoring fires for \"10,000 queries in 5 minutes,\" you need to know which agent triggered them and why. Every tool call traces back to a specific reasoning step.


```python
class ToolCallTracker:
def __init__(self, max_calls_per_tool: int = 50):
self.max_calls = max_calls_per_tool
self.call_counts: dict[str, int] = {}
self.call_history: list[dict] = []

def record(self, tool_name: str, reasoning_step: dict) -> None:
self.call_counts[tool_name] = self.call_counts.get(tool_name, 0) + 1
self.call_history.append({
\"tool\": tool_name,
\"step\": reasoning_step,
\"call_number\": self.call_counts[tool_name],
})

if self.call_counts[tool_name] > self.max_calls:
raise ToolCallLimitExceeded(
f\"Tool '{tool_name}' called {self.call_counts[tool_name]} times \"
f\"(limit {self.max_calls})\"
)

def get_distribution(self) -> dict[str, int]:
return dict(self.call_counts)

```

This pairs with the cost ledger: you can see not just how much the agent spent, but which tool drove the spend. If search_database accounts for 80% of tool calls, the agent might be over-relying on retrieval when a simpler tool would suffice.

Pillar 4: Dependency Observability — Mapping the Agent's External World

An agent depends on LLM providers, tool APIs, vector databases, other agents, and infrastructure services. When something breaks, you need to know whether it's your agent or a dependency.

The Dependency Health Map

Your health check shouldn't just return {\"status\": \"ok\"}. It should test each dependency and report granular status:


```python
async def agent_health_check(agent) -> dict:
start = time.time()

# Test LLM connectivity and latency
llm_start = time.time()
llm_response = await agent.llm.complete(\"Say 'healthy'\")
llm_latency = (time.time() - llm_start) * 1000

# Test each tool's availability
tools_status = {}
for tool in agent.tools:
try:
await tool.ping(timeout=3.0)
tools_status[tool.name] = \"healthy\"
except Exception as e:
tools_status[tool.name] = f\"error: {type(e).__name__}\"

# Run canary eval
canary_start = time.time()
canary_result = await agent.run(\"What is 2+2?\")
canary_latency = (time.time() - canary_start) * 1000
canary_passed = \"4\" in canary_result

return {
\"status\": \"healthy\" if canary_passed and all(
s == \"healthy\" for s in tools_status.values()
) else \"degraded\",
\"llm_latency_ms\": round(llm_latency),
\"model_version\": agent.llm.model,
\"tools\": tools_status,
\"canary_passed\": canary_passed,
\"canary_latency_ms\": round(canary_latency),
\"uptime_seconds\": agent.uptime(),
}

```

This endpoint runs every 60 seconds and feeds into your monitoring dashboard. When the vector database goes down, you see vector_search: \"error: ConnectionRefusedError\" instead of wondering why the agent's accuracy dropped 40%.

Agent-to-Agent Tracing

Multi-agent systems are the hardest to debug because failures cascade. Agent A produces bad output → Agent B consumes it and amplifies the error → Agent C approves the result because it only saw the final draft.

The fix is distributed tracing across agent boundaries. Each agent execution becomes a span, and child spans capture individual LLM calls and tool invocations:


trace: user-query-abc123
```python
├── span: agent.research (2.4s, $0.12)
│   ├── span: gen_ai.chat — query planning (0.3s)
│   ├── span: tool.vector_search (0.8s)
│   ├── span: tool.web_search (0.6s)
│   └── span: gen_ai.chat — synthesize findings (0.7s)
├── span: agent.writer (1.8s, $0.08)
│   ├── span: gen_ai.chat — draft generation (1.2s)
│   └── span: gen_ai.chat — self-review (0.6s)
└── span: agent.reviewer (1.1s, $0.05)
├── span: gen_ai.chat — quality check (0.8s)
└── span: gen_ai.chat — scoring (0.3s)

```

This trace answers the questions that flat LLM logging cannot: which agent was slowest, which agent was most expensive, what data flowed between agents, and where the chain broke.

The OpenTelemetry GenAI semantic conventions (still experimental as of early 2026) define standard attributes for this: gen_ai.system, gen_ai.request.model, gen_ai.usage.input_tokens, and the new agent-specific gen_ai.agent.name and gen_ai.agent.id. If you're building the instrumentation layer yourself, follow these conventions — they make your data portable across observability backends.

The Dashboard That Actually Helps

Your agent dashboard should answer five questions at a glance:

How much are we spending right now? — real-time token burn rate with budget gauges
Is quality holding? — canary pass rate, confidence score distribution over time
Are agents behaving normally? — tool call distribution histogram, reasoning depth distribution
Any cascading issues? — dependency map with live status per tool/API/model
Which users/sessions are affected? — error rate by user segment, not just aggregate

Here's the structure I use:


┌─────────────────────────────────────────────────┐
│ COST REAL-TIME                                  │
│ Today: $4.23  |  Current run: $0.12  |  Budget: OK │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   │
│ Burn rate: 450 tok/s (normal: 50-200) [WARN]    │
├─────────────────────────────────────────────────┤
│ QUALITY                                         │
│ Canary: 9/10 passed (90%)  |  Drift: 0.82 OK   │
│ Confidence avg: 0.84  |  Loops avg: 2.3         │
├─────────────────────────────────────────────────┤
│ BEHAVIORAL                                      │
│ Tool calls (last hour):                          │
│   search_docs: 847  |  query_db: 423  |          │
│   send_email: 12  |  create_ticket: 8           │
│ Reasoning depth histogram:  1:60%  2:25%        │
│                                  3:10%  4+:5%    │
│                                  [WARN]          │
├─────────────────────────────────────────────────┤
│ DEPENDENCIES                                    │
│ LLM (Sonnet 4): [OK] 312ms                      │
│ Vector DB:  [OK] 12ms                            │
│ GitHub API: [WARN] 2.1s (degraded)              │
│ Email SMTP: [OK] 45ms                              │
└─────────────────────────────────────────────────┘

Where Platforms Like Nebula Fit In

Building this observability stack from scratch means instrumenting every agent loop, every tool call, every LLM response, wiring up the cost ledger, setting up the canary suite, and maintaining the tracing infrastructure. It's necessary work, but it's not the work that differentiates your product.

Platforms like Nebula handle the observability layer as part of the agent runtime. Every agent execution is traced end-to-end with cost attribution, token budgets are enforced at the platform level (not in your application code), and tool-call attribution happens automatically because every tool is registered in the platform's service registry.

The architecture looks like this:


Agent Definition (your config)
│
▼
┌─────────────────────────────────┐
│   Nebula Agent Runtime           │
│   ┌───────────────────────────┐  │
│   │  Execution Tracer          │  │
│   │  - Token ledger           │  │
│   │  - Cost attribution       │  │
│   │  - Span propagation       │  │
│   └───────────────────────────┘  │
│   ┌───────────────────────────┐  │
│   │  Guardrails Engine         │  │
│   │  - Budget enforcement     │  │
│   │  - Tool call limits       │  │
│   │  - Cycle caps             │  │
│   └───────────────────────────┘  │
│   ┌───────────────────────────┐  │
│   │  Quality Monitor           │  │
│   │  - Canary evals           │  │
│   │  - Output validation      │  │
│   │  - Drift detection        │  │
│   └───────────────────────────┘  │
└─────────────────────────────────┘
│
├──→ Tool: GitHub MCP Server
├──→ Tool: Database Query MCP
├──→ Tool: Web Search MCP
│
▼
Traces + Metrics → Your Dashboard


You define what the agent does. The platform ensures you can see when it goes wrong.

For teams already invested in an observability stack (Datadog, Grafana, New Relic), the agent traces export in OpenTelemetry format — standard spans you can ingest alongside your application telemetry. This means your SRE team doesn't need a separate dashboard for AI; agents are first-class signals in the same environment as the rest of your infrastructure.

Grafana Cloud recently shipped AI Observability in public preview, and it follows exactly this pattern: agent sessions become first-class telemetry, correlated with traces, metrics, and logs. The key insight both Grafana and Nebula share is that you don't need a new observability paradigm for AI — you need the existing one extended to understand agent semantics.

Actionable Takeaways

Start with cost tracking on day one. A token ledger with a hard budget cap prevents the $2,000-at-3-AM scenario. Everything else is optimization.

Run canary queries in production, not just in CI. Your evals before deployment tell you nothing about model degradation, tool API changes, or infrastructure drift that happens after you ship.

Structure your logs as events, not text. JSON telemetry with consistent schemas (trace ID, span ID, tool name, cost, reasoning depth) is queryable. Text logs require grep and guesswork.

Track reasoning depth and tool call counts per run. Two numbers that catch 80% of agent failure modes: reasoning depth exceeding 8 = stuck agent, tool call count exceeding expected range = wrong tool selection loop.

Map your dependencies explicitly. When the agent breaks, the first question should be \"is it us or a dependency?\" — and your health check should answer that instantly.

Export in OpenTelemetry format. Whether you use a managed platform or self-host, OTLP-standard spans mean you can swap observability backends without reinstrumenting. Don't lock yourself into a vendor's tracing format.

The agent observability gap isn't a technology problem — it's a mental model shift. Your agents are probabilistic systems executing non-deterministic workflows across external dependencies. Monitor them like distributed systems, not like HTTP endpoints. Do that, and the 3 AM wake-up calls become rare.

This article is part of the Building Production AI Agents series on Dev.to.

The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
AI Agent Observability: The 4 Pillars That Keep Your Agents from Burning $2,000 at 3 AM
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools