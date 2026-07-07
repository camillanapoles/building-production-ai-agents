# Por que 90% dos projetos de agentes de IA falham (e os padrões que corrigem isso)

> Artigo #10 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/why-90-of-ai-agent-projects-fail-and-the-patterns-that-fix-it-5bgp)

---

Seu agente trabalha na demonstração. Ele lida perfeitamente com cinco casos de teste. Suas partes interessadas estão impressionadas.

Então chega à produção. Ele alucina um ID de cliente, percorre a mesma chamada de API quarenta vezes, esgota seu orçamento mensal em uma tarde e trava com um erro que ninguém consegue reproduzir porque não há registros.

Você não está sozinho. Um estudo da RAND Corporation descobriu que 80-90% dos projetos de IA nunca passam da prova de conceito. Para agentes de IA — sistemas que realizam ações autônomas e em várias etapas — a taxa de falha é ainda maior porque as consequências da falha não são apenas respostas erradas. São ações erradas.

Mas aqui está a parte que a maioria dos artigos sobre esta estatística erram: as falhas não ocorrem porque “a IA não está pronta”. São falhas arquitetônicas com soluções conhecidas. Depois de estudar dezenas de implantações de agentes de produção e construir a nossa própria, cinco modos de falha são responsáveis ​​por quase todas as mortes de agentes que vimos.

Aqui estão esses cinco modos, com código Python executável para corrigir cada um deles.

Modo de falha nº 1: o antipadrão do agente divino

O erro mais comum é construir um agente que faça tudo. Você começa com um assistente simples, depois aplica o tratamento de e-mail, depois o gerenciamento de calendário, depois a análise de dados e, por fim, a geração de código. Em pouco tempo, seu prompt do sistema tem 10.000 tokens, você tem mais de 20 ferramentas registradas e o modelo está roteando para a ferramenta errada 30% das vezes.

Este é o Agente Divino – um monólito que tenta ser onisciente.

Por que falha: Grandes modelos de linguagem degradam-se de forma previsível à medida que o contexto aumenta. Mais ferramentas significam mais decisões de roteamento e a precisão do roteamento cai de forma não linear com a contagem de ferramentas. Um agente de 5 ferramentas pode rotear corretamente 95% das vezes. Um agente de 25 ferramentas pode atingir 70%. Essa taxa de erro de 30% aumenta em fluxos de trabalhos de várias etapas.

A solução: decomponha-se em agentes especializados.


```python
from dataclasses import dataclass
from enum import Enum

class TaskType(Enum):
EMAIL = \"email\"
CALENDAR = \"calendar\"
CODE = \"code\"
RESEARCH = \"research\"

@dataclass
class AgentSpec:
name: str
task_type: TaskType
tools: list[str]
system_prompt: str
max_tokens: int = 4096

# Each specialist has a tight scope and few tools
email_agent = AgentSpec(
name=\"email-handler\",
task_type=TaskType.EMAIL,
tools=[\"read_inbox\", \"send_email\", \"search_emails\"],
system_prompt=\"You handle email operations. You can read, search, and send emails. Nothing else.\",
max_tokens=2048
)

research_agent = AgentSpec(
name=\"researcher\",
task_type=TaskType.RESEARCH,
tools=[\"web_search\", \"scrape_page\", \"summarize\"],
system_prompt=\"You research topics using web search and summarization. Nothing else.\",
max_tokens=4096
)

def route_to_specialist(task: str, agents: list[AgentSpec]) -> AgentSpec:
\"\"\"Simple keyword router. In production, use an LLM classifier
with \u003C5 options for high accuracy.\"\"\"
task_lower = task.lower()
routing_map = {
TaskType.EMAIL: [\"email\", \"inbox\", \"send\", \"reply\", \"forward\"],
TaskType.CALENDAR: [\"meeting\", \"schedule\", \"calendar\", \"invite\"],
TaskType.CODE: [\"code\", \"debug\", \"function\", \"script\", \"bug\"],
TaskType.RESEARCH: [\"search\", \"find\", \"research\", \"look up\", \"what is\"],
}
for agent in agents:
keywords = routing_map.get(agent.task_type, [])
if any(kw in task_lower for kw in keywords):
return agent
return agents[0]  # fallback to first agent

```

O principal insight: um roteador escolhendo entre quatro agentes especializados é um problema dramaticamente mais simples do que um agente escolhendo entre 20 ferramentas. Cada especialista tem de 3 a 5 ferramentas e um prompt do sistema focado em menos de 500 tokens. A precisão do roteamento permanece acima de 95% e cada especialista tem melhor desempenho porque seu contexto não é diluído.

Abordamos detalhadamente os padrões de orquestração para isso em Orquestração multiagente: um guia para padrões que funcionam.

Modo de falha nº 2: a armadilha do caminho feliz (sem recuperação de erros)

Agentes que funcionam perfeitamente em demonstrações travam na produção porque as demonstrações nunca testam casos de falha. No mundo real, as APIs retornam 429s, as conexões atingem o tempo limite, os serviços externos ficam inativos e as respostas retornam malformadas.

Os dados reais de produção mostram uma imagem clara: as tarefas de automação do navegador falham aproximadamente 30% das vezes devido a problemas de carregamento de página, os limites de taxa atingem 20-25% dos fluxos de trabalhos com uso pesado de API e serviços de terceiros retornam respostas inesperadas em 5-10% das chamadas.

Um agente sem recuperação de erros trata cada falha como fatal. Uma API timeout elimina um workflow inteiro de várias etapas.

A solução: padrão de disjuntor para chamadas de ferramenta de agente.


```python
import time
import random
from typing import Callable, Any
from dataclasses import dataclass, field

@dataclass
class CircuitBreaker:
\"\"\"Prevents cascading failures in agent tool calls.\"\"\"
max_retries: int = 3
base_delay: float = 1.0
max_delay: float = 30.0
failure_threshold: int = 5
reset_timeout: float = 60.0

_failure_count: int = field(default=0, init=False)
_last_failure_time: float = field(default=0.0, init=False)
_state: str = field(default=\"closed\", init=False)  # closed, open, half-open

def call(self, func: Callable, *args, **kwargs) -> Any:
# Check if circuit is open
if self._state == \"open\":
if time.time() - self._last_failure_time > self.reset_timeout:
self._state = \"half-open\"  # Allow one test call
else:
raise RuntimeError(
f\"Circuit open. Tool unavailable. \"
f\"Retry after {self.reset_timeout}s.\"
)
```

# Tentativa com espera exponencial
para tentativa no intervalo (self.max_retries):
tentar:
```python
result = func(*args, **kwargs)
self._on_success()
return result
except Exception as e:
delay = min(
self.base_delay * (2 ** attempt) + random.uniform(0, 1),
self.max_delay
)
if attempt \u003C self.max_retries - 1:
print(f\"Tool call failed (attempt {attempt + 1}): {e}. \"
f\"Retrying in {delay:.1f}s...\")
time.sleep(delay)
else:
self._on_failure()
raise

def _on_success(self):
self._failure_count = 0
self._state = \"closed\"

def _on_failure(self):
self._failure_count += 1
self._last_failure_time = time.time()
if self._failure_count >= self.failure_threshold:
self._state = \"open\"

# Usage: wrap every external tool call
api_breaker = CircuitBreaker(max_retries=3, failure_threshold=5)

def safe_tool_call(tool_fn, *args, fallback=None, **kwargs):
\"\"\"Execute a tool call with circuit breaker protection.\"\"\"
try:
return api_breaker.call(tool_fn, *args, **kwargs)
except RuntimeError:
if fallback:
return fallback(*args, **kwargs)
return {\"error\": \"Tool unavailable\", \"action\": \"escalate_to_human\"}

```

O padrão é simples: repetir com espera para falhas transitórias, desarmar o disjuntor para falhas persistentes e sempre ter um caminho fallback — mesmo que esse fallback seja \"diga ao usuário que você não pode fazer isso agora.\" Um agente que se degrada normalmente é infinitamente mais útil do que aquele que trava silenciosamente.

Modo de falha nº 3: Falência da janela de contexto

Cada mensagem, resultado de ferramenta e etapa da cadeia de pensamento consome tokens. Os agentes que colocam tudo no contexto acabam falindo – atingem o limite de tokens e começam a perder instruções críticas ou, pior, continuam trabalhando, mas com compreensão degradada.

Os sintomas são sutis no início: o agente “esquece” suas restrições de prompt do sistema, começa a alucinar nomes de ferramentas ou dá respostas que contradizem instruções anteriores. Quando você percebe, ele está produzindo resultados não confiáveis ​​há horas.

A solução: memória em camadas com remoção de contexto.


```python
from dataclasses import dataclass, field
from typing import Optional
import json

@dataclass
class MemoryTier:
\"\"\"Three-tier memory prevents context bankruptcy.\"\"\"
# Tier 1: Working memory (always in context)
system_prompt: str = \"\"
current_task: str = \"\"

# Tier 2: Session memory (summarized, pruned)
conversation_summary: str = \"\"
recent_messages: list = field(default_factory=list)
max_recent: int = 10

# Tier 3: Long-term memory (retrieved on demand)
persistent_store: dict = field(default_factory=dict)

def add_message(self, role: str, content: str):
self.recent_messages.append({\"role\": role, \"content\": content})
if len(self.recent_messages) > self.max_recent:
# Summarize oldest messages before evicting
evicted = self.recent_messages[:3]
self._summarize_and_evict(evicted)
self.recent_messages = self.recent_messages[3:]

def _summarize_and_evict(self, messages: list):
\"\"\"Compress old messages into running summary.\"\"\"
# In production, use an LLM to summarize
key_points = []
for msg in messages:
# Extract action items and decisions only
content = msg[\"content\"][:200]  # Truncate
key_points.append(f\"- [{msg['role']}]: {content}\")

addition = \"\
\".join(key_points)
self.conversation_summary += f\"\
```
{adição}\"
# Tamanho do resumo do limite também
se len(self.conversation_summary) > 2000:
```python
self.conversation_summary = self.conversation_summary[-2000:]

def build_context(self, max_tokens: int = 8000) -> list[dict]:
\"\"\"Build context window with priority ordering.\"\"\"
context = []

# Tier 1: Always included (highest priority)
context.append({
\"role\": \"system\",
\"content\": self.system_prompt
})
if self.current_task:
context.append({
\"role\": \"system\",
\"content\": f\"Current task: {self.current_task}\"
})
```

# Camada 2: Resumo + mensagens recentes
se self.conversation_summary:
contexto.append({
```python
\"role\": \"system\",
\"content\": f\"Conversation history summary:\
{self.conversation_summary}\"
})
context.extend(self.recent_messages)

return context

def store_long_term(self, key: str, value: str):
\"\"\"Save to persistent memory (database, file, etc).\"\"\"
self.persistent_store[key] = value

def recall(self, key: str) -> Optional[str]:
\"\"\"Retrieve from long-term memory on demand.\"\"\"
return self.persistent_store.get(key)

```

O princípio: as instruções do sistema e a tarefa atual estão sempre em contexto (Nível 1). A conversa recente é mantida, mas eliminada continuamente (Nível 2). Todo o resto é armazenado externamente e recuperado somente quando necessário (Nível 3). Isso mantém o uso do contexto previsível e evita a degradação slow que mata agentes de longa execução.

Para um mergulho mais profundo nesses padrões, consulte nosso guia Engenharia de Contexto para Agentes de IA.

Modo de falha nº 4: Loops infinitos e espirais de custos

Este é o modo de falha que custa dinheiro real. Um agente fica preso em um ciclo de raciocínio — talvez esteja novamentetentando uma abordagem que falhou, ou continua chamando a mesma ferramenta esperando resultados diferentes, ou sua cadeia de pensamento se transforma em um raciocínio cada vez mais abstrato que nunca converge.

Os números são preocupantes. Um relatório de produção amplamente citado mostrou custos operacionais mensais de US$ 236, com o modelo mais caro (Claude Opus) usado em apenas 1% das solicitações, mas representando uma parcela desproporcional dos gastos. Sem proteções, um único agente fugitivo pode esgotar seu orçamento de API em horas.

A solução: limites de etapas e armadilhas orçamentárias.


```python
import time
from dataclasses import dataclass, field

@dataclass
class AgentBudget:
\"\"\"Kill switch for runaway agents.\"\"\"
max_steps: int = 25
max_cost_usd: float = 1.00
max_runtime_seconds: float = 300.0
max_consecutive_same_tool: int = 3

_step_count: int = field(default=0, init=False)
_total_cost: float = field(default=0.0, init=False)
_start_time: float = field(default=0.0, init=False)
_tool_history: list = field(default_factory=list, init=False)

def start(self):
self._start_time = time.time()
self._step_count = 0
self._total_cost = 0.0
self._tool_history = []

def check_budget(self, tool_name: str = \"\", step_cost: float = 0.0):
\"\"\"Call before every agent step. Raises if budget exceeded.\"\"\"
self._step_count += 1
self._total_cost += step_cost
if tool_name:
self._tool_history.append(tool_name)
```

# Verifique o limite de passos
se self._step_count > self.max_steps:
```python
raise BudgetExceeded(
f\"Step limit reached ({self.max_steps}). \"
f\"Agent terminated to prevent runaway.\"
)
```

# Verifique o limite de custo
se self._total_cost > self.max_cost_usd:
```python
raise BudgetExceeded(
f\"Cost limit reached (${self.max_cost_usd:.2f}). \"
f\"Spent: ${self._total_cost:.4f}\"
)

# Check runtime
elapsed = time.time() - self._start_time
if elapsed > self.max_runtime_seconds:
raise BudgetExceeded(
f\"Runtime limit reached ({self.max_runtime_seconds}s). \"
f\"Elapsed: {elapsed:.1f}s\"
)

# Detect loops: same tool called N times in a row
if len(self._tool_history) >= self.max_consecutive_same_tool:
recent = self._tool_history[-self.max_consecutive_same_tool:]
if len(set(recent)) == 1:
raise BudgetExceeded(
f\"Loop detected: '{recent[0]}' called \"
f\"{self.max_consecutive_same_tool} times consecutively.\"
)

class BudgetExceeded(Exception):
pass

# Usage in your agent loop
budget = AgentBudget(max_steps=25, max_cost_usd=0.50)
budget.start()

for step in agent_steps:
try:
# Estimate cost: ~$0.01 per GPT-4o call with 1K tokens
budget.check_budget(
tool_name=step.tool_name,
step_cost=0.01
)
result = execute_step(step)
except BudgetExceeded as e:
print(f\"Agent stopped: {e}\")
# Graceful shutdown: save state, notify user
save_partial_results()
notify_user(f\"Task incomplete: {e}\")
break

```

Quatro proteções em uma classe: contagem de etapas (evita loops infinitos), limite de custo (evita estouros de orçamento), limite de tempo de execução (evita agentes travados) e detecção de loop (pega o agente martelando a mesma ferramenta). Todo agente de produção precisa dos quatro.

Para obter o tratamento completo sobre a economia dos agentes, consulte Como interromper as espirais de custos dos agentes de IA antes que elas comecem.

Modo de falha nº 5: a caixa preta (sem observabilidade)

Você não pode depurar o que não pode ver. A maioria dos projetos de agente não tem nenhum logging estruturado — quando algo dá errado, a equipe fica vasculhando o stdout, tentando reconstruir o que o agente fez e por quê.

Este não é apenas um problema de depuração. É um problema de confiança. Se você não consegue explicar por que seu agente executou uma ação, você não pode confiar nele tarefas importantes. E se a liderança não conseguir ver o que o agente está a fazer, não o financiará após o piloto.

A correção: etapa estruturada logging.


```python
import json
import time
from dataclasses import dataclass, field, asdict
from typing import Optional

@dataclass
class StepLog:
\"\"\"Structured log for every agent step.\"\"\"
step_number: int
timestamp: float
action: str  # \"llm_call\", \"tool_call\", \"decision\"
tool_name: Optional[str] = None
input_summary: str = \"\"  # Truncated input
output_summary: str = \"\"  # Truncated output
tokens_used: int = 0
cost_usd: float = 0.0
duration_ms: float = 0.0
error: Optional[str] = None

def to_dict(self) -> dict:
return {k: v for k, v in asdict(self).items() if v is not None}

@dataclass
class AgentTracer:
\"\"\"Collects structured traces for debugging and auditing.\"\"\"
task_id: str
agent_name: str
steps: list[StepLog] = field(default_factory=list)
_start_time: float = field(default=0.0, init=False)

def start(self):
self._start_time = time.time()
print(json.dumps({
\"event\": \"agent_start\",
\"task_id\": self.task_id,
\"agent\": self.agent_name,
\"timestamp\": self._start_time
}))

def log_step(self, action: str, **kwargs) -> StepLog:
step = StepLog(
step_number=len(self.steps) + 1,
timestamp=time.time(),
action=action,
**kwargs
)
self.steps.append(step)
# Emit structured log (ship to your logging pipeline)
print(json.dumps(step.to_dict()))
return step

def finish(self, status: str = \"completed\"):
duration = time.time() - self._start_time
total_cost = sum(s.cost_usd for s in self.steps)
total_tokens = sum(s.tokens_used for s in self.steps)

summary = {
\"event\": \"agent_complete\",
\"task_id\": self.task_id,
\"agent\": self.agent_name,
\"status\": status,
\"total_steps\": len(self.steps),
\"total_duration_s\": round(duration, 2),
\"total_tokens\": total_tokens,
\"total_cost_usd\": round(total_cost, 4),
\"errors\": [s.error for s in self.steps if s.error]
}
print(json.dumps(summary))
return summary

# Usage
tracer = AgentTracer(task_id=\"task_abc123\", agent_name=\"email-handler\")
tracer.start()

# Log each step as the agent works
tracer.log_step(
action=\"tool_call\",
tool_name=\"read_inbox\",
input_summary=\"query: unread from last 24h\",
output_summary=\"Found 12 emails\",
tokens_used=450,
cost_usd=0.003,
duration_ms=1200
)

tracer.log_step(
action=\"llm_call\",
input_summary=\"Classify 12 emails by priority\",
output_summary=\"3 urgent, 5 normal, 4 low\",
tokens_used=800,
cost_usd=0.006,
duration_ms=2100
)

result = tracer.finish(status=\"completed\")
# Output: {\"event\": \"agent_complete\", \"total_steps\": 2,
#          \"total_cost_usd\": 0.009, \"errors\": []}

```

Cada etapa recebe uma entrada de registro estruturada com o que aconteceu, quanto custou e quanto tempo demorou. Quando algo quebra às 3 da manhã, você não precisa adivinhar – você reproduz o rastreamento e vê exatamente qual etapa falhou e por quê. É também assim que você constrói os conjuntos de dados de avaliação que permitem melhorar o agente ao longo do tempo.

O espectro de confiabilidade: onde está seu agente?

Nem todos os agentes precisam ser totalmente autônomos. Ajuda pensar na confiabilidade como um espectro:

Nível\tDescrição\tExemplo\tO que é necessário
L1\tDemo-impressionante, falha na produção\tA maioria dos projetos de hackathon\tNada — este é o padrão
L2\tFunciona na maioria das vezes, precisa de verificações humanas\tChatGPT com chamada de função\tTratamento básico de erros
L3\tPronto para produção para tarefas restritas\tAssistentes de codificação com bom escopo\tTodas as 5 correções acima, escopo restrito
L4\tOperação autônoma confiável\tRaro — requer todos os padrões + avaliações\tObservabilidade total, conjunto de avaliações, controles de orçamento

Autoavaliação rápida – responda honestamente:

```python
Does your agent have fewer than 8 tools? (Scope)
Does every tool call have retry + fallback logic? (Error recovery)
Can you tell me how many tokens your agent uses per task? (Observability)
Is there a hard kill switch for cost/steps? (Budget control)
Does conversation context get pruned automatically? (Memory management)
```

Se você respondeu “não” a três ou mais, seu agente provavelmente está em L1-L2, independentemente da qualidade da demonstração. A boa notícia: cada correção é independente. Comece com o que está causando mais problemas – geralmente recuperação de erros (nº 2) ou controles de orçamento (nº 4) – e vá avançando.

Comece com o agente de produção mínimo viável

Você não precisa de todos os cinco padrões no primeiro dia. O agente de produção mínimo viável tem três:

Escopo especializado — Um agente, um trabalho, no máximo 3 a 5 ferramentas
Recuperação de erros — Disjuntor em todas as chamadas externas
Limite de orçamento – limites rígidos de etapas, custo e tempo de execução

Essa combinação lida com as três interrupções de produção mais comuns: roteamento errado de ferramentas, falhas em cascata de APIs instáveis ​​e custos descontrolados. Adicione gerenciamento de contexto e observabilidade à medida que você dimensiona.

Plataformas como o Nebula incorporam esses padrões — delegação de agentes com escopo definido, novas tentativas automáticas, orçamentos de etapas e rastreamentos de execução — para que você possa ignorar a infraestrutura e se concentrar na lógica real do seu agente. Mas quer você construa ou compre, os padrões são os mesmos.

A taxa de fracasso de 90% é real, mas não é uma sentença de morte. É um reflexo de equipes que ignoram práticas de engenharia conhecidas. Os padrões existem. Eles não são complicados. A questão é se você irá implementá-los antes que seu agente entre em produção ou depois que ele trave.

Qual modo de falha eliminou seu último projeto de agente? Deixe nos comentários - estou curioso para saber qual é o mais comum na natureza.

Isso faz parte da série Building Production AI Agents. Anteriormente: Como testar agentes de IA antes que eles queimem seu orçamento, Orquestração multiagente: padrões que funcionam, Engenharia de contexto para agentes de IA e Como impedir espirais de custos de agentes de IA.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
Por que 90% dos projetos de agentes de IA falham (e os padrões que corrigem isso)
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção