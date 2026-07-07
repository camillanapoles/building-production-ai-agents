# Resultados estruturados versus chamada de ferramenta: quando seu agente realmente precisa de qual

> Artigo #17 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/structured-outputs-vs-tool-calling-when-your-agent-actually-needs- Which-kgk)

---

Se você está construindo um agente de IA, provavelmente já se perguntou o seguinte: devo usar saídas estruturadas ou chamada de ferramenta para obter dados do modelo?

A resposta é mais importante do que a maioria dos guias sugere. A escolha errada significa análise frágil, tokens desperdiçados em etapas desnecessárias ou um agente que alucina com segurança um esquema JSON que nunca seguiu.

Executei ambos os padrões em produção em dezenas de fluxos de trabalhos de agentes. Esta é a estrutura de decisão que uso.

Os dois padrões
Resultados Estruturados

Você diz ao modelo: “Responda exatamente neste formato”. Você fornece um esquema (Pydantic, JSON Schema ou um prompt do sistema descrevendo campos) e o LLM produz uma única resposta que corresponde.


```python
from openai import OpenAI
from pydantic import BaseModel

class WeatherResponse(BaseModel):
city: str
temperature_c: float
condition: str
wind_kmh: float
summary: str

client = OpenAI()
response = client.beta.chat.completions.parse(
model="gpt-4o-2026-04-20",
messages=[{"role": "user", "content": "What's the weather in Tokyo?"}],
response_format=WeatherResponse,
)
result: WeatherResponse = response.choices[0].message.parsed
print(result.temperature_c)

```

O modelo retorna um objeto JSON que satisfaz o esquema. Sem etapas intermediárias, sem decisões.

Chamada de ferramenta (chamada de função)

Você registra funções que podem ser chamadas com o modelo. O modelo decide qual ferramenta chamar, com quais argumentos e retorna uma solicitação de chamada de ferramenta em vez de uma resposta de texto.


```python
from openai import OpenAI

client = OpenAI()
response = client.chat.completions.create(
model="gpt-4o-2026-04-20",
messages=[{"role": "user", "content": "What's the weather in Tokyo?"}],
tools=[{
"type": "function",
"function": {
"name": "get_weather",
"description": "Get current weather for a city. Returns temperature, condition, and wind speed.",
"parameters": {
"type": "object",
"properties": {
"city": {"type": "string", "description": "City name"},
"units": {"type": "string", "enum": ["metric", "imperial"]},
},
"required": ["city"],
},
},
}],
)

tool_call = response.choices[0].message.tool_calls[0]
print(tool_call.function.arguments)
# '{"city": "Tokyo", "units": "metric"}'

```

O modelo retorna uma decisão de chamar uma função. Você o executa, retorna os resultados e o modelo produz uma resposta final.

A diferença central

Os resultados estruturados dizem: "Forneça-me dados neste formato, agora."

A chamada de ferramenta diz: "Aqui estão as ações que você pode realizar. Decida quais usar."

Essa distinção orienta todas as decisões arquitetônicas que se seguem.

Quando usar saída estruturada
1. Extração em etapa única

Sua entrada é uma bolha de texto. Sua saída é uma extração estruturada. Não são necessárias ações intermediárias.


```python
class IssueExtraction(BaseModel):
severity: str  # LOW, MEDIUM, HIGH, CRITICAL
affected_service: str
root_cause_summary: str
suggested_fix: str
estimated_hours: int

response = client.beta.chat.completions.parse(
model="gpt-4o-2026-04-20",
messages=[
{"role": "system", "content": "Extract structured incident data from the following report."},
{"role": "user", "content": open('incident-report.txt').read()},
],
response_format=IssueExtraction,
)

```

O modelo lê o relatório e produz um IssueExtraction estruturado. Uma chamada de API. Nenhuma execução de função. Use isto quando a transformação for somente leitura — o modelo analisa e extrai, mas não atua.

Sinal: entrada → extração → concluído.

2. Pipelines com muita validação

O response_format da OpenAI com saídas estruturadas faz a aplicação do esquema no nível da API. Se o modelo produzir JSON inválido, a API tentará novamente internamente. Você obtém um objeto Pydantic válido ou um erro - saída nunca malformada que você precisa analisar defensivamente.

Os blocos tool_use da Anthropic oferecem aplicação semelhante, os beta.tools da Anthropic com modelos de resposta impõem o esquema. Mas para extração pura, as APIs de saída estruturadas têm uma integração mais estreita.


```python
# This always returns a valid IssueExtraction or raises an API error
# You don't need try/except around json.loads()
extraction = response.choices[0].message.parsed
assert isinstance(extraction, IssueExtraction)  # Always true
```

3. Extração paralela em múltiplas entradas

Você deseja extrair o mesmo esquema de 500 documentos. A saída estruturada significa que você envia 500 chamadas de API independentes, sem compartilhamento de estado entre elas.

A chamada de ferramenta exigiria que você rastreasse qual ferramenta foi chamada para qual documento, retornasse os resultados à conversa e gerenciasse uma janela de contexto crescente. Para extração em lote, isso é uma complexidade desnecessária.

4. Fluxos de trabalho sensíveis aos custos

A saída estruturada é uma chamada de API. A chamada de ferramenta é no mínimo duas (o modelo decide chamar a ferramenta, você executa, o modelo recebe o resultado e gera a resposta final). Para pipelines de extração de altothroughput, a diferença de chamada de 2x é importante.

Quando usar a chamada de ferramenta
1. Fluxos de trabalho de várias etapas

O usuário pergunta: “Encontre o erro mais caro no Sentry esta semana, verifique qual implantação o desencadeou e crie um problema no GitHub”.

Uma saída estruturada não pode fazer isso. O modelo precisa:

```python
Query Sentry (tool call)
Read the result (context in next message)
Query deployment logs (tool call)
Cross-reference the data
Create a GitHub issue (tool call)
Summarize the finding (final text response)
```

Este é o loop do agente: decidir → ligar → observar → decidir novamente.


```python
tools = [
{"type": "function", "function": {
"name": "query_sentry",
"description": "Query Sentry for issues. Returns issue details including count and stack trace.",
"parameters": {"type": "object", "properties": {
"filter": {"type": "string"},
"sort_by": {"type": "string", "enum": ["count", "date"]},
"limit": {"type": "integer"},
}},
}},
{"type": "function", "function": {
"name": "query_deployments",
"description": "Query recent deployments. Returns deployment hashes and timestamps.",
"parameters": {"type": "object", "properties": {
"environment": {"type": "string"},
"since": {"type": "string"},
}},
}},
{"type": "function", "function": {
"name": "create_github_issue",
"description": "Create a GitHub issue with title, body, and labels.",
"parameters": {"type": "object", "properties": {
"title": {"type": "string"},
"body": {"type": "string"},
"labels": {"type": "array", "items": {"type": "string"}},
}},
}},
]

# The agent loop
messages = [{"role": "user", "content": "Find the most expensive error in Sentry..."}]

for _ in range(5):  # Max iterations to prevent infinite loops
response = client.chat.completions.create(
model="gpt-4o-2026-04-20",
messages=messages,
tools=tools,
)

if response.choices[0].message.tool_calls:
tool_call = response.choices[0].message.tool_calls[0]
result = execute_tool(tool_call)
messages.append(response.choices[0].message)
messages.append({"role": "tool", "content": result, "tool_call_id": tool_call.id})
else:
# Final text response
print(response.choices[0].message.content)
break

```

O modelo orquestra três sistemas diferentes. Os resultados estruturados podem expressar o resultado final, mas não levam você até lá.

2. Execução Condicional

O agente precisa decidir se deve chamar uma ferramenta. Os resultados estruturados sempre produzem resultados. A chamada de ferramenta permite que o modelo diga “Não preciso de nenhuma ferramenta para isso”.


# O usuário pergunta: "Quanto é 2 + 2?"
# O modelo NÃO deve chamar query_sentry para isso.
# Com chamada de ferramenta, o modelo responde diretamente com texto.
# Com saída estruturada, o modelo é forçado a preencher o esquema — mesmo que a pergunta não justifique isso.


Isso é muito importante na produção. Os usuários perguntam coisas que não correspondem às suas ferramentas, e um agente que sempre tem alucinações sobre chamadas de ferramentas (porque o esquema força isso) é pior do que aquele que às vezes diz "Não posso ajudar com isso".

3. Chamadas de ferramentas paralelas

Os modelos modernos suportam a chamada de várias ferramentas em uma única resposta. Se o modelo precisar verificar o clima em três cidades simultaneamente:


# Retornos do modelo:
{
```python
"tool_calls": [
{"function": {"name": "get_weather", "arguments": '{"city": "Tokyo"}'}, "id": "call_1"},
{"function": {"name": "get_weather", "arguments": '{"city": "London"}'}, "id": "call_2"},
{"function": {"name": "get_weather", "arguments": '{"city": "NYC"}'}, "id": "call_3"},
]
}
```

# Você executa todos os três em parallel e depois retorna os resultados


Este é o assassino de desempenho para sistemas de agentes. Em vez de cinco chamadas de API sequenciais, você recebe uma chamada que aciona três pesquisas parallel. A saída estruturada não pode expressar "chamar esta função três vezes com argumentos diferentes".

4. Exposição do servidor MCP

Se você estiver construindo um servidor MCP, chamada de ferramenta é o protocolo. MCP é, em sua essência, uma padronização de chamada de função sobre uma camada de transporte (stdio ou Streamable HTTP). A especificação MCP define ferramentas, recursos e prompts – ferramentas sendo as primitivas de chamada de função.

Quando seu agente se conecta a um servidor MCP, ele está usando chamada de ferramenta nos bastidores. O servidor MCP anuncia seus recursos, o agente escolhe quais ferramentas invocar e envia solicitações JSON-RPC.


```python
# MCP client side — this is structured tool calling over a transport
from mcp import ClientSession

async with client_session(server) as session:
tools = await session.list_tools()  # Discover available tools
result = await session.call_tool("get_weather", {"city": "Tokyo"})

```

Se o seu agente precisar se conectar a sistemas externos via MCP, chamada de ferramenta não é opcional — é a interface.

Matriz de Decisão
Cenário Melhor Padrão Porquê
Extrair entidades do texto Saída estruturada Chamada única, imposta por esquema
Respondendo a perguntas que precisam de dados externos Chamada de ferramenta O modelo decide qual ferramenta chamar
Raciocínio de várias etapas (marque A e depois decida B) Chamada de ferramenta Loop de agente permite encadeamento condicional
Extração em lote de 1.000 documentos Saída estruturada Independente, paralelaizável, mais barata
Conecte-se a sistemas via ferramenta MCP chamando protocolo MCP = chamada de função
Valide e retorne um único formato de resposta Aplicação de esquema em nível de API de saída estruturada
Lidar com solicitações ambíguas normalmente Chamada de ferramenta O modelo pode pular ferramentas e responder diretamente
Chamadas de API paralelas em uma rodada Chamada de ferramenta Várias chamadas de ferramenta por resposta
O padrão híbrido que funciona melhor na prática

A maioria dos agentes reais fluxos de trabalhos usa ambos. O padrão é assim:

Ferramenta de chamada para a camada de orquestração — o modelo decide quais sistemas consultar e em que ordem.
Saída estruturada para o resultado final – os dados analisados ​​e validados nos quais você persiste ou age.
```python
# Layer 1: Tool calling to gather data
tools = [query_database, search_docs, check_status]
# ... agent loop ...

# Layer 2: Structured output for the final synthesis
analysis = client.beta.chat.completions.parse(
model="gpt-4o-2026-04-20",
messages=[system_msg] + conversation_history,
response_format=IncidentReport,
)

# analysis is a validated IncidentReport object, not raw text
save_to_database(analysis.model_dump())

```

Isso proporciona orquestração flexível (o modelo escolhe as ferramentas certas na ordem certa) e saída confiável (a resposta final é validada pelo esquema, pronta para processamento downstream).

Antipadrões comuns
Chamada de ferramenta para extração simples

Construindo um pipeline de cinco ferramentas quando você realmente precisa apenas extrair um objeto JSON do texto. Cada chamada de ferramenta extra adiciona latência, custo e uma oportunidade para o modelo escolher a ferramenta errada. Se a tarefa for somente leitura, use a saída estruturada.

Saída estruturada para comportamento do agente

Forçar um modelo em um esquema fixo quando a tarefa requer decisões. "Extrair um ticket" onde um campo é next_action: ToolChoice - e então você executa com base nisso - é uma solução alternativa fragile. A chamada de ferramenta adequada dá ao modelo a agência para tomar essas decisões de forma nativa.

Faltando o protetor de iteração máxima

Cada loop de chamada de ferramenta deve ter um hard cap. Os modelos podem entrar em loops de reflexão onde chamam a mesma ferramenta três vezes com argumentos ligeiramente diferentes, convencidos de que o primeiro resultado não estava correto.


para iteração no intervalo(5): # SEMPRE impõe um máximo
```python
response = call_model(messages, tools)
if not response.tool_calls:
break
messages += execute_and_append(response.tool_calls)
else:
messages.append({"role": "system", "content": "Max iterations reached. Summarize what you found so far."})

No Fallback on Tool Failure
```

Quando uma chamada de ferramenta gera uma exceção, devolva o erro ao modelo como uma resposta da ferramenta. O modelo pode tentar novamente, escolher uma ferramenta diferente ou degradar normalmente:


mensagens.append({
```python
"role": "tool",
"content": f"Error: Database connection timeout. Retry or try a different query.",
"tool_call_id": call_id,
})

```

Falhas silenciosas de ferramentas significam a paralisação do loop do agente. Mensagens de erro explícitas significam que o agente se adapta.

Onde plataformas como a Nebulosa se encaixam

A questão de saída estruturada versus chamada de ferramenta fica significativamente mais difícil quando seu agente precisa de padrões, gerenciamento de estado, execução de ferramenta paralela e proteções. Executar o loop completo do agente — com iterações máximas, tratamento de erros, aplicação do orçamento de ferramentas e análise estruturada de resultados — não é uma infraestrutura trivial.

Plataformas como Nebula abstraem isso: você define as ferramentas do agente (que ficam expostas ao MCP), define restrições (chamadas máximas, orçamentos) e a plataforma gerencia o loop de orquestração, análise de saída estruturada e persistência de estado entre chamadas de ferramenta.

A arquitetura fica assim:


Mensagem do usuário
│
▼
┌─────────────────────────┐
│ Orquestrador (Nebulosa) │
│ - Proteção máxima de iteração │
│ - Acompanhamento de orçamento de ferramenta │
│ - Compressão de contexto │
│ - Análise estruturada │
└─────────────────────────┘
│
```python
├──→ Tool: database query (MCP)
├──→ Tool: web search (MCP)
├──→ Tool: GitHub API (MCP)
│
▼
┌─────────────────────────┐
│  Structured Output Layer │
│  - Validate final result  │
│  - Persist to store       │
│  - Deliver to user        │
└─────────────────────────┘

```

Se você estiver criando um único agente com algumas ferramentas, a API chat.completions bruta funciona bem. Se seus agentes abrangem vários servidores MCP, precisam de aplicação de orçamento de ferramentas ou exigem validação de saída estruturada em escala, uma plataforma que lida com o loop de orquestração vale a pena a sobrecarga.

Conclusões

Saída estruturada para extração, chamada de ferramenta para ação. Se o modelo precisar ler e retornar dados — use saída estruturada. Se precisar decidir, agir, observar e decidir novamente — use chamada de ferramenta.

Híbrido é o padrão de produção. Ferramenta de orquestração, saída estruturada para o resultado final validado. Não force ninguém a fazer os dois trabalhos.

Sempre limite as iterações. Um loop de chamada de ferramenta sem contagem máxima de iterações é um incidente de produção esperando para acontecer.

Alimente os erros de volta ao modelo. Chamadas de ferramenta com falha devem retornar mensagens de erro estruturadas, e não exceções silenciosas. O modelo pode tentar novamente ou se adaptar se souber o que deu errado.

MCP = chamada de ferramenta padronizada. Se o seu agente se conectar a sistemas externos, o MCP é o protocolo para expor suas funções. Cada servidor MCP é um registro de chamada de ferramenta.

A escolha entre saídas estruturadas e chamada de ferramenta não é apenas uma preferência de API – é uma decisão de design sobre quanta agência seu agente realmente precisa.

Este artigo faz parte da série Building Production AI Agents no Dev.to, cobrindo os desafios reais de engenharia da execução de agentes de IA autônomos — da arquitetura do servidor MCP às proteções do agente.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
Resultados estruturados versus chamada de ferramenta: quando seu agente realmente precisa de qual
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção