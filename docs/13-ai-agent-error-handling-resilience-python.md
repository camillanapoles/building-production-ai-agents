# AI Agent Error Handling: 4 Resilience Patterns in Python

> Artigo #13 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/ai-agent-error-handling-4-resilience-patterns-in-python-2gl5)

---

Your AI agent works flawlessly in development. Then it hits production, OpenAI returns a 429, your fallback prompt throws a validation error, and the entire pipeline crashes at 2 AM with nobody watching.

This is not a testing problem. It is an AI agent error handling problem. LLM APIs fail in ways traditional software never does -- rate limits, non-deterministic outputs, content policy rejections, and context window overflows are not edge cases. They are daily operational realities at any meaningful scale.

This guide covers four battle-tested resilience patterns -- retry with backoff, model fallback chains, circuit breakers, and graceful degradation -- with pure Python implementations you can drop into any project. No framework lock-in, no heavy dependencies.

Why AI Agents Fail Differently Than Traditional Software

Traditional APIs fail predictably. A database is down, you get a connection error. An auth token expires, you get a 401. You can write deterministic tests for these.

LLM-powered agents introduce a fundamentally different failure model:

Rate limits (429) hit unpredictably based on tokens-per-minute quotas that fluctuate with provider load
Context window overflow happens silently as your agent accumulates tool results and conversation history
Content policy rejections vary between providers and trigger on inputs you never anticipated
Response format drift occurs when providers update models -- your perfectly structured JSON prompt returns subtly different output
Partial or malformed responses break downstream parsing without throwing obvious errors

The critical insight: these failures are not bugs to eliminate. They are operational realities to engineer around. Every production AI agent needs a resilience layer between its business logic and the LLM APIs it depends on.

Here are the four patterns that provide that layer.

Pattern 1: Smart Retry with Exponential Backoff

Retries are your first line of defense against transient failures. But naive retries on LLM APIs are dangerous -- they amplify failures, waste tokens, and can drain your budget during an outage.

The key principle: not all errors deserve a retry. Retrying a permanent failure (bad API key, malformed request) wastes time and money. Failing fast on a transient error (rate limit, timeout) loses a request that would have succeeded on the second try.

Start by classifying errors:


from enum import Enum

class ErrorType(Enum):
TRANSIENT = "transient"    # Retry with backoff
PERMANENT = "permanent"    # Fail immediately
DEGRADED = "degraded"      # Switch to fallback

def classify_error(error: Exception) -> ErrorType:
"""Classify an LLM API error to determine recovery strategy."""
error_str = str(error).lower()
status = getattr(error, 'status_code', None)

# Transient: retry with backoff
if status in (429, 500, 502, 503) or 'timeout' in error_str:
return ErrorType.TRANSIENT

# Degraded: switch to fallback model
if 'context_length' in error_str or 'content_filter' in error_str:
return ErrorType.DEGRADED

# Permanent: fail immediately
return ErrorType.PERMANENT


Now build the retry logic. The implementation uses exponential backoff with jitter -- the jitter prevents the "thundering herd" problem where multiple agent instances all retry at exactly the same intervals after a shared rate limit:


import time
import random
import logging

logger = logging.getLogger(__name__)

def retry_with_backoff(
func,
max_retries: int = 3,
base_delay: float = 1.0,
max_delay: float = 60.0,
jitter: float = 1.0,
):
"""Retry a function with exponential backoff and jitter.

Only retries on transient errors. Permanent errors fail immediately.
Degraded errors are re-raised for the fallback layer to handle.
"""
last_exception = None

for attempt in range(max_retries + 1):
try:
return func()
except Exception as e:
last_exception = e
error_type = classify_error(e)

if error_type == ErrorType.PERMANENT:
logger.error(f"Permanent error, not retrying: {e}")
raise

if error_type == ErrorType.DEGRADED:
logger.warning(f"Degraded error, passing to fallback: {e}")
raise

if attempt == max_retries:
logger.error(f"All {max_retries} retries exhausted: {e}")
raise

# Exponential backoff: 1s, 2s, 4s... capped at max_delay
delay = min(base_delay * (2 ** attempt), max_delay)
# Add random jitter to prevent thundering herd
delay += random.uniform(0, jitter)

logger.warning(
f"Transient error (attempt {attempt + 1}/{max_retries}): {e}. "
f"Retrying in {delay:.1f}s"
)
time.sleep(delay)

raise last_exception


Usage with any LLM provider:


import openai

client = openai.OpenAI()

def call_llm():
return client.chat.completions.create(
model="gpt-4o",
messages=[{"role": "user", "content": "Explain circuit breakers"}],
timeout=30,
)

# Retries transient errors up to 3 times with backoff
response = retry_with_backoff(call_llm, max_retries=3)


Two details that matter in production:

Always set a timeout on LLM calls. A request that hangs for 5 minutes during a retry cycle blocks your entire agent pipeline. 30 seconds is a reasonable default.
Track token spend across retries. Three retries of a 4K-token prompt cost 12K tokens. Add a budget cap if your agent runs autonomously.
Pattern 2: Model Fallback Chains

Retries handle transient failures within a single provider. But what happens when the provider itself is down, or when a content policy rejection is provider-specific, or when you need a model with a larger context window?

Fallback chains route requests to alternative models automatically when the primary fails:


from dataclasses import dataclass
from typing import Callable, Any

@dataclass
class ModelConfig:
name: str
call_fn: Callable
cost_per_1k_tokens: float  # Track cost at each tier

class FallbackChain:
"""Routes LLM requests through a prioritized chain of models.

Each model gets retry_with_backoff protection. If retries exhaust,
the chain moves to the next model.
"""

def __init__(self, models: list[ModelConfig], max_retries: int = 2):
self.models = models
self.max_retries = max_retries

def call(self, messages: list[dict], **kwargs) -> dict:
errors = []

for i, model in enumerate(self.models):
try:
result = retry_with_backoff(
lambda m=model: m.call_fn(messages, **kwargs),
max_retries=self.max_retries,
)
if i > 0:
logger.info(
f"Fallback succeeded: {model.name} "
f"(after {i} failed model(s))"
)
return {
"content": self._extract_content(result, model.name),
"model": model.name,
"fallback_used": i > 0,
}
except Exception as e:
errors.append({"model": model.name, "error": str(e)})
logger.warning(f"Model {model.name} failed: {e}")
# Permanent errors (auth, bad request) should not fall through
error_type = classify_error(e)
if error_type == ErrorType.PERMANENT:
raise
continue

raise RuntimeError(f"All {len(self.models)} models failed: {errors}")

def _extract_content(self, result, model_name: str) -> str:
"""Normalize response format across providers."""
# OpenAI format
if hasattr(result, 'choices'):
return result.choices[0].message.content
# Anthropic format
if hasattr(result, 'content'):
return result.content[0].text
# Dict format
if isinstance(result, dict):
return result.get('content', str(result))
return str(result)


Set up a practical fallback chain:


import openai
import anthropic

oai = openai.OpenAI()
anth = anthropic.Anthropic()

def call_gpt4o(messages, **kwargs):
return oai.chat.completions.create(
model="gpt-4o", messages=messages, timeout=30, **kwargs
)

def call_claude_sonnet(messages, **kwargs):
system = next((m["content"] for m in messages if m["role"] == "system"), "")
user_msgs = [m for m in messages if m["role"] != "system"]
return anth.messages.create(
model="claude-sonnet-4-20250514", system=system,
messages=user_msgs, max_tokens=4096, timeout=30,
)

def call_gpt4o_mini(messages, **kwargs):
return oai.chat.completions.create(
model="gpt-4o-mini", messages=messages, timeout=30, **kwargs
)

chain = FallbackChain([
ModelConfig("gpt-4o", call_gpt4o, cost_per_1k_tokens=0.005),
ModelConfig("claude-sonnet", call_claude_sonnet, cost_per_1k_tokens=0.003),
ModelConfig("gpt-4o-mini", call_gpt4o_mini, cost_per_1k_tokens=0.00015),
])

# Automatically falls through: GPT-4o -> Claude -> GPT-4o-mini
result = chain.call([{"role": "user", "content": "Analyze this data..."}])
print(f"Answered by: {result['model']}, fallback: {result['fallback_used']}")


The fallback order matters. Organize by: quality first, then different provider, then cost-optimized. If GPT-4o is rate-limited, Claude Sonnet (different provider) will likely succeed. GPT-4o-mini is the last resort -- cheaper, faster, lower quality, but always available.

One design decision worth highlighting: the FallbackChain wraps each model call in retry_with_backoff. This means each model gets its own retry attempts before the chain moves on. Retries handle transient blips; fallbacks handle sustained outages.

Pattern 3: Circuit Breaker for Tool Calls

Retries and fallbacks handle individual request failures. Circuit breakers solve a different problem: what happens when a provider or tool is down for 10 minutes and every request in your system wastes 30 seconds retrying before failing?

Without a circuit breaker, a flaky external API turns every agent request into a slow failure. Your users wait, your token budget burns, and the struggling provider gets hammered with retry traffic that prevents recovery.

A circuit breaker monitors failure rates and "trips" when they exceed a threshold, immediately rejecting requests instead of attempting them:


import time
import threading

class CircuitBreaker:
"""Prevents cascading failures by fast-failing when a service is down.

States:
CLOSED  - Normal operation, requests pass through
OPEN    - Service is down, requests fail immediately
HALF_OPEN - Testing if service recovered (one probe request)
"""

def __init__(
self,
name: str,
failure_threshold: int = 5,
reset_timeout: float = 60.0,
success_threshold: int = 2,
):
self.name = name
self.failure_threshold = failure_threshold
self.reset_timeout = reset_timeout
self.success_threshold = success_threshold

self._state = "CLOSED"
self._failure_count = 0
self._success_count = 0
self._last_failure_time = 0.0
self._lock = threading.Lock()

@property
def state(self) -> str:
with self._lock:
if self._state == "OPEN":
# Check if reset timeout has elapsed
if time.time() - self._last_failure_time >= self.reset_timeout:
self._state = "HALF_OPEN"
self._success_count = 0
return self._state

def call(self, func, *args, **kwargs):
"""Execute function through circuit breaker protection."""
current_state = self.state

if current_state == "OPEN":
raise CircuitOpenError(
f"Circuit '{self.name}' is OPEN. "
f"Service unavailable, retrying in "
f"{self.reset_timeout - (time.time() - self._last_failure_time):.0f}s"
)

try:
result = func(*args, **kwargs)
self._on_success()
return result
except Exception as e:
self._on_failure()
raise

def _on_success(self):
with self._lock:
if self._state == "HALF_OPEN":
self._success_count += 1
if self._success_count >= self.success_threshold:
self._state = "CLOSED"
self._failure_count = 0
logger.info(f"Circuit '{self.name}' CLOSED (recovered)")
else:
self._failure_count = 0

def _on_failure(self):
with self._lock:
self._failure_count += 1
self._last_failure_time = time.time()

if self._state == "HALF_OPEN":
self._state = "OPEN"
logger.warning(f"Circuit '{self.name}' re-OPENED (probe failed)")
elif self._failure_count >= self.failure_threshold:
self._state = "OPEN"
logger.warning(
f"Circuit '{self.name}' OPENED "
f"after {self._failure_count} consecutive failures"
)


class CircuitOpenError(Exception):
"""Raised when a circuit breaker is open."""
pass


The state machine is simple but powerful:


CLOSED (normal)     -- failures hit threshold -->  OPEN (fast-fail)
|
timeout expires
|
HALF_OPEN (probe)
/        \
success       failure
/              \
CLOSED             OPEN


Use a separate circuit breaker for each external dependency:


# One breaker per service -- never share across providers
openai_breaker = CircuitBreaker("openai", failure_threshold=5, reset_timeout=60)
search_breaker = CircuitBreaker("web-search", failure_threshold=3, reset_timeout=30)
db_breaker = CircuitBreaker("database", failure_threshold=3, reset_timeout=45)

def agent_search(query: str) -> list[dict]:
"""Agent tool: web search with circuit breaker protection."""
try:
return search_breaker.call(web_search_api, query)
except CircuitOpenError:
logger.warning("Search unavailable, using cached results")
return get_cached_results(query)
except Exception:
return []  # Graceful degradation: empty results, not a crash


The critical detail: one breaker per external dependency. If OpenAI is down, you do not want the breaker to block Anthropic calls too. And the success_threshold=2 parameter prevents a single lucky request from restoring full traffic to an unstable service.

Pattern 4: Graceful Degradation

Sometimes everything fails. Your primary model is rate-limited, the fallback provider is down, and the circuit breaker is open. Traditional error handling crashes. Graceful degradation delivers something useful instead of nothing.

The principle: users tolerate reduced capability far more than they tolerate crashes or hung requests.


from dataclasses import dataclass
from typing import Optional

@dataclass
class AgentResponse:
content: str
quality_tier: str    # "full", "reduced", "cached", "static"
model_used: str
warning: Optional[str] = None

class ResilientAgent:
"""Agent with tiered degradation: full -> reduced -> cached -> static."""

def __init__(self, fallback_chain: FallbackChain, cache: dict = None):
self.chain = fallback_chain
self.cache = cache or {}

def run(self, messages: list[dict]) -> AgentResponse:
# Tier 1: Full capability via fallback chain
try:
result = self.chain.call(messages)
# Cache successful responses for future degradation
cache_key = messages[-1]["content"][:100]
self.cache[cache_key] = result["content"]
return AgentResponse(
content=result["content"],
quality_tier="full" if not result["fallback_used"] else "reduced",
model_used=result["model"],
)
except RuntimeError:
pass  # All models failed

# Tier 2: Cached response from similar previous query
cache_key = messages[-1]["content"][:100]
if cache_key in self.cache:
return AgentResponse(
content=self.cache[cache_key],
quality_tier="cached",
model_used="cache",
warning="This response is from cache and may be outdated.",
)

# Tier 3: Static fallback -- honest about limitations
return AgentResponse(
content=(
"I'm experiencing temporary difficulties connecting to AI services. "
"Please try again in a few minutes. If this persists, check "
"https://status.openai.com for provider status."
),
quality_tier="static",
model_used="none",
warning="All AI services are currently unavailable.",
)


The quality_tier field is important for downstream logic. Your application can make decisions based on response quality:


agent = ResilientAgent(chain)
response = agent.run([{"role": "user", "content": "Summarize today's metrics"}])

if response.quality_tier == "static":
# Don't send automated reports with static fallback content
notify_ops_team("Agent degraded, manual review needed")
elif response.quality_tier == "cached":
# Send the report but flag it
send_report(response.content, caveat="Based on cached data")
else:
send_report(response.content)

Putting It All Together: A Resilient Agent Pipeline

The real power comes from composing all four patterns into a layered defense. Here is the execution order from outermost to innermost:


Your Agent Logic
|
Graceful Degradation (always returns something)
|
Fallback Chain (tries alternative models)
|
Circuit Breaker (fast-fails during outages)
|
Retry with Backoff (handles transient errors)
|
LLM Provider API


Here is a complete, working pipeline that wires everything together:


def build_resilient_agent() -> ResilientAgent:
"""Build an agent with all four resilience patterns composed."""

# Layer 1: Circuit breakers per provider
oai_breaker = CircuitBreaker("openai", failure_threshold=5, reset_timeout=60)
anth_breaker = CircuitBreaker("anthropic", failure_threshold=5, reset_timeout=60)

# Layer 2: Provider calls wrapped with circuit breakers
oai_client = openai.OpenAI()
anth_client = anthropic.Anthropic()

def gpt4o_with_breaker(messages, **kwargs):
return oai_breaker.call(
lambda: oai_client.chat.completions.create(
model="gpt-4o", messages=messages, timeout=30, **kwargs
)
)

def claude_with_breaker(messages, **kwargs):
system = next((m["content"] for m in messages if m["role"] == "system"), "")
user_msgs = [m for m in messages if m["role"] != "system"]
return anth_breaker.call(
lambda: anth_client.messages.create(
model="claude-sonnet-4-20250514", system=system,
messages=user_msgs, max_tokens=4096, timeout=30,
)
)

def gpt4o_mini_with_breaker(messages, **kwargs):
return oai_breaker.call(
lambda: oai_client.chat.completions.create(
model="gpt-4o-mini", messages=messages, timeout=30, **kwargs
)
)

# Layer 3: Fallback chain with retry built in
chain = FallbackChain(
models=[
ModelConfig("gpt-4o", gpt4o_with_breaker, 0.005),
ModelConfig("claude-sonnet", claude_with_breaker, 0.003),
ModelConfig("gpt-4o-mini", gpt4o_mini_with_breaker, 0.00015),
],
max_retries=2,
)

# Layer 4: Graceful degradation wraps everything
return ResilientAgent(chain)


# Usage
agent = build_resilient_agent()
response = agent.run([
{"role": "system", "content": "You are a helpful assistant."},
{"role": "user", "content": "What are the key trends in AI this week?"},
])

print(f"Quality: {response.quality_tier}")
print(f"Model: {response.model_used}")
print(f"Response: {response.content[:200]}")


Notice how the layers compose: retry happens inside the circuit breaker, which happens inside the fallback chain, which happens inside the degradation wrapper. If retries exhaust their attempts, the circuit breaker records a failure. After enough failures, the circuit opens and the fallback chain skips that provider entirely -- no retries, no waiting.

Quick Reference: When to Use Each Pattern
Pattern	Best For	Avoid When
Exponential Backoff	Rate limits, transient 5xx errors	Permanent failures (auth, bad request)
Model Fallback	Provider outages, cost optimization	Task needs a specific model's capabilities
Circuit Breaker	Flaky external APIs, sustained outages	Internal computations that don't call external services
Graceful Degradation	Multi-source tasks, user-facing agents	Binary success/fail operations (payments, writes)
Key Metrics to Track

Resilience patterns are only as good as your ability to observe them. Track these in production:

Retry rate per provider -- Spike above 20%? Something is degraded upstream. Set alerts.
Fallback activation rate -- If your primary model fails more than 10% of the time, reconsider your provider choice.
Circuit breaker state changes -- Every OPEN/CLOSE transition should trigger an alert. Frequent cycling means an unstable dependency.
Degradation tier distribution -- What percentage of responses are served from cache or static fallback? This is the real quality metric your users experience.
Cost per successful request -- Fallbacks to more expensive models inflate costs. Track this to catch budget overruns before they become a problem.
Wrapping Up

Production AI agents need resilience as a first-class architectural concern, not an afterthought bolted on after the first 2 AM outage. The four patterns in this guide -- retry with backoff, model fallbacks, circuit breakers, and graceful degradation -- form a defense-in-depth strategy that keeps your agents running when everything around them is breaking.

The code in this article is framework-agnostic Python you can drop into any project. Start with retry + classify (the highest-ROI pattern), add fallback chains when you depend on a single provider, and layer in circuit breakers when your agents call external tools at scale.

If you want to skip building this resilience plumbing yourself, platforms like Nebula handle retry logic, model fallbacks, and tool circuit breakers at the infrastructure level -- so you can focus on what your agent does instead of how it recovers.

The complete code from this article is ready to copy-paste. Build something resilient.

Building Production AI Agents (26 Part Series)
The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
AI Agent Error Handling: 4 Resilience Patterns in Python
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools
DEV Community

Work through these 3 parts to earn the exclusive Google AI Studio Builder badge!

This track will guide you through Google AI Studio's new "Build apps with Gemini" feature, where you can turn a simple text prompt into a fully functional, deployed web application in minutes.

Read more →

Read More