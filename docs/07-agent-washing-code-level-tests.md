# Agent Washing: 5 Code-Level Tests to Tell Real AI Agents from Fakes

> Artigo #7 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/agent-washing-5-code-level-tests-to-tell-real-ai-agents-from-fakes-2bg6)

---

A RAND study found that 80-90% of AI agent projects fail. But here is the part nobody talks about: many of those projects were never agents to begin with.

Welcome to agent washing -- the practice of slapping the word "agent" onto any product that calls an LLM. Chatbots become "conversational agents." Cron jobs with GPT become "autonomous agents." A single API call wrapped in a while True loop becomes an "agentic workflow."

The term has been making rounds in enterprise circles -- Forbes, Gartner, and Reddit's r/AI_Agents have all flagged it. But most of the coverage targets procurement teams and enterprise buyers. Nobody has written the developer version: concrete, code-level tests you can run against your own tools.

This article fixes that. Five Python test functions. Run them against anything claiming to be an "AI agent." The results will tell you more than any vendor demo.

What Makes Something an Agent (Not Marketing)

Before the tests, we need a working definition. Not the marketing version -- the architecture version.

An AI agent is a system with a loop that includes:

Perception -- reads environment state (APIs, databases, user input)
Reasoning -- decides what to do next, including when to stop
Action -- executes tools with real side effects
Memory -- persists context across invocations
Autonomy -- operates without human input for routine decisions

The key word is loop. A pipeline runs A then B then C. An agent runs A, observes the result, decides whether to run B or D, handles failure, and keeps going until the goal is met.

Here is the difference in pseudocode:


# Pipeline (NOT an agent)
def pipeline(input):
result_a = step_a(input)
result_b = step_b(result_a)
return step_c(result_b)  # always the same path

# Agent (has a reasoning loop)
def agent(goal):
state = perceive()
while not goal_met(state):
plan = reason(goal, state, memory)
result = execute(plan.next_action)
memory.store(result)
state = perceive()  # re-evaluate after every action
return state


If the system you are evaluating looks more like the first function than the second, it is a pipeline with good marketing. Nothing wrong with pipelines -- but they solve different problems than agents do.

The 5 Code-Level Tests

These tests are framework-agnostic. They work whether you are evaluating a vendor tool, an open-source framework, or something you built yourself. Each test targets one of the five architectural properties above.

Test 1: Multi-Step Autonomy

What it checks: Can the system complete a multi-step task without returning control to the user between steps?


def test_multi_step_autonomy(agent):
"""Give the agent a task requiring 3+ sequential actions.
A real agent completes them autonomously.
A fake agent asks for confirmation after each step."""

task = "Find the top Python repos created this week, "
"summarize their READMEs, and save a report to disk."

result = agent.run(task)

# Check: did it complete without intermediate human prompts?
assert result.completed, "Agent did not complete the task"
assert result.human_interventions == 0, (
f"Agent required {result.human_interventions} human interventions "
f"-- a real agent handles routine multi-step tasks autonomously"
)
assert result.steps_executed >= 3, (
f"Only {result.steps_executed} steps -- task requires search, "
f"summarization, and file writing at minimum"
)


FAIL pattern: The system generates a plan but then asks "Should I proceed?" after every step. This is a copilot, not an agent.

PASS pattern: The system searches GitHub, reads READMEs, writes the report, and returns the finished result. You see the output, not a series of permission prompts.

Test 2: Error Recovery

What it checks: When a tool call fails, does the system adapt or crash?


def test_error_recovery(agent, mock_api):
"""Simulate a tool failure. Real agents retry or find alternatives.
Fake agents throw the error back at the user."""

# Make the primary API fail
mock_api.github.search.side_effect = RateLimitError("429")

task = "Find trending Python repositories from this week."
result = agent.run(task)

# A real agent should try an alternative approach
assert not result.raised_exception, (
"Agent crashed on API failure instead of recovering"
)
assert result.recovery_attempts > 0, (
"Agent made no attempt to recover from the failure"
)
# Check it tried a different strategy (web search, cache, etc.)
assert any(
step.tool != "github.search"
for step in result.steps
), "Agent only retried the same failing call"


FAIL pattern: Error: GitHub API rate limit exceeded. Please try again later. The system hands the error to you because it has no fallback logic.

PASS pattern: The system catches the rate limit, switches to a web search or cached data source, and still delivers results. The error is logged internally, not surfaced as a blocker.

Test 3: Dynamic Tool Selection

What it checks: Does the system choose tools based on the task, or does it follow a hardcoded path?


def test_dynamic_tool_selection(agent):
"""Give two different tasks. A real agent selects different tools.
A fake agent runs the same fixed workflow regardless."""

task_a = "What is the current price of Bitcoin?"
task_b = "Summarize the README of facebook/react on GitHub."

result_a = agent.run(task_a)
result_b = agent.run(task_b)

tools_a = {step.tool for step in result_a.steps}
tools_b = {step.tool for step in result_b.steps}

# The tool sets should be different
assert tools_a != tools_b, (
f"Agent used identical tools for completely different tasks: "
f"{tools_a}. This suggests hardcoded routing, not reasoning."
)
# Each should use contextually appropriate tools
assert any("search" in t or "price" in t for t in tools_a), (
"Bitcoin price task should invoke a search or price API"
)
assert any("github" in t for t in tools_b), (
"GitHub README task should invoke the GitHub API"
)


FAIL pattern: Both tasks route through the exact same chain: parse_input -> call_llm -> format_output. The "tool selection" is actually a single prompt template.

PASS pattern: Task A invokes a web search API. Task B invokes the GitHub API. The agent selected different tools because the tasks required different capabilities.

Test 4: Memory Persistence

What it checks: Does the system remember context from previous sessions?


def test_memory_persistence(agent):
"""Run a task, then later reference it without re-providing context.
Real agents remember. Fake agents start fresh every time."""

# Session 1: Give the agent information
agent.run("My project uses Python 3.12 and FastAPI. Remember this.")

# Clear the conversation (new session)
agent.new_session()

# Session 2: Ask something that requires the stored context
result = agent.run(
"Based on my project setup, suggest a testing framework."
)

# The agent should reference Python/FastAPI without being reminded
response_text = result.output.lower()
assert any(
term in response_text
for term in ["fastapi", "python", "pytest", "httpx"]
), (
"Agent showed no awareness of previously stored project context. "
"It either has no memory or memory does not persist across sessions."
)


FAIL pattern: "I don't have information about your project setup. Could you share the details?" The system has no memory beyond the current chat window.

PASS pattern: "Since your project uses FastAPI on Python 3.12, I would recommend pytest with httpx for async endpoint testing." The system recalled stored context from a previous session.

Test 5: Goal Decomposition

What it checks: Given a high-level goal, can the system break it into sub-tasks and execute them?


def test_goal_decomposition(agent):
"""Give a complex, high-level goal. Real agents decompose it.
Fake agents try to handle everything in one LLM call."""

goal = (
"Audit my project's dependencies for security vulnerabilities, "
"prioritize them by severity, and create a fix plan."
)

result = agent.run(goal)

# Check that the agent broke this into distinct sub-tasks
assert result.steps_executed >= 3, (
f"Only {result.steps_executed} step(s) for a complex audit task. "
f"A real agent decomposes this into scanning, analysis, "
f"and planning phases."
)

# Check for distinct phases (not just repeated LLM calls)
step_descriptions = [step.description for step in result.steps]
has_scan = any("scan" in d or "check" in d for d in step_descriptions)
has_analyze = any("priorit" in d or "sever" in d for d in step_descriptions)
has_plan = any("fix" in d or "plan" in d or "patch" in d for d in step_descriptions)

assert has_scan and has_analyze and has_plan, (
"Agent did not decompose the goal into scan -> analyze -> plan. "
"It likely tried to answer everything in a single prompt."
)


FAIL pattern: The system returns a single LLM response: "Here are some common vulnerabilities to watch for..." It generated advice instead of actually executing an audit.

PASS pattern: The system runs pip audit or safety check, parses the results, groups findings by severity, and generates a prioritized remediation plan with specific package versions to upgrade.

The Agent Spectrum: Not Everything Needs to Be an Agent

Here is something the agent washing discourse gets wrong: it implies you should always want a "real agent." You should not.

Agent capabilities exist on a spectrum:


LLM Wrapper    Pipeline    Workflow    Copilot    Agent
|             |           |          |          |
Single       Fixed       Branching   Human-    Autonomous
prompt +     chain of    logic with  in-the-   loop with
response     LLM calls   conditions  loop      reasoning


Each level is the right tool for something:

LLM Wrapper: Summarize this text. Translate this string. One input, one output. Perfect.
Pipeline: Process this CSV through three transformation steps. Predictable, repeatable. Great.
Workflow: Route this support ticket based on category, escalate if sentiment is negative. Branching logic without full autonomy. Solid.
Copilot: Help me write this code, suggest the next step, but let me approve before executing. Human stays in control. Ideal for high-stakes decisions.
Agent: Monitor my infrastructure, detect anomalies, investigate root causes, and apply fixes -- without waking me up at 3am. Full autonomy for complex, multi-step tasks.

The problem with agent washing is not that simpler tools exist. The problem is that simpler tools get marketed as agents, so you build expectations for autonomous behavior and get a chatbot instead.

The cost of getting this wrong goes both ways:

Over-engineering: Building an agent for a task that needs a pipeline wastes compute, adds latency, and creates failure modes that a simpler system would not have.
Under-engineering: Calling a chatbot an "agent" means teams build workflows expecting adaptive error handling and persistent memory that never materializes.
How to Evaluate Any AI Agent Tool

Beyond the five code tests, ask these questions before committing to any tool that claims to be an agent:

Can I see the decision trace? Real agents expose their reasoning chain -- which tools they considered, why they chose one over another, what they tried when something failed. If the system is a black box, you cannot debug it, and you cannot trust it.

Does it handle failures without my intervention? Try breaking something on purpose. Kill an API, feed it malformed input, give it a contradictory instruction. What happens? A real agent recovers or escalates gracefully. A wrapper crashes or hallucinates.

Does memory persist across sessions? Close the chat window. Come back tomorrow. Does it remember your project context, your preferences, your prior conversations? If every interaction starts from zero, it is a chatbot with good UX.

Can it use tools I did not hardcode? Give it access to a new API it has never seen. Can it read the docs and figure out how to use it? Real agents with dynamic tool selection can adapt to new capabilities. Fixed workflows cannot.

Can I trace exactly what happened and why? After a task completes, can you get a full execution log? Which tools were called, in what order, with what parameters, and what results? Observability is not optional for production agents.

Systems like Nebula are built around these principles -- autonomous sub-agents, persistent memory, tool orchestration with execution traces -- but the tests above apply to any tool you are evaluating. The point is not which product you pick. The point is that you test before you trust.

The Bottom Line

Agent washing wastes developer time and erodes trust in tools that could genuinely transform how we build software. The fix is straightforward: stop evaluating agents by their demos and start evaluating them by their behavior under pressure.

Run the five tests:

Multi-step autonomy -- does it finish the job without hand-holding?
Error recovery -- does it adapt when things break?
Dynamic tool selection -- does it reason about which tools to use?
Memory persistence -- does it remember context across sessions?
Goal decomposition -- does it break complex goals into executable steps?

If a tool passes all five, you have a real agent. If it fails most of them, you have a wrapper with good marketing. Both can be useful -- but only if you know which one you are actually buying.

Run these tests on your current stack this week. You might be surprised by what you find.

Building Production AI Agents (26 Part Series)
The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
Agent Washing: 5 Code-Level Tests to Tell Real AI Agents from Fakes
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools
DEV Community

Build Apps with Google AI Studio 🧱

This track will guide you through Google AI Studio's new "Build apps with Gemini" feature, where you can turn a simple text prompt into a fully functional, deployed web application in minutes.

Read more →

Read More