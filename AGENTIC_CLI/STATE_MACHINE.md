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
- `→ error_fatal*` se compaction LLM retornar lixo persistente (após retry)
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

---

## 3. Step State Machine

Cada step (uma iteração do loop) tem máquina própria.

```
[pending] → [generating] → [tool_use_proposed] → [tool_executed] → [observing] → [done*]
                                  ↓                      ↓
                              [denied]              [tool_failed]
                                  ↓                      ↓
                              [done*]              [observing] (retry)
```

### Estados

| Estado | Persistência | Significado |
|---|---|---|
| `pending` | row em `messages` com role='assistant', sem content | step alocado, nada gerado ainda |
| `generating` | content acumulando via stream | modelo emitindo tokens |
| `tool_use_proposed` | tool_use no content; entry em `tool_calls` com status='pending' | tool_use parseado, awaiting permission |
| `denied` | `approvals` row com decision='deny' | permission negou; tool result = error |
| `tool_executed` | `tool_calls.status='done'` ou `error` | tool retornou resultado |
| `observing` | tool result anexado ao próximo `messages` row | aguardando próxima geração |
| `done` | step completo, contribuiu para sessão | terminal |

### Invariantes globais

1. Cada step tem **exatamente um** assistant message
2. Toda tool_use tem **exatamente uma** tool_result correspondente (mesmo se erro)
3. `tokens_in/tokens_out` são finalizados em `done`
4. Custo é finalizado em `done`

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
| `compact_failed` | compacting → error_fatal | Compaction LLM (after retries) |
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

## 11. Insight final

Estado vago é bug latente. Cada estado mal-definido é uma transição que vai quebrar em produção em algum cliente, em algum cenário, em algum momento — geralmente quando você não está olhando.

A regra é: **se você não consegue desenhar o diagrama, não consegue implementar o sistema.**

Esses estados não são exaustivos — são o **mínimo necessário** para que crash recovery, eval reproducível, e debug determinístico funcionem. Adicionar estado é fácil; remover é breaking change pra quem depende do invariant.
