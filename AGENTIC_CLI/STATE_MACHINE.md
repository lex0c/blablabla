# STATE_MACHINE

Máquinas de estado formais do `AGENTIC_CLI`. Estados explícitos, transições válidas, invariantes em cada estado, semântica de recovery após crash.

Sem máquina de estado formal, "running" e "paused" e "error" viram conceitos vagos que cada implementador interpreta diferente. Estado formal é a base de **resume confiável**, **debug determinístico**, e **eval reproducível**.

---

## 0. Princípios

1. **Estado é dado, não memória in-process.** Cada transição é persistida no SQLite **antes** da ação correspondente.
2. **Invariantes são checadas em entrada e saída** de cada estado. Violação = bug fatal, não graceful degradation.
3. **Crash em qualquer ponto deve permitir resume** com perda mínima e bem-definida.
4. **Estados terminais não voltam.** `done`, `exhausted`, `error_fatal` nunca transicionam de novo.
5. **Transições são as únicas válidas declaradas.** Tentar uma transição não declarada = bug.
6. **Toda transição emite span OTEL** com estado origem, destino, gatilho.

---

## 1. Convenção

```
[STATE]                         estado
 ↓ <trigger>                    transição
[STATE']                        estado destino

invariantes em [STATE]:
  - <coisa que é sempre verdade enquanto em STATE>

ações ao entrar em [STATE]:
  - <side effect determinístico>

ações ao sair de [STATE]:
  - <side effect>
```

Estados terminais marcados com `*`.

---

## 2. Session State Machine

### 2.1 Diagrama

```
                        ┌──────────────┐
                        │ initializing │
                        └──────┬───────┘
                               │ trust + hooks_session_start ok
                               ▼
                        ┌──────────────┐
            ┌───────────│    idle      │◀──────────┐
            │           └──────┬───────┘           │
            │                  │ user_prompt       │
            │                  ▼                   │
            │           ┌──────────────┐           │
            │           │   running    │           │
            │           └──────┬───────┘           │
            │                  │                   │
            │     ┌────────────┼─────────────┐     │
            │     │            │             │     │
            │     ▼            ▼             ▼     │
            │  ┌────────┐  ┌────────┐    ┌──────┐ │
            │  │awaiting│  │compact-│    │tool_ │ │
            │  │ _user  │  │  ing   │    │exec  │ │
            │  └───┬────┘  └───┬────┘    └──┬───┘ │
            │      │           │            │     │
            │      └───────────┴────────────┘     │
            │                  │                   │
            │   ┌──────────────┼──────────────┐   │
            │   │              │              │   │
            │   ▼              │              ▼   │
            │ ┌─────────┐      │        ┌────────┐│
            │ │paused   │──────┘        │interrup││
            │ └────┬────┘                │ ted    ││
            │      │                     └──┬─────┘│
            │      ▼                        │      │
            │  done back to idle            │      │
            │                               ▼      │
            │                          back to idle│
            │                                      │
            └──────────────────────────────────────┘

      Terminal:
            ┌─────┐  ┌──────────┐  ┌───────────┐
            │done*│  │exhausted*│  │error_fatal│*
            └─────┘  └──────────┘  └───────────┘
```

**Estados transientes não ilustrados** (mesma classe de `awaiting_user`/`compacting` — pausam o loop e retomam):
- `regrounding` — disparado por drift detector antes de side-effect (`§2.2`, `§11`); transita pra `tool_exec` (confirma), `running` (re-plan), ou `awaiting_user` (edit goal).
- `clarifying` — disparado por `clarify()` com `blast_radius: high` ou batched em `pre_execution` mode (`§2.2`, `§12`); transita pra `running` (resolved/skipped) ou `awaiting_user` (escalated → goal edit).

Diagrama ASCII evita inflar branches; estados transientes são definidos em §2.2 com transições completas em §9.

### 2.2 Estados e invariantes

#### `[initializing]`

Sessão sendo aberta. Carregando config, verificando trust, rodando hooks `SessionStart`.

**Invariantes:**
- `sessions.status = 'initializing'`
- Nenhum step persistido ainda
- `cwd` lockado (lockfile em `.agent/lock`)

**Ações ao entrar:**
- INSERT em `sessions` com status='initializing'
- Avalia trust prompt (se aplicável)
- Carrega permissions, hooks, memory index
- Dispara hooks `SessionStart`

**Transições:**
- `→ idle` quando init OK
- `→ error_fatal*` em falha de carga (config inválida, trust negado, hook block)

#### `[idle]`

Aguardando input do usuário (prompt) ou comando (slash).

**Invariantes:**
- `sessions.status = 'idle'`
- Nenhum step em andamento
- Nenhum tool call pending
- Histórico em SQLite é coerente (último message tem role correto pra próximo input)

**Transições:**
- `→ running` em `user_prompt` (input enviado)
- `→ initializing` em `/clear` (re-entra config)
- `→ done*` em `Ctrl+D` ou `/exit`
- `→ paused` em `/pause` (raro)

#### `[running]`

Loop do agente ativo. Modelo gerando ou step em construção.

**Invariantes:**
- `sessions.status = 'running'`
- Existe step ativo em `messages` com `role='assistant'` e ainda sem `output_complete`
- Budget não esgotado (`steps_used < max_steps`, `cost < max_cost`)

**Sub-estados internos** (não persistidos como state distinto, são fases do step):
- `generating` — modelo streaming tokens
- `decoding_tool_use` — modelo emitiu tool_use, aguardando args completos
- `validating_tool_args` — schema check antes de invocar

**Transições:**
- `→ tool_exec` quando modelo emite tool_use válido
- `→ awaiting_user` quando tool requer confirmação
- `→ compacting` quando context > 70%
- `→ idle` quando modelo emite `stop` sem tool_use
- `→ interrupted` em Ctrl+C
- `→ exhausted*` em budget hit
- `→ error_fatal*` em provider error não-recoverable

#### `[tool_exec]`

Tool sendo executada (sub-estado de running, mas com semântica distinta para crash recovery).

**Invariantes:**
- `tool_calls.status = 'running'` para a tool corrente
- Checkpoint criado **se** tool tem `writes: true` (verificado por presence de checkpoint_id no step)
- AbortSignal armado

**Ações ao entrar:**
- Hook `PreToolUse` (bloqueável)
- Permission check
- Checkpoint snapshot se `writes: true`
- INSERT em `tool_calls` com status='running'

**Ações ao sair:**
- UPDATE `tool_calls` com status final + duration_ms + output
- Hook `PostToolUse` (não-bloqueável)

**Transições:**
- `→ running` em sucesso (tool result vai pro contexto)
- `→ running` em erro recoverable (modelo decide retry)
- `→ awaiting_user` se permission engine forçou confirmação mid-execution
- `→ interrupted` em Ctrl+C
- `→ error_fatal*` em erro de checkpoint failure (não pode prosseguir sem snapshot)

#### `[compacting]`

Compaction em progresso (chamada LLM separada).

**Invariantes:**
- `sessions.status = 'compacting'`
- Step do usuário não foi consumido ainda — fica preservado para retomar após
- Hook `PreCompact` já passou (não bloqueou)

**Transições:**
- `→ running` ao finalizar compaction (contexto compactado, retoma loop)
- `→ running` via **fallback determinístico** se LLM falhar 2× ou exceder budget (`ORCHESTRATION.md §4.6`); contexto degradado mas sessão prossegue
- `→ error_fatal*` apenas se `PreCompact` hook retornou `hard_cancel: true` (`ORCHESTRATION.md §5.1.1` — opt-in explícito)
- Compaction **não é cancelável** por Ctrl+C — Ctrl+C agenda interrupt para depois

#### `[awaiting_user]`

Aguardando humano em modal (permission, trust, memory write, plan approval).

**Invariantes:**
- `sessions.status = 'awaiting_user'`
- Existe entry em `pending_decisions` com tipo e payload
- Modelo **não está gerando** (cache de stream pausado)

**Transições:**
- `→ running` ou `→ tool_exec` quando user decide (allow/confirm/edit)
- `→ idle` quando user rejeita (cancela operação corrente; histórico fica intacto)
- `→ interrupted` em Ctrl+C (timeout do prompt = 5min, configurável)

#### `[regrounding]`

Estado transiente disparado por drift detector (`§11`) antes de tool com side-effect. Tool call proposta **não foi invocada**; user/modelo decide se prossegue, re-planeja, ou edita goal.

**Invariantes:**
- `sessions.status = 'regrounding'`
- `pending_decisions` tem entry `kind='regrounding_prompt'` com `payload = drift_check_result`
- Tool call proposta preservada em buffer (não persistida em `tool_calls` ainda)
- `active_goal` (`§2.3`) imutável durante regrounding (mudança de goal só em `awaiting_user` via opção `(c)`)

**Ações ao entrar:**
- Re-injeta `[regrounding_context]` no próximo turno (formato em `§11.3`)
- UI mostra modal com 3 opções (default: re-plan after 30s)

**Transições:**
- `→ tool_exec` em `regrounding_confirmed` (drift confirmado como falso positivo)
- `→ running` em `regrounding_replan` (modelo gera próximo step do zero)
- `→ awaiting_user` em `regrounding_update_goal` (modal pra editar goal)
- `→ interrupted` em Ctrl+C

#### `[clarifying]`

Estado transiente disparado por `clarify()` (`§12`) com `blast_radius: high`, ou batched em playbook com `clarify_mode: pre_execution`.

**Invariantes:**
- `sessions.status = 'clarifying'`
- `pending_decisions` tem entry `kind='clarification_prompt'` (timeout 60s — override de §8)
- Tool `clarify` em flight; outras tool calls **não invocadas** durante este estado
- `active_goal` (`§2.3`) imutável durante clarification

**Ações ao entrar:**
- UI mostra modal com até 3 perguntas batched (uma per `clarify()` pendente)
- Default em 60s = `(skip and proceed with auto_assumed[0])` registrado em `assumptions[]`

**Transições:**
- `→ running` em `clarification_resolved` (user respondeu)
- `→ running` em `clarification_skipped` (timeout / user pulou; auto-choice em `assumptions[]`)
- `→ awaiting_user` em `clarification_escalated` (user indicou que o goal está errado — não é só ambiguidade)
- `→ interrupted` em Ctrl+C

Detalhamento completo em `§12`.

#### `[paused]`

Sessão pausada deliberadamente. Não consome budget. Persiste indefinidamente.

**Invariantes:**
- `sessions.status = 'paused'`
- Nenhuma operação ativa
- Pode ser resumida com `/resume <id>`

**Transições:**
- `→ idle` em `/resume`
- `→ done*` em `/exit`

#### `[interrupted]`

Estado transiente — usuário cancelou e agente está em cleanup (matando subprocess, salvando estado, etc).

**Invariantes:**
- `sessions.status = 'interrupted'`
- Cleanup em progresso

**Ações ao entrar:**
- AbortSignal propaga para tools, subagents, hooks em execução
- Aguarda 5s de grace
- SIGKILL no que ainda estiver rodando

**Transições:**
- `→ idle` quando cleanup termina (sessão pode continuar)
- `→ done*` se user pediu exit ao interromper

#### Estados terminais

- `[done*]` — fim normal. `sessions.ended_at` setado.
- `[exhausted*]` — budget atingido. Estado preservado para review/resume com novo budget.
- `[error_fatal*]` — erro irrecoverable. Logs preservados; sessão não é resumível com mesmo state.

### 2.3 Goal stack

> **Cross-refs:** drift detector consome `active_goal` em §11; auto-rehydrate em §7.6; multi-goal sessions em `CONTEXT_TUNING.md §10.4`; recap em `RECAP.md §3`.

`goal.text` (`CONTEXT_TUNING.md §10`) modela **um** goal por sessão. Sessão real frequentemente tem aninhamento: "refactor → adicionar testes → atualizar docs → voltar pro refactor". Tratar tudo como string monolítica obriga o modelo a inferir qual sub-objetivo está ativo, e drift detector (§11) não tem o que comparar.

Goal stack formaliza aninhamento: **stack ordenada de goals**, com push/pop registrados, sempre com `decided_by`.

#### 2.3.1 Schema

```sql
goal_stack(
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  parent_id TEXT,                      -- NULL = root goal; senão referência ao goal pai
  text TEXT NOT NULL,                  -- literal, ≤ 1000 chars
  status TEXT NOT NULL,                -- active | suspended | done | abandoned
  pushed_at INTEGER NOT NULL,
  popped_at INTEGER,
  pushed_by TEXT NOT NULL,             -- user | model_proposed_user_approved | playbook
  decided_by TEXT,                     -- ao pop: user | model | playbook | timeout
  pop_reason TEXT,                     -- completion | abandoned | superseded | error
  source_step_id TEXT
);
```

**Invariantes:**

- Sessão tem **exatamente um** goal com `status='active'` em qualquer instante (`active_goal`).
- Goal raiz (`parent_id IS NULL`) é o primeiro user prompt da sessão; criado em `initializing`.
- `pop` requer `decided_by` não-null — sem isso é bug fatal (registro de auditoria incompleto).
- Goal `done` ou `abandoned` é terminal; não pode reativar (cria-se novo goal idêntico se necessário, com referência via `pop_reason='superseded'`).
- Profundidade máxima da stack: **5**. Push em depth 6 → erro com sugestão de fechar/colapsar antes.

#### 2.3.2 Operações

```
push_goal(text, pushed_by, parent_id?)
  → INSERT goal com status='active'
  → UPDATE goal pai (se houver): status='suspended'
  → emit span goal.pushed

pop_goal(decided_by, pop_reason)
  → UPDATE goal ativo: status=<done|abandoned>, popped_at=<now>
  → UPDATE goal pai (se houver): status='active'
  → emit span goal.popped

suspend_goal(reason)
  → raro; usado quando user pausa sub-task explicitamente
  → goal pai NÃO é reativado (stack permanece com gap; user resolve depois)
```

API de slash commands:

```
/goal push "<text>"             # push manual
/goal pop [reason]              # pop manual com reason livre
/goal list                      # mostra stack atual
/goal abandon                   # pop com reason=abandoned (UI confirma)
```

Modelo propõe push/pop via tool `goal_push(text)` / `goal_pop(reason)` — vai pra `awaiting_user` para confirmação modal (idêntico a memory writes).

#### 2.3.3 Como `active_goal` é injetado

Substitui `goal.text` em todos os pontos de injeção:

- **Goal re-injection** (`CONTEXT_TUNING.md §10`):
  ```
  Goal (active): <active_goal.text>
  Goal stack:
    - [done]    refactor queue retry logic
    > [active]  add 5 tests for retry edge cases       ← active marcado
    - [pending] update docs
  ```
- **Auto-rehydrate** (§7.6): `active_goal.text` em vez de `goal.text`; stack completa em bloco separado.
- **Drift detector** (§11): compara `next_action` contra `active_goal.text`, não goal raiz.

#### 2.3.4 Interação com playbooks

Playbook (`PLAYBOOKS.md`) que tem fases declaradas pode emitir push/pop automaticamente:

```yaml
playbook: refactor
phases:
  - name: extract-pure-function
    on_enter: goal_push("extract pure function preserving semantics")
    on_complete: goal_pop("completion")
  - name: add-tests
    on_enter: goal_push("add tests for extracted function")
    on_complete: goal_pop("completion")
```

Sem isso, push/pop é manual (user/model). Auto via playbook é opt-in.

#### 2.3.5 Recap

`RECAP.md §3` é estendido:

```yaml
recap_intermediate:
  goal:
    text: string                       # active_goal.text no fim da sessão (ou último popped se sessão terminou)
    source_step_id: string
  goal_stack:                          # novo campo
    - { text: string, status: string, decided_by: string, pop_reason: string, duration_ms: int }
```

`/recap` renderiza tree:

```
Goal stack:
  ✓ refactor queue retry logic (4m12s — completion, decided_by: user)
    ✓ extract pure function (2m30s — completion, decided_by: playbook)
    ✓ add 5 tests (1m15s — completion, decided_by: user)
    ✗ update docs (abandoned, decided_by: user — out of scope)
```

#### 2.3.6 Crash recovery

Tabela `goal_stack` é durable (SQLite). Resume preserva stack inteira; `active_goal` é recomputado: o goal mais recente com `status='active'` (sempre exatamente um por invariante).

Se invariante violada no resume (zero ou múltiplos active) → state machine entra em walk-back: marca todos exceto o mais recente como `abandoned` com `pop_reason='resume_recovery'`; loga em `failure_events`.

#### 2.3.7 Anti-patterns

- **Pop sem `decided_by`.** Audit trail incompleto — invariante quebrada.
- **Stack profunda (>3 níveis na prática).** Sinal de que tarefa devia ser sessões separadas. Cap de 5 é defesa contra runaway, não target.
- **Goal raiz mutado.** Goal raiz é imutável após `initializing`. Se user quer mudar tarefa fundamental, cria-se nova sessão (ou push de novo goal raiz com `pop_reason='superseded'` no anterior).
- **Push em playbook sem pop correspondente.** Playbook deve garantir simetria; eval verifica via property-based test (sessão termina com stack vazia exceto raiz).

---

## 3. Step State Machine

Cada step (uma iteração do loop) tem máquina própria.

```
              [pending]
                  ↓ provider_request_started
              [generating]
                  ↓ stop_reason
        ┌─────────┴──────────┐
        ↓                    ↓
   [tool_use_proposed]   [no_tool_use]
        ↓ permission           ↓ critique_optional
   ┌────┴────┐             [persist]
   ↓         ↓                 ↓
[denied]  [allowed]          [done*]
   ↓         ↓
[done*]   [executing]
            ↓ tool_complete
        [observing]
            ↓ post_tool_hook (fire-and-forget)
        [done*]
```

### 3.1 Estados (referência canônica)

| Estado | Persistência | Significado | Pode interromper? |
|---|---|---|---|
| `pending` | row em `messages` com role='assistant', sem content | step alocado, nada gerado ainda | sim (descarta row) |
| `generating` | content acumulando via stream em buffer | modelo emitindo tokens | sim (Ctrl+C: descarta; Esc Esc: preserva visual) |
| `tool_use_proposed` | tool_use parseado em buffer; entry em `tool_calls` ainda não persistida | aguarda permission/hook | sim |
| `denied` | `approvals` row com decision='deny'; tool_result sintético | permission negou; modelo recebe erro estruturado | n/a (já terminal logical) |
| `allowed` | `approvals` row com decision='allow'; checkpoint criado se `writes:true` | aguarda execute | sim (graceful) |
| `executing` | `tool_calls.status='running'` | tool em runtime (com AbortSignal) | sim (graceful 5s + SIGKILL) |
| `tool_complete` | `tool_calls.status='done'` | tool retornou (sucesso ou erro estruturado) | n/a (rapidíssimo) |
| `observing` | tool result na content do step | tool_result no contexto; loop volta a generating se necessário (multi-tool) | sim |
| `no_tool_use` | output completo sem tool_use; aguarda critique opt-in | step quase terminal | sim |
| `critique_running` | critique LLM call ativa (opt-in `critique.mode`) | critic revisando output | sim (cancel critique; persist sem revisão) |
| `persist` | output buffer indo pra `messages` | commit de transação SQLite | não (atomic; ≤ms) |
| `done` * | step completo; row em `messages` final | terminal | n/a |

### 3.2 Transições nomeadas

| Trigger | De → Para | Quem dispara |
|---|---|---|
| `provider_request_started` | pending → generating | Provider Layer |
| `stop_reason` (= 'tool_use') | generating → tool_use_proposed | Stream parser |
| `stop_reason` (= 'end_turn') | generating → no_tool_use | Stream parser |
| `permission_allow` | tool_use_proposed → allowed | Permission Engine |
| `permission_deny` | tool_use_proposed → denied | Permission Engine |
| `pre_tool_hook_block` | tool_use_proposed → denied | Hooks Dispatcher |
| `tool_invoke` | allowed → executing | Tool Registry |
| `tool_complete` | executing → tool_complete | Tool runtime |
| `tool_observed` | tool_complete → observing | Harness |
| `post_tool_hook` | observing → done (fire-and-forget) | Harness |
| `multi_tool_continue` | observing → tool_use_proposed (next tool in same step) | Harness |
| `critique_start` | no_tool_use → critique_running (se opt-in) | Harness |
| `critique_complete` | critique_running → persist (se OK) ou re-generating (se redo) | Critic |
| `commit` | persist → done | SQLite |
| `interrupt` | qualquer → step_interrupted | Ctrl+C / Esc Esc |
| `error_persistent` | qualquer → step_error | Failure handler |

### 3.3 Multi-tool-use loop interno

Step pode ter **múltiplos** tool_use blocks. Comportamento:

```
generating (1ª stop_reason='tool_use')
  → tool_use_proposed (tool A)
  → allowed → executing → observing
  → tool_use_proposed (tool B)              ← se ainda há tool_use blocks
  → allowed → executing → observing
  → ...
  → done (após último tool_result observado)
```

Sequencial por default; paralelo opt-in via `parallel_safe: true` em tool definition (ver `ORCHESTRATION.md` §1.3).

**Invariante:** todos os tool_use+tool_result do step ficam na **mesma transação SQLite**. Crash mid-multi-tool: rollback do step inteiro; resume re-gera.

### 3.3.1 Wait & Monitor sub-states

Step com `wait_for` ou `monitor` ativos: estado canônico continua `executing` (mesmo de tool comum), mas com **sub-state visualizado** em UI:

| Sub-state | Quando | UI render |
|---|---|---|
| `executing.waiting_for` | tool `wait_for` em flight | `<WaitIndicator>` com condition + elapsed + timeout |
| `executing.monitoring` | tool `monitor` em flight | `<MonitorStream>` com events conforme chegam |

**Características:**
- LLM **não é chamado** durante o wait/monitor (zero LLM cost)
- Wall-clock conta pra `maxWallClockMs`
- Cancellable via Ctrl+C / Esc Esc (AbortSignal cascateia ao subsystem do wait)
- Hooks `Notification` podem disparar em eventos de `monitor`
- Step finaliza normalmente em `done` quando wait/monitor retorna

Não há transição de top-level state — `executing` engloba ambos.

### 3.4 Invariantes globais

1. Cada step tem **exatamente um** assistant message persistido (em `done`)
2. Toda tool_use tem **exatamente uma** tool_result correspondente (mesmo se erro)
3. `tokens_in/tokens_out` são finalizados em `done`
4. Custo é finalizado em `done`; thinking + critique cost em spans separados
5. Step em `executing` que crasha vira `tool_call.status='error'` no resume com `details: 'interrupted by crash'`
6. Multi-tool em mesmo step: ordem em `tool_calls` table = ordem em `messages.content`

---

## 4. Tool Call State Machine

```
[pending] → [allowed] → [running] → [done*]
              ↓             ↓
           [denied]     [error*]
              ↓             ↓
           [done*]      [done*]
```

| Estado | Trigger pra próximo |
|---|---|
| `pending` | criado quando model emite tool_use |
| `allowed` | permission engine OK (incluindo confirmação) |
| `denied` | permission engine NEG, ou hook PreToolUse block |
| `running` | execute() iniciado |
| `done` | resultado persistido com sucesso (status final no row) |
| `error` | exception/timeout (ainda persiste row, mas marca status) |

### Invariantes

- `pending` só pode existir por ≤ 30s (timeout pra resolver permission)
- `running` tem `started_at` setado; `done`/`error` tem `ended_at` setado
- `denied` **não tem** `running` no histórico (não executou)
- Crash em `running` → no resume vira `error` com `details: "interrupted by crash"`

---

## 5. DAG Node State Machine (profile `orchestrated`)

```
[waiting_inputs] → [ready] → [executing] → [validating] → [done*]
                                 ↓             ↓
                             [llm_error]  [validation_failed]
                                 ↓             ↓
                             [retrying]   [retrying] (≤2)
                                 ↓             ↓
                             [executing]  [executing]
                                 ↓             ↓
                             [failed*]    [failed*]
```

### Estados

- `waiting_inputs` — node aguarda outputs de `inputs_from`
- `ready` — todos inputs disponíveis, em fila
- `executing` — chamando LLM ou função determinística
- `validating` — validators rodando
- `validation_failed` — pelo menos um validator retornou `ok: false`
- `retrying` — preparando novo prompt com `retry_hint`
- `done` — output válido, persistido em `step_outputs`
- `failed` — esgotou retries; trigger `on_failure` do DAG

### Invariantes

- Cada node executa **no máximo** `max_retries + 1` vezes (default 3)
- Output só vai pra `step_outputs` em `done` (transação)
- `retrying` tem contador `retry_count` persistido (sobrevive crash)
- DAG completo termina quando todos os terminal nodes (sem successors) estão em `done` ou `failed`

---

## 6. Subagent Lifecycle

```
[spawned] → [running] → [terminating] → [output_persisted*]
                ↓             ↓
            [killed]     [crashed]
                ↓             ↓
            [output...   [output_persisted*]
            persisted*]   (status=error)
```

### Estados (em `subagent_outputs`)

- `spawned` — processo iniciado, ainda não escreveu nada
- `running` — heartbeat detectado em `subagent_outputs.last_heartbeat`
- `terminating` — recebeu SIGTERM, em cleanup
- `killed` — SIGKILL aplicado (após grace de 5s)
- `crashed` — processo morreu sem output
- `output_persisted` — terminal; output em `subagent_outputs.payload`

### Invariantes

- Subagent **sempre** termina em `output_persisted*` (mesmo em crash, com `status: error`)
- Worktree (se isolation) é cleanup'd em terminação (graceful ou forçado)
- Pai bloqueia em leitura do output até `output_persisted` ou timeout

---

## 6.5 MCP Server Lifecycle

> **Cross-refs:** contrato em `CONTRACTS.md §11`; threat model em `SECURITY_GUIDELINE.md §3, §5`; failure mode `adversarial.mcp_server.changed_manifest` em `FAILURE_MODES.md §14.2`; spec consolidada em `MCP.md`.

```
[disconnected] → [handshaking] → [trust_pending] → [trusted] → [active]
       ▲                ↓              ↓                          ↓
       │           [error]        [denied]                   [degraded]
       │                                                          ↓
       └──────────────────────────────────────────────────── [disconnected]
                              ▲
                              │
                          (manifest_changed em qualquer estado pós-trusted →
                           força [trust_pending] na próxima sessão)
```

### Estados (em `mcp_servers`)

- `disconnected` — nenhuma conexão; estado inicial e pós-erro
- `handshaking` — transport aberto; `initialize` enviado, aguardando resposta
- `trust_pending` — manifest recebido; trust prompt pendente para o user (decisão modal em `pending_decisions`)
- `trusted` — user aprovou hash do manifest; tools registradas mas server ainda não ativo no turno
- `active` — server pode receber `tools/call`; tools visíveis ao modelo
- `degraded` — schema violation em output ou heartbeat lento; tools ainda visíveis mas chamadas extras com warning
- `denied` — user recusou trust; server fica in-config mas inativo (slash command pra reverter)
- `error` — falha de protocolo terminal (version mismatch, manifest malformado); user é notificado, ação requer fix

### Transições nomeadas

| Evento | De → Para | Side effect |
|---|---|---|
| `mcp.connect_requested` | `disconnected` → `handshaking` | spawn (stdio) ou open (HTTP/SSE) |
| `mcp.initialize_ok` | `handshaking` → `trust_pending` | manifest hash gravado em `mcp_manifest_history` |
| `mcp.initialize_protocol_mismatch` | `handshaking` → `error` | `failure_event` `mcp.protocol.version_mismatch` |
| `mcp.trust_granted` | `trust_pending` → `trusted` | `approvals` row com hash; tools registradas com `visible_to_model: true` |
| `mcp.trust_denied` | `trust_pending` → `denied` | tools registradas mas `visible_to_model: false` |
| `mcp.session_first_call` | `trusted` → `active` | (lazy activation; primeiro `tools/call` da sessão) |
| `mcp.output_invalid` | `active` → `degraded` | warning em audit; modelo recebe `mcp.output.invalid` |
| `mcp.recover_ok` | `degraded` → `active` | N calls válidas consecutivas (default 3) |
| `mcp.transport_error` | `*` → `disconnected` | classifica em `failure_events` por kind |
| `mcp.manifest_changed` | `trusted`/`active`/`degraded` → `trust_pending` | detectado em SessionStart; tools ficam invisíveis até nova aprovação |
| `mcp.user_revoked` | `*` → `denied` | via `/trust mcp <name> revoke` slash command |

### Invariantes

- Server **sempre** começa em `disconnected` ao spawn do agente; trust não é "lembrado entre sessões sem manifest match"
- Trust é **per-manifest-hash**: hash novo = trust novo, mesmo que o `<name>` do server seja igual
- `active` é estado **lazy** — server só vira active quando o modelo efetivamente chama uma tool; evita conexões pré-emptivas custosas
- `degraded` **não** torna tools invisíveis ao modelo; só adiciona warning em audit. Decisão deliberada: degradação intermitente vs invisibilidade abrupta favorece o segundo apenas quando crítico (manifest change, transport break)
- Em `denied`/`error`: tools **nunca** são apresentadas ao modelo no `tool_schemas` cache breakpoint (`CONTEXT_TUNING.md §2`), mesmo que estejam registradas

### Heartbeat e degradation

Não há heartbeat MCP no protocolo. Harness usa **proxy de saúde**:

- **Stdio:** processo vivo (`kill(pid, 0)`); pipe leitura sem `EPIPE`
- **SSE/HTTP:** última resposta < `mcp.heartbeat_max_age` (default 60s)

Falha de proxy → `mcp.transport_error` → `disconnected`.

### Manifest change detection (cross-session)

Em `SessionStart`, para cada server em config:
1. Conecta, envia `initialize`, recebe manifest
2. Computa `hash = SHA256(canonical(manifest))`
3. Compara com último `mcp_manifest_history.hash` para esse `<name>`
4. **Match:** transita direto para `trusted` (skip prompt)
5. **Mismatch:** transita para `trust_pending`, registra `mcp_manifest_history` com novo hash + `previous_hash`, tools ficam invisíveis até user aprovar
6. **Server não responde:** transita para `disconnected`; tools invisíveis; warning em UI

Step 4 é o "skip" justificado: re-prompting a cada sessão pra hash idêntico vira ruído de UX. Step 5 é o defense-in-depth contra `adversarial.mcp_server.changed_manifest` (`FAILURE_MODES.md §14.2`).

### Crash recovery interaction

Tabela do `§7.1` aplicada a MCP:

| Estado pré-crash | Estado após resume | Recovery |
|---|---|---|
| `disconnected` | `disconnected` | nada |
| `handshaking` | `disconnected` | reconnect na próxima invocação |
| `trust_pending` | `trust_pending` | modal re-mostrado (preservado em `pending_decisions`) |
| `trusted`/`active` | `disconnected` | reconnect lazy; manifest hash re-validado (Step 3 acima) |
| `degraded` | `disconnected` | mesmo |
| `denied` | `denied` | preservado entre sessões; ação explícita do user pra reverter |
| `error` | `error` | preservado; `mcp doctor` mostra causa |

Server crash mid-tool-call: cobre `§7.1` linha "tool_exec (durante invocação)" — tool result vira `{ error: "mcp.server.crashed", server }`, modelo decide próxima ação.

### O que **não** garantimos

- **Reentrant calls:** se modelo chama `mcp:foo:bar` e `mcp:foo:baz` em paralelo (parallel_safe tools), server precisa suportar JSON-RPC concorrente — harness não serializa pra ele. Server não-reentrant declara `parallel_safe: false` em manifest extension (campo `_meta.agentic_cli.parallel_safe`)
- **State entre `tools/call`:** server pode manter estado interno; harness não garante isolamento. Se o user precisa de isolamento, é problema do server (ex: spawn worker per call)
- **Server-side persistência:** server pode escrever em FS/DB próprios; harness não captura em `checkpoints`. Tool MCP que escreve algo persistente é **fora do contrato de checkpoint** — `/undo` não cobre

---

## 7. Crash Recovery (resume após SIGKILL/power loss)

### 7.1 Onde a sessão pode estar quando o processo morre

| Estado pré-crash | Estado após resume | Recovery |
|---|---|---|
| `idle` | `idle` | resume direto; nada perdido |
| `running` (gerando) | `running` (re-emite último user prompt) | re-prompt; **eval cobre** que é idempotente |
| `running` (tool args parseando) | `running` (volta antes da tool) | tool_use é descartada, re-gera |
| `tool_exec` (antes de invocar) | `running` (sem tool result) | tool é re-proposta |
| `tool_exec` (durante invocação) | `error` no tool_call; `running` no step | tool result = `{ error: "interrupted" }` ; modelo decide |
| `tool_exec` (após invocar, antes de persistir) | `error` no tool_call (sem garantia de side effect) | mesmo acima; user é avisado se tool tinha `writes: true` mas checkpoint não foi consultado pra rollback |
| `compacting` | `running` (com contexto não-compactado, retry compaction) | compaction é idempotente |
| `awaiting_user` | `awaiting_user` (modal re-mostrado) | decisão pendente preservada em `pending_decisions` |

### 7.2 Detecção de crash

`SessionStart` pra cada `--resume`:
1. Lê `sessions.status` na DB
2. Se status ∈ {`running`, `tool_exec`, `compacting`}: **crash detectado**
3. Procura último estado consistente walk-back nas tabelas
4. Aplica recovery por classe (tabela acima)
5. Atualiza `sessions.status` antes de retomar

### 7.3 Garantias de durabilidade

Antes de declarar transição completa, harness:

1. Escreve em SQLite (com `PRAGMA synchronous = FULL` em prod)
2. Aguarda fsync
3. Só então procede

Custo: ~5ms por step. Ganho: power-loss não corrompe.

### 7.4 O que **não** garantimos no resume

- Side effect de tool com `writes: true` que não chegou a criar checkpoint (race entre snapshot e crash) → user avisado, **não automaticamente revertido**
- Background processes (`bash_background`) — não sobrevivem ao crash do agente; são marcados como `killed` no resume
- Subagent worktree pendurado — `worktree gc` no SessionStart limpa
- Stream parcial de tokens — descartado; modelo re-gera

### 7.5 Visibilidade em listagem (`agent --list-sessions`, `<SessionPicker>`)

Sessões em estados não-terminais (`running`, `tool_exec`, `compacting`, `awaiting_user`) que não foram resumidas aparecem em listagem com **flag visual** distinto:

- `incomplete: true` no `recap_mini` schema (ver `RECAP.md` §3.1)
- Label visual: `⚠ interrupted by crash` ou `⚠ awaiting decision (stale)`
- Diferenciado de status terminais (`done`/`exhausted`/`error_fatal`) em cor + ícone

Permite ao user ver: "essa sessão crashou — vale resumir e recovery, ou descartar?". Sem essa flag, sessão crashada parece igual a `done` em listagem casual e user pode perder trabalho.

**Sessão duplicada em outra instância:** se outro processo `agent` está ativo no mesmo `cwd` (lockfile detecta), sua sessão aparece com label `(active in PID N)` em listings de outras instâncias — evita race de duplo-resume.

### 7.6 Auto-rehydrate no resume

> **Cross-refs:** schema em `RECAP.md §3`; goal re-injection format em `CONTEXT_TUNING.md §10.2-10.3`; pinned context em `CONTEXT_TUNING.md §12.4`.

Resume restaura **estado** da máquina, mas isso não é suficiente: contexto in-memory foi perdido e o modelo do próximo turno começa sem `goal`, sem `decisions[]`, sem `todos`. O efeito prático é que sessão > 1h crashada e retomada **age como sessão nova com histórico opaco** — modelo lê tool calls antigos sem entender por que foram feitos.

Auto-rehydrate ataca isso: ao transitar `* → idle` via `resume_request` (seja crash recovery ou `/resume`), antes do primeiro `user_prompt` consumir o turno, harness injeta literalmente em `[current_turn]`:

```
[resume_context]
Goal (original task): <recap_intermediate.goal.text — literal, byte-by-byte>

Decisions taken before crash:
  - step 7: <decisions[0].what> — <decisions[0].why> (decided_by: <who>)
  - step 12: <decisions[1].what> — <decisions[1].why> (decided_by: <who>)
  - ... (até 5 decisões mais recentes)

Pinned context:
  - <pin_1.text>
  - <pin_2.text>
  - ... (todas, pins não são truncados)

Open todos (last todo_write):
  - [done] extract pure function
  - [in progress] add 5 tests
  - [pending] update docs

Resumed at: <ts>; previous status: <pre_crash_status>; loss_bound: <descrição>.
[/resume_context]
```

**Garantias:**

- **Determinístico.** Projeção é `RecapIntermediate` (`RECAP.md §3`) → texto via template fixo. Sem LLM no caminho crítico de resume (Haiku no resume = latência inaceitável + ponto de falha extra).
- **Idempotente.** Mesmo crash retomado N vezes ⇒ mesmo `[resume_context]` injetado. Resume não acumula camadas de rehydrate.
- **Preservado entre `/clear`.** `[resume_context]` é parte do *primeiro turno após resume*, não do system prompt; `/clear` limpa normalmente.
- **Pinned context é always-include.** Pins (`CONTEXT_TUNING.md §12.4`) são re-injetados independentemente de truncamento de decisions/todos.

**Quando NÃO injetar:**

- Resume de sessão em estado terminal (`done`, `exhausted`, `error_fatal`) — não há próximo turno; user está revisitando histórico.
- Resume de sessão `paused` com < 5 turns desde último user prompt — contexto in-memory não foi perdido (paused preserva), rehydrate vira ruído.
- Sessão sem `RecapIntermediate` cacheado e sem hook `Stop` ter rodado — fallback: injeta apenas `goal.text` + `Resumed at: <ts>`, marca `incomplete: true`.

**Truncation budget:** `[resume_context]` cap em 2k tokens. Se decisions + todos + pins excedem, decisions são head+tail-truncadas (preserva primeiras 2 + últimas 2; meio elide com `... N decisions elided ...`). Pins **nunca** são truncados — se pins sozinhos > 2k, é bug do user (alertar via warning).

**Visibilidade:** UI mostra `🔄 Resumed from <pre_crash_status> — N decisions, M pins, K todos rehydrated` no primeiro turno após resume. User vê o que foi recarregado e pode fazer `/recap` para inspecionar.

**Eval acoplado:** `evals/resume/auto_rehydrate/` — fixture com sessão multi-decisão crashada em `tool_exec`; resume sem rehydrate falha 60%+ das tarefas que dependem de decisão pré-crash; com rehydrate falha < 5%. Threshold é PR-bloqueante.

---

## 8. Estado de pending decisions

Tabela auxiliar para `awaiting_user`:

```sql
pending_decisions(
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  kind TEXT NOT NULL,        -- permission | trust | memory_write | plan_approval
  payload JSONB NOT NULL,
  created_at INTEGER NOT NULL,
  expires_at INTEGER,        -- 5min default
  resolved_at INTEGER,
  decision JSONB
);
```

**Invariantes:**
- Sessão em `awaiting_user` tem **exatamente uma** row com `resolved_at IS NULL`
- Decisão expirada (`now > expires_at`) é tratada como `denied` no resume
- Decisão resolvida não é re-mostrada (idempotência de resume)

---

## 9. Eventos de transição (lista completa)

Todo subsistema dispara um destes eventos quando força transição. Telemetria emite span por evento.

| Evento | De → Para | Quem dispara |
|---|---|---|
| `session_init_ok` | initializing → idle | Session Manager |
| `session_init_failed` | initializing → error_fatal | Session Manager |
| `user_prompt` | idle → running | UI |
| `model_done` | running → idle | Provider stream |
| `model_tool_use` | running → tool_exec (via permission check) | Provider stream |
| `permission_allow` | running → tool_exec | Permission Engine |
| `permission_deny` | running → running (tool_call=denied) | Permission Engine |
| `permission_confirm_required` | running → awaiting_user | Permission Engine |
| `user_decision` | awaiting_user → running/tool_exec/idle | UI |
| `tool_done` | tool_exec → running | Tool runtime |
| `tool_error` | tool_exec → running | Tool runtime |
| `tool_writes_blocked` | tool_exec → error_fatal | Checkpoint Manager (snapshot fail) |
| `compact_trigger` | running → compacting | Context Engine |
| `compact_done` | compacting → running | Compaction LLM |
| `compact_fallback_used` | compacting → running | Compaction Engine (LLM failed 2× / budget exceeded — `ORCHESTRATION.md §4.6`) |
| `compact_hard_cancelled` | compacting → error_fatal | PreCompact hook (`hard_cancel: true` — `ORCHESTRATION.md §5.1.1`) |
| `drift_detected` | running → regrounding | Drift Detector (§11) |
| `regrounding_confirmed` | regrounding → tool_exec | UI (user/model confirmed action despite drift signal) |
| `regrounding_replan` | regrounding → running | UI (re-plan from scratch) |
| `regrounding_update_goal` | regrounding → awaiting_user | UI (goal edit modal) |
| `goal_pushed` | (sem mudança de session state) | Goal Stack (`§2.3`) — user / model_proposed / playbook |
| `goal_popped` | (sem mudança de session state) | Goal Stack (`§2.3`) — requer `decided_by` + `pop_reason` |
| `goal_suspended` | (sem mudança de session state) | Goal Stack (`§2.3`) — pausa explícita de sub-task |
| `clarification_requested` | running → clarifying | Modelo emite `clarify(...)` com `blast_radius: high` (`§12`) |
| `clarification_resolved` | clarifying → running | UI (user respondeu; tool `clarify` retorna escolha) |
| `clarification_skipped` | clarifying → running | UI/timeout (user pulou; assumptions[] registra auto-choice) |
| `clarification_escalated` | clarifying → awaiting_user | UI (user indicou que goal está errado, não só ambíguo) |
| `clarification_auto_low` | (sem mudança de session state) | Tool `clarify` com `blast_radius: low` — registra em `assumptions[]` sem modal |
| `interrupt_signal` | * → interrupted | UI (Ctrl+C) |
| `interrupt_complete` | interrupted → idle/done | Cleanup |
| `budget_exhausted` | running/tool_exec → exhausted | Budget tracker |
| `provider_unavailable` | running → error_fatal | Provider (after retries) |
| `pause_request` | idle/running → paused | UI (`/pause`) |
| `resume_request` | paused → idle | UI (`/resume`) |
| `exit_request` | idle/paused → done | UI (Ctrl+D, `/exit`) |

---

## 10. Como verificar a máquina de estado

### 10.1 Static check

Eval estático no CI:
- Toda transição em código tem entry na tabela acima
- Toda entrada na tabela tem teste correspondente
- Estados terminais nunca transicionam (assertion em runtime + teste)

### 10.2 Property-based test

Em CI, gera sequências aleatórias de eventos válidos; verifica:
- Sessão sempre termina em estado terminal ou retorna a `idle`
- Nenhum estado intermediário tem `ended_at` setado prematuramente
- Total de tokens / custo é monotônico crescente
- Cada `tool_use` tem `tool_result` correspondente

### 10.3 Crash recovery test

Eval específico que mata o processo em pontos aleatórios:
- Em `generating` mid-stream
- Em `tool_exec` antes de invoke
- Em `tool_exec` durante invoke (matando o subprocess da tool)
- Em `compacting`

Resume tem que retornar a estado **consistente** (todas invariantes satisfeitas).

---

## 11. Drift detection (foco invariant)

> **Cross-refs:** goal re-injection em `CONTEXT_TUNING.md §10`; pinned context em `CONTEXT_TUNING.md §12.4`; eval-driven thresholds e config tunáveis em `TOKEN_TUNING.md §13.5`.

Re-injetar `goal.text` literal (`CONTEXT_TUNING.md §10`) garante que o **goal está no prompt** — não garante que o **modelo está perseguindo o goal**. Em sessão > 30 turns, modelo pode estar focado em sub-problema tangencial enquanto a tarefa original ficou meio resolvida 15 turns atrás. Isso é *drift de foco*, e é invisível por construção: tool calls parecem coerentes localmente.

Drift detection é uma checagem barata, **antes de ações com side-effect**, comparando intent declarado da próxima ação contra o goal ativo.

### 11.1 Quando dispara

Trigger: transição `running → tool_exec` para tool com `writes: true` **ou** com `network_egress: true`. Não dispara para tools read-only (custo > benefício para `read_file`/`grep`).

Skip:
- Sessão < 10 turns (não há histórico para drift)
- Tool faz parte de `parallel_safe` batch já validado no batch-anchor
- `drift_detection.enabled = false` em config (default `true` em `autonomous`/`hybrid`; `false` em `orchestrated` — DAG já tem validators próprios)

### 11.2 Mecânica

Antes de entrar em `tool_exec`:

```
1. Construir DriftCheck input (≤ 400 tokens):
   - goal.text (literal)
   - pinned_context[] (literal, sempre)
   - last_decision.what (mais recente em decisions[])
   - next_action: { tool: <name>, args_summary: <derived>, intent: <model self-declared> }
2. Chamar Haiku com prompt versionado prompts/drift/v1.j2:
   "Given goal G, is action A advancing G, exploring G, or unrelated to G?
    Output JSON: { alignment: aligned|exploring|drifted, confidence: 0..1, reason: string }"
3. Validar schema; se inválido, default a aligned (fail-open — drift detector nunca bloqueia falsamente).
4. Decisão:
   - aligned (confidence ≥ 0.7) → continua para tool_exec (caminho normal)
   - exploring (qualquer confidence) → continua, mas registra drift_event(kind="exploring")
   - drifted (confidence ≥ 0.7) → transição → regrounding (§11.3)
   - drifted (confidence < 0.7) → continua com warning visível
```

**Custo:** ~300 tokens input + ~50 tokens output via Haiku ≈ **$0.0003 por check**. Em sessão de 50 turns com 20 tool_exec qualificados, custo total ~ $0.006 — uma ordem de magnitude abaixo de uma compaction.

**Latência:** ≤ 400ms p95. Colocado **em paralelo** com hook `PreToolUse` quando possível (não bloqueia hook chain). Bloqueia entrada em `tool_exec` apenas se `drifted` com alta confiança.

#### 11.2.1 Schema `drift_events`

```sql
drift_events(
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  step_id TEXT NOT NULL,
  proposed_tool TEXT NOT NULL,
  proposed_intent TEXT,
  active_goal_id TEXT NOT NULL,        -- referência a goal_stack
  alignment TEXT NOT NULL,             -- aligned | exploring | drifted
  confidence REAL NOT NULL,            -- 0.0 - 1.0
  reason TEXT,                         -- detector_reason
  outcome TEXT NOT NULL,               -- proceeded | regrounded | warned | skipped
  created_at INTEGER NOT NULL,
  detector_cost_usd REAL,
  detector_latency_ms INTEGER
);
```

**Toda chamada do detector** insere uma row, independente do outcome (incluindo `aligned` e `skipped` por fail-open). Permite:
- Eval offline com replay (§11.5)
- Calibração de thresholds via análise de histórico
- `/recap forensics` mostra drift events da sessão

### 11.3 Estado `[regrounding]`

Novo estado transiente:

**Invariantes:**
- `sessions.status = 'regrounding'`
- `pending_decisions` tem entry `kind='regrounding_prompt'` com `payload = drift_check_result`
- Tool call proposta **não** foi invocada

**Ações ao entrar:**
- Re-injeta `[regrounding_context]` no próximo turno:
  ```
  [regrounding_context]
  Goal: <literal>
  Pinned: <literal>

  Last 3 decisions:
    - ...

  Next action you proposed: <tool> with intent: <intent>

  Drift detector flagged this as: drifted (confidence: 0.84)
  Reason: <detector_reason>

  Choose:
    (a) Confirm action — continue as proposed
    (b) Re-plan — discard this tool_use, generate new approach
    (c) Update goal — current goal is stale; user input needed
  [/regrounding_context]
  ```
- UI mostra modal com 3 opções; default escolhido em 30s = `(b) Re-plan`.

**Timeout override:** `pending_decisions.expires_at` default é 5min (`§8`); para `kind='regrounding_prompt'` é override pra 30s. Razão: drift modal interrompe trabalho ativo; default longo de 5min cria pausa custosa. Auto-replan em vez de auto-deny (default de §8) é seguro: tool proposta é descartada mas sessão continua produtiva. Override declarado em código de Permission Engine, não config.

**Transições:**
- `→ tool_exec` em `(a) Confirm` (drift confirmado pelo user/modelo como falso positivo)
- `→ running` em `(b) Re-plan` (modelo gera próximo step do zero)
- `→ awaiting_user` em `(c) Update goal` (modal pra editar goal; raro)
- `→ interrupted` em Ctrl+C

### 11.4 Calibração e fail-open

Drift detector é **fail-open por construção**:
- Schema violation no Haiku output → default `aligned`
- Provider down → skip detection, proceed (warning em `failure_events`)
- Confidence < 0.7 com sinal de drift → continua mas registra evento

Razão: drift detector que bloqueia ação válida é **mais caro** do que drift que escapa. False positive interrompe trabalho legítimo; false negative custa um eval round depois.

### 11.5 Eval acoplado

`evals/drift/` com 30 fixtures:
- 10 sessões "alinhadas" (todas as ações servem o goal) — espera-se **0 false positives**
- 10 sessões "drift gradual" (foco escorrega) — espera-se **≥ 80% recall** no detector
- 10 sessões "drift abrupto" (modelo entra em rabbit hole) — espera-se **≥ 95% recall**

Métricas: precision (false positives), recall (drift detectado), latency p95, cost per check.

PR-bloqueante: precision < 0.95 (fail-open mas raro), recall < 0.80 em "gradual", recall < 0.95 em "abrupto", custo médio > $0.001/check.

### 11.6 O que **não** garantimos

- Detector funciona em **multi-goal sessions** (`§2.3` — goal stack) com goal ativo bem definido. Sessões com goals empilhados sem clear active marker degradam para skip + warning.
- Detector funciona em sessões com pin loaded (mantém constraint visível). Sessão sem pin + sem decisions[] = baseline degradado (recall cai pra ~60%).
- Modelo de detecção (Haiku) calibrado para natural language goals. Goals em formato exclusivamente structured (DAG node spec) usam validators de `ORCHESTRATION.md §2.2` em vez de drift detector.

---

## 12. Clarification gate (anti-presumption)

> **Cross-refs:** tool spec em `CONTRACTS.md §2.6.5e`; per-playbook override em `PLAYBOOKS.md §1.1` (`clarify_mode`); estado relacionado `[regrounding]` em `§11.3` (mesma família: pause + ask + resume).

Drift detector (`§11`) ataca *modelo focado em coisa errada*. Clarification gate ataca *modelo presumindo decisão ambígua sem perguntar*. São duas falhas distintas:

- **Drift:** goal era X, modelo está fazendo Y.
- **Presumption:** goal era ambíguo entre X1/X2; modelo escolheu X1 silenciosamente, sem sinalizar.

Modelo default tende a presumir e prosseguir. Em workflow de side-effect, presumir errado custa uma sessão inteira. Gate força modelo a externalizar a ambiguidade quando blast radius for alto, e a registrar em `assumptions[]` quando for baixo.

### 12.1 Tool `clarify` (interface)

Modelo emite `clarify(question, options, why_it_matters, blast_radius)` quando enfrenta decisão interpretativa. Schema completo em `CONTRACTS.md §2.6.5e`.

```
clarify(
  question: "qual `validateOrder` é o alvo do refactor?",
  options: [
    { id: "a", label: "src/orders.ts:142 (createOrder flow)" },
    { id: "b", label: "src/checkout.ts:89 (paymentIntent flow)" },
    { id: "c", label: "ambos" }
  ],
  why_it_matters: "blast radius diferente; (a) toca 3 arquivos, (b) toca 8",
  blast_radius: "high"
)
```

`blast_radius: low | medium | high`:
- **`low`** — decisão estética (nome de variável, ordem de import). Tool retorna *imediatamente* com `auto_assumed: <option[0]>`; registrado em `assumptions[]` do output schema do playbook. **Não interrompe.**
- **`medium`** — afeta um arquivo / um caso de uso. Bufferizado — agrupado com outras `medium` pendentes; modal disparado **antes do próximo write** (batched).
- **`high`** — afeta múltiplos arquivos / contrato externo / decisão irreversível. Modal **imediato** (transição → `[clarifying]`).

### 12.2 Modos por playbook (`clarify_mode`)

| Modo | Comportamento |
|---|---|
| `pre_execution` | modelo emite todas `clarify()` em fase exploratória; modal único batched antes da 1ª ação write. Falha em emitir clarification em fase write = drift detector flagga (`§11`). |
| `on_high_blast` | só `high` interrompe; `medium`/`low` viram `assumptions[]` registradas. Default. |
| `off` | tool `clarify` indisponível ao modelo; tudo vira `assumptions[]`. Workflows read-only e exploratórios (`explain`, `code-review`). |

### 12.3 Estado `[clarifying]`

Estado transiente disparado por `clarify()` com `blast_radius: high` (ou batched em `pre_execution` mode).

**Invariantes:**
- `sessions.status = 'clarifying'`
- `pending_decisions` tem entry `kind='clarification_prompt'` com `payload = clarification_request`
- Tool `clarify` ainda em flight; outras tool calls **não invocadas** durante este estado
- `active_goal` (`§2.3`) imutável durante clarification (mudança de goal só via `regrounding` ou explicit `/goal push`)

**Ações ao entrar:**
- UI mostra modal com até 3 perguntas batched, cada uma com opções + `why_it_matters`
- Default escolhido em 60s = `(skip and proceed with auto_assumed[0])` — registra em `assumptions[]` com `auto_chosen: true`

**Transições:**
- `→ running` em `clarification_resolved` (user respondeu; tool retorna com escolhas; modelo continua)
- `→ running` em `clarification_skipped` (timeout ou user clicou "skip"; assumptions[] registra auto-choice)
- `→ awaiting_user` em `clarification_escalated` (user clicou "edit goal" — caso raro de ambiguidade que indica goal errado)
- `→ interrupted` em Ctrl+C

**Timeout override:** `pending_decisions.expires_at` para `kind='clarification_prompt'` é 60s (vs default §8 de 5min). Razão: clarification interrompe trabalho ativo; timeout longo amplifica custo. Auto-skip com registro em `assumptions[]` é seguro — reverso do `regrounding` (que default-replan), aqui default-proceed-with-auto.

### 12.4 Schema `clarification_events`

```sql
clarification_events(
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  step_id TEXT NOT NULL,
  question TEXT NOT NULL,
  options_json TEXT NOT NULL,          -- JSON array de options
  why_it_matters TEXT,
  blast_radius TEXT NOT NULL,          -- low | medium | high
  outcome TEXT NOT NULL,               -- resolved | skipped | escalated | auto_low
  chosen_option_id TEXT,               -- NULL se skipped/escalated
  user_text TEXT,                      -- se user respondeu free-form em vez de option
  created_at INTEGER NOT NULL,
  resolved_at INTEGER,
  latency_ms INTEGER
);
```

Ratio `resolved / total` é proxy de qualidade da feature: alto = perguntas valiosas; baixo = ruído (user pula maioria).

### 12.5 Eval acoplado

`evals/clarification/` com 25 fixtures:
- 10 sessões com **ambiguidade real semeada** (user prompt deliberadamente ambíguo): espera `clarification_recall ≥ 0.8` (modelo emite `clarify` na maioria)
- 10 sessões **sem ambiguidade**: espera `clarification_precision ≥ 0.7` (≤ 30% das `clarify()` emitidas resultam em "user pula" — sinal de ruído)
- 5 sessões **trivia-trap**: prompts onde modelo ingênuo perguntaria detalhe estilístico; espera **zero** `clarify()` com `blast_radius: low` chegando ao modal (todas viram assumptions[] silentemente)

Métricas:
- `clarification_precision` = `outcome=resolved / (outcome=resolved + outcome=skipped)`
- `clarification_recall` = `outcome=resolved em fixture-com-ambiguidade-real / total ambiguidades semeadas`
- `noise_rate` = `outcome=skipped / total` (proxy de UX cansativa)

Threshold PR-bloqueante: `precision ≥ 0.7`, `recall ≥ 0.8` em fixture-com-ambiguidade, `noise_rate ≤ 0.3`.

### 12.6 Anti-patterns

- **Clarification de trivia.** "Posso usar `let` ou `const`?" — vai pra `assumptions[]` automático; `clarify()` com low blast pra detalhe estético é ruído. Eval `noise_rate` pega.
- **Clarification que ignora opções óbvias.** Pergunta sem `options[]` (free-form) força user a digitar. Tool exige ≥ 2 options; free-form único = bug do prompt do modelo.
- **Clarification em fase de execução** com `clarify_mode: pre_execution`. Modelo deveria ter perguntado na fase exploratória; emitir tarde viola contrato. Drift detector flagga.
- **Pergunta retórica.** "Você tem certeza que quer fazer X?" — não é clarification, é confirmation; usa `awaiting_user` via permission engine (`§2.2`).
- **Bypass de assumptions[].** Modelo emite `clarify()` com low blast pra **evitar** registrar em `assumptions[]` (parece mais zeloso). Eval pega via `outcome=auto_low rate`.
- **Multi-goal clarification.** Pergunta sobre 2 sub-goals diferentes ao mesmo tempo. Goal stack (`§2.3`) força um goal ativo; clarification opera no escopo do active_goal apenas.

### 12.7 Interação com outras primitivas

| Primitiva | Relação |
|---|---|
| Drift detector (`§11`) | Disjuntos: drift compara intent vs goal (foco); clarification expõe ambiguidade *no próprio entendimento do goal*. Em sessão saudável, clarification roda na fase pre_execution e drift roda em writes. |
| Goal stack (`§2.3`) | Clarification opera no `active_goal`. Clarification que termina em `escalated` pode levar a `goal_push` (sub-goal nasce da resposta do user). |
| Self-critique (`ORCH §6`) | Critique pode flagga "modelo presumiu sem clarificar" — sugere voltar pra fase pre_execution. |
| `assumptions[]` em playbook output | Clarification `low` + skipped/auto-choice viram `assumptions[]`. É a integração canônica. |
| Permission engine (`awaiting_user`) | Distintos: permission é "pode executar essa tool?"; clarification é "qual interpretação do goal?". |

---

## 13. Insight final

Estado vago é bug latente. Cada estado mal-definido é uma transição que vai quebrar em produção em algum cliente, em algum cenário, em algum momento — geralmente quando você não está olhando.

A regra é: **se você não consegue desenhar o diagrama, não consegue implementar o sistema.**

Esses estados não são exaustivos — são o **mínimo necessário** para que crash recovery, eval reproducível, e debug determinístico funcionem. Adicionar estado é fácil; remover é breaking change pra quem depende do invariant.
