Nº 5 ferramentas de código aberto para testar agentes de IA antes que interrompam a produção

> Artigo #23 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/5-open-source-tools-for-testing-ai-agents-before-they-break-production-2g1m)

---

Seu agente de IA passa em todos os testes unitários. O prompt parece correto. Você implanta. Em seguida, um usuário relata que o agente de suporte começou a recomendar políticas de reembolso em vez de etapas de solução de problemas. Sem acidente. Nenhum rastreamento de pilha. Apenas silenciosamente errado.

Esta é a classe de bug mais difícil em sistemas agentes: regressões silenciosas. Você muda uma coisa e o comportamento do agente muda de uma forma que os testes tradicionais não conseguem detectar. O agente retorna 200, chama algumas ferramentas, produz uma saída — mas não a saída correta para a nova configuração.

A avaliação do agente não é mais opcional. Em 2026, com os ecossistemas de ferramentas MCP abrangendo mais de 177.000 APIs e a orquestração multiagente se tornando padrão, a lacuna entre "trabalho em minha máquina" e "trabalho em produção" nunca foi tão grande.

Aqui está uma comparação prática de cinco ferramentas que resolvem esse problema – desde avaliadores locais leves até plataformas LLMOps completas.

DR
Ferramenta melhor para local primeiro?	CI/CD pronto?	Preço
Detecção de regressão de linha de base EvalView Golden Sim Sim (GitHub Actions) Grátis, Apache 2.0
agentevals Pontuação baseada em OpenTelemetry sem novas execuções Sim Sim Grátis, Apache 2.0
Avaliações YAML do primeiro terminal do AgentV, qualquer agente CLI Sim Sim Grátis, MIT
Simulações de agente LangWatch + plataforma LLMOps completa Não (núcleo de código aberto) Sim Nível gratuito / Empresarial
Gerenciamento de prompt da equipe Agenta + interface de avaliação Sim (auto-hospedeiro) Sim Gratuito, código aberto
O que torna o teste de agente diferente

Os testes tradicionais são determinísticos: dada a entrada X, espera-se a saída Y. Os agentes quebram este contrato porque:

Não determinismo: o mesmo prompt produz resultados diferentes nas execuções
A trajetória da ferramenta é importante: duas saídas podem parecer semelhantes, mas uma chamada helm_list_releases e a outra alucinou um comando
Efeitos da janela de contexto: uma troca de modelo de GPT-4o para Claude pode alterar quais ferramentas o agente descobre
Facilidade de prompt: Adicionar "seja mais amigável" a um prompt do sistema pode interromper silenciosamente chamadas de ferramentas críticas

Um bom teste de agentes precisa validar trajetórias, não apenas respostas. Cada ferramenta abaixo aborda isso de maneira diferente.

1. EvalView — pytest para agentes de IA

Estrelas: Crescendo rapidamente · Licença: Apache 2.0

O EvalView adota a abordagem mais simples: captura um instantâneo do comportamento do seu agente e, em seguida, compara cada execução subsequente com ele. Pense nisso como git diff para o comportamento do agente.


pip instalar avaliação de avaliação
evalview init # Detecta agente, cria conjunto inicial
instantâneo do evalview # Salva o comportamento atual como linha de base
evalview check # Captura regressões após cada alteração


As quatro camadas de pontuação são onde tudo fica interessante:

Chamadas de ferramentas + sequência (gratuita) — O agente chamou as ferramentas certas na ordem certa?
Verificações baseadas em código (gratuitas) — Regex, validação de esquema JSON nas saídas
Similaridade semântica (~$0,00004/teste) — Comparação de resultados baseada em incorporação
LLM como juiz (~$0,01/teste) — pontuação GPT, Claude ou Gemini com critérios personalizados

O que diferencia o EvalView são as linhas de base com múltiplas referências. agentes não determinísticos podem ter até 5 variantes de resposta válidas, e o EvalView verifica todas elas em vez de forçar você a escolher uma resposta "de ouro".

Pontos fortes: Funciona sem chaves de API (totalmente offline com Ollama). A comparação da linha de base dourada é única – nenhuma outra ferramenta faz instantâneos automáticos antes/depois do comportamento do agente. O teste do contrato MCP detecta desvios na interface.

Pontos fracos: requer que você mantenha instantâneos de linha de base. Se o comportamento esperado do seu agente mudar legitimamente, você precisará fazer um novo instantâneo manualmente.

Ideal para: equipes que desejam "pytest for agents" — detecção de regressão rápida, local e baseada em linha de base.

2. agentevals – Pontuação de agentes a partir de rastreamentos, sem reexecuções

Estrelas: 115+ · Licença: Apache 2.0 · Idioma: Python

agentevals resolve um problema diferente: "Já tenho rastreamentos de produção. Posso avaliá-los sem reproduzir chamadas LLM caras?"

A resposta é sim. Ele lê rastreamentos OpenTelemetry (de LangChain, Google ADK, OpenAI Agents SDK ou qualquer estrutura instrumentada por OTel) e os pontua em relação aos conjuntos de avaliação que você define.


agentevals executa samples/helm.json \
--eval-set amostras/eval_set_helm.json \
-m tool_trajectory_avg_score

[PASS] tool_trajectory_avg_score 1 PASSADO 1 0ms


O principal insight é a separação entre registro e avaliação. Você registra uma vez (de tráfego de produção, execuções de teste ou sessões ao vivo) e, em seguida, avalia quantas vezes quiser com diferentes métricas — sem chamadas de API adicionais, sem custos de token.

O modo de código zero é particularmente limpo:


# Terminal 1: Inicie o receptor
agentevals serve --dev

# Terminal 2: indique-o ao seu agente
exportar OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
python seu_agente.py


Os rastreamentos são transmitidos para a UI da web integrada em localhost:8001 em tempo real.

Pontos fortes: Sem custo de reexecução. OTel nativo significa que funciona com qualquer estrutura que suporte OpenTelemetry. CLI + UI da Web + servidor MCP. Gráfico Helm para implantação do Kubernetes.

Fraquezas: Você precisa primeiro da instrumentação OTel instalada. Se a estrutura do seu agente não emitir rastros, você precisará adicioná-lo. O formato do conjunto de avaliações é YAML e requer algum esforço de configuração.

Melhor para: Equipes que já usam OpenTelemetry e desejam avaliar rastreamentos de produção sem queimar tokens em replays.

3. AgentV – Avaliações YAML do primeiro terminal

Estrelas: Nicho, mas em crescimento · Licença: MIT · Idioma: TypeScript

AgentV adota a abordagem mínima: arquivos de teste YAML, executados no terminal, resultam em JSONL.


# avaliações/math.yaml
descrição: Resolução de problemas matemáticos
testes:
- id: adição
entrada: Quanto é 15 + 27?
saída_esperada: "42"
afirmações:
- tipo: contém
valor: "42"

avaliação do agentev evals/math.yaml


Tudo reside no Git – arquivos de avaliação, solicitações de juízes e resultados. O sistema de classificação híbrido combina verificações de código determinístico com classificadores LLM personalizáveis:


afirmações:
- tipo: contém
valor: "efervescente"
- tipo: classificador de código
comando: ./validators/check_syntax.py
- digite: llm-nivelador
prompt: ./graders/correctness.md


O comando agentv compare é subestimado – ele compara resultados entre vários alvos (diferentes modelos, diferentes prompts, diferentes versões de agente) para que você possa ver exatamente onde o comportamento mudou.

Pontos fortes: Sem servidor, sem inscrição, sem dependência de nuvem. Funciona em segundos. Saída XML JUnit para CI pipelines. Funciona com qualquer agente CLI – Claude Code, Codex, Copilot, modelos locais.

Pontos fracos: Comunidade menor que LangSmith ou Promptfoo. O formato YAML é simples, mas pode se tornar detalhado para conversas complexas com vários turnos.

Melhor para: desenvolvedores individuais e equipes pequenas que desejam avaliações no Git, não em um painel.

4. LangWatch – Simulações de Agente + LLMOps

Núcleo de código aberto · Plataforma em nuvem disponível

LangWatch se posiciona como a alternativa LangSmith que não requer o ecossistema LangChain. O diferencial é o teste de simulação de agente – em vez de avaliar pares de entrada/saída individuais, ele testa agentes multivoltas fluxos de trabalhos de ponta a ponta.

O mecanismo de simulação funciona em cenários complexos:

Conversas múltiplas com uso de ferramentas
Fluxos de entrada multimodais (texto + imagem + código)
Interações multiagentes em que um agente passa para outro

Isso detecta uma classe de bugs que a avaliação baseada em rastreamento deixa passar: os turnos individuais podem parecer bons, mas o fluxo entre os turnos tem um bug. Pense nisso como teste de integração versus teste de unidade para agentes.

Pontos fortes: Framework-agnóstico (OpenAI, Anthropic, CrewAI, Pydantic AI, customizado). As simulações de agentes detectam bugs no nível do fluxo. Integração DSPy para otimização automatizada de prompts. Certificação ISO 27001/SOC 2.

Pontos fracos: O núcleo de código aberto é apenas auto-hospedado; a plataforma em nuvem é onde reside todo o conjunto de recursos. Pegada operacional mais pesada do que as ferramentas locais.

Ideal para: equipes que executam fluxos de trabalhos complexos de vários turnos que precisam de validação em nível de simulação antes da implantação.

5. Agenta – Gerenciamento de Prompt de Equipe + Avaliação

Código aberto · Licença: Apache 2.0

A Agenta resolve o problema organizacional: “Meus engenheiros mudaram um prompt, mas ninguém avisou a equipe de produto, e agora o agente avalia os tickets dos clientes de forma diferente”.

É uma plataforma LLMOps construída em torno da colaboração em equipe:

Gerenciamento imediato com edição de UI para especialistas de domínio, paridade total de API para desenvolvedores
Playground lado a lado para comparar prompts e modelos com histórico de versões
Avaliação automatizada com LLM como juiz, avaliadores integrados ou código personalizado
Anotação de rastreamento com feedback da equipe — transforme rastreamentos de produção em testes com um clique

O fluxo de trabalho multifuncional é o valor real aqui. Os gerentes de produto podem editar prompts por meio da IU, os engenheiros criam versões deles por meio do Git e os avaliadores executam automaticamente em ambas as versões para sinalizar mudanças comportamentais.

Pontos fortes: Melhor colaboração em equipe da categoria. As comparações do Playground facilitam a visualização do impacto imediato antes do envio. Observabilidade total do rastreamento. Auto-hospedagem baseada em Docker.

Pontos fracos: Mais difícil de implantar do que as ferramentas CLI. Requer um back-end de banco de dados (PostgreSQL). Mais sobrecarga de configuração para desenvolvedores individuais.

Melhor para: Equipes onde mudanças imediatas envolvem partes interessadas técnicas e não técnicas.

Arquitetura: Construindo um Pipeline de Avaliação

Veja como essas ferramentas se encaixam em um pipeline de CI/CD real:


Fase de Desenvolvimento:
Prompt de alterações do desenvolvedor → agentv eval evals/regression.yaml
│
├── PASSAR → Mesclar PR
└── FALHA → Revisar diferença, corrigir prompt

Fase de preparação:
Implantar no teste → agentevals executam production_traces/
│ --eval-set staging_eval_set.json
└── Pontuações abaixo do limite → Bloquear implantação

Fase de produção:
agente é executado → EvalView captura tráfego ao vivo
│ → verificação de avaliação (de hora em hora)
├── REGRESSÃO detectada → Alerta de folga + reversão
└── Tudo limpo → Continuar monitoramento


A principal conclusão é que nenhuma ferramenta cobre todas as três fases. As simulações LangWatch detectam casos extremos antes que qualquer coisa seja enviada. AgentV bloqueia fusões de relações públicas. EvalView monitora a produção. agentevals avalia o tráfego de teste gratuitamente (reutilizando rastreamentos registrados).

É aqui que plataformas como o Nebula se tornam interessantes – em vez de unir cinco ferramentas de desenvolvimento, preparação e produção, o ciclo de vida da avaliação é integrado à própria plataforma do agente. Os agentes em execução no Nebula herdam rastreamento, avaliação e monitoramento como parte do tempo de execução, portanto, o pipeline "captura → avaliação → alerta" não requer infraestrutura separada.

Mas se você estiver construindo sua própria pilha, as cinco ferramentas acima estão prontas para produção e são gratuitas.

Escolhendo a ferramenta certa

Aqui está a estrutura de decisão que eu usaria:

“Só quero saber se meu agente ainda funciona após uma mudança” → EvalView. Linhas de base douradas, um comando para verificar.
“Tenho rastros de produção e quero avaliar sem custos” → agentevals. Leia rastreamentos OTel existentes, pontue em conjuntos de avaliação, custo zero de nova execução.
“Quero avaliações no meu repositório, ao lado do meu código” → AgentV. Arquivos YAML, execução de terminal, nativos do Git.
"Meus agentes têm fluxos complexos de múltiplas voltas" → LangWatch. Simule os fluxos antes que eles atinjam os usuários.
"Várias equipes tocam nas instruções do meu agente" → Agenta. UI para partes interessadas não técnicas, paridade de API para engenheiros.
O resultado final

A avaliação do agente não é mais algo “bom de se ter”. Quando seu agente controla fluxos de trabalhos de reembolso, implantações de código ou respostas voltadas ao cliente, o custo de "parecia bom localmente" aumentou drasticamente.

A boa notícia: as ferramentas de avaliação de código aberto em 2026 estão maduras o suficiente para que você não tenha desculpa. EvalView para detecção de regressão, agentevals para pontuação baseada em rastreamento, AgentV para avaliações de terminal, LangWatch para testes de simulação, Agenta para workflows de equipe — escolha o que corresponde ao seu workflow e comece a controlar as implantações de seu agente.

Os agentes que enviam de forma confiável são aqueles que são testados antes de serem enviados. Não depois do primeiro e-mail irritado do cliente.

O antipadrão do agente de Deus: por que sua IA quebra com 20 ferramentas
Seu agente de IA está com amnésia: conserte-o com estes 4 padrões de memória
...
Mais 22 peças...
5 ferramentas de código aberto para testar agentes de IA antes que interrompam a produção
O modelo de segurança de 5 camadas que todo agente de IA precisa na produção
Construindo servidores MCP personalizados: um guia do desenvolvedor para ferramentas de agente de IA de nível de produção