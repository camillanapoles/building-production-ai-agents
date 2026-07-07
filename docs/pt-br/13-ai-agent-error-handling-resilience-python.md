# Tratamento de erros do agente AI: 4 padrões de resiliência em Python

> Artigo #13 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/ai-agent-error-handling-4-resilience-patterns-in-python-2gl5)

---

Seu agente de IA funciona perfeitamente no desenvolvimento. Em seguida, ele atinge a produção, o OpenAI retorna um 429, seu prompt fallback gera um erro de validação e todo o pipeline trava às 2 da manhã sem ninguém observando.

Este não é um problema de teste. É um problema de tratamento de erros do agente de IA. As APIs LLM falham de uma forma que o software tradicional nunca falha: limites de taxa, resultados não determinísticos, rejeições de política de conteúdo e estouros de janela de contexto não são casos extremos. São realidades operacionais diárias em qualquer escala significativa.

Este guia cobre quatro padrões de resiliência testados em batalha - nova tentativa com espera, cadeias de fallback modelo, disjuntors e degradação graciosa - com implementações Python puras que você pode inserir em qualquer projeto. Sem dependência de estrutura, sem dependências pesadas.

Por que os agentes de IA falham de maneira diferente dos softwares tradicionais

APIs tradicionais falham de forma previsível. Um banco de dados está inativo, você recebe um erro de conexão. Um token de autenticação expira, você recebe um 401. Você pode escrever testes determinísticos para eles.

agentes alimentados por LLM introduzem um modelo de falha fundamentalmente diferente:

Os limites de taxa (429) atingem imprevisivelmente com base em cotas de tokens por minuto que flutuam com a carga do provedor
O estouro da janela de contexto ocorre silenciosamente à medida que seu agente acumula resultados de ferramentas e histórico de conversas
As rejeições da política de conteúdo variam entre os provedores e são desencadeadas por informações que você nunca imaginou
O desvio no formato de resposta ocorre quando os provedores atualizam os modelos – seu prompt JSON perfeitamente estruturado retorna uma saída sutilmente diferente
Respostas parciais ou malformadas interrompem a análise posterior sem gerar erros óbvios

O insight crítico: essas falhas não são bugs a serem eliminados. São realidades operacionais para serem projetadas. Todo agente de IA de produção precisa de uma camada de resiliência entre sua lógica de negócios e as APIs LLM das quais depende.

Aqui estão os quatro padrões que fornecem essa camada.

Padrão 1: Nova tentativa inteligente com espera exponencial

As novas tentativas são sua primeira linha de defesa contra falhas transitórias. Mas novas tentativas ingênuas em APIs LLM são perigosas – elas amplificam falhas, desperdiçam tokens e podem esgotar seu orçamento durante uma interrupção.

O princípio fundamental: nem todos os erros merecem uma nova tentativa. Tentar novamente uma falha permanente (chave de API incorreta, solicitação malformada) desperdiça tempo e dinheiro. Falha rápida em um erro transitório (limite de taxa, timeout) perde uma solicitação que teria sido bem-sucedida na segunda tentativa.

Comece classificando os erros:


```python
from enum import Enum

class ErrorType(Enum):
TRANSIENT = "transient"    # Retry with backoff
PERMANENT = "permanent"    # Fail immediately
DEGRADED = "degraded"      # Switch to fallback

def classify_error(error: Exception) -> ErrorType:
"""Classify an LLM API error to determine recovery strategy."""
error_str = str(error).lower()
status = getattr(error, 'status_code', None)
```

# Transitório: tentar novamente com espera
se o status estiver em (429, 500, 502, 503) ou 'timeout' em error_str:
retornar ErrorType.TRANSIENT

# Degradado: mude para o modelo fallback
se 'context_length' em error_str ou 'content_filter' em error_str:
retornar ErrorType.DEGRADED

# Permanente: falha imediatamente
retornar ErrorType.PERMANENT


Agora crie a lógica retry. A implementação usa espera exponencial com jitter - o jitter evita o problema do "rebanho trovejante", onde múltiplas instâncias de agente retry exatamente nos mesmos intervalos após um limite de taxa compartilhado:


```python
import time
import random
import logging

logger = logging.getLogger(__name__)

def retry_with_backoff(
func,
max_retries: int = 3,
base_delay: float = 1.0,
max_delay: float = 60.0,
jitter: float = 1.0,
):
"""Retry a function with exponential backoff and jitter.
```

Apenas tenta novamente em erros transitórios. Erros permanentes falham imediatamente.
Erros degradados são gerados novamente para serem manipulados pela camada fallback.
"""
last_exception = Nenhum

para tentativa no intervalo (max_retries + 1):
tentar:
```python
return func()
except Exception as e:
last_exception = e
error_type = classify_error(e)

if error_type == ErrorType.PERMANENT:
logger.error(f"Permanent error, not retrying: {e}")
raise

if error_type == ErrorType.DEGRADED:
logger.warning(f"Degraded error, passing to fallback: {e}")
raise

if attempt == max_retries:
logger.error(f"All {max_retries} retries exhausted: {e}")
raise

# Exponential backoff: 1s, 2s, 4s... capped at max_delay
delay = min(base_delay * (2 ** attempt), max_delay)
# Add random jitter to prevent thundering herd
delay += random.uniform(0, jitter)

logger.warning(
f"Transient error (attempt {attempt + 1}/{max_retries}): {e}. "
f"Retrying in {delay:.1f}s"
)
time.sleep(delay)

raise last_exception

```

Uso com qualquer provedor LLM:


```python
import openai

client = openai.OpenAI()

def call_llm():
return client.chat.completions.create(
model="gpt-4o",
messages=[{"role": "user", "content": "Explain circuit breakers"}],
timeout=30,
)

# Retries transient errors up to 3 times with backoff
response = retry_with_backoff(call_llm, max_retries=3)

```

Dois detalhes que importam na produção:

Sempre defina um timeout nas chamadas LLM. Uma solicitação que fica suspensa por 5 minutos durante um ciclo de repetição bloqueia todo o pipeline do agente. 30 segundos é um padrão razoável.
Acompanhe o gasto de tokens em todas as tentativas. Três novas tentativas de um prompt de token de 4K custam 12K tokens. Adicione um limite de orçamento se o seu agente funcionar de forma autônoma.
Padrão 2: Modelo de Cadeias de Fallback

As novas tentativas tratam de falhas transitórias em um único provedor. Mas o que acontece quando o próprio provedor está inativo, ou quando uma rejeição de política de conteúdo é específica do provedor, ou quando você precisa de um modelo com uma janela de contexto maior?

As cadeias de fallback encaminham solicitações para modelos alternativos automaticamente quando o primário falha:


```python
from dataclasses import dataclass
from typing import Callable, Any

@dataclass
class ModelConfig:
name: str
call_fn: Callable
cost_per_1k_tokens: float  # Track cost at each tier

class FallbackChain:
"""Routes LLM requests through a prioritized chain of models.

Each model gets retry_with_backoff protection. If retries exhaust,
```
a cadeia passa para o próximo modelo.
"""

```python
def __init__(self, models: list[ModelConfig], max_retries: int = 2):
self.models = models
self.max_retries = max_retries

def call(self, messages: list[dict], **kwargs) -> dict:
errors = []

for i, model in enumerate(self.models):
try:
result = retry_with_backoff(
lambda m=model: m.call_fn(messages, **kwargs),
max_retries=self.max_retries,
)
if i > 0:
logger.info(
f"Fallback succeeded: {model.name} "
f"(after {i} failed model(s))"
)
return {
"content": self._extract_content(result, model.name),
"model": model.name,
"fallback_used": i > 0,
}
except Exception as e:
errors.append({"model": model.name, "error": str(e)})
logger.warning(f"Model {model.name} failed: {e}")
# Permanent errors (auth, bad request) should not fall through
error_type = classify_error(e)
if error_type == ErrorType.PERMANENT:
raise
continue

raise RuntimeError(f"All {len(self.models)} models failed: {errors}")

def _extract_content(self, result, model_name: str) -> str:
```
"""Normalizar o formato de resposta entre provedores."""
#Formato OpenAI
if hasattr(resultado, 'escolhas'):
retornar resultado.escolhas[0].message.content
# Formato antrópico
if hasattr(resultado, 'conteúdo'):
retornar resultado.content[0].text
# Formato de ditado
if isinstance(resultado, ditado):
```python
return result.get('content', str(result))
return str(result)

```

Configure uma cadeia prática de fallback:


```python
import openai
import anthropic

oai = openai.OpenAI()
anth = anthropic.Anthropic()

def call_gpt4o(messages, **kwargs):
return oai.chat.completions.create(
model="gpt-4o", messages=messages, timeout=30, **kwargs
)

def call_claude_sonnet(messages, **kwargs):
system = next((m["content"] for m in messages if m["role"] == "system"), "")
user_msgs = [m for m in messages if m["role"] != "system"]
return anth.messages.create(
model="claude-sonnet-4-20250514", system=system,
messages=user_msgs, max_tokens=4096, timeout=30,
)

def call_gpt4o_mini(messages, **kwargs):
return oai.chat.completions.create(
model="gpt-4o-mini", messages=messages, timeout=30, **kwargs
)

chain = FallbackChain([
ModelConfig("gpt-4o", call_gpt4o, cost_per_1k_tokens=0.005),
ModelConfig("claude-sonnet", call_claude_sonnet, cost_per_1k_tokens=0.003),
ModelConfig("gpt-4o-mini", call_gpt4o_mini, cost_per_1k_tokens=0.00015),
])

# Automatically falls through: GPT-4o -> Claude -> GPT-4o-mini
result = chain.call([{"role": "user", "content": "Analyze this data..."}])
print(f"Answered by: {result['model']}, fallback: {result['fallback_used']}")

```

A ordem fallback é importante. Organize por: primeiro a qualidade, depois o fornecedor diferente e depois a otimização de custos. Se o GPT-4o tiver taxa limitada, Claude Sonnet (fornecedor diferente) provavelmente terá sucesso. GPT-4o-mini é o último recurso – mais barato, mais rápido, de qualidade inferior, mas sempre disponível.

Uma decisão de design que vale a pena destacar: o FallbackChain envolve cada chamada de modelo em retry_with_backoff. Isso significa que cada modelo tem suas próprias tentativas de retry antes que a cadeia prossiga. As novas tentativas tratam de mensagens transitórias; fallbacks lidam com interrupções sustentadas.

Padrão 3: Disjuntor para Chamadas de Ferramentas

Novas tentativas e fallbacks tratam de falhas de solicitações individuais. Os disjuntores resolvem um problema diferente: o que acontece quando um provedor ou ferramenta fica inativo por 10 minutos e cada solicitação em seu sistema perde 30 segundos tentando novamenteantes de falhar?

Sem um disjuntor, uma API externa instável transforma cada solicitação do agente em uma falha slow. Seus usuários esperam, seu orçamento de token queima e o provedor em dificuldades é atingido por um tráfego de novas tentativas que impede a recuperação.

Um disjuntor monitora as taxas de falha e "disparos" quando excedem um limite, rejeitando imediatamente as solicitações em vez de tentar realizá-las:


```python
import time
import threading

class CircuitBreaker:
"""Prevents cascading failures by fast-failing when a service is down.

States:
CLOSED  - Normal operation, requests pass through
OPEN    - Service is down, requests fail immediately
HALF_OPEN - Testing if service recovered (one probe request)
"""

def __init__(
self,
name: str,
failure_threshold: int = 5,
reset_timeout: float = 60.0,
success_threshold: int = 2,
):
self.name = name
self.failure_threshold = failure_threshold
self.reset_timeout = reset_timeout
self.success_threshold = success_threshold

self._state = "CLOSED"
self._failure_count = 0
self._success_count = 0
self._last_failure_time = 0.0
self._lock = threading.Lock()

@property
def state(self) -> str:
with self._lock:
if self._state == "OPEN":
# Check if reset timeout has elapsed
if time.time() - self._last_failure_time >= self.reset_timeout:
self._state = "HALF_OPEN"
self._success_count = 0
return self._state

def call(self, func, *args, **kwargs):
"""Execute function through circuit breaker protection."""
current_state = self.state

if current_state == "OPEN":
raise CircuitOpenError(
f"Circuit '{self.name}' is OPEN. "
f"Service unavailable, retrying in "
f"{self.reset_timeout - (time.time() - self._last_failure_time):.0f}s"
)

try:
result = func(*args, **kwargs)
self._on_success()
return result
except Exception as e:
self._on_failure()
raise

def _on_success(self):
with self._lock:
if self._state == "HALF_OPEN":
self._success_count += 1
if self._success_count >= self.success_threshold:
self._state = "CLOSED"
self._failure_count = 0
logger.info(f"Circuit '{self.name}' CLOSED (recovered)")
else:
self._failure_count = 0

def _on_failure(self):
with self._lock:
self._failure_count += 1
self._last_failure_time = time.time()

if self._state == "HALF_OPEN":
self._state = "OPEN"
logger.warning(f"Circuit '{self.name}' re-OPENED (probe failed)")
elif self._failure_count >= self.failure_threshold:
self._state = "OPEN"
logger.warning(
f"Circuit '{self.name}' OPENED "
f"after {self._failure_count} consecutive failures"
)


class CircuitOpenError(Exception):
"""Raised when a circuit breaker is open."""
pass

```

A máquina de estados é simples, mas poderosa:


```python
CLOSED (normal)     -- failures hit threshold -->  OPEN (fast-fail)
|
timeout expires
|
HALF_OPEN (probe)
/        \
success       failure
/              \
CLOSED             OPEN

```

Use um disjuntor separado para cada dependência externa:


```python
# One breaker per service -- never share across providers
openai_breaker = CircuitBreaker("openai", failure_threshold=5, reset_timeout=60)
search_breaker = CircuitBreaker("web-search", failure_threshold=3, reset_timeout=30)
db_breaker = CircuitBreaker("database", failure_threshold=3, reset_timeout=45)

def agent_search(query: str) -> list[dict]:
"""Agent tool: web search with circuit breaker protection."""
try:
return search_breaker.call(web_search_api, query)
except CircuitOpenError:
logger.warning("Search unavailable, using cached results")
return get_cached_results(query)
except Exception:
return []  # Graceful degradation: empty results, not a crash

```

O detalhe crítico: um disjuntor por dependência externa. Se o OpenAI estiver inativo, você não deseja que o disjuntor bloqueie as chamadas Antrópicas também. E o parâmetro success_threshold=2 evita que uma única solicitação bem-sucedida restaure o tráfego total para um serviço instável.

Padrão 4: Degradação Graciosa

Às vezes tudo falha. Seu modelo principal tem taxa limitada, o provedor fallback está inativo e o disjuntor está aberto. Travamentos tradicionais no tratamento de erros. A degradação graciosa oferece algo útil em vez de nada.

O princípio: os usuários toleram muito mais capacidade reduzida do que falhas ou solicitações interrompidas.


```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class AgentResponse:
content: str
quality_tier: str    # "full", "reduced", "cached", "static"
model_used: str
warning: Optional[str] = None

class ResilientAgent:
"""Agent with tiered degradation: full -> reduced -> cached -> static."""

def __init__(self, fallback_chain: FallbackChain, cache: dict = None):
self.chain = fallback_chain
self.cache = cache or {}

def run(self, messages: list[dict]) -> AgentResponse:
# Tier 1: Full capability via fallback chain
try:
result = self.chain.call(messages)
# Cache successful responses for future degradation
cache_key = messages[-1]["content"][:100]
self.cache[cache_key] = result["content"]
return AgentResponse(
content=result["content"],
quality_tier="full" if not result["fallback_used"] else "reduced",
model_used=result["model"],
)
except RuntimeError:
pass  # All models failed

# Tier 2: Cached response from similar previous query
cache_key = messages[-1]["content"][:100]
if cache_key in self.cache:
return AgentResponse(
content=self.cache[cache_key],
quality_tier="cached",
model_used="cache",
warning="This response is from cache and may be outdated.",
)

# Tier 3: Static fallback -- honest about limitations
return AgentResponse(
content=(
"I'm experiencing temporary difficulties connecting to AI services. "
"Please try again in a few minutes. If this persists, check "
"https://status.openai.com for provider status."
),
quality_tier="static",
model_used="none",
warning="All AI services are currently unavailable.",
)

```

O campo quality_tier é importante para a lógica downstream. Seu aplicativo pode tomar decisões com base na qualidade da resposta:


```python
agent = ResilientAgent(chain)
response = agent.run([{"role": "user", "content": "Summarize today's metrics"}])

if response.quality_tier == "static":
# Don't send automated reports with static fallback content
notify_ops_team("Agent degraded, manual review needed")
elif response.quality_tier == "cached":
# Send the report but flag it
send_report(response.content, caveat="Based on cached data")
else:
send_report(response.content)

Putting It All Together: A Resilient Agent Pipeline
```

O verdadeiro poder vem da composição de todos os quatro padrões em uma defesa em camadas. Aqui está a ordem de execução do mais externo para o mais interno:


Sua lógica de agente
|
```python
Graceful Degradation (always returns something)
|
Fallback Chain (tries alternative models)
|
Circuit Breaker (fast-fails during outages)
|
Retry with Backoff (handles transient errors)
|
LLM Provider API

```

Aqui está um pipeline completo e funcional que conecta tudo:


```python
def build_resilient_agent() -> ResilientAgent:
"""Build an agent with all four resilience patterns composed."""

# Layer 1: Circuit breakers per provider
oai_breaker = CircuitBreaker("openai", failure_threshold=5, reset_timeout=60)
anth_breaker = CircuitBreaker("anthropic", failure_threshold=5, reset_timeout=60)

# Layer 2: Provider calls wrapped with circuit breakers
oai_client = openai.OpenAI()
anth_client = anthropic.Anthropic()

def gpt4o_with_breaker(messages, **kwargs):
return oai_breaker.call(
lambda: oai_client.chat.completions.create(
model="gpt-4o", messages=messages, timeout=30, **kwargs
)
)

def claude_with_breaker(messages, **kwargs):
system = next((m["content"] for m in messages if m["role"] == "system"), "")
user_msgs = [m for m in messages if m["role"] != "system"]
return anth_breaker.call(
lambda: anth_client.messages.create(
model="claude-sonnet-4-20250514", system=system,
messages=user_msgs, max_tokens=4096, timeout=30,
)
)

def gpt4o_mini_with_breaker(messages, **kwargs):
return oai_breaker.call(
lambda: oai_client.chat.completions.create(
model="gpt-4o-mini", messages=messages, timeout=30, **kwargs
)
)

# Layer 3: Fallback chain with retry built in
chain = FallbackChain(
models=[
ModelConfig("gpt-4o", gpt4o_with_breaker, 0.005),
ModelConfig("claude-sonnet", claude_with_breaker, 0.003),
ModelConfig("gpt-4o-mini", gpt4o_mini_with_breaker, 0.00015),
],
max_retries=2,
)

# Layer 4: Graceful degradation wraps everything
return ResilientAgent(chain)


# Usage
agent = build_resilient_agent()
response = agent.run([
{"role": "system", "content": "You are a helpful assistant."},
{"role": "user", "content": "What are the key trends in AI this week?"},
])

print(f"Quality: {response.quality_tier}")
print(f"Model: {response.model_used}")
print(f"Response: {response.content[:200]}")

```

Observe como as camadas se compõem: retry acontece dentro do disjuntor, que acontece dentro da cadeia fallback, que acontece dentro do wrapper de degradação. Se as novas tentativas esgotarem suas tentativas, o disjuntor registra uma falha. Após falhas suficientes, o circuito abre e a cadeia fallback ignora totalmente esse provedor - sem novas tentativas, sem espera.

Referência rápida: quando usar cada padrão
Padrão melhor para evitar quando
```python
Exponential Backoff	Rate limits, transient 5xx errors	Permanent failures (auth, bad request)
Model Fallback	Provider outages, cost optimization	Task needs a specific model's capabilities
Circuit Breaker	Flaky external APIs, sustained outages	Internal computations that don't call external services
Graceful Degradation	Multi-source tasks, user-facing agents	Binary success/fail operations (payments, writes)
Key Metrics to Track
```

Os padrões de resiliência são tão bons quanto a sua capacidade de observá-los. Acompanhe-os na produção:

Taxa de novas tentativas por provedor – pico acima de 20%? Algo está degradado a montante. Defina alertas.
Taxa de ativação de fallback – Se o seu modelo principal falhar mais de 10% das vezes, reconsidere a escolha do seu provedor.
Mudanças no estado do disjuntor – Cada transição ABRIR/FECHAR deve acionar um alerta. Ciclismo frequente significa uma dependência instável.
Distribuição do nível de degradação – Qual porcentagem de respostas é servida a partir do cache ou do fallback estático? Esta é a verdadeira métrica de qualidade que seus usuários experimentam.
Custo por solicitação bem-sucedida – Os substitutos para modelos mais caros aumentam os custos. Acompanhe isso para detectar estouros no orçamento antes que se tornem um problema.
Concluindo

Os agentes de IA de produção precisam de resiliência como uma preocupação arquitetônica de primeira classe, e não uma reflexão tardia após a primeira interrupção às 2 da manhã. Os quatro padrões neste guia - nova tentativa com espera, fallbacks de modelo, disjuntors e degradação graciosa - formam uma estratégia de defesa profunda que mantém seus agentes em execução quando tudo ao seu redor está quebrando.

O código neste artigo é Python independente de estrutura, que você pode inserir em qualquer projeto. Comece com repetir + classificar (o padrão de ROI mais alto), adicione cadeias fallback quando você depender de um único provedor e coloque em camadas disjuntoress quando seus agentes chamarem ferramentas externas em grande escala.

Se você quiser pular a construção desse encanamento de resiliência por conta própria, plataformas como Nebula lidam com lógica retry, fallbacks de modelos e disjuntoress de ferramentas no nível da infraestrutura - para que você possa se concentrar no que seu agente faz em vez de como ele se recupera.

O código completo deste artigo está pronto para copiar e colar. Construa algo resiliente.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
Tratamento de erros do agente AI: 4 padrões de resiliência em Python
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção