# Arquitetura de Sistemas de IA

Como desenhar **aplicações que usam IA** (LLMs, modelos de ML) em produção — não como o modelo funciona por dentro (`AI.md`) nem como o modelo é atacado (`AI_SECURITY.md`), mas como orquestrar inferência, dados, ferramentas, memória, guardrails e humanos em um sistema que **serve usuários de forma útil, segura, econômica e confiável**.

Construir sistemas de IA exige revisitar muito do que `SOFTWARE_ARCHITECTURE.md` ensina. Três trade-offs em vez de dois; não-determinismo em vez de lógica pura; custos variáveis; modelo como dependência mutável; falha silenciosa por alucinação em vez de stack trace.

## Por Que É Diferente

Software tradicional tem trade-offs canônicos: **correção × performance × custo de desenvolvimento**. Comportamento é determinístico (mesmo input → mesmo output), falhas são exceções visíveis, custo em produção é CPU/memória/storage bem modelado.

Sistemas de IA adicionam:

1. **Não-determinismo**: `temperature > 0` e amostragem fazem mesma entrada produzir saídas diferentes. Mesmo `temperature = 0` não garante determinismo entre versões/providers.
2. **Falha silenciosa**: alucinação não dá stack trace. Sistema "parece funcionar" enquanto responde errado.
3. **Custos variáveis por request** (tokens × preço × modelo). "Um usuário pesado custa 100× um leve" sem cap.
4. **Latência variável e alta** (~100 ms a 30 s). Afeta UX radicalmente.
5. **Modelo evolui**: versões novas mudam comportamento sem warning. Sua app quebra silenciosamente.
6. **Input é linguagem natural**: validação clássica não cobre.
7. **Instrução + dado compartilham canal**: prompt injection é novo (ver `AI_SECURITY.md`).
8. **Avaliação é probabilística**: "correto em 87% dos casos" — não binário.

## Trade-off Central: Custo × Latência × Qualidade

Triângulo inescapável em aplicações LLM:

```
              Qualidade
              /      \
             /        \
            /          \
        Custo ———— Latência
```

- **Qualidade alta**: modelo maior (GPT-5, Claude Opus) → caro + lento.
- **Latência baixa**: modelo menor + streaming + cache → menos qualidade + mais trabalho de engenharia.
- **Custo baixo**: modelo pequeno + batch + caching agressivo → qualidade decrescente + latência inconsistente.

Arquitetura madura **varia o ponto no triângulo por contexto**: roteador leve para query simples, modelo top para complexa, cache para repetidas.

## Arquiteturas de RAG

**Retrieval-Augmented Generation**: LLM + base de conhecimento externa. Dominante em aplicações com dados proprietários.

### Naive RAG

```
query → embed → vector search → top-k chunks → concat em prompt → LLM → resposta
```

Funciona para demos; falha em produção em queries multi-hop, ambíguas, ou com necessidade de filtro estrutural.

### Hybrid Retrieval

Combina lexical (BM25) + semântico (embeddings) + fusão (RRF). Ver `SEARCH_AND_RANKING.md`. Vence naive consistentemente.

### Re-ranking

Retrieval traz top-100; cross-encoder (BGE Reranker, Cohere Rerank) reordena top-5. Cross-encoder é ~10× mais caro que bi-encoder mas aplica-se só no top-N. Melhora precisão dramaticamente.

### Query Rewrite / Expansion

- **Decomposition**: "Qual a política X para Y em 2025?" → queries sub-atomizadas.
- **HyDE** (Hypothetical Document Embeddings): LLM gera resposta hipotética, embed disso busca documentos similares. Funciona em domínios específicos.
- **Multi-query**: gerar N reformulações, unir resultados.
- **Step-back**: generalizar pergunta antes de recuperar.

### Agentic RAG

Agente LLM decide **quando** e **o que** recuperar, múltiplas vezes se preciso, reformulando.

- Útil em queries complexas onde uma recuperação não basta.
- Custo alto (múltiplos LLM calls). Latência idem.

### Hierarchical RAG

- Documentos em árvore (summaries → chunks → sub-chunks).
- Navega top-down.
- Útil em corpus grande com estrutura.
- Implementações: LlamaIndex Tree, RAPTOR.

### Contextual Retrieval (Anthropic 2024)

Prepend contexto de documento a cada chunk antes de embedar. "Este chunk do documento X sobre Y..." + chunk. Reduz ambiguidade de chunks soltos; recall melhora ~35-50%.

### Self-querying

LLM extrai filtros estruturados da query natural ("após 2023" → filter `date > 2023-01-01`), combina com busca semântica. Base de assistentes sobre dados estruturados.

### GraphRAG (Microsoft 2024)

Constrói grafo de conhecimento do corpus; queries navegam por entidades/relações. Melhor em síntese global ("resumo do corpus inteiro") mas caro no ingestion.

### Chunking — decisão crítica

- **Fixed size** (200-500 tokens): baseline.
- **Recursive**: tenta quebrar em pontos naturais (parágrafo, sentença, palavra).
- **Semantic chunking**: agrupa por similaridade.
- **Document-aware**: respeita estrutura (Markdown headers, código, tabelas).
- **Late chunking** (2024): embedar documento inteiro em chunks depois — preserva contexto global.

Trade-off: chunk grande dilui; pequeno perde contexto. **Overlap** ~15-20% mitiga.

## Arquiteturas de Agentes

Agente = LLM + ferramentas + loop de decisão + (às vezes) memória. Escalada de autonomia.

### ReAct (Reasoning + Acting)

Loop clássico (Yao et al., 2022):

```
Thought: o que sei, o que preciso fazer?
Action: usar ferramenta X com argumento Y
Observation: resultado de X
Thought: dado isso, o que agora?
... até Final Answer.
```

Simples, poderoso, largamente copiado. Problema: se uma ação dá errado, recuperação é frágil.

### Plan-and-Execute

1. LLM **planeja** sequência inteira primeiro.
2. Executa passos sem re-planejamento.

Barato (menos LLM calls). Falha quando plano encontra obstáculo inesperado.

### Plan-and-Execute + Replan

Executa com capacidade de re-planejar quando observa falha. Equilíbrio prático.

### Reflexion

Após falhar, agente **reflete** sobre o erro e adiciona lição à memória para próxima tentativa. Paper Shinn et al. 2023. Útil quando se pode iterar.

### Tree of Thoughts (ToT)

Agente ramifica exploração de múltiplas linhas de raciocínio + avalia + poda. Ganha em problemas como Game of 24, Sudoku. Caro.

### Graph of Thoughts

Generalização de ToT para grafo arbitrário — permite fusão de ramos.

### Multi-Agent

Múltiplos LLMs especializados colaboram:

- **Router / Orchestrator** despacha tarefa a agente adequado.
- **Sub-agentes especializados** (pesquisa, código, planejamento, escrita).
- **Handoff explícito**: agente A transfere contexto e controle a B.
- **Debate**: agentes argumentam até convergir.
- **Swarm** (OpenAI 2024): lightweight handoff-based.

Vantagens: modularidade, especialização, testabilidade.
Desvantagens: latência (serializado), custo, propagação de erro.

### Anthropic's "Building Effective Agents" (2024)

Papers influentes sintetizam padrões:

- **Workflow**: caminho pré-definido em código (maior parte dos casos resolve).
- **Agent**: LLM escolhe caminho dinamicamente (reserve para casos onde é necessário).
- Padrões workflow: **chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer**.

Regra: **comece com workflow, evolua para agent só se necessário**.

### MCP — Model Context Protocol

Padrão Anthropic (2024) para expor ferramentas e recursos a LLMs. Servidor MCP publica capabilities; cliente LLM invoca. Desacopla tool implementation do agente.

Equivalente funcional: function calling OpenAI, tool use Anthropic — MCP como protocolo aberto.

## Tool Use e Function Calling

Primitiva arquitetural mais importante depois do próprio LLM.

### Anatomia

1. **Declaração**: tool com nome, descrição, schema de argumentos (JSON Schema).
2. **LLM decide invocar**: retorna intent + argumentos estruturados.
3. **Runtime executa** ferramenta.
4. **Resultado volta ao LLM** para continuação.

### Padrões

- **Read-only tools** (search, get, list): seguras; pouca análise antes de executar.
- **Mutating tools** (send, delete, transfer): exigem confirmação, logs, rollback.
- **Expensive tools** (rodar query, treinar modelo): rate limit, budget.
- **Tool families**: grupos de tools correlatas (DB cursor + query + commit).

### Desafios

- **Tool selection error**: LLM chama ferramenta errada.
- **Argument hallucination**: inventa IDs, parâmetros.
- **Loop infinito**: agente chama ferramenta repetidamente sem progresso.
- **Descrições ambíguas**: tool descriptions são **prompts para o modelo**. Escrita clara importa.

### Mitigações

- **Schema validation** antes de executar.
- **Max iterations** no loop.
- **Budget** (tokens + $ + calls).
- **Dry-run mode** para agentes em desenvolvimento.
- **Human-in-the-loop** em tools destrutivas.
- **Tool description tests**: aplicar agent em casos de uso; avaliar seleção correta.

## Orquestração Multi-Modelo

Nenhuma aplicação séria usa um único modelo para tudo.

### Roteamento

Classificar query → modelo apropriado:

- **Query classification**: modelo pequeno classifica intent.
- **Heuristic**: comprimento, formato, palavras-chave.
- **Embedding-based**: comparar com exemplos.
- **Direto**: deixar primeiro modelo grande decidir se precisa escalar.

Exemplo: 80% das queries → modelo pequeno barato; 20% complexas → modelo grande.

### Cascata

Tentar do mais barato ao mais caro. Parar quando resposta satisfatória (confidence threshold, eval passa, fallback trigger).

```
GPT-4o-mini → evaluator → se não OK → GPT-4o → evaluator → se não OK → GPT-5
```

Reduz custo médio significativamente em workload onde queries simples dominam.

### Ensemble

Múltiplos modelos respondem; combinar:

- **Majority vote** em classificação.
- **Self-consistency**: amostrar N respostas do mesmo modelo, maioria.
- **Hierarchical**: modelo grande revisa resposta do pequeno.

Custa N× mas melhora qualidade. Raro em prod por custo.

### Specialization

- **Reasoning** em modelo otimizado (o3, DeepSeek-R1): caro, lento, mas profundo.
- **Fast** em modelo pequeno (GPT-4o-mini, Haiku, Flash).
- **Local** (Llama, Qwen, Mistral) para privacidade ou baixo custo.
- **Embedding** dedicado (text-embedding-3, BGE).
- **Re-ranker** dedicado.
- **Code** dedicado (Codestral, Qwen-Coder).
- **Vision / audio / multimodal** por modalidade.

## Serving Patterns

Infraestrutura que serve inferência.

### Continuous Batching

Servidor junta requisições em batch dinâmico; à medida que um gera tokens, outros podem iniciar. Sem esperar batch inteiro completar. **vLLM, TGI, TensorRT-LLM** implementam. Throughput 5-20× vs static batching.

### PagedAttention

vLLM: gerencia KV-cache em "pages" (analogia memória virtual), permite compartilhamento entre requests com prefix comum (system prompt). Reduz memória significativamente.

### Prefix Caching

Requisições com mesmo system prompt / documentos comuns reutilizam KV-cache já computado. OpenAI, Anthropic implementam transparentemente (50-90% desconto em tokens cacheados).

### Speculative Decoding

Modelo pequeno propõe N tokens; grande valida em paralelo → se concorda, aceita todos. Acelera 2-3× em casos típicos. Implementações em vLLM, TGI.

### KV-cache

Cache interno do modelo por sequência. Em chat multi-turn, preserva contexto sem recompute. Custa memória (GB por sessão longa). Estratégias de eviction (LRU, session close).

### Streaming

Retorne tokens conforme gerados (SSE, WebSocket). Latência percebida cai drasticamente (time-to-first-token é a métrica).

### Quantização

INT8, INT4, FP8 reduzem tamanho 2-4× e aceleram. Ver `GPU.md`. Perda de qualidade típica 1-3%.

## Memória

LLMs são stateless por default. Arquitetura define memória.

### Context Window (Short-term)

Tudo cabe em tokens enviados. Limites crescentes (128K-2M em 2024-2025) mas:

- **Custo escala com tokens**.
- **Atenção degrada**: "lost in the middle" — informação no meio é ignorada relativo às pontas.
- **Latência sobe**.

### Sumarização Rolante

Conversation history longo: sumarizar mensagens antigas em "memória compactada"; manter últimas N mensagens verbatim + sumário.

### Vector Store (Long-term)

Embeddings de memórias; recuperar relevantes por similaridade. LangChain "Conversation Memory with Vector Store".

### Episodic Memory

Estruturar memórias como episódios (timestamp, atores, localização, resultado). Permite queries tipo "o que aconteceu quando fulano falou X".

### Semantic Memory

Extração de fatos permanentes. "User prefere café sem açúcar" persiste entre sessões.

### Working Memory

Estado da tarefa atual (variáveis, intermediate results). Equivalente a scratchpad.

### Hierarchical Memory

Combina múltiplos níveis: working (tarefa) → episodic (sessão) → semantic (usuário) → world (domínio).

Frameworks: **LangGraph, Mem0, Letta (antes MemGPT), Zep**.

## Guardrails

Componente arquitetural, não afterthought.

### Input

- **Classifiers**: detectar prompt injection, abuse, PII, off-topic.
- **Rate limit** por usuário/tenant/IP.
- **Schema** quando possível.
- **PII redaction** antes de enviar a LLM.

### Output

- **Structured output** (JSON Schema constrained decoding): elimina classe de erros.
- **Content policy** classifier: rejeita tóxico/ilegal.
- **Factuality checks** contra sources conhecidos.
- **Format validation**.

### Runtime (agentes)

- **Action allowlist**: só tools explicitamente aprovadas.
- **Budget cap**: max tokens, max calls, max $.
- **Blast radius limiting**: sandbox, permissions mínimas.
- **Human confirmation** em ações críticas.

Ferramentas: **NeMo Guardrails, Guardrails AI, Llama Guard, Prompt Shields (Azure), Shield (OpenAI)**.

## Fallback e Graceful Degradation

### Cascata defensiva

```
GPT-5 (primary)
  ↓ fail (timeout, rate limit, error)
GPT-4 (fallback)
  ↓ fail
Rule-based heuristic
  ↓ fail
Error message "contact support"
```

Padrão para resiliência. Latência cap por nível.

### Provider Diversity

Redundância multi-provider (OpenAI + Anthropic + Google) protege contra outage de um. Complexidade operacional real. LLM gateways (Portkey, OpenRouter, LiteLLM) simplificam.

### Partial Response

Quando não consegue resposta ideal, degradar explicitamente: "não tenho certeza sobre X, mas Y..."

## Human-in-the-Loop (HITL)

Padrões:

- **Approve**: humano OK antes de ação (ex.: enviar e-mail).
- **Review**: humano vê e edita resultado antes de publicar.
- **Override**: humano pode interceptar/corrigir em runtime.
- **Feedback loop**: humano rotula output; dados alimentam fine-tuning.
- **Escalation**: alta confiança → autônomo; baixa → humano.

Calibração de **confidence** determina qual padrão ativar.

## Arquiteturas Híbridas

LLMs não resolvem tudo. Sistemas robustos combinam:

### LLM + classical ML

- Classical ML para features numéricas, séries temporais, dados tabulares (GBM, regressão).
- LLM para texto livre, síntese, raciocínio.
- **Exemplo**: detector de fraude = GBM (features transacionais) + LLM (analisar descrição livre).

### LLM + regras (symbolic)

- Regras determinísticas para conformidade, compliance, pricing.
- LLM para interpretação, interface com usuário.
- **Neurosymbolic AI** área de pesquisa ativa.

### LLM + knowledge graph

- KG para consulta estruturada, consistente.
- LLM para linguagem natural em cima.

### LLM + search

- Retrieval clássico tem precision em termos exatos.
- Vector em paráfrase.
- Combina.

### LLM + humano

Ver HITL.

## Caching Estratégico

### Exact cache

Hash de prompt → resposta. Útil em FAQ, respostas repetidas.

### Semantic cache

Embed query nova; se similar a query cacheada (cosine > threshold), reutilizar resposta. Ganho grande em queries frequentes com paráfrases. **GPTCache, Redis + vector, Portkey**.

Risco: similaridade ≠ equivalência. Calibrar threshold; monitorar falsos positivos (resposta errada servida).

### Prefix cache

Já mencionado — automático em OpenAI, Anthropic.

### Partial computation cache

Em pipeline multi-step, cachear intermediários (retrieval results, embeddings).

## Deploy Topologies

### Cloud API

Modelo de terceiro (OpenAI, Anthropic, Google). Mais simples; menos controle; custo variável; dados externos.

### Cloud hosted

Modelo open-weights em sua cloud (AWS Bedrock, Azure AI, GCP Vertex, Replicate, Together, Fireworks, Anyscale).

### Self-hosted

Sua própria infra com vLLM/TGI/llama.cpp. Máximo controle; paga GPU direto; precisa ops. Ver `GPU.md`.

### Edge / On-device

Apple Intelligence (on-device), mobile (Gemini Nano, Llama 3 quantizado), desktop (Ollama). Latência baixa, privacidade, offline. Modelos menores (1-8B).

### Hybrid

Combinação. Comum: edge para simples + cloud para complexo; enterprise on-prem para dados sensíveis + API para mais inteligência.

## Observability

Observability genérica (`OBSERVABILITY.md`) + específica de IA:

- **Traces**: prompt completo, modelo, parâmetros, resposta, tools chamadas, tempo por passo.
- **Token metrics**: input, output, cached, por request.
- **Cost per request**: $ em dólar e em ciclo de billing.
- **Latency breakdown**: time-to-first-token, tokens/sec, time-to-last-token.
- **Quality metrics**: evals periódicos, user feedback, thumbs up/down.
- **Hallucination signals**: fora de distribuição, contradição com source, refusal rate.
- **Drift**: performance em subsetes (usuário X, tópico Y) ao longo do tempo.

Ferramentas: **Langfuse, LangSmith, Arize Phoenix, Helicone, Braintrust, Weave (W&B), Honeycomb com custom spans**.

## Evals-in-the-Loop

Avaliação não é one-shot; é contínua.

### Tipos

- **Unit-like evals**: assertion em output (contém X, formato Y).
- **Reference-based**: comparar com ground truth. BLEU, ROUGE (fracos para LLM), exact match, embedding similarity.
- **LLM-as-judge**: outro LLM avalia resposta. Escalável, tem biases (length bias, position bias).
- **Human evals**: gold standard caro.
- **Online evals**: A/B, thumbs, session completion rate.
- **Task-specific**: accuracy em QA, code pass@k, factuality via retrieval check.

### Regression gate

Antes de deploy, eval suite roda. Deploy bloqueado se qualidade cai em datasets críticos. Análogo a testes em CI/CD.

### Continuous improvement

Production traces rotulados periodicamente → novo eval set → detecta drift + alimenta melhorias.

## Versionamento

Na IA, versionam-se **juntos**:

- **Modelo** (nome + versão, hash de weights se self-hosted).
- **Prompt** (system + templates).
- **Dataset** (para RAG: snapshot do corpus; para fine-tune: snapshot do training).
- **Configuração** (temperature, max tokens, tool list).

Deploy é **tupla** `(modelo, prompt, config, dataset)`. Mudar qualquer um = nova versão. Rollback deve restaurar tupla inteira.

Ferramentas: **Promptfoo, LangSmith prompt hub, Helicone prompt management, dedicated prompt registry**.

## Custo — Modelagem Explícita

Custo de sistema LLM deve ser modelado como:

```
custo/sessão = Σ (input_tokens · preço_in + output_tokens · preço_out)
               + (embedding calls)
               + (reranker calls)
               + (vector DB ops)
               + (GPU hours se self-hosted)
```

Dimensionar em planilha; monitorar em produção; alerta em desvio.

Otimizações comuns por ROI:

1. **Caching** de prompt prefix (10-90% redução de tokens).
2. **Cascata de modelos** (30-70% de custo em workloads mistos).
3. **Quantização** se self-hosted.
4. **Reduzir chunks desnecessários** em RAG.
5. **Prompt compression** (LLMLingua).
6. **Paragrafar/truncar output** quando verbose custa.

## Segurança Arquitetural

Ver `AI_SECURITY.md` para ameaças. Arquitetonicamente:

- **Separar** contexto confiável (instruções) de não-confiável (input usuário, docs). Marcar claramente em prompt.
- **Least authority** em tools do agente.
- **Sandbox** de execução de código.
- **Egress control**: agente não pode chamar qualquer URL.
- **PII handling**: redação antes de prompt, minimização, não logar dados sensíveis verbatim.
- **Prompt leak**: assumir que system prompt vai vazar. Não colocar segredos ali.

## Anti-Padrões

1. **LLM para tudo**: quando regex + lookup basta, LLM adiciona custo/latência/ambiguidade.
2. **Um único modelo para tudo**: roteamento economiza muito.
3. **Contexto infinito como muleta**: engorda prompt sem melhorar — "lost in the middle".
4. **Evals só offline**: production drift fica invisível.
5. **Prompt como arte, não como código**: sem versionamento, test, review → comportamento varia misteriosamente.
6. **Agentes com acesso total**: confused deputy explorável (ver `AI_SECURITY.md`).
7. **RAG naïve com chunk size 1**: perde contexto; com 10k tokens: perde foco.
8. **Sem fallback**: provider down = app down.
9. **Sem rate limit por usuário**: um usuário abusivo derruba ou dispara conta.
10. **Re-treinar como primeira solução**: prompt engineering + RAG resolvem 80-90% dos casos com muito menos custo.

## Fluxograma de Decisão

Ao desenhar sistema LLM, em ordem:

1. **Prompt bem feito** resolve? (engenharia de prompt, few-shot)
2. Se não, **RAG** com conhecimento relevante resolve?
3. Se não, **tool use / function calling** resolve (acesso a APIs, cálculos)?
4. Se não, **agente** (decisão multi-step) é necessário?
5. Se não, **fine-tuning** com dados próprios resolve?
6. Se não, revisitar se problema é realmente bem-adequado a LLM.

Cada passo adiciona complexidade + custo; resistir à escalada prematura.

## Ferramentas e Frameworks

### Orquestração

- **LangChain** (Python/JS): amplo, pesado, fluxo de fundamentos.
- **LlamaIndex**: foco em RAG.
- **LangGraph**: state machines para agents; abstrações claras.
- **Haystack**: enterprise RAG.
- **Pydantic AI**: typed, minimalista.
- **Mastra, Inngest**: alternativas leves.

### Evals

- **Promptfoo**: CLI eval local.
- **Braintrust**: SaaS evals + experiment.
- **Arize Phoenix**: open source.
- **DeepEval, RAGAS**: RAG-specific.
- **LangSmith**: Langchain-native.

### Serving

- **vLLM, TGI, TensorRT-LLM, SGLang**: self-hosted high-throughput.
- **Ollama, LM Studio**: local dev.
- **Replicate, Together, Fireworks, Anyscale, Groq, Modal**: cloud API de open models.

### Observability

- **Langfuse, Helicone, LangSmith, Phoenix, Weave, Portkey**.

### Vector DBs

Ver `SEARCH_AND_RANKING.md`.

## Recursos

- **"Building Effective Agents"** — Anthropic engineering blog, 2024. Referência concisa.
- **"LLM Powered Autonomous Agents"** — Lilian Weng, OpenAI.
- **"Prompt Engineering Guide"** (promptingguide.ai).
- **"Hands-On Large Language Models"** — Jay Alammar, Maarten Grootendorst.
- **"Designing Machine Learning Systems"** — Chip Huyen (parte AI systems).
- **Eugene Yan blog** (eugeneyan.com) — padrões aplicados.
- **Hamel Husain blog** — avaliação, fine-tuning.
- **Simon Willison** — prompt injection, patterns.
- **OpenAI Cookbook, Anthropic docs, Google AI docs** — patterns oficiais.

## Princípios

1. **Comece simples**. Prompt + few-shot resolve mais do que se imagina. Não adicione RAG/agent sem necessidade.
2. **Modelo é dependência mutável**. Versão + prompt + dataset junto; regression evals bloqueiam deploy.
3. **Custo × latência × qualidade sempre, simultâneos**. Trade-off triplo é nova normal.
4. **Falha silenciosa é o inimigo**. Evals contínuos + observability rica + feedback loop.
5. **Roteamento reduz custo em ordem de magnitude**. Modelo grande só quando preciso.
6. **Guardrails arquiteturais, não cosmética**. Menor autoridade, approval, sandbox.
7. **Cache sempre que fizer sentido**. Prefix, semântico, intermediário.
8. **Não-determinismo é feature, não bug**. Design assumindo; teste com N amostras.
9. **Humano no loop em decisões reversíveis-caras**. Confidence calibrada define onde.
10. **Observability específica**. Tokens, custo, traces com prompts são primeiro-cidadão.
