# ORCHESTRATION

Coordenação de timing entre subsistemas do `AGENTIC_CLI`. Este doc é a **autoridade operacional** para perguntas "quando X roda relativo a Y".

`AGENTIC_CLI.md` é spec arquitetural (**o quê**). Este doc é spec operacional (**quando**, em que ordem, com que paralelismo, com que cancellation).

Sem timing explícito, dois implementadores divergem silenciosamente — bugs sutis em prod. Este doc resolve as ambiguidades.

---

## 0. Princípios

1. **Toda coordenação é explícita.** Sem timing implícito.
2. **Cancellation é paralela**, não sequencial. Wall-clock total bound.
3. **Concorrência é declarada**, nunca emergent.
4. **Compaction é entre steps**, nunca no meio.
5. **Hooks bloqueáveis e não-bloqueáveis** têm path distinto.
6. **Subagent semantics** depende da tool (sync vs async).
7. **Budget é compartilhado**, não pre-alocado.
8. **Decisões de timing honram a premissa raiz** — meça duas vezes (validators, hooks, critique), corte uma (single execution path).

---

## 1. Master loop (autonomous profile)

Diagrama com pontos de hook anotados:

```
┌─────────────────────────────────────────────────────────────┐
│ user_prompt received                                         │
│      ↓                                                       │
│ [SessionStart hooks] (if first turn) — BLOCKING              │
│      ↓                                                       │
│ [UserPromptSubmit hooks] — BLOCKING (block possível)         │
│      ↓                                                       │
│ ┌─ Step Loop ─────────────────────────────────────────────┐  │
│ │ Step N start (tx SQLite begin)                           │  │
│ │     ↓                                                    │  │
│ │ Context assembly                                         │  │
│ │     ↓                                                    │  │
│ │ Compaction check (token count vs 70% threshold)          │  │
│ │     ├─ trigger: PreCompact hook → compact (§4)           │  │
│ │     └─ continue                                          │  │
│ │     ↓                                                    │  │
│ │ Provider call (stream)                                   │  │
│ │     ↓                                                    │  │
│ │ Parse: text + tool_use blocks                            │  │
│ │     ↓                                                    │  │
│ │ if tool_use proposed:                                    │  │
│ │   [PreToolUse hook] — BLOCKING (deny possível)           │  │
│ │     ↓                                                    │  │
│ │   Permission check (allow/deny/confirm)                  │  │
│ │     ↓                                                    │  │
│ │   if writes: true → [PreCheckpoint hook] → snapshot      │  │
│ │     ↓                                                    │  │
│ │   Tool execute (with AbortSignal)                        │  │
│ │     ↓                                                    │  │
│ │   Output sanitization (ANSI strip, secret redact)        │  │
│ │     ↓                                                    │  │
│ │   [PostToolUse hook] — FIRE-AND-FORGET                   │  │
│ │     ↓                                                    │  │
│ │   Tool result → context                                  │  │
│ │     ↓                                                    │  │
│ │ if no tool_use (model emitted stop):                     │  │
│ │   [Self-critique pass] (if mode applicable, §6)          │  │
│ │     ↓                                                    │  │
│ │   Persist final assistant message                        │  │
│ │     ↓                                                    │  │
│ │ Step done (tx SQLite commit) → idle                      │  │
│ └──────────────────────────────────────────────────────────┘  │
│      ↓                                                       │
│ [Stop hook] (em fim de sessão) — BLOCKING                    │
└──────────────────────────────────────────────────────────────┘
```

### 1.1 Insertion points formalizados

| Ponto | Tipo | Bloqueia loop? | Pode cancelar ação? |
|---|---|---|---|
| `SessionStart` | hook | sim | não (no-op em block) |
| `UserPromptSubmit` | hook | sim | sim (rejeita prompt) |
| Compaction trigger check | check determinístico | sim (suspende step) | não |
| `PreCompact` | hook | sim | sim (cancela compaction) |
| Compaction LLM call | call | sim (próximo step espera) | não |
| `PreToolUse` | hook | sim | sim (nega tool) |
| Permission engine | check | sim | sim (deny por policy) |
| `PreCheckpoint` | hook | sim | não |
| Checkpoint snapshot | FS atomic op | sim (atomicidade) | não |
| Tool execute | runtime | sim (até retorno ou abort) | n/a (já executando) |
| Output sanitization | filtro | sim (~ms) | não |
| `PostToolUse` | hook | **não — fire-and-forget** | não |
| Self-critique | LLM | sim (opt-in) | sim (block persist) |
| `Notification` | hook | não — fire-and-forget | não |
| `MemoryWrite` | hook | sim | sim (bloqueia write) |
| `Stop` | hook | sim | não |

### 1.2 Atomicidade transacional

Cada step é uma **transação SQLite**. Begin no start, commit no done. Crash mid-step:
- INSERT/UPDATE rolled back
- State machine §7 cuida do recovery

Compaction também é transação (§4.3).

### 1.3 Multi-tool-use em único step

Modelos modernos (Anthropic, OpenAI) podem emitir **múltiplos tool_use blocks** em uma única assistant message. Comportamento:

- **Default: sequencial** na ordem emitida pelo modelo
- Cada tool é unidade independente: falha de A **não cancela** B
- `PreToolUse` hook dispara **N vezes** (uma por tool); permission engine roda **por tool**
- Single step contém todos os pares tool_use+tool_result; commit no fim
- Tool_result no contexto na **mesma ordem** dos tool_use (Anthropic respeita; outros providers normalizam)

**Opt-in paralelismo:** tool definition pode declarar `parallel_safe: true` (read-only tools). Se **todos** tool_use no mesmo step são `parallel_safe`, harness roda em paralelo (limite `max_concurrent_tool_calls`). Default: sequencial. Tools com `writes: true` **nunca** rodam paralelo (race em FS).

```ts
interface Tool {
  // ... existing fields
  parallel_safe?: boolean   // default false; só read-only tools devem declarar true
}
```

### 1.4 Thinking tokens (extended thinking)

Anthropic com thinking enabled (Opus 4.x) emite `thinking_delta` antes do output. OpenAI `o1`/`o3` têm reasoning tokens não-streamáveis. Comportamento:

- **Display default: oculto.** Thinking não aparece no main view; toggle via `--show-thinking` ou `/thinking on`
- **Cost:** span próprio em traces (`reasoning.cost_usd`); **somado** ao step.cost_usd no `/cost` total mas linha separada no breakdown
- **Persistence em `messages`:** **NÃO** persiste (não-reprocessável em retry; provider gera novo a cada call)
- **Compaction:** descarta automaticamente (não vai pro summary)
- **Replay:** re-gera (não-determinístico mesmo com `temperature=0`)

UI: `<ThinkingIndicator>` opcional no rodapé mostra `🧠 thinking... (12s)` durante `thinking_delta` events. Removido quando output começa.

### 1.5 Provider error mid-stream

Provider retorna 5xx ou stream cuts mid-output. Comportamento:

- **Partial output descartado** — não persiste em `messages`
- **Cost dos partial tokens:** registrado em `failed_attempts.cost_usd` (separado de `step.cost_usd`); aparece em `/cost` como linha de waste
- **Retry:** backoff exponencial 200ms/800ms/3.2s; até 3 tentativas
- **UI feedback:** rodapé mostra `↻ retry 1/3 (provider 5xx)` discreto
- **Idempotency:** novo request usa **mesma conversation history**; não há seed; modelo regera de forma independente (esperado, aceito)
- **3 retries falham:** step marcado `error`; modelo **não recebe** o erro como tool_result (não é tool error); sessão pode entrar em `error_fatal` ou aguardar user (depende do FAILURE_MODES.md §2.1-2.3)

### 1.6 Loop degenerado detection

Além de `maxToolErrors` (5 erros consecutivos), harness detecta **loop degenerado** por padrão repetitivo:

- **Mesma tool com mesma input hash** invocada 3× em 5 steps consecutivos → `degenerate_loop` warning
- 5 erros em 7 steps consecutivos (qualquer tool) → `exhausted_errors`
- Action em qualquer detection: aborta loop com mensagem clara ao user; sessão entra em `error` (recoverable via resume)

Hash da input: SHA256 de `JSON.stringify(args, sortedKeys)`. Persistido em `tool_calls.input_hash` pra detecção rápida.

---

## 2. DAG execution model (orchestrated profile)

DAG é alternativa ao step loop. Mesmas primitivas (hooks, permissions, checkpoints), orquestração diferente.

### 2.1 Lifecycle de um DAG run

```
┌──────────────────────────────────────────────────┐
│ DAG load + validate (cycles, orphans, schemas)   │
│      ↓                                            │
│ [SessionStart hooks if first turn]                │
│      ↓                                            │
│ Topological sort                                  │
│      ↓                                            │
│ ┌─ Wave Schedule Loop ────────────────────────┐   │
│ │ Para cada wave (nodes ready):               │   │
│ │   spawn N nodes em paralelo (max_concurrent)│   │
│ │     ↓                                        │   │
│ │   execute_node(N) cada                       │   │
│ │     ↓                                        │   │
│ │   Persist node output (tx SQLite)            │   │
│ │     ↓                                        │   │
│ │   Aguarda toda wave completar                │   │
│ │     ↓                                        │   │
│ │   on_node_failure: aplica policy             │   │
│ │     ↓                                        │   │
│ │   Próxima wave                               │   │
│ └─────────────────────────────────────────────┘   │
│      ↓                                            │
│ Aggregate output                                  │
│      ↓                                            │
│ [Stop hook]                                       │
└──────────────────────────────────────────────────┘
```

### 2.2 Node execution detail

```
execute_node(N):
  if N.type == llm:
    Provider call (constrained se outputSchema declarado)
      ↓
    if tool_use proposed: dispatch via master loop tool flow (§1)
      ↓
    Validators (sequential, fast first)
      if any fatal: failed
      if any non-fatal + retries left: retry with hint (max_retries)

  if N.type == deterministic:
    Execute fn (puro TS)
      ↓
    Validators

  if N.type == tool:
    Permission check + checkpoint + execute (master loop tool flow)

  if N.type == subgraph:
    Recursive execute_dag (inline expansion, §9)
```

### 2.3 Paralelismo declarado

Header do DAG:

```yaml
execution:
  max_concurrent: 3                # default 3; cap 8 (§11)
  on_node_failure: continue | abort_siblings | abort_dag
  fail_strategy: fast | greedy
```

Defaults: `max_concurrent=3`, `on_node_failure=abort_siblings`, `fail_strategy=fast`.

**Comportamento:**
- Nodes sem dependência ⇒ rodam em paralelo até `max_concurrent`
- Nodes com dependência ⇒ aguardam predecessores
- `fast`: aborta wave atual no primeiro fail
- `greedy`: deixa wave terminar mesmo com fails (útil pra evals)

### 2.4 Per-node budget

```yaml
- id: locate
  type: llm
  budget:
    max_tokens: 4000
    timeout_ms: 30000
    max_retries: 2                 # default 2
    max_cost_usd: 0.05             # opcional; herda do DAG global
```

Default herda do DAG-level budget se ausente. DAG global herda da sessão.

### 2.5 Crash mid-DAG

Crash do executor com DAG em execução:

| Estado pré-crash | Recovery |
|---|---|
| Node em `done` (output persisted) | preservado |
| Node em `executing` | descartado; re-executa no resume |
| Node em `validating` | output buffer descartado; re-executa node |
| Node em `retrying` | retry counter preservado em SQLite; continua |

Nodes determinísticos são **idempotent by definition**; LLM nodes regeneram (custo extra aceito).

---

## 3. Subagent spawn semantics

### 3.1 Duas tools distintas

```ts
task_sync(playbook: string, prompt: string, budget?: Budget): SubagentOutput
task_async(playbook: string, prompt: string, budget?: Budget): SubagentHandle
task_await(handle: SubagentHandle, timeout_ms?: number): SubagentOutput
task_cancel(handle: SubagentHandle): void
```

`task` (alias legado) = `task_sync`.

### 3.2 Sync vs async

| Tool | Pai bloqueia? | Uso |
|---|---|---|
| `task_sync` | sim, até subagent terminar | default; coordenação simples |
| `task_async` | não; retorna handle imediato | múltiplos subagents paralelos |

### 3.3 Paralelismo

Step do pai pode emitir múltiplos `task_async()` em sequência:
1. Cada um spawna subagent **imediato**
2. Limite global `max_concurrent_subagents = 3` (default; cap 8)
3. Excedeu: spawn aguarda slot livre (não rejeita)

`task_sync` em sequência ⇒ subagents **sequenciais** (latência soma). Pra paralelismo: `task_async`.

### 3.4 Coleta de outputs

Pattern típico paralelo:

```ts
const h1 = task_async("explore", "find auth files");
const h2 = task_async("explore", "find queue files");
const h3 = task_async("explore", "find migrations");
const [out1, out2, out3] = await Promise.all([h1, h2, h3].map(task_await));
```

Pai vê **só os outputs finais**. History intermediária dos subagents nunca chega ao pai (§11.1 do AGENTIC_CLI.md — contexto isolado).

### 3.5 Budget shared, não pre-aloca

- Pai tem $5 budget restante
- Spawna 3 subagents com `task_async`
- Cada um pode usar até $5 (todo disponível) **competindo**
- Hit do limite global: subagent ativo recebe sinal de finalizar; novos spawns rejeitam com `budget_exhausted`
- Audit em `failure_events`

### 3.6 Cancel cascading

`task_cancel(handle)` ou Ctrl+C no pai:
- SIGTERM no subagent
- 5s graceful → SIGKILL
- Worktree (se isolation) cleanup
- `task_await` retorna `{ status: 'interrupted', ... }`

Cancel é paralelo (§7).

---

## 4. Compaction timing

### 4.1 Quando dispara

Trigger: tokens contados > **70%** do `context_window` do provider corrente.

Check **após cada step terminar**, **nunca no meio**:
- Step com tool em curso conclui antes
- Stream do modelo completa antes
- Validators (orchestrated) terminam antes
- DAG: check após **subgraph** terminar, não nodo individual

### 4.2 Sequência

```
Step N termina → idle
  ↓
context_token_count check
  ↓
if count > 70% × context_window:
    suspend step loop
    ↓
    [PreCompact hook] — BLOCKING (pode cancelar)
    ↓
    if cancelled: continue with truncation fallback
                  (cut tool results > 1KB com pointer)
    ↓
    Compaction LLM call (modelo configurável; default cheap)
    ↓
    Validate compaction output (schema check)
    ↓
    if invalid: retry once
    if invalid 2x OR budget exceeded: deterministic fallback (§4.6)
    ↓
    Atomic swap: turns >3 antigos ⇒ summary
    Goal re-injection literal + pinned context (CONTEXT_TUNING.md §12.4)
    ↓
    Resume step loop (próximo step usa contexto compactado)
```

### 4.3 Atomicidade

Compaction é **transação**:
- Old context preservado em SQLite até commit
- Falha mid-compaction: rollback pra old context (próximo step usa contexto não-compactado, com warning)
- Sucesso: turns marcados `compacted=true`; summary persisted

**Não há "estado intermediário" visível.** Compaction é all-or-nothing.

### 4.4 Compaction em DAG (orchestrated)

DAG node tem context próprio (mais isolado que session step). Compaction não dispara mid-DAG por default.

Se DAG é longo (50+ nodes) e context global cresce:
- Trigger: após cada **subgraph terminar** (não nodo individual)
- Mesma sequência

### 4.5 Modelo da compaction

Default por profile:
- `autonomous` Anthropic: Haiku
- `autonomous` OpenAI: gpt-4o-mini
- `autonomous` outro vendor: cheap model do mesmo provider
- `orchestrated`: backend local (mesmo modelo do executor)
- `hybrid`: planner model (default Haiku)

Override via `compaction.model` em config (ver `AGENTIC_CLI.md` §6 Compaction).

### 4.6 Fallback determinístico

> **Cross-refs:** pinned context em `CONTEXT_TUNING.md §12.4`; selective elision em `CONTEXT_TUNING.md §12.1-12.2`; failure mode `compaction.llm.unavailable` em `FAILURE_MODES.md`.

Compaction-via-LLM (§4.5) é o caminho default, mas tem três modos de falha que **não devem** levar a `error_fatal`:

1. **LLM falha 2× consecutivas** (§4.2 step "if invalid 2x") — provider indisponível, schema violation persistente, rate limit duro.
2. **Budget excedido** — `compaction.max_cost_usd` ou `compaction.max_duration_ms` estouraram (defaults: $0.05, 30s).
3. **PreCompact hook cancelou** (§4.2 step "if cancelled") — caminho já existente, agora unificado aqui.

Em qualquer um dos três, harness aplica fallback estático **sem LLM**:

```
Eviction policy (deterministic):
  1. Identify pinned items (CONTEXT_TUNING.md §12.4) — always preserved.
  2. Preserve last K turns literally (default K=3, configurable).
  3. Preserve goal + sub-goals literal (CONTEXT_TUNING.md §10).
  4. Drop tool_results from turns older than K, EXCEPT:
       - tool_results referenced em decisions[] (RECAP.md §3) — keep
       - tool_results com writes:true que não foram revertidos — keep metadata, drop body
       - pinned tool_results — keep
  5. Replace dropped tool_results com pointer:
       <tool_result tool="..." step="..." elided="size_bytes" reason="static_fallback">
  6. If still > 70% após drop: truncate oldest assistant turns head+tail
     (preserve first 200 chars + last 200 chars; insert "... N tokens elided ...").
  7. Re-inject goal + pinned context.
```

**Garantias:**

- **Idempotente.** Mesmo input ⇒ mesmo output. Sem LLM ⇒ sem variabilidade.
- **Bounded.** Custo zero (USD), latência < 50ms (puro SQL + string ops).
- **Audited.** Cada drop registrado em `sessions.elided_tool_results_count` (`CONTEXT_TUNING.md §12.2`); `failure_event` `compaction.fallback.used` com motivo (`llm_failed` | `budget_exceeded` | `hook_cancelled`).
- **Observável.** UI mostra warning `⚠ compaction degraded: static fallback (reason)` no próximo turno; `/recap` inclui o evento.

**O que se perde:** sumarização semântica de turns antigos. Modelo vê pointer em vez de "N rodadas de exploração resumidas em 2 frases". Trade-off explícito: **degradação visível > erro fatal**. Sessão continua; user pode `/clear` e recomeçar se quiser contexto limpo.

**Quando NÃO cair em fallback:**

- LLM falhou **1×** apenas: retry uma vez antes (já coberto em §4.2).
- Provider down mas backup provider configurado: tenta backup antes do fallback (`PROVIDERS.md` cobre fallback chain).
- `PreCompact` hook retornou com `hard_cancel: true` (`§5.1.1`): respeita decisão do user, não força fallback — vai pra `error_fatal` (raro; opt-in explícito).

**Eval acoplado:** `evals/compaction/static_fallback/` — fixture com provider mockado retornando lixo; verifica que sessão não quebra, contexto post-compaction satisfaz invariante de tokens, e modelo seguinte consegue completar tarefa simples (read-modify-write).

---

## 5. Hook chain composition

### 5.1 Bloqueáveis vs não-bloqueáveis

Tabela canônica:

| Evento | Bloqueia loop? | Block ação? | Schema do payload |
|---|---|---|---|
| `SessionStart` | sim | não | `{ session_id, cwd, profile }` |
| `UserPromptSubmit` | sim | sim (rejeita prompt) | `{ session_id, prompt }` |
| `PreToolUse` | sim | sim (nega tool) | `{ session_id, tool_name, args }` |
| `PostToolUse` | **não — fire-and-forget** | não | `{ session_id, tool_name, args, output, duration_ms }` |
| `PreCompact` | sim | sim (cancela compaction) | `{ session_id, current_tokens, threshold }` |
| `Notification` | não — fire-and-forget | não | `{ session_id, kind, message }` |
| `PreCheckpoint` | sim | não | `{ session_id, files_to_snapshot }` |
| `MemoryWrite` | sim | sim (bloqueia write) | `{ session_id, scope, name, body, source }` |
| `Stop` | sim (final) | não | `{ session_id, status, total_cost_usd }` |

### 5.1.1 Payload de retorno (hooks bloqueáveis)

Hook bloqueável retorna JSON com formato:

```json
{
  "decision": "allow" | "block",
  "reason": "<string opcional>",
  "hard_cancel": true | false        // opt-in; ver abaixo
}
```

**`hard_cancel: true`** (opt-in, default `false`) altera o tratamento do bloqueio em hooks específicos:

- **`PreCompact`** com `hard_cancel: true` → compaction não cai em fallback determinístico (`§4.6`); transita pra `error_fatal*`. Sem `hard_cancel`, bloqueio cai em fallback (sessão prossegue degradada).
- **Outros hooks bloqueáveis:** `hard_cancel` ignorado (sem semântica especial); decisão `block` sempre cancela a operação corrente sem matar a sessão.

Razão: default é **fail-soft** (degradar > parar). User que precisa de fail-stop em compaction (ex: política regulatória que proíbe contexto incompleto) ativa `hard_cancel` explicitamente.

### 5.2 Ordem de execução

Pra mesmo evento com N hooks, ordem hierárquica:

1. **Enterprise hooks** (em ordem de declaração)
2. **User hooks**
3. **Project hooks**

**Sequencial dentro do mesmo nível.** Primeiro hook que retorna `block` em evento bloqueável **interrompe a chain**.

### 5.3 Timeouts

| Limite | Default | Configurável até |
|---|---|---|
| Por hook | 5s | 30s |
| Chain total | 15s | 30s (`max_hook_chain_ms`) |

Se chain hit total: hooks remanescentes não rodam; warning loggado em `hook_runs`.

Em evento bloqueável: assume `allow` se chain não terminou (a menos `fail_closed: true` no hook).

### 5.4 Fire-and-forget para não-bloqueáveis

`PostToolUse`, `Notification`:
- **Não bloqueiam** loop principal
- Loop principal **inicia próximo step imediato**
- Hook roda em background; output ignorado pelo loop
- Falha do hook não afeta loop (audit em `hook_runs`)
- Hook órfão (timeout extrapolado): SIGTERM em background; loop não espera

### 5.5 Hooks paralelos (NÃO suportado em v1)

Hooks **sempre sequenciais**. Paralelismo entre hooks = race condition em side effects (auto-format conflitando com lint, etc).

V2 pode adicionar `parallel: true` flag opt-in com responsabilidade do user.

---

## 6. Self-critique placement

### 6.1 Quando roda

`critique.mode` determina:

| Mode | Quando |
|---|---|
| `off` (default) | nunca |
| `on_writes` | step com tool_use de `writes: true` propõe **antes do invoke**; step que termina sem tool_use **antes do persist** |
| `always` | toda step LLM (excluí read-only tool steps como `read_file`/`grep`) |

### 6.2 Sequência

```
Step N: provider call retorna output
  ↓
Parse output → buffer (NÃO persist ainda)
  ↓
if critique.mode applicable to this step:
    Critic LLM call (input: step input + output buffer)
      ↓
    Parse critique → structured issues (schema fixo)
      ↓
    Filter por threshold (default 0.7)
      ↓
    if issues filtered:
        Show <CritiqueOverlay>
        User: ignore | redo | abort
          ↓
        if ignore: persist buffer → context (registra `critique.warning_ignored`)
        if redo: discard buffer; re-run step com hint do critic injetado
        if abort: discard buffer; step → error_user_aborted
    else:
        persist buffer → context
  ↓
Step done
```

### 6.3 Custo & telemetry

- Critique cost: span próprio em traces (`critique.cost_usd`, `critique.duration_ms`)
- **Não somado** ao cost do step principal
- Aparece em `/cost` breakdown como linha separada

### 6.4 Critique não recursivo

Critic LLM **não tem critique próprio** (loop infinito). É plain LLM call, sem self-critique encadeado.

### 6.5 Critique para tool_use

Para `on_writes` em tool com `writes: true`:
- Critique do **plano de invocação** (args do tool antes de invoke)
- Não do tool result (que já tem permission engine)
- Se critic detecta plano ruim: bloqueia invoke; user decide redo/abort

Para `always` em tool read-only:
- Critique do output completo do step (texto + tool_use propostos)

### 6.6 Trade-offs em hybrid

Em profile hybrid:
- `critique.model` pode ser local (qwen) mesmo com executor frontier
- Ou inverso: critique frontier mesmo com executor local
- Casos de uso:
  - Local executor + frontier critique = "second opinion" pra catches
  - Frontier executor + local critique = double-check barato

---

## 7. Cancellation cascading

### 7.1 AbortSignal paralelo

User Ctrl+C → `interrupt_signal` propagado simultaneamente para:

- Tool em curso (HTTP abort, signal handler)
- Hook em curso (SIGTERM)
- Subagents (SIGTERM via process tree)
- LLM stream (HTTP abort do request)
- DAG executor (cascade pra todos os nodes)
- Critique LLM call (HTTP abort)

**NÃO recebe** AbortSignal:
- Background processes (`bash_background`) — preservados via heartbeat; ver §3 PERFORMANCE
- Compaction em curso (atomic; trata como graceful complete; aborta na próxima vez)
- Checkpoint snapshot em curso (atomicidade FS)
- SQLite transaction commit (atomicidade DB)

### 7.2 Wall-clock budget

```
T+0:    interrupt_signal disparado paralelo
T+5s:   graceful deadline — tudo deve terminar
T+6s:   SIGKILL — forced kill em processos sobreviventes
T+6s:   sessão entra em `idle` ou `done`
```

Total ≤ 6s pra `interrupt_complete`.

Subsistemas que sobrevivem após SIGKILL: marcados como zombie em `failure_events.zombie_process`; sessão prossegue (não bloqueia).

### 7.3 Estado após interrupt

- Step em curso, **não persistido**: descartado
- Step com tool_calls **persistidos** (mesmo se tool não terminou): marcado `interrupted` em `tool_calls.status`
- Subagent: output sintético `{ status: 'interrupted' }`
- DAG: nodes em `executing` marcados `interrupted`; `done` preservados

Próxima ação: aguarda input do user (`idle`) ou exit (`done`).

### 7.4 Double Ctrl+C

- 1× Ctrl+C: `interrupt_signal` (graceful)
- 2× Ctrl+C dentro de 1s: força SIGKILL imediato (skip 5s graceful)
- 3× Ctrl+C: exit do processo do agente (mata tudo, incluindo background processes)

Audit registra cada nível.

### 7.5 Stream interrupt UX (mid-token)

User interrompe **enquanto modelo está streamando tokens**:

| Atalho | Estado loop | Ação |
|---|---|---|
| `Esc Esc` | streaming | interrompe stream; **tokens parciais permanecem na tela** com label `[interrupted at token N]`; input editável com prompt original carregado; loop volta a `idle` |
| `Esc Esc` | tool_exec | interrompe tool (graceful 5s + SIGKILL); tool_result sintético `{ status: 'interrupted' }`; volta a `idle` |
| `Esc Esc` | compacting | **ignorado** (compaction é atomic, ≤ 3s; aguarda terminar) |
| `Ctrl+C` 1× | streaming | full interrupt cascading (§7.1); tokens parciais somem da tela; cost dos parciais registrado em `failed_attempts.cost_usd` |
| `Ctrl+C` 1× | tool_exec | mostra prompt "tool em execução; press de novo pra forçar"; 5s timeout volta ao normal |
| `Ctrl+C` 2× rápido | qualquer | force SIGKILL imediato |
| `Ctrl+C` 3× | qualquer | exit do agent |

**Diferença chave Esc Esc vs Ctrl+C em streaming:**
- `Esc Esc` é **suave** — pra "ah, quero re-prompt diferente"; texto visível preservado pra contexto humano
- `Ctrl+C` é **cancel total** — pra "para tudo"; tela limpa do step incompleto

Em ambos: **tokens parciais NÃO vão pra `messages`** (rollback de step transaction). Cost de tokens emitidos é cobrado pelo provider e registrado separado.

---

## 8. Budget cascading

### 8.1 Hierarquia

```
Session budget (max_cost_usd, max_steps, max_wall_clock_ms)
  └─ shared com:
       Subagent budget (≤ session, sem pre-alocação)
       DAG execution budget (≤ session)
       Compaction call budget (≤ session)
       Critique call budget (≤ session)
       Provider call budget (per-call timeout, retry budget)
```

Tudo compete pelo budget remanescente.

### 8.2 Hit do limite

| Limite | Comportamento |
|---|---|
| Hard cap (`max_cost_usd`) | step ativo recebe sinal de finalizar; novos spawns rejeitados; sessão eventualmente marca `exhausted` |
| Soft warning (90%) | UI alerta; user pode `/budget extend $N` |
| `max_steps` hit | sessão imediatamente `exhausted` no fim do step atual |
| `max_wall_clock_ms` hit | `interrupt_signal` paralelo; cleanup |

### 8.3 Subagent vs pai

- Subagent hit antes do pai: subagent termina com `status: exhausted`; pai continua com output parcial
- Pai hit antes de subagent: pai sinaliza subagent pra terminar; aguarda graceful

### 8.4 Allocation hints

User pode hintar (não enforce):

```bash
agent --max-cost 5 --subagent-budget-hint 1.5 "..."
```

Hint usado pelo planner pra:
- Pre-rejeitar `task_async` que provavelmente excede o hint
- UI mostrar "subagent excedeu hint mas continua dentro do budget global"

Pode ser ignorado pelo modelo se uso real divergir.

---

## 9. Subgraph expansion model

### 9.1 Inline expansion

Subgraph node é **expandido em load time** do DAG:

```yaml
# DAG principal
nodes:
  - id: prep
    type: deterministic
    fn: load_baseline
  - id: refactor
    type: subgraph
    ref: edit_function       # nome do DAG referenciado
    inputs_from: [prep]
```

```yaml
# Subgraph edit_function (em outro arquivo)
nodes:
  - id: locate
    type: llm
    inputs: external          # vem de inputs_from do parent
  - id: apply
    type: tool
    inputs_from: [locate]
```

Em load:
```
[prep] → [edit_function.locate] → [edit_function.apply]
```

Mesma execução, mesmo processo, mesmo session.

### 9.2 Inputs/outputs

- `inputs_from: [prep]` no subgraph node = passado como input do **first node(s)** do subgraph (declarado com `inputs: external`)
- Output do **last node** do subgraph = output do subgraph node

Validation em load: se subgraph não tem first node com `inputs: external` ou last node com schema declarado, falha.

### 9.3 Cancelamento e budget

- AbortSignal cascateia pra todos nodes expandidos
- Budget herda do DAG pai (não tem budget próprio)
- Falhas reportam path completo: `parent.refactor.locate (validation_failed)`

### 9.4 Quando NÃO usar subgraph

Use `task_async()`/subagent quando:
- Quer **isolation real** (worktree dedicado)
- Quer **budget próprio** que não afete pai
- Subgraph **muito longo** (40+ nodes) poluiria histórico/audit do pai

Subgraph é **inline + cheap** (sem isolation overhead). Subagent é **isolado + caro** (processo separado, contexto novo).

---

## 10. Hybrid routing grammar (v1)

### 10.1 Lista de regras simples

Cada regra: **uma condição, uma ação**.

```toml
[[profile.hybrid.rule]]
when = { validator_failures_consecutive_gte = 2 }
then = "fallback_to_frontier"

[[profile.hybrid.rule]]
when = { node_type = "refactor" }
then = "use_executor_local"

[[profile.hybrid.rule]]
when = { step_cost_estimate_gt_usd = 0.10 }
then = "ask_user"

[[profile.hybrid.rule]]
when = { context_tokens_gt = 30000 }
then = "use_executor_frontier"      # context grande precisa janela larga

# Catch-all default (recomendado)
[[profile.hybrid.rule]]
when = "always"
then = "use_executor_local"
```

### 10.2 Conditions suportadas (v1)

| Condition | Tipo |
|---|---|
| `validator_failures_consecutive_gte` | int |
| `node_type` | string |
| `tool_name` | string |
| `step_cost_estimate_gt_usd` | float |
| `context_tokens_gt` | int |
| `model_capability_required` | string (`vision` \| `extended_cache` \| `tools_native`) |
| `always` | (catch-all) |

### 10.3 Actions suportadas (v1)

| Action | Comportamento |
|---|---|
| `use_executor_local` | step roda em local model |
| `use_executor_frontier` | step roda em frontier model |
| `fallback_to_frontier` | escala atual pra próximo tier |
| `ask_user` | apresenta modal de escolha |
| `abort_step` | marca step como falha sem tentar |

### 10.4 Precedência

Regras avaliadas em **ordem de declaração**. Primeira que match aplica. Sem AND/OR/NOT em v1.

**Catch-all sempre recomendado** como última regra (evita comportamento undefined).

### 10.5 v2 — DSL composta (deferred)

Quando demanda real chegar:
- Operadores (AND, OR, NOT)
- Nested conditions
- Aliasing de regras
- Validation em config-load

Por enquanto: enxuto e suficiente.

---

## 11. Concurrency limits matrix

Resumo dos limites operacionais:

| Recurso | Default | Hard cap | Configurável? |
|---|---|---|---|
| Sessões ativas (mesmo cwd) | 1 | 1 (lockfile) | não |
| Subagents paralelos por sessão | 3 | 8 | sim (`max_concurrent_subagents`) |
| Tool calls em flight (DAG) | 5 | 16 | sim (`max_concurrent_tool_calls`) |
| LLM calls concorrentes (DAG) | 5 | 16 | sim (`max_concurrent_llm_calls`) |
| Background processes | 5 | 20 | sim |
| Hooks paralelos por evento | 1 | 1 (sequencial) | não em v1 |
| MCP server connections | 10 | 30 | sim |
| Validators sequential per node | 5 | 20 | sim |
| Critique calls em flight | 1 | 1 (sequencial) | não |
| Compaction calls em flight | 1 | 1 (atomic) | não |
| `wait_for` em flight (per session) | 3 | 8 | sim |
| `monitor` em flight (per session) | 2 | 5 | sim |

---

## 12. Anti-patterns

| Anti-pattern | Por quê ruim |
|---|---|
| Compaction mid-step | Quebra atomicidade; tool result órfão |
| `task_sync` em sequência pra tarefas paralelizáveis | Latência cumulativa; use `task_async` |
| `task_async` sem `task_await` correspondente | Subagent zombie; budget queimado sem benefício |
| Hook não-bloqueável bloqueando loop | Quebra contrato; bug |
| Critique recursivo (critique do critique) | Loop infinito; proibido |
| Subgraph com isolation real | Use `task_async`; subgraph é inline |
| Cancellation sequencial | Wall-clock acumula; user fica esperando |
| Budget pre-alocado entre subagents | Desperdiça reservatório; competição shared é eficiente |
| Hybrid routing sem catch-all default | Comportamento undefined em edge cases |
| Compaction LLM com mesmo modelo do step principal | Custo desnecessário; use cheap model |
| Hooks paralelos com side effects sobrepostos | Race condition; format vs lint conflict |
| DAG sem `max_concurrent` declarado | Implementação default pode divergir |
| `PostToolUse` que bloqueia (espera response) | Quebra fire-and-forget; trava loop |

---

## 13. Insight final

Orquestração não é "qual ordem o agente faz coisa". É **contrato de timing entre subsistemas** que precisa ser explícito pra implementadores não divergirem.

Spec arquitetural (`AGENTIC_CLI.md`, `CONTRACTS.md`, `STATE_MACHINE.md`) diz **o quê** e **com que invariantes**. Este doc diz **quando, em que ordem, com que paralelismo, com que cancellation**.

A regra é: **toda decisão de timing tomada uma vez aqui, replicada perfeita em todo lugar.** Sem timing implícito. Sem "fica a gosto da implementação". Sem ambiguidade que vira bug em prod.

E como tudo no projeto: **meça duas vezes, corte uma.** Validators medem antes de corte. Hooks medem antes/depois de corte. Compaction mede contexto antes de cortar (resumir). Critique mede output antes de comprometê-lo ao contexto. Cada decisão de timing aqui é instância dessa premissa aplicada à coordenação.
