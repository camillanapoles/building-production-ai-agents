# How to Test AI Agents (Before They Burn Your Budget)

> Artigo #9 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/how-to-test-ai-agents-before-they-burn-your-budget-5c1j)

---

Your agent passed your demo. It handled the five prompts you tested by hand. The stakeholders nodded. You shipped it.

Then it burned $153 in 30 minutes on a Tuesday afternoon.

The agent hit an ambiguous query, entered a reasoning loop, and called GPT-4 47 times trying to resolve a question it should have escalated after three attempts. Nobody noticed until the billing alert fired. This is not a hypothetical -- it's the most common failure pattern in production AI agents, and it's entirely preventable.

The problem isn't that teams skip testing. It's that they apply traditional testing patterns to systems that break those patterns by design. AI agents are non-deterministic, multi-step, and capable of generating confident-sounding output that is completely wrong. You can't assert response == expected when the same input produces different valid outputs on every run.

Here are five testing patterns that actually work for AI agents. Each one is framework-agnostic, includes copy-paste Python code, and targets a specific class of production failure.

Why Traditional Testing Breaks for AI Agents

Unit tests assume deterministic output. Integration tests assume stable APIs. Both assume that identical inputs produce identical outputs. AI agents violate all three assumptions simultaneously.

The core challenges:

Non-determinism: The same prompt yields different responses across runs. Both responses might be correct, but assertEqual still fails.
Multi-step state: An agent that routes correctly in step 1 might still fail in step 4 because of accumulated context drift. Testing individual steps misses cascading failures.
Tool side effects: Your agent calls real APIs, modifies databases, and sends messages. A test that exercises the happy path might accidentally send 200 emails to a production mailing list.

What we need instead is a layered testing approach that tests properties and behaviors rather than exact outputs.

Approach	Traditional Testing	Agent Testing
Assertions	Exact output match	Behavioral properties
Determinism	Same input = same output	Same input = output within bounds
Scope	Single function	Multi-step trace
Side effects	Mocked	Mocked + budget-capped
Quality	Binary pass/fail	Scored on a rubric
Pattern 1: Deterministic Skeleton Tests

The first insight: you don't need to test the LLM. OpenAI and Anthropic employ thousands of engineers to ensure their models generate coherent text. What you need to test is your code -- the routing logic, tool selection, state management, and orchestration that wraps around the model.

Mock the LLM. Test the skeleton.


from unittest.mock import MagicMock

def test_agent_routes_search_queries():
"""Agent should select web_search for factual questions."""
mock_llm = MagicMock()
mock_llm.complete.return_value = {
"tool": "web_search",
"args": {"query": "population of Tokyo 2026"}
}

agent = ResearchAgent(llm=mock_llm)
result = agent.plan("What is the current population of Tokyo?")

assert result.selected_tool == "web_search"
assert "Tokyo" in result.tool_args["query"]
# We're not testing if the LLM chose well --
# we're testing if our code handles the choice correctly.

def test_agent_rejects_dangerous_tool_calls():
"""Agent must block destructive actions without confirmation."""
mock_llm = MagicMock()
mock_llm.complete.return_value = {
"tool": "delete_database",
"args": {"target": "production"}
}

agent = TaskAgent(llm=mock_llm)
result = agent.plan("Clean up old records")

assert result.status == "requires_confirmation"
assert result.selected_tool != "delete_database"  # Should be blocked


Skeleton tests run in milliseconds because they never call an LLM API. They catch the bugs that matter most: routing logic that sends queries to the wrong tool, missing safety checks on destructive operations, and state management that drops context between steps.

Run these on every commit. They're free.

Pattern 2: Golden Path Replay Testing

Once your agent handles a task correctly, record the entire execution trace -- inputs, tool calls, intermediate results, and final output. That recording becomes a regression test.


import json
from pathlib import Path
from difflib import SequenceMatcher

def record_golden_run(agent, prompt, tag):
"""Record a successful agent run as a golden trace."""
trace = agent.run(prompt, record=True)
path = Path(f"tests/golden/{tag}.json")
path.write_text(json.dumps({
"prompt": prompt,
"tool_sequence": [t.name for t in trace.tool_calls],
"tool_args": [t.args for t in trace.tool_calls],
"output": trace.final_output,
"cost": trace.total_cost,
}, indent=2))
return trace

def test_golden_path_regression():
"""Current agent behavior should match recorded golden run."""
golden = json.loads(Path("tests/golden/summarize_docs.json").read_text())
trace = agent.run(golden["prompt"], record=True)

# Tool sequence should be identical
assert [t.name for t in trace.tool_calls] == golden["tool_sequence"]

# Output should be semantically similar (not identical)
similarity = SequenceMatcher(
None, trace.final_output, golden["output"]
).ratio()
assert similarity > 0.70, f"Output drift detected: {similarity:.0%} similar"

# Cost should not spike
assert trace.total_cost <= golden["cost"] * 1.5, (
f"Cost regression: ${trace.total_cost:.2f} vs golden ${golden['cost']:.2f}"
)


Golden path tests answer one question: "Does the agent still work the way it did when we last verified it?" Run them before every model upgrade, after prompt changes, and as a weekly regression suite.

Their limitation is obvious -- they only catch regressions, not novel failures. That's what the next pattern handles.

Pattern 3: Property-Based Agent Assertions

This is the most valuable pattern in this article. Instead of asserting what the agent should output, assert what it must never do.

Every production agent has invariants -- properties that must hold true regardless of input. Write them down. Then write tests that enforce them.


from dataclasses import dataclass
from typing import Callable

@dataclass
class AgentProperty:
name: str
check: Callable
severity: str  # "critical" | "warning"

PRODUCTION_PROPERTIES = [
AgentProperty(
name="no_destructive_without_confirm",
check=lambda trace: not any(
t.tool in ["delete", "drop", "truncate"] and not t.user_confirmed
for t in trace.tool_calls
),
severity="critical",
),
AgentProperty(
name="budget_per_request",
check=lambda trace: trace.total_cost < 0.50,
severity="critical",
),
AgentProperty(
name="max_reasoning_steps",
check=lambda trace: len(trace.steps) <= 10,
severity="warning",
),
AgentProperty(
name="no_hallucinated_urls",
check=lambda trace: all(
url_exists(u) for u in extract_urls(trace.final_output)
),
severity="critical",
),
AgentProperty(
name="cites_sources_when_claiming_data",
check=lambda trace: (
not contains_statistics(trace.final_output)
or contains_citations(trace.final_output)
),
severity="warning",
),
]

def test_agent_properties():
"""Run diverse inputs against all agent invariants."""
test_inputs = load_test_corpus("tests/inputs/diverse_queries.jsonl")

violations = []
for prompt in test_inputs:
trace = agent.run(prompt, record=True)
for prop in PRODUCTION_PROPERTIES:
if not prop.check(trace):
violations.append({
"property": prop.name,
"severity": prop.severity,
"input": prompt,
})

critical = [v for v in violations if v["severity"] == "critical"]
assert len(critical) == 0, (
f"{len(critical)} critical property violations: "
f"{[v['property'] for v in critical]}"
)


Property-based testing catches the failures that scare you most: the agent that silently deletes production data, the agent that spends $50 on a single request, the agent that cites a URL that doesn't exist. These are the failures that erode trust and cost real money.

Start with three properties: no destructive actions without confirmation, budget cap per request, and maximum reasoning steps. Add more as you discover new failure modes in production.

Pattern 4: Budget Tripwire Tests

Dedicated tests for the most expensive failure mode: agent loops. An agent that enters a reasoning loop will keep calling the LLM until something stops it. If nothing stops it, your bill does.


import time

def test_ambiguous_query_does_not_loop():
"""Ambiguous input should trigger clarification, not infinite reasoning."""
agent = Agent(max_iterations=20, budget_limit_usd=1.00)

result = agent.run("Do the thing with the stuff from last time")

assert result.iterations < 8, (
f"Agent looped {result.iterations} times on ambiguous input"
)
assert result.cost < 0.30, f"Spent ${result.cost:.2f} on ambiguous query"
assert result.status in ["completed", "clarification_needed", "escalated"]

def test_tool_error_does_not_cascade():
"""Failed tool calls should not trigger retry spirals."""
agent = Agent(max_iterations=20, budget_limit_usd=1.00)

# Inject a tool that always fails
agent.tools["flaky_api"] = lambda **kwargs: raise_error("503 Service Unavailable")

result = agent.run("Fetch the latest report from flaky_api")

retry_count = sum(
1 for t in result.trace.tool_calls if t.tool == "flaky_api"
)
assert retry_count <= 3, f"Agent retried flaky tool {retry_count} times"
assert result.status != "stuck"

def test_worst_case_cost():
"""Maximum possible cost for a single request stays under threshold."""
expensive_prompts = [
"Analyze every commit in the repository and summarize each one",
"Compare all products in our catalog with competitor pricing",
"Research the full history of this company from founding to present",
]

for prompt in expensive_prompts:
result = agent.run(prompt, budget_limit_usd=2.00)
assert result.cost < 2.00, (
f"Request exceeded budget: ${result.cost:.2f} for: {prompt[:50]}..."
)


The $153 incident from the intro? A single budget tripwire test would have caught it. The agent entered a loop because nobody tested what happens when the LLM can't resolve ambiguity. A two-line assertion -- iterations < 10 and cost < 1.00 -- would have flagged it before deployment.

Budget tripwires are slow (they actually run the agent), so run them nightly or before releases rather than on every commit.

Pattern 5: LLM-as-Judge Evaluation

When output quality matters -- summaries, recommendations, analysis -- you need a judge that understands language. Use a second LLM to evaluate the first.


import openai

def llm_judge(output: str, criteria: str, context: str = "") -> int:
"""Score agent output on a 1-5 scale using a cheap judge model."""
response = openai.chat.completions.create(
model="gpt-4o-mini",  # Cheap judge, not the agent's model
messages=[{
"role": "user",
"content": f"""Rate this AI agent output from 1 (poor) to 5 (excellent).

Criteria: {criteria}
{f'Context: {context}' if context else ''}

Agent output:
{output}

Respond with ONLY a single integer 1-5."""
}],
temperature=0,
)
return int(response.choices[0].message.content.strip())

def test_summary_quality():
"""Agent summaries should be accurate, complete, and concise."""
source_doc = load_test_doc("tests/fixtures/quarterly_report.md")
result = agent.run(f"Summarize this document: {source_doc}")

scores = {
"accuracy": llm_judge(result.output, "factual accuracy relative to source", source_doc),
"completeness": llm_judge(result.output, "covers all key points from source", source_doc),
"conciseness": llm_judge(result.output, "no unnecessary filler or repetition"),
}

assert scores["accuracy"] >= 4, f"Accuracy too low: {scores['accuracy']}/5"
assert scores["completeness"] >= 3, f"Missing key points: {scores['completeness']}/5"
assert scores["conciseness"] >= 3, f"Too verbose: {scores['conciseness']}/5"


A caveat: LLM judges have their own biases. GPT-4 tends to rate GPT-4 output generously. Use LLM-as-judge for directional quality tracking ("is this getting better or worse?") rather than absolute pass/fail decisions. Cross-validate with human review on a sample of outputs periodically.

Putting It All Together

You don't need all five patterns on day one. Here's a decision table:

Pattern	Best For	Cost	Speed	Run When
Skeleton Tests	Tool routing, safety checks	Free	Milliseconds	Every commit
Golden Path Replay	Regression after changes	Low	Seconds	Before model upgrades
Property Assertions	Safety violations, invariants	Low-Medium	Seconds	Every PR
Budget Tripwires	Cost explosions, loops	Medium	Minutes	Nightly / pre-release
LLM-as-Judge	Output quality tracking	Low	Seconds	Weekly / pre-release

Start with patterns 1 + 3 + 4. Skeleton tests catch routing bugs for free. Property assertions catch the scary failures (destructive actions, budget blowouts). Budget tripwires catch the expensive failures (loops, retry spirals). Together, these three patterns prevent roughly 80% of production agent incidents.

Add golden path replay when you're upgrading models or making major prompt changes. Add LLM-as-judge when your agent's output is customer-facing and quality directly impacts business outcomes.

Ship Agents You Can Sleep On

AI agents aren't functions -- they're systems with emergent behavior. A well-tested agent doesn't just produce correct output. It fails safely, fails cheaply, and fails in ways you've already anticipated.

The patterns in this guide are framework-agnostic by design. Whether you're building with LangGraph, CrewAI, OpenAI's Agents SDK, or raw API calls, the testing layer wraps around your agent the same way.

Platforms like Nebula build several of these patterns into the agent runtime itself -- step-by-step task tracking provides the trace data that golden path and property tests need, built-in memory gives you observability into agent reasoning between steps, and scoped credentials prevent the "agent with root access" failure mode entirely.

But you don't need any platform to start. Write three property assertions today. Test what your agent must never do. That's the highest-ROI testing investment you'll make this quarter.

Building Production AI Agents (26 Part Series)
The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
How to Test AI Agents (Before They Burn Your Budget)
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools
DEV Community

Build Apps with Google AI Studio 🧱

This track will guide you through Google AI Studio's new "Build apps with Gemini" feature, where you can turn a simple text prompt into a fully functional, deployed web application in minutes.

Read more →

Read More