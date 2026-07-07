# 5 AI Agent Error Handling Patterns That Keep Your Agent Running at 3 AM

> Artigo #18 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/5-ai-agent-error-handling-patterns-that-keep-your-agent-running-at-3-am-5d5o)

---

Last year, a deployment went sideways. An AI agent was running a data enrichment pipeline: pull records from an API, map fields into a schema, write to a database. Every API call returned 200 OK. The agent's dashboard showed green across the board. The agent reported success on every step.

Six hours later, a downstream team flagged the data. Half the field mappings were hallucinated. The agent had confidently mapped company_revenue to employee_count, invented values for missing fields, and written duplicates for records it had already processed. Hundreds of bad rows, all marked as verified.

Nobody noticed because nothing "failed."

This is the fundamental problem with AI agents in production: the most dangerous failures look exactly like success. Traditional error handling — try/catch blocks, HTTP status code checks, crash monitoring — was built for deterministic software. AI agents are probabilistic systems. They don't crash when they're wrong; they confidently produce garbage with a 200 status code.

After living through that incident and building dozens of production agents since, I've distilled five error handling patterns that actually work. Each pattern handles a failure mode the previous one can't catch. Together, they form a defense-in-depth strategy that keeps agents running, prevents silent data corruption, and gives you sleep on weeknight deployments.

Pattern 1: Circuit Breakers for LLM Quality Failures (Not Just HTTP Errors)

The classic circuit breaker pattern — closed, open, half-open — is standard infrastructure engineering. But when it comes to AI agents, the traditional version is incomplete. It tracks HTTP failures. It misses quality failures.

The Problem

After a model provider degradation, an agent started returning malformed JSON. Every API call succeeded. The HTTP status was 200. We burned 40 minutes of compute before anyone noticed because nothing in the error handling checked output quality — only transport status.

The Solution: Quality-Aware Circuit BREAKER

A circuit breaker for agents needs to track quality failures: outputs that violate schema, fail semantic invariants, or produce unsafe actions — even when the API itself succeeds.


```python
import time
from enum import Enum
from dataclasses import dataclass, field

class CircuitState(Enum):
CLOSED = "closed"
OPEN = "open"
HALF_OPEN = "half_open"

@dataclass
class QualityCircuitBreaker:
failure_threshold: int = 3
reset_timeout: float = 60.0
state: CircuitState = CircuitState.CLOSED
failures: int = 0
last_failure_time: float = field(default_factory=time.time)

def record_quality_failure(self) -> None:
"""Called when LLM output fails validation (not HTTP error)."""
self.failures += 1
self.last_failure_time = time.time()
if self.failures >= self.failure_threshold:
self.state = CircuitState.OPEN

def record_success(self) -> None:
if self.state == CircuitState.HALF_OPEN:
self.state = CircuitState.CLOSED
self.failures = 0

def allow_request(self) -> bool:
if self.state == CircuitState.CLOSED:
return True

if self.state == CircuitState.OPEN:
elapsed = time.time() - self.last_failure_time
if elapsed >= self.reset_timeout:
self.state = CircuitState.HALF_OPEN
return True  # One probe request
return False  # Circuit is open, reject
```

# Half-open: allow exactly one probe
return True

```python
def should_block(self) -> bool:
return not self.allow_request()

```

Usage in an agent loop:


```python
breaker = QualityCircuitBreaker(failure_threshold=3, reset_timeout=30.0)

while agent_running:
if breaker.should_block():
logger.warning("Circuit breaker OPEN — skipping LLM call")
sleep(5)
continue

response = call_llm(prompt, system=system)
validated = validate_output(response)

if not validated:
breaker.record_quality_failure()
logger.error(
f"Quality failure #{breaker.failures} — "
f"state: {breaker.state.value}"
)
else:
breaker.record_success()
process_response(response)

```

Key insight: When the circuit opens, stop. Don't burn tokens on a model producing garbage. Wait for the cooldown, send one probe request, and if it passes schema validation, close the circuit.

A natural extension is a model fallback chain: when the circuit opens, switch to a cheaper model with tighter constraints (lower temperature, stricter schema, fewer allowed tools). Circuit breakers tell you when to stop trusting a model; fallback chains tell you where to go next.

Pattern 2: Validation Gates Before Tool Execution

An agent mapped a delete_all_records action to what it interpreted as "cleanup." The API accepted it. 47 records gone before the next human review. The agent was confident. The action was syntactically valid. The intent was completely wrong.

The Rule

Never let an agent's output directly trigger a side effect. Always validate before execution.


```python
from typing import Any
from dataclasses import dataclass
import json

@dataclass
class ToolCallValidation:
tool_name: str
parameters: dict[str, Any]

ALLOWED_TOOLS = {"query_database", "send_notification", "update_record"}
DESTRUCTIVE_TOOLS = {"delete_record", "archive_project"}
MAX_DELETE_COUNT = 10

class ValidationGate:
def validate(self, call: ToolCallValidation) -> tuple[bool, str]:
"""Returns (is_valid, reason)"""

# 1. Schema: is this a known tool?
if call.tool_name not in ALLOWED_TOOLS | DESTRUCTIVE_TOOLS:
return False, f"Unknown tool: {call.tool_name}"

# 2. Sanity: does the action make sense?
if call.tool_name == "delete_record":
count = call.parameters.get("count", 1)
if count > MAX_DELETE_COUNT:
return False, (
f"Delete count {count} exceeds limit of {MAX_DELETE_COUNT}"
)

# 3. Boundary: is the agent in its allowed scope?
if call.tool_name == "query_database":
table = call.parameters.get("table")
if table == "production_billing":
return False, "Agent cannot access production_billing"

return True, "OK"

```

Three layers of validation, each catching a different failure class:

Schema validation — Is the output structurally correct? Missing required field? Wrong type? Malformed JSON?
Sanity checks — Does the action make sense? Deleting 10,000 records? Probably not.
Boundary enforcement — Is the agent operating within its allowed scope? Cross-tenant access? Targeting a production table from a staging workflow?

This gates every tool call before it executes. No separate validation function to remember to call — it's in the critical path.


```python
gate = ValidationGate()

def execute_tool_call(raw_llm_output: str):
call = parse_tool_call(raw_llm_output)
is_valid, reason = gate.validate(call)

if not is_valid:
logger.warning(f"Tool call blocked: {reason}")
return f"Action blocked: {reason}. Please revise."

# Only reach here after validation passes
return run_actual_tool(call)

```

Design principle: Constrain what the agent can do, and you prevent most errors at the source. This pairs directly with the insight from tool design work — splitting one monolithic tool into eight focused tools eliminates most validation failures before they can occur.

Pattern 3: Idempotent Sagas for Multi-Step Workflows

A three-step agent workflow failed on step 2. Step 1 had already created a customer record. The retry created a duplicate. Two hundred orphaned records found a week later, each triggering duplicate billing notifications.

The math is uncomfortable: an agent that succeeds 95% of the time on each step has only a 60% chance of completing a 10-step workflow cleanly (0.95^10). At 90% per step, a 10-step workflow succeeds just 35% of the time. Every step compounds the risk, and without idempotency, every retry doubles the side effects.

The Solution: Checkpoint-Then-Execute with Compensation Actions

Borrowing from the saga pattern in distributed systems, each step records its completion before execution and defines a compensation action for rollback.


```python
from dataclasses import dataclass
from enum import Enum
from typing import Callable, Optional
import sqlite3

class StepStatus(Enum):
PENDING = "pending"
COMPLETED = "completed"
COMPENSATED = "compensated"

@dataclass
class SagaStep:
name: str
execute: Callable
compensate: Optional[Callable]  # None means read-only or irreversible
status: StepStatus = StepStatus.PENDING

class SagaExecutor:
def __init__(self, db_path: str = ":memory:"):
self.conn = sqlite3.connect(db_path)
self._init_checkpoint_table()

def _init_checkpoint_table(self):
self.conn.execute("""
CREATE TABLE IF NOT EXISTS checkpoints (
step_name TEXT PRIMARY KEY,
status TEXT,
result TEXT
)
""")
self.conn.commit()

def _is_completed(self, step_name: str) -> bool:
row = self.conn.execute(
"SELECT status FROM checkpoints WHERE step_name = ?",
(step_name,)
).fetchone()
return row is not None and row[0] == "completed"

def _record_checkpoint(self, step_name: str, result: str):
self.conn.execute(
"INSERT OR REPLACE INTO checkpoints (step_name, status, result) "
"VALUES (?, 'completed', ?)",
(step_name, result)
)
self.conn.commit()

def execute(self, steps: list[SagaStep]) -> dict[str, str]:
completed_steps = []

for step in steps:
```
if self._is_completed(step.name):
# Idempotent: skip already-completed steps on retry
continue

try:
```python
result = step.execute()
self._record_checkpoint(step.name, str(result))
completed_steps.append(step)
except Exception as e:
# Rollback: execute compensation in reverse order
logger.error(
f"Step '{step.name}' failed: {e}. "
f"Rolling back {len(completed_steps)} steps."
)
for completed in reversed(completed_steps):
if completed.compensate:
try:
completed.compensate()
except Exception as comp_err:
logger.critical(
f"Compensation for '{completed.name}' failed: "
f"{comp_err}"
)
raise

return {s.name: "completed" for s in steps}


Usage:


def fetch_data(): return api.get_records()
def transform(data): return [process(r) for r in data]
def write_to_db(data):
db.insert_batch(data)
return {"id": db.last_insert_id}
def rollback_write(ctx):
db.delete_batch(ctx["id"])

def send_notification():
smtp.send("Pipeline complete")
def send_correction():
smtp.send("Correction: pipeline was rolled back")

saga = SagaExecutor()

steps = [
SagaStep("fetch_data", execute=fetch_data, compensate=None),  # Read-only
SagaStep("transform", execute=transform, compensate=None),    # Pure function
SagaStep("write_to_db", execute=write_to_db, compensate=rollback_write),
SagaStep("send_notification", execute=send_notification, compensate=send_correction),
]

saga.execute(steps)

```

Classify every step:

```python
Read-only — safe to retry freely (fetches, queries)
Pure function — safe to retry (transforms, computations)
Reversible — can undo (delete what you created)
Compensatable — can't undo but can correct (send a follow-up notification)
Irreversible — can't undo at all (payment processed). These need the most validation before execution.
Pattern 4: Budget Guardrails for Runaway Loops
```

Agents can enter reasoning loops. The model keeps calling tools, generating responses, re-evaluating — never reaching a terminal state. The tokens pile up. The bill climbs. Nothing crashes, which is the problem.

Token Budget

A hard cap on tokens per agent session. When the budget is exhausted, the agent must stop and either return results so far or escalate.


```python
@dataclass
class TokenBudget:
max_tokens: int = 50_000  # Set based on your cost tolerance
used_tokens: int = 0
cost_per_1k: float = 0.01  # Adjust for your model

def remaining(self) -> int:
return max(0, self.max_tokens - self.used_tokens)

def is_exhausted(self) -> bool:
return self.used_tokens >= self.max_tokens

def estimate_cost(self) -> float:
return (self.used_tokens / 1_000) * self.cost_per_1k

def track(self, response_text: str):
# Rough estimate: ~1 token per 4 chars for English
self.used_tokens += len(response_text) // 4

Cycle Budget
```

Beyond tokens, limit the number of reasoning cycles (turns) the agent can take. This prevents infinite tool-call loops even if each individual call is cheap.


```python
@dataclass
class CycleBudget:
max_cycles: int = 15
current_cycle: int = 0

def increment(self) -> bool:
"""Returns True if cycles remain, False if exhausted."""
self.current_cycle += 1
return self.current_cycle <= self.max_cycles

def remaining(self) -> int:
return max(0, self.max_cycles - self.current_cycle)

```

Integrated into an agent loop:


```python
token_budget = TokenBudget(max_tokens=30_000)
cycle_budget = CycleBudget(max_cycles=12)

while agent_running:
if token_budget.is_exhausted():
logger.warning(
f"Token budget exhausted ({token_budget.used_tokens} used). "
f"Estimated cost: ${token_budget.estimate_cost():.2f}"
)
# Return partial results or escalate
break

if not cycle_budget.increment():
logger.warning(
f"Cycle budget exhausted after {cycle_budget.max_cycles} turns. "
f"Agent may be in a reasoning loop."
)
break

# ... execute agent step ...
token_budget.track(llm_response_text)

if cycle_budget.remaining() <= 3:
logger.info(f"Agent in danger zone: {cycle_budget.remaining()} cycles left")

```

These budgets protect against the failure mode where an agent "works" perfectly — no exceptions, no crashes — while slowly burning through your API budget.

Pattern 5: Human Escalation for High-Risk Decisions

Some actions are too risky for an agent to make autonomously. Deleting production data. Sending customer-facing communications. Changing infrastructure configuration.

Confidence-Based Escalation

Ask the model for its confidence level alongside its reasoning. Below a threshold, route to a human instead of executing.


ESCALATION_THRESHOLD = 0.7
```python
HIGH_RISK_ACTIONS = {"delete_production_data", "send_customer_email",
"modify_infrastructure", "approve_payment"}

def should_escalate(tool_name: str, confidence: float,
validation_result: str) -> bool:
reasons = []

if tool_name in HIGH_RISK_ACTIONS:
reasons.append(f"high-risk action: {tool_name}")

if confidence < ESCALATION_THRESHOLD:
reasons.append(f"low confidence: {confidence:.2f}")

if "unsure" in validation_result.lower():
reasons.append("model expressed uncertainty")

if len(reasons) >= 2:
logger.warning(f"Escalating: {', '.join(reasons)}")
return True

return False
```

# In the agent loop:
if should_escalate(call.tool_name, parsed.confidence, validation_reason):
escalation_queue.put({
```python
"tool": call.tool_name,
"params": call.parameters,
"model_confidence": parsed.confidence,
"model_reasoning": parsed.reasoning,
"suggested_action": call.parameters,
})
response = "Action queued for human review."
else:
response = execute_tool_call(call)

The Escalation Queue
```

Persisted escalation means it survives crashes, restarts, and redeployments. A simple file-backed queue works:


```python
import json
from pathlib import Path

class EscalationQueue:
def __init__(self, path: str = "escalations.json"):
self.path = Path(path)
self._ensure_file()

def _ensure_file(self):
if not self.path.exists():
self.path.write_text("[]")

def enqueue(self, item: dict):
items = json.loads(self.path.read_text())
items.append({**item, "enqueued_at": time.time()})
self.path.write_text(json.dumps(items, indent=2))

def pending(self) -> list[dict]:
return json.loads(self.path.read_text())

def resolve(self, index: int, decision: str):
items = json.loads(self.path.read_text())
items[index]["decision"] = decision
items[index]["resolved_at"] = time.time()
self.path.write_text(json.dumps(items, indent=2))


Principle: The agent should stop deciding and start asking. Define clear escalation criteria upfront — action type, confidence threshold, validation failure patterns — and honor them. Nothing erodes trust in an agent faster than an autonomous mistake that a human would have caught in three seconds.

Putting It All Together
```

These five patterns form layers:


┌─────────────────────────────────────────┐
│ 5. Human Escalation                     │ ← When to stop deciding
│ 4. Budget Guardrails (tokens + cycles) │ ← When to stop spending
│ 3. Idempotent Sagas                     │ ← How to recover from partial failure
│ 2. Validation Gates                     │ ← What is allowed to execute
│ 1. Circuit Breakers (quality-aware)     │ ← When the model itself is unreliable
└─────────────────────────────────────────┘


Layer 1 catches model degradation before it burns tokens. Layer 2 blocks dangerous actions before they execute. Layer 3 contains partial failures when they inevitably occur. Layer 4 limits blast radius when the agent spirals. Layer 5 ensures humans are in the loop for irreversible decisions.

Together, they transform an agent from a probabilistic liability into a system you can deploy at 5 PM on a Friday and sleep through the night.

Actionable Takeaways
Start with circuit breakers on output quality, not just HTTP status. Validate schema conformance on every LLM response. Three consecutive failures → open circuit → switch to fallback model.
Gate every tool call. Schema, sanity, and boundary checks in the critical path — not as an afterthought.
Idempotence is non-negotiable for multi-step workflows. Checkpoint before execution, compensate on failure, skip completed steps on retry.
Set hard token and cycle budgets from day one. Cost surprises come from "working" agents in loops, not from crashed ones.
Define escalation criteria before writing agent code. If you can't state when a human should review an action, the agent isn't ready for production.

The production agent journey doesn't start with making the agent smarter. It starts with making the agent reliable. Do that, and the smartness will follow.

The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
5 AI Agent Error Handling Patterns That Keep Your Agent Running at 3 AM
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools