# RETRIEVAL

Subsistema de **retrieval** do `AGENTIC_CLI`: pipeline `query → candidates → expansion → ranking → compression → context slot`. Decide **WHAT** entra no contexto, dado um goal. `CONTEXT_TUNING.md` decide **HOW** formatar o que já foi decidido entrar.

Não é "knowledge graph". O grafo é detalhe de schema (`§8`). O objeto de primeira classe é o pipeline e os signals de ranking. Sem isso declarado, "memória infinita" ou "RAG" viram cargo cult: muito armazenamento, retrieval ruim, contexto inchado, modelo pior.

Premissa raiz (igual ao resto do AGENTIC_CLI): *meça antes de cortar tokens*. Retrieval é a etapa que mede relevância; sem ela, `CONTEXT_TUNING` corta às cegas.

---

## 0. Princípios (não-negociáveis)

1. **Relevância > recência > similaridade.** Embedding diz "similar", não "relevante". Ranking precisa de sinais estruturais e causais, não só cosine.
2. **Lexical/estrutural primeiro, semântico opt-in.** BM25 + symbol graph cobre 80%+ dos casos em código. Vector store entra quando dor for medida, não antes (`ANTI_PATTERNS.md §2.2`, `CODE_INDEX.md §0.8`).
3. **Graph é storage, não conceito de venda.** A spec descreve operações; storage pode ser SQLite com edges, in-memory adjacency, ou o que for — desde que respeite invariantes (`§8`).
4. **Três views, um pipeline.** `workspace` (estática, `CODE_INDEX`), `session` (execução, `STATE_MACHINE`), `memory` (persistente, `MEMORY.md`). Sources diferentes, retrieval igual.
5. **Expansion bounded.** Traversal sem depth-limit explode. Cada hop custa; hop budget é declarado per-workflow.
6. **Ranking auditável.** Cada candidato emitido com score breakdown. Sem black box. Trace é obrigatório, não opcional (`§10`).
7. **Compression é ranking final, não pre-processing.** Summarize só depois de ranquear — comprimir antes desperdiça candidato bom.
8. **Sem esquecimento silencioso.** O que foi cortado fica em trail; modelo pode pedir expansão explícita.
9. **Context budget é load-bearing.** Cada query carrega budget; sem budget, retrieval degenera para "joga tudo no prompt".
10. **Decay opt-in, não default.** Session decai rápido (recovery foca recente); memory decai devagar (precedent envelhece); workspace **não** decai (FS é autoritativo).

---

## 1. Escopo

### 1.1 O que resolve

- "Dado goal atual, que subset de workspace/session/memory entra no contexto?"
- "Como ranquear N candidatos quando budget < N?"
- "Como comprimir sem perder load-bearing detail?"
- "Por que isso entrou e aquilo saiu?" (auditabilidade)

### 1.2 O que **não** resolve (escopo declarado)

- **Geração do índice de symbols.** Owner: `CODE_INDEX.md`.
- **Persistência de memory.** Owner: `MEMORY.md`.
- **Estado de execução.** Owner: `STATE_MACHINE.md`.
- **Shape final do prompt.** Owner: `CONTEXT_TUNING.md`. Retrieval entrega *candidatos selecionados + compression*; tuning posiciona.
- **Decisão de quando consultar.** Owner: orchestrator (`ORCHESTRATION.md`). Retrieval é tool, não driver.

### 1.3 Não-objetivos explícitos

- "Memória infinita". Objetivo é **densidade informacional**, não volume.
- Vector embedding como primeira linha. Reconsiderar quando lexical + estrutural for *medidamente* insuficiente.
- Cross-view linking automático em v1 (memory pode citar workspace symbol manualmente; auto-link em v2).

---

## 2. Pipeline canônico

```
query
  ├─ candidate generation  (per view)
  ├─ expansion             (bounded traversal)
  ├─ ranking               (multi-signal fusion)
  ├─ compression           (per budget)
  └─ context slot          (handed to CONTEXT_TUNING)
```

Cada etapa observável (`§10`), cada etapa com budget, cada etapa com trail.

### 2.1 Query types

| Tipo | Exemplo | Views primárias |
|---|---|---|
| **Symbol** | "find `AuthService`" | workspace |
| **Semântica** | "auth timeout fix" | todas as três |
| **Causal** | "por que esse step falhou" | session |
| **Precedent** | "fizemos isso antes?" | memory |
| **Navigational** | "callers de `validateToken`" | workspace |

Driver (orchestrator) decide tipo; retrieval respeita.

---

## 3. Candidate generation

### 3.1 As três views

| View | Source | Edges típicas | Use-case dominante |
|---|---|---|---|
| **Workspace** | `CODE_INDEX` | `imports`, `calls`, `references` | refactor, explain, navigate |
| **Session** | `STATE_MACHINE` + tool calls | `goal→task`, `task→edit`, `edit→failure` | recovery, replan |
| **Memory** | `MEMORY.md` store | `mentions`, `similar_to`, `fixed_by` | precedent lookup |

Cada view exporta `search(query) → [candidates]`. Pipeline funde resultados (`§5.2`).

### 3.2 Per-view bootstrap (sinais iniciais)

- **Workspace:** BM25 sobre symbols + path + outline. Hit → nó(s) seed.
- **Session:** match sobre goal text, task description, failure reason. Recência boost.
- **Memory:** BM25 sobre title/body + tag match. Embedding opt-in (v2).

### 3.3 Sem união cega

Candidato sai com `(node_id, view, bootstrap_score, reason)`. Reason permite trace.

### 3.4 Skills gating (quarto candidato, mecanismo distinto)

Skills não são uma quarta view — não têm grafo, traversal, nem decay temporal. Mas resolvem o mesmo problema upstream: **dado goal atual, que subset de N skills disponíveis merece aparecer no prompt?** Pipeline acima é overkill; ranking estrutural não se aplica (skill não tem `calls`/`imports`).

Mecanismo: **description-based progressive disclosure** (mesmo padrão que Claude Code usa pra skills/subagents/MCP tools). Não é busca semântica via embedding — é gating via LLM lendo descrições curtas.

#### 3.4.1 Estrutura por skill

```
---
name: <slug>
description: <1 linha, ≤120 chars, optimizada pra LLM decidir relevância>
trigger_keywords: [opcional, lista]   # pré-filtro lexical barato
---
<body — carrega lazy>
```

`description` é load-bearing. Vago ("ajuda com código") = nunca invocada ou sempre invocada errado. Específica ("renomeia símbolos via tree-sitter respeitando escopo") = gate funciona.

#### 3.4.2 Duas fases

| Fase | Quando | Custo | Saída |
|---|---|---|---|
| **Surface** | sempre, no system prompt | `≈ N × 30t` (só nome + description) | catálogo visível |
| **Load** | quando modelo invoca | corpo da skill | body completo, contexto da turn |

Análogo a `MEMORY §4` (eager index, lazy content). Skill nunca entra `full` antes de invocada — caso contrário N grandes skills inflacionam todo prompt.

#### 3.4.3 Pré-filtro opcional (quando N grande)

Se catálogo passa de ~30 skills (ordem de grandeza, não dogma), surface vira ruidoso. Adicionar pré-filtro lexical antes do surface:

```
relevant = [s for s in skills if
  any(kw in goal_text for kw in s.trigger_keywords)
  or BM25(s.description, goal_text) > threshold]
```

Pré-filtro é **opt-in** e **lexical** — mesmo princípio de `§0.2`. Embedding sobre descriptions só entra se medir dor.

Skills sem `trigger_keywords` e sem hit BM25 ainda entram via fallback "general" group (sempre surfaceado), pra evitar invisibilidade silenciosa.

#### 3.4.4 Por que não ranquear via grafo

- Skill não tem edge natural pro código (`calls`, `imports`) — forçar isso é cargo cult.
- Goal-alignment via cosine entre `description` e goal text **é** ranking semântico, e cai na trava de `§0.2` (opt-in v2).
- Modelo é melhor selecionador que weighted-sum aqui: descrição curta + nome basta pra decisão tool-call-like.

#### 3.4.5 Audit

Mesmo princípio de `§5.3`: registrar quais skills foram **surfaced**, quais foram **invoked**, quais foram **filtered out** no pré-filtro. Permite afinar `description` e `trigger_keywords` com base em miss/false-positive.

```
{
  query_id,
  skills_surfaced:   [name, ...],
  skills_invoked:    [name, ...],
  skills_filtered:   [{name, reason: "no kw match"}, ...]
}
```

#### 3.4.6 Anti-patterns específicos

| Pattern | Por que dói |
|---|---|
| `description` vaga ou genérica | gate quebra; skill nunca invocada ou invocada errado |
| Carregar body de todas skills eager | infla system prompt linearmente com N |
| Ranquear skills por cosine vs goal embedding | adiciona infra (embed) pra problema que LLM resolve com 30t de description |
| Skill como "tudo que pode ser útil às vezes" | catálogo cresce, sinal/ruído degrada, modelo deixa de invocar até as boas |
| Skills sem audit | impossível afinar descriptions com base em uso real |

---

## 4. Expansion (bounded traversal)

### 4.1 Hop budget

| Workflow | Hops default | Justificativa |
|---|---|---|
| Review | 1 | diff já delimita |
| Refactor | 3 | callers + callers-of-callers |
| Explain | 2 | symbol + dependências diretas |
| Debug/Recovery | 2 (session) + 1 (workspace) | failure path + código relacionado |
| Default | 2 | meio termo |

Hard cap: `5`. Acima disso, retrieval rejeita request (config error, não silent truncate).

### 4.2 Edge weights default

| Edge kind | Peso | Source |
|---|---|---|
| `calls` | 0.9 | workspace |
| `imports` | 0.8 | workspace |
| `references` | 0.7 | workspace |
| `defined_in` | 0.9 | workspace |
| `mentioned_in` | 0.4 | cross-view |
| `similar_to` (embedding) | 0.5 | opcional |
| `fixed_by` | 0.8 | memory |
| `caused_by` | 0.8 | session |
| `goal_of` | 1.0 | session (sempre relevante) |
| `precedent_for` | 0.6 | memory |

Pesos são default; per-workflow override é permitido (`§5.2`).

### 4.3 Decay temporal

| View | Half-life | Razão |
|---|---|---|
| Session | 1h | recovery foca recente; goal antigo perdeu relevância |
| Memory | 30d | precedent envelhece mas não morre |
| Workspace | — | sem decay; FS é estado, não evento |

Decay aplica multiplicativamente sobre peso da edge no momento do ranking, **não** muta o storage.

### 4.4 Pruning

Durante traversal, candidato com `running_score < threshold` é descartado antes de expandir. Threshold per-workflow; default `0.2`. Evita explosão.

---

## 5. Ranking

### 5.1 Signals

| Signal | Descrição | Custo |
|---|---|---|
| **Structural** | distância no grafo + soma de edge weights no path | baixo |
| **Lexical** | BM25 sobre nome/conteúdo/outline | baixo |
| **Semantic** | embedding cosine (opt-in v2) | médio (eval), alto (store) |
| **Temporal** | recência ponderada por half-life da view | baixo |
| **Usage** | quantas sessions citaram nó (uplift) | baixo |
| **Goal-alignment** | match contra goal canônico (`CONTEXT_TUNING §1.6`) | baixo |

### 5.2 Fusion (weighted sum)

```
score(c) = Σ w_i · signal_i(c)
```

Pesos per-workflow (exemplos):

| Workflow | structural | lexical | semantic | temporal | usage | goal |
|---|---|---|---|---|---|---|
| Review | 0.4 | 0.3 | 0.0 | 0.0 | 0.1 | 0.2 |
| Refactor | 0.5 | 0.2 | 0.0 | 0.0 | 0.1 | 0.2 |
| Debug | 0.3 | 0.2 | 0.0 | 0.4 | 0.0 | 0.1 |
| Explain | 0.3 | 0.3 | 0.1 | 0.0 | 0.1 | 0.2 |
| Precedent lookup | 0.1 | 0.3 | 0.2 | 0.1 | 0.1 | 0.2 |

Pesos somam 1.0; normalização explícita evita drift.

### 5.3 Score breakdown obrigatório

Cada candidato emerge com:

```
{
  node_id, view, final_score,
  signals: { structural: 0.7, lexical: 0.5, ... },
  path: [seed → hop1 → hop2],
  reason: "called by AuthService.validate (workspace, 2 hops)"
}
```

Sem breakdown, ranking vira black box e elimina audit (`AUDIT.md`).

---

## 6. Compression

### 6.1 Hierarquia de representação

| Nível | Conteúdo | Custo (tokens, ordem) |
|---|---|---|
| `full` | conteúdo bruto (`read_file`, full memory entry) | 100s–1000s |
| `outline` | symbols + signatures, sem corpos | 10s–100s |
| `summary` | resumo curto, cacheado | 10s |
| `ref` | só `path:lineno` ou `memory#id` | 1–3 |

### 6.2 Decisão por budget

Algoritmo simples, declarado:

```
remaining = budget
for c in ranked:
  for level in [full, outline, summary, ref]:
    if cost(c, level) <= remaining:
      include(c, level)
      remaining -= cost(c, level)
      break
  else:
    skip(c)  # registrado em trail
```

Top-K sempre tenta `full` primeiro; tail degrada.

### 6.3 Summaries cacheados

- Summary de symbol/file: invalida com hash do conteúdo (`CODE_INDEX §...`).
- Summary de memory entry: invalida com `updated_at` da entry.
- Summary de session task: cacheado por task_id, imutável após task done.

### 6.4 Trail de cortes

Compression registra:

```
{ included: [...], skipped: [{node, reason, would_cost}] }
```

Modelo pode requisitar expansão explícita: "skipped `UserRepo` (would cost 800t); pedir full?" — decisão dele, não silent loss.

---

## 7. Context budget

### 7.1 Per-slot (orientativo)

Workflow define a divisão; valores são exemplos, não dogma:

| Workflow | workspace | session | memory |
|---|---|---|---|
| Review | 70% | 20% | 10% |
| Refactor | 60% | 25% | 15% |
| Debug/Recovery | 25% | 55% | 20% |
| Explain | 50% | 20% | 30% |

Total ≤ budget global definido por `CONTEXT_TUNING §N`.

### 7.2 Overflow handling

Budget estourou em uma view → **degrada nessa view** (outline em vez de full), não rouba de outra. Cross-view stealing introduz acoplamento ruim.

---

## 8. Schema mínimo (storage-agnóstico)

### 8.1 Node

```
{
  id:          <stable identifier, unique por (source, kind, key)>,
  source:      workspace | session | memory,
  kind:        symbol | file | task | edit | failure | goal | memory_entry | ...,
  payload:     <kind-specific; pode ser ref a outra store, ex: CODE_INDEX symbol_id>,
  created_at,
  updated_at
}
```

Payload **não precisa duplicar** o que vive em CODE_INDEX/MEMORY. Pode ser apenas `{ symbol_id: "..." }` e a view resolve no read.

### 8.2 Edge

```
{
  src:         <node_id>,
  dst:         <node_id>,
  kind:        <calls | imports | references | mentions | fixed_by | ...>,
  weight:      0.0..1.0  (default por kind se omitido),
  derivation:  derived  | declared,
  created_at
}
```

`derived` = inferido (ex: tree-sitter import edge). `declared` = humano/agent escreveu (ex: memory `fixed_by`).

### 8.3 Invariantes

- Node tem `source`; edge entre nós de sources diferentes é **permitido** (memory entry → workspace symbol via `mentions`).
- Edge tem `weight`; ausência ⇒ default por `kind`. Sem default ⇒ erro de schema, não 0.
- Storage pode ser SQLite com tabelas `node`/`edge`, KV+índices, in-memory adjacency, ou Neo4j. Spec não obriga.
- Reindex (de qualquer view) invalida edges `derived` da view; edges `declared` sobrevivem.

### 8.4 Não-invariantes deliberados

- **Sem schema rígido por kind.** Payload é opaco; só a view que produziu sabe interpretar.
- **Sem unicidade global de edges** (mesmo `kind`, `src`, `dst` pode aparecer com `derivation` diferente — útil pra precedência).

---

## 9. Operações mínimas

Toda implementação expõe (assinaturas conceituais, não API):

| Op | Entrada | Saída |
|---|---|---|
| `search(query, view?, limit)` | texto + filtros | candidates ordenados por bootstrap_score |
| `neighbors(node, edge_kinds?, max)` | nó | edges saindo, filtradas |
| `traverse(seeds, hops, edge_filter, prune_threshold)` | seeds + bounds | subgrafo expandido + paths |
| `rank(candidates, workflow)` | candidates | candidates com `final_score` + breakdown |
| `compress(ranked, budget, hierarchy)` | ranked + budget | context slot pronto |
| `trace(query_id)` | id | dump completo do pipeline (debug/audit) |

`search` + `traverse` + `rank` + `compress` é o pipeline. Cada uma é testável isoladamente.

---

## 10. Observabilidade

### 10.1 Trace obrigatório

Cada query produz registro:

```
{
  query_id, query_text, workflow, budget,
  candidates_raw:        [...],     # antes de expansion
  candidates_expanded:   [...],     # depois de traversal
  candidates_ranked:     [...],     # com score breakdown
  context_slot:          {...},     # final, com per-candidate level
  skipped:               [...],     # com razão e custo hipotético
  timings:               { search_ms, expand_ms, rank_ms, compress_ms }
}
```

Integra com `AUDIT.md` (decisão é auditável).

### 10.2 Métricas

- **Budget utilization** (média / por workflow).
- **Eviction rate** (% candidatos ranqueados mas cortados).
- **Diversity** (entropia de views no context slot — evita slot monopolizado).
- **Latency** por etapa (p50/p95).
- **Cache hit rate** em summaries.

### 10.3 Offline eval

Replay de queries históricas com ground truth (humano marcou "isto deveria estar no contexto"). Compara recall@K, precision@K, MRR. Pré-requisito para qualquer mudança de pesos em `§5.2`.

---

## 11. Anti-patterns (link com `ANTI_PATTERNS.md`)

| Pattern | Por que dói |
|---|---|
| Embedding como primeira linha sem medir lexical | custo alto, ganho marginal em código |
| Traversal sem depth cap | explosion, latência ruim, contexto poluído |
| Memory sem decay | precedent envelhecido domina sobre recente |
| Black-box ranking | impossível debugar contexto ruim |
| Compression sem trail | esquecimento silencioso, modelo "perde" sem saber |
| "Knowledge graph" sem query plan declarado | virou Neo4j + buzzwords (citação direta do dump original) |
| Cross-view budget stealing | acoplamento ruim, debug pesadelar |
| Pesos fixos sem eval offline | drift de qualidade silencioso entre versões |

---

## 12. v1 / v2 / v3

### v1 (mínimo viável)

- Três views funcionando, cada uma com `search` próprio.
- Pipeline completo: candidates → expansion (estrutural) → ranking (estrutural + lexical + temporal) → compression hierárquica.
- Sem embedding. Sem cross-view edges automáticas.
- Trace obrigatório, métricas básicas.

### v2

- Embedding opt-in (configurável); semantic signal entra no fusion.
- Cross-view edges derivadas (memory entry menciona path → edge `mentions` workspace).
- Offline eval framework.
- Usage signal habilitado (requer histórico).

### v3

- Aprendizado de pesos via session feedback (modelo marcou candidate como inútil → peso decai).
- Active subgraph cache (`hot subgraph` por sessão).
- Adaptive hop budget (modelo pede expansão sob demanda).

---

## 13. Relação com outros docs

| Doc | Relação |
|---|---|
| `CODE_INDEX.md` | source de dados workspace; retrieval consome, não regenera. |
| `MEMORY.md` | source de dados memory; mesma relação. |
| `STATE_MACHINE.md` | source de dados session; eventos viram nós/edges. |
| `CONTEXT_TUNING.md` | consumidor downstream; recebe context slot, decide posicionamento. |
| `TOKEN_TUNING.md` | budget global vem daqui; retrieval respeita. |
| `AUDIT.md` | trace de retrieval é input pra audit. |
| `ORCHESTRATION.md` | driver decide quando chamar retrieval e com que workflow. |
| `ANTI_PATTERNS.md` | §11 referencia patterns canônicos. |

---

## 14. Insight final (não-instrução, contexto)

O LLM lê tokens lineares. O grafo não muda isso. O que muda é **quais tokens merecem existir naquele prompt**. Retrieval é a função que responde essa pergunta com trail; tudo o mais — schema, storage, edges — é mecânica que serve a essa função.

Contexto linear escala mal porque a função de relevância é implícita ("últimas N mensagens"). Retrieval declarado torna a função explícita, mensurável, auditável. Esse é o ganho real, não o grafo em si.

---

## 15. Integration map

Esta seção responde "onde isso vive fisicamente". Resposta curta: **o grafo não é um store novo**. É projeção lógica sobre os subsistemas existentes. RETRIEVAL é adapter + ranker + compressor; quem é dono de dado continua dono.

### 15.1 Onde cada view já vive

| View | Store físico | Estruturas já existentes que **são edges** |
|---|---|---|
| Workspace | `CODE_INDEX` SQLite (`CODE_INDEX §2`) | tabelas `references_`, `imports`, `symbols` — cada linha é `(src, dst, kind)` |
| Session | storage de `STATE_MACHINE` (goal stack, step events) | DAG implícito `goal → task → step → tool_call → edit → failure` |
| Memory | arquivos em `~/.config/agent/memory/` + SQLite de eventos (`MEMORY §3`) | frontmatter `[[name]]` links + tags |

Nada disso precisa migrar. RETRIEVAL consome o que está lá.

### 15.2 Cross-store joins

`CODE_INDEX §10` já autoriza `ATTACH DATABASE` para combinações declaradas (`code_index + sessions`). RETRIEVAL usa esse mecanismo para query cross-view, **não** abre nova superfície de acoplamento.

Combinações canônicas (declaradas; fora disso = recusa):

| Combinação | Use-case |
|---|---|
| `code_index + sessions` | "que arquivos foram editados no último goal X" |
| `sessions + memory` | "essa failure já aconteceu antes?" |
| `code_index + memory` | "memory cita esse symbol?" |

### 15.3 O que é genuinamente novo

Só três coisas, todas pequenas:

| Componente | Onde mora | Tamanho |
|---|---|---|
| Ranking module (`§5`) | código puro, sem store | módulo |
| Trace store (`§10.1`) | nova tabela `retrieval_trace` no SQLite de sessions | 1 tabela |
| Cross-view declared edges | **frontmatter dos arquivos de memory** (ex: `fixed_by: session/abc#step42`) | zero infra; parsing on read |

Sem Neo4j. Sem migração. Sem store dedicado de grafo.

### 15.4 Insertion points no loop

Plugando em `ORCHESTRATION §1.1`:

```
[step start]
  ├─ goal re-injection           (CONTEXT_TUNING §1.6)
  ├─ RETRIEVAL.query(goal, workflow)     ◄── aqui
  │    └─► context_slot (full | outline | summary | ref)
  ├─ CONTEXT_TUNING shapes prompt
  └─ model turn
[step end]
  ├─ STATE_MACHINE update (novos nós session: edit, failure, tool_call)
  └─ RETRIEVAL.invalidate(affected_paths)  ◄── invalida summaries cacheados
```

Em DAG profile (`ORCHESTRATION §2`), cada node chama `retrieval.query` com seu próprio `workflow` (refactor/review/explain/debug). Nodes paralelos podem ter slots concorrentes; budget é per-node (`ORCHESTRATION §2.4`).

### 15.5 Interação com compaction

`ORCHESTRATION §4` (compaction) ganha um critério novo, declarado: em vez de heurística "drop oldest tool_results", compaction pergunta ao RETRIEVAL:

> "dado o goal ativo, que candidatos do contexto atual ainda estariam no top-K se eu refizesse a query agora?"

O que **não** estaria é candidato natural a evict. Compaction vira *retrieval re-run com budget menor* — usa a mesma função de relevância, garante coerência entre "o que entrou" e "o que fica".

Fallback determinístico (`ORCHESTRATION §4.6`) sobrevive: se retrieval estiver indisponível (degradação), compaction cai pra heurística antiga, com warning no trace.

### 15.6 Diagrama de dependência

```
   CODE_INDEX  ─┐
   STATE_MACH. ─┼─►  RETRIEVAL  ─►  CONTEXT_TUNING  ─►  model
   MEMORY      ─┘       ▲                                  │
                        │                                  ▼
                  ORCHESTRATION  ◄────────────  new session events
                        │
                        └─► AUDIT (consome trace de §10)
```

Direção das setas é direção de dependência. RETRIEVAL depende dos três stores; CONTEXT_TUNING depende de RETRIEVAL; AUDIT depende do trace. Nenhuma seta de volta — RETRIEVAL não muta CODE_INDEX, MEMORY, ou STATE_MACHINE.

### 15.7 Profiles

| Profile (`AGENTIC_CLI §...`) | RETRIEVAL ativo? | Notas |
|---|---|---|
| `headless --json` | sim, opcional | sem trace persistente (stateless); ranking ainda aplica |
| `interactive` | sim | trace local, sem upload |
| `orchestrated` (local-first, Ollama) | sim | cross-view joins permitidos; embedding ainda v2 |
| `autonomous` | sim | retrieval é load-bearing aqui (loop longo, compaction frequente) |

Degradação: RETRIEVAL indisponível ⇒ `CONTEXT_TUNING` cai pra inclusão default (read_file de paths citados explicitamente no goal). Sessão funciona pior, mas funciona — paritário com `CODE_INDEX §0.9` ("amenity, não dependência runtime obrigatória").

### 15.8 Resumo da integração

- **Zero novo store de grafo.** Edges já existem nos schemas atuais.
- **Uma tabela nova** (`retrieval_trace`) no SQLite de sessions.
- **Frontmatter de memory** vira fonte de cross-view declared edges (parsing on read).
- **Dois insertion points** no loop: `query` no step start, `invalidate` no step end.
- **Compaction** passa a delegar critério de evict ao RETRIEVAL.
- **Nenhum subsistema existente perde ownership** de seus dados.
