# Context Engineering for AI Agents: A Practical Guide

> Artigo #5 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/context-engineering-for-ai-agents-a-practical-guide-4lp6)

---

Your AI agent works perfectly in demos. Then you deploy it to production, hand it a real workflow with 30 conversation turns, 15 tool definitions, and a pile of retrieved documents -- and it starts hallucinating, ignoring instructions, and picking the wrong tools.

The model didn't get dumber. Your context engineering failed.

Context engineering is the practice of curating everything an LLM sees before it responds -- not just the prompt, but the system instructions, tool definitions, conversation history, retrieved documents, and previous step results. While prompt engineering focuses on crafting the right instruction, context engineering decides what information makes it into the context window and what gets left out.

This matters because in production agent systems, the context window is prime real estate. Every token competes for attention, and the wrong mix of information degrades performance faster than a weaker model would. This guide gives you four concrete strategies to engineer your agent's context -- each with runnable Python code you can use today.

What Is Context Engineering (and Why Prompts Aren't Enough)

Prompt engineering is writing the question on an exam. Context engineering is choosing which reference materials you bring into the exam room.

In a simple chatbot, prompt engineering is often sufficient. The user sends a message, you wrap it in a system prompt, and the model responds. The context is small and predictable.

Agent systems break this model completely. A typical production agent's context includes:

System prompt: 500-1,500 tokens defining behavior and constraints
Tool definitions: 200-500 tokens per tool (10 tools = 2,000-5,000 tokens)
Conversation history: grows linearly with every turn
Retrieved documents: 500-2,000 tokens per chunk
Previous step results: variable, often large API responses

By the time the user speaks, 60-80% of the context window may already be consumed. The model has less room to reason, important instructions get buried in the middle where attention is weakest, and performance degrades silently.

The fix isn't a bigger context window. GPT-4o supports 128K tokens, but research consistently shows that model performance degrades well before the technical limit. The fix is engineering what goes in.

The Context Budget: Know What You're Spending

Before optimizing anything, you need to measure. A context budget calculator tells you exactly how much space each component consumes and how much remains for the model to work with.


```python
import tiktoken

def calculate_context_budget(
model_limit: int,
system_prompt: str,
tools: list[dict],
history: list[dict],
retrieved_docs: list[str],
encoding_name: str = \"o200k_base\",
) -> dict:
\"\"\"Calculate context budget breakdown and remaining capacity.\"\"\"
enc = tiktoken.get_encoding(encoding_name)

def count(text: str) -> int:
return len(enc.encode(text))

system_tokens = count(system_prompt)
tool_tokens = sum(count(str(t)) for t in tools)
history_tokens = sum(count(str(m)) for m in history)
doc_tokens = sum(count(d) for d in retrieved_docs)

total_used = system_tokens + tool_tokens + history_tokens + doc_tokens
remaining = model_limit - total_used
utilization = (total_used / model_limit) * 100

budget = {
\"model_limit\": model_limit,
\"system_prompt\": system_tokens,
\"tools\": tool_tokens,
\"history\": history_tokens,
\"retrieved_docs\": doc_tokens,
\"total_used\": total_used,
\"remaining\": remaining,
\"utilization_pct\": round(utilization, 1),
}

if utilization > 60:
budget[\"warning\"] = (
f\"Context is {utilization:.0f}% full before user input. \"
\"Apply context engineering strategies below.\"
)

return budget


# Example: a typical agent with 10 tools and some history
budget = calculate_context_budget(
model_limit=128_000,
system_prompt=\"You are a helpful research assistant...\",  # ~50 tokens
tools=[{\"name\": f\"tool_{i}\", \"description\": \"...\", \"parameters\": {}} for i in range(10)],
history=[{\"role\": \"user\", \"content\": \"Analyze this dataset...\"} for _ in range(20)],
retrieved_docs=[\"Document chunk...\" * 50 for _ in range(5)],
)

for key, value in budget.items():
print(f\"{key}: {value}\")

```

The rule of thumb: if your context is more than 60% full before the user's current message, you have a context engineering problem. Run this calculator on your agent and you'll likely be surprised how much space tools and history consume.

4 Context Engineering Strategies for Production Agents

Once you know where your budget goes, apply these strategies to reclaim it.

Strategy 1: Sliding Window with Summarization

The simplest and highest-impact strategy. Keep the last N conversation turns verbatim (the model needs recent context for coherence), and compress everything older into a summary.


```python
from openai import OpenAI

client = OpenAI()

def manage_conversation_context(
messages: list[dict],
system_prompt: str,
max_recent_turns: int = 6,
summary_model: str = \"gpt-4o-mini\",
) -> list[dict]:
\"\"\"Keep recent turns verbatim, summarize older ones.\"\"\"
if len(messages) \u003C= max_recent_turns:
return [{\"role\": \"system\", \"content\": system_prompt}] + messages

old_messages = messages[:-max_recent_turns]
recent_messages = messages[-max_recent_turns:]

# Summarize old messages with a cheap, fast model
summary_response = client.chat.completions.create(
model=summary_model,
messages=[
{
\"role\": \"system\",
\"content\": (
\"Summarize this conversation in under 200 tokens. \"
\"Preserve: key decisions made, data retrieved, \"
\"user preferences stated, and any unresolved tasks.\"
),
},
*old_messages,
],
max_tokens=200,
)

summary = summary_response.choices[0].message.content

return [
{\"role\": \"system\", \"content\": system_prompt},
{\"role\": \"system\", \"content\": f\"Previous conversation summary: {summary}\"},
*recent_messages,
]


# Before: 40 messages = ~20,000 tokens of history
# After: 1 summary (~200 tokens) + 6 recent messages (~3,000 tokens)
# Savings: ~17,000 tokens per agent call

```

When to use: Long-running chat agents, multi-step workflows, any agent that accumulates more than 10 conversation turns.

Tradeoff: You lose granular detail from early turns. Mitigate this by tuning the summary prompt to preserve whatever your agent needs most -- decisions, data points, or user preferences.

Cost note: The summarization call uses gpt-4o-mini at $0.15/1M input tokens. Summarizing 15,000 tokens of old history costs ~$0.002 and saves you ~$0.04 in reduced input tokens on the main call (at GPT-4o pricing). It pays for itself 20x over.

Strategy 2: Relevance Scoring for Retrieved Context

RAG systems often dump every retrieved chunk into the context. This is wasteful. Not all retrieved documents are equally relevant to the current agent step.

Instead of injecting all retrieved results, score them by relevance and only include those above a threshold:

Embedding similarity: rank chunks by cosine similarity to the current query
Recency weighting: boost documents updated recently if freshness matters
```python
Source priority: rank authoritative sources (official docs) above informal ones (forum posts)
Threshold filtering: drop anything below a relevance score of 0.7 (tune this empirically)
```

For long documents that pass the threshold, extract only the most relevant paragraphs rather than including the full text. A 2,000-token document often has 200 tokens of actually useful content for the current step.

When to use: Any agent with RAG, knowledge base lookups, or document retrieval. The more documents you retrieve, the more this matters.

The key insight: retrieval and context injection are separate decisions. Your retriever finds candidates. Your context engineer decides which candidates earn a spot in the window.

Strategy 3: Dynamic Tool Injection

If your agent has 15+ tools, their definitions alone can consume 5,000-10,000 tokens. Worse, a large tool list increases the chance the model picks the wrong tool -- attention gets diluted across too many options.

The fix: don't load all tools into every call. Use a lightweight classifier to predict which tools are relevant for the current step, and inject only those.


```python
from openai import OpenAI
import json

client = OpenAI()

def select_tools_for_step(
task_description: str,
all_tools: list[dict],
max_tools: int = 5,
classifier_model: str = \"gpt-4o-mini\",
) -> list[dict]:
\"\"\"Select relevant tools for this step using a cheap classifier.\"\"\"
tool_names = [t[\"function\"][\"name\"] for t in all_tools]

response = client.chat.completions.create(
model=classifier_model,
messages=[
{
\"role\": \"system\",
\"content\": (
\"You are a tool router. Given a task, return a JSON array \"
\"of the most relevant tool names from the available list. \"
f\"Return at most {max_tools} tools. Return ONLY the JSON array.\"
),
},
{
\"role\": \"user\",
\"content\": (
f\"Task: {task_description}\
\
\"
f\"Available tools: {json.dumps(tool_names)}\"
),
},
],
max_tokens=100,
)

try:
selected_names = json.loads(response.choices[0].message.content)
except json.JSONDecodeError:
return all_tools[:max_tools]  # Fallback: return first N tools

selected = [t for t in all_tools if t[\"function\"][\"name\"] in selected_names]
return selected if selected else all_tools[:max_tools]


# Example: agent has 20 tools, but this step only needs 3
all_tools = [
{\"type\": \"function\", \"function\": {\"name\": \"search_web\", \"description\": \"Search the web\", \"parameters\": {}}},
{\"type\": \"function\", \"function\": {\"name\": \"read_file\", \"description\": \"Read a file\", \"parameters\": {}}},
{\"type\": \"function\", \"function\": {\"name\": \"send_email\", \"description\": \"Send an email\", \"parameters\": {}}},
{\"type\": \"function\", \"function\": {\"name\": \"query_database\", \"description\": \"Query SQL database\", \"parameters\": {}}},
# ... 16 more tools
]

relevant_tools = select_tools_for_step(
task_description=\"Find the latest sales figures from the database\",
all_tools=all_tools,
max_tools=3,
)
# Returns: [query_database, read_file] -- saves ~4,000 tokens

```

When to use: Agents with more than 10 tools, especially those using MCP where users can plug in arbitrary tool sets. If you've read our piece on MCP Tool Overload, this is the code-level fix.

Cost of the classifier: The routing call uses ~200 input tokens on gpt-4o-mini -- roughly $0.00003. It saves 3,000-8,000 tokens on the main agent call, which at GPT-4o pricing saves $0.007-$0.020. A 200x+ return on investment per call.

Strategy 4: Step Result Compression

Multi-step agents pass results between steps. A search step returns 10 results with full metadata. A database query returns 500 rows. An API call returns a nested JSON blob with 50 fields.

The next step rarely needs all of that. Compress intermediate results to only the fields the downstream step requires:

Search results: Compress from full metadata to title + URL + 1-line summary per result
API responses: Extract only the fields referenced in the next step's prompt
Database results: Aggregate or sample instead of passing raw rows
Web scrapes: Strip navigation, ads, and boilerplate -- keep only the content body

The principle: every step should receive the minimum viable context to do its job. If a step only needs 5 fields from a 50-field API response, the other 45 fields are wasting tokens and diluting attention.

When to use: Pipeline agents, sequential multi-step workflows, any agent where one step's output feeds into another.

Context Engineering vs Memory vs RAG: When to Use What

These three concepts are complementary, not competing. Here's how they relate:

\tContext Engineering\tMemory\tRAG
What it does\tSelects what goes into THIS call's context window\tPersists information across sessions\tRetrieves from external knowledge bases
Scope\tSingle LLM call\tCross-session, long-term\tExternal data sources
Question it answers\t\"What should the model see right now?\"\t\"What should the agent remember?\"\t\"What external knowledge is relevant?\"
When it runs\tEvery agent step\tOn save/load boundaries\tOn retrieval queries
Example\tDropping 15 old messages, keeping 5 recent ones\tStoring user preference: \"prefers Python examples\"\tFetching product docs matching the user's question

RAG retrieves candidate information. Memory persists important facts. Context engineering selects what actually makes it into the window from all available sources -- including RAG results and memory.

A production agent needs all three. RAG without context engineering stuffs irrelevant chunks into the window. Memory without context engineering loads every saved fact regardless of relevance. Context engineering without memory or RAG has nothing to select from.

If you're building memory into your agents, our previous article on agent memory patterns covers the storage and retrieval side. This article covers the selection side -- what happens after you retrieve, right before the LLM call.

Putting It Together

The four strategies layer naturally:

Budget first. Run the context calculator on your agent. Know your baseline.
Compress history. Sliding window summarization is the highest-impact, lowest-effort fix. Start here.
Filter retrieval. Score and threshold your RAG results instead of dumping everything.
Inject tools dynamically. If you have 10+ tools, route to subsets per step.
Compress step results. Extract only what the next step needs.

The metrics to watch:

Context utilization %: how full is the window before the user speaks? Target: under 50%.
Tokens per step: are your agent calls getting more expensive over time? If yes, history or results are bloating.
Task accuracy on multi-step workflows: better context engineering directly improves accuracy on complex tasks where the model needs to track goals across many steps.

Platforms like Nebula handle context engineering automatically at the architecture level -- when a parent agent delegates to a sub-agent, each sub-agent receives only the context relevant to its specific task, with shared memory handling cross-agent continuity. But whether you use a platform or build from scratch, the principles are the same: measure your budget, compress what's old, filter what's retrieved, and inject only what's needed.

Key Takeaways

The best AI agents aren't the ones with the most powerful models or the largest context windows. They're the ones that put the right information in front of the model at the right time.

Start with the context budget calculator. If your agent is over 60% utilization before the user speaks, apply the strategies in order: summarize history, filter retrieval, route tools dynamically, compress step results. Each one compounds.

Context engineering is the missing piece between \"my agent works in demos\" and \"my agent works in production.\" The model already knows how to reason. Your job is to give it exactly what it needs to reason well.

This is part of the Building Production AI Agents series. Previous: How to Stop AI Agent Cost Spirals Before They Start. See also: MCP Tool Overload: Why More Tools Make Your Agent Worse.

The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
Context Engineering for AI Agents: A Practical Guide
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools