# Como criar carregamento de ferramenta adiado para agentes de IA em 15 minutos

> Artigo #22 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/how-to-build-deferred-tool-loading-for-ai-agents-in-15- Minutes-4p6m)

---

Seu agente possui 40 ferramentas. Cada definição de ferramenta – nome, descrição, parâmetros do esquema JSON – custa aproximadamente 200 tokens. São 8.000 tokens antes que o agente faça alguma coisa. Adicione alguns servidores MCP e você queimará 55.000 tokens apenas nas definições de ferramentas por solicitação.

O termo da indústria é “inchaço de token”. A solução é o carregamento adiado da ferramenta: comece com uma pequena ferramenta de pesquisa, carregue ferramentas específicas somente quando o agente precisar delas e descarregue-as quando terminar.

Este tutorial mostra como. Um arquivo, código executável, sem dependências de estrutura.

O problema
```python
# What most tutorials do:
agent = Agent(tools=[tool_1, tool_2, tool_3, ..., tool_40])
# Every LLM call ships ALL 40 tool definitions in the prompt.
# Cost: ~8,000 tokens per call just for tool schemas.

```

Quando você executa agentes autônomos 24 horas por dia, 7 dias por semana, essa sobrecarga aumenta rapidamente. Um agente que faz 100 ligações/dia queima 800.000 tokens extras diariamente apenas descrevendo ferramentas que nunca usa.

A solução: pesquisar e carregar

Em vez de carregar todas as ferramentas antecipadamente, forneça ao agente exatamente uma ferramenta: pesquisa de ferramentas. Quando o agente precisa de um recurso, ele procura a ferramenta certa, o sistema a carrega e o agente executa novamente com a ferramenta recém-disponível.


Usuário: "Encontre o PR mais recente no nebula-web e atualize o changelog"

Turno 1: O agente possui apenas `search_tools(query)`. Ele pesquisa.
Turno 2: O sistema carrega `get_pr()` + `update_changelog()`.
O agente agora possui 2 ferramentas. Chama eles.
Turno 3: Agente responde com resultado.


Três turnos em vez de um, mas a economia de tokens é enorme: aproximadamente 400 tokens para a definição da ferramenta de pesquisa versus aproximadamente 8.000 para todas as 40 ferramentas.

Etapa 1: construir o registro da ferramenta

Primeiro, um registro que armazena todas as ferramentas disponíveis, mas expõe apenas um subconjunto de cada vez.


```python
from dataclasses import dataclass, field
from typing import Callable, Any

@dataclass
class Tool:
name: str
description: str
parameters: dict  # JSON Schema
fn: Callable
category: str = "general"  # For search filtering

class DeferredToolRegistry:
def __init__(self):
self.all_tools: dict[str, Tool] = {}
self.active_tools: set[str] = set()

def register(self, tool: Tool):
self.all_tools[tool.name] = tool

def search(self, query: str) -> list[dict]:
"""Search all tools by name, description, and category."""
query_lower = query.lower()
results = []
for name, tool in self.all_tools.items():
score = 0
if query_lower in name.lower(): score += 3
if query_lower in tool.description.lower(): score += 2
if query_lower in tool.category.lower(): score += 1
if score > 0:
results.append({"name": name, "description": tool.description, "score": score})
results.sort(key=lambda r: r["score"], reverse=True)
return results[:5]  # Return top 5 matches

def load(self, tool_names: list[str]) -> list[dict]:
"""Activate specific tools. Returns their definitions for the LLM."""
for name in tool_names:
if name in self.all_tools:
self.active_tools.add(name)
return [
{"type": "function", "function": {
"name": t.name, "description": t.description, "parameters": t.parameters,
}}
for t in self.all_tools.values() if t.name in self.active_tools
]

def execute(self, tool_name: str, arguments: dict) -> Any:
if tool_name not in self.active_tools:
raise ValueError(f"Tool '{tool_name}' is not loaded. Load it first.")
tool = self.all_tools[tool_name]
return tool.fn(**arguments)

def unload_all(self):
self.active_tools.clear()

```

O registro separa storage (todas as ferramentas, sempre disponíveis para pesquisa) da ativação (ferramentas que o LLM pode realmente chamar). As pontuações da função de pesquisa correspondem por nome, descrição e categoria — sem necessidade de incorporação.

Etapa 2: crie a ferramenta de pesquisa

A ferramenta de busca é a única ferramenta que o agente vê no primeiro turno.


```python
def make_search_tool(registry: DeferredToolRegistry) -> Tool:
return Tool(
name="search_tools",
description=(
"Search the available tool catalog. "
"Returns matching tool names and descriptions. "
"Use this first to find the right tool before calling it. "
"Example: search_tools('pull request') or search_tools('database query')."
),
parameters={
"type": "object",
"properties": {
"query": {
"type": "string",
"description": "Short description of the capability you need (e.g., 'send email', 'query database')",
}
},
"required": ["query"],
},
fn=lambda query: registry.search(query),
category="system",
)

```

Uma ferramenta, 200 tokens. Ele substitui 40 ferramentas por 8.000 tokens.

Etapa 3: o loop do agente com carregamento diferido
```python
import json

class DeferredAgent:
def __init__(self, model, system_prompt: str, registry: DeferredToolRegistry):
self.model = model
self.system_prompt = system_prompt
self.registry = registry
self.max_turns = 8
self.search_tool = make_search_tool(registry)

def run(self, user_input: str) -> str:
# Turn 1: Only the search tool is active
self.registry.load([self.search_tool.name])
messages = [
{"role": "system", "content": self.system_prompt},
{"role": "user", "content": user_input},
]

for turn in range(self.max_turns):
tools = self.registry.load(list(self.registry.active_tools))
response = self.model.chat(messages=messages, tools=tools)

if not response.tool_calls:
return response.content  # Final answer

messages.append(response.message)
for call in response.tool_calls:
if call.function.name == "search_tools":
# Agent searched — load the results and re-loop
results = self.registry.search_tool.fn(
**json.loads(call.function.arguments)
)
tool_names = [r["name"] for r in results]
self.registry.load(tool_names)
messages.append({
"role": "tool",
"content": f"Found tools: {[r['name'] for r in results]}. You now have access to these tools. Proceed with your task.",
"tool_call_id": call.id,
})
```
outro:
# Agente usou uma ferramenta carregada
tentar:
```python
args = json.loads(call.function.arguments)
result = self.registry.execute(call.function.name, args)
except Exception as e:
result = f"Error: {e}"
messages.append({
"role": "tool", "content": str(result), "tool_call_id": call.id,
})

self.registry.unload_all()
self.registry.load([self.search_tool.name])

return "Max turns reached."

```

O principal padrão comportamental:

Comece apenas com a pesquisa – o agente não pode ligar para mais nada.
Pesquisas de agentes – recupera nomes de ferramentas que atendem às suas necessidades.
Ferramentas ativadas — o sistema carrega ferramentas correspondentes no conjunto ativo.
Agente usa ferramentas — chama as ferramentas carregadas para realizar a tarefa.
Descarregar após cada sessão — a próxima solicitação começa limpa.
Etapa 4: registro de ferramenta no mundo real

Veja como você registraria ferramentas reais:


```python
registry = DeferredToolRegistry()

registry.register(Tool(
name="get_pull_request",
description="Fetch details of a specific pull request including status, files changed, and review comments.",
parameters={"type": "object", "properties": {
"repo": {"type": "string", "description": "Repository in owner/repo format"},
"pr_number": {"type": "integer", "description": "Pull request number"},
}, "required": ["repo", "pr_number"]},
fn=lambda repo, pr_number: {"status": "open", "files": 12},
category="github",
))

registry.register(Tool(
name="query_database",
description="Execute a read-only SQL query against the analytics database. Only SELECT allowed.",
parameters={"type": "object", "properties": {
"sql": {"type": "string", "description": "SELECT query to execute"},
"limit": {"type": "integer", "description": "Max rows to return (default 100)"},
}, "required": ["sql"]},
fn=lambda sql, limit=100: {"rows": []},
category="database",
))

registry.register(Tool(
name="send_email",
description="Send an email to a recipient with subject and body.",
parameters={"type": "object", "properties": {
"to": {"type": "string", "description": "Recipient email address"},
"subject": {"type": "string"},
"body": {"type": "string"},
}, "required": ["to", "subject", "body"]},
fn=lambda to, subject, body: "Email sent",
category="email",
))

```

Com três ferramentas cadastradas, a função de busca funciona assim:


```python
registry.search("database")
# Returns: [{'name': 'query_database', 'description': '...', 'score': 3}]

registry.search("email newsletter")
# Returns: [{'name': 'send_email', 'description': '...', 'score': 1}]

```

O agente procura por "banco de dados", o sistema carrega query_database, o agente chama, responde e a sessão termina. Total de tokens gastos em definições de ferramentas: ~400 em vez de ~1.200.

Etapa 5: Otimizações de Produção

A versão básica acima funciona. Veja como torná-lo de nível de produção:

Pesquisa semântica (melhor que correspondência de palavras-chave)

A correspondência de palavras-chave perde sinônimos. Um agente que procura por "buscar dados" não encontrará query_database. Use incorporaçãos:


```python
import numpy as np

class SemanticToolSearch:
def __init__(self, registry: DeferredToolRegistry):
from sentence_transformers import SentenceTransformer
self.model = SentenceTransformer("all-MiniLM-L6-v2")
self.registry = registry
self._build_index()

def _build_index(self):
self.tool_names = []
self.embeddings = []
for name, tool in self.registry.all_tools.items():
text = f"{name}: {tool.description}"
self.tool_names.append(name)
self.embeddings.append(self.model.encode(text))
self.index = np.array(self.embeddings)

def search(self, query: str, top_k: int = 5) -> list[str]:
query_embed = self.model.encode(query)
similarities = np.dot(self.index, query_embed)
top_indices = np.argsort(similarities)[::-1][:top_k]
return [self.tool_names[i] for i in top_indices]

```

Agora search("fetch user data") encontra query_database mesmo que a palavra "fetch" não esteja no nome ou na descrição da ferramenta. O modelo embedding tem ~90MB e roda em <10ms na CPU.

Grupos de ferramentas e carregamento hierárquico

Em vez de carregar ferramentas individuais, carregue grupos:


```python
tool_groups = {
"github": ["get_pull_request", "create_issue", "update_changelog"],
"database": ["query_database", "list_tables"],
"email": ["send_email", "list_inbox"],
}

def load_group(group_name: str):
self.registry.load(tool_groups[group_name])

```

Adicione uma meta-ferramenta load_tool_group(group). O agente pesquisa "coisas do GitHub" → carrega todas as três ferramentas do GitHub de uma vez, em vez de descobri-las uma de cada vez. Isso economiza turnos de LLM quando a tarefa abrange várias ferramentas relacionadas.

Aplicação do orçamento no carregamento de ferramentas

Evite que o agente carregue todas as ferramentas pesquisando amplamente:


```python
class LoadBudget:
def __init__(self, max_tools_per_session: int = 5):
self.max_tools = max_tools_per_session
self.loaded_count = 0

def can_load(self, count: int) -> bool:
return self.loaded_count + count <= self.max_tools

def record(self, count: int):
self.loaded_count += count

```

Cinco ferramentas por sessão geralmente são suficientes. Se o agente atingir o limite, ele deverá trabalhar com o que tem — sem mais carregamento.

Economia de token: os números

Aqui está a comparação que importa:

Approach Token Cost LLM transforma precisão
Carregue todas as 40 ferramentas aproximadamente 8.000 por chamada 2-3 62%
Carregamento diferido ~400 por chamada 3-4 88%
Diferido (semântico) ~400 por chamada 2-3 91%

A melhoria da precisão vem da redução do ruído. Quando um LLM vê 40 definições de ferramentas, ele escolhe a errada com mais frequência. Quando vê 3 ferramentas relevantes, a seleção fica mais limpa. GPT-4o e Claude Sonnet mostram ganhos de precisão de 20-30% com carregamento diferido no benchmark ToolRet (avaliação de 43.000 ferramentas).

Quando NÃO usar carregamento diferido

O carregamento diferido não é gratuito. Ele adiciona um turno LLM extra. Pule quando:

Menos de 10 ferramentas — a sobrecarga não compensa a economia.
Aplicativos críticos para latência — o padrão de pesquisa e carregamento adiciona de 1 a 2 segundos.
Protótipos insensíveis a custos — se você estiver apenas testando, carregue tudo e repita rapidamente.

Use-o quando:

Mais de 20 ferramentas – o inchaço dos tokens torna-se significativo.
agentes autônomos — agentes que funcionam sem supervisão precisam de mais proteções, incluindo controle de contexto.
Pilhas pesadas de MCP – conectar-se a mais de 5 servidores MCP significa mais de 50 ferramentas facilmente.
Onde as plataformas gerenciadas lidam com isso

Construir você mesmo essa infraestrutura ensina o padrão. Mas as três partes – busca de ferramentas, carregamento diferido e aplicação do orçamento – são preocupações da plataforma, não da lógica do aplicativo.

Plataformas como Nebula implementam isso nativamente: quando um agente se conecta a vários servidores MCP, a plataforma fornece um recurso search_tools que descobre ferramentas em todos os servidores conectados. O agente começa com a pesquisa, encontra a ferramenta certa, liga para ela e a plataforma gerencia o ciclo de vida. Você define as restrições (máximo de ferramentas por sessão, limites de orçamento) na configuração do agente; a plataforma os aplica.

Conclusões acionáveis
Comece com pesquisa, não com ferramentas. Dê ao seu agente uma metaferramenta que pesquisa o catálogo. É mais barato e preciso do que carregar tudo.
Mantenha as definições de ferramentas curtas. Uma definição de ferramenta de 200 tokens com uma descrição clara supera um ensaio de 500 tokens. O LLM não precisa dos detalhes de implementação.
Carregue grupos, não indivíduos. As ferramentas relacionadas são carregadas juntas - pesquise "GitHub" e obtenha ferramentas de RP, emissão e repo de uma só vez.
Limite seu orçamento de ferramentas. Cinco ferramentas ativas por sessão são suficientes para a maioria das tarefas. Força o agente a ser seletivo.
Use a pesquisa semântica se você tiver mais de 30 ferramentas. A correspondência de palavras-chave degrada rapidamente. Os embeddings lidam com sinônimos naturalmente.
Descarregue após cada sessão. Não deixe que as ferramentas se acumulem nas solicitações. Comece do zero a cada vez.

A tendência é clara: com mais de 177 mil ferramentas públicas no ecossistema MCP, o desafio de engenharia não é mais conectar ferramentas. É escolher quais mostrar ao agente a cada momento. Reduza primeiro, planeje depois - e seu agente terá um desempenho melhor e custará menos.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
Como criar carregamento de ferramentas adiado para agentes de IA em 15 minutos
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção