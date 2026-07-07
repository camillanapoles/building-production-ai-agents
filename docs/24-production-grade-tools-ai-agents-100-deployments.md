# Building Production-Grade Tools for AI Agents: What Works After 100 Deployments

> Artigo #24 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/building-production-grade-tools-for-ai-agents-what-works-after-100-deployments-20om)

---

Most developers who build AI agents make the same mistake: they spend weeks designing the orchestration layer, tuning the system prompt, and picking the right model — then hand the LLM a pile of hastily wrapped API endpoints and wonder why it fails in production.

Here's the hard truth from teams shipping agents daily: tool design has a larger impact on agent reliability than prompt engineering. A well-crafted tool prevents hallucinations at the structural level. A poorly crafted tool guarantees them.

This article walks through what we've learned from building, deploying, and debugging production AI agents across dozens of real-world workflows. You'll get concrete patterns, working code examples, and the anti-patterns that cost us the most in production incidents.

The Contract Between Deterministic and Non-Deterministic Code

When you write a function for another developer, you're working between two deterministic systems. Same input, same output. The calling code knows exactly what to expect.

An AI tool is a fundamentally different contract. You're writing an interface between deterministic code (your backend service, database, or API) and a non-deterministic consumer (the LLM). The model might:

Call your tool when you expected it to use something else
Send malformed arguments because the description was ambiguous
Retry your tool three times because the error message didn't tell it why it failed
Ignore your tool entirely because the description didn't explain when to use it

This means every tool needs five components that traditional APIs never bothered with: a precise name, a rich description, a strict input schema, structured error handling, and a predictable output format. Let's build each one.

1. Naming: The First Signal the LLM Evaluates

The tool name is the first thing the model scans when deciding which tool to call. It functions like a class name in a codebase — it sets expectations before any other signal.


# Bad: vague, could mean anything
@mcp_tool(name="process")
def process(data):
...

# Bad: too generic, overlaps with other tools
@mcp_tool(name="get_data")
def get_data(query: str):
...

# Good: specific verb + noun, clear scope
@mcp_tool(name="list_overdue_invoices")
def list_overdue_invoices(customer_id: str):
...

# Good: resource_action pattern
@mcp_tool(name="invoice_send_reminder")
def invoice_send_reminder(invoice_id: str, channel: str):
...


Pick one convention — verb_noun or resource_action — and enforce it across every tool on your server. Mixing conventions forces the LLM to learn two mental models, and under load, it will confuse them. We saw a 23% drop in correct tool selection on a production agent when the team had get_user, user_create, and delete_file all coexisting with no pattern.

2. Descriptions: Embedded Prompt Engineering

The tool description is the most underestimated field in tool design. The LLM reads this to decide when to use the tool and what it will get back. It's prompt engineering baked into the tool definition itself.


MISMATCHED_DESCRIPTION = "Searches the database"

GOOD_DESCRIPTION = """\
Full-text search across the company knowledge base.\
Use when the user asks to find internal documentation, policies, or technical specs.\
Returns up to 10 results ranked by relevance, each with title, snippet, and URL.\
Does NOT search emails or chat messages — use search_communications for those."""


Notice what the good description does: it says what it does, tells the LLM when to use it, describes the output shape, and explicitly states what it won't do. That last part is critical — explicit negative boundaries prevent the LLM from reaching for the wrong tool when it's close-but-not-right.

A real measurement from our deployments: improving tool descriptions alone — no code changes — cut task completion time by 40% and reduced wrong-tool selection by 60%.

3. Input Schemas: Never Trust the LLM

Models hallucinate parameter values, confuse types, and invent fields that don't exist. Your tool must validate every input before processing. JSON Schema constraints are your first line of defense:


GOOD_INPUT_SCHEMA = {
"type": "object",
"properties": {
"query": {
"type": "string",
"minLength": 1,
"maxLength": 500,
"description": "Natural language search query or exact document title"
},
"limit": {
"type": "integer",
"minimum": 1,
"maximum": 50,
"default": 10,
"description": "Maximum number of results to return"
},
"category": {
"type": "string",
"enum": ["engineering", "hr", "finance", "legal", "all"],
"default": "all",
"description": "Restrict search to a specific document category"
}
},
"required": ["query"],
"additionalProperties": False
}


Enums eliminate entire classes of failures. When environment accepts only "staging" or "production", the LLM can't invent "prod-us-east" and crash your deployment script. We've found that using enums and regex patterns for parameters eliminated 80% of runtime validation errors in production.

Poka-Yoke Parameters

Take it a step further with poka-yoke design — making misuse structurally impossible:


# Instead of accepting free-text paths that cause path traversal:
{"path": {"type": "string"}}  # bad

# Use enums with absolute paths for known configs:
{"config": {
"type": "string",
"enum": ["/etc/prod/config.yaml", "/etc/staging/config.yaml"]
}}  # good

4. Error Handling: Errors Are Prompts for the LLM

When a tool fails, the LLM needs enough information to decide whether to retry, try a different tool, or ask the user for help. Opaque errors like "Internal Server Error" leave the model stranded.

MCP has two error mechanisms, and conflating them causes silent failures:

Protocol-level errors (JSON-RPC): unknown tool, malformed arguments, server unavailable. The call never reached your tool logic.
Tool execution errors (isError: true): the tool ran but failed. The agent can reason about these.
# Bad: generic error, LLM cannot reason about what went wrong
return {"error": "Something went wrong"}

# Good: structured error with actionable context via isError
return {
"isError": True,
"content": [{
"type": "text",
"text": json.dumps({
"error": "RATE_LIMIT_EXCEEDED",
"message": "Search API rate limit reached. Maximum 10 requests per minute.",
"retryAfterSeconds": 30,
"suggestion": "Wait 30 seconds before retrying, or narrow the query to reduce result processing time."
})
}]
}


This pattern — machine-readable code, human-readable explanation, retry guidance, and an actionable suggestion — eliminates a large class of agent failures where the model receives a cryptic error and hallucinates a recovery path.

5. Output Format: Consistency Is Everything

Unpredictable output formats force the LLM to guess, which increases the chance of misinterpretation and downstream errors.


# Bad: inconsistent output shape
def search(term):
results = db.query(term)
if results:
return results  # list of dicts
return "No results found"  # string — different type entirely!

# Good: consistent envelope, always the same shape
def search(term, limit=10):
results = db.query(term, limit=limit+1)
return {
"status": "success",
"resultCount": min(len(results), limit),
"results": [
{
"title": r.title,
"snippet": r.snippet[:200],
"url": r.url,
"relevanceScore": r.score
}
for r in results[:limit]
],
"hasMore": len(results) > limit
}


The agent always knows what shape to expect. It doesn't need to branch on isinstance(result, str) vs isinstance(result, list). That predictability compounds across multi-step workflows.

6. Token Efficiency: The Hidden Cost That Kills ROI

Every tool response goes into the LLM's context window. Verbose responses burn tokens, increase cost, and degrade reasoning quality as context fills up.

Three strategies that work in production:

Paginate aggressively. Return 10 results with a cursor, not 1,000 records. The agent can page if it needs more.

Support summary modes. Offer detailed=True/False parameters. Default to False. Let the agent request more detail only when needed.

Strip internal metadata. The agent doesn't need database IDs, internal timestamps, or ORM fields. Return only what the LLM needs to understand and act on the result.


# Internal DB record (terrible for agent context):
{
"id": "a1b2c3d4-e5f6-7890",
"_created_at": "2026-04-15T08:23:11.442Z",
"_updated_at": "2026-04-30T14:07:33.101Z",
"_tenant_id": "org_48291",
"name": "John Smith",
"role": "Product Manager",
"email": "john@acme.com",
"status": "active",
"preferences": {"theme": "dark", "notifications": True, ...}
}

# Agent-friendly output:
{
"name": "John Smith",
"role": "Product Manager",
"email": "john@acme.com",
"status": "active"
}


We measured a 3.2x reduction in per-task token consumption just by stripping internal metadata from tool outputs. At scale, that's the difference between a profitable agent and a cost center.

7. Behavioral Annotations: Signals the Agent Can Act On

The MCP 2025-03-26 spec introduced tool annotations — metadata fields that help agents make smarter decisions about tool invocation:


tool_annotations = {
"readOnlyHint": True,       # Safe to call without confirmation
"destructiveHint": False,   # Won't mutate state
"idempotentHint": True,     # Safe to retry with same args
"openWorldHint": False      # Only reads from known database
}


These annotations drive real behavior in agent clients. A destructiveHint: true tool triggers a confirmation gate before execution. An idempotentHint: true tool lets the client retry safely on timeout. But remember: annotations are hints, not guardrails. The agent client decides whether to honor them.

Anti-Patterns We've Seen in Production
The God Tool
@mcp_tool(name="process_customer_request")
def process_customer_request(request_text: str):
# Parses intent, searches DB, sends email, updates CRM, creates ticket...
# This is 6 operations fused into one. When step 3 fails, the agent
# cannot retry steps 4-6 independently.


Keep tools atomic. One tool, one purpose. If it needs to do X and Y, it should be two tools that the agent composes. Atomic tools are easier to test, easier for the LLM to reason about, and easier to compose into complex workflows.

Tool Description Drift

Your tool description says "returns a list of users." Six months later, after a refactor, it returns a paginated object with users and total_count fields. The description was never updated. The agent breaks silently.

Treat tool descriptions as living documentation. When you run evals (and you should), include description accuracy checks in your validation pass.

Silent Failure Swallowing
def get_metric(name):
try:
return metrics_api.get(name)
except Exception:
return {"data": []}  # agent thinks everything is fine


The agent received what looks like a valid but empty response. It proceeds with wrong assumptions. Always return the failure visibly — isError: true with context — so the agent can reason about recovery.

A Real Production Tool, End to End

Here's a complete MCP tool definition that follows every principle above, from a production deployment monitoring service:


@server.tool(
name="deploy_service",
description=(
"Deploy a service to the specified environment. "
"Use this for production and staging deployments. "
"For rollbacks, use rollback_service instead. "
"Returns the deployment ID, target version, and current status."
),
input_schema={
"type": "object",
"properties": {
"service": {
"type": "string",
"description": "Service name from the service registry. Use list_services to find available names."
},
"environment": {
"type": "string",
"enum": ["staging", "production"],
"description": "Target environment for the deployment."
},
"version": {
"type": "string",
"pattern": r"^v\d+\.\d+\.\d+$",
"description": "Semantic version to deploy, e.g., v2.4.1."
}
},
"required": ["service", "environment", "version"],
"additionalProperties": False
},
annotations={
"destructiveHint": True,
"idempotentHint": True,
"openWorldHint": False
}
)
async def deploy_service(service: str, environment: str, version: str):
try:
result = await deploy_api.deploy(service, environment, version)
return {
"status": "success",
"deployment_id": result.id,
"target_version": version,
"environment": environment,
"started_at": result.started_at.isoformat()
}
except DeploymentError as e:
return {
"isError": True,
"content": [{
"type": "text",
"text": json.dumps({
"error": "DEPLOYMENT_FAILED",
"message": str(e),
"service": service,
"environment": environment,
"version": version,
"suggestion": "Check the build status with check_build_status before retrying. If the build passed, verify the environment has capacity."
})
}]
}
except Exception as e:
return {
"isError": True,
"content": [{
"type": "text",
"text": json.dumps({
"error": "INTERNAL_ERROR",
"message": f"Unexpected error during deployment: {str(e)}",
"suggestion": "This is not a retryable error. Escalate to the infrastructure team."
})
}]
}


Every principle is represented: precise name, rich description with cross-reference, strict schema with enum and pattern validation, behavioral annotations, structured success output, and structured failure output with actionable suggestions.

Testing Tools With LLMs, Not Just Unit Tests

Unit tests verify your tool returns the right data. They don't verify the LLM can figure out which tool to call, construct valid arguments, or recover from errors.

The only real test for a tool is: put it in front of an LLM and give it a task. Run an evaluation with 20-50 real-world prompts and measure:

Tool selection accuracy: Did the LLM pick the right tool?
Argument correctness: Did it send valid parameters?
Error recovery: When the tool fails, does the LLM retry productively or hallucinate?
Token efficiency: How many tokens does the tool response consume?

Automate this. Run evals on every PR that changes a tool definition. If a tool description change drops selection accuracy from 95% to 80%, it's a regression — even if the code itself is perfect.

When to NOT Build a Tool

Not every API endpoint needs to be a tool. Some operations are too risky (delete production data), too expensive (run a model training job), or too complex (multi-step workflows that the agent can't verify). Implement those as workflow primitives in your orchestration layer instead — deterministic code that the agent triggers but doesn't directly call.

The rule of thumb: if the worst-case outcome of the LLM calling this tool wrong is "the user sees a weird message," build it as a tool. If it's "someone loses money" or "the system breaks," wrap it in your orchestration layer with guardrails first.

The TL;DR is simple: treat every tool definition as if it's the product, because for an AI agent, it is. The model reads tool descriptions like source code — every word, every constraint, every example matters. Get this right and your agents become dramatically more reliable without touching a single line of prompt engineering.

Building Production AI Agents (26 Part Series)
The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
Building Production-Grade Tools for AI Agents: What Works After 100 Deployments
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools
DEV Community

Build Apps with Google AI Studio 🧱

This track will guide you through Google AI Studio's new "Build apps with Gemini" feature, where you can turn a simple text prompt into a fully functional, deployed web application in minutes.

Read more →

Read More