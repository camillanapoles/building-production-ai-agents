Nº 5 Padrões de tratamento de erros do agente de IA que mantêm seu agente funcionando às 3 da manhã

> Artigo #18 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/5-ai-agent-error-handling-patterns-that-keep-your-agent-running-at-3-am-5d5o)

---

No ano passado, uma implantação deu errado. Um agente de IA estava executando um pipeline de enriquecimento de dados: extrair registros de uma API, mapear campos em um esquema, gravar em um banco de dados. Cada chamada de API retornou 200 OK. O painel do agente ficou verde em todo o quadro. O agente relatou sucesso em todas as etapas.

Seis horas depois, uma equipe downstream sinalizou os dados. Metade dos mapeamentos de campo foram alucinados. O agente mapeou com segurança a receita da empresa para a contagem dos funcionários, inventou valores para campos ausentes e gravou duplicatas para registros que já havia processado. Centenas de linhas incorretas, todas marcadas como verificadas.

Ninguém percebeu porque nada “falhou”.

Este é o problema fundamental dos agentes de IA na produção: as falhas mais perigosas parecem exatamente com o sucesso. O tratamento de erros tradicional – blocos try/catch, verificações de código de status HTTP, monitoramento de falhas – foi desenvolvido para software determinístico. AI agentes são sistemas probabilísticos. Eles não quebram quando estão errados; eles produzem lixo com segurança com um código de status 200.

Depois de passar por esse incidente e construir dezenas de agentes de produção desde então, destilei cinco padrões de tratamento de erros que realmente funcionam. Cada padrão lida com um modo de falha que o anterior não consegue detectar. Juntos, eles formam uma estratégia de defesa profunda que mantém os agentes em execução, evita a corrupção silenciosa de dados e proporciona sono em implantações durante a semana.

Padrão 1: Disjuntores para falhas de qualidade LLM (não apenas erros HTTP)

O padrão clássico de disjuntor — fechado, aberto, semiaberto — é a engenharia de infraestrutura padrão. Mas quando se trata de agentes de IA, a versão tradicional está incompleta. Ele rastreia falhas de HTTP. Ele ignora falhas de qualidade.

O problema

Após uma degradação do provedor de modelo, um agente começou a retornar JSON malformado. Todas as chamadas de API foram bem-sucedidas. O status HTTP era 200. Queimamos 40 minutos de computação antes que alguém percebesse, porque nada no tratamento de erros verificava a qualidade da saída – apenas o status do transporte.

A solução: DISJUNTOR DE Circuito Consciente da Qualidade

Um disjuntor para agentes precisa rastrear falhas de qualidade: saídas que violam o esquema, falham em invariantes semânticas ou produzem ações inseguras — mesmo quando a própria API é bem-sucedida.


```python
import time
from enum import Enum
from dataclasses import dataclass, field

class CircuitState(Enum):
CLOSED = "closed"
OPEN = "open"
HALF_OPEN = "half_open"

@dataclass
class QualityCircuitBreaker:
failure_threshold: int = 3
reset_timeout: float = 60.0
state: CircuitState = CircuitState.CLOSED
failures: int = 0
last_failure_time: float = field(default_factory=time.time)

def record_quality_failure(self) -> None:
"""Called when LLM output fails validation (not HTTP error)."""
self.failures += 1
self.last_failure_time = time.time()
if self.failures >= self.failure_threshold:
self.state = CircuitState.OPEN

def record_success(self) -> None:
if self.state == CircuitState.HALF_OPEN:
self.state = CircuitState.CLOSED
self.failures = 0

def allow_request(self) -> bool:
if self.state == CircuitState.CLOSED:
return True

if self.state == CircuitState.OPEN:
elapsed = time.time() - self.last_failure_time
if elapsed >= self.reset_timeout:
self.state = CircuitState.HALF_OPEN
return True  # One probe request
return False  # Circuit is open, reject
```

# Semi-aberto: permite exatamente uma sonda
retornar verdadeiro

```python
def should_block(self) -> bool:
return not self.allow_request()

```

Uso em um loop de agente:


```python
breaker = QualityCircuitBreaker(failure_threshold=3, reset_timeout=30.0)

while agent_running:
if breaker.should_block():
logger.warning("Circuit breaker OPEN — skipping LLM call")
sleep(5)
continue

response = call_llm(prompt, system=system)
validated = validate_output(response)

if not validated:
breaker.record_quality_failure()
logger.error(
f"Quality failure #{breaker.failures} — "
f"state: {breaker.state.value}"
)
else:
breaker.record_success()
process_response(response)

```

Visão principal: quando o circuito abrir, pare. Não queime tokens em um modelo que produz lixo. Aguarde o resfriamento, envie uma solicitação de investigação e, se passar na validação do esquema, feche o circuito.

Uma extensão natural é uma cadeia de fallback de modelo: quando o circuito abrir, mude para um modelo mais barato com restrições mais rígidas (temperatura mais baixa, esquema mais rígido, menos ferramentas permitidas). Os disjuntores informam quando parar de confiar em um modelo; fallback cadeias informam para onde ir em seguida.

Padrão 2: Portas de validação antes da execução da ferramenta

Um agente mapeou uma ação delete_all_records para o que interpretou como “limpeza”. A API aceitou. 47 registros foram perdidos antes da próxima revisão humana. O agente estava confiante. A ação era sintaticamente válida. A intenção estava completamente errada.

A regra

Nunca deixe que a resposta de um agente desencadeie diretamente um efeito colateral. Sempre valide antes da execução.


```python
from typing import Any
from dataclasses import dataclass
import json

@dataclass
class ToolCallValidation:
tool_name: str
parameters: dict[str, Any]

ALLOWED_TOOLS = {"query_database", "send_notification", "update_record"}
DESTRUCTIVE_TOOLS = {"delete_record", "archive_project"}
MAX_DELETE_COUNT = 10

class ValidationGate:
def validate(self, call: ToolCallValidation) -> tuple[bool, str]:
"""Returns (is_valid, reason)"""

# 1. Schema: is this a known tool?
if call.tool_name not in ALLOWED_TOOLS | DESTRUCTIVE_TOOLS:
return False, f"Unknown tool: {call.tool_name}"

# 2. Sanity: does the action make sense?
if call.tool_name == "delete_record":
count = call.parameters.get("count", 1)
if count > MAX_DELETE_COUNT:
return False, (
f"Delete count {count} exceeds limit of {MAX_DELETE_COUNT}"
)

# 3. Boundary: is the agent in its allowed scope?
if call.tool_name == "query_database":
table = call.parameters.get("table")
if table == "production_billing":
return False, "Agent cannot access production_billing"

return True, "OK"

```

Três camadas de validação, cada uma capturando uma classe de falha diferente:

Validação de esquema — A saída está estruturalmente correta? Campo obrigatório ausente? Tipo errado? JSON malformado?
Verificações de sanidade – A ação faz sentido? Excluindo 10.000 registros? Provavelmente não.
Aplicação de limites — O agente está operando dentro do escopo permitido? Acesso entre locatários? Visando uma tabela de produção de um fluxo de trabalho de teste?

Isso bloqueia todas as chamadas de ferramenta antes de serem executadas. Nenhuma função de validação separada para lembrar de chamar — ela está no caminho crítico.


```python
gate = ValidationGate()

def execute_tool_call(raw_llm_output: str):
call = parse_tool_call(raw_llm_output)
is_valid, reason = gate.validate(call)

if not is_valid:
logger.warning(f"Tool call blocked: {reason}")
return f"Action blocked: {reason}. Please revise."

# Only reach here after validation passes
return run_actual_tool(call)

```

Princípio de design: restrinja o que o agente pode fazer e você evitará a maioria dos erros na origem. Isso combina diretamente com o insight do trabalho de design de ferramentas – dividir uma ferramenta monolítica em oito ferramentas focadas elimina a maioria das falhas de validação antes que elas possam ocorrer.

Padrão 3: Sagas Idempotentes para Fluxos de Trabalho de Várias Etapas

Um agente fluxo de trabalho de três etapas falhou na etapa 2. A etapa 1 já havia criado um registro de cliente. A retry criou uma duplicata. Duzentos registros órfãos foram encontrados uma semana depois, cada um acionando notificações de cobrança duplicada.

A matemática é desconfortável: um agente que obtém sucesso em 95% das vezes em cada etapa tem apenas 60% de chance de concluir um fluxo de trabalho de 10 etapas de forma limpa (0,95^10). A 90% por etapa, um fluxo de trabalho de 10 etapas é bem-sucedido em apenas 35% das vezes. Cada etapa aumenta o risco e, sem idempotência, cada nova tentativa duplica os efeitos colaterais.

A solução: Checkpoint-Then-Execute com ações de compensação

Tomando emprestado o padrão saga em sistemas distribuídos, cada etapa registra sua conclusão antes da execução e define uma ação de compensação para reversão.


```python
from dataclasses import dataclass
from enum import Enum
from typing import Callable, Optional
import sqlite3

class StepStatus(Enum):
PENDING = "pending"
COMPLETED = "completed"
COMPENSATED = "compensated"

@dataclass
class SagaStep:
name: str
execute: Callable
compensate: Optional[Callable]  # None means read-only or irreversible
status: StepStatus = StepStatus.PENDING

class SagaExecutor:
def __init__(self, db_path: str = ":memory:"):
self.conn = sqlite3.connect(db_path)
self._init_checkpoint_table()

def _init_checkpoint_table(self):
self.conn.execute("""
CREATE TABLE IF NOT EXISTS checkpoints (
step_name TEXT PRIMARY KEY,
status TEXT,
result TEXT
)
""")
self.conn.commit()

def _is_completed(self, step_name: str) -> bool:
row = self.conn.execute(
"SELECT status FROM checkpoints WHERE step_name = ?",
(step_name,)
).fetchone()
return row is not None and row[0] == "completed"

def _record_checkpoint(self, step_name: str, result: str):
self.conn.execute(
"INSERT OR REPLACE INTO checkpoints (step_name, status, result) "
"VALUES (?, 'completed', ?)",
(step_name, result)
)
self.conn.commit()

def execute(self, steps: list[SagaStep]) -> dict[str, str]:
completed_steps = []

for step in steps:
```
se self._is_completed(step.name):
# Idempotente: pule as etapas já concluídas em tentar novamente
continuar

tentar:
```python
result = step.execute()
self._record_checkpoint(step.name, str(result))
completed_steps.append(step)
except Exception as e:
# Rollback: execute compensation in reverse order
logger.error(
f"Step '{step.name}' failed: {e}. "
f"Rolling back {len(completed_steps)} steps."
)
for completed in reversed(completed_steps):
if completed.compensate:
try:
completed.compensate()
except Exception as comp_err:
logger.critical(
f"Compensation for '{completed.name}' failed: "
f"{comp_err}"
)
raise

return {s.name: "completed" for s in steps}


Usage:


def fetch_data(): return api.get_records()
def transform(data): return [process(r) for r in data]
def write_to_db(data):
db.insert_batch(data)
return {"id": db.last_insert_id}
def rollback_write(ctx):
db.delete_batch(ctx["id"])

def send_notification():
smtp.send("Pipeline complete")
def send_correction():
smtp.send("Correction: pipeline was rolled back")

saga = SagaExecutor()

steps = [
SagaStep("fetch_data", execute=fetch_data, compensate=None),  # Read-only
SagaStep("transform", execute=transform, compensate=None),    # Pure function
SagaStep("write_to_db", execute=write_to_db, compensate=rollback_write),
SagaStep("send_notification", execute=send_notification, compensate=send_correction),
]

saga.execute(steps)

```

Classifique cada etapa:

```python
Read-only — safe to retry freely (fetches, queries)
Pure function — safe to retry (transforms, computations)
Reversible — can undo (delete what you created)
Compensatable — can't undo but can correct (send a follow-up notification)
Irreversible — can't undo at all (payment processed). These need the most validation before execution.
Pattern 4: Budget Guardrails for Runaway Loops
```

Os agentes podem entrar em ciclos de raciocínio. O modelo continua chamando ferramentas, gerando respostas, reavaliando – nunca atingindo um estado terminal. As fichas se acumulam. A conta sobe. Nada trava, esse é o problema.

Orçamento de token

Um limite máximo de tokens por sessão do agente. Quando o orçamento se esgota, o agente deve parar e retornar os resultados até o momento ou escalar.


```python
@dataclass
class TokenBudget:
max_tokens: int = 50_000  # Set based on your cost tolerance
used_tokens: int = 0
cost_per_1k: float = 0.01  # Adjust for your model

def remaining(self) -> int:
return max(0, self.max_tokens - self.used_tokens)

def is_exhausted(self) -> bool:
return self.used_tokens >= self.max_tokens

def estimate_cost(self) -> float:
return (self.used_tokens / 1_000) * self.cost_per_1k

def track(self, response_text: str):
# Rough estimate: ~1 token per 4 chars for English
self.used_tokens += len(response_text) // 4

Cycle Budget
```

Além dos tokens, limite o número de ciclos de raciocínio (turnos) que o agente pode realizar. Isso evita loops infinitos de chamadas de ferramentas, mesmo que cada chamada individual seja barata.


```python
@dataclass
class CycleBudget:
max_cycles: int = 15
current_cycle: int = 0

def increment(self) -> bool:
"""Returns True if cycles remain, False if exhausted."""
self.current_cycle += 1
return self.current_cycle <= self.max_cycles

def remaining(self) -> int:
return max(0, self.max_cycles - self.current_cycle)

```

Integrado em um loop de agente:


```python
token_budget = TokenBudget(max_tokens=30_000)
cycle_budget = CycleBudget(max_cycles=12)

while agent_running:
if token_budget.is_exhausted():
logger.warning(
f"Token budget exhausted ({token_budget.used_tokens} used). "
f"Estimated cost: ${token_budget.estimate_cost():.2f}"
)
# Return partial results or escalate
break

if not cycle_budget.increment():
logger.warning(
f"Cycle budget exhausted after {cycle_budget.max_cycles} turns. "
f"Agent may be in a reasoning loop."
)
break

# ... execute agent step ...
token_budget.track(llm_response_text)

if cycle_budget.remaining() <= 3:
logger.info(f"Agent in danger zone: {cycle_budget.remaining()} cycles left")

```

Esses orçamentos protegem contra o modo de falha em que um agente "funciona" perfeitamente — sem exceções, sem travamentos — enquanto slowgasta lentamente seu orçamento de API.

Padrão 5: Escalação Humana para Decisões de Alto Risco

Algumas ações são muito arriscadas para um agente realizar de forma autônoma. Excluindo dados de produção. Envio de comunicações voltadas para o cliente. Alterando a configuração da infraestrutura.

Escalação Baseada em Confiança

Pergunte ao modelo qual é o seu nível de confiança juntamente com o seu raciocínio. Abaixo de um limite, encaminhe para um humano em vez de executar.


ESCALAÇÃO_LIMITE = 0,7
```python
HIGH_RISK_ACTIONS = {"delete_production_data", "send_customer_email",
"modify_infrastructure", "approve_payment"}

def should_escalate(tool_name: str, confidence: float,
validation_result: str) -> bool:
reasons = []

if tool_name in HIGH_RISK_ACTIONS:
reasons.append(f"high-risk action: {tool_name}")

if confidence < ESCALATION_THRESHOLD:
reasons.append(f"low confidence: {confidence:.2f}")

if "unsure" in validation_result.lower():
reasons.append("model expressed uncertainty")

if len(reasons) >= 2:
logger.warning(f"Escalating: {', '.join(reasons)}")
return True

return False
```

# No loop do agente:
if should_escalate(call.tool_name, parsed.confidence, validação_reason):
escalation_queue.put({
```python
"tool": call.tool_name,
"params": call.parameters,
"model_confidence": parsed.confidence,
"model_reasoning": parsed.reasoning,
"suggested_action": call.parameters,
})
response = "Action queued for human review."
else:
response = execute_tool_call(call)

The Escalation Queue
```

O escalonamento persistente significa que ele sobrevive a falhas, reinicializações e reimplantações. Uma fila simples baseada em arquivo funciona:


```python
import json
from pathlib import Path

class EscalationQueue:
def __init__(self, path: str = "escalations.json"):
self.path = Path(path)
self._ensure_file()

def _ensure_file(self):
if not self.path.exists():
self.path.write_text("[]")

def enqueue(self, item: dict):
items = json.loads(self.path.read_text())
items.append({**item, "enqueued_at": time.time()})
self.path.write_text(json.dumps(items, indent=2))

def pending(self) -> list[dict]:
return json.loads(self.path.read_text())

def resolve(self, index: int, decision: str):
items = json.loads(self.path.read_text())
items[index]["decision"] = decision
items[index]["resolved_at"] = time.time()
self.path.write_text(json.dumps(items, indent=2))


Principle: The agent should stop deciding and start asking. Define clear escalation criteria upfront — action type, confidence threshold, validation failure patterns — and honor them. Nothing erodes trust in an agent faster than an autonomous mistake that a human would have caught in three seconds.

Putting It All Together
```

Esses cinco padrões formam camadas:


┌──────────────────── ─────────────────────┐
│ 5. Escalada Humana │ ← Quando parar de decidir
│ 4. Proteção orçamentária (tokens + ciclos) │ ← Quando parar de gastar
│ 3. Sagas Idempotentes │ ← Como se recuperar de uma falha parcial
│ 2. Portas de validação │ ← O que é permitido executar
│ 1. Disjuntores (conscientes de qualidade) │ ← Quando o modelo em si não é confiável
└──────────────────── ─────────────────────┘


A camada 1 captura a degradação do modelo antes de queimar os tokens. A camada 2 bloqueia ações perigosas antes de serem executadas. A camada 3 contém falhas parciais quando elas ocorrem inevitavelmente. A camada 4 limita o raio de explosão quando o agente gira em espiral. A Camada 5 garante que os humanos estejam no controle de decisões irreversíveis.

Juntos, eles transformam um agente de um passivo probabilístico em um sistema que você pode implantar às 17h de uma sexta-feira e dormir a noite toda.

Conclusões acionáveis
Comece com disjuntors na qualidade de saída, não apenas no status HTTP. Valide a conformidade do esquema em cada resposta do LLM. Três falhas consecutivas → circuito aberto → mudança para o modelo fallback.
Bloqueie todas as chamadas de ferramenta. Verificações de esquema, sanidade e limites no caminho crítico – não como uma reflexão tardia.
A idempotência não é negociável para fluxos de trabalhos de várias etapas. Ponto de verificação antes da execução, compensação em caso de falha, pular etapas concluídas em repetir.
Defina tokens rígidos e orçamentos de ciclo desde o primeiro dia. Surpresas de custo vêm de agentes "trabalhando" em loops, e não de agentes travados.
Defina critérios de escalonamento antes de escrever o código do agente. Se não for possível indicar quando um humano deve revisar uma ação, o agente não estará pronto para produção.

A jornada do agente de produção não começa tornando o agente mais inteligente. Tudo começa tornando o agente confiável. Faça isso e a inteligência o seguirá.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
5 padrões de tratamento de erros do agente AI que mantêm seu agente funcionando às 3 da manhã
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção