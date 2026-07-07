# Como testar agentes de IA (antes que eles queimem seu orçamento)

> Artigo #9 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/how-to-test-ai-agents-before-they-burn-your-budget-5c1j)

---

Seu agente passou na demonstração. Ele lidou com os cinco prompts que você testou manualmente. As partes interessadas assentiram. Você enviou.

Em seguida, queimou US$ 153 em 30 minutos em uma tarde de terça-feira.

O agente fez uma consulta ambígua, entrou em um ciclo de raciocínio e ligou para o GPT-4 47 vezes tentando resolver uma questão que deveria ter sido escalada após três tentativas. Ninguém percebeu até que o alerta de cobrança foi disparado. Isso não é hipotético – é o padrão de falha mais comum em agentes de IA de produção e é totalmente evitável.

O problema não é que as equipes ignorem os testes. É que eles aplicam padrões de testes tradicionais a sistemas que quebram esses padrões por design. Os agentes de IA são não determinísticos, têm várias etapas e são capazes de gerar resultados que parecem confiantes e completamente errados. Você não pode afirmar resposta == esperada quando a mesma entrada produz saídas válidas diferentes em cada execução.

Aqui estão cinco padrões de teste que realmente funcionam para agentes de IA. Cada um deles é independente de estrutura, inclui copiar e colar código Python e tem como alvo uma classe específica de falha de produção.

Por que os testes tradicionais são interrompidos para agentes de IA

Os testes unitários assumem saída determinística. Os testes de integração pressupõem APIs estáveis. Ambos assumem que entradas idênticas produzem saídas idênticas. AI agentes viola todas as três suposições simultaneamente.

Os principais desafios:

Não determinismo: o mesmo prompt produz respostas diferentes nas execuções. Ambas as respostas podem estar corretas, mas assertEqual ainda falha.
Estado de várias etapas: um agente que roteia corretamente na etapa 1 ainda pode falhar na etapa 4 devido ao desvio de contexto acumulado. Testar etapas individuais ignora falhas em cascata.
Efeitos colaterais da ferramenta: seu agente chama APIs reais, modifica bancos de dados e envia mensagens. Um teste que exercita o caminho feliz pode enviar acidentalmente 200 e-mails para uma lista de discussão de produção.

Em vez disso, o que precisamos é de uma abordagem de teste em camadas que teste propriedades e comportamentos em vez de resultados exatos.

Abordagem de teste de agente de teste tradicional
Asserções Correspondência exata de saída Propriedades comportamentais
Determinismo Mesma entrada = mesma saída Mesma entrada = saída dentro dos limites
Escopo Função única Rastreamento em várias etapas
Efeitos colaterais Zombado Zombado + com limite de orçamento
Qualidade Binária aprovado/reprovado Pontuado em uma rubrica
Padrão 1: Testes de Esqueleto Determinístico

O primeiro insight: você não precisa testar o LLM. OpenAI e Anthropic empregam milhares de engenheiros para garantir que seus modelos gerem texto coerente. O que você precisa testar é o seu código – a lógica de roteamento, seleção de ferramentas, gerenciamento de estado e orquestração que envolve o modelo.

Zombe do LLM. Teste o esqueleto.


de unittest.mock importar MagicMock

```python
def test_agent_routes_search_queries():
"""Agent should select web_search for factual questions."""
mock_llm = MagicMock()
mock_llm.complete.return_value = {
"tool": "web_search",
"args": {"query": "population of Tokyo 2026"}
}

agent = ResearchAgent(llm=mock_llm)
result = agent.plan("What is the current population of Tokyo?")

assert result.selected_tool == "web_search"
assert "Tokyo" in result.tool_args["query"]
# We're not testing if the LLM chose well --
# we're testing if our code handles the choice correctly.

def test_agent_rejects_dangerous_tool_calls():
"""Agent must block destructive actions without confirmation."""
mock_llm = MagicMock()
mock_llm.complete.return_value = {
"tool": "delete_database",
"args": {"target": "production"}
}

agent = TaskAgent(llm=mock_llm)
result = agent.plan("Clean up old records")

assert result.status == "requires_confirmation"
assert result.selected_tool != "delete_database"  # Should be blocked

```

Os testes de esqueleto são executados em milissegundos porque nunca chamam uma API LLM. Eles detectam os bugs mais importantes: lógica de roteamento que envia consultas para a ferramenta errada, falta de verificações de segurança em operações destrutivas e gerenciamento de estado que elimina o contexto entre as etapas.

Execute-os em cada commit. Eles são gratuitos.

Padrão 2: Teste de repetição do Golden Path

Depois que seu agente lidar com uma tarefa corretamente, registre todo o rastreamento de execução – entradas, chamadas de ferramentas, resultados intermediários e saída final. Essa gravação se torna um teste de regressão.


```python
import json
from pathlib import Path
from difflib import SequenceMatcher

def record_golden_run(agent, prompt, tag):
"""Record a successful agent run as a golden trace."""
trace = agent.run(prompt, record=True)
path = Path(f"tests/golden/{tag}.json")
path.write_text(json.dumps({
"prompt": prompt,
"tool_sequence": [t.name for t in trace.tool_calls],
"tool_args": [t.args for t in trace.tool_calls],
"output": trace.final_output,
"cost": trace.total_cost,
}, indent=2))
return trace

def test_golden_path_regression():
"""Current agent behavior should match recorded golden run."""
golden = json.loads(Path("tests/golden/summarize_docs.json").read_text())
trace = agent.run(golden["prompt"], record=True)

# Tool sequence should be identical
assert [t.name for t in trace.tool_calls] == golden["tool_sequence"]

# Output should be semantically similar (not identical)
similarity = SequenceMatcher(
None, trace.final_output, golden["output"]
).ratio()
assert similarity > 0.70, f"Output drift detected: {similarity:.0%} similar"

# Cost should not spike
assert trace.total_cost <= golden["cost"] * 1.5, (
f"Cost regression: ${trace.total_cost:.2f} vs golden ${golden['cost']:.2f}"
)

```

Os testes do Golden Path respondem a uma pergunta: "O agente ainda funciona da maneira que funcionava quando o verificamos pela última vez?" Execute-os antes de cada atualização de modelo, após alterações imediatas e como um conjunto de regressão semanal.

A sua limitação é óbvia – eles apenas detectam regressões, não novas falhas. É isso que o próximo padrão trata.

Padrão 3: Asserções de Agente Baseadas em Propriedades

Este é o padrão mais valioso neste artigo. Em vez de afirmar o que o agente deve produzir, afirme o que ele nunca deve fazer.

Todo agente de produção possui invariantes – propriedades que devem ser verdadeiras independentemente da entrada. Escreva-os. Em seguida, escreva testes que os apliquem.


```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class AgentProperty:
name: str
check: Callable
severity: str  # "critical" | "warning"

PRODUCTION_PROPERTIES = [
AgentProperty(
name="no_destructive_without_confirm",
check=lambda trace: not any(
t.tool in ["delete", "drop", "truncate"] and not t.user_confirmed
for t in trace.tool_calls
),
severity="critical",
),
AgentProperty(
name="budget_per_request",
check=lambda trace: trace.total_cost < 0.50,
severity="critical",
),
AgentProperty(
name="max_reasoning_steps",
check=lambda trace: len(trace.steps) <= 10,
severity="warning",
),
AgentProperty(
name="no_hallucinated_urls",
check=lambda trace: all(
url_exists(u) for u in extract_urls(trace.final_output)
),
severity="critical",
),
AgentProperty(
name="cites_sources_when_claiming_data",
check=lambda trace: (
not contains_statistics(trace.final_output)
or contains_citations(trace.final_output)
),
severity="warning",
),
]

def test_agent_properties():
"""Run diverse inputs against all agent invariants."""
test_inputs = load_test_corpus("tests/inputs/diverse_queries.jsonl")

violations = []
for prompt in test_inputs:
trace = agent.run(prompt, record=True)
for prop in PRODUCTION_PROPERTIES:
if not prop.check(trace):
violations.append({
"property": prop.name,
"severity": prop.severity,
"input": prompt,
})

critical = [v for v in violations if v["severity"] == "critical"]
assert len(critical) == 0, (
f"{len(critical)} critical property violations: "
f"{[v['property'] for v in critical]}"
)


Property-based testing catches the failures that scare you most: the agent that silently deletes production data, the agent that spends $50 on a single request, the agent that cites a URL that doesn't exist. These are the failures that erode trust and cost real money.
```

Comece com três propriedades: nenhuma ação destrutiva sem confirmação, limite de orçamento por solicitação e máximo de etapas de raciocínio. Adicione mais à medida que descobre novos modos de falha na produção.

Padrão 4: Testes Orçamentários Tripwire

Testes dedicados para o modo de falha mais caro: loops de agentes. Um agente que entra em um loop de raciocínio continuará chamando o LLM até que algo o interrompa. Se nada impedir, sua conta o fará.


```python
import time

def test_ambiguous_query_does_not_loop():
"""Ambiguous input should trigger clarification, not infinite reasoning."""
agent = Agent(max_iterations=20, budget_limit_usd=1.00)

result = agent.run("Do the thing with the stuff from last time")

assert result.iterations < 8, (
f"Agent looped {result.iterations} times on ambiguous input"
)
assert result.cost < 0.30, f"Spent ${result.cost:.2f} on ambiguous query"
assert result.status in ["completed", "clarification_needed", "escalated"]

def test_tool_error_does_not_cascade():
"""Failed tool calls should not trigger retry spirals."""
agent = Agent(max_iterations=20, budget_limit_usd=1.00)

# Inject a tool that always fails
agent.tools["flaky_api"] = lambda **kwargs: raise_error("503 Service Unavailable")

result = agent.run("Fetch the latest report from flaky_api")

retry_count = sum(
1 for t in result.trace.tool_calls if t.tool == "flaky_api"
)
assert retry_count <= 3, f"Agent retried flaky tool {retry_count} times"
assert result.status != "stuck"

def test_worst_case_cost():
"""Maximum possible cost for a single request stays under threshold."""
expensive_prompts = [
"Analyze every commit in the repository and summarize each one",
"Compare all products in our catalog with competitor pricing",
"Research the full history of this company from founding to present",
]

for prompt in expensive_prompts:
result = agent.run(prompt, budget_limit_usd=2.00)
assert result.cost < 2.00, (
f"Request exceeded budget: ${result.cost:.2f} for: {prompt[:50]}..."
)


The $153 incident from the intro? A single budget tripwire test would have caught it. The agent entered a loop because nobody tested what happens when the LLM can't resolve ambiguity. A two-line assertion -- iterations < 10 and cost < 1.00 -- would have flagged it before deployment.
```

Os tripwires de orçamento são slow (eles realmente executam o agente), portanto, execute-os todas as noites ou antes dos lançamentos, em vez de em cada commit.

Padrão 5: Avaliação LLM como Juiz

Quando a qualidade dos resultados é importante – resumos, recomendações, análises – você precisa de um juiz que entenda a linguagem. Use um segundo LLM para avaliar o primeiro.


```python
import openai

def llm_judge(output: str, criteria: str, context: str = "") -> int:
"""Score agent output on a 1-5 scale using a cheap judge model."""
response = openai.chat.completions.create(
model="gpt-4o-mini",  # Cheap judge, not the agent's model
messages=[{
"role": "user",
"content": f"""Rate this AI agent output from 1 (poor) to 5 (excellent).

Criteria: {criteria}
{f'Context: {context}' if context else ''}
```

Saída do agente:
{saída}

Responda APENAS com um único número inteiro de 1 a 5."""
```python
}],
temperature=0,
)
return int(response.choices[0].message.content.strip())

def test_summary_quality():
"""Agent summaries should be accurate, complete, and concise."""
source_doc = load_test_doc("tests/fixtures/quarterly_report.md")
result = agent.run(f"Summarize this document: {source_doc}")

scores = {
"accuracy": llm_judge(result.output, "factual accuracy relative to source", source_doc),
"completeness": llm_judge(result.output, "covers all key points from source", source_doc),
"conciseness": llm_judge(result.output, "no unnecessary filler or repetition"),
}

assert scores["accuracy"] >= 4, f"Accuracy too low: {scores['accuracy']}/5"
assert scores["completeness"] >= 3, f"Missing key points: {scores['completeness']}/5"
assert scores["conciseness"] >= 3, f"Too verbose: {scores['conciseness']}/5"


A caveat: LLM judges have their own biases. GPT-4 tends to rate GPT-4 output generously. Use LLM-as-judge for directional quality tracking ("is this getting better or worse?") rather than absolute pass/fail decisions. Cross-validate with human review on a sample of outputs periodically.

Putting It All Together
```

Você não precisa de todos os cinco padrões no primeiro dia. Aqui está uma tabela de decisão:

Padrão melhor para execução de velocidade de custo quando
Roteamento de ferramentas de testes de esqueleto, verificações de segurança Milissegundos grátis Cada commit
Golden Path Replay Regressão após alterações Baixo Segundos antes das atualizações do modelo
Afirmações de propriedade Violações de segurança, invariantes Baixo-Médio Segundos Cada PR
Orçamento Tripwires Explosões de custos, loops Médios Minutos Todas as noites / pré-lançamento
LLM-as-Juiz Rastreamento de qualidade de saída Low Seconds Weekly / pré-lançamento

Comece com os padrões 1 + 3 + 4. Os testes de esqueleto detectam bugs de roteamento gratuitamente. As afirmações de propriedade captam as falhas assustadoras (ações destrutivas, estouros orçamentários). Os disparadores de orçamento detectam as falhas dispendiosas (loops, novas tentativas espirais). Juntos, esses três padrões evitam cerca de 80% dos incidentes com agentes de produção.

Adicione a repetição do caminho dourado ao atualizar modelos ou fazer grandes alterações imediatas. Adicione o LLM como juiz quando o resultado do seu agente for voltado para o cliente e a qualidade impactar diretamente os resultados do negócio.

Agentes de navios com quem você pode dormir

AI agentes não são funções – são sistemas com comportamento emergente. Um agente bem testado não produz apenas resultados corretos. Ele falha com segurança, falha de forma barata e falha de maneiras que você já previu.

Os padrões neste guia são independentes de estrutura por design. Esteja você construindo com LangGraph, CrewAI, Agents SDK da OpenAI ou chamadas de API brutas, a camada de teste envolve seu agente da mesma maneira.

Plataformas como o Nebula incorporam vários desses padrões no próprio tempo de execução do agente - o rastreamento de tarefas passo a passo fornece os dados de rastreamento que o caminho dourado e os testes de propriedade precisam, a memória integrada oferece observabilidade no raciocínio do agente entre as etapas e as credenciais com escopo evitam totalmente o modo de falha "agente com acesso root".

Mas você não precisa de nenhuma plataforma para começar. Escreva três afirmações de propriedade hoje. Teste o que seu agente nunca deve fazer. Esse é o investimento em testes com maior ROI que você fará neste trimestre.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
Como testar agentes de IA (antes que eles gastem seu orçamento)
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção