# Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns

> Artigo #2 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/your-ai-agent-has-amnesia-fix-it-with-these-4-memory-patterns-4n3g)

---

You build an AI agent that works perfectly in testing. It triages your inbox, summarizes Slack threads, files GitHub issues. You deploy it on a schedule and walk away satisfied.

Then you check back the next morning. The inbox triage agent archived the same emails twice. The standup summarizer doesn't remember yesterday's summary. The research agent re-discovers the same information it found 24 hours ago.

Your agent has amnesia.

This is the most common failure mode I see in production AI agents -- not bad prompts, not wrong models, but complete memory loss between runs. There are exactly 4 memory patterns that fix this, and most developers either use the wrong one or skip memory entirely. Here's which pattern to use and when.

Why Agents Forget (And Why It Matters)

LLMs are stateless by design. Every API call starts from zero. The context window is the only "memory" an LLM has, and it resets completely between calls.

This creates three expensive problems in production:

Wasted API calls. Your agent re-fetches data it already processed. An email triage agent that can't remember which emails it already handled will re-process your entire inbox every run.
Inconsistent behavior. Without memory of past decisions, your agent makes contradictory choices. It flags the same sender as "high priority" one run and "low priority" the next.
No learning curve. Humans get faster at recurring tasks because they remember patterns. A stateless agent is permanently stuck on day one.

The fix isn't one-size-fits-all. You need different memory patterns for different problems. Here are the four that cover every production use case I've encountered.

Pattern 1: Conversation Buffer (Short-Term Memory)

The simplest pattern. Store the last N messages in a list and pass them to the LLM as context.


class ConversationBuffer:
def __init__(self, max_tokens=4000):
self.messages = []
self.max_tokens = max_tokens

def add(self, role: str, content: str):
self.messages.append({"role": role, "content": content})
self._trim()

def _trim(self):
while self._token_count() > self.max_tokens and len(self.messages) > 1:
self.messages.pop(0)

def _token_count(self) -> int:
return sum(len(m["content"]) // 4 for m in self.messages)

def get_context(self) -> list:
return self.messages


This is a ring buffer with token-aware truncation. When the buffer exceeds your token budget, it drops the oldest messages first.

Use it when: You're building a single-session chatbot or a simple Q&A agent where the conversation starts and ends in one sitting.

Don't use it when: Your agent runs on a schedule, handles multi-step workflows, or needs to remember anything between sessions. The buffer evaporates when the session ends.

The conversation buffer is the default in most frameworks, which is why most agents have amnesia -- developers assume this is enough and never add persistent memory.

Pattern 2: Summary Memory (Compressed Long-Term)

When conversations get long, you can't keep every message. Summary memory uses the LLM itself to compress older turns into a running summary, keeping recent messages verbatim.


def compress_memory(messages, summary_so_far, model="gpt-4o-mini"):
old_messages = messages[:-5]  # Keep last 5 verbatim
recent = messages[-5:]

new_summary = llm_call(
model=model,
prompt=f"""Previous summary: {summary_so_far}
New messages to incorporate: {format_messages(old_messages)}
Write an updated summary capturing all key facts,
decisions, and context. Be specific -- names, numbers,
and decisions matter more than pleasantries."""
)
return new_summary, recent


Every N turns (or when you hit a token threshold), summarize the oldest messages and keep only the compressed version plus recent context.

Use it when: Your agent handles long support conversations, extended debugging sessions, or any interaction that regularly exceeds the context window.

The trade-off: Summaries are lossy. Every compression cycle risks losing details that seem unimportant now but matter later. An agent handling a 2-hour customer support thread might summarize away the customer's original error message -- then struggle to resolve the issue when it comes back up 45 minutes later.

Cost note: Each compression costs an LLM call, but a cheap one. A gpt-4o-mini call to summarize 20 messages costs a fraction of a cent -- far cheaper than expanding your context window indefinitely.

Pattern 3: Retrieval Memory (Semantic Search)

Store every past interaction in a vector database. Before each agent run, embed the current task and retrieve the most relevant past experiences using semantic search.


import chromadb
from uuid import uuid4

class RetrievalMemory:
def __init__(self, collection_name="agent_memory"):
self.client = chromadb.PersistentClient(path="./memory_db")
self.collection = self.client.get_or_create_collection(
name=collection_name
)

def store(self, text: str, metadata: dict = None):
self.collection.add(
documents=[text],
ids=[f"mem_{uuid4().hex[:12]}"],
metadatas=[metadata or {}]
)

def recall(self, query: str, top_k: int = 5) -> list:
results = self.collection.query(
query_texts=[query],
n_results=top_k
)
return results["documents"][0]


This is the only memory pattern that scales to thousands of past interactions without blowing up your context window. Instead of stuffing everything into the prompt, you search for what's relevant.

Use it when: Your agent accumulates hundreds of past interactions and needs to recall specific ones. Research agents, customer support bots with case history, or any agent where "remember that time we..." matters.

The critical gotcha: Retrieval quality depends entirely on embedding quality. Bad embeddings return irrelevant memories, and irrelevant memories are worse than no memory at all -- they actively confuse the agent. Always test your retrieval pipeline with real queries before trusting it in production.

A quick sanity check: embed 10 real queries your agent will handle, retrieve the top 5 results for each, and manually verify relevance. If fewer than 3 out of 5 results are useful, your embedding model or chunking strategy needs work.

Pattern 4: Structured State (Key-Value Memory)

This is the most underrated pattern -- and the one that matters most in production. Instead of storing raw conversations, explicitly store the facts your agent has learned.


import json
from datetime import datetime
from pathlib import Path

class StructuredMemory:
def __init__(self, db_path="agent_state.json"):
self.db_path = Path(db_path)
self.state = self._load()

def remember(self, key: str, value: str, category: str = "general"):
if category not in self.state:
self.state[category] = {}
self.state[category][key] = {
"value": value,
"updated_at": datetime.now().isoformat()
}
self._save()

def recall(self, key: str, category: str = "general"):
return self.state.get(category, {}).get(key, {}).get("value")

def recall_category(self, category: str) -> dict:
return {k: v["value"] for k, v in self.state.get(category, {}).items()}

def _load(self) -> dict:
if self.db_path.exists():
return json.loads(self.db_path.read_text())
return {}

def _save(self):
self.db_path.write_text(json.dumps(self.state, indent=2))


This stores explicit, queryable facts: user preferences, routing rules, entity mappings, operational checkpoints.

Consider an email triage agent. It needs to remember that emails from boss@company.com are always high priority, that "Project Atlas" is the internal code name for the database migration, and that the last processed email ID was msg_abc123. These are structured facts, not conversations to retrieve.

Use it when: Your agent learns user preferences, tracks recurring entities, or maintains operational state between runs. Any agent running on a schedule needs this pattern.

Why most guides skip this: Memory architecture articles love discussing vector databases and retrieval-augmented generation. Structured state isn't sexy -- it's a JSON file. But in production, it solves 80% of agent memory needs with zero retrieval latency and perfect accuracy. There's no embedding quality to worry about, no semantic search to fine-tune. You store a fact, you get the exact fact back.

The Decision Framework: Which Pattern When

Here's the cheat sheet:

What You Need	Pattern	Why
Remember this conversation	Buffer	Cheapest, simplest, session-scoped
Handle very long conversations	Summary	Compressed, lossy but practical
Recall specific past events from hundreds of runs	Retrieval	Scales via vector search
Remember learned facts and preferences	Structured State	Explicit, reliable, zero retrieval noise
Production autonomous agent	Structured State + Retrieval	Facts for reliability, retrieval for context

Most production agents use a combination. Here's the stack I recommend:

Foundation: Structured state (Pattern 4) for everything your agent has learned -- preferences, rules, entity mappings, operational checkpoints.
Context layer: Retrieval memory (Pattern 3) for recalling relevant past interactions when handling new tasks.
Session layer: Conversation buffer (Pattern 1) for the current execution run.
Optional: Summary memory (Pattern 2) only if individual sessions regularly exceed your context window.

Start with structured state alone. It's the highest-impact, lowest-complexity pattern. Add retrieval only when your agent has accumulated enough history that structured facts alone can't capture the relevant context.

What This Looks Like in Practice

A production email triage agent combining all four patterns:

Structured state stores sender priorities (boss@company.com → high), project codenames (Atlas → database migration), and the last processed email timestamp.
Retrieval memory stores summaries of past email threads, so when a new email arrives from a known sender, the agent has context on previous conversations.
Conversation buffer tracks the current triage session -- which emails were processed and what actions were taken.
The agent starts each run by loading structured state, queries retrieval memory as needed for specific emails, and uses the buffer within the session.

Platforms like Nebula handle this natively -- structured state persists across scheduled runs, agents share context through multi-agent delegation, and memory keys survive between sessions without external database setup. But the patterns work with any framework. The architecture matters more than the tooling.

The Bottom Line

The difference between a demo agent and a production agent is memory. Most agents fail not because they're not smart enough, but because they can't remember what they learned yesterday.

Start with Pattern 4 (structured state). It solves 80% of production memory needs with a JSON file and zero complexity. Add Pattern 3 (retrieval) when your agent accumulates enough history that structured facts alone don't capture the full picture.

Don't let your agent start every day from scratch. The patterns are simple. The impact is immediate.

What's the weirdest thing your agent forgot that caused a production incident? Drop it in the comments -- I'll share mine.

Building Production AI Agents (26 Part Series)
The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools
DEV Community

Work through these 3 parts to earn the exclusive Google AI Studio Builder badge!

This track will guide you through Google AI Studio's new "Build apps with Gemini" feature, where you can turn a simple text prompt into a fully functional, deployed web application in minutes.

Read more →

Read More