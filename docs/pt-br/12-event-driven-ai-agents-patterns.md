# Agentes de IA orientados a eventos: padrões que escalam

> Artigo #12 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/event-driven-ai-agents-patterns-that-scale-3kf4)

---

A maioria dos tutoriais de agentes de IA ensinam você a construir um chatbot que aguarda a entrada do usuário. Mas os agentes de produção não esperam – eles reagem. Uma implantação é concluída e seu agente executa testes de fumaça. Um cliente se inscreve e seu agente envia uma sequência de integração personalizada. Um limite de monitoramento é acionado e seu agente chama o engenheiro de plantão antes mesmo que um humano perceba.

A arquitetura que torna isso possível é o design orientado a eventos. E acertar é a diferença entre agentes que demonstram bem e agentes que executam suas operações.

Este guia aborda quatro padrões de arquitetura orientada a eventos para agentes de IA, cada um com código Python executável que você pode adaptar hoje mesmo. Sem dependência de fornecedor, sem necessidade de Kafka, sem discurso de vendas corporativo – apenas padrões que funcionam.

Por que a pesquisa falha para agentes de produção

Antes de mergulhar nos padrões, vamos esclarecer por que a abordagem padrão falha.

A votação é a solução ingênua: seu agente verifica um banco de dados, API ou caixa de entrada em um cronômetro. "Algum e-mail novo? Não? Verifique novamente em 30 segundos." Funciona em demonstrações. Ele falha na produção por três motivos:

Computação desperdiçada. Seu agente queima a verificação de cota de CPU e API em busca de alterações que não aconteceram. Em escala, isso aumenta rapidamente.
Piso de latência. Seu tempo de resposta é igual ao intervalo de pesquisa. Uma enquete de 30 segundos significa até 30 segundos de atraso em cada evento. Para resposta a incidentes, isso é uma eternidade.
Conexões quadráticas. Se N agentes cada pesquisa M serviços, você terá N x M conexões. Adicione agentes e o sistema se tornará incontrolável.

A arquitetura orientada a eventos elimina todos os três problemas. Os agentes assinam fluxos de eventos e reagem apenas quando algo realmente acontece. A complexidade da conexão cai de O(N x M) para O(N + M). A latência cai para milissegundos. A computação é gasta em trabalho real, não em verificação.

Pesquisas de implantações de produção mostram que sistemas orientados a eventos reduzem a latência de resposta do agente de IA em 70-90% em comparação com abordagens de pesquisa. Isso não é uma melhoria teórica – é a diferença entre detectar uma interrupção em 200 ms e descobri-la 30 segundos depois.

Padrão 1: Fila de Eventos com Agentes Trabalhadores

O padrão orientado a eventos mais simples: os eventos vão para uma fila, os trabalhadores agentes os extraem e os processam. Este é o seu ponto de partida para qualquer sistema de agente orientado a eventos.

Quando usar
agentes de propósito único que processam um tipo de evento
```python
Workloads where ordering matters (FIFO processing)
Systems where you need guaranteed delivery (no dropped events)
Implementation
```

Aqui está uma implementação mínima, mas pronta para produção, usando Redis Streams (você pode trocar em RabbitMQ, SQS ou qualquer corretor de mensagens):


```python
import asyncio
import json
import redis.asyncio as redis
from datetime import datetime
from openai import AsyncOpenAI

client = AsyncOpenAI()
rdb = redis.Redis(host="localhost", port=6379, decode_responses=True)

STREAM = "agent:events"
GROUP = "agent-workers"
CONSUMER = "worker-1"

async def ensure_group():
"""Create consumer group if it does not exist."""
try:
await rdb.xgroup_create(STREAM, GROUP, id="0", mkstream=True)
except redis.ResponseError as e:
if "BUSYGROUP" not in str(e):
raise

async def publish_event(event_type: str, payload: dict):
"""Publish an event to the stream."""
event = {
"type": event_type,
"payload": json.dumps(payload),
"timestamp": datetime.utcnow().isoformat(),
}
event_id = await rdb.xadd(STREAM, event)
print(f"Published {event_type} -> {event_id}")
return event_id

async def process_event(event_id: str, event: dict):
"""Route event to the appropriate AI handler."""
event_type = event["type"]
payload = json.loads(event["payload"])

handlers = {
"deploy.completed": handle_deploy,
"alert.triggered": handle_alert,
"email.received": handle_email,
}

handler = handlers.get(event_type)
if handler:
await handler(payload)
else:
print(f"No handler for event type: {event_type}")

# Acknowledge the event so it is not redelivered
await rdb.xack(STREAM, GROUP, event_id)

async def handle_deploy(payload: dict):
"""AI agent handles deployment verification."""
response = await client.chat.completions.create(
model="gpt-4o",
messages=[
{"role": "system", "content": "You are a deployment verification agent."},
{"role": "user", "content": f"Verify this deployment: {json.dumps(payload)}"}
],
)
print(f"Deploy check: {response.choices[0].message.content[:100]}")

async def handle_alert(payload: dict):
"""AI agent triages monitoring alerts."""
response = await client.chat.completions.create(
model="gpt-4o",
messages=[
{"role": "system", "content": "You are an incident triage agent. Classify severity and suggest next steps."},
{"role": "user", "content": f"Alert: {json.dumps(payload)}"}
],
)
print(f"Alert triage: {response.choices[0].message.content[:100]}")

async def worker_loop():
"""Main worker loop: pull events, process, repeat."""
await ensure_group()
print(f"Worker {CONSUMER} listening on {STREAM}...")

while True:
# Block for up to 5 seconds waiting for new events
messages = await rdb.xreadgroup(
GROUP, CONSUMER, {STREAM: ">"}, count=1, block=5000
)
for stream_name, events in messages:
for event_id, event_data in events:
try:
await process_event(event_id, event_data)
except Exception as e:
print(f"Error processing {event_id}: {e}")
# Event stays unacknowledged -> will be reclaimed

if __name__ == "__main__":
asyncio.run(worker_loop())

```

Isso oferece vários recursos de produção prontos para uso:

Grupos de consumidores: Vários trabalhadores agentes dividem a carga. Adicione trabalhadores para dimensionar horizontalmente.
Agradecimentos: Os eventos não são removidos até que sejam explicitamente reconhecidos. Se um trabalhador travar, os eventos não confirmados serão reenviados.
Contrapressão: os trabalhadores realizam eventos em seu próprio ritmo. Um aumento nas filas de eventos em vez de sobrecarregar seus agentes.
Dimensionando

Para adicionar mais trabalhadores, basta executar instâncias adicionais com nomes CONSUMER diferentes. O Redis lida com a atribuição de partições automaticamente. Três trabalhadores processando eventos de implantação? Cada um pega aproximadamente um terço da carga sem nenhuma alteração de configuração.

Padrão 2: Distribuição para Processamento de Agente Paralelo

Às vezes, um único evento precisa acionar vários agentes simultaneamente. Um novo cliente se cadastra e você precisa: enviar um email de boas-vindas, provisionar sua conta, atualizar o CRM e avisar a equipe de vendas. Sequencialmente, isso leva 4x o tempo que deveria.

O fan-out resolve isso transmitindo um evento para N assinantes, cada um processando em paralelo.

Implementação

Redis Pub/Sub é o mecanismo de distribuição mais simples. Para maior durabilidade, você usaria vários grupos de consumidores no mesmo Redis Stream (cada grupo recebe cada mensagem de forma independente).


```python
import asyncio
import json
import redis.asyncio as redis

rdb = redis.Redis(host="localhost", port=6379, decode_responses=True)
STREAM = "events:customer"

async def setup_fan_out():
"""Create independent consumer groups for parallel processing."""
groups = ["email-agent", "provisioning-agent", "crm-agent", "sales-notifier"]
for group in groups:
try:
await rdb.xgroup_create(STREAM, group, id="0", mkstream=True)
except redis.ResponseError:
pass  # Group already exists

async def agent_worker(group_name: str, handler):
"""Generic agent worker that processes events for its group."""
consumer = f"{group_name}-1"
print(f"[{group_name}] Listening...")

while True:
messages = await rdb.xreadgroup(
group_name, consumer, {STREAM: ">"}, count=1, block=5000
)
for _, events in messages:
for event_id, data in events:
payload = json.loads(data.get("payload", "{}"))
try:
await handler(payload)
await rdb.xack(STREAM, group_name, event_id)
except Exception as e:
print(f"[{group_name}] Failed: {e}")

async def email_handler(payload):
print(f"[email] Sending welcome to {payload.get('email')}")
await asyncio.sleep(0.5)  # Simulate API call

async def provision_handler(payload):
print(f"[provision] Creating workspace for {payload.get('user_id')}")
await asyncio.sleep(1.0)

async def crm_handler(payload):
print(f"[crm] Adding {payload.get('email')} to CRM")
await asyncio.sleep(0.3)

async def sales_handler(payload):
print(f"[sales] Notifying team about {payload.get('plan')} signup")
await asyncio.sleep(0.2)

async def main():
await setup_fan_out()
# All agents run in parallel, each gets every event independently
await asyncio.gather(
agent_worker("email-agent", email_handler),
agent_worker("provisioning-agent", provision_handler),
agent_worker("crm-agent", crm_handler),
agent_worker("sales-notifier", sales_handler),
)

if __name__ == "__main__":
asyncio.run(main())

```

O principal insight: cada grupo de consumidores mantém seu próprio cursor de leitura. Publicar um evento customer.signup significa que todos os quatro agentes o processam de forma independente. Se o agente de email for slow, ele não bloqueará o agente de provisionamento. Se o agente CRM travar, ele continuará a partir do último evento reconhecido sem afetar os demais.

Quando o Fan-Out fica complicado

A distribuição é poderosa, mas introduz um desafio de coordenação: o que acontece quando vários agentes precisam ser concluídos antes de uma ação final? Por exemplo, você deseja enviar um e-mail de "configuração concluída" somente após a conclusão do provisionamento E das atualizações do CRM.

A solução mais limpa é o padrão de evento de conclusão: cada agente emite um evento de conclusão quando termina. Um agente coordenador assina todos os eventos de conclusão e aciona a ação final quando todos os pré-requisitos são atendidos.


```python
# Each agent emits when done:
await publish_event("provision.completed", {"user_id": uid})
await publish_event("crm.updated", {"user_id": uid})

# Coordinator checks if all steps are complete:
async def coordinator(payload):
user_id = payload["user_id"]
status = await rdb.hgetall(f"onboarding:{user_id}")
if status.get("provisioned") and status.get("crm_updated"):
await publish_event("onboarding.complete", {"user_id": user_id})

Pattern 3: Event Sourcing for Auditable Agent Decisions
```

Em setores regulamentados ou aplicações de alto risco, você precisa saber exatamente o que seu agente fez e por quê. A fonte de eventos registra cada mudança de estado como um evento imutável, criando uma trilha de auditoria completa que você pode reproduzir para reconstruir qualquer estado passado.

Isso é importante para agentes de IA porque os resultados do LLM são estocásticos. A mesma entrada pode produzir saídas diferentes. Quando um cliente pergunta “por que seu agente rejeitou minha inscrição?”, você precisa mostrar as entradas exatas, a saída exata do modelo e a lógica de decisão exata – o que não é uma estimativa melhor.

Implementação
```python
import json
import hashlib
from dataclasses import dataclass, asdict
from datetime import datetime
from typing import Any

@dataclass
class AgentEvent:
event_id: str
event_type: str
agent_id: str
timestamp: str
payload: dict
parent_event_id: str | None = None  # For causal chains
checksum: str = ""

def __post_init__(self):
if not self.checksum:
content = json.dumps(
{"type": self.event_type, "payload": self.payload,
"agent": self.agent_id, "ts": self.timestamp},
sort_keys=True,
)
self.checksum = hashlib.sha256(content.encode()).hexdigest()[:16]

class EventStore:
"""Append-only event store for agent decision auditing."""

def __init__(self):
self._events: list[AgentEvent] = []

def append(self, event: AgentEvent):
self._events.append(event)

def get_agent_history(self, agent_id: str) -> list[AgentEvent]:
return [e for e in self._events if e.agent_id == agent_id]

def get_causal_chain(self, event_id: str) -> list[AgentEvent]:
"""Trace the full decision chain for a given event."""
chain = []
current_id = event_id
while current_id:
event = next((e for e in self._events if e.event_id == current_id), None)
if not event:
break
chain.append(event)
current_id = event.parent_event_id
return list(reversed(chain))

def replay_to(self, timestamp: str) -> list[AgentEvent]:
"""Get all events up to a point in time for state reconstruction."""
return [e for e in self._events if e.timestamp <= timestamp]

# Usage: recording an agent's loan decision
store = EventStore()

# Step 1: Application received
store.append(AgentEvent(
event_id="evt_001",
event_type="application.received",
agent_id="loan-reviewer",
timestamp=datetime.utcnow().isoformat(),
payload={"applicant": "user_42", "amount": 50000},
))

# Step 2: Agent analyzed credit data
store.append(AgentEvent(
event_id="evt_002",
event_type="credit.analyzed",
agent_id="loan-reviewer",
timestamp=datetime.utcnow().isoformat(),
payload={"score": 720, "risk_level": "medium", "model": "gpt-4o",
"prompt_hash": "a3f2c1d8", "raw_output": "Applicant shows..."},
parent_event_id="evt_001",
))

# Step 3: Decision made
store.append(AgentEvent(
event_id="evt_003",
event_type="application.approved",
agent_id="loan-reviewer",
timestamp=datetime.utcnow().isoformat(),
payload={"decision": "approved", "conditions": ["income_verification"]},
parent_event_id="evt_002",
))

# Audit: trace how the decision was made
chain = store.get_causal_chain("evt_003")
for event in chain:
print(f"{event.event_type}: {event.payload}")

```

O campo parent_event_id cria uma cadeia causal. Cada decisão do agente está vinculada ao evento que a desencadeou. Quando os auditores perguntam “como o agente decidiu aprovar este empréstimo?”, você percorre a cadeia: solicitação recebida -> crédito analisado (com modelo exato, prompt e resultado) -> decisão tomada.

Somas de verificação para detecção de adulteração

Observe o campo de soma de verificação. Cada evento recebe um hash SHA-256 de seu conteúdo. Se alguém modificar um evento após o fato, a soma de verificação não corresponderá. Isto é essencial para a conformidade em aplicações financeiras, de saúde e jurídicas, onde é necessário provar que a trilha de auditoria não foi alterada.

Padrão 4: Orquestração Saga para fluxos de trabalho de várias etapas

Os fluxos de trabalho dos agentes do mundo real abrangem várias etapas, vários serviços e, às vezes, vários dias. Uma saga coordena esses fluxos de trabalhos de longa duração e - criticamente - lida com falhas com ações compensatórias que desfazem o trabalho parcial.

Considere um agente de atendimento de comércio eletrônico: ele precisa cobrar o cartão, reservar estoque, agendar remessa e enviar confirmação. Se o envio falhar, você precisará liberar o estoque e reembolsar o cartão. Sem a orquestração da saga, as falhas parciais deixam o sistema em um estado inconsistente.

Implementação
```python
import asyncio
from dataclasses import dataclass, field
from enum import Enum
from typing import Callable, Any

class StepStatus(Enum):
PENDING = "pending"
RUNNING = "running"
COMPLETED = "completed"
FAILED = "failed"
COMPENSATED = "compensated"

@dataclass
class SagaStep:
name: str
execute: Callable
compensate: Callable  # Undo action if later steps fail
status: StepStatus = StepStatus.PENDING
result: Any = None
error: str = ""

@dataclass
class Saga:
name: str
steps: list[SagaStep] = field(default_factory=list)
context: dict = field(default_factory=dict)

async def run(self) -> bool:
"""Execute all steps. On failure, compensate completed steps."""
completed: list[SagaStep] = []

for step in self.steps:
step.status = StepStatus.RUNNING
try:
step.result = await step.execute(self.context)
step.status = StepStatus.COMPLETED
completed.append(step)
print(f"  [ok] {step.name}")
except Exception as e:
step.status = StepStatus.FAILED
step.error = str(e)
print(f"  [FAIL] {step.name}: {e}")
# Compensate all completed steps in reverse order
await self._compensate(completed)
return False

return True

async def _compensate(self, completed: list[SagaStep]):
"""Undo completed steps in reverse order."""
print("  Rolling back...")
for step in reversed(completed):
try:
await step.compensate(self.context)
step.status = StepStatus.COMPENSATED
print(f"  [undo] {step.name}")
except Exception as e:
print(f"  [undo-FAIL] {step.name}: {e}")
# Log for manual intervention

# Define the workflow steps
async def charge_card(ctx):
# Call payment API
ctx["charge_id"] = "ch_abc123"
return {"charged": 99.99}

async def refund_card(ctx):
print(f"    Refunding charge {ctx.get('charge_id')}")

async def reserve_inventory(ctx):
ctx["reservation_id"] = "res_xyz"
return {"reserved": True}

async def release_inventory(ctx):
print(f"    Releasing reservation {ctx.get('reservation_id')}")

async def schedule_shipping(ctx):
# Simulate a failure
raise Exception("Carrier API timeout")

async def cancel_shipping(ctx):
print("    Cancelling shipping request")

async def main():
saga = Saga(
name="order-fulfillment",
steps=[
SagaStep("charge_card", charge_card, refund_card),
SagaStep("reserve_inventory", reserve_inventory, release_inventory),
SagaStep("schedule_shipping", schedule_shipping, cancel_shipping),
],
context={"order_id": "order_789"},
)

print(f"Running saga: {saga.name}")
success = await saga.run()
print(f"Result: {'Success' if success else 'Rolled back'}")

if __name__ == "__main__":
asyncio.run(main())

```

Saída quando o envio falha:


Saga em execução: cumprimento de pedidosllment
[ok] cartão_de_carga
[ok] reserva_inventário
[FAIL] agendamento_shipping: API da operadora timeout
Revertendo...
[desfazer] reserve_inventory
Liberando reserva res_xyz
[desfazer] charge_card
Cobrança de reembolso ch_abc123
Resultado: revertido


O padrão saga é essencial para agentes de IA que interagem com serviços externos. As chamadas LLM podem timeout, as APIs podem retornar erros e os limites de taxa podem atingir a qualquer momento. Sem ações compensatórias, cada falha deixa seu sistema em um estado desconhecido.

Endurecimento de produção: novas tentativas, letras mortas e observabilidade

Os quatro padrões acima fornecem a arquitetura. Mas os sistemas de produção precisam de mais três peças para serem confiáveis.

Tentar novamente com espera exponencial

Falhas transitórias são comuns – falhas de rede, limites de taxa, inicializações a frio. Tentar novamente com espera exponencial lida com eles normalmente:


```python
import asyncio
import random

async def retry_with_backoff(fn, max_retries=3, base_delay=1.0):
"""Retry a function with exponential backoff and jitter."""
for attempt in range(max_retries + 1):
try:
return await fn()
except Exception as e:
if attempt == max_retries:
raise  # Final attempt failed, propagate
delay = base_delay * (2 ** attempt) + random.uniform(0, 0.5)
print(f"Retry {attempt + 1}/{max_retries} in {delay:.1f}s: {e}")
await asyncio.sleep(delay)

```

O jitter (adição aleatória) evita problemas de rebanho trovejantes quando vários agentes novas tentativas simultaneamente no mesmo serviço.

Filas de cartas mortas

Quando as novas tentativas se esgotam, os eventos vão para uma fila de mensagens não entregues (DLQ) em vez de serem descartados silenciosamente. Isso oferece uma rede de segurança para investigação manual:


```python
DLQ_STREAM = "agent:dead-letters"

async def send_to_dlq(event_id: str, event: dict, error: str):
"""Move a failed event to the dead letter queue."""
await rdb.xadd(DLQ_STREAM, {
"original_event_id": event_id,
"original_stream": STREAM,
"event_data": json.dumps(event),
"error": error,
"failed_at": datetime.utcnow().isoformat(),
"retry_count": "3",
})
# Acknowledge the original event so it stops being redelivered
await rdb.xack(STREAM, GROUP, event_id)

```

Verifique seu DLQ diariamente. Os padrões em eventos com letras mortas revelam problemas sistêmicos: se o mesmo tipo de evento continuar falhando, você terá um bug, não um erro transitório.

Log estruturado para observabilidade do agente

Os sistemas orientados a eventos são mais difíceis de depurar do que os sistemas de solicitação-resposta porque não há uma única solicitação thread a seguir. logging estruturado com IDs de correlação resolve isso:


```python
import structlog

log = structlog.get_logger()

async def process_event(event_id: str, event: dict):
logger = log.bind(
event_id=event_id,
event_type=event["type"],
correlation_id=event.get("correlation_id", event_id),
)
logger.info("event.received")

try:
result = await handle(event)
logger.info("event.processed", result=result)
except Exception as e:
logger.error("event.failed", error=str(e))
raise

```

O correlação_id segue um evento por toda a cadeia de distribuição. Quando quatro agentes processam a mesma inscrição de cliente, você pode filtrar os logs por ID de correlação para ter uma visão completa.

Escolhendo o padrão certo

Aqui está uma referência rápida para combinar problemas com padrões:

Padrão de cenário Por que
Processe eventos um por vez Fila + Trabalhadores Simples, ordenado e escalável
Um evento aciona vários agentes Fan-Out Processamento paralelo e independente
Requisitos de conformidade ou auditoria Fonte de eventos Trilha imutável e reproduzível
workflows de várias etapas com Saga de reversão Ações de compensação em caso de falha
Altothroughput com necessidades mistas Combine padrões Fila para ingestão, distribuição para distribuição

Na prática, os sistemas de produção combinam padrões. Você pode usar uma fila para ingestão, distribuição para agentes especializados, fornecimento de eventos para a trilha de auditoria e sagas para fluxos de trabalhos que abrangem serviços externos.

Plataformas como Nebula cuidam de grande parte dessa infraestrutura para você – gatilhos orientados a eventos, novas tentativas automáticas e coordenação multiagente são integrados ao tempo de execução do agente, para que você possa se concentrar na lógica do agente em vez de no encanamento. Mas compreender os padrões ajuda a depurar problemas e tomar melhores decisões arquitetônicas, independentemente da plataforma usada.

O que construir a seguir

Se você estiver criando agentes orientados a eventos hoje, comece com o Padrão 1 (fila + trabalhadores) para seu tipo de evento mais comum. Faça o básico funcionar: publique eventos, consuma-os, reconheça-os, lide com falhas.

Em seguida, adicione distribuição quando precisar de processamento paralelo, fornecimento de eventos quando precisar de auditabilidade e sagas quando seus fluxos de trabalhos abrangem vários serviços.

Os padrões neste guia são independentes de estrutura por design. Esteja você usando LangGraph, CrewAI, PydanticAI ou chamadas de API brutas, a camada de arquitetura orientada a eventos fica abaixo da estrutura do agente. É a base que torna tudo o mais confiável.

Os melhores agentes não são aqueles com as instruções mais inteligentes. São eles que nunca abandonam um evento, nunca deixam um fluxo de trabalho pela metade e nunca perdem a noção do que fizeram e por quê.

Esta é a Parte 5 da série Building Production AI Agents. Anterior: Padrões de arquitetura de agentes de IA orientados a eventos.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
Agentes de IA orientados a eventos: padrões que escalam
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção