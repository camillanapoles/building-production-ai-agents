# Observabilidade do agente de IA: os 4 pilares que evitam que seus agentes gastem US$ 2.000 às 3 da manhã

> Artigo #20 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/ai-agent-observability-the-4-pillars-that-keep-your-agents-from-burning-2000-at-3-am-5882)

---

No mês passado, um agente de produção de uma startup começou a ter alucinações com dados de clientes. Cada chamada de API retornou 200 OK. O painel do agente ficou verde em todo o quadro. A latência era normal. A taxa de erro foi zero.

Seis horas depois, a cobrança chegou: US$ 2.847 em tokens para uma única consulta de usuário que entrou em um ciclo de raciocínio e nunca mais parou.

Isto não é uma hipótese. O Operator Collective documentou faturas de agentes fugitivos de US$ 47.000 em 2025. Em 2026, os números só aumentarão à medida que os agentes se tornarem mais autônomos e mais baratos por token.

O problema é fundamental: o monitoramento tradicional foi construído para software determinístico. A solicitação chega, o código é executado, a resposta sai. A mesma entrada sempre produz a mesma saída. Você sabe como é “saudável”.

AI agentes quebra esse contrato. O mesmo prompt pode passar por 15 iterações de raciocínio, chamar 40 ferramentas e produzir uma resposta confiável, mas completamente errada – todas retornando HTTP 200. Seus painéis permanecem verdes. Seu agente está pegando fogo.

Depois de executar agentes autônomos 24 horas por dia, 7 dias por semana na infraestrutura de produção e observá-los falhar de maneiras que nunca imaginei, construí a pilha de observabilidade que realmente detecta essas falhas. Tudo se resume a quatro pilares que o APM tradicional não cobre.

Pilar 1: Observabilidade de Custos — Rastreamento de Token com Detecção de Anomalias

O modo de falha mais urgente não é a correção – é o custo. Um agente em um loop de raciocínio não tem ponto de parada natural, a menos que você imponha um na camada de observabilidade.

Atribuição de custo por execução

Cada execução de agente precisa de identificadores exclusivos: ID de rastreamento, ID de sessão e ID de agente. Cada chamada de LLM e invocação de ferramenta é marcada com esses identificadores para que você possa atribuir custos em qualquer granularidade.


```python
import time
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class TokenLedger:
\"\"\"Track token usage per agent, per run, per call.\"\"\"
trace_id: str
session_id: str
agent_id: str

input_tokens: int = 0
output_tokens: int = 0
cost_usd: float = 0.0
```

# Configuração de preços – atualização por modelo
input_cost_per_1k: float = 0,003 # Soneto 4 entrada
output_cost_per_1k: float = 0,015 # Saída do Soneto 4

```python
def record(self, input_tok: int, output_tok: int) -> float:
\"\"\"Record a single LLM call. Returns the cost.\"\"\"
self.input_tokens += input_tok
self.output_tokens += output_tok
call_cost = (
(input_tok / 1_000) * self.input_cost_per_1k
+ (output_tok / 1_000) * self.output_cost_per_1k
)
self.cost_usd += call_cost
return call_cost

def running_cost(self) -> float:
return self.cost_usd

```

Integrado em um ciclo de agentes com um orçamento rígido:


MAX_RUN_COST = 0,50 # Limite máximo por sessão do agente

```python
ledger = TokenLedger(
trace_id=\"trace_abc123\",
session_id=\"sess_xyz789\",
agent_id=\"support-agent-v2\",
)

while agent_running:
if ledger.running_cost() >= MAX_RUN_COST:
logger.error(
f\"Budget exceeded: ${ledger.running_cost():.3f} \"
f\"(limit ${MAX_RUN_COST:.2f}). Aborting run.\"
)
# Save partial results, escalate to human
break

response = call_llm(prompt, model=\"claude-sonnet-4\")
ledger.record(
input_tok=response.usage.input_tokens,
output_tok=response.usage.output_tokens,
)

# ... process response ...

Real-Time Anomaly Alerts
```

Os limites orçamentais evitam corridas catastróficas. A detecção de anomalias detecta aumentos crescentes de custos antes que eles ultrapassem o limite.


```python
from collections import deque
import time

class CostAnomalyDetector:
\"\"\"Alert when token burn rate exceeds 3x the rolling average.\"\"\"

def __init__(self, window_seconds: int = 300, threshold_multiplier: float = 3.0):
self.window = window_seconds
self.threshold = threshold_multiplier
self.burn_rates: deque[tuple[float, float]] = deque()  # (timestamp, rate)

def record_burn(self, tokens_per_second: float) -> Optional[str]:
\"\"\"Returns alert message if anomaly detected, else None.\"\"\"
now = time.time()
self.burn_rates.append((now, tokens_per_second))
```

# Eliminar entradas antigas
corte = agora - self.window
```python
self.burn_rates = deque(
(ts, rate) for ts, rate in self.burn_rates if ts >= cutoff
)

if len(self.burn_rates) \u003C 5:
return None  # Not enough data

avg_rate = sum(r for _, r in self.burn_rates) / len(self.burn_rates)
if tokens_per_second > avg_rate * self.threshold:
return (
f\"ANOMALY: burn rate {tokens_per_second:.0f} tps \"
f\"is {tokens_per_second/avg_rate:.1f}x above average \"
f\"({avg_rate:.0f} tps)\"
)
return None

```

O alerta é acionado em segundos – não quando chega a fatura mensal. Eu executo isso como um processo secundário que pesquisa o razão do agente a cada 10 segundos e dispara um webhook do Slack se a taxa de consumo aumentar.

Pilar 2: Observabilidade da Qualidade – Avaliações Canárias na Produção

As avaliações pré-implantação informam o desempenho do agente antes de entrar em operação. O monitoramento da qualidade da produção informa se ela ainda está funcionando corretamente.

Conjunto de consultas canário

Escolha de 10 a 20 perguntas com respostas corretas conhecidas. Execute-os a cada 5 minutos. Alerta quando a precisão cai abaixo de 90%.


```python
from dataclasses import dataclass

@dataclass
class CanaryQuery:
question: str
expected_answer: str
evaluation_prompt: str  # LLM-as-judge prompt

CANARY_QUERIES = [
CanaryQuery(
question=\"What's the status of deployment d-4821?\",
expected_answer=\"completed\",
evaluation_prompt=(
\"Does the agent's response indicate that deployment d-4821 \"
\"was completed successfully? Answer YES or NO.\"
),
),
CanaryQuery(
question=\"Which region has the most errors today?\",
expected_answer=\"us-east-1\",
evaluation_prompt=(
\"Did the agent correctly identify us-east-1 as the region \"
\"with the most errors? Answer YES or NO.\"
),
),
]

def run_canary_suite(agent, queries: list[CanaryQuery]) -> dict:
results = {}
for q in queries:
response = agent.run(q.question)
# LLM-as-judge evaluation
judge_result = llm.ask(
f\"{q.evaluation_prompt}\
```
\
Resposta do agente: {response}\"
```python
)
passed = \"yes\" in judge_result.text.lower()
results[q.question] = passed

pass_rate = sum(results.values()) / len(results)
return {\"pass_rate\": pass_rate, \"details\": results}

```

Se pass_rate cair abaixo de 0,9, algo mudou: um provedor de modelo foi degradado, uma API de ferramenta alterou seu formato de resposta ou uma atualização de prompt quebrou alguma coisa. A suíte canário detecta isso antes que os clientes percebam.

Detecção de desvio semântico

Além de aprovação/reprovação binária, monitore se as respostas do agente estão se tornando menos semanticamente semelhantes às boas respostas históricas. Isso detecta a degradação slow que as consultas canário podem perder.


```python
import numpy as np
from sentence_transformers import SentenceTransformer

class SemanticDriftDetector:
def __init__(self, baseline_responses: list[str]):
self.model = SentenceTransformer(\"all-MiniLM-L6-v2\")
self.baseline_embeddings = self.model.encode(baseline_responses)

def check_drift(self, current_responses: list[str]) -> float:
current_embeddings = self.model.encode(current_responses)
# Cosine similarity between baseline and current
similarities = np.mean(
np.dot(current_embeddings, self.baseline_embeddings.T), axis=1
)
return float(np.mean(similarities))  # 1.0 = identical, 0.0 = unrelated

```

O desvio abaixo de 0,7 significa que o estilo ou conteúdo de saída do agente mudou fundamentalmente – é hora de reverter e investigar.

Pilar 3: Observabilidade Comportamental – Rastreando o Raciocínio do Agente

É aqui que a observabilidade do agente diverge mais acentuadamente do APM tradicional. Você precisa ver o que o agente decidiu, não apenas quais solicitações HTTP ele fez.

O log do agente estruturado

Cada ação do agente produz telemetria estruturada. Não são registros de texto que exigem análise de regex, mas objetos JSON com um esquema consistente:


{
```python
\"timestamp\": \"2026-04-30T04:23:45Z\",
\"trace_id\": \"trace_abc123\",
\"span_id\": \"span_001\",
\"event_type\": \"tool_call\",
\"agent_name\": \"support-agent-v2\",
\"tool_name\": \"query_database\",
\"tool_input\": {\"query\": \"SELECT status FROM deployments WHERE id = 'd-4821'\"},
\"tool_output_summary\": \"status=completed\",
\"latency_ms\": 142,
\"input_tokens\": 245,
\"output_tokens\": 1023,
\"cost_usd\": 0.002,
\"reasoning_depth\": 3,
\"confidence_score\": 0.87,
\"parent_span_id\": null
}

```

Os campos-chave além do logging padrão:

reasoning_profundidade — quantas vezes o agente entrou em loop. Se esse número ultrapassar 8, o agente provavelmente está em uma espiral de raciocínio.
trust_score — a confiança autoavaliada do agente (se o seu prompt solicitar). Baixa confiança + alta profundidade de raciocínio = quase certamente um agente travado.
tool_output_summary — não a saída completa (muito cara para armazenar), mas uma visualização truncada mais um indicador de status (sucesso/erro/parcial).
Atribuição de chamada de ferramenta

Quando o monitoramento do seu banco de dados é acionado para "10.000 consultas em 5 minutos", você precisa saber qual agente as acionou e por quê. Cada chamada de ferramenta remonta a uma etapa de raciocínio específica.


```python
class ToolCallTracker:
def __init__(self, max_calls_per_tool: int = 50):
self.max_calls = max_calls_per_tool
self.call_counts: dict[str, int] = {}
self.call_history: list[dict] = []

def record(self, tool_name: str, reasoning_step: dict) -> None:
self.call_counts[tool_name] = self.call_counts.get(tool_name, 0) + 1
self.call_history.append({
\"tool\": tool_name,
\"step\": reasoning_step,
\"call_number\": self.call_counts[tool_name],
})

if self.call_counts[tool_name] > self.max_calls:
raise ToolCallLimitExceeded(
f\"Tool '{tool_name}' called {self.call_counts[tool_name]} times \"
f\"(limit {self.max_calls})\"
)

def get_distribution(self) -> dict[str, int]:
return dict(self.call_counts)

```

Isso combina com o registro de custos: você pode ver não apenas quanto o agente gastou, mas também qual ferramenta gerou o gasto. Se search_database for responsável por 80% das chamadas de ferramenta, o agente pode estar confiando demais na recuperação quando uma ferramenta mais simples seria suficiente.

Pilar 4: Observabilidade de Dependência — Mapeando o Mundo Externo do Agente

Um agente depende de provedores LLM, APIs de ferramentas, bancos de dados vetoriaiss, outros agentes e serviços de infraestrutura. Quando algo quebra, você precisa saber se é o seu agente ou uma dependência.

O Mapa de Saúde da Dependência

Sua verificação de integridade não deve retornar apenas {\"status\": \"ok\"}. Ele deve testar cada dependência e relatar o status granular:


```python
async def agent_health_check(agent) -> dict:
start = time.time()

# Test LLM connectivity and latency
llm_start = time.time()
llm_response = await agent.llm.complete(\"Say 'healthy'\")
llm_latency = (time.time() - llm_start) * 1000

# Test each tool's availability
tools_status = {}
for tool in agent.tools:
try:
await tool.ping(timeout=3.0)
tools_status[tool.name] = \"healthy\"
except Exception as e:
tools_status[tool.name] = f\"error: {type(e).__name__}\"

# Run canary eval
canary_start = time.time()
canary_result = await agent.run(\"What is 2+2?\")
canary_latency = (time.time() - canary_start) * 1000
canary_passed = \"4\" in canary_result

return {
\"status\": \"healthy\" if canary_passed and all(
s == \"healthy\" for s in tools_status.values()
) else \"degraded\",
\"llm_latency_ms\": round(llm_latency),
\"model_version\": agent.llm.model,
\"tools\": tools_status,
\"canary_passed\": canary_passed,
\"canary_latency_ms\": round(canary_latency),
\"uptime_seconds\": agent.uptime(),
}

```

Esse endpoint é executado a cada 60 segundos e alimenta seu painel de monitoramento. Quando o banco de dados de vetores fica inativo, você vê vector_search: \"error: ConnectionRefusedError\" em vez de se perguntar por que a precisão do agente caiu 40%.

Rastreamento de agente para agente

Os sistemas multiagentes são os mais difíceis de depurar porque as falhas se propagam em cascata. O Agente A produz uma saída ruim → O Agente B consome e amplifica o erro → O Agente C aprova o resultado porque viu apenas o rascunho final.

A correção é distribuída tracing através dos limites do agente. Cada execução do agente se torna um intervalo, e os intervalos filhos capturam chamadas LLM individuais e invocações de ferramentas:


rastreamento: consulta do usuário-abc123
```python
├── span: agent.research (2.4s, $0.12)
│   ├── span: gen_ai.chat — query planning (0.3s)
│   ├── span: tool.vector_search (0.8s)
│   ├── span: tool.web_search (0.6s)
│   └── span: gen_ai.chat — synthesize findings (0.7s)
├── span: agent.writer (1.8s, $0.08)
│   ├── span: gen_ai.chat — draft generation (1.2s)
│   └── span: gen_ai.chat — self-review (0.6s)
└── span: agent.reviewer (1.1s, $0.05)
├── span: gen_ai.chat — quality check (0.8s)
└── span: gen_ai.chat — scoring (0.3s)

```

Esse rastreamento responde às perguntas que o logging simples do LLM não consegue: qual agente estava slooeste, qual agente era mais caro, quais dados fluíram entre agentes e onde a cadeia quebrou.

As convenções semânticas OpenTelemetry GenAI (ainda experimentais no início de 2026) definem atributos padrão para isso: gen_ai.system, gen_ai.request.model, gen_ai.usage.input_tokens e os novos gen_ai.agent.name e gen_ai.agent.id específicos do agente. Se você mesmo estiver construindo a camada de instrumentação, siga estas convenções: elas tornam seus dados portáteis em back-ends de observabilidade.

O painel que realmente ajuda

O painel do seu agente deve responder rapidamente a cinco perguntas:

Quanto estamos gastando agora? — taxa de queima de tokens em tempo real com medidores de orçamento
A qualidade está se mantendo? — taxa de aprovação canário, distribuição de pontuação de confiança ao longo do tempo
Os agentes estão se comportando normalmente? — histograma de distribuição de chamadas de ferramenta, distribuição de profundidade de raciocínio
Algum problema em cascata? — mapa de dependências com status ativo por ferramenta/API/modelo
Quais usuários/sessões são afetados? — taxa de erro por segmento de usuário, não apenas agregado

Aqui está a estrutura que uso:


┌──────────────────────── ─────────────────────────┐
│ CUSTO EM TEMPO REAL │
│ Hoje: $ 4,23 |  Corrida atual: $ 0,12 |  Orçamento: OK │
│ ━━━━━━━━━━━━━━━━━━━━━━ ━━━━━━━━━━━━━━━━━━━━━━ │
│ Taxa de queima: 450 tok/s (normal: 50-200) [AVISO] │
├──────────────────────── ─────────────────────────┤
│ QUALIDADE │
│ Canário: 9/10 aprovado (90%) |  Deriva: 0,82 OK │
│ Média de confiança: 0,84 |  Média de loops: 2,3 │
├──────────────────────── ─────────────────────────┤
│ COMPORTAMENTO │
│ Chamadas de ferramentas (última hora): │
│ search_docs: 847 |  consulta_db: 423 |          │
│ enviar_e-mail: 12 |  criar_ticket: 8 │
│ Histograma de profundidade de raciocínio: 1:60% 2:25% │
│ 3:10% 4+:5% │
│ [AVISO] │
├──────────────────────── ─────────────────────────┤
│ DEPENDÊNCIAS │
│ LLM (Soneto 4): [OK] 312ms │
│ Banco de dados vetorial: [OK] 12ms │
│ API GitHub: [AVISO] 2.1s (degradado) │
│ E-mail SMTP: [OK] 45ms │
└──────────────────────── ─────────────────────────┘

Onde plataformas como a Nebulosa se encaixam

Construir essa pilha de observabilidade do zero significa instrumentar cada loop de agente, cada chamada de ferramenta, cada resposta LLM, conectar o livro-razão de custos, configurar o conjunto canário e manter a infraestrutura de rastreamento. É um trabalho necessário, mas não é o trabalho que diferencia o seu produto.

Plataformas como o Nebula lidam com a camada de observabilidade como parte do tempo de execução do agente. Cada execução do agente é rastreada de ponta a ponta com atribuição de custo, orçamento de tokens são aplicados no nível da plataforma (não no código do seu aplicativo) e a atribuição de chamada de ferramenta acontece automaticamente porque cada ferramenta é registrada no registro de serviço da plataforma.

A arquitetura fica assim:


Definição do agente (sua configuração)
│
▼
┌─────────────────────────────────┐
│ Tempo de execução do agente nebulosa │
│ ┌───────────────────────────┐ │
│ │ Rastreador de Execução │ │
│ │ - Livro razão de tokens │ │
│ │ - Atribuição de custos │ │
│ │ - Propagação de extensão │ │
│ └───────────────────────────┘ │
│ ┌───────────────────────────┐ │
│ │ Motor de guarda-corpos │ │
│ │ - Execução orçamentária │ │
│ │ - Limites de chamada de ferramenta │ │
│ │ - Limites de ciclo │ │
│ └───────────────────────────┘ │
│ ┌───────────────────────────┐ │
│ │ Monitor de Qualidade │ │
│ │ - Avaliações canárias │ │
│ │ - Validação de saída │ │
│ │ - Detecção de desvio │ │
│ └───────────────────────────┘ │
└─────────────────────────────────┘
│
├──→ Ferramenta: Servidor GitHub MCP
├──→ Ferramenta: Consulta de Banco de Dados MCP
├──→ Ferramenta: Pesquisa na Web MCP
│
▼
Traces + Métricas → Seu painel


Você define o que o agente faz. A plataforma garante que você veja quando algo dá errado.

Para equipes que já investiram em uma pilha de observabilidade (Datadog, Grafana, New Relic), os rastreamentos do agente são exportados no formato OpenTelemetry – intervalos padrão que você pode ingerir junto com a telemetria do seu aplicativo. Isso significa que sua equipe de SRE não precisa de um painel separado para IA; agentes são sinais de primeira classe no mesmo ambiente que o resto da sua infraestrutura.

A Grafana Cloud lançou recentemente o AI Observability em versão prévia pública e segue exatamente este padrão: as sessões do agente tornam-se telemetria de primeira classe, correlacionadas com rastreamentos, métricas e logs. O principal insight compartilhado por Grafana e Nebula é que você não precisa de um novo paradigma de observabilidade para IA – você precisa do existente estendido para entender a semântica do agente.

Conclusões acionáveis

Comece com o rastreamento de custos no primeiro dia. Um livro-razão de tokens com um limite orçamentário rígido evita o cenário de US$ 2.000 às 3 da manhã. Todo o resto é otimização.

Execute consultas canário na produção, não apenas na CI. Suas avaliações antes da implantação não informam nada sobre degradação do modelo, alterações na API da ferramenta ou desvios na infraestrutura que ocorrem após o envio.

Estruture seus logs como eventos, não como texto. A telemetria JSON com esquemas consistentes (ID de rastreamento, ID de intervalo, nome da ferramenta, custo, profundidade de raciocínio) pode ser consultada. Os logs de texto exigem grep e suposições.

Acompanhe a profundidade do raciocínio e a contagem de chamadas de ferramentas por execução. Dois números que capturam 80% dos modos de falha do agente: profundidade de raciocínio excedendo 8 = agente travado, contagem de chamadas de ferramenta excedendo o intervalo esperado = loop de seleção de ferramenta errada.

Mapeie suas dependências explicitamente. Quando o agente quebra, a primeira pergunta deve ser “somos nós ou uma dependência?” – e seu exame de saúde deve responder a isso instantaneamente.

Exporte no formato OpenTelemetry. Quer você use uma plataforma gerenciada ou auto-hospedada, os spans padrão OTLP significam que você pode trocar back-ends de observabilidade sem reinstrumentação. Não se prenda ao formato tracing de um fornecedor.

A lacuna de observabilidade do agente não é um problema tecnológico – é uma mudança de modelo mental. Seus agentes são sistemas probabilísticos que executam fluxos de trabalhos não determinísticos em dependências externas. Monitore-os como sistemas distribuídos, não como terminais HTTP. Faça isso e as chamadas de despertar às 3 da manhã tornar-se-ão raras.

Este artigo faz parte da série Building Production AI Agents em Dev.to.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
Observabilidade do agente de IA: os 4 pilares que evitam que seus agentes gastem US$ 2.000 às 3 da manhã
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção