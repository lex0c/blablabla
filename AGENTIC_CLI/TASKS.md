# TASKS

Subsistema de **decomposição de problema** do `AGENTIC_CLI`. Cobre desde checklist leve em single-agent (planning visível ao usuário) até decomposição executável em multi-agent (DAG + subagents). Decide **COMO** um problema vira N passos rastreáveis — sem virar tool de meta-cognição.

Não é "todo list app". O objeto de primeira classe é a **fronteira entre planejamento e ação**: ação consome slot do catálogo de tools (`CONTRACTS.md §2.6`); planejamento consome ciclo do harness. Sem essa fronteira declarada, o tool budget é gasto em ferramentas que não mexem em nada e a auditabilidade do plano vira folclore.

Premissa raiz (igual ao resto do AGENTIC_CLI): *meça duas vezes, corte uma*. Aqui a medição é **prospectiva** — o que vamos cortar, em que ordem, com que budget? Sem isso declarado, o agente pula direto pra `bash` e descobre na execução.

---

## 0. Princípios (não-negociáveis)

1. **Meta-cognição não é tool.** Modelo decide ações **no mundo** (FS, shell, subagent, fetch). Decisões **sobre o modelo** (planejar, lembrar, comprimir contexto) são responsabilidade do harness. Princípio guia herdado de `CONTRACTS.md §2.6.8`.
2. **Plano ≠ ação.** Emissão de checklist em prosa não consome tool slot. Execução de cada item consome — `edit_file`, `bash`, `task_*`, etc.
3. **Dois tracks, uma tabela.** Single-agent (checklist via stream parsing) e multi-agent (DAG + subagents) escrevem na mesma tabela `todos` — granularidade diferente, schema igual.
4. **Source of truth é SQLite.** UI renderiza a partir da tabela, **não** do stream cru. Crash mid-session reconstrói plano lendo `todos`.
5. **Auditabilidade obrigatória.** Cada item emitido é evento (`todo_added`); cada transição de status é evento (`todo_status_changed`). Sem isso, princípio 7 ("Trace tudo") tem buraco.
6. **Sem planning silencioso.** Falha de parse vira `failure_event` classificado, não item perdido. Modelo é notificado e re-emite.
7. **Budget é load-bearing.** Plano sem budget degenera em "lista infinita de TODOs". Cada nó (todo, DAG node, subagent) carrega budget ou herda do pai.
8. **Decomposição é opt-in pro modelo.** Spec não força checklist — modelo emite se julgar útil. Forçar planning prematuro polui sessões triviais ("fix typo" não precisa de 5-item plan).

---

## 1. Escopo

### 1.1 O que resolve

- "Como modelo decompõe problema em N passos sem virar tool de meta-cognição?"
- "Como UI mostra progresso live sem novo round-trip ao modelo?"
- "Como reconstruir plano após crash mid-session?"
- "Como auditar decisão de quebrar (ou não quebrar) um problema?"
- "Quando decomposição vira multi-agent (DAG) vs single-agent (checklist inline)?"

### 1.2 O que **não** resolve (escopo declarado)

- **Execução do passo.** Owner: tools individuais (`CONTRACTS.md`). `todos` só registra intenção e status.
- **Compaction de contexto.** Owner: `CONTEXT_TUNING.md`. Plano pode ser comprimido como qualquer outro span.
- **Orquestração do DAG.** Owner: `ORCHESTRATION.md §2`. `todos` recebe events de nó, não executa.
- **UI rendering.** Owner: `UI.md §2.6` (TodoChecklistPane). Este doc define schema; UI define pixels.
- **Memory.** Owner: `MEMORY.md`. Todos são session-scoped por default; promoção pra memory é decisão separada.

---

## 2. Dois tracks

| Track | Quando | Granularidade | Execução | Owner |
|---|---|---|---|---|
| **A. Checklist inline** | single-agent, plan curto-médio (≤ 15 itens) | item = 1 tool call ou small chain | inline no step loop | este doc |
| **B. DAG + subagents** | multi-agent, paralelismo, isolamento de contexto | nó = subagent independente | profile `orchestrated` | `ORCHESTRATION.md §2` |

Os dois escrevem na mesma tabela `todos` (§3). DAG nodes que spawnam subagents geram `todos` com `parent_todo_id` apontando pro nó pai e `subagent_session_id` populado.

### 2.1 Critério de promoção A → B

Sinais que indicam que checklist inline não basta:

- Itens são **independentes** e paralelizáveis (latência > 30s cada).
- Itens requerem **contextos isolados** (subagent explora dir X, outro dir Y, sem cross-talk).
- Plano tem **fan-out ≥ 3** com agregação posterior.
- Algum item requer **budget separado** (custo > 20% do budget da sessão).

Sem nenhum dos quatro, fica em track A. Promover prematuro adiciona overhead de DAG load + crash recovery sem ganho.

---

## 3. Schema (`todos`)

Tabela canônica em `AUDIT.md §1` (herda de `CONTRACTS.md §2.6.8 B.1`):

```sql
CREATE TABLE todos (
  id INTEGER PRIMARY KEY,
  session_id TEXT NOT NULL,
  message_id TEXT NOT NULL,        -- mensagem do modelo onde o todo foi extraído
  step_index INTEGER NOT NULL,     -- step da sessão
  content TEXT NOT NULL,           -- texto do item (≤ 200 chars, ver §4.3)
  status TEXT NOT NULL,            -- 'pending' | 'in_progress' | 'completed' | 'cancelled' | 'blocked'
  parent_todo_id INTEGER,          -- subtask; FK soft a todos.id
  dag_node_id TEXT,                -- preenchido se origem é DAG (track B); FK lógica a ORCHESTRATION.md §2.2
  subagent_session_id TEXT,        -- preenchido se item executou em subagent; FK lógica a sessions.id
  budget_usd REAL,                 -- opcional; herdado do pai/sessão se NULL
  created_at INTEGER NOT NULL,
  started_at INTEGER,
  completed_at INTEGER,
  cancelled_reason TEXT            -- 'superseded' | 'user_abort' | 'budget_exceeded' | 'parse_drift'
);

CREATE INDEX idx_todos_session ON todos(session_id, status);
CREATE INDEX idx_todos_parent ON todos(parent_todo_id);
```

### 3.1 Status transitions

```
pending ─┬─► in_progress ─┬─► completed
         │                ├─► blocked ─► in_progress | cancelled
         │                └─► cancelled
         └─► cancelled
```

- `in_progress`: harness seta quando primeira tool call associada ao item começa (heurística: próxima tool call após anúncio do item).
- `blocked`: modelo declara explicitamente no stream (`- [-] item (blocked: razão)`); transição sai com novo step que tira o `[-]`.
- `cancelled`: modelo declara superseded, OU budget excedido, OU `parse_drift` (modelo mudou texto do item entre messages).

### 3.2 Invariantes

- `started_at IS NOT NULL` ⇒ `status IN ('in_progress', 'completed', 'cancelled', 'blocked')`.
- `completed_at IS NOT NULL` ⇒ `status = 'completed'`.
- `subagent_session_id IS NOT NULL` ⇒ `dag_node_id IS NOT NULL` (subagent só via DAG).
- Ciclo proibido em `parent_todo_id` (validado em insert).
- Todo com `status != 'completed'` ao final da sessão marca sessão como `incomplete` (visível em `RECAP.md §1` pre-compact).

---

## 4. Track A — checklist inline

### 4.1 Formato canônico no stream

Documentado em `CONTEXT_TUNING.md` como markdown checklist; reproduzido aqui pra referência:

```markdown
**Plan:**
- [ ] First item — short imperative
- [ ] Second item
  - [ ] Subtask (indented 2 spaces)
- [-] Blocked item (waiting on X)
- [x] Done item
```

Regras do parser:

- Fence começa com bullet `- [ ]`, `- [x]`, `- [-]`, ou `- [~]` (cancelled).
- Indentação 2-spaces ⇒ subtask. Profundidade máxima: 3 níveis.
- Texto após `—` ou `:` é descrição auxiliar (não indexada).
- Re-emissão de plano no mesmo step com **mesma ordem + textos similares** (Levenshtein ≤ 10%) ⇒ update, não insert.
- Texto divergente ⇒ item antigo vira `cancelled_reason = 'parse_drift'`, novo entra como `pending`.

### 4.2 Pipeline de extração

```
modelo stream ─► token buffer ─► markdown lexer ─► checklist detector
                                                          │
                              ┌───────────────────────────┤
                              ▼                           ▼
                       diff vs todos da sessão     emit failure_event
                       (insert/update/cancel)      (parse falhou)
                              │
                              ▼
                       INSERT/UPDATE em todos
                              │
                              ▼
                       audit_timeline event
                       (todo_added | todo_status_changed)
                              │
                              ▼
                       UI tail (TodoChecklistPane)
```

Parser é **TS puro determinístico**, sem LLM. Mesmo input ⇒ mesma diff.

### 4.3 Limites

| Limite | Valor | Razão |
|---|---|---|
| `content.length` | ≤ 200 chars | UI renderiza em uma linha; mais que isso é descrição de design, não item |
| Itens por sessão | ≤ 100 ativos | Acima disso, modelo está usando checklist como scratch-pad |
| Profundidade | 3 níveis | Subtask de subtask vira árvore que ninguém lê |
| Re-emissão por step | 1 plano completo | Múltiplos planos no mesmo step = drift; parser pega primeiro, ignora resto, emite `failure_event` |

### 4.4 Heurística de `in_progress`

Harness seta `in_progress` automaticamente quando:

- Próxima tool call após o anúncio do plano cita o texto do item (substring match, case-insensitive).
- OU: o modelo cita o item explicitamente no thinking/prosa antes da tool call ("vou começar pelo item 2").

Sem match, item fica `pending`. Sem isso, status diverge da realidade — pior que não ter status.

---

## 5. Track B — DAG + subagents

Track B reusa primitivas de `ORCHESTRATION.md §2` (DAG) e §3 (subagent spawn). Este subsistema **não duplica** lógica de execução; só registra metadata em `todos`.

### 5.1 Mapeamento DAG node → todo

Ao carregar um DAG (`ORCHESTRATION.md §2.1`), executor cria 1 row em `todos` por nó:

```
dag_node_id      ← node.id do DAG
content          ← node.description (truncada a 200 chars)
parent_todo_id   ← todo do nó upstream (se houver 1; NULL se múltiplos)
budget_usd       ← node.budget.max_cost_usd
status           ← 'pending'
```

Transições:

- Nó entra em execução ⇒ `started_at = now()`, `status = 'in_progress'`.
- Nó spawna subagent ⇒ `subagent_session_id = subagent.session_id`.
- Nó termina ⇒ `status = 'completed' | 'cancelled'`, `completed_at = now()`.
- DAG aborta (`on_node_failure: abort_dag`) ⇒ todos os nós `pending`/`in_progress` viram `cancelled` com `cancelled_reason = 'dag_aborted'`.

### 5.2 Por que não promover `task_*` a tool de checklist

Tentação: já que `task_async` spawna subagent, por que não fazer `task_async` automaticamente criar todo?

Razão pra **não** acoplar:

- Subagent pode rodar fora de DAG (`task_sync` em step loop normal). Forçar criação de todo pra cada subagent polui sessões simples.
- Granularidade diferente: um todo pode ser "explorar autenticação" e disparar 3 `task_async` paralelos internamente. 1 todo, 3 subagents.

Acoplamento fica em `ORCHESTRATION.md §2` (DAG executor escreve em `todos`); `task_*` standalone só registra em `tool_calls` como qualquer tool.

---

## 6. Audit events

Adicionados a `audit_timeline` (`AUDIT.md §2.1`):

| Event | Quando | Payload |
|---|---|---|
| `todo_added` | INSERT em `todos` | `{ todo_id, content, parent_todo_id, source: 'stream' \| 'dag' }` |
| `todo_status_changed` | UPDATE status | `{ todo_id, from, to, reason? }` |
| `todo_parse_failure` | parser rejeita | `{ message_id, raw_excerpt, reason }` |
| `todo_drift_detected` | re-emissão divergente | `{ old_todo_id, new_todo_id, levenshtein_pct }` |

Eventos são forenses, não narrativos — `RECAP.md` projeta narrativa em cima deles.

---

## 7. UI

Owner: `UI.md §2.6` (`<TodoChecklistPane>`). Resumo do contrato deste lado:

- Pane lê de `todos WHERE session_id = current AND status != 'cancelled'` ordenado por `created_at`.
- Atualiza via SQLite tail (`UI.md §3.1` event stream), não via re-render do modelo.
- Itens `in_progress` recebem spinner; `blocked` recebem badge com razão; `completed` recebem strikethrough.
- Subtasks colapsáveis (3 níveis máximo).
- Click em item filtra `audit_timeline` por `todo_id` (drill-down — quais tool calls aconteceram dentro desse item).

UI **nunca** edita `todos` diretamente. Único caminho de escrita: parser do stream ou DAG executor.

---

## 8. Failure modes

| Modo | Causa | Resposta do harness |
|---|---|---|
| `todo.parse_failure` | bullet malformado, indentação inconsistente | `failure_event`; UI mostra warning; modelo notificado no próximo turn ("plano não foi parseado, reformat") |
| `todo.drift` | re-emissão com textos muito divergentes | item antigo `cancelled` com `parse_drift`; novo inserido; `audit_timeline` registra |
| `todo.orphaned` | item `in_progress` sem tool call associada por > N steps | move pra `blocked` com `cancelled_reason = 'no_progress'`; modelo é notificado |
| `todo.budget_exceeded` | item track B excedeu `budget_usd` | DAG executor aborta nó; todo vira `cancelled` |
| `todo.dag_aborted` | DAG global abortou | cascata: todos `pending`/`in_progress` viram `cancelled` |
| `todo.cycle_detected` | `parent_todo_id` formaria ciclo | INSERT rejeitado; `failure_event` |

---

## 9. Gatilhos de reconsideração (promover a tool)

Posição atual (rejected como tool — herdada de `CONTRACTS.md §2.6.8 B`) revisitada se:

1. **Stream parser falha em > 5% das sessões.** Medido via `todo.parse_failure / sessions`. Sinal de que modelos não conseguem manter formato; tool com schema rígido reduziria a 0%.
2. **Modelos locais** (`LOCAL_MODELS.md`) emitem formato instável demais. Nesse caso, `todo_write` vira tool **apenas no profile `orchestrated`** — DAG já é meta-cognição harness-side, simetria preservada.
3. **Workflows multi-step >= 20 itens** viram comuns e checklist em prosa começa a fragmentar entre messages (parser não consegue casar continuação). Resposta possível: tool com `append: true` em vez de re-emissão.

Sem nenhum dos três, fica como UI affordance + parser. Decisão revisada a cada release.

---

## 10. Anti-patterns

- ❌ **Forçar checklist em toda sessão.** "Fix typo em README" não precisa de plan. Spec **não** injeta system prompt obrigando planning.
- ❌ **Modelo gerenciando próprio plano via tool call.** Plano emerge na prosa; tool gasta budget sem ganho de qualidade (`CONTRACTS.md §2.6.8`).
- ❌ **UI editando `todos` direto.** Single source of write: parser/DAG. UI write criaria divergência stream ↔ tabela.
- ❌ **Promover todo a memory automaticamente.** Sessão-scoped por default; promoção pra memory é decisão consciente do user (`MEMORY.md`).
- ❌ **Cancelar silenciosamente em parse drift.** Sempre emite `todo_drift_detected`; modelo precisa saber que mudou de ideia.
- ❌ **Acoplar `task_async` a criação automática de todo.** Granularidades diferentes (§5.2).

---

## 11. Cross-refs

- `CONTRACTS.md §2.6.8 B` — decisão original de rejeitar `todo_write` como tool.
- `AUDIT.md §1` — tabela `todos` (canônica).
- `ORCHESTRATION.md §2` — DAG execution (track B).
- `ORCHESTRATION.md §3` — subagent spawn semantics.
- `UI.md §2.6` — `<TodoChecklistPane>`.
- `CONTEXT_TUNING.md` — formato canônico do checklist no stream.
- `RECAP.md §1` — `incomplete: true` quando todos pending ao fim.
- `STATE_MACHINE.md` — step loop que dispara o parser.
- `FAILURE_MODES.md` — registro dos modos de §8.
- `LOCAL_MODELS.md` — gatilho de reconsideração #2.
