# Engenharia de contexto para agentes de IA: um guia prático

> Artigo #5 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/context-engineering-for-ai-agents-a-practical-guide-4lp6)

---

Seu agente de IA funciona perfeitamente em demonstrações. Então você o implanta na produção, entrega um verdadeiro fluxo de trabalho com 30 turnos de conversa, 15 definições de ferramentas e uma pilha de documentos recuperados - e ele começa a ter alucinações, ignorando instruções e escolhendo as ferramentas erradas.

O modelo não ficou mais burro. Sua engenharia de contexto falhou.

Engenharia de contexto é a prática de curar tudo que um LLM vê antes de responder – não apenas o prompt, mas as instruções do sistema, definições de ferramentas, histórico de conversas, documentos recuperados e resultados de etapas anteriores. Enquanto a engenharia imediata se concentra na elaboração da instrução correta, a engenharia de contexto decide quais informações entram na janela de contexto e o que fica de fora.

Isso é importante porque em sistemas de agentes de produção, a janela de contexto é um espaço privilegiado. Cada token compete por atenção, e a combinação errada de informações degrada o desempenho mais rapidamente do que um modelo mais fraco faria. Este guia fornece quatro estratégias concretas para projetar o contexto do seu agente – cada uma com código Python executável que você pode usar hoje.

O que é engenharia de contexto (e por que os prompts não são suficientes)

A engenharia imediata está escrevendo a pergunta em um exame. A engenharia de contexto é escolher quais materiais de referência você leva para a sala de exame.

Em um chatbot simples, a engenharia imediata costuma ser suficiente. O usuário envia uma mensagem, você a agrupa em um prompt do sistema e o modelo responde. O contexto é pequeno e previsível.

Os sistemas de agentes quebram completamente esse modelo. O contexto típico de um agente de produção inclui:

Prompt do sistema: 500-1.500 tokens definindo comportamento e restrições
Definições de ferramentas: 200-500 tokens por ferramenta (10 ferramentas = 2.000-5.000 tokens)
Histórico de conversas: cresce linearmente a cada turno
Documentos recuperados: 500-2.000 tokens por bloco
Resultados da etapa anterior: respostas de API variáveis, geralmente grandes

No momento em que o usuário fala, 60-80% da janela de contexto já pode estar consumida. O modelo tem menos espaço para raciocinar, instruções importantes ficam enterradas no meio, onde a atenção é mais fraca, e o desempenho se degrada silenciosamente.

A correção não é uma janela de contexto maior. GPT-4o oferece suporte a tokens de 128 mil, mas pesquisas mostram consistentemente que o desempenho do modelo diminui muito antes do limite técnico. A solução é projetar o que acontece.

O orçamento de contexto: saiba o que você está gastando

Antes de otimizar qualquer coisa, você precisa medir. Uma calculadora de orçamento de contexto informa exatamente quanto espaço cada componente consome e quanto resta para o modelo funcionar.


```python
import tiktoken

def calculate_context_budget(
model_limit: int,
system_prompt: str,
tools: list[dict],
history: list[dict],
retrieved_docs: list[str],
encoding_name: str = \"o200k_base\",
) -> dict:
\"\"\"Calculate context budget breakdown and remaining capacity.\"\"\"
enc = tiktoken.get_encoding(encoding_name)

def count(text: str) -> int:
return len(enc.encode(text))

system_tokens = count(system_prompt)
tool_tokens = sum(count(str(t)) for t in tools)
history_tokens = sum(count(str(m)) for m in history)
doc_tokens = sum(count(d) for d in retrieved_docs)

total_used = system_tokens + tool_tokens + history_tokens + doc_tokens
remaining = model_limit - total_used
utilization = (total_used / model_limit) * 100

budget = {
\"model_limit\": model_limit,
\"system_prompt\": system_tokens,
\"tools\": tool_tokens,
\"history\": history_tokens,
\"retrieved_docs\": doc_tokens,
\"total_used\": total_used,
\"remaining\": remaining,
\"utilization_pct\": round(utilization, 1),
}

if utilization > 60:
budget[\"warning\"] = (
f\"Context is {utilization:.0f}% full before user input. \"
\"Apply context engineering strategies below.\"
)

return budget


# Example: a typical agent with 10 tools and some history
budget = calculate_context_budget(
model_limit=128_000,
system_prompt=\"You are a helpful research assistant...\",  # ~50 tokens
tools=[{\"name\": f\"tool_{i}\", \"description\": \"...\", \"parameters\": {}} for i in range(10)],
history=[{\"role\": \"user\", \"content\": \"Analyze this dataset...\"} for _ in range(20)],
retrieved_docs=[\"Document chunk...\" * 50 for _ in range(5)],
)

for key, value in budget.items():
print(f\"{key}: {value}\")

```

A regra geral: se o seu contexto estiver mais de 60% cheio antes da mensagem atual do usuário, você terá um problema de engenharia de contexto. Execute esta calculadora em seu agente e você provavelmente ficará surpreso com a quantidade de espaço que as ferramentas e o histórico consomem.

4 Estratégias de Engenharia de Contexto para Agentes de Produção

Depois de saber para onde vai seu orçamento, aplique essas estratégias para recuperá-lo.

Estratégia 1: Janela Deslizante com Sumarização

A estratégia mais simples e de maior impacto. Mantenha as últimas N conversas literalmente (o modelo precisa de contexto recente para coerência) e comprima tudo o que é mais antigo em um resumo.


```python
from openai import OpenAI

client = OpenAI()

def manage_conversation_context(
messages: list[dict],
system_prompt: str,
max_recent_turns: int = 6,
summary_model: str = \"gpt-4o-mini\",
) -> list[dict]:
\"\"\"Keep recent turns verbatim, summarize older ones.\"\"\"
if len(messages) \u003C= max_recent_turns:
return [{\"role\": \"system\", \"content\": system_prompt}] + messages

old_messages = messages[:-max_recent_turns]
recent_messages = messages[-max_recent_turns:]

# Summarize old messages with a cheap, fast model
summary_response = client.chat.completions.create(
model=summary_model,
messages=[
{
\"role\": \"system\",
\"content\": (
\"Summarize this conversation in under 200 tokens. \"
\"Preserve: key decisions made, data retrieved, \"
\"user preferences stated, and any unresolved tasks.\"
),
},
*old_messages,
],
max_tokens=200,
)

summary = summary_response.choices[0].message.content

return [
{\"role\": \"system\", \"content\": system_prompt},
{\"role\": \"system\", \"content\": f\"Previous conversation summary: {summary}\"},
*recent_messages,
]


# Before: 40 messages = ~20,000 tokens of history
# After: 1 summary (~200 tokens) + 6 recent messages (~3,000 tokens)
# Savings: ~17,000 tokens per agent call

```

Quando usar: agentes de bate-papo de longa duração, fluxo de trabalhos de várias etapas, qualquer agente que acumule mais de 10 turnos de conversa.

Compensação: você perde detalhes granulares nos primeiros turnos. Mitigue isso ajustando o prompt de resumo para preservar o que seu agente mais precisa: decisões, pontos de dados ou preferências do usuário.

Nota de custo: a chamada de resumo usa gpt-4o-mini com tokens de entrada de US$ 0,15/1 milhão. Resumir 15.000 tokens de histórico antigo custa aproximadamente US$ 0,002 e economiza aproximadamente US$ 0,04 em tokens de entrada reduzidos na chamada principal (com preço GPT-4o). Ele se paga 20 vezes mais.

Estratégia 2: Pontuação de Relevância para Contexto Recuperado

Os sistemas RAG geralmente despejam cada pedaço recuperado no contexto. Isso é um desperdício. Nem todos os documentos recuperados são igualmente relevantes para a etapa atual do agente.

Em vez de injetar todos os resultados recuperados, classifique-os por relevância e inclua apenas aqueles acima de um limite:

Incorporação de similaridade: classifique os pedaços por similaridade de cosseno com a consulta atual
Ponderação de atualidade: aumente os documentos atualizados recentemente se a atualização for importante
```python
Source priority: rank authoritative sources (official docs) above informal ones (forum posts)
Threshold filtering: drop anything below a relevance score of 0.7 (tune this empirically)
```

Para documentos longos que ultrapassam o limite, extraia apenas os parágrafos mais relevantes em vez de incluir o texto completo. Um documento de 2.000 tokens geralmente possui 200 tokens de conteúdo realmente útil para a etapa atual.

Quando usar: Qualquer agente com RAG, pesquisas na base de conhecimento ou recuperação de documentos. Quanto mais documentos você recupera, mais isso importa.

O principal insight: a recuperação e a injeção de contexto são decisões separadas. Seu retriever encontra candidatos. Seu engenheiro de contexto decide quais candidatos ganham um lugar na janela.

Estratégia 3: Injeção Dinâmica de Ferramentas

Se o seu agente tiver mais de 15 ferramentas, apenas suas definições poderão consumir de 5.000 a 10.000 tokens. Pior ainda, uma grande lista de ferramentas aumenta a chance de o modelo escolher a ferramenta errada – a atenção fica diluída em muitas opções.

A solução: não carregue todas as ferramentas em cada chamada. Use um classificador leve para prever quais ferramentas são relevantes para a etapa atual e injete apenas essas.


```python
from openai import OpenAI
import json

client = OpenAI()

def select_tools_for_step(
task_description: str,
all_tools: list[dict],
max_tools: int = 5,
classifier_model: str = \"gpt-4o-mini\",
) -> list[dict]:
\"\"\"Select relevant tools for this step using a cheap classifier.\"\"\"
tool_names = [t[\"function\"][\"name\"] for t in all_tools]

response = client.chat.completions.create(
model=classifier_model,
messages=[
{
\"role\": \"system\",
\"content\": (
\"You are a tool router. Given a task, return a JSON array \"
\"of the most relevant tool names from the available list. \"
f\"Return at most {max_tools} tools. Return ONLY the JSON array.\"
),
},
{
\"role\": \"user\",
\"content\": (
f\"Task: {task_description}\
\
\"
f\"Available tools: {json.dumps(tool_names)}\"
),
},
],
max_tokens=100,
)

try:
selected_names = json.loads(response.choices[0].message.content)
except json.JSONDecodeError:
return all_tools[:max_tools]  # Fallback: return first N tools

selected = [t for t in all_tools if t[\"function\"][\"name\"] in selected_names]
return selected if selected else all_tools[:max_tools]


# Example: agent has 20 tools, but this step only needs 3
all_tools = [
{\"type\": \"function\", \"function\": {\"name\": \"search_web\", \"description\": \"Search the web\", \"parameters\": {}}},
{\"type\": \"function\", \"function\": {\"name\": \"read_file\", \"description\": \"Read a file\", \"parameters\": {}}},
{\"type\": \"function\", \"function\": {\"name\": \"send_email\", \"description\": \"Send an email\", \"parameters\": {}}},
{\"type\": \"function\", \"function\": {\"name\": \"query_database\", \"description\": \"Query SQL database\", \"parameters\": {}}},
# ... 16 more tools
]

relevant_tools = select_tools_for_step(
task_description=\"Find the latest sales figures from the database\",
all_tools=all_tools,
max_tools=3,
)
# Returns: [query_database, read_file] -- saves ~4,000 tokens

```

Quando usar: Agentes com mais de 10 ferramentas, especialmente aqueles que usam MCP, onde os usuários podem conectar conjuntos de ferramentas arbitrários. Se você leu nosso artigo sobre sobrecarga de ferramenta MCP, esta é a correção em nível de código.

Custo do classificador: a chamada de roteamento usa aproximadamente 200 tokens de entrada no gpt-4o-mini – aproximadamente US$ 0,00003. Ele economiza de 3.000 a 8.000 tokens na chamada do agente principal, o que com o preço GPT-4o economiza entre US$ 0,007 e US$ 0,020. Um retorno do investimento de mais de 200x por chamada.

Estratégia 4: Compressão do resultado da etapa

agentes de várias etapas passam resultados entre as etapas. Uma etapa de pesquisa retorna 10 resultados com metadados completos. Uma consulta ao banco de dados retorna 500 linhas. Uma chamada de API retorna um blob JSON aninhado com 50 campos.

A próxima etapa raramente precisa de tudo isso. Compacte os resultados intermediários apenas nos campos exigidos pela etapa posterior:

Resultados da pesquisa: compactação de metadados completos até título + URL + resumo de 1 linha por resultado
Respostas da API: extraia apenas os campos referenciados no prompt da próxima etapa
Resultados do banco de dados: agregado ou amostra em vez de passar linhas brutas
Rascunhos da Web: remova a navegação, os anúncios e os clichês - mantenha apenas o corpo do conteúdo

O princípio: cada etapa deve receber o contexto mínimo viável para realizar seu trabalho. Se uma etapa precisar apenas de 5 campos de uma resposta de API de 50 campos, os outros 45 campos estarão desperdiçando tokens e diluindo a atenção.

Quando usar: Pipeline agentes, workflows sequencial de várias etapas, qualquer agente em que a saída de uma etapa alimenta outra.

Engenharia de contexto vs memória vs RAG: quando usar o quê

Esses três conceitos são complementares e não concorrentes. Veja como eles se relacionam:

\tEngenharia de Contexto\tMemória\tRAG
O que ele faz\tSeleciona o que entra na janela de contexto desta chamada\tPersiste informações entre sessões\tRecupera de bases de conhecimento externas
Escopo\tChamada LLM única\tSessão cruzada, longo prazo\tFontes de dados externas
A pergunta responde\t\"O que o modelo deve ver agora?\"\t\"O que o agente deve lembrar?\"\t\"Qual conhecimento externo é relevante?\"
Quando ele é executado\tCada etapa do agente\tNos limites de salvar/carregar\tNas consultas de recuperação
Exemplo\tDescartando 15 mensagens antigas, mantendo 5 recentes\tArmazenando a preferência do usuário: \"prefere exemplos de Python\"\tBuscando documentos de produtos que correspondam à pergunta do usuário

RAG recupera informações do candidato. A memória persiste fatos importantes. A engenharia de contexto seleciona o que realmente aparece na janela de todas as fontes disponíveis – incluindo resultados RAG e memória.

Um agente de produção precisa de todos os três. RAG sem engenharia de contexto coloca pedaços irrelevantes na janela. A memória sem engenharia de contexto carrega todos os fatos salvos, independentemente da relevância. A engenharia de contexto sem memória ou RAG não tem nada para selecionar.

Se você estiver construindo memória em seus agentes, nosso artigo anterior sobre padrões de memória de agente cobre o lado de armazenamento e recuperação. Este artigo aborda o lado da seleção – o que acontece depois da recuperação, logo antes da chamada do LLM.

Juntando tudo

As quatro estratégias se sobrepõem naturalmente:

Orçamento primeiro. Execute a calculadora de contexto no seu agente. Conheça sua linha de base.
Comprimir histórico. O resumo da janela deslizante é a solução de maior impacto e menor esforço. Comece aqui.
Recuperação de filtro. Pontue e limite seus resultados RAG em vez de descartar tudo.
Injete ferramentas dinamicamente. Se você tiver mais de 10 ferramentas, direcione para subconjuntos por etapa.
Compactar resultados da etapa. Extraia apenas o que a próxima etapa precisa.

As métricas a serem observadas:

% de utilização do contexto: quão cheia está a janela antes de o usuário falar? Meta: menos de 50%.
Tokens por etapa: as chamadas do seu agente estão ficando mais caras com o tempo? Se sim, o histórico ou os resultados estão inchados.
Precisão da tarefa em fluxos de trabalhos de várias etapas: uma melhor engenharia de contexto melhora diretamente a precisão em tarefas complexas em que o modelo precisa rastrear metas em muitas etapas.

Plataformas como Nebula lidam com a engenharia de contexto automaticamente no nível da arquitetura – quando um agente pai delega a um subagente, cada subagente recebe apenas o contexto relevante para sua tarefa específica, com memória compartilhada lidando com a continuidade entre agentes. Mas quer você use uma plataforma ou construa do zero, os princípios são os mesmos: medir seu orçamento, compactar o que é antigo, filtrar o que é recuperado e injetar apenas o que é necessário.

Principais conclusões

Os melhores agentes de IA não são aqueles com os modelos mais poderosos ou as maiores janelas de contexto. São eles que colocam as informações certas na frente do modelo no momento certo.

Comece com a calculadora de orçamento de contexto. Se o seu agente estiver com mais de 60% de utilização antes do usuário falar, aplique as estratégias em ordem: resumir o histórico, filtrar a recuperação, encaminhar ferramentas dinamicamente, compactar os resultados da etapa. Cada um compõe.

A engenharia de contexto é a peça que faltava entre “meu agente trabalha em demonstrações” e “meu agente trabalha em produção”. O modelo já sabe como raciocinar. Seu trabalho é dar exatamente o que ele precisa para raciocinar bem.

Isso faz parte da série Building Production AI Agents. Anterior: Como interromper as espirais de custos dos agentes de IA antes que elas comecem. Consulte também: Sobrecarga de ferramentas MCP: por que mais ferramentas tornam seu agente pior.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
Engenharia de contexto para agentes de IA: um guia prático
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção