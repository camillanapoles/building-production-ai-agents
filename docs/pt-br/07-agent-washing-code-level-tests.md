# Lavagem de agente: 5 testes em nível de código para distinguir agentes reais de IA de falsificações

> Artigo #7 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/agent-washing-5-code-level-tests-to-tell-real-ai-agents-from-fakes-2bg6)

---

Um estudo da RAND descobriu que 80-90% dos projetos de agentes de IA falham. Mas aqui está a parte sobre a qual ninguém fala: para começar, muitos desses projetos nunca foram agentes.

Bem-vindo à lavagem com agente - a prática de colocara palavra "agente" em qualquer produto que chame um LLM. Os chatbots tornam-se “agentes conversacionais”. Cron jobs com GPT tornam-se "agentes autônomos". Uma única chamada de API agrupada em um loop while True se torna um "fluxo de trabalho agente".

O termo tem circulado nos círculos empresariais – Forbes, Gartner e r/AI_Agents do Reddit o sinalizaram. Mas a maior parte da cobiça tem como alvo equipes de compras e compradores empresariais. Ninguém escreveu a versão do desenvolvedor: testes concretos em nível de código que você pode executar em suas próprias ferramentas.

Este artigo corrige isso. Cinco funções de teste Python. Execute-os contra qualquer coisa que afirme ser um “agente de IA”. Os resultados dirão mais do que qualquer demonstração de fornecedor.

O que torna algo um agente (não marketing)

Antes dos testes, precisamos de uma definição funcional. Não a versão de marketing – a versão de arquitetura.

Um agente de IA é um sistema com um loop que inclui:

Percepção – lê o estado do ambiente (APIs, bancos de dados, entrada do usuário)
Raciocínio – decide o que fazer a seguir, incluindo quando parar
Ação – executa ferramentas com efeitos colaterais reais
Memória – persiste o contexto entre invocações
Autonomia – opera sem intervenção humana para decisões rotineiras

A palavra-chave é loop. Um pipeline executa A, depois B e depois C. Um agente executa A, observa o resultado, decide se executa B ou D, lida com falhas e continua até que a meta seja atingida.

Aqui está a diferença no pseudocódigo:


```python
# Pipeline (NOT an agent)
def pipeline(input):
result_a = step_a(input)
result_b = step_b(result_a)
return step_c(result_b)  # always the same path

# Agent (has a reasoning loop)
def agent(goal):
state = perceive()
while not goal_met(state):
plan = reason(goal, state, memory)
result = execute(plan.next_action)
memory.store(result)
state = perceive()  # re-evaluate after every action
return state

```

Se o sistema que você está avaliando se parece mais com a primeira função do que com a segunda, é um pipeline com bom marketing. Não há nada de errado com pipelines - mas eles resolvem problemas diferentes dos agentes.

Os 5 testes em nível de código

Esses testes são independentes de estrutura. Eles funcionam quer você esteja avaliando uma ferramenta de fornecedor, uma estrutura de código aberto ou algo que você mesmo construiu. Cada teste tem como alvo uma das cinco propriedades arquitetônicas acima.

Teste 1: Autonomia em Várias Etapas

O que verifica: O sistema pode concluir uma tarefa de várias etapas sem devolver o controle ao usuário entre as etapas?


def test_multi_step_autonomy(agente):
"""Dê ao agente uma tarefa que requer mais de 3 ações sequenciais.
Um verdadeiro agente os completa de forma autônoma.
Um agente falso pede confirmação após cada etapa."""

```python
task = "Find the top Python repos created this week, "
"summarize their READMEs, and save a report to disk."

result = agent.run(task)
```

# Verifique: foi concluído sem avisos humanos intermediários?
assert result.completed, "Agente não concluiu a tarefa"
```python
assert result.human_interventions == 0, (
f"Agent required {result.human_interventions} human interventions "
f"-- a real agent handles routine multi-step tasks autonomously"
)
assert result.steps_executed >= 3, (
f"Only {result.steps_executed} steps -- task requires search, "
f"summarization, and file writing at minimum"
)


FAIL pattern: The system generates a plan but then asks "Should I proceed?" after every step. This is a copilot, not an agent.

PASS pattern: The system searches GitHub, reads READMEs, writes the report, and returns the finished result. You see the output, not a series of permission prompts.

Test 2: Error Recovery
```

O que verifica: Quando uma chamada de ferramenta falha, o sistema se adapta ou trava?


def test_error_recovery(agente, mock_api):
"""Simule uma falha de ferramenta. agentes tente novamente reais ou encontre alternativas.
agentes falsos devolvem o erro ao usuário."""

```python
# Make the primary API fail
mock_api.github.search.side_effect = RateLimitError("429")

task = "Find trending Python repositories from this week."
result = agent.run(task)

# A real agent should try an alternative approach
assert not result.raised_exception, (
"Agent crashed on API failure instead of recovering"
)
assert result.recovery_attempts > 0, (
"Agent made no attempt to recover from the failure"
)
# Check it tried a different strategy (web search, cache, etc.)
assert any(
step.tool != "github.search"
for step in result.steps
), "Agent only retried the same failing call"


FAIL pattern: Error: GitHub API rate limit exceeded. Please try again later. The system hands the error to you because it has no fallback logic.

PASS pattern: The system catches the rate limit, switches to a web search or cached data source, and still delivers results. The error is logged internally, not surfaced as a blocker.

Test 3: Dynamic Tool Selection
```

O que verifica: O sistema escolhe ferramentas com base na tarefa ou segue um caminho codificado?


```python
def test_dynamic_tool_selection(agent):
"""Give two different tasks. A real agent selects different tools.
A fake agent runs the same fixed workflow regardless."""

task_a = "What is the current price of Bitcoin?"
task_b = "Summarize the README of facebook/react on GitHub."

result_a = agent.run(task_a)
result_b = agent.run(task_b)

tools_a = {step.tool for step in result_a.steps}
tools_b = {step.tool for step in result_b.steps}

# The tool sets should be different
assert tools_a != tools_b, (
f"Agent used identical tools for completely different tasks: "
f"{tools_a}. This suggests hardcoded routing, not reasoning."
)
# Each should use contextually appropriate tools
assert any("search" in t or "price" in t for t in tools_a), (
"Bitcoin price task should invoke a search or price API"
)
assert any("github" in t for t in tools_b), (
"GitHub README task should invoke the GitHub API"
)


FAIL pattern: Both tasks route through the exact same chain: parse_input -> call_llm -> format_output. The "tool selection" is actually a single prompt template.

PASS pattern: Task A invokes a web search API. Task B invokes the GitHub API. The agent selected different tools because the tasks required different capabilities.

Test 4: Memory Persistence
```

O que verifica: O sistema lembra o contexto das sessões anteriores?


def test_memory_persistence(agente):
"""Execute uma tarefa e depois referencie-a sem fornecer contexto novamente.
Verdadeiros agentes lembrem-se. agentes falsos sempre começam do zero."""

```python
# Session 1: Give the agent information
agent.run("My project uses Python 3.12 and FastAPI. Remember this.")

# Clear the conversation (new session)
agent.new_session()

# Session 2: Ask something that requires the stored context
result = agent.run(
"Based on my project setup, suggest a testing framework."
)

# The agent should reference Python/FastAPI without being reminded
response_text = result.output.lower()
assert any(
term in response_text
for term in ["fastapi", "python", "pytest", "httpx"]
), (
"Agent showed no awareness of previously stored project context. "
"It either has no memory or memory does not persist across sessions."
)


FAIL pattern: "I don't have information about your project setup. Could you share the details?" The system has no memory beyond the current chat window.

PASS pattern: "Since your project uses FastAPI on Python 3.12, I would recommend pytest with httpx for async endpoint testing." The system recalled stored context from a previous session.

Test 5: Goal Decomposition
```

O que verifica: Dado um objetivo de alto nível, o sistema pode dividi-lo em subtarefas e executá-las?


def test_goal_decomposition(agente):
"""Forneça uma meta complexa e de alto nível. agentes reais a decompõem.
agentes falsos tentam lidar com tudo em uma chamada LLM."""

```python
goal = (
"Audit my project's dependencies for security vulnerabilities, "
"prioritize them by severity, and create a fix plan."
)

result = agent.run(goal)

# Check that the agent broke this into distinct sub-tasks
assert result.steps_executed >= 3, (
f"Only {result.steps_executed} step(s) for a complex audit task. "
f"A real agent decomposes this into scanning, analysis, "
f"and planning phases."
)

# Check for distinct phases (not just repeated LLM calls)
step_descriptions = [step.description for step in result.steps]
has_scan = any("scan" in d or "check" in d for d in step_descriptions)
has_analyze = any("priorit" in d or "sever" in d for d in step_descriptions)
has_plan = any("fix" in d or "plan" in d or "patch" in d for d in step_descriptions)

assert has_scan and has_analyze and has_plan, (
"Agent did not decompose the goal into scan -> analyze -> plan. "
"It likely tried to answer everything in a single prompt."
)


FAIL pattern: The system returns a single LLM response: "Here are some common vulnerabilities to watch for..." It generated advice instead of actually executing an audit.

PASS pattern: The system runs pip audit or safety check, parses the results, groups findings by severity, and generates a prioritized remediation plan with specific package versions to upgrade.

The Agent Spectrum: Not Everything Needs to Be an Agent
```

Aqui está algo que o discurso da lavagem de agentes erra: implica que você deve sempre querer um “agente de verdade”. Você não deveria.

As capacidades do agente existem em um espectro:


Agente copiloto do fluxo de trabalho do pipeline LLM Wrapper
|             |           |          |          |
Ramificação Fixa Única Humana - Autônoma
prompt + cadeia de lógica com in-the-loop com
resposta LLM chama condições loop raciocínio


Cada nível é a ferramenta certa para alguma coisa:

LLM Wrapper: Resuma este texto. Translate esta string. Uma entrada, uma saída. Perfeito.
Pipeline: processe este CSV por meio de três etapas de transformação. Previsível, repetível. Ótimo.
Fluxo de trabalho: encaminhe este ticket de suporte com base na categoria e encaminhe se o sentimento for negativo. Lógica de ramificação sem autonomia total. Sólido.
Copiloto: Ajude-me a escrever este código, sugira o próximo passo, mas deixe-me aprovar antes de executar. O humano permanece no controle. Ideal para decisões de alto risco.
Agente: monitore minha infraestrutura, detecte anomalias, investigue as causas raízes e aplique correções – sem me acordar às 3 da manhã. Autonomia total para tarefas complexas e de várias etapas.

O problema com a lavagem com agente não é a existência de ferramentas mais simples. O problema é que ferramentas mais simples são comercializadas como agentes, então você cria expectativas de comportamento autônomo e, em vez disso, obtém um chatbot.

O custo de errar é em ambos os sentidos:

Excesso de engenharia: construir um agente para uma tarefa que precisa de um pipeline desperdiça computação, adiciona latência e cria modos de falha que um sistema mais simples não teria.
Falta de engenharia: chamar um chatbot de “agente” significa que as equipes constroem fluxos de trabalhos esperando tratamento adaptativo de erros e memória persistente que nunca se materializa.
Como avaliar qualquer ferramenta de agente de IA

Além dos cinco testes de código, faça estas perguntas antes de se comprometer com qualquer ferramenta que afirma ser um agente:

Posso ver o rastreamento da decisão? agentes reais expõem sua cadeia de raciocínio – quais ferramentas eles consideraram, por que escolheram uma em vez de outra, o que tentaram quando algo falhou. Se o sistema for uma caixa preta, você não poderá depurá-lo e não poderá confiar nele.

Ele lida com falhas sem minha intervenção? Tente quebrar algo de propósito. Mate uma API, alimente-a com informações malformadas, dê-lhe instruções contraditórias. O que acontece? Um verdadeiro agente se recupera ou escala normalmente. Um invólucro quebra ou alucina.

A memória persiste entre as sessões? Feche a janela de bate-papo. Volte amanhã. Ele se lembra do contexto do seu projeto, das suas preferências, das suas conversas anteriores? Se toda interação começar do zero, é um chatbot com boa UX.

Ele pode usar ferramentas que não codifiquei? Dê acesso a uma nova API que ele nunca viu. Ele pode ler os documentos e descobrir como usá-los? agentes reais com seleção dinâmica de ferramentas podem se adaptar a novos recursos. Corrigido workflows não pode.

Posso rastrear exatamente o que aconteceu e por quê? Após a conclusão de uma tarefa, você consegue obter um log de execução completo? Quais ferramentas foram chamadas, em que ordem, com quais parâmetros e quais resultados? A observabilidade não é opcional para agentes de produção.

Sistemas como o Nebula são construídos em torno destes princípios – subagentes autônomos, memória persistente, orquestração de ferramentas com rastreamentos de execução – mas os testes acima se aplicam a qualquer ferramenta que você esteja avaliando. A questão não é qual produto você escolhe. A questão é que você teste antes de confiar.

O resultado final

A lavagem de agentes desperdiça o tempo do desenvolvedor e corrói a confiança em ferramentas que poderiam realmente transformar a forma como construímos software. A solução é simples: pare de avaliar os agentes pelas suas demonstrações e comece a avaliá-los pelo seu comportamento sob pressão.

Execute os cinco testes:

Autonomia em várias etapas – ele termina o trabalho sem precisar segurar as mãos?
Recuperação de erros – ela se adapta quando algo quebra?
Seleção dinâmica de ferramentas – ela raciocina sobre quais ferramentas usar?
Persistência de memória – lembra do contexto entre sessões?
Decomposição de metas – ela divide metas complexas em etapas executáveis?

Se uma ferramenta passar em todos os cinco, você terá um agente real. Se falhar na maioria deles, você tem um invólucro com um bom marketing. Ambos podem ser úteis – mas apenas se você souber qual deles está realmente comprando.

Execute esses testes em sua pilha atual esta semana. Você pode se surpreender com o que encontrar.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
Lavagem de agente: 5 testes em nível de código para distinguir agentes reais de IA de falsificações
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção