# Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools

> Artigo #26 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/building-custom-mcp-servers-a-developers-guide-to-production-grade-ai-agent-tools-3pjm)

---

The Model Context Protocol (MCP) has become the default standard for connecting AI agents to external tools and APIs. Governed by the Linux Foundation since early 2025 and adopted by OpenAI, Anthropic, Microsoft, and Vercel, MCP is the USB-C port of the AI ecosystem — one protocol that lets any LLM application talk to any tool server.

But there's a gap between reading the spec and building something that works reliably in production. I've spent the last few months building MCP servers for production agent workflows, and this guide captures the patterns that actually matter.

If you've read the "6 Agent Gateway Platforms" roundups, you know which MCP servers to consume. This is the guide for when you need to build one yourself.

What We're Building

By the end of this guide, you'll have built a production-ready MCP server that:

Exposes typed tools with JSON Schema validation
Uses Streamable HTTP transport (the 2026 recommended standard)
Handles errors gracefully with structured responses
Includes proper authentication for sensitive operations
Is testable with the MCP Inspector

Let's start with the foundation.

Architecture: The Three MCP Building Blocks

Before writing code, understand what your server can expose. MCP defines three primitives:

Feature	What It Does	Who Controls It
Tools	Functions the AI model calls (write, compute, act)	Model decides when to invoke
Resources	Read-only data (files, DB schemas, API docs)	Application retrieves and provides
Prompts	Pre-built templates for common workflows	User triggers explicitly

For a tool server — the most common production pattern — you'll focus on tools. Resources and prompts are optional but useful for providing context and guiding the model's behavior.

Setting Up a TypeScript MCP Server

The official TypeScript SDK is the most widely adopted way to build MCP servers. It's what Claude Desktop, Cursor, and Windsurf use internally.


// server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
CallToolRequestSchema,
ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import { z } from "zod";

// Define a tool with Zod validation
const CodeReviewInput = z.object({
repoPath: z.string().min(1, "Repository path is required"),
prNumber: z.number().int().positive("PR number must be positive"),
strictness: z.enum(["basic", "standard", "deep"]).default("standard"),
});

type CodeReviewInput = z.infer<typeof CodeReviewInput>;

// Server instance
const server = new Server(
{
name: "code-review-mcp",
version: "1.0.0",
},
{
capabilities: {
tools: {},
},
}
);

// Tool registration
server.setRequestHandler(ListToolsRequestSchema, async () => ({
tools: [
{
name: "review_pull_request",
description:
"Perform a code review on a pull request at the given path with configurable strictness",
inputSchema: {
type: "object",
properties: {
repoPath: {
type: "string",
description: "Absolute path to the local repository",
},
prNumber: {
type: "number",
description: "Pull request number to review",
},
strictness: {
type: "string",
enum: ["basic", "standard", "deep"],
description: "How thorough the review should be",
},
},
required: ["repoPath", "prNumber"],
},
},
],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
if (request.params.name === "review_pull_request") {
const args = CodeReviewInput.parse(request.params.arguments);

try {
const review = await performReview(args.repoPath, args.prNumber, args.strictness);
return {
content: [
{
type: "text",
text: JSON.stringify(review, null, 2),
},
],
};
} catch (error) {
return {
content: [
{
type: "text",
text: `Review failed: ${error instanceof Error ? error.message : "Unknown error"}`,
},
],
isError: true,
};
}
}

throw new Error(`Unknown tool: ${request.params.name}`);
});

// Start with stdio transport (for local development)
const transport = new StdioServerTransport();
await server.connect(transport);


This is the skeleton. Every MCP server follows this pattern: declare capabilities, define tool schemas, implement handlers, connect a transport.

Writing Tools That Agents Actually Use Well

The biggest mistake I see in MCP server designs is writing tools the way you'd write REST endpoints for other developers. Agents don't read documentation the way humans do. Your tool names, descriptions, and schemas need to be optimized for an LLM to discover and use correctly.

Naming Conventions

Use descriptive, action-oriented names:

Good: search_codebase, create_jira_ticket, deploy_to_staging
Bad: exec, run, helper, util
Descriptions That Work

Your tool description is the agent's documentation. Be explicit about when to use it, what it does, and edge cases.


{
name: "deploy_service",
description:
"Deploy a service to the staging environment. Use when the user asks to deploy, push to staging, or test a deployment. Does NOT deploy to production — use deploy_to_production for that. Requires the service to have passed CI checks.",
}

Input Schema Design

Keep required parameters minimal. Agents get confused by complex schemas with many required fields. Use sensible defaults wherever possible.


{
inputSchema: {
type: "object",
properties: {
serviceName: {
type: "string",
description: "Name of the service to deploy (e.g., 'api-gateway', 'worker')",
},
version: {
type: "string",
description: "Semantic version to deploy. If omitted, uses the latest built version.",
},
region: {
type: "string",
enum: ["us-west-2", "us-east-1", "eu-west-1"],
description: "Target region. Defaults to us-west-2.",
},
},
required: ["serviceName"],
},
}


One required field, optional parameters with clear defaults. The agent can succeed with minimal information and ask for more when needed.

Streamable HTTP: The Production Transport

Stdio transport is fine for local development (Claude Desktop, VS Code), but for production deployments you need HTTP. In 2026, Streamable HTTP has replaced Server-Sent Events (SSE) as the recommended standard.


// http-server.ts
import express from "express";
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";

const app = express();
app.use(express.json());

const mcpServer = new Server(
{ name: "production-mcp", version: "2.0.0" },
{ capabilities: { tools: {} } }
);

// Register tools (same as before)
mcpServer.setRequestHandler(ListToolsRequestSchema, async () => ({
tools: [
// ... tool definitions
],
}));

// HTTP endpoint
app.post("/mcp", async (req, res) => {
const transport = new StreamableHTTPServerTransport({
sessionId: req.headers["mcp-session-id"] as string | undefined,
});

transport.onerror = (error) => {
console.error("Transport error:", error);
};

await transport.handleRequest(req.body, req.headers, res);

if (transport.sessionId) {
res.setHeader("mcp-session-id", transport.sessionId);
}
});

app.listen(3000, () => {
console.log("MCP server listening on port 3000");
});


The key advantage of Streamable HTTP over SSE is that connections are short-lived and stateless. Each request-response pair is independent, making it trivial to deploy behind load balancers and auto-scaling groups.

Testing with MCP Inspector

The MCP Inspector is the Postman of the MCP world. Run it against your server during development to validate your tool schemas and responses before any agent touches them:


npx @modelcontextprotocol/inspector node dist/server.js


For HTTP servers:


npx @modelcontextprotocol/inspector http://localhost:3000/mcp


This gives you a web UI where you can browse available tools and their schemas, execute tools with custom parameters, inspect raw JSON-RPC messages, and validate error handling paths.

Always run your tools through the Inspector before deploying. I've caught more schema bugs in the Inspector than in actual agent conversations.

Production Security Patterns

MCP's security model is intentionally permissive at the protocol level — the host application implements the guardrails.

Tool-Level Approval Gates

For sensitive operations, add an approval layer:


const sensitiveTools = ["delete_resource", "modify_production_config"];

server.setRequestHandler(CallToolRequestSchema, async (request) => {
if (sensitiveTools.includes(request.params.name)) {
return {
content: [
{
type: "text",
text: `This operation requires approval. Please confirm by calling approve_operation with session ID.`,
},
],
isError: false,
};
}

// Normal tool handling...
});

Input Validation with Zod

Never trust the model's arguments. Even well-behaved agents can hallucinate parameter shapes:


const args = z
.object({
email: z.string().email(),
template: z.string().min(1).max(100),
variables: z.record(z.string()),
})
.strict()
.parse(request.params.arguments);

Rate Limiting and Quotas

MCP servers don't have built-in rate limiting — add it yourself:


import { rateLimit } from "express-rate-limit";

const limiter = rateLimit({
windowMs: 60 * 1000,
max: 100,
});

app.use("/mcp", limiter);

Deployment Options
Approach	Best For	Transport
Local stdio	Development, personal tools	stdio
Docker + reverse proxy	Internal team tools	Streamable HTTP
Vercel (via @vercel/mcp-adapter)	Serverless, public endpoints	Streamable HTTP
Azure Container Apps	Enterprise, Microsoft ecosystem	Streamable HTTP
Kubernetes	Multi-region, high-scale	Streamable HTTP
The Full Picture: Where MCP Servers Fit

MCP servers are the tool layer in a larger agent architecture. You build servers that encapsulate specific capabilities — a Jira server, a GitHub server, a database query server — and an orchestrator like Nebula routes the right server to the right agent based on the user's intent.

Debugging Common Issues
Schema mismatches: Validate with Zod, return descriptive errors
Missing descriptions: Write descriptions that specify when (and when not) to use the tool
State assumptions: Make tools stateless — accept all needed context in arguments
Timeout failures: Return an operation ID immediately, provide a status-check tool
Key Takeaways
MCP is the standard for AI agent tooling in 2026
Build tools optimized for agents, not humans
Use Streamable HTTP for production
Always validate inputs with Zod
Test every tool with MCP Inspector
MCP servers are the tool layer; orchestrators like Nebula handle routing and state
Building Production AI Agents (26 Part Series)
The God Agent Anti-Pattern: Why Your AI Breaks at 20 Tools
Your AI Agent Has Amnesia: Fix It With These 4 Memory Patterns
...
22 more parts...
The 5-Layer Security Model Every AI Agent Needs in Production
Building Custom MCP Servers: A Developer's Guide to Production-Grade AI Agent Tools
DEV Community

Build Apps with Google AI Studio 🧱

This track will guide you through Google AI Studio's new "Build apps with Gemini" feature, where you can turn a simple text prompt into a fully functional, deployed web application in minutes.

Read more →

Read More