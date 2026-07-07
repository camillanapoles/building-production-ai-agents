# 5 Open-Source Tools for Testing AI Agents Before They Break Production

> Artigo #23 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/5-open-source-tools-for-testing-ai-agents-before-they-break-production-2g1m)

---

Your AI agent passes all unit tests. The prompt looks right. You deploy. Then a user reports that the support agent started recommending refund policies instead of troubleshooting steps. No crash. No stack trace. Just quietly wrong.

This is the hardest class of bug in agentic systems: silent regressions. You change one thing and the agent's behavior drifts in ways traditional testing can't catch. The agent returns 200, calls some tools, produces output — just not the right output for the new configuration.

Agent evaluation is no longer optional. In 2026, with MCP tool ecosystems spanning 177,000+ APIs and multi-agent orchestration becoming standard, the gap between "works on my machine" and "works in production" has never been wider.

Here's a practical comparison of five tools solving this problem — from lightweight local evaluators to full LLMOps platforms.

TL;DR
Tool	Best For	Local-First?	CI/CD Ready?	Price
EvalView	Golden baseline regression detection	Yes	Yes (GitHub Actions)	Free, Apache 2.0
agentevals	OpenTelemetry-based scoring without re-runs	Yes	Yes	Free, Apache 2.0
AgentV	Terminal-first YAML evals, any CLI agent	Yes	Yes	Free, MIT
LangWatch	Agent simulations + full LLMOps platform	No (open-source core)	Yes	Free tier / Enterprise
Agenta	Team prompt management + evaluation UI	Yes (self-host)	Yes	Free, open-source
What Makes Agent Testing Different

Traditional tests are deterministic: given input X, expect output Y. Agents break this contract because:

Non-determinism: The same prompt produces different outputs across runs
Tool trajectory matters: Two outputs might look similar, but one called helm_list_releases and the other hallucinated a command
Context window effects: A model swap from GPT-4o to Claude might change which tools the agent discovers
Prompt fragility: Adding "be more friendly" to a system prompt can silently drop critical tool calls

Good agent testing needs to validate trajectories, not just answers. Every tool below approaches this differently.

1. EvalView — pytest for AI Agents

Stars: Growing fast · License: Apache 2.0

EvalView takes the simplest approach: snapshot your agent's behavior, then diff every subsequent run against it. Think of it like git diff for agent behavior.


pip install evalview
evalview init        # Detect agent, create starter suite
evalview snapshot    # Save current behavior as baseline
evalview check       # Catch regressions after every change


The four scoring layers are where it gets interesting:

Tool calls + sequence (free) — Did the agent call the right tools in the right order?
Code-based checks (free) — Regex, JSON schema validation on outputs
Semantic similarity (~$0.00004/test) — Embedding-based output comparison
LLM-as-judge (~$0.01/test) — GPT, Claude, or Gemini scoring with custom criteria

What sets EvalView apart is multi-reference baselines. Non-deterministic agents can have up to 5 valid response variants, and EvalView checks against all of them instead of forcing you to pick one "golden" answer.

Strengths: Works without API keys (fully offline with Ollama). Golden baseline diffing is unique — no other tool does automatic before/after snapshots of agent behavior. MCP contract testing catches interface drift.

Weaknesses: Requires you to maintain baseline snapshots. If your agent's expected behavior changes legitimately, you need to re-snapshot manually.

Best for: Teams that want "pytest for agents" — fast, local, baseline-driven regression detection.

2. agentevals — Score Agents from Traces, No Re-Runs

Stars: 115+ · License: Apache 2.0 · Language: Python

agentevals solves a different problem: "I already have traces from production. Can I evaluate them without replaying expensive LLM calls?"

The answer is yes. It reads OpenTelemetry traces (from LangChain, Google ADK, OpenAI Agents SDK, or any OTel-instrumented framework) and scores them against evaluation sets you define.


agentevals run samples/helm.json \
--eval-set samples/eval_set_helm.json \
-m tool_trajectory_avg_score

[PASS]  tool_trajectory_avg_score    1  PASSED   1  0ms


The key insight is separation of recording and evaluation. You record once (from production traffic, test runs, or live sessions), then evaluate as many times as you want with different metrics — no additional API calls, no token costs.

The zero-code mode is particularly clean:


# Terminal 1: Start the receiver
agentevals serve --dev

# Terminal 2: Point your agent at it
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
python your_agent.py


Traces stream to the built-in web UI at localhost:8001 in real-time.

Strengths: No re-execution cost. OTel-native means it works with any framework that supports OpenTelemetry. CLI + Web UI + MCP server. Helm chart for Kubernetes deployment.

Weaknesses: You need OTel instrumentation in place first. If your agent framework doesn't emit traces, you'll need to add it. The eval set format is YAML and requires some setup effort.

Best for: Teams already using OpenTelemetry who want to evaluate production traces without burning tokens on replays.

3. AgentV — Terminal-First YAML Evals

Stars: Niche but growing · License: MIT · Language: TypeScript

AgentV takes the minimal approach: YAML test files, executed from the terminal, results in JSONL.


# evals/math.yaml
description: Math problem solving
tests:
- id: addition
input: What is 15 + 27?
expected_output: "42"
assertions:
- type: contains
value: "42"

agentv eval evals/math.yaml


Everything lives in Git — eval files, judge prompts, and results. The hybrid grader system combines deterministic code checks with customizable LLM graders:


assertions:
- type: contains
value: "fizz"
- type: code-grader
command: ./validators/check_syntax.py
- type: llm-grader
prompt: ./graders/correctness.md


The agentv compare command is underrated — it diffs results across multiple targets (different models, different prompts, different agent versions) so you can see exactly where behavior changed.

Strengths: No server, no signup, no cloud dependency. Runs in seconds. JUnit XML output for CI pipelines. Works with any CLI agent — Claude Code, Codex, Copilot, local models.

Weaknesses: Smaller community than LangSmith or Promptfoo. The YAML format is simple but can become verbose for complex multi-turn conversations.

Best for: Solo developers and small teams who want evals in Git, not in a dashboard.

4. LangWatch — Agent Simulations + LLMOps

Open-source core · Cloud platform available

LangWatch positions itself as the LangSmith alternative that doesn't require the LangChain ecosystem. The differentiator is agent simulation testing — instead of evaluating individual input/output pairs, it tests multi-turn agent workflows end-to-end.

The simulation engine runs through complex scenarios:

Multi-turn conversations with tool usage
Multi-modal input flows (text + image + code)
Multi-agent interactions where one agent hands off to another

This catches a class of bugs that trace-based evaluation misses: the individual turns might look fine, but the flow between turns has a bug. Think of it as integration testing vs unit testing for agents.

Strengths: Framework-agnostic (OpenAI, Anthropic, CrewAI, Pydantic AI, custom). Agent simulations catch flow-level bugs. DSPy integration for automated prompt optimization. ISO 27001 / SOC 2 certified.

Weaknesses: The open-source core is self-hosting only; the cloud platform is where the full feature set lives. Heavier operational footprint than local-first tools.

Best for: Teams running complex multi-turn agentic workflows who need simulation-level validation before deployment.

5. Agenta — Team Prompt Management + Evaluation

Open-source · License: Apache 2.0

Agenta solves the organizational problem: "My engineers changed a prompt, but nobody told the product team, and now the agent grades customer tickets differently."

It's an LLMOps platform built around team collaboration:

Prompt management with UI editing for domain experts, full API parity for developers
Side-by-side playground for comparing prompts and models with version history
Automated evaluation with LLM-as-a-judge, built-in evaluators, or custom code
Trace annotation with team feedback — turn production traces into tests with one click

The cross-functional workflow is the real value here. Product managers can edit prompts through the UI, engineers version them through Git, and evaluators run automatically against both versions to flag behavioral changes.

Strengths: Best-in-class team collaboration. Playground comparisons make it easy to see prompt impact before shipping. Full trace observability. Docker-based self-hosting.

Weaknesses: Heavier to deploy than CLI tools. Requires a database backend (PostgreSQL). More setup overhead for individual developers.

Best for: Teams where prompt changes involve both technical and non-technical stakeholders.

Architecture: Building an Evaluation Pipeline

Here's how these tools fit into a real CI/CD pipeline:


Development Phase:
Developer changes prompt → agentv eval evals/regression.yaml
│
├── PASS → Merge PR
└── FAIL → Review diff, fix prompt

Staging Phase:
Deploy to staging → agentevals run production_traces/
│                   --eval-set staging_eval_set.json
└── Scores below threshold → Block deploy

Production Phase:
agent runs → EvalView capture live traffic
│            → evalview check (hourly)
├── REGRESSION detected → Slack alert + rollback
└── All clear → Continue monitoring


The key insight is that no single tool covers all three phases. LangWatch simulations catch edge cases before anything ships. AgentV gates PR merges. EvalView monitors production. agentevals evaluates staging traffic for free (re-using recorded traces).

This is where platforms like Nebula become interesting — instead of stitching together five tools across dev, staging, and production, the evaluation lifecycle is built into the agent platform itself. Agents running on Nebula inherit tracing, evaluation, and monitoring as part of the runtime, so the "capture → evaluate → alert" pipeline doesn't require separate infrastructure.

But if you're building your own stack, the five tools above are production-ready and free.

Choosing the Right Tool

Here's the decision framework I'd use:

"I just want to know if my agent still works after a change" → EvalView. Golden baselines, one command to check.
"I have production traces and want to evaluate without costs" → agentevals. Read existing OTel traces, score against eval sets, zero re-run cost.
"I want evals in my repo, next to my code" → AgentV. YAML files, terminal execution, Git-native.
"My agents have complex multi-turn flows" → LangWatch. Simulate the flows before they hit users.
"Multiple teams touch my agent prompts" → Agenta. UI for non-technical stakeholders, API parity for engineers.
The Bottom Line

Agent evaluation is no longer a "nice to have." When your agent controls refund workflows, code deployments, or customer-facing responses, the cost of "it seemed fine locally" has gone up dramatically.

The good news: open-source evaluation tools in 2026 are mature enough that you have no excuse. EvalView for regression detection, agentevals for trace-based scoring, AgentV for terminal-first evals, LangWatch for simulation testing, Agenta for team workflows — pick what matches your workflow and start gating your agent deployments.

The agents that ship reliably are the ones that get tested before they ship. Not after the first angry customer email.

Building Production AI Agents (26 Part Series)
The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
5 Open-Source Tools for Testing AI Agents Before They Break Production
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools
DEV Community

Build Apps with Google AI Studio 🧱

This track will guide you through Google AI Studio's new "Build apps with Gemini" feature, where you can turn a simple text prompt into a fully functional, deployed web application in minutes.

Read more →

Read More