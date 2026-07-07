# Construindo ferramentas de nível de produção para agentes de IA: o que funciona após 100 implantações

> Artigo #24 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/building-production-grade-tools-for-ai-agents-what-works-after-100-deployments-20om)

---

A maioria dos desenvolvedores que constroem agentes de IA cometem o mesmo erro: passam semanas projetando a camada de orquestração, ajustando o prompt do sistema e escolhendo o modelo certo – depois entregam ao LLM uma pilha de endpoints de API empacotados às pressas e se perguntam por que ele falha na produção.

Aqui está a dura verdade das equipes que enviam agentes diariamente: o design da ferramenta tem um impacto maior na confiabilidade do agente do que a engenharia imediata. Uma ferramenta bem elaborada evita alucinaçõess no nível estrutural. Uma ferramenta mal elaborada garante isso.

Este artigo aborda o que aprendemos ao criar, implantar e depurar agentes de IA de produção em dezenas de fluxos de trabalhos do mundo real. Você obterá padrões concretos, exemplos de código funcionais e os antipadrões que mais nos custam em incidentes de produção.

O contrato entre código determinístico e não determinístico

Ao escrever uma função para outro desenvolvedor, você está trabalhando entre dois sistemas determinísticos. Mesma entrada, mesma saída. O código de chamada sabe exatamente o que esperar.

Uma ferramenta de IA é um contrato fundamentalmente diferente. Você está escrevendo uma interface entre código determinístico (seu serviço de back-end, banco de dados ou API) e um consumidor não determinístico (o LLM). O modelo pode:

Chame sua ferramenta quando você esperava que ela usasse outra coisa
Envie argumentos malformados porque a descrição era ambígua
Tente sua ferramenta novamente três vezes porque a mensagem de erro não informa por que ela falhou
Ignore totalmente sua ferramenta porque a descrição não explica quando usá-la

Isso significa que cada ferramenta precisa de cinco componentes com os quais as APIs tradicionais nunca se preocuparam: um nome preciso, uma descrição rica, um esquema de entrada rigoroso, tratamento estruturado de erros e um formato de saída previsível. Vamos construir cada um.

1. Nomenclatura: o primeiro sinal que o LLM avalia

O nome da ferramenta é a primeira coisa que o modelo verifica ao decidir qual ferramenta chamar. Funciona como um nome de classe em uma base de código – define expectativas antes de qualquer outro sinal.


```python
# Bad: vague, could mean anything
@mcp_tool(name="process")
def process(data):
...

# Bad: too generic, overlaps with other tools
@mcp_tool(name="get_data")
def get_data(query: str):
...

# Good: specific verb + noun, clear scope
@mcp_tool(name="list_overdue_invoices")
def list_overdue_invoices(customer_id: str):
...

# Good: resource_action pattern
@mcp_tool(name="invoice_send_reminder")
def invoice_send_reminder(invoice_id: str, channel: str):
...

```

Escolha uma convenção — verb_noun ou resource_action — e aplique-a em todas as ferramentas do seu servidor. A mistura de convenções força o LLM a aprender dois modelos mentais e, sob carga de trabalho, irá confundi-los. Vimos uma queda de 23% na seleção correta de ferramentas em um agente de produção quando a equipe tinha get_user, user_create e delete_file coexistindo sem nenhum padrão.

2. Descrições: Engenharia de Prompt Incorporada

A descrição de ferramentas é o campo mais subestimado no design de ferramentas. O LLM lê isso para decidir quando usar a ferramenta e o que ela receberá de volta. É uma engenharia imediata incorporada à própria definição da ferramenta.


```python
MISMATCHED_DESCRIPTION = "Searches the database"

GOOD_DESCRIPTION = """\
```
Pesquisa de texto completo na base de conhecimento da empresa.\
Use quando o usuário solicitar documentação interna, políticas ou especificações técnicas.\
Retorna até 10 resultados classificados por relevância, cada um com título, snippet e URL.\
NÃO pesquisa e-mails ou mensagens de bate-papo — use search_communications para eles."""


Observe o que a boa descrição faz: ela diz o que faz, informa ao LLM quando usá-la, descreve o formato de saída e declara explicitamente o que não fará. Essa última parte é crítica – limites negativos explícitos evitam que o LLM busque a ferramenta errada quando ela está próxima, mas não está certa.

Uma medida real de nossas implantações: melhorar apenas as descrições das ferramentas — sem alterações de código — reduziu o tempo de conclusão de tarefas em 40% e reduziu a seleção errada de ferramentas em 60%.

3. Esquemas de entrada: nunca confie no LLM

Os modelos alucinam valores de parâmetros, confundem tipos e inventam campos que não existem. Sua ferramenta deve validar cada entrada antes do processamento. As restrições do esquema JSON são sua primeira linha de defesa:


```python
GOOD_INPUT_SCHEMA = {
"type": "object",
"properties": {
"query": {
"type": "string",
"minLength": 1,
"maxLength": 500,
"description": "Natural language search query or exact document title"
},
"limit": {
"type": "integer",
"minimum": 1,
"maximum": 50,
"default": 10,
"description": "Maximum number of results to return"
},
"category": {
"type": "string",
"enum": ["engineering", "hr", "finance", "legal", "all"],
"default": "all",
"description": "Restrict search to a specific document category"
}
},
"required": ["query"],
"additionalProperties": False
}

```

Enums eliminam classes inteiras de falhas. Quando o ambiente aceita apenas "preparação" ou "produção", o LLM não pode inventar "prod-us-east" e travar seu script de implantação. Descobrimos que o uso de enums e padrões regex para parâmetros eliminou 80% dos erros de validação de tempo de execução na produção.

Parâmetros Poka Yoke

Dê um passo adiante com o design poka-yoke – tornando o uso indevido estruturalmente impossível:


# Em vez de aceitar caminhos de texto livre que causam travessia de caminho:
{"path": {"type": "string"}} # ruim

# Use enums com caminhos absolutos para configurações conhecidas:
{"configuração": {
```python
"type": "string",
"enum": ["/etc/prod/config.yaml", "/etc/staging/config.yaml"]
}}  # good
```

4. Tratamento de erros: erros são solicitações para o LLM

Quando uma ferramenta falha, o LLM precisa de informações suficientes para decidir se deve tentar novamente, tentar uma ferramenta diferente ou pedir ajuda ao usuário. Erros opacos como “Erro interno do servidor” deixam o modelo parado.

O MCP tem dois mecanismos de erro e combiná-los causa falhas silenciosas:

Erros em nível de protocolo (JSON-RPC): ferramenta desconhecida, argumentos malformados, servidor indisponível. A chamada nunca atingiu a lógica da sua ferramenta.
Erros de execução da ferramenta (isError: true): a ferramenta foi executada, mas falhou. O agente pode raciocinar sobre isso.
# Ruim: erro genérico, o LLM não consegue raciocinar sobre o que deu errado
return {"erro": "Algo deu errado"}

# Bom: erro estruturado com contexto acionável via isError
retornar {
```python
"isError": True,
"content": [{
"type": "text",
"text": json.dumps({
"error": "RATE_LIMIT_EXCEEDED",
"message": "Search API rate limit reached. Maximum 10 requests per minute.",
"retryAfterSeconds": 30,
"suggestion": "Wait 30 seconds before retrying, or narrow the query to reduce result processing time."
})
}]
}

```

Esse padrão — código legível por máquina, explicação legível por humanos, orientação de repetir e uma sugestão acionável — elimina uma grande classe de falhas de agente em que o modelo recebe um erro enigmático e alucina um caminho de recuperação.

5. Formato de saída: consistência é tudo

Formatos de saída imprevisíveis forçam o LLM a adivinhar, o que aumenta a chance de má interpretação e erros posteriores.


```python
# Bad: inconsistent output shape
def search(term):
results = db.query(term)
if results:
return results  # list of dicts
return "No results found"  # string — different type entirely!

# Good: consistent envelope, always the same shape
def search(term, limit=10):
results = db.query(term, limit=limit+1)
return {
"status": "success",
"resultCount": min(len(results), limit),
"results": [
{
"title": r.title,
"snippet": r.snippet[:200],
"url": r.url,
"relevanceScore": r.score
}
for r in results[:limit]
],
"hasMore": len(results) > limit
}

```

O agente sempre sabe que formato esperar. Não é necessário ramificar isinstance(result, str) vs isinstance(result, list). Essa previsibilidade aumenta em fluxos de trabalhos de várias etapas.

6. Eficiência do token: o custo oculto que mata o ROI

Cada resposta da ferramenta vai para a janela de contexto do LLM. Respostas detalhadas queimam tokens, aumentam custos e degradam a qualidade do raciocínio à medida que o contexto é preenchido.

Três estratégias que funcionam na produção:

Paginar agressivamente. Retorne 10 resultados com um cursor, não 1.000 registros. O agente pode enviar uma mensagem se precisar de mais.

Modos de resumo de suporte. Oferece parâmetros detalhados=True/False. Padrão como falso. Deixe o agente solicitar mais detalhes somente quando necessário.

Remova os metadados internos. O agente não precisa de IDs de banco de dados, carimbos de data/hora internos ou campos ORM. Retorne apenas o que o LLM precisa para entender e agir de acordo com o resultado.


# Registro interno do banco de dados (péssimo para o contexto do agente):
{
```python
"id": "a1b2c3d4-e5f6-7890",
"_created_at": "2026-04-15T08:23:11.442Z",
"_updated_at": "2026-04-30T14:07:33.101Z",
"_tenant_id": "org_48291",
"name": "John Smith",
"role": "Product Manager",
"email": "john@acme.com",
"status": "active",
"preferences": {"theme": "dark", "notifications": True, ...}
}
```

# Saída amigável ao agente:
{
```python
"name": "John Smith",
"role": "Product Manager",
"email": "john@acme.com",
"status": "active"
}

```

Medimos uma redução de 3,2x no consumo de tokens por tarefa apenas removendo os metadados internos dos resultados da ferramenta. Em escala, essa é a diferença entre um agente lucrativo e um centro de custo.

7. Anotações comportamentais: sinais sobre os quais o agente pode agir

A especificação MCP 2025-03-26 introduziu anotações de ferramentas — campos de metadados que ajudam agentes a tomar decisões mais inteligentes sobre a invocação de ferramentas:


```python
tool_annotations = {
"readOnlyHint": True,       # Safe to call without confirmation
"destructiveHint": False,   # Won't mutate state
"idempotentHint": True,     # Safe to retry with same args
"openWorldHint": False      # Only reads from known database
}

```

Essas anotações orientam o comportamento real dos clientes agentes. Uma ferramenta destructiveHint: true aciona uma porta de confirmação antes da execução. Uma ferramenta idempotentHint: true permite que o cliente retry com segurança em timeout. Mas lembre-se: as anotações são dicas, não proteções. O cliente agente decide se os honrará.

Antipadrões que vimos na produção
A ferramenta de Deus
```python
@mcp_tool(name="process_customer_request")
def process_customer_request(request_text: str):
# Parses intent, searches DB, sends email, updates CRM, creates ticket...
# This is 6 operations fused into one. When step 3 fails, the agent
# cannot retry steps 4-6 independently.

```

Mantenha as ferramentas atômicas. Uma ferramenta, um propósito. Se precisar fazer X e Y, devem ser duas ferramentas que o agente compõe. As ferramentas atômicas são mais fáceis de testar, mais fáceis para o LLM raciocinar e mais fáceis de compor em fluxos de trabalhos complexos.

Desvio de descrição da ferramenta

A descrição da sua ferramenta diz "retorna uma lista de usuários". Seis meses depois, após uma refatoração, ele retorna um objeto paginado com usuários e campos total_count. A descrição nunca foi atualizada. O agente quebra silenciosamente.

Trate as descrições das ferramentas como documentação viva. Ao executar avaliações (e deveria), inclua verificações de precisão da descrição em seu passe de validação.

Falha silenciosa na deglutição
```python
def get_metric(name):
try:
return metrics_api.get(name)
except Exception:
return {"data": []}  # agent thinks everything is fine

```

O agente recebeu o que parece ser uma resposta válida, mas vazia. Ele procede com suposições erradas. Sempre retorne a falha de forma visível — isError: true com contexto — para que o agente possa raciocinar sobre a recuperação.

Uma verdadeira ferramenta de produção, de ponta a ponta

Aqui está uma definição completa da ferramenta MCP que segue todos os princípios acima, a partir de um serviço de monitoramento de implantação de produção:


```python
@server.tool(
name="deploy_service",
description=(
"Deploy a service to the specified environment. "
"Use this for production and staging deployments. "
"For rollbacks, use rollback_service instead. "
"Returns the deployment ID, target version, and current status."
),
input_schema={
"type": "object",
"properties": {
"service": {
"type": "string",
"description": "Service name from the service registry. Use list_services to find available names."
},
"environment": {
"type": "string",
"enum": ["staging", "production"],
"description": "Target environment for the deployment."
},
"version": {
"type": "string",
"pattern": r"^v\d+\.\d+\.\d+$",
"description": "Semantic version to deploy, e.g., v2.4.1."
}
},
"required": ["service", "environment", "version"],
"additionalProperties": False
},
annotations={
"destructiveHint": True,
"idempotentHint": True,
"openWorldHint": False
}
)
async def deploy_service(service: str, environment: str, version: str):
try:
result = await deploy_api.deploy(service, environment, version)
return {
"status": "success",
"deployment_id": result.id,
"target_version": version,
"environment": environment,
"started_at": result.started_at.isoformat()
}
except DeploymentError as e:
return {
"isError": True,
"content": [{
"type": "text",
"text": json.dumps({
"error": "DEPLOYMENT_FAILED",
"message": str(e),
"service": service,
"environment": environment,
"version": version,
"suggestion": "Check the build status with check_build_status before retrying. If the build passed, verify the environment has capacity."
})
}]
}
except Exception as e:
return {
"isError": True,
"content": [{
"type": "text",
"text": json.dumps({
"error": "INTERNAL_ERROR",
"message": f"Unexpected error during deployment: {str(e)}",
"suggestion": "This is not a retryable error. Escalate to the infrastructure team."
})
}]
}

```

Cada princípio é representado: nome preciso, descrição rica com referência cruzada, esquema estrito com enumeração e validação de padrão, anotações comportamentais, saída de sucesso estruturada e saída de falha estruturada com sugestões acionáveis.

Ferramentas de teste com LLMs, não apenas testes unitários

Os testes de unidade verificam se sua ferramenta retorna os dados corretos. Eles não verificam se o LLM consegue descobrir qual ferramenta chamar, construir argumentos válidos ou se recuperar de erros.

O único teste real para uma ferramenta é: colocá-la na frente de um LLM e atribuir-lhe uma tarefa. Execute uma avaliação com 20 a 50 solicitações do mundo real e meça:

Precisão na seleção da ferramenta: O LLM escolheu a ferramenta certa?
Correção do argumento: enviou parâmetros válidos?
Recuperação de erros: quando a ferramenta falha, o LLM tenta novamente de forma produtiva ou tem alucinações?
Eficiência do token: Quantos tokens a resposta da ferramenta consome?

Automatize isso. Execute avaliações em cada PR que altere a definição de uma ferramenta. Se uma alteração na descrição da ferramenta reduzir a precisão da seleção de 95% para 80%, será uma regressão — mesmo que o código em si seja perfeito.

Quando NÃO construir uma ferramenta

Nem todo endpoint de API precisa ser uma ferramenta. Algumas operações são muito arriscadas (excluir dados de produção), muito caras (executar um trabalho de treinamento de modelo) ou muito complexas (fluxos de trabalhos de várias etapas que o agente não pode verificar). Em vez disso, implemente-os como primitivos workflow em sua camada de orquestração — código determinístico que o agente aciona, mas não chama diretamente.

A regra prática: se o pior resultado do LLM chamar essa ferramenta de errada for “o usuário vê uma mensagem estranha”, construa-a como uma ferramenta. Se for “alguém perde dinheiro” ou “o sistema quebra”, envolva-o primeiro em sua camada de orquestração com grades de proteção.

O TL;DR é simples: trate cada definição de ferramenta como se fosse o produto, porque para um agente de IA, é. O modelo lê descrições de ferramentas como código-fonte – cada palavra, cada restrição, cada exemplo é importante. Faça isso direito e seus agentes se tornarão dramaticamente mais confiáveis ​​sem tocar em uma única linha de engenharia imediata.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
Construindo ferramentas de nível de produção para agentes de IA: o que funciona após 100 implantações
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção