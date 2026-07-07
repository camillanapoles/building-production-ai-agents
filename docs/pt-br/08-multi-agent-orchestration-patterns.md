# Orquestração multiagente: um guia para padrões que funcionam

> Artigo #8 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/multi-agent-orchestration-a-guide-to-patterns-that-work-4881)

---

Cada artigo multiagente abre com o mesmo argumento: seu único agente está falhando, você precisa de cinco agentes, aqui estão os padrões. A internet está cheia de catálogos de padrões. O que falta é honestidade sobre quando a orquestração multiagente realmente ajuda – e quando torna tudo pior.

Este artigo é diferente. Abordaremos os padrões, mas começaremos com a pergunta que ninguém faz: você realmente precisa de multiagente? Em seguida, examinaremos os quatro padrões que cobrem 90% dos casos de uso de produção, mostraremos o código Python independente de estrutura e faremos as contas de custos que a maioria dos guias ignora.

Isso faz parte da série Building Production AI Agents. Se você ainda não leu Por que seu fluxo de trabalho de IA quebra em escala, comece por aí: ele cobre as três paredes (prompts monolíticos, explosão de ferramentas, estado de amnésia) que empurram as equipes para o multiagente em primeiro lugar.

Quando você realmente precisa de multiagente

Antes de escolher um padrão, confirme se você tem um problema multiagente. A maioria das equipes adota o multiagente porque parece sofisticado. Depois, eles passam meses depurando a sobrecarga de coordenação que um único agente bem estruturado teria evitado completamente.

Você precisa de orquestração multiagente quando atinge um ou mais destes limites:

1. Saturação da janela de contexto. O histórico de conversas, os resultados da ferramenta e o raciocínio intermediário do seu agente consomem mais de 80% da janela de contexto. O modelo gasta mais tokens lendo o histórico do que pensando na etapa atual. Um rastreamento ReAct de 20 etapas com saídas de ferramenta pode facilmente atingir 50.000 tokens de contexto acumulado. Nesse ponto, a atenção do modelo é diluída em tanta história que ele começa a perder instruções e a alucinar chamadas de ferramentas.

2. A contagem de ferramentas excede 12. A pesquisa da Anthropic confirma que a precisão do agente degrada significativamente após 15 ferramentas e entra em colapso após 50. Cada definição de ferramenta na janela de contexto compete pela atenção do modelo durante a seleção. Abordamos isso detalhadamente em Sobrecarga de ferramenta MCP. Se o seu único agente tiver mais de 20 ferramentas, a decomposição não é opcional – é a solução.

3. Erro composto em cadeias longas. Um erro no passo 4 de uma cadeia de 20 passos se propaga silenciosamente. No passo 18, você tem um resultado confiante, coerente e completamente errado. Cadeias mais curtas com transferências entre agentes especializados criam pontos de verificaçãos naturais onde os erros surgem em vez de se tornarem uma bola de neve.

Se nada disso se aplicar, continue como agente único. Adicione um roteador para escolher o prompt do sistema correto por solicitação. Agrupe suas ferramentas em clusters carregados dinamicamente. Estas são soluções mais baratas e simples do que introduzir a coordenação de agentes.

Os 4 padrões que cobrem 90% dos casos de uso

A internet descreve 6, 7 e até 10 padrões de orquestração. Na produção, quatro padrões cuidam de quase tudo. O resto – swarm, mesh, chat em grupo – são de nicho ou ativamente prejudiciais quando aplicados em excesso. Comece simples. Passe para a complexidade somente quando o padrão simples falhar comprovadamente.

Padrão 1: Pipeline Sequencial

Os agentes funcionam em uma ordem fixa. Cada agente recebe a saída do anterior. Pense: pesquise, depois escreva, depois revise e depois edite.


Entrada do usuário --> [Pesquisador] --> [Escritor] --> [Revisor] --> Resultado Final


Cada agente carrega um prompt específico e um pequeno conjunto de ferramentas. O pesquisador possui pesquisa na web. O escritor possui ferramentas de formatação. O revisor tem diretrizes de estilo. Nenhum agente fica sobrecarregado.

Quando usar: Seu workflow possui uma ordem de operações definida. Cada etapa depende da saída anterior. Você precisa de uma trilha de auditoria da contribuição de cada agente.

Quando evitar: As subtarefas são independentes e podem ser executadas em paralelo. Você está perdendo tempo executando-os sequencialmente.

Perfil de custo: Total de tokens = entrada + soma de todas as saídas repassadas. Um pipeline de 3 agentes com entrada de 1.000 tokens e saídas de 500 tokens custa aproximadamente 1K + 1,5K + 2K = 4.500 tokens. Isso é 4,5x uma chamada de agente único.

Padrão 2: Fan-Out/Fan-In (Simultâneo)

A mesma entrada vai para vários agentes simultaneamente. Suas saídas são coletadas e mescladas por um agregador.


+--> [Revisor de Segurança] --+
Entrada do usuário ----> +--> [Revisor de desempenho] ​​--+--> [Agregador] --> Resultado final
+--> [Revisor de estilo] --+


Quando usar: as subtarefas são genuinamente independentes. A velocidade é mais importante do que o custo do token. Você pode escrever uma estratégia de agregação significativa.

Quando evitá-lo: as subtarefas dependem dos resultados umas das outras. Seu agregador é fraco – um sintetizador ruim perde informações ou introduz contradições.

```python
Cost profile: Total tokens = (N agents x input tokens) + aggregation. More predictable than sequential, but scales linearly with agent count.

Pattern 3: Router / Handoff

A lightweight classifier picks which specialist handles the request. Only one specialist runs per request. This is the most common production pattern because most workloads are not "do five things at once" -- they are "figure out which one thing to do, then do it well."


User Input --> [Router] --> [Email Agent]
--> [Calendar Agent]
--> [Code Review Agent]

```

Quando usar: as solicitações se enquadram em categorias distintas. Cada categoria precisa de ferramentas diferentes, avisos diferentes, comportamento diferente. Você não pode prever a categoria antecipadamente.

Quando evitá-lo: Toda solicitação exige a contribuição de todos os especialistas. Isso é fan-out, não roteamento.

Perfil de custo: padrão multiagente mais barato. Uma pequena chamada de roteamento + uma chamada de especialista. A sobrecarga total é apenas o custo de classificação do roteador.

```python
Pattern 4: Hierarchical (Manager + Teams)

A manager agent decomposes a complex task, delegates subtasks to team leads, and team leads coordinate workers. Each level adds abstraction: the manager reasons about strategy, team leads handle tactics, workers execute.


[Manager]
|-- [Research Lead] --> [Web Searcher] --> [Doc Analyzer]
|-- [Writing Lead]  --> [Drafter] --> [Editor]

```

Quando usar: A tarefa abrange vários domínios. Nenhum agente isolado pode conter todo o contexto. Você precisa de mais de 10 agentes organizados em grupos lógicos.

Quando evitá-lo: Sua tarefa cabe em um domínio. A coordenação hierárquica adiciona 2 a 4 segundos de latência por nível. Uma hierarquia de dois níveis em uma tarefa simples é como contratar um gerente de projeto para um projeto individual.

Perfil de custo: Maior sobrecarga. Cada nível adiciona uma chamada completa de LLM para planejamento e síntese. Uma hierarquia de 3 níveis adiciona no mínimo 6 segundos de latência antes de qualquer trabalho ser iniciado.

Seleção de padrões: combine sintomas com soluções

Em vez de um fluxograma, use este diagnóstico. Encontre seu sintoma, obtenha seu padrão.

Seu padrão de sintomas Por que
Agente esquece instruções no meio da tarefa, janela de contexto está cheia Pipeline sequencial Divida o trabalho em etapas. Cada agente obtém um contexto novo e focado.
O agente escolhe a ferramenta errada em mais de 30% das vezes Roteador/Handoff Dê a cada especialista apenas as ferramentas necessárias. O roteador escolhe o especialista, não a ferramenta.
As tarefas demoram muito porque as subtarefas são executadas sequencialmente, mas não dependem umas das outras. Fan-Out/Fan-In Paraleliza o trabalho independente. Hora do relógio = sloagente oeste, não soma de todos os agentes.
Um único agente não pode conter contexto suficiente em vários domínios. Hierárquico Distribuir contexto entre níveis. Nenhum agente precisa de uma visão completa.
Nenhuma das opções acima - o agente funciona bem, mas poderia ser melhor Fique agente único Sério. Otimize seu prompt, agrupe suas ferramentas, adicione resultados estruturados. Não adicione sobrecarga de coordenação desnecessária.
O código: um roteador multiagente independente de estrutura

Aqui está o padrão roteador/transferência em Python simples. Sem dependências de estrutura. Funciona com qualquer API LLM que suporte chamada de função.


```python
import json
from openai import OpenAI

client = OpenAI()

# --- Specialist definitions ---
SPECIALISTS = {
"email": {
"prompt": "You are an email specialist. Triage, draft, and manage emails.",
"tools": [send_email, search_inbox, archive_email],
},
"calendar": {
"prompt": "You are a calendar specialist. Check availability and schedule meetings.",
"tools": [check_calendar, create_event],
},
"research": {
"prompt": "You are a research specialist. Search the web and summarize findings.",
"tools": [web_search, scrape_page, summarize],
},
}

# --- Router: one cheap LLM call to classify ---
def route_request(user_input: str) -> str:
"""Classify the request into a specialist category."""
response = client.chat.completions.create(
model="gpt-4o-mini",  # cheap, fast model for routing
messages=[
{"role": "system", "content": (
"Classify the user request into exactly one category: "
f"{", ".join(SPECIALISTS.keys())}. "
"Return JSON: {\"category\": \"...\", \"confidence\": 0.0-1.0}"
)},
{"role": "user", "content": user_input},
],
response_format={"type": "json_object"},
)
result = json.loads(response.choices[0].message.content)
return result["category"] if result["confidence"] > 0.7 else "unknown"

# --- Execute: specialist runs with focused context ---
def run_specialist(category: str, user_input: str) -> str:
"""Run the specialist agent with its dedicated prompt and tools."""
spec = SPECIALISTS[category]
response = client.chat.completions.create(
model="gpt-4o",  # capable model for execution
messages=[
{"role": "system", "content": spec["prompt"]},
{"role": "user", "content": user_input},
],
tools=[to_openai_tool(t) for t in spec["tools"]],
)
return handle_tool_calls(response)

# --- Main loop ---
def handle_request(user_input: str) -> str:
category = route_request(user_input)
if category == "unknown":
return "I'm not sure what you need. Could you clarify?"
return run_specialist(category, user_input)

```

Observe três coisas:

O roteador usa um modelo barato. gpt-4o-mini para classificação, gpt-4o para execução. O roteamento é uma tarefa de classificação simples – não desperdice tokens caros nisso.

Cada especialista vê apenas suas próprias ferramentas. O agente de e-mail possui 3 ferramentas. O agente do calendário tem 2. Ninguém tem 10+. Esta é a correção de explosão da ferramenta aplicada estruturalmente.

Sem estrutura. São 40 linhas de Python. Você pode trocar OpenAI por Anthropic, Google ou um modelo local. O padrão é o mesmo, independentemente do fornecedor.

Os custos sobre os quais ninguém fala

Cada artigo multiagente mostra o diagrama de arquitetura. Nenhum deles mostra a conta.

Multiplicação de tokens

Os sistemas multiagentes não apenas usam mais tokens – eles os multiplicam.

pipeline sequencial (3 agentes, entrada de 1K, saídas de 500 tokens):

Agente 1: 1.000 tokens de entrada
Agente 2: 1.000 + 500 = 1.500 tokens de entrada
Agente 3: 1.000 + 500 + 500 = 2.000 tokens de entrada
Total: 4.500 tokens de entrada (4,5x uma chamada agente único)

Fan-out (3 agentes, entrada de 1K):

Cada agente: 1.000 tokens de entrada
```python
Aggregator: 1,000 + (3 x 500) = 2,500 input tokens
Total: 5,500 input tokens (5.5x a single-agent call)
```

Bate-papo em grupo (o padrão que a maioria dos artigos recomenda primeiro, mas deve ser recomendado por último):

5 agentes, 10 rodadas, mensagens com 300 tokensrage
Cada rodada: cada agente lê TODAS as mensagens anteriores
Contexto total: mais de 15.000 tokens apenas no histórico acumulado – antes de qualquer raciocínio real

O padrão de roteador/transferência? Uma chamada de roteamento (~200 tokens) mais uma chamada especializada (~1.000 tokens). 1.200 fichas no total. É por isso que é o padrão mais comum na produção.

Empilhamento de latência

Cada agente na cadeia adiciona de 1 a 3 segundos de LLM latência. Um pipeline sequencial de 3 agentes adiciona no mínimo de 3 a 9 segundos. Um sistema hierárquico com 3 níveis adiciona mais de 6 segundos antes que qualquer trabalhador comece a executar. Para fluxos de trabalhos em lote, tudo bem. Para bate-papo em tempo real, é um obstáculo.

Sobrecarga de depuração

Um bug em um sistema agente único significa ler um rastreamento de conversa. Um bug em um sistema multiagente significa reconstruir fluxos de mensagens entre 3-10 agentes, descobrir qual agente tomou a decisão errada e entender como essa decisão se propagou através do pipeline. Invista em logging em cada decisão de roteamento, em cada entrada/saída de agente e em cada chamada de ferramenta. Sem observabilidade, os sistemas multiagentes são caixas pretas.

Antipadrões que matam sistemas multiagentes
O orquestrador prematuro

Você tem 3 ferramentas e 1 caso de uso, mas construiu um roteador com 4 agentes especializados porque o diagrama de arquitetura parecia legal. Seu sistema agora é mais lentomais caro e mais difícil de depurar do que seria um único agente com 3 ferramentas. Multiagente é uma estratégia de expansão, não uma estratégia inicial.

O Deus Orquestrador

Seu agente orquestrador tem um prompt do sistema de 3.000 tokens que descreve cada especialista, cada regra de roteamento e cada caso extremo. Você recriou o problema do prompt do monolito - apenas o moveu um nível acima. Deixe o orquestrador mudo: classifique o pedido, escolha o especialista, entregue. Nada mais.

A malha tagarela

Você leu sobre padrões de malha e conectou 8 agentes em uma malha completa. Cada agente pode conversar com todos os outros agentes. Agora você tem 28 canais de comunicação possíveis, nenhum deles observável, e seu custo de token está nas alturas. A malha completa funciona com 3-4 agentes fortemente acoplados iterando em um artefato compartilhado. Além disso, decomponha-se em grupos menores.

A armadilha da estrutura

Você adotou uma estrutura multiagente antes de entender o padrão necessário. Agora seu código está bloqueado nas abstrações do framework e quando você precisa alterar a lógica de orquestração, você está lutando contra o framework em vez de resolver o problema. Aprenda os padrões primeiro. Adote uma estrutura somente quando a versão simples do Python se tornar realmente difícil de manter.

Quando começar de forma simples

Se você estiver construindo agentes de IA que precisam coordenar várias ferramentas e serviços, o padrão roteador/transferência leva você a 80% do caminho com 20% de complexidade. Plataformas como Nebula implementam esse padrão nativamente - agentes delegam para sub-agentes especializados, cada um com suas próprias ferramentas e contexto, com estado e autenticação tratados automaticamente em todas as execuções. Quer você mesmo construa ou use uma plataforma, o princípio é o mesmo: comece com a orquestração mais simples que resolva seu problema.

A progressão que funciona na prática:

Agente único com ferramentas agrupadas e resultados estruturados
Roteador + especialistas quando a contagem de ferramentas excede 12 ou os tipos de solicitação divergem
pipeline sequencial quando as tarefas têm uma ordem de estágio natural
Distribua quando subtarefas independentes precisarem de velocidade paralela
Hierárquico quando o problema abrange genuinamente vários domínios em escala

Ignore as etapas 3 a 5 até que a etapa 2 falhe comprovadamente. A maioria dos sistemas de agentes de produção nunca passa da etapa 2 – e isso é um sinal de boa engenharia, não uma limitação.

Isso faz parte da série Building Production AI Agents. Anterior: Por que seu fluxo de trabalho de IA é interrompido em grande escala. A seguir: pipelines de avaliação que capturam regressões do agente antes dos usuários.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
Orquestração multiagente: um guia para padrões que funcionam
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção