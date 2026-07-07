# Como impedir espirais de custos de agentes de IA antes que elas comecem

> Artigo #4 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/how-to-stop-ai-agent-cost-spirals-before-they-start-2k1a)

---

Você acorda com uma conta OpenAI de US$ 500. Seu agente correu durante a noite, realizando uma tarefa de pesquisa que deveria levar dois minutos. Cada iteração reenviou o histórico completo da conversa, tentou novamente chamadas de ferramenta com falha três vezes cada e usou GPT-4 para cada etapa – incluindo aquelas que apenas formataram JSON.

Essa é a espiral de custos dos agentes de IA e atinge quase todas as equipes que enviam agentes para produção. O padrão é previsível: janela de contextoincha, novas tentativas tempestades aumentam e cadeias de ferramentas ilimitadas drenam orçamentos enquanto você dorme.

A maioria dos guias ensina como cortar custos após o dano. Este artigo adota a abordagem oposta. Aqui estão cinco padrões de otimização de custos de agentes de IA que evitam o início de espirais - cada um com código Python de produção que você pode colocar em sua pilha hoje mesmo.

Padrão 1: Orçamentos de token por tarefa

O controle de custos mais simples é um teto rígido. Antes de iniciar qualquer execução do agente, defina um orçamento de token. Quando o orçamento se esgota, a corrida é interrompida – sem exceções.

A maioria das equipes ignora isso porque presume que os custos são imprevisíveis. Eles não são. Após algumas execuções, você pode estimar o uso de token esperado para cada tipo de tarefa. Defina o orçamento em 3x o uso esperado para lidar com a variação e você pegará fugas sem bloquear o trabalho legítimo.


```python
class TokenBudget:
def __init__(self, max_tokens: int):
self.max_tokens = max_tokens
self.used = 0

def track(self, input_tokens: int, output_tokens: int):
self.used += input_tokens + output_tokens
if self.used >= self.max_tokens:
raise BudgetExceeded(
f"Token budget exhausted: {self.used}/{self.max_tokens}"
)

@property
def remaining(self) -> int:
return max(0, self.max_tokens - self.used)


class BudgetExceeded(Exception):
pass


# Usage: wrap your LLM calls
budget = TokenBudget(max_tokens=50_000)  # ~$0.50 at GPT-4 pricing

for step in agent.run(task):
response = llm.chat(step.messages)
budget.track(
input_tokens=response.usage.prompt_tokens,
output_tokens=response.usage.completion_tokens,
)
# BudgetExceeded fires automatically if limit is hit

```

O principal insight: orçamentos de tokens devem ser por tarefa, não globais. Uma tarefa de pesquisa pode precisar de 100 mil tokens. Uma tarefa de formatação precisa de 5K. Um único orçamento global mascara os dispendiosos valores discrepantes. Defina orçamentos por tipo de tarefa e você detectará desperdícios imediatamente.

Regra prática: estime os tokens esperados, multiplique por 3 e use isso como teto. Aperte com o tempo à medida que você coleta dados.

Padrão 2: Roteamento de modelo em camadas

Nem todas as etapas do agente precisam do seu modelo mais caro. A maioria dos fluxos de trabalhos são 60-70% de tarefas rotineiras - classificação, formatação, extração simples - que um modelo barato lida perfeitamente.

A solução é um roteador que classifica cada etapa e escolhe o modelo mais barato que pode lidar com isso de maneira confiável:


```python
import openai

MODEL_TIERS = {
"flash": "gpt-4o-mini",       # $0.15/$0.60 per 1M tokens
"standard": "gpt-4o",          # $2.50/$10 per 1M tokens
"complex": "o3-mini",          # $1.10/$4.40 per 1M tokens (reasoning)
}

def route_model(task_description: str, requires_reasoning: bool = False) -> str:
"""Pick the cheapest model that can handle the task."""
if requires_reasoning:
return MODEL_TIERS["complex"]

low_complexity_signals = [
"format", "extract", "classify", "summarize",
"parse", "convert", "list", "filter",
]
task_lower = task_description.lower()
if any(signal in task_lower for signal in low_complexity_signals):
return MODEL_TIERS["flash"]

return MODEL_TIERS["standard"]


# Example: route based on what the agent is doing
model = route_model("extract email addresses from this text")
# Returns: gpt-4o-mini (flash tier — 17x cheaper than standard)

model = route_model("debug this race condition", requires_reasoning=True)
# Returns: o3-mini (complex tier — reasoning needed)

```

As economias são dramáticas. Se 60% das etapas do seu agente forem tarefas de nível flash, você reduzirá esses custos em 17x. Em uma conta de agente de US$ 1.000/mês, são cerca de US$ 500 economizados com a alteração de algumas linhas da lógica de roteamento.

Para sistemas de produção, considere um classificador leve que examine o prompt e as rotas automaticamente. O próprio classificador é executado na camada flash – custando frações de centavo por classificação e economizando dólares por chamada roteada.

Padrão 3: remoção de janela de contexto

O inchaço da janela de contexto é o assassino silencioso do orçamento. Cada chamada de agente envia o prompt do sistema, histórico completo de conversas, esquemas de ferramentas e documentos recuperados. Um único turno pode atingir 30.000 tokens de entrada – e seu agente faz dezenas de turnos por tarefa.

Três estratégias de poda, ordenadas por esforço de implementação:

Janela deslizante — mantenha apenas as últimas N mensagens. Simples e eficaz para tarefas onde o contexto recente é mais importante do que o histórico completo.

Compressão de resumo – a cada K giros, comprima a conversa em um resumo. Este é o ponto ideal para a maioria das cargas de trabalho dos agentes:


```python
def compress_history(
messages: list[dict],
llm_client,
keep_recent: int = 4,
max_summary_tokens: int = 300,
) -> list[dict]:
"""Compress old messages into a summary, keep recent ones intact."""
if len(messages) <= keep_recent:
return messages

old_messages = messages[:-keep_recent]
recent_messages = messages[-keep_recent:]

summary_response = llm_client.chat.completions.create(
model="gpt-4o-mini",  # Use cheap model for summarization
messages=[
{
"role": "system",
"content": (
"Summarize this conversation in under "
f"{max_summary_tokens} tokens. "
"Preserve key decisions, results, and context."
),
},
*old_messages,
],
max_tokens=max_summary_tokens,
)

summary = summary_response.choices[0].message.content
return [
{"role": "system", "content": f"Previous context: {summary}"},
*recent_messages,
]


Relevant-only retrieval — instead of injecting all tool results into context, store them in a scratchpad and retrieve only what's relevant to the current step. This works best for agents with many tool calls.
```

Os números: uma conversa de 20 turnos com histórico completo carrega cerca de 50 mil tokens de contexto. Após a compactação do resumo, isso cai para aproximadamente 5K – uma redução de 90% nos tokens de entrada. Com o preço do GPT-4o, isso economiza cerca de US$ 0,11 por ciclo de compactação. Em centenas de execuções diárias de agentes, o resultado aumenta rapidamente.

Padrão 4: Disjuntores para Loops de Agente

O padrão de custo mais perigoso é o loop infinito. Um agente encontra um erro, tenta novamente, encontra o mesmo erro, tenta novamente – sempre enviando o contexto completo. Sem um disjuntor, uma única tarefa travada pode consumir todo o seu orçamento diário.

Os disjuntores detectam comportamento descontrolado e interrompem o circuito antes que ele esgote sua carteira:


```python
import time
from dataclasses import dataclass, field


@dataclass
class CircuitBreaker:
max_steps: int = 25
max_cost_usd: float = 2.00
max_consecutive_errors: int = 3
_step_count: int = field(default=0, init=False)
_total_cost: float = field(default=0.0, init=False)
_consecutive_errors: int = field(default=0, init=False)

def record_step(self, cost_usd: float, success: bool):
self._step_count += 1
self._total_cost += cost_usd

if success:
self._consecutive_errors = 0
else:
self._consecutive_errors += 1

self._check_breakers()

def _check_breakers(self):
if self._step_count >= self.max_steps:
raise CircuitOpen(
f"Step limit reached: {self._step_count}/{self.max_steps}"
)
if self._total_cost >= self.max_cost_usd:
raise CircuitOpen(
f"Cost limit reached: ${self._total_cost:.2f}/${self.max_cost_usd:.2f}"
)
if self._consecutive_errors >= self.max_consecutive_errors:
raise CircuitOpen(
f"Error streak: {self._consecutive_errors} consecutive failures"
)


class CircuitOpen(Exception):
pass


# Usage
breaker = CircuitBreaker(max_steps=25, max_cost_usd=2.00)

for step in agent.run(task):
try:
result = execute_step(step)
cost = calculate_step_cost(result)
breaker.record_step(cost_usd=cost, success=True)
except StepError as e:
breaker.record_step(cost_usd=cost, success=False)
# CircuitOpen fires after 3 consecutive failures

```

As três condições do disjuntor – contagem de etapas, teto de custo e faixas de erros – detectam diferentes modos de falha. Os limites de passos capturam loops infinitos. Os tetos de custos abrangem execuções caras, mas tecnicamente bem-sucedidas. Sequências de erros capturam agentes que estão travados, mas continuam repetindo.

Quando o circuito abrir, não falhe silenciosamente. Registre o estado da tarefa, notifique a equipe e coloque a tarefa na fila para revisão humana. Os US$ 2 que você gastou para acertar o disjuntor não são nada comparados aos US$ 200 que você gastaria sem ele.

Padrão 5: Resultados da Ferramenta Determinística de Cache

Muitas chamadas de ferramenta de agente sempre retornam o mesmo resultado. Leituras de arquivos, pesquisas de API para dados estáticos, verificações de configuração — elas não mudam entre as chamadas, mas os agentes as executam novamente a cada execução.

Um cache simples com reconhecimento de tempo elimina as chamadas redundantes:


```python
import hashlib
import json
import time
from functools import wraps


def cached_tool(ttl_seconds: int = 3600):
"""Cache tool results based on input arguments."""
cache: dict[str, tuple[float, any]] = {}

def decorator(func):
@wraps(func)
def wrapper(*args, **kwargs):
key = hashlib.sha256(
json.dumps({"args": args, "kwargs": kwargs}, sort_keys=True, default=str).encode()
).hexdigest()

if key in cache:
timestamp, result = cache[key]
if time.time() - timestamp < ttl_seconds:
return result  # Cache hit — zero API cost

result = func(*args, **kwargs)
cache[key] = (time.time(), result)
return result

return wrapper
return decorator


# Apply to your agent's tools
@cached_tool(ttl_seconds=3600)  # Cache for 1 hour
def lookup_user(user_id: str) -> dict:
return database.query(f"SELECT * FROM users WHERE id = '{user_id}'")


@cached_tool(ttl_seconds=86400)  # Cache for 24 hours
def get_config(key: str) -> str:
return config_service.get(key)

```

# Primeira chamada: acessa o banco de dados. Segunda chamada: retorna o resultado armazenado em cache.
lookup_user("usr_123") # consulta ao banco de dados
lookup_user("usr_123") # Acerto no cache — instantâneo, gratuito


Defina o TTL com base nos requisitos de atualização de dados: 5 minutos para dados voltados ao usuário, 1 hora para dados de referência, 24 horas para configuração. Mesmo uma estratégia de cache conservadora normalmente elimina de 30 a 50% das chamadas de ferramentas redundantes em fluxos de trabalhos de agentes.

O que não armazenar em cache: qualquer coisa que seja sensível ao tempo (preços de ações, status ao vivo), mutações específicas do usuário (operações de gravação) ou resultados que dependam do estado externo que muda com frequência.

Juntando tudo: a pilha de agentes conscientes dos custos

Esses cinco padrões se unem em uma arquitetura de custos de defesa profunda:

Capturas de economia típicas de esforço de padrão de camada
1 Token Orçamentos 30 min 10-20% Tarefas descontroladas
2 Roteamento de modelo 1-2 horas 30-50% de gasto excessivo do modelo
3 Poda de Contexto 2-4 horas 15-30% Inchaço de Contexto
4 Disjuntores 1-2 horas 5-15% Loops infinitos
5 Cache de resultados 1-2 horas 10-20% Chamadas redundantes

Comece com roteamento de modelo e orçamentos de tokens. Cobrem 80% dos problemas de custos com o mínimo esforço de implementação. Adicione remoção de contexto quando seus agentes lidarem com conversas em vários turnos. Adicione disjuntoress antes que qualquer agente seja executado sem supervisão. Adicione o cache por último – é o mais situacional.

A ordem é importante. Os orçamentos simbólicos estabelecem o teto para que nada possa espiralar. O roteamento modelo reduz o custo básico de cada chamada. A remoção de contexto reduz a carga útil. Os disjuntores capturam os casos extremos. O cache elimina o trabalho repetido.

Plataformas como Nebula vão além, incorporando controles de custos na própria arquitetura do agente – orçamentos escalonados por tarefa, roteamento automático de camadas de modelo e delegação multiagente que isola os custos por subagente para que uma tarefa descontrolada não possa esgotar o orçamento compartilhado.

DR

Cinco padrões para interromper as espirais de custos dos agentes de IA antes que elas comecem:

Orçamentos de token por tarefa – defina um limite máximo antes do início da execução. 3x uso esperado.
Roteamento de modelo em camadas – use o modelo mais barato que possa lidar com cada etapa. 60-70% das etapas não precisam do seu melhor modelo.
Remoção de janela de contexto – comprima o histórico de conversas antigas em resumos. Redução de 90% em tokens de contexto.
Disjuntores - eliminam loops descontrolados após N etapas, custo $X ou K erros consecutivos.
Resultados determinísticos em cache — não execute novamente chamadas de ferramentas que retornem os mesmos dados.

Implemente-os em ordem. O roteamento de modelo e o orçamento de token por si só reduzirão a conta do agente pela metade.

Este artigo faz parte da série Building Production AI Agents. Anterior: Seu agente de IA está a um passo de um desastre. Veja também: 5 falhas de agente de IA em produção e agente único versus multiagente: por que os monólitos falham.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
Como impedir espirais de custos de agentes de IA antes que elas comecem
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção