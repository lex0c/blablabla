# CONTRACTS

Contratos formais entre subsistemas do `AGENTIC_CLI`. Cada par de camadas declara, explicitamente, **o que cada lado garante ao outro** e **o que acontece quando a garantia é quebrada**.

Sem esses contratos, implementadores diferentes fazem escolhas conflitantes e a integração quebra silenciosamente. Spec sem contratos é sugestão, não engenharia.

---

## 0. Princípios

1. **Cada contrato é bidirecional.** "A promete X a B" tem que ter "B promete Y a A".
2. **Quebra de contrato é fatal por default.** Se A não cumpre, B não tenta consertar — registra, escala, aborta.
3. **Contratos são versionados.** Mudança em garantia = bump de versão + migration path.
4. **Side effects declarados.** Se uma camada pode escrever em SQLite, FS, ou network, declara explicitamente.
5. **Determinism boundaries declarados.** Camada determinística não pode chamar LLM no caminho crítico.
6. **Idempotência declarada.** Operação idempotente é dita; resto é "at-most-once".

---

## 1. Notação

Cada contrato usa esta estrutura:

```
### <Nome do contrato>

A: <camada A>
B: <camada B>
Direção: A → B (A chama B), A ↔ B (bidirecional), A ← B (callback)

A garante a B:
  - <invariante 1>
  - ...

B garante a A:
  - <invariante 1>
  - ...

Failure semantics:
  <o que cada lado faz se a garantia da outra é violada>

Side effects permitidos:
  <de cada lado>

Versão: v1 (qualquer mudança = v2 + migration)
```

---

## 2. Tool ↔ Harness

A: **Agent Harness** (ou Step Graph Executor)
B: **Tool implementation**
Direção: A → B (A invoca tool)

### A (Harness) garante a B (Tool):

- `args` recebidos satisfazem `inputSchema` da tool — validação JSON Schema **antes** da invocação. Tool não precisa re-validar.
- `ctx` (ToolContext) inclui:
  - `signal: AbortSignal` — válido durante toda execução
  - `cwd: string` — diretório absoluto, existe, é diretório
  - `session_id: string`, `step_id: string` — para correlação
  - `permissions: PermissionsView` — só-leitura, snapshot do momento da invocação
- `signal` é **honrado** — se tool retornar `AbortError`, harness não considera erro
- Tool é chamada em `try/catch`; exception **não derruba** o harness
- Output é persistido **antes** do próximo step ser construído (atomicidade entre tool result e contexto)

### B (Tool) garante a A (Harness):

- Resolve em **tempo finito**: respeita `signal`; se ignorar, harness mata após `2 × ctx.timeout`
- Output satisfaz `outputSchema` (se declarado); fora do schema = bug, não graceful degradation
- **Sem mutação do `ctx`** — `ctx` é só-leitura
- **Sem state global mutável** entre invocações (tool é função, não classe com estado)
- Side effects **explícitos**:
  - `read_file`/`grep`/`glob`: read-only no FS
  - `write_file`/`edit_file`: declara `writes: true` em metadata → harness cria checkpoint **antes**
  - `bash`: declara `writes: true` (pessimista) salvo `bash:read_only` flag
  - `web_fetch`: network access; respeita `deny_hosts`
- Erros são **estruturados** seguindo schema canônico (não exception bare):
  ```ts
  interface ToolError {
    is_error: true                     // discriminator
    error_code: string                 // e.g., "tool.timeout", "tool.exception", "tool.schema_violation"
    error_message: string              // human-readable
    retryable: boolean                 // modelo pode tentar de novo?
    hint?: string                      // sugestão pro modelo
    details?: object                   // debug info estruturado, opcional
  }
  ```
  Tool result com `is_error: true` vai pro contexto do modelo no formato esperado pelo provider; modelo decide próxima ação. Erro de tool **nunca é exception** propagada pro harness.
- **Não escreve em SQLite** diretamente — só via output que harness persiste

### Failure semantics:

- Tool excede timeout → harness emite `AbortSignal`; aguarda 2s graceful; SIGKILL no processo (se aplicável); resultado vira `{ error: "timeout" }`
- Tool retorna fora do schema → harness loga, devolve ao modelo como `{ error: "tool returned invalid output", hint: <validation error> }` — modelo decide retry
- Tool exception não-capturada → harness captura, vira `{ error: "tool crashed", details: ... }`; **não propaga**
- Tool não respeita `writes: true` declarado mas escreve assim mesmo → checkpoint não foi criado, `/undo` não cobre. **Bug, não graceful degradation.** Detectado por hash do `cwd` antes/depois em modo dev.

### Side effects permitidos:
- Harness: persiste `tool_calls` no SQLite, emite spans OTEL
- Tool: side effects declarados em metadata + descrição

### Versão: **v1**

---

## 3. Hook ↔ Harness

A: **Hooks Dispatcher**
B: **Hook command** (shell process do usuário)
Direção: A → B (eventual com timeout)

### A (Dispatcher) garante a B (Hook):

- Input via **stdin** com JSON único terminado em `\n`:
  ```json
  {"schema": "v1", "event": "PreToolUse", "session_id": "...", "data": { ... }}
  ```
- Variáveis no comando expandidas via template (`{{tool.input.path}}`)
- Env limpa: apenas `PATH`, `HOME`, `AGENT_SESSION_ID`, `AGENT_CWD`
- `cwd` do processo = `cwd` da sessão
- Timeout default 5s; configurável até 30s; **nunca** ilimitado

### B (Hook) garante a A (Dispatcher):

- Termina em ≤ timeout — caso contrário é morto (SIGTERM → 1s → SIGKILL)
- **Exit code é o sinal**:
  - `0` — allow (continua fluxo normal)
  - `1` — block silencioso (aborta operação alvo, sem mensagem)
  - `2` — block com mensagem (stdout do hook vira motivo)
  - `>2` — erro de hook (loggado mas não bloqueia, salvo se `fail_closed: true` no config)
- stdout ≤ 4KB; truncado se exceder
- stderr é log (não afeta decisão)
- Sem assumir estado entre invocações (cada call é fresh process)

### Failure semantics:

- Timeout → `exit_code = 124` registrado; ação alvo procede a menos que `fail_closed: true`
- Comando não existe → `exit_code = 127` registrado; **não bloqueia** (configurável)
- stdin malformado → bug do dispatcher, não do hook; aborta operação e loga
- Hook produz subprocess órfão → fora do contrato; documentado como limitação (recomendar `setsid` no comando)

### Side effects permitidos:
- Dispatcher: grava `hook_runs`; nenhuma mutação de estado de tool
- Hook: qualquer side effect (é shell command); responsabilidade do usuário

### Eventos com regra especial:

| Evento | `exit 1` significa | Stdout do hook |
|---|---|---|
| `PreToolUse` | Nega tool call | Mensagem ao modelo |
| `UserPromptSubmit` | Rejeita prompt | Mensagem ao usuário |
| `PreCompact` | Cancela compaction | Mensagem em log |
| `MemoryWrite` | Bloqueia write | Motivo da rejeição |
| `PostToolUse`, `Notification`, `Stop`, `SessionStart`, `PreCheckpoint` | (não-bloqueável) | Apenas log |

### Versão: **v1**

---

## 4. Provider ↔ Context Engine

A: **Context Engine** (monta o request)
B: **Provider** (executa o request, retorna stream)
Direção: A → B (request) ↔ A (stream events)

### A (Context Engine) garante a B (Provider):

- `messages[]` é cronológico, sem buracos, sem duplicatas
- System prompt é **primeiro elemento** do request, único
- Tool schemas estão presentes apenas se a sessão tem tools habilitadas (não null, não array vazio quando o Provider não suporta)
- Sequence de cache breakpoints respeita ordem do layout (§6 do AGENTIC_CLI)
- Tokens contados via `provider.countTokens(messages)` antes de enviar; assertiva é abortar se `> context_window - reserve`

### B (Provider) garante a A (Context Engine):

- **Não reordena, não muta** `messages` recebido
- **Não injeta** system prompt extra (Anthropic adiciona instructions internas — está fora do nosso contrato; documentado)
- Stream events são **bem-formados e ordenados**:
  ```
  start → (delta | tool_use_start → (tool_use_delta) → tool_use_stop)* → stop
  ```
- Em caso de erro durante stream, emite `{ kind: "error", code, message, retryable }` antes de fechar
- `countTokens` é **pure function** — mesmo input = mesmo output
- Capabilities declaradas são **honestas**: `supports_tools: true` ⇒ tool calling funciona em 100% dos casos suportados
- Constrained output: se `outputSchema` é dado e `capabilities.constrained != false`, output **garantidamente** parseia contra o schema; falha = erro do provider, não do modelo

### Failure semantics:

- Provider 5xx / timeout → backoff exponencial (200ms, 800ms, 3.2s); 3 tentativas; depois aborta com `{ error: "provider unavailable", retryable: true }`
- Rate limit (429) → respeita header `Retry-After` se presente; senão backoff; após 3 retries vira erro fatal pra step (não pra sessão)
- Stream interrompido mid-tool-use → harness considera tool-use **inválido**, não tenta executar tool parcial
- Provider mente sobre capabilities → bug crítico; eval específico detecta; modelo sai do registry

### Side effects permitidos:
- Context Engine: read-only em SQLite (carrega memory, repo map)
- Provider: network call; cache server-side (transparente)

### Streaming pipeline canônico

Order-of-operations entre raw HTTP chunks e step persisted:

```
Provider HTTP/SSE chunks (raw bytes)
  ↓
Step 1: Stream parser per provider
  - Anthropic: SSE → JSON events (content_block_*, input_json_delta)
  - OpenAI: SSE → choice.delta accumulator
  - Ollama: NDJSON → message accumulator
  - llama.cpp: SSE → token deltas
  ↓
Step 2: Normalize → canonical StreamEvent
  - kind ∈ { start, text_delta, tool_use_start, tool_use_delta, tool_use_stop, thinking_delta, stop, error }
  - Adapter mapeia quirks de cada provider pro tipo único
  ↓
Step 3: Buffer (pra UI batching)
  - text_delta acumulado em batches de 33ms (30fps target)
  - tool_use_delta acumulado até tool_use_stop
  - Outros: emit imediato
  ↓
Step 4: Emit pra UI (streaming visible)
  - <StreamingMessage> recebe text_delta batched
  - <ToolCallCard> recebe tool_use_start
  - thinking_delta → <ThinkingIndicator>
  ↓
Step 5: Em tool_use_stop OU stop event:
  - Validate args estruturado (JSON parse + schema)
  - Falha de parse: tool_use descartado; modelo recebe ToolError
  ↓
Step 6: Persist em messages + tool_calls (transação SQLite)
  - text → messages.content
  - tool_use → tool_calls row
  - Buffer descartado após persist
```

**Backpressure:** se UI render mais lento que stream emit, buffer cresce até `max_stream_buffer_kb = 256`; depois pausa stream (HTTP slow read). Provider eventualmente timeout; raríssimo em prática.

**Mid-stream cancel atomicity:** Ctrl+C / Esc Esc dispara HTTP abort:
- Buffer **descartado** (não persistido)
- `failed_attempts.cost_usd` registra tokens cobrados pelo provider
- UI: tokens visíveis somem (Ctrl+C) ou permanecem com label `[interrupted]` (Esc Esc) — ver ORCHESTRATION.md §7.5

**Partial output never persists.** Único caso de persist parcial: provider crash mid-stream com tokens já parciais — esses ficam em `failed_attempts`, não em `messages`.

### Versão: **v1**

---

## 5. Subagent ↔ Parent

A: **Subagent** (processo filho)
B: **Parent agent**
Direção: B → A (spawn) ↔ A → B (output) e B → A (cancel)

### B (Parent) garante a A (Subagent):

- Input é único, completo, no spawn — sem updates posteriores
- Subagent recebe:
  - `prompt: string` (system prompt + task)
  - `tools: string[]` (whitelist, subset das tools do pai)
  - `budget: RunBudget` (≤ budget do pai)
  - `cwd: string` (worktree dedicado se `isolation: worktree`, senão cwd do pai)
  - `parent_session_id: string`
- Cancel via SIGTERM em processo do filho; SIGKILL após 5s de grace
- **Não interfere** no contexto do filho durante execução (sem injeção, sem updates)

### A (Subagent) garante a B (Parent):

- **Output estruturado único** ao terminar, escrito em `subagent_outputs(id, parent_id, ...)`:
  ```ts
  {
    status: "done" | "error" | "interrupted" | "exhausted",
    output: string,            // texto pra contexto do pai
    artifacts?: { kind, ref }[], // diff, branch, file (se worktree)
    cost_usd: number,
    steps_used: number,
    errors?: string[]
  }
  ```
- **Não vaza history** — pai não vê messages intermediárias do filho
- Crash do filho → output é `{ status: "error", output: "subagent crashed", errors: [<crash log>] }`
- Worktree (se aplicável) está em estado consistente: ou diff aplicado limpo, ou rolled back, **nunca pendurado**
- Honra cancel em **≤ 5s** (graceful) ou é morto

### Failure semantics:

- Filho ignora SIGTERM → parent emite SIGKILL; output presumido `{ status: "interrupted", errors: ["forced kill"] }`; worktree pode ficar corrompido — `worktree gc` na próxima sessão
- Filho escreve fora do worktree (quando isolado) → quebra de contrato; eval de teste cobre; mata feature flag em prod
- Output não-parseável → pai trata como `{ status: "error" }` e segue
- Filho excede budget próprio → termina com `{ status: "exhausted" }`; **não escala** pro budget do pai

### Side effects permitidos:
- Parent: escreve em `messages` (do pai), lê `subagent_outputs`
- Subagent: escreve em sessão própria + `subagent_outputs(parent_id=<id_do_pai>)`

### Versão: **v1**

---

## 6. Validator ↔ DAG Step (profile `orchestrated`)

A: **DAG Step Executor** (chama validator)
B: **Validator** (puro ou semi-puro)
Direção: A → B

### A (Step) garante a B (Validator):

- `output` recebido é o output cru do step (ou de step anterior, via `inputs_from`)
- `ctx.step_id` válido; `ctx.dag_id` válido
- **Sem reentrancy** — mesmo validator não é chamado recursivamente
- Validator chamado **uma vez** por output (sem retry interno do executor — retry é decisão do executor, não do validator)

### B (Validator) garante a A (Step):

- **Função pura ou semi-pura**: pode ler FS (`FileExistsValidator`) mas **não escreve** nada
- Resolve em **≤ 200ms** para validators "fast" (built-ins determinísticos); validators "slow" declaram explicitamente
- Resultado satisfaz tipo `ValidationResult`:
  ```ts
  | { ok: true; value: O }
  | { ok: false; error: string; retry_hint?: string; fatal?: boolean }
  ```
- `fatal: true` significa "não tente retry"; default é `false` (retry permitido)
- `retry_hint` é texto que o executor pode injetar no próximo prompt do step
- **Não chama LLM** (validator é determinístico ou semi-determinístico)
- **Não muta** o output recebido — retorna `value` (mesma referência ou cópia limpa)

### Failure semantics:

- Validator excede timeout → considerado `{ ok: false, error: "validator timeout", fatal: true }`
- Validator lança exception → captado, vira `{ ok: false, error: <message>, fatal: true }`
- Validator retorna formato fora do contrato → bug; step aborta com erro pra DAG executor
- Retry budget do step esgota com validator sempre falhando → step falha; DAG executor aplica `on_failure` policy

### Side effects permitidos:
- Step Executor: escreve em `step_outputs`, emite spans
- Validator: read-only (FS, schema, AST parse)

### Versão: **v1**

---

## 7. Memory ↔ Context Engine

A: **Memory Manager**
B: **Context Engine**
Direção: B → A (read), A → B (push de memórias auto-trigger)

### A (Memory Manager) garante a B (Context Engine):

- `getIndex(scope)` retorna índice em ≤ 50ms (cacheado em RAM, invalidate em write)
- Índice é texto markdown válido, ≤ 200 linhas (truncado se exceder)
- `getContent(name, scope)` retorna conteúdo do `.md` ou `null` se não existe
- Memórias com `trust: untrusted` **não são incluídas** no resultado de `getIndex` por default — só via `getIndex(scope, includeUntrusted: true)`
- Memórias expiradas (`expires < now`) **não aparecem** no índice (filtradas em runtime)
- Triggers (`triggers: [git_commit, ...]`) avaliados em B; A só serve

### B (Context Engine) garante a A (Memory Manager):

- **Não cacheia** índice fora de uma assembly; sempre re-pede em sessão nova
- **Não muta** conteúdo recebido — read-only
- Carrega índice em **posição fixa** do prompt (após CLAUDE.md, antes de tool schemas)
- Loga toda leitura via `memory_events(action: "read")`

### Failure semantics:

- Memory file corrupto (frontmatter inválido) → A retorna `{ error: "corrupt", path }`; B inclui no log mas **não inclui** no contexto; UI alerta
- Index file ausente → A reconstrói do conteúdo dos arquivos no diretório (uma operação, não em loop)
- Path traversal em `name` → A retorna erro; **nunca** segue `..` ou paths absolutos

### Side effects permitidos:
- Memory Manager: grava `memory_events`; lê arquivos em `~/.config/agent/memory/` e `./.agent/memory/`
- Context Engine: lê índice e conteúdo

### Versão: **v1**

---

## 8. Checkpoint Manager ↔ Filesystem

A: **Checkpoint Manager**
B: **Filesystem** (via git ou cp --reflink)
Direção: A → B

### A garante a B:

- `snapshot()` chamado **apenas antes** de tool com `writes: true`
- `cwd` é diretório válido, contém ou não repo git
- Operações são **atomicamente registradas**: ou checkpoint criado e linha em `checkpoints` table existe, ou nada (transação)

### B (FS) garante a A:

- Em git repo: `git stash create` é atômico (single object); ref em `refs/agent/checkpoints/<session>` é atualização atômica
- Em FS com reflink (btrfs, xfs, apfs): `cp --reflink=auto` é atômico (no level de inode)
- Em FS sem reflink: fallback é `cp -r` simples — **NÃO é atômico**, documentado como limitação

### Failure semantics:

- Disk full durante snapshot → checkpoint falha; harness **aborta tool call** (não executa sem checkpoint); estado do FS inalterado
- Git ref creation falha (ex: corrupt index) → fallback pra reflink; se também falha, fallback pra cp; se também falha, aborta
- Restore (via `/undo`) com checkpoint corrompido → erro fatal; usuário recebe instrução manual ("git stash list")
- Race entre dois agentes no mesmo cwd → primeiro lock, segundo aguarda 5s, depois aborta

### Side effects permitidos:
- Checkpoint Manager: cria refs/snapshots, escreve em `checkpoints` table
- FS: nenhum lado de fora do snapshot

### Versão: **v1**

---

## 9. Permission Engine ↔ Tool Registry

A: **Tool Registry** (recebe pedido de invocação)
B: **Permission Engine** (decide allow/deny/confirm)
Direção: A → B (consulta), B → A (decisão)

### A garante a B:

- Sempre consulta antes de invocar tool (sem caminho de bypass interno)
- Passa metadata completa: `{ tool_name, args, cwd, session_id, source }` (`source = "model" | "user_command" | "hook"`)
- Honra decisão **literalmente**: `allow` → invoca; `deny` → retorna erro pro modelo; `confirm` → modal UI antes de invocar

### B garante a A:

- Decisão em **≤ 10ms** (matching de glob é rápido)
- Decisão é **determinística** dado mesmo input (sem random, sem network)
- Decisão é **uma de três**: `{ kind: "allow" } | { kind: "deny", reason } | { kind: "confirm", preview }`
- Em modo `bypass`: deny rules ainda valem; **não há override total**
- Hierarquia de policy (`enterprise → user → project → session`) sempre respeitada; `locked` em enterprise **nunca** é overridable

### Failure semantics:

- Policy file corrupto / inválido → engine recusa carregar; harness recusa iniciar sessão (fail closed)
- Glob pattern com regex acidental (parênteses, etc) → erro de carga; sessão não inicia
- Decisão ambígua (allow + deny match no mesmo input) → **deny vence** (princípio least privilege)

### Side effects permitidos:
- Tool Registry: escreve em `tool_calls` (resultado posterior)
- Permission Engine: escreve em `approvals` (cada decisão)

### Versão: **v1**

---

## 10. Hooks Dispatcher ↔ Subsistemas

A: **Hooks Dispatcher**
B: **Subsistemas que disparam eventos** (Harness, Memory, Permission, Compaction, Session)
Direção: B → A (publica evento), A → B (devolve decisão se evento bloqueável)

### A garante a B:

- Para evento bloqueável: bloqueia até hooks terminarem (com timeout coletivo `max_hook_chain_ms = 15s`)
- Para não-bloqueável: retorna imediatamente; hooks rodam fire-and-forget
- Ordem de execução de hooks pra mesmo evento:
  1. enterprise hooks (em ordem de declaração)
  2. user hooks
  3. project hooks
- **Primeiro hook que retorna `block` interrompe a chain** em eventos bloqueáveis
- Schema do payload é versionado (`schema: "v1"` no JSON); subsistema sabe seu schema

### B garante a A:

- Publica evento **antes** da ação (para `Pre*`) ou **depois** (para `Post*`); não dispara em meio
- Não dispara `Pre*` se a ação já foi decidida em outro lugar (ex: `PreToolUse` não dispara se policy é `deny`)
- Payload contém **todo dado** necessário pra hook decidir; sem state hidden

### Failure semantics:

- Hook timeout coletivo → continuação assume "ninguém bloqueou" para eventos bloqueáveis; warning loggado
- Subsistema dispara evento mas dispatcher down (impossível em-process, mas em design SOA futuro) → fail closed se `fail_closed_on_dispatcher_down: true`

### Side effects permitidos:
- Dispatcher: escreve em `hook_runs`
- Subsystems: nenhum (via dispatcher)

### Versão: **v1**

---

## 11. Telemetry ↔ Subsistemas

A: **Telemetry**
B: **Todos os subsistemas que emitem spans**
Direção: B → A

### A garante a B:

- `tracer.span(name, attrs)` retorna handle em ≤ 1µs (no-op se telemetry desabilitada)
- Spans são **non-blocking** — emit é fire-and-forget no NDJSON
- Em modo `--no-telemetry`: emit vira no-op completo (zero overhead)
- Spans **não falham** — buffer cheio descarta old, não bloqueia caller

### B garante a A:

- Não inclui dados sensíveis em attrs (paths absolutos com username, tokens, conteúdo de `.env`)
- Sampling configurável; default = 100% (local-first é cheap)
- `attrs` são tipos primitivos ou strings ≤ 1KB

### Failure semantics:

- Disco cheio em escrita NDJSON → buffer in-memory cresce até `max_buffer_mb = 50`; depois descarta old com warning
- Subsistema emite attrs com PII → não-detectável automaticamente; eval de telemetria roda regex check em CI

### Side effects permitidos:
- Telemetry: escreve NDJSON, exporta OTEL se configurado
- Subsystems: chamam `tracer.span()` apenas

### Versão: **v1**

---

## 12. Tabela-resumo de versionamento

| Contrato | Versão | Compatibilidade |
|---|---|---|
| Tool ↔ Harness | v1 | breaking change requer bump SDK de tools |
| Hook ↔ Harness | v1 | schema do payload tem campo `schema` |
| Provider ↔ Context Engine | v1 | provider declara `protocol_version` em capabilities |
| Subagent ↔ Parent | v1 | binary version match obrigatório (mesmo binário) |
| Validator ↔ DAG Step | v1 | validators built-in + plugin API separada (v2+) |
| Memory ↔ Context Engine | v1 | frontmatter tem `version: 1` |
| Checkpoint ↔ FS | v1 | git ref namespace `refs/agent/v1/checkpoints/...` |
| Permission ↔ Tool Registry | v1 | policy YAML tem `version: 1` |
| Hooks Dispatcher ↔ Subsystems | v1 | event payload tem `schema: "v1"` |
| Telemetry ↔ Subsystems | v1 | OTEL semantic conventions seguidas onde aplicável |

Mudança em qualquer contrato:
1. Bump da versão local
2. Migration path documentado
3. Período de coexistência (suportar v1 e v2 por N releases)
4. Eval cobre ambas versões durante transição

---

## 13. O que **não** é contratual (intencionalmente)

Coisas que ficam de fora dos contratos por design:

- **Conteúdo dos prompts** — pode mudar a qualquer momento; eval cobre
- **Tool descriptions** — texto que o modelo lê; mudanças não quebram contrato (só métricas)
- **UI rendering** — Ink components são internos; mudam livremente
- **Schema interno do SQLite além das tabelas declaradas** — índices, triggers, vacuum strategy
- **Latência em microbenchmarks** — coberto em `PERFORMANCE.md`, não aqui
- **Heurísticas** (detection de injection, capability detection) — best-effort, sem garantia formal

---

## 14. Como verificar contratos

Cada contrato tem **eval de contrato** dedicado:

```yaml
# evals/contracts/tool_harness.yaml
contract: tool_harness_v1
checks:
  - name: tool_receives_validated_args
    fixture: invalid_args
    expect: harness_rejects_before_invocation
  - name: tool_respects_abort_signal
    fixture: tool_loop_5s
    expect: aborts_within_2s_after_signal
  - name: tool_writes_with_writes_flag_creates_checkpoint
    fixture: write_file_call
    expect: checkpoint_id_in_step
```

Eval roda em CI. Falha = breaking change não-anunciada = PR bloqueado.

---

## 15. Insight final

Contrato bom não é o que **descreve** comportamento. É o que **constraini** comportamento para que duas implementações independentes interoperem.

Spec sem contrato é descrição de como **uma** implementação funciona. Spec com contrato é o **protocolo** que múltiplas implementações respeitam.

Esses contratos não são exaustivos — são o **mínimo necessário** para que o projeto não vire integration nightmare. Acrescentar contratos é fácil; remover é breaking change.
