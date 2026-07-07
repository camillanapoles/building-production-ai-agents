# Event-Driven AI Agents: Patterns That Scale

> Artigo #12 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/event-driven-ai-agents-patterns-that-scale-3kf4)

---

Most AI agent tutorials teach you to build a chatbot that waits for user input. But production agents do not wait -- they react. A deploy finishes and your agent runs smoke tests. A customer signs up and your agent sends a personalized onboarding sequence. A monitoring threshold trips and your agent pages the on-call engineer before a human even notices.

The architecture that makes this possible is event-driven design. And getting it right is the difference between agents that demo well and agents that run your operations.

This guide covers four event-driven architecture patterns for AI agents, each with runnable Python code you can adapt today. No vendor lock-in, no Kafka required, no enterprise sales pitch -- just patterns that work.

Why Polling Fails for Production Agents

Before diving into patterns, let's be clear about why the default approach breaks down.

Polling is the naive solution: your agent checks a database, API, or inbox on a timer. "Any new emails? No? Check again in 30 seconds." It works in demos. It fails in production for three reasons:

Wasted compute. Your agent burns CPU and API quota checking for changes that have not happened. At scale, this adds up fast.
Latency floor. Your response time equals your polling interval. A 30-second poll means up to 30 seconds of delay on every event. For incident response, that is an eternity.
Quadratic connections. If N agents each poll M services, you have N x M connections. Add agents, and the system becomes unmanageable.

Event-driven architecture eliminates all three problems. Agents subscribe to event streams and react only when something actually happens. Connection complexity drops from O(N x M) to O(N + M). Latency drops to milliseconds. Compute is spent on real work, not checking.

Research from production deployments shows event-driven systems reduce AI agent response latency by 70-90% compared to polling approaches. That is not a theoretical improvement -- it is the difference between catching an outage in 200ms versus discovering it 30 seconds later.

Pattern 1: Event Queue with Worker Agents

The simplest event-driven pattern: events go into a queue, worker agents pull and process them. This is your starting point for any event-driven agent system.

When to Use It
Single-purpose agents that process one type of event
```python
Workloads where ordering matters (FIFO processing)
Systems where you need guaranteed delivery (no dropped events)
Implementation
```

Here is a minimal but production-ready implementation using Redis Streams (you could swap in RabbitMQ, SQS, or any message broker):


```python
import asyncio
import json
import redis.asyncio as redis
from datetime import datetime
from openai import AsyncOpenAI

client = AsyncOpenAI()
rdb = redis.Redis(host="localhost", port=6379, decode_responses=True)

STREAM = "agent:events"
GROUP = "agent-workers"
CONSUMER = "worker-1"

async def ensure_group():
"""Create consumer group if it does not exist."""
try:
await rdb.xgroup_create(STREAM, GROUP, id="0", mkstream=True)
except redis.ResponseError as e:
if "BUSYGROUP" not in str(e):
raise

async def publish_event(event_type: str, payload: dict):
"""Publish an event to the stream."""
event = {
"type": event_type,
"payload": json.dumps(payload),
"timestamp": datetime.utcnow().isoformat(),
}
event_id = await rdb.xadd(STREAM, event)
print(f"Published {event_type} -> {event_id}")
return event_id

async def process_event(event_id: str, event: dict):
"""Route event to the appropriate AI handler."""
event_type = event["type"]
payload = json.loads(event["payload"])

handlers = {
"deploy.completed": handle_deploy,
"alert.triggered": handle_alert,
"email.received": handle_email,
}

handler = handlers.get(event_type)
if handler:
await handler(payload)
else:
print(f"No handler for event type: {event_type}")

# Acknowledge the event so it is not redelivered
await rdb.xack(STREAM, GROUP, event_id)

async def handle_deploy(payload: dict):
"""AI agent handles deployment verification."""
response = await client.chat.completions.create(
model="gpt-4o",
messages=[
{"role": "system", "content": "You are a deployment verification agent."},
{"role": "user", "content": f"Verify this deployment: {json.dumps(payload)}"}
],
)
print(f"Deploy check: {response.choices[0].message.content[:100]}")

async def handle_alert(payload: dict):
"""AI agent triages monitoring alerts."""
response = await client.chat.completions.create(
model="gpt-4o",
messages=[
{"role": "system", "content": "You are an incident triage agent. Classify severity and suggest next steps."},
{"role": "user", "content": f"Alert: {json.dumps(payload)}"}
],
)
print(f"Alert triage: {response.choices[0].message.content[:100]}")

async def worker_loop():
"""Main worker loop: pull events, process, repeat."""
await ensure_group()
print(f"Worker {CONSUMER} listening on {STREAM}...")

while True:
# Block for up to 5 seconds waiting for new events
messages = await rdb.xreadgroup(
GROUP, CONSUMER, {STREAM: ">"}, count=1, block=5000
)
for stream_name, events in messages:
for event_id, event_data in events:
try:
await process_event(event_id, event_data)
except Exception as e:
print(f"Error processing {event_id}: {e}")
# Event stays unacknowledged -> will be reclaimed

if __name__ == "__main__":
asyncio.run(worker_loop())

```

This gives you several production features out of the box:

Consumer groups: Multiple worker agents split the load. Add workers to scale horizontally.
Acknowledgments: Events are not removed until explicitly acknowledged. If a worker crashes, unacknowledged events get redelivered.
Backpressure: Workers pull events at their own pace. A spike in events queues up instead of overwhelming your agents.
Scaling It

To add more workers, just run additional instances with different CONSUMER names. Redis handles partition assignment automatically. Three workers processing deploy events? Each picks up roughly one-third of the load with zero configuration changes.

Pattern 2: Fan-Out for Parallel Agent Processing

Sometimes a single event needs to trigger multiple agents simultaneously. A new customer signs up and you need to: send a welcome email, provision their account, update the CRM, and notify the sales team. Sequentially, that takes 4x as long as it should.

Fan-out solves this by broadcasting one event to N subscribers, each processing in parallel.

Implementation

Redis Pub/Sub is the simplest fan-out mechanism. For durability, you would use multiple consumer groups on the same Redis Stream (each group gets every message independently).


```python
import asyncio
import json
import redis.asyncio as redis

rdb = redis.Redis(host="localhost", port=6379, decode_responses=True)
STREAM = "events:customer"

async def setup_fan_out():
"""Create independent consumer groups for parallel processing."""
groups = ["email-agent", "provisioning-agent", "crm-agent", "sales-notifier"]
for group in groups:
try:
await rdb.xgroup_create(STREAM, group, id="0", mkstream=True)
except redis.ResponseError:
pass  # Group already exists

async def agent_worker(group_name: str, handler):
"""Generic agent worker that processes events for its group."""
consumer = f"{group_name}-1"
print(f"[{group_name}] Listening...")

while True:
messages = await rdb.xreadgroup(
group_name, consumer, {STREAM: ">"}, count=1, block=5000
)
for _, events in messages:
for event_id, data in events:
payload = json.loads(data.get("payload", "{}"))
try:
await handler(payload)
await rdb.xack(STREAM, group_name, event_id)
except Exception as e:
print(f"[{group_name}] Failed: {e}")

async def email_handler(payload):
print(f"[email] Sending welcome to {payload.get('email')}")
await asyncio.sleep(0.5)  # Simulate API call

async def provision_handler(payload):
print(f"[provision] Creating workspace for {payload.get('user_id')}")
await asyncio.sleep(1.0)

async def crm_handler(payload):
print(f"[crm] Adding {payload.get('email')} to CRM")
await asyncio.sleep(0.3)

async def sales_handler(payload):
print(f"[sales] Notifying team about {payload.get('plan')} signup")
await asyncio.sleep(0.2)

async def main():
await setup_fan_out()
# All agents run in parallel, each gets every event independently
await asyncio.gather(
agent_worker("email-agent", email_handler),
agent_worker("provisioning-agent", provision_handler),
agent_worker("crm-agent", crm_handler),
agent_worker("sales-notifier", sales_handler),
)

if __name__ == "__main__":
asyncio.run(main())

```

The key insight: each consumer group maintains its own read cursor. Publishing one customer.signup event means all four agents process it independently. If the email agent is slow, it does not block the provisioning agent. If the CRM agent crashes, it resumes from its last acknowledged event without affecting the others.

When Fan-Out Gets Tricky

Fan-out is powerful but introduces a coordination challenge: what happens when multiple agents need to complete before a final action? For example, you want to send a "setup complete" email only after provisioning AND CRM updates both finish.

The cleanest solution is the completion event pattern: each agent emits a completion event when done. A coordinator agent subscribes to all completion events and triggers the final action when all prerequisites are met.


```python
# Each agent emits when done:
await publish_event("provision.completed", {"user_id": uid})
await publish_event("crm.updated", {"user_id": uid})

# Coordinator checks if all steps are complete:
async def coordinator(payload):
user_id = payload["user_id"]
status = await rdb.hgetall(f"onboarding:{user_id}")
if status.get("provisioned") and status.get("crm_updated"):
await publish_event("onboarding.complete", {"user_id": user_id})

Pattern 3: Event Sourcing for Auditable Agent Decisions
```

In regulated industries or high-stakes applications, you need to know exactly what your agent did and why. Event sourcing records every state change as an immutable event, creating a complete audit trail that you can replay to reconstruct any past state.

This matters for AI agents because LLM outputs are stochastic. The same input can produce different outputs. When a customer asks "why did your agent reject my application?", you need to show the exact inputs, the exact model output, and the exact decision logic -- not a best guess.

Implementation
```python
import json
import hashlib
from dataclasses import dataclass, asdict
from datetime import datetime
from typing import Any

@dataclass
class AgentEvent:
event_id: str
event_type: str
agent_id: str
timestamp: str
payload: dict
parent_event_id: str | None = None  # For causal chains
checksum: str = ""

def __post_init__(self):
if not self.checksum:
content = json.dumps(
{"type": self.event_type, "payload": self.payload,
"agent": self.agent_id, "ts": self.timestamp},
sort_keys=True,
)
self.checksum = hashlib.sha256(content.encode()).hexdigest()[:16]

class EventStore:
"""Append-only event store for agent decision auditing."""

def __init__(self):
self._events: list[AgentEvent] = []

def append(self, event: AgentEvent):
self._events.append(event)

def get_agent_history(self, agent_id: str) -> list[AgentEvent]:
return [e for e in self._events if e.agent_id == agent_id]

def get_causal_chain(self, event_id: str) -> list[AgentEvent]:
"""Trace the full decision chain for a given event."""
chain = []
current_id = event_id
while current_id:
event = next((e for e in self._events if e.event_id == current_id), None)
if not event:
break
chain.append(event)
current_id = event.parent_event_id
return list(reversed(chain))

def replay_to(self, timestamp: str) -> list[AgentEvent]:
"""Get all events up to a point in time for state reconstruction."""
return [e for e in self._events if e.timestamp <= timestamp]

# Usage: recording an agent's loan decision
store = EventStore()

# Step 1: Application received
store.append(AgentEvent(
event_id="evt_001",
event_type="application.received",
agent_id="loan-reviewer",
timestamp=datetime.utcnow().isoformat(),
payload={"applicant": "user_42", "amount": 50000},
))

# Step 2: Agent analyzed credit data
store.append(AgentEvent(
event_id="evt_002",
event_type="credit.analyzed",
agent_id="loan-reviewer",
timestamp=datetime.utcnow().isoformat(),
payload={"score": 720, "risk_level": "medium", "model": "gpt-4o",
"prompt_hash": "a3f2c1d8", "raw_output": "Applicant shows..."},
parent_event_id="evt_001",
))

# Step 3: Decision made
store.append(AgentEvent(
event_id="evt_003",
event_type="application.approved",
agent_id="loan-reviewer",
timestamp=datetime.utcnow().isoformat(),
payload={"decision": "approved", "conditions": ["income_verification"]},
parent_event_id="evt_002",
))

# Audit: trace how the decision was made
chain = store.get_causal_chain("evt_003")
for event in chain:
print(f"{event.event_type}: {event.payload}")

```

The parent_event_id field creates a causal chain. Every agent decision links back to the event that triggered it. When auditors ask "how did the agent decide to approve this loan?", you walk the chain: application received -> credit analyzed (with exact model, prompt, and output) -> decision made.

Checksums for Tamper Detection

Notice the checksum field. Each event gets a SHA-256 hash of its content. If anyone modifies an event after the fact, the checksum will not match. This is essential for compliance in finance, healthcare, and legal applications where you need to prove the audit trail has not been altered.

Pattern 4: Saga Orchestration for Multi-Step Workflows

Real-world agent workflows span multiple steps, multiple services, and sometimes multiple days. A saga coordinates these long-running workflows and -- critically -- handles failures with compensating actions that undo partial work.

Consider an e-commerce fulfillment agent: it needs to charge the card, reserve inventory, schedule shipping, and send confirmation. If shipping fails, you need to release the inventory and refund the card. Without saga orchestration, partial failures leave your system in an inconsistent state.

Implementation
```python
import asyncio
from dataclasses import dataclass, field
from enum import Enum
from typing import Callable, Any

class StepStatus(Enum):
PENDING = "pending"
RUNNING = "running"
COMPLETED = "completed"
FAILED = "failed"
COMPENSATED = "compensated"

@dataclass
class SagaStep:
name: str
execute: Callable
compensate: Callable  # Undo action if later steps fail
status: StepStatus = StepStatus.PENDING
result: Any = None
error: str = ""

@dataclass
class Saga:
name: str
steps: list[SagaStep] = field(default_factory=list)
context: dict = field(default_factory=dict)

async def run(self) -> bool:
"""Execute all steps. On failure, compensate completed steps."""
completed: list[SagaStep] = []

for step in self.steps:
step.status = StepStatus.RUNNING
try:
step.result = await step.execute(self.context)
step.status = StepStatus.COMPLETED
completed.append(step)
print(f"  [ok] {step.name}")
except Exception as e:
step.status = StepStatus.FAILED
step.error = str(e)
print(f"  [FAIL] {step.name}: {e}")
# Compensate all completed steps in reverse order
await self._compensate(completed)
return False

return True

async def _compensate(self, completed: list[SagaStep]):
"""Undo completed steps in reverse order."""
print("  Rolling back...")
for step in reversed(completed):
try:
await step.compensate(self.context)
step.status = StepStatus.COMPENSATED
print(f"  [undo] {step.name}")
except Exception as e:
print(f"  [undo-FAIL] {step.name}: {e}")
# Log for manual intervention

# Define the workflow steps
async def charge_card(ctx):
# Call payment API
ctx["charge_id"] = "ch_abc123"
return {"charged": 99.99}

async def refund_card(ctx):
print(f"    Refunding charge {ctx.get('charge_id')}")

async def reserve_inventory(ctx):
ctx["reservation_id"] = "res_xyz"
return {"reserved": True}

async def release_inventory(ctx):
print(f"    Releasing reservation {ctx.get('reservation_id')}")

async def schedule_shipping(ctx):
# Simulate a failure
raise Exception("Carrier API timeout")

async def cancel_shipping(ctx):
print("    Cancelling shipping request")

async def main():
saga = Saga(
name="order-fulfillment",
steps=[
SagaStep("charge_card", charge_card, refund_card),
SagaStep("reserve_inventory", reserve_inventory, release_inventory),
SagaStep("schedule_shipping", schedule_shipping, cancel_shipping),
],
context={"order_id": "order_789"},
)

print(f"Running saga: {saga.name}")
success = await saga.run()
print(f"Result: {'Success' if success else 'Rolled back'}")

if __name__ == "__main__":
asyncio.run(main())

```

Output when shipping fails:


Running saga: order-fulfillment
[ok] charge_card
[ok] reserve_inventory
[FAIL] schedule_shipping: Carrier API timeout
Rolling back...
[undo] reserve_inventory
Releasing reservation res_xyz
[undo] charge_card
Refunding charge ch_abc123
Result: Rolled back


The saga pattern is essential for AI agents that interact with external services. LLM calls can timeout, APIs can return errors, and rate limits can hit at any point. Without compensating actions, every failure leaves your system in an unknown state.

Production Hardening: Retry, Dead Letters, and Observability

The four patterns above give you the architecture. But production systems need three more pieces to be reliable.

Retry with Exponential Backoff

Transient failures are common -- network blips, rate limits, cold starts. Retrying with exponential backoff handles them gracefully:


```python
import asyncio
import random

async def retry_with_backoff(fn, max_retries=3, base_delay=1.0):
"""Retry a function with exponential backoff and jitter."""
for attempt in range(max_retries + 1):
try:
return await fn()
except Exception as e:
if attempt == max_retries:
raise  # Final attempt failed, propagate
delay = base_delay * (2 ** attempt) + random.uniform(0, 0.5)
print(f"Retry {attempt + 1}/{max_retries} in {delay:.1f}s: {e}")
await asyncio.sleep(delay)

```

The jitter (random addition) prevents thundering herd problems when multiple agents retry simultaneously against the same service.

Dead Letter Queues

When retries are exhausted, events go to a dead letter queue (DLQ) instead of being dropped silently. This gives you a safety net for manual investigation:


```python
DLQ_STREAM = "agent:dead-letters"

async def send_to_dlq(event_id: str, event: dict, error: str):
"""Move a failed event to the dead letter queue."""
await rdb.xadd(DLQ_STREAM, {
"original_event_id": event_id,
"original_stream": STREAM,
"event_data": json.dumps(event),
"error": error,
"failed_at": datetime.utcnow().isoformat(),
"retry_count": "3",
})
# Acknowledge the original event so it stops being redelivered
await rdb.xack(STREAM, GROUP, event_id)

```

Check your DLQ daily. Patterns in dead-lettered events reveal systemic issues: if the same event type keeps failing, you have a bug, not a transient error.

Structured Logging for Agent Observability

Event-driven systems are harder to debug than request-response systems because there is no single request thread to follow. Structured logging with correlation IDs solves this:


```python
import structlog

log = structlog.get_logger()

async def process_event(event_id: str, event: dict):
logger = log.bind(
event_id=event_id,
event_type=event["type"],
correlation_id=event.get("correlation_id", event_id),
)
logger.info("event.received")

try:
result = await handle(event)
logger.info("event.processed", result=result)
except Exception as e:
logger.error("event.failed", error=str(e))
raise

```

The correlation_id follows an event through the entire fan-out chain. When four agents process the same customer signup, you can filter logs by correlation ID to see the complete picture.

Choosing the Right Pattern

Here is a quick reference for matching problems to patterns:

Scenario	Pattern	Why
Process events one at a time	Queue + Workers	Simple, ordered, scalable
One event triggers multiple agents	Fan-Out	Parallel, independent processing
Compliance or audit requirements	Event Sourcing	Immutable, replayable trail
Multi-step workflows with rollback	Saga	Compensating actions on failure
High-throughput with mixed needs	Combine patterns	Queue for ingestion, fan-out for distribution

In practice, production systems combine patterns. You might use a queue for ingestion, fan-out for distribution to specialized agents, event sourcing for the audit trail, and sagas for workflows that span external services.

Platforms like Nebula handle much of this infrastructure for you -- event-driven triggers, automatic retries, and multi-agent coordination are built into the agent runtime, so you can focus on the agent logic rather than the plumbing. But understanding the patterns helps you debug issues and make better architectural decisions regardless of what platform you use.

What to Build Next

If you are building event-driven agents today, start with Pattern 1 (queue + workers) for your most common event type. Get the basics working: publish events, consume them, acknowledge them, handle failures.

Then add fan-out when you need parallel processing, event sourcing when you need auditability, and sagas when your workflows span multiple services.

The patterns in this guide are framework-agnostic by design. Whether you are using LangGraph, CrewAI, PydanticAI, or raw API calls, the event-driven architecture layer sits beneath your agent framework. It is the foundation that makes everything else reliable.

The best agents are not the ones with the smartest prompts. They are the ones that never drop an event, never leave a workflow half-finished, and never lose track of what they did and why.

This is Part 5 of the Building Production AI Agents series. Previous: Event-Driven AI Agent Architecture Patterns.

The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
Event-Driven AI Agents: Patterns That Scale
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools