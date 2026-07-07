# Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção

> Artigo #26 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/building-custom-mcp-servers-a-developers-guide-to-production-grade-ai-agent-tools-3pjm)

---

O Model Context Protocol (MCP) tornou-se o padrão padrão para conectar agentes de IA a ferramentas e APIs externas. Governado pela Linux Foundation desde o início de 2025 e adotado pela OpenAI, Anthropic, Microsoft e Vercel, o MCP é a porta USB-C do ecossistema de IA – um protocolo que permite que qualquer aplicativo LLM se comunique com qualquer servidor de ferramenta.

Mas há uma lacuna entre a leitura das especificações e a construção de algo que funcione de maneira confiável na produção. Passei os últimos meses construindo servidores MCP para fluxos de trabalhos de agentes de produção, e este guia captura os padrões que realmente importam.

Se você leu os resumos “6 plataformas de gateway de agente”, você sabe quais servidores MCP consumir. Este é o guia para quando você precisar construir um sozinho.

O que estamos construindo

Ao final deste guia, você terá criado um servidor MCP pronto para produção que:

Expõe ferramentas digitadas com validação de esquema JSON
Usa transporte HTTP Streamable (o padrão recomendado para 2026)
Lida com erros normalmente com respostas estruturadas
Inclui autenticação adequada para operações confidenciais
É testável com o Inspetor MCP

Vamos começar com a base.

Arquitetura: os três blocos de construção do MCP

Antes de escrever código, entenda o que seu servidor pode expor. MCP define três primitivas:

Apresenta o que faz quem o controla
Ferramentas Funções que o modelo de IA chama (gravar, calcular, agir) O modelo decide quando invocar
Recursos Dados somente leitura (arquivos, esquemas de banco de dados, documentos de API) O aplicativo recupera e fornece
Solicita modelos pré-construídos para fluxos de trabalhos acionadores de usuário comuns explicitamente

Para um servidor de ferramentas — o padrão de produção mais comum — você se concentrará nas ferramentas. Recursos e prompts são opcionais, mas úteis para fornecer contexto e orientar o comportamento do modelo.

Configurando um servidor TypeScript MCP

O TypeScript SDK oficial é a forma mais amplamente adotada de construir servidores MCP. É o que Claude Desktop, Cursor e Windsurf usam internamente.


//servidor.ts
```python
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

```

Este é o esqueleto. Todo servidor MCP segue este padrão: declarar capacidades, definir esquemas de ferramentas, implementar manipuladores, conectar um transporte.

Ferramentas de escrita que os agentes realmente usam bem

O maior erro que vejo nos designs de servidores MCP é escrever ferramentas da mesma forma que você escreveria endpoints REST para outros desenvolvedores. Os agentes não leem a documentação como os humanos. Os nomes, descrições e esquemas de suas ferramentas precisam ser otimizados para que um LLM seja descoberto e usado corretamente.

Convenções de nomenclatura

Use nomes descritivos e orientados para a ação:

Bom: search_codebase, create_jira_ticket, deploy_to_staging
Ruim: exec, run, helper, util
Descrições que funcionam

A descrição da sua ferramenta é a documentação do agente. Seja explícito sobre quando usá-lo, o que ele faz e os casos extremos.


{
```python
name: "deploy_service",
description:
"Deploy a service to the staging environment. Use when the user asks to deploy, push to staging, or test a deployment. Does NOT deploy to production — use deploy_to_production for that. Requires the service to have passed CI checks.",
}

Input Schema Design
```

Mantenha os parâmetros necessários mínimos. Os agentes ficam confusos com esquemas complexos com muitos campos obrigatórios. Use padrões sensatos sempre que possível.


{
esquema de entrada: {
```python
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

```

Um campo obrigatório, parâmetros opcionais com padrões claros. O agente pode ter sucesso com o mínimo de informações e pedir mais quando necessário.

HTTP streamável: o transporte de produção

O transporte Stdio é adequado para desenvolvimento local (Claude Desktop, VS Code), mas para implantações de produção você precisa de HTTP. Em 2026, Streamable HTTP substituiu Server-Sent Events (SSE) como o padrão recomendado.


// http-servidor.ts
```python
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

```

A principal vantagem do Streamable HTTP sobre SSE é que as conexões têm vida curta e são sem estado. Cada par solicitação-resposta é independente, tornando trivial a implantação atrás de balanceadores de carga e grupos de escalonamento automático.

Testando com o Inspetor MCP

O Inspetor MCP é o Carteiro do mundo MCP. Execute-o em seu servidor durante o desenvolvimento para validar os esquemas e as respostas da ferramenta antes que qualquer agente os toque:


npx @modelcontextprotocol/nó inspetor dist/server.js


Para servidores HTTP:


npx @modelcontextprotocol/inspector http://localhost:3000/mcp


Isso fornece uma IU da web onde você pode navegar pelas ferramentas disponíveis e seus esquemas, executar ferramentas com parâmetros personalizados, inspecionar mensagens JSON-RPC brutas e validar caminhos de tratamento de erros.

Sempre execute suas ferramentas por meio do Inspetor antes de implantar. Encontrei mais erros de esquema no Inspetor do que nas conversas reais dos agentes.

Padrões de segurança de produção

O modelo de segurança do MCP é intencionalmente permissivo no nível do protocolo – o aplicativo host implementa as proteções.

Portões de aprovação em nível de ferramenta

Para operações confidenciais, adicione uma camada de aprovação:


```python
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
```

Nunca confie nos argumentos do modelo. Mesmo agentes bem comportados podem ter alucinações sobre formatos de parâmetros:


const argumentos = z
.objeto({
```python
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
```
Abordagem melhor para transporte
Desenvolvimento de estúdio local, ferramentas pessoais stdio
Docker + proxy reverso Ferramentas internas da equipe Streamable HTTP
Vercel (via @vercel/mcp-adapter) Endpoints públicos e sem servidor HTTP streamável
Azure Container Apps Enterprise, ecossistema da Microsoft Streamable HTTP
HTTP streamável de alta escala e multirregional do Kubernetes
O quadro completo: onde cabem os servidores MCP

Os servidores MCP são a camada de ferramentas em uma arquitetura de agente maior. Você cria servidores que encapsulam recursos específicos — um servidor Jira, um servidor GitHub, um servidor de consulta de banco de dados — e um orquestrador como o Nebula roteia o servidor certo para o agente certo com base na intenção do usuário.

Depurando problemas comuns
Incompatibilidades de esquema: valide com Zod, retorne erros descritivos
Descrições ausentes: escreva descrições que especifiquem quando (e quando não) usar a ferramenta
Suposições de estado: torne as ferramentas sem estado - aceite todo o contexto necessário nos argumentos
Falhas de tempo limite: retornar um ID de operação imediatamente, fornecer uma ferramenta de verificação de status
Principais conclusões
MCP é o padrão para ferramentas de agente de IA em 2026
Crie ferramentas otimizadas para agentes, não para humanos
Use HTTP Streamable para produção
Sempre valide as entradas com Zod
Teste todas as ferramentas com o MCP Inspector
Os servidores MCP são a camada de ferramentas; orchestradoress como Nebula lidam com roteamento e estado
O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção