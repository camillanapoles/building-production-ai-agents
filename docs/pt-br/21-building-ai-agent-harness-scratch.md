# Construindo um chicote de agente de IA do zero: a arquitetura entre LLM e agente

> Artigo #21 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/building-an-ai-agent-harness-from-scratch-the-architecture-between-llm-and-agent-3a1o)

---

Todo mundo fala sobre o modelo. Ninguém fala sobre o arnês.

Dê a Claude Sonnet ou GPT-4o uma interface de bate-papo e você terá uma IA conversacional. Envolva-o em um loop que pode chamar ferramentas externas, manter o estado entre turnos, impor limites orçamentários e validar seus próprios resultados – e você terá um agente. A diferença não é o LLM. É tudo em torno do LLM.

A equipe da AWS publicou um guia sobre “aproveitamento de agentes” esta semana, e isso me fez pensar: a maioria dos tutoriais mostra como chamar um LLM ou como registrar uma ferramenta. Quase nenhum deles mostra a camada de orquestração que faz com que essas peças individuais se comportem como um sistema coerente.

Eu construí agentes que funcionam de forma autônoma na infraestrutura de produção 24 horas por dia, 7 dias por semana. Os erros que cometi no início não foram sobre escolher o modelo errado. Eles queriam pular o cinto de segurança – presumindo que o modelo iria “simplesmente descobrir”. O aproveitamento é o que torna um agente confiável, e a confiabilidade é a única métrica que importa quando você passa da fase de demonstração.

Veja como construir um do zero.

O que é realmente um chicote de agente?

Um chicote de agente é o ambiente de execução que fica entre o usuário e o LLM. Não é o prompt. Não é o modelo. É a infraestrutura que:

Gerencia o ciclo de conversação – receber informações, chamar o modelo, rotear chamadas de ferramentas, retornar resultados, repetir até o encerramento
Ferramentas de registro e despacho — mantendo um catálogo de funções que podem ser chamadas, validando argumentos, executando-os com segurança e retornando resultados estruturados
Mantém a memória – armazenando o histórico de conversas, injetando contexto relevante, compactando mensagens antigas para permanecer dentro dos limites do contexto
Aplica proteções - limitando orçamento de tokens, limitando contagens de chamadas de ferramentas, evitando loops infinitos, bloqueando ações perigosas
Lida com falhas — tentar novamenteerros transitórios, degradar normalmente quando uma ferramenta está indisponível, escalar para revisão humana quando a confiança é baixa

Sem equipamento, você tem uma chamada de API sem estado. Com um arnês, você tem um sistema.

O chicote mínimo do agente

Vamos começar com a menor versão útil. Um chicote precisa de três coisas: uma interface de modelo, um registro de ferramenta e um loop.


```python
import json
from typing import Callable, Any
from dataclasses import dataclass, field

@dataclass
class Tool:
name: str
description: str
parameters: dict  # JSON Schema
fn: Callable

class AgentHarness:
def __init__(self, model, system_prompt: str = \"\"):
self.model = model
self.system_prompt = system_prompt
self.tools: dict[str, Tool] = {}
self.max_iterations = 10

def register_tool(self, tool: Tool):
self.tools[tool.name] = tool

def tool_list(self) -> list[dict]:
return [
{\"type\": \"function\", \"function\": {
\"name\": t.name, \"description\": t.description,
\"parameters\": t.parameters,
}}
for t in self.tools.values()
]

def run(self, user_input: str) -> str:
messages = [
{\"role\": \"system\", \"content\": self.system_prompt},
{\"role\": \"user\", \"content\": user_input},
]
for i in range(self.max_iterations):
response = self.model.chat(
messages=messages, tools=self.tool_list() if self.tools else None,
)
if not response.tool_calls:
return response.content
messages.append(response.message)
for call in response.tool_calls:
tool = self.tools.get(call.function.name)
if not tool:
result = f\"Error: Unknown tool '{call.function.name}'\"
else:
try:
args = json.loads(call.function.arguments)
result = tool.fn(**args)
except Exception as e:
result = f\"Error: {type(e).__name__}: {e}\"
messages.append({\"role\": \"tool\", \"content\": str(result), \"tool_call_id\": call.id})
return \"Max iterations reached.\"


That's the skeleton. It loops: call model, check for tool calls, execute, feed back. Seven lines of core logic. It works for demos. It breaks in production. Let's see why.

Problem 1: The Tool Registry Lies
```

Você registra uma ferramenta, o agente a chama e ela trava porque a validação de entrada está errada. A descrição da ferramenta prometeu determinados parâmetros, o modelo cumpriu, mas a função subjacente tem requisitos mais rígidos. Isso não é culpa do modelo – é um problema de aproveitamento: o registro da ferramenta deve ser validado antes do envio.


```python
class ToolRegistry:
def __init__(self):
self.tools: dict[str, Tool] = {}
self.call_counts: dict[str, int] = {}

def register(self, tool: Tool):
self.tools[tool.name] = tool
self.call_counts[tool.name] = 0

def validate_call(self, tool_name: str, arguments: dict) -> tuple[bool, str]:
if tool_name not in self.tools:
return False, f\"Unknown tool: {tool_name}\"
schema = self.tools[tool_name].parameters
for field in schema.get(\"required\", []):
if field not in arguments:
return False, f\"Missing required parameter: {field}\"
for arg_name, arg_value in arguments.items():
if arg_name not in schema.get(\"properties\", {}):
return False, f\"Unexpected parameter: {arg_name}\"
return True, \"OK\"

def execute(self, tool_name: str, arguments: dict) -> Any:
self.call_counts[tool_name] += 1
return self.tools[tool_name].fn(**arguments)

```

O registro atua como um guardião e não apenas como um despachante. Antes de qualquer ferramenta ser acionada, o chicote valida a existência, os campos obrigatórios, a correção do tipo e os parâmetros alucinados. Isso detecta 60-70% dos erros de chamada de ferramenta antes que eles atinjam o código do aplicativo.

Problema 2: O inchaço da memória mata o contexto

Dez vezes, a conversa contém o prompt original, quatro pares de chamada/resposta de ferramenta e um rascunho parcial. A janela de contexto está sendo preenchida. No turno 20, o modelo começa a esquecer o prompt do sistema. A solução é o gerenciamento inteligente de contexto: comprima o que você não precisa, preserve o que você faz.


```python
import tiktoken
from dataclasses import dataclass

@dataclass
class MemoryConfig:
max_context_tokens: int = 64_000
keep_recent_messages: int = 8
always_preserve_system: bool = True

class AgentMemory:
def __init__(self, config: MemoryConfig):
self.config = config
self.messages: list[dict] = []
self.encoder = tiktoken.encoding_for_model(\"gpt-4o\")

def add(self, role: str, content: str, **kwargs):
self.messages.append({\"role\": role, \"content\": content, **kwargs})

def get_messages(self) -> list[dict]:
total = sum(len(self.encoder.encode(m.get(\"content\", \"\"))) + 4 for m in self.messages)
if total \u003C= self.config.max_context_tokens:
return self.messages
return self._compress()

def _compress(self) -> list[dict]:
keep = self.config.keep_recent_messages
system_msg = None
if self.config.always_preserve_system:
system_msgs = [m for m in self.messages if m[\"role\"] == \"system\"]
if system_msgs:
system_msg = system_msgs[0]
recent = self.messages[-keep:]
old = self.messages[:-keep]
if not old:
return [system_msg] + recent if system_msg else recent
# Summarize old messages (in production, call a cheap model like Haiku)
old_text = \"\
\".join(f\"[{m['role']}]: {m.get('content', '')[:200]}\" for m in old)
summary = \" | \".join([line[:100] for line in old_text.split(\"\
\") if any(kw in line.lower() for kw in [\"tool:\", \"result:\", \"error:\"])][:10])
compressed = [{\"role\": \"system\", \"content\": f\"[EARLIER CONTEXT: {summary}]\"}]
if system_msg:
compressed = [system_msg] + compressed
compressed.extend(recent)
return compressed

```

Trate a janela de contexto como a memória do sistema operacional: as mensagens recentes são o seu cache ativo, as mensagens antigas são o espaço de troca e o prompt do sistema é a memória do kernel - nunca a pagina.

Problema 3: O Loop funciona para sempre

O modelo entra numa espiral de raciocínio. Ele chama search_database, obtém um resultado, chama novamente com parâmetros ligeiramente diferentes e repete indefinidamente. As fichas se acumulam. A aplicação do orçamento é a proteção mais crítica e pertence ao sistema, e não ao alerta.


```python
from dataclasses import dataclass
import time

@dataclass
class BudgetConfig:
max_tokens: int = 30_000
max_tool_calls: int = 25
max_time_seconds: float = 300.0
max_per_tool_calls: int = 5

class BudgetEnforcer:
def __init__(self, config: BudgetConfig):
self.config = config
self.tokens_used = 0
self.tool_calls_total = 0
self.tool_calls_per_tool: dict[str, int] = {}
self.start_time = time.time()

def record_tokens(self, input_tokens: int, output_tokens: int):
self.tokens_used += input_tokens + output_tokens

def record_tool_call(self, tool_name: str):
self.tool_calls_total += 1
self.tool_calls_per_tool[tool_name] = self.tool_calls_per_tool.get(tool_name, 0) + 1

def check(self) -> str | None:
if self.tokens_used >= self.config.max_tokens:
return f\"Token budget exceeded: {self.tokens_used} (limit {self.config.max_tokens})\"
if self.tool_calls_total >= self.config.max_tool_calls:
return f\"Tool call budget exceeded: {self.tool_calls_total}\"
if time.time() - self.start_time >= self.config.max_time_seconds:
return \"Time budget exceeded\"
for tool, count in self.tool_calls_per_tool.items():
if count >= self.config.max_per_tool_calls:
return f\"Per-tool limit: '{tool}' called {count} times\"
return None

```

Quatro orçamentos, qualquer um dos quais interrompe o agente antes da espiral de custos: orçamento de token, orçamento de chamada de ferramenta, orçamento de tempo e orçamento por ferramenta.

Problema 4: Erros engolidos, não tratados

Uma chamada de ferramenta gera ConnectionError. O chicote o detecta, retorna “Error: ConnectionError” e o modelo fica confuso. Ele não sabe se deve tentar novamente, tentar uma ferramenta diferente ou desistir. A formatação de erros é um problema de design do agente. O modelo precisa de mensagens de erro estruturadas que informem o que deu errado e o que fazer.


```python
from enum import Enum
from dataclasses import dataclass

class ErrorType(Enum):
TRANSIENT = \"transient\"
PERMANENT = \"permanent\"
UNAVAILABLE = \"unavailable\"

@dataclass
class ToolError:
error_type: ErrorType
message: str
suggestion: str

def format_tool_error(error: ToolError) -> str:
parts = [f\"[TOOL ERROR: {error.error_type.value.upper()}]\"]
parts.append(error.message)
if error.suggestion:
parts.append(f\"Suggested action: {error.suggestion}\")
return \"\
\".join(parts)


Examples:

Transient: Rate limit hit → \"Retry with different parameters or try an alternative tool.\"
Permanent: DELETE query rejected → \"Use SELECT queries to read data instead.\"
Unavailable: Weather service down → \"Inform the user data is unavailable.\"

A bare exception traceback tells the model nothing. A structured error with a suggested action gives it a decision tree.

Problem 5: The Harness Has No State
```

O chicote mínimo não tem estado entre as execuções. Para persistência entre sessões, você precisa de uma camada de estado:


```python
import json
import sqlite3
from datetime import datetime, UTC

class AgentState:
def __init__(self, db_path: str = \"agent_state.db\"):
self.db = sqlite3.connect(db_path)
self.db.execute(\"\"\"CREATE TABLE IF NOT EXISTS sessions (
session_id TEXT PRIMARY KEY, created_at TEXT,
last_active TEXT, user_id TEXT)\"\"\")
self.db.execute(\"\"\"CREATE TABLE IF NOT EXISTS tool_invocations (
id INTEGER PRIMARY KEY AUTOINCREMENT,
session_id TEXT, turn_number INTEGER,
tool_name TEXT, arguments TEXT, result TEXT,
success INTEGER, duration_ms INTEGER, timestamp TEXT)\"\"\")
self.db.commit()

def create_session(self, session_id: str, user_id: str):
self.db.execute(
\"INSERT INTO sessions VALUES (?, ?, ?, ?)\",
(session_id, datetime.now(UTC).isoformat(), datetime.now(UTC).isoformat(), user_id))
self.db.commit()

def record_tool_invocation(self, session_id: str, turn: int,
tool: str, args: dict, result: str,
success: bool, duration_ms: int):
self.db.execute(
\"INSERT INTO tool_invocations VALUES (NULL, ?, ?, ?, ?, ?, ?, ?, ?)\",
(session_id, turn, tool, json.dumps(args), result,
int(success), duration_ms, datetime.now(UTC).isoformat()))
self.db.commit()

def get_analytics(self, session_id: str) -> dict:
total = self.db.execute(\"SELECT COUNT(*) FROM tool_invocations WHERE session_id = ?\", (session_id,)).fetchone()[0]
rate = self.db.execute(\"SELECT AVG(success) FROM tool_invocations WHERE session_id = ?\", (session_id,)).fetchone()[0] or 0
return {\"total_invocations\": total, \"success_rate\": round(rate * 100, 1)}

```

A camada de estado oferece persistência de sessão, logs de auditoria de invocação de ferramentas e análises integradas – essenciais para depurar sessões com falha.

A Arquitetura Completa

Todas as cinco peças se encaixam:


Entrada do usuário
▼
┌───────────────────────────────┐
│ Budget Enforcer │ ← Verificações antes de cada iteração
├───────────────────────────────┤
│ Memória do Agente │ ← Compacta o contexto antigo
├───────────────────────────────┤
│ Chamada LLM │ ← Com definições de ferramentas
├─────────────────┬─────────────┤
│ chamadas de ferramentas?   │ não → retornar
├─────────────────┤
│ Registro de Ferramenta │ ← Esquema + validação de tipo
├───────────────────────────────┤
│ Execução Segura │ ← Erros estruturados com sugestões
├───────────────────────────────┤
│ Estado do Agente │ ← Turno de registro + invocação de ferramenta
└───────────────────────────────┘
voltar atrás


Cada componente tem uma única responsabilidade. O arnês os coordena. O modelo é apenas um nó no gráfico.

Onde as plataformas gerenciadas se encaixam

Construir esse equipamento do zero ensina exatamente o que está envolvido. Mas os cinco componentes — registro de ferramentas, gerenciamento de memória, aplicação de orçamento, tratamento de erros e persistência de estado — são infraestrutura, não lógica de negócios. Eles são idênticos quer você esteja criando um agente GitHub, um agente de conteúdo ou um agente de suporte ao cliente.

Plataformas como Nebula abstraem exatamente essa camada. Você define as ferramentas (expostas automaticamente ao MCP), o prompt do sistema e restrições como iterações máximas e orçamento de tokens. A plataforma cuida de todos os recursos: validação de ferramenta, compactação de contexto, rastreamento de orçamento, formatação de erros e persistência de sessão. Cada execução do agente é rastreada de ponta a ponta com atribuição de custos, e o painel de observabilidade mostra distribuições de chamadas de ferramentas, taxas de sucesso e consumo de orçamento em tempo real.

Você se concentra no que o agente faz. A plataforma garante que você veja quando algo dá errado.

Conclusões acionáveis

Comece com o loop, não com o modelo. O padrão chamar-observar-decidir-repetir é fundamental. Escolha qualquer LLM capaz e concentre-se em acertar o equipamento.

Valide as chamadas de ferramenta antes do envio. A validação do esquema detecta 60-70% dos erros antes que eles atinjam o código do aplicativo.

Comprima o contexto agressivamente. Use um padrão de hot-cache: mantenha as mensagens recentes, resuma as antigas, preserve o prompt do sistema.

Aplique orçamentos em código, não em prompts. Um campo max_iterations no seu prompt é uma sugestão. Um BudgetEnforcer que interrompe a execução é uma garantia.

Estruture seus erros. Classifique os erros como transitórios (nova tentativa), permanentes (redirecionamento) ou indisponíveis (degradação normal), sempre com uma sugestão de ação.

Registre tudo. Invocações de ferramentas com argumentos, resultados, durações e status de sucesso. Quando uma sessão dá errado, os logs são a única maneira de reconstruir o que aconteceu.

Construa primeiro o chicote e depois otimize o modelo. Um GPT-3.5 bem aproveitado sempre supera um GPT-4o não aproveitado.

O arnês do agente não é glamoroso. Mas é a diferença entre um agente que trabalha uma vez em um notebook e outro que trabalha às 2h de uma terça-feira, quando ninguém está olhando. Construa corretamente e o modelo se tornará a parte menos interessante do seu sistema.

Este artigo faz parte da série Building Production AI Agents em Dev.to.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
Construindo um chicote de agente de IA do zero: a arquitetura entre LLM e agente
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção