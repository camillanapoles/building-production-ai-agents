# Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória

> Artigo #2 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/your-ai-agent-has-amnesia-fix-it-with-these-4-memory-patterns-4n3g)

---

Você constrói um agente de IA que funciona perfeitamente em testes. Ele faz a triagem de sua caixa de entrada, resume threads do Slack e arquiva problemas do GitHub. Você o implanta de acordo com um cronograma e sai satisfeito.

Então você verifica novamente na manhã seguinte. O agente de triagem da caixa de entrada arquivou os mesmos e-mails duas vezes. O resumidor standup não se lembra do resumo de ontem. O agente de pesquisa redescobre a mesma informação que encontrou há 24 horas.

Seu agente está com amnésia.

Este é o modo de falha mais comum que vejo em agentes de IA de produção - nem prompts ruins, nem modelos errados, mas perda total de memória entre as execuções. Existem exatamente 4 padrões de memória que corrigem isso, e a maioria dos desenvolvedores usa o padrão errado ou pula totalmente a memória. Veja qual padrão usar e quando.

Por que os agentes esquecem (e por que isso é importante)

LLMs são apátridas por design. Cada chamada de API começa do zero. A janela de contexto é a única "memória" que um LLM possui e é redefinida completamente entre as chamadas.

Isso cria três problemas caros na produção:

Chamadas de API desperdiçadas. Seu agente busca novamente os dados já processados. Um agente de triagem de e-mail que não consegue lembrar quais e-mails já foram processados ​​irá reprocessar toda a sua caixa de entrada a cada execução.
Comportamento inconsistente. Sem memória de decisões passadas, seu agente faz escolhas contraditórias. Ele sinaliza o mesmo remetente como “alta prioridade” em uma execução e “baixa prioridade” na próxima.
Sem curva de aprendizado. Os humanos ficam mais rápidos em tarefas recorrentes porque se lembram de padrões. Um agente apátrida fica permanentemente preso no primeiro dia.

A solução não é única para todos. Você precisa de padrões de memória diferentes para problemas diferentes. Aqui estão os quatro que cobrem todos os casos de uso de produção que encontrei.

Padrão 1: buffer de conversa (memória de curto prazo)

O padrão mais simples. Armazene as últimas N mensagens em uma lista e passe-as para o LLM como contexto.


```python
class ConversationBuffer:
def __init__(self, max_tokens=4000):
self.messages = []
self.max_tokens = max_tokens

def add(self, role: str, content: str):
self.messages.append({"role": role, "content": content})
self._trim()

def _trim(self):
while self._token_count() > self.max_tokens and len(self.messages) > 1:
self.messages.pop(0)

def _token_count(self) -> int:
return sum(len(m["content"]) // 4 for m in self.messages)

def get_context(self) -> list:
return self.messages

```

Este é um buffer de anel com truncamento com reconhecimento de token. Quando o buffer excede seu orçamento de token, ele descarta primeiro as mensagens mais antigas.

Use-o quando: Você estiver criando um chatbot de sessão única ou um simples agente de perguntas e respostas onde a conversa começa e termina de uma só vez.

Não use-o quando: Seu agente executa um cronograma, lida com fluxos de trabalhos de várias etapas ou precisa se lembrar de algo entre as sessões. O buffer evapora quando a sessão termina.

O buffer de conversação é o padrão na maioria dos frameworks, e é por isso que a maioria dos agentes tem amnésia - os desenvolvedores assumem que isso é suficiente e nunca adicionam memória persistente.

Padrão 2: Memória de resumo (compactada de longo prazo)

Quando as conversas ficam longas, você não consegue guardar todas as mensagens. A memória de resumo usa o próprio LLM para compactar curvas mais antigas em um resumo contínuo, mantendo as mensagens recentes literalmente.


```python
def compress_memory(messages, summary_so_far, model="gpt-4o-mini"):
old_messages = messages[:-5]  # Keep last 5 verbatim
recent = messages[-5:]

new_summary = llm_call(
model=model,
prompt=f"""Previous summary: {summary_so_far}
```
Novas mensagens a serem incorporadas: {format_messages(old_messages)}
```python
Write an updated summary capturing all key facts,
decisions, and context. Be specific -- names, numbers,
and decisions matter more than pleasantries."""
)
return new_summary, recent


Every N turns (or when you hit a token threshold), summarize the oldest messages and keep only the compressed version plus recent context.
```

Use-o quando: Seu agente lidar com longas conversas de suporte, sessões de depuração estendidas ou qualquer interação que exceda regularmente a janela de contexto.

A compensação: os resumos apresentam perdas. Cada ciclo de compactação corre o risco de perder detalhes que parecem sem importância agora, mas que importam mais tarde. Um agente que cuida de um thread de suporte ao cliente de 2 horas pode resumir a mensagem de erro original do cliente e depois se esforçar para resolver o problema quando ele voltar 45 minutos depois.

Nota sobre custos: cada compactação custa uma chamada LLM, mas é barata. Uma chamada gpt-4o-mini para resumir 20 mensagens custa uma fração de centavo – muito mais barata do que expandir sua janela de contexto indefinidamente.

Padrão 3: Memória de Recuperação (Pesquisa Semântica)

Armazene todas as interações anteriores em um banco de dados vetorial. Antes da execução de cada agente, incorpore a tarefa atual e recupere as experiências anteriores mais relevantes usando a pesquisa semântica.


```python
import chromadb
from uuid import uuid4

class RetrievalMemory:
def __init__(self, collection_name="agent_memory"):
self.client = chromadb.PersistentClient(path="./memory_db")
self.collection = self.client.get_or_create_collection(
name=collection_name
)

def store(self, text: str, metadata: dict = None):
self.collection.add(
documents=[text],
ids=[f"mem_{uuid4().hex[:12]}"],
metadatas=[metadata or {}]
)

def recall(self, query: str, top_k: int = 5) -> list:
results = self.collection.query(
query_texts=[query],
n_results=top_k
)
return results["documents"][0]

```

Este é o único padrão de memória que se adapta a milhares de interações passadas sem explodir sua janela de contexto. Em vez de colocar tudo no prompt, você procura o que é relevante.

Use-o quando: Seu agente acumula centenas de interações anteriores e precisa se lembrar de interações específicas. Pesquise agentes, bots de suporte ao cliente com histórico de caso ou qualquer agente onde "lembre-se daquela vez que..." é importante.

A pegadinha crítica: a qualidade da recuperação depende inteiramente da qualidade da incorporação. incorporação ruins retornam memórias irrelevantes, e memórias irrelevantes são piores do que nenhuma memória - elas confundem ativamente o agente. Sempre teste sua recuperação pipeline com consultas reais antes de confiá-la na produção.

Uma verificação rápida de sanidade: incorpore 10 consultas reais que seu agente irá tratar, recupere os 5 principais resultados de cada uma e verifique manualmente a relevância. Se menos de 3 de 5 resultados forem úteis, seu modelo de incorporação ou estratégia de agrupamento precisa ser melhorado.

Padrão 4: Estado Estruturado (Memória de Valor-Chave)

Este é o padrão mais subestimado – e o que mais importa na produção. Em vez de armazenar conversas brutas, armazene explicitamente os fatos que seu agente aprendeu.


```python
import json
from datetime import datetime
from pathlib import Path

class StructuredMemory:
def __init__(self, db_path="agent_state.json"):
self.db_path = Path(db_path)
self.state = self._load()

def remember(self, key: str, value: str, category: str = "general"):
if category not in self.state:
self.state[category] = {}
self.state[category][key] = {
"value": value,
"updated_at": datetime.now().isoformat()
}
self._save()

def recall(self, key: str, category: str = "general"):
return self.state.get(category, {}).get(key, {}).get("value")

def recall_category(self, category: str) -> dict:
return {k: v["value"] for k, v in self.state.get(category, {}).items()}

def _load(self) -> dict:
if self.db_path.exists():
return json.loads(self.db_path.read_text())
return {}

def _save(self):
self.db_path.write_text(json.dumps(self.state, indent=2))

```

Isso armazena fatos explícitos e consultáveis: preferências do usuário, regras de roteamento, mapeamentos de entidades, pontos de verificaçãos operacionais.

Considere um agente de triagem de e-mail. É preciso lembrar que os e-mails de boss@company.com são sempre de alta prioridade, que "Project Atlas" é o codinome interno para a migração do banco de dados e que o último ID de e-mail processado foi msg_abc123. Estes são fatos estruturados, não conversas a serem recuperadas.

Use-o quando: Seu agente aprender as preferências do usuário, rastrear entidades recorrentes ou manter o estado operacional entre execuções. Qualquer agente em execução em um agendamento precisa desse padrão.

Por que a maioria dos guias ignora isso: Os artigos sobre arquitetura de memória adoram discutir bancos de dados vetoriaisse geração com recuperação aumentada. O estado estruturado não é atraente – é um arquivo JSON. Mas na produção, ele resolve 80% das necessidades de memória do agente com recuperação zero latência e precisão perfeita. Não há qualidade de incorporação com que se preocupar, nem pesquisa semântica para ajustar. Você armazena um fato e obtém o fato exato de volta.

A estrutura de decisão: qual padrão quando

Aqui está a folha de dicas:

O que você precisa Padrão Por que
Lembre-se desta conversa Buffer Mais barato, mais simples, com escopo de sessão
Lidar com conversas muito longas Resumo Comprimido, com perdas, mas prático
Lembre-se de eventos passados ​​específicos de centenas de execuções de escalas de recuperação por meio de pesquisa vetorial
Lembrar fatos e preferências aprendidos Estado Estruturado Explícito, confiável, sem ruído de recuperação
Agente autônomo de produção Estado Estruturado + Recuperação de Fatos para confiabilidade, recuperação para contexto

A maioria dos agentes de produção usa uma combinação. Aqui está a pilha que recomendo:

Fundação: Estado estruturado (Padrão 4) para tudo o que seu agente aprendeu – preferências, regras, mapeamentos de entidades, pontos de verificaçãos operacionais.
Camada de contexto: Memória de recuperação (Padrão 3) para relembrar interações passadas relevantes ao lidar com novas tarefas.
Camada de sessão: Buffer de conversa (Padrão 1) para a execução atual.
Opcional: Memória de resumo (Padrão 2) somente se sessões individuais excederem regularmente sua janela de contexto.

Comece apenas com o estado estruturado. É o padrão de maior impacto e menor complexidade. Adicione a recuperação somente quando seu agente tiver acumulado histórico suficiente para que os fatos estruturados por si só não consigam capturar o contexto relevante.

Como isso se parece na prática

Um agente de triagem de e-mail de produção combinando todos os quatro padrões:

O estado estruturado armazena as prioridades do remetente (boss@company.com → high), codinomes do projeto (Atlas → migração do banco de dados) e o carimbo de data/hora do último e-mail processado.
A memória de recuperação armazena resumos de e-mails anteriores threads, portanto, quando um novo e-mail chega de um remetente conhecido, o agente tem contexto sobre conversas anteriores.
O buffer de conversa rastreia a sessão de triagem atual – quais emails foram processados ​​e quais ações foram tomadas.
O agente inicia cada execução carregando o estado estruturado, consulta a memória de recuperação conforme necessário para emails específicos e usa o buffer dentro da sessão.

Plataformas como Nebula lidam com isso nativamente - o estado estruturado persiste durante as execuções agendadas, agentes compartilham contexto por meio de delegação multi-agente e as chaves de memória sobrevivem entre sessões sem configuração de banco de dados externo. Mas os padrões funcionam com qualquer estrutura. A arquitetura é mais importante do que as ferramentas.

O resultado final

A diferença entre um agente de demonstração e um agente de produção é a memória. A maioria dos agentes falha não porque não sejam inteligentes o suficiente, mas porque não conseguem se lembrar do que aprenderam ontem.

Comece com o Padrão 4 (estado estruturado). Ele resolve 80% das necessidades de memória de produção com um arquivo JSON e complexidade zero. Adicione o Padrão 3 (recuperação) quando seu agente acumula histórico suficiente para que os fatos estruturados por si só não capturem o quadro completo.

Não deixe seu agente começar todos os dias do zero. Os padrões são simples. O impacto é imediato.

Qual foi a coisa mais estranha que seu agente esqueceu e que causou um incidente de produção? Deixe nos comentários - vou compartilhar o meu.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção