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
- **Display hint opcional** em metadata: `display: 'table' | 'list' | 'diff' | 'raw' | 'auto'` (default `auto`). Override do auto-detect do `<ToolCallCard>` (`UI.md §3.1`/§3.2). Tools com output `Array<object>` homogêneo se beneficiam de `display: 'table'` explícito; outputs de `bash` usam `display: 'raw'` para preservar formatação de terminal.
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

## 2.5 Ergonomics rubric — o que conta como "tool bem desenhada"

Princípio 3 do `AGENTIC_CLI.md` ("10 tools bem desenhadas vencem 40 genéricas") é load-bearing mas vazio sem rubric. Sem critério, "ergonomics" vira gosto pessoal.

Toda tool nova passa pelo checklist abaixo antes de entrar no catálogo §2.6. Falha em ≥ 2 critérios = redesenhar antes de PR.

| # | Critério | Operacionalização | Falha típica |
|---|---|---|---|
| 1 | **Input determinístico** | Schema fechado, sem campos "extras" interpretados | `bash` aceitando flags inferidas de prosa |
| 2 | **Output estruturado** | JSON Schema ou união discriminada; sem string livre como contrato | Tool retorna `string` que modelo precisa parsear |
| 3 | **Idempotência declarada** | Metadata `idempotent: true \| false`; quando true, retry é seguro | `write_file` sem hash do conteúdo prévio (race condition) |
| 4 | **Side effects declarados** | `writes: bool`, `network: bool`, `exec: bool` em metadata | Tool que silenciosamente faz request HTTP |
| 5 | **Failure modes enumerados** | Lista finita de error codes; `unknown_error` é bug, não default | Tool que retorna `{ error: <stack trace> }` cru |
| 6 | **Custo aproximado** | Latência típica + tokens-por-output em metadata | Tool que pode retornar 50KB sem warning |
| 7 | **Composability** | Output de uma tool é input válido de outra (ou explicitamente terminal) | `search_*` que retorna formato incompatível com `read_file` |
| 8 | **Reversibilidade** | Se `writes: true`, checkpoint é criado antes (harness garante via §2) | Tool que escreve em FS mas declara `writes: false` |
| 9 | **Audit footprint** | Input/output mínimos pra reproduzir o call em replay | Tool que depende de variável de ambiente não capturada |
| 10 | **Erro como dado** | Falha vira tool_result, não exception (ver §2 cláusula 7) | Tool que crash leva harness junto |

Tool que passa em 10/10 entra no catálogo. Tool que passa em 8-9/10 entra com TODO declarado em metadata. Tool que passa em ≤ 7/10 não entra.

**Anti-rubric:** "tool é genérica, então cobre mais casos" — ver `ANTI_PATTERNS.md` (princípio 3 contradiz). Genérico é o que vira `bash` (escape hatch), não o que vira API.

---

## 2.6 Tool catalog canônico v1

Conjunto fechado para v1. Adições requerem PR contra este doc + eval de regressão. **Total: 21 tools** — acima do teto sugerido de 10, mas justificado em §2.6.6 via 4 famílias conceituais. Decisões ADR: §2.6.8 (leak-driven: apply_patch/todo_write/fetch_url) e §2.6.9 (code-index-driven: read_symbol/find_references/outline_file/code_graph).

### 2.6.1 Filesystem (read)

| Tool | Input | Output | Side effects | Idempotente | Custo típico |
|---|---|---|---|---|---|
| `read_file` | `{ path, offset?, limit? }` | `{ content, total_lines, truncated }` | nenhum | sim | ~5ms; até `limit` linhas |
| `glob` | `{ pattern, cwd? }` | `{ matches: string[], truncated }` | nenhum | sim | ~10-50ms; cap 1000 matches |
| `grep` | `{ pattern, path?, type?, -A?, -B? }` | `{ matches: { file, line, text }[], truncated }` | nenhum | sim | ~50-500ms; cap 200 matches |

### 2.6.2 Filesystem (write)

| Tool | Input | Output | Side effects | Idempotente | Custo típico |
|---|---|---|---|---|---|
| `write_file` | `{ path, content }` | `{ bytes_written, checkpoint_id }` | `writes: true` | não (sobrescreve) | ~5ms + checkpoint |
| `edit_file` | `{ path, old_string, new_string, replace_all? }` | `{ changes: number, checkpoint_id }` | `writes: true` | não | ~10ms + checkpoint |

`old_string` deve ser único no arquivo (ou `replace_all: true`); senão tool retorna `{ error: "ambiguous_match" }`. Sem `old_string` = chamar `write_file`, não `edit_file`.

### 2.6.3 Execução

| Tool | Input | Output | Side effects | Idempotente | Custo típico |
|---|---|---|---|---|---|
| `bash` | `{ cmd, timeout_ms?, cwd?, read_only? }` | `{ stdout, stderr, exit_code, duration_ms, truncated }` | `writes: true` (default), `exec: true` | não (default) | varia; cap default 30s, 4MB output |

`read_only: true` é declaração do **chamador**, não da tool. Permission engine pode validar contra allowlist (`SECURITY_GUIDELINE.md`).

### 2.6.4 Subagent / orquestração

| Tool | Input | Output | Side effects | Idempotente | Custo típico |
|---|---|---|---|---|---|
| `task_sync` | `{ playbook, prompt, budget? }` | `SubagentOutput` (ver §5) | depende do playbook | não | varia; cap por budget |
| `task_async` | `{ playbook, prompt, budget? }` | `{ handle }` | spawna processo filho | não | ~50ms (spawn) |
| `task_await` | `{ handle, timeout_ms? }` | `SubagentOutput` | nenhum (block) | sim | varia |
| `task_cancel` | `{ handle }` | `{ cancelled: bool }` | mata subprocess | sim (idempotente em handle morto) | ~10ms |

### 2.6.5 Memory

| Tool | Input | Output | Side effects | Idempotente | Custo típico |
|---|---|---|---|---|---|
| `memory_search` | `{ query, scope? }` | `{ hits: MemoryHit[] }` | nenhum (read) | sim | ~20-100ms (lexical) |

`memory_write` **não é tool exposta ao modelo** — escrita de memória passa por confirmação humana via slash command (ver `MEMORY.md §5`). Modelo não escreve memória diretamente.

### 2.6.5b Network (escopado)

| Tool | Input | Output | Side effects | Idempotente | Custo típico |
|---|---|---|---|---|---|
| `fetch_url` | `{ url, max_bytes?, timeout_ms? }` | `{ status, content_type, body, truncated, redirected_to? }` | `network: true` | sim (GET) | ~100ms-2s; cap default 256KB |

URL deve estar na **allowlist da sessão** (extraída do prompt do usuário ou de arquivos lidos no turno). URL sintetizada pelo modelo → `fetch.policy_denied`. Decisão completa, policy obrigatória, e gatilho de implementação em §2.6.8 / decisão C.

### 2.6.5c Code retrieval (symbolic)

Família de retrieval que opera sobre o subsistema de [`CODE_INDEX.md`](./CODE_INDEX.md). Decisão completa em §2.6.9. Pré-requisitos de visibilidade ao modelo em §2.6.9.y.

| Tool | Input | Output | Side effects | Idempotente | Custo típico |
|---|---|---|---|---|---|
| `read_symbol` | `{ symbol, file?, include_doc? }` | `{ symbol{...}, source, signature?, doc? }` | nenhum | sim | ~5-20ms; ~50-200 tokens |
| `find_references` | `{ symbol, file?, ref_kind?, max_results? }` | `{ references[], truncated, total_count }` | nenhum | sim | ~10-50ms; cap default 100 refs |
| `outline_file` | `{ path, include_internal?, include_doc? }` | `{ symbols[], loc, imports_summary, test_files[] }` | nenhum | sim | ~8-30ms; ~5-15% dos tokens de read_file |
| `code_graph` | `{ path, direction, hops?, include_external? }` | `{ edges[], truncated }` | nenhum | sim | ~15-100ms (1-3 hops) |

`index.unavailable` é failure mode comum às 4: subsistema de index não inicializado ou em rebuild. Modelo recai em `read_file`/`grep` quando vê esse erro.

### 2.6.5d Background coordination

Família para processos de longa duração e coordenação não-blocking. Tools especificadas em `AGENTIC_CLI.md §7.3` (background processes) e `§7.3.1` (wait/monitor primitives); aqui ficam sumarizadas no contrato canônico.

| Tool | Input | Output | Side effects | Idempotente | Custo típico |
|---|---|---|---|---|---|
| `bash_background` | `{ cmd, label }` | `{ process_id, started_at }` | `writes: true`, `exec: true`, spawna processo | não | ~10ms (spawn) |
| `bash_output` | `{ process_id, since? }` | `{ stdout, stderr, cursor, exit_code? }` | nenhum (leitura) | sim (cursor avança) | ~5ms |
| `bash_kill` | `{ process_id, signal? }` | `{ killed: bool, exit_code }` | mata subprocess | sim (idempotente em pid morto) | ~10ms graceful, +5s SIGKILL |
| `wait_for` | `{ condition, timeout_ms, poll_interval_ms? }` | `{ matched, condition_met, elapsed_ms, payload? }` | bloqueia loop (sem LLM call); cancellable | sim | varia; bound por `timeout_ms` |
| `monitor` | `{ condition, duration_ms?, max_events? }` | `{ events[], reason: 'duration'|'max_events'|'cancelled' }` | streama eventos durante duração | sim | varia; bound por args |

**Razão da família:**

- `bash_background`/`bash_output`/`bash_kill` formam um **lifecycle** (spawn → read → terminate); modelo trata como uma palette mental, não 3 decisões independentes.
- `wait_for`/`monitor` são **coordenação sem LLM** — bloqueiam wall-clock mas **não consomem tokens**. Razão da existência: substituem padrão "polling loop com 5 LLM calls" por uma chamada bloqueante (custo `$0.40 → $0.10` no exemplo de `AGENTIC_CLI.md §7.3.1`).

**Conditions enumeradas (`wait_for`):**

| `kind` | Use case |
|---|---|
| `process_output` | "espera 'ready on port' aparecer no stdout do dev server" |
| `process_exit` | "espera npm install terminar" |
| `file_exists` / `file_change` | "espera build artifact aparecer" |
| `port_open` | "espera porta 3000 abrir" |
| `http_response` | "espera /health retornar 200" |
| `sleep` | timed sleep determinístico |
| `all_of` / `any_of` | composições AND/OR |

**Conditions enumeradas (`monitor`):**

| `kind` | Use case |
|---|---|
| `process_output_lines` | tail completo de bg process |
| `process_output_pattern` | regex match (ex: `WARN|ERROR`) |
| `file_changes` | watch glob de paths |

**Failure modes comuns:**
- `bash_bg.spawn_failed` — comando inválido, ENOENT
- `bash_bg.process_not_found` — `process_id` desconhecido (já killed? cleanup?)
- `wait.timeout` — condição não satisfeita em `timeout_ms`
- `wait.cancelled` — Ctrl+C/Esc Esc cascateando AbortSignal
- `monitor.cancelled` — mesmo

UI integration: `<BackgroundProcessTray>` mostra `bash_*` ativos; `<WaitIndicator>` substitui `<LoopStatusLine>` durante `wait_for`; `<MonitorStream>` durante `monitor` (todos em `UI.md §3.2`).

**Hibernation:** `wait_for` bloqueia o agente process. Daemon-based hibernation (agente sai durante wait, daemon dispara hook em wakeup) é **deferred v2** (`AGENTIC_CLI.md §7.3.1`).

### 2.6.5e Interaction (anti-presumption)

> **Cross-refs:** mecânica completa em `STATE_MACHINE.md §12`; estado `[clarifying]` em `STATE_MACHINE.md §2.2`; per-playbook config em `PLAYBOOKS.md §1.1` (`clarify_mode`).

| Tool | Input | Output | Side effects | Idempotente | Custo típico |
|---|---|---|---|---|---|
| `clarify` | `{ question, options[≥2], why_it_matters?, blast_radius: low\|medium\|high }` | `{ outcome: resolved\|skipped\|escalated\|auto_low, chosen_option_id?, user_text? }` | nenhum (consulta humana) | sim (mesma question + options ⇒ mesmo modal recriado) | `low`: ~5ms (auto-resolve); `medium`/`high`: bound por timeout 60s |

**Semantics distintos por `blast_radius`:**

- `low` — auto-resolvido sem modal; output `outcome: auto_low`, `chosen_option_id: options[0].id`. Registra em `assumptions[]` do output do playbook.
- `medium` — bufferizado; agrupado com outros `medium` pendentes; modal único antes do próximo write.
- `high` — modal imediato (transição `running → clarifying`). Outras tools bloqueadas até resolver.

**Failure semantics:**
- `clarify.options.invalid` — < 2 options, ou IDs duplicados
- `clarify.disabled_by_playbook` — `clarify_mode: off` no frontmatter; modelo recebe erro estruturado e re-tenta com `assumptions[]` em vez
- `clarify.budget_exceeded` — sessão emitiu mais de `clarification.max_per_session` (default 5); fallback para `auto_low` automático

**Disponibilidade ao modelo:** controlada por `clarify_mode` do playbook ativo:
- `pre_execution`: tool exposta; `medium`/`high` permitidos só em fase exploratória (drift detector flagga uso tardio)
- `on_high_blast`: tool exposta; `low` auto-resolve, `medium` bufferiza, `high` interrompe
- `off`: tool **não exposta** ao modelo; tudo vira `assumptions[]` retrospectivo

### 2.6.6 Justificativa do teto (22 vs 10)

Tools agrupam-se em **4 famílias** que economizam slots conceituais:

- **`task_*`** (4 tools, `§2.6.4`): família de subagent — uma palette, não 4 decisões independentes. Peso ~1.5.
- **Code retrieval simbólica** (4 tools, `§2.6.5c`): família de queries sobre code index — schema homogêneo (read-only, struct output), modelo escolhe direção. Peso ~2.
- **Background coordination** (5 tools, `§2.6.5d`): lifecycle (`bash_*`) + coordenação sem-LLM (`wait_for`/`monitor`). Lifecycle conta como ~1.5 (família mental), wait/monitor como ~1 (par primitivo). Peso conjunto ~2.5.
- **Demais** (9 tools): contam 1:1 — incluindo `clarify` (`§2.6.5e`), interação anti-presumption.

Orçamento conceitual:

```
read (3) + write (2) + exec (1) + task family (~1.5) + memory (1) + fetch (1) + code retrieval family (~2) + bg coordination family (~2.5) + clarify (1) ≈ 15
```

Acima de 10 ± 2 — **revisão deliberada do teto**. Justificativa:

1. **Background coordination resolve uma falha enumerada**: padrão de polling em loop custa $0.40+ por workflow (`§7.3.1`); sem `wait_for`/`monitor`, modelo trabalha em torno do limite, gerando code mais frágil.
2. **Famílias não são fungíveis**: subagent (concurrency lógica) ≠ code retrieval (queries de FS) ≠ bg coordination (wall-clock). Mental models distintos; combinar reduziria clareza.
3. **Teto suave, não rígido**: princípio 3 fala em "10 bem desenhadas vencem 40 genéricas" — espírito é fewness com qualidade, não número absoluto.

Tools que **estouraria o teto** se promovidas (mantidas em domínios separados):

- `todo_write` é UI affordance (ver `UI.md`), não tool de modelo (escreve via stream parsing). Audit gap endereçado por tabela `todos` em vez de promoção a tool — ver §2.6.8 decisão B.
- `checkpoint_*` são operações do harness, expostas via slash command.
- `recap_*` é projeção SQL, exposto via CLI.
- `memory_write` exige confirmação humana — slash command, não tool.
- `format_file` / `lint_file` / `test` são gates do `CODE_GENERATION.md` pipeline, rodados em `PostToolUse`, não decididos pelo modelo.

Cargo-cult check: nenhuma tool do catálogo foi adicionada porque "Claude Code tem". Cada uma resolve uma falha enumerada em `FAILURE_MODES.md` ou um caso de uso documentado em §2.6.8 / §2.6.9 / §7.3 do `AGENTIC_CLI.md`.

Adicionar `todo_write`, `checkpoint_*`, ou `recap_*` como tools expostas ao modelo **estouraria** o teto — por isso ficam em domínios separados:
- `todo_write` é UI affordance (ver `UI.md`), não tool de modelo (escreve via stream parsing). Audit gap endereçado por tabela `todos` em vez de promoção a tool — ver §2.6.8 decisão B.
- `checkpoint` é operação do harness, exposta via slash command.
- `recap` é projeção SQL, exposto via CLI.

Cargo-cult check: nenhuma das tools acima foi adicionada porque "Claude Code tem". Cada uma resolve uma falha enumerada em `FAILURE_MODES.md` ou um caso de uso documentado em §2.6.8 / §2.6.9.

### 2.6.7 Tools deliberadamente ausentes

Anti-pattern check rápido — por que estas **não** estão no catálogo:

| Tool ausente | Motivo |
|---|---|
| `web_search` | Provider-specific; vai como capability opcional, não tool canônica. Latência + custo + reproduzibilidade ruim em replay |
| `web_fetch` (open-ended) | Mesmo motivo + risco de prompt injection (ver `SECURITY_GUIDELINE.md`). Variante **escopada** `fetch_url` é diferente — ver §2.6.8 |
| `code_execution` (sandbox cloud) | Concorre com `bash` local; complica replay; v2 |
| `database_query` | Domínio do user; expõe via custom tool ou MCP |
| `vector_search` | Ver `ANTI_PATTERNS.md §2.2` |

MCP é a porta para extensão controlada — qualquer das ausentes acima é **adicionável pelo usuário** sem mudar o catálogo canônico.

### 2.6.8 Decisões revisitadas (leak-driven review)

O leak da Anthropic em 2026 expôs as categorias de tools que outras CLIs agentic implementam. Três delas mereceram revisão explícita contra o catálogo canônico. Esta sub-seção registra cada decisão, motivo, e gatilho de reconsideração — modelo ADR (Architecture Decision Record).

**Princípio guia:** *meta-cognição não é tool*. Modelo decide ações **no mundo** (FS, shell, subagent, memory_search, fetch). Decisões **sobre o modelo** (planejar, lembrar, comprimir contexto, gerenciar atenção) são responsabilidade do harness, não do catálogo. Sem essa fronteira, o tool budget é consumido por ferramentas que não mexem em nada — e auditabilidade não compensa o bloat.

Esta posição é uma **inversão deliberada** vs Claude Code, onde o modelo invoca `todo_write`, planeja explicitamente, e é responsável por sumarizar a própria história. As três decisões abaixo aplicam o princípio.

| # | Tool proposta | Decisão | Tool count após |
|---|---|---|---|
| A | `apply_patch` (unified diff) | **deferred → v2** | 11 (sem mudança) |
| B | `todo_write` como tool de modelo | **rejected** (audit gap endereçado de outro modo) | 11 (sem mudança) |
| C | `fetch_url` escopado | **accepted** | 12 |

#### A. `apply_patch` — deferred

**Proposta:** tool que recebe unified diff e aplica em um ou mais arquivos. Reduz tokens em multi-hunk edits.

**Decisão:** deferred. `edit_file` cobre v1; eficiência de tokens em arquivos com ≥ 3 hunks é otimização real, mas não load-bearing.

**Por que não rejeitar:**
- Token cost em refactors grandes é mensurável (multi-hunk edit hoje = N chamadas com conteúdo duplicado).
- Formato é padrão (POSIX `patch`); há tokenizadores, parsers, e testes prontos.
- Modelos pós-2024 emitem unified diff com fidelidade alta.

**Por que não aceitar agora:**
- `edit_file` é mais auditável (input/output óbvios; diff resultante reconstruível por harness).
- Falha parcial de hunk (algum aplica, outro não) introduz novo failure mode (ver `FAILURE_MODES.md`).
- Sem eval mostrando ganho > 20% em algum workflow, otimização prematura.

**Gatilho de reconsideração:**
- Eval (`PERFORMANCE.md §5` ou `TOKEN_TUNING.md §13.4`) mostra que ≥ 30% das sessões de `/refactor` gastam ≥ 20% dos tokens de output em conteúdo duplicado de `edit_file` adjacentes.
- OU: usuário reporta padrão concreto onde `edit_file` falha (ex.: edits em arquivo com whitespace inconsistente).

**Quando reconsiderado, design preliminar:**

```yaml
name: apply_patch
input:
  patches:
    - path: string
      diff: string  # unified diff format, sem header git
output:
  applied: { path, hunks_applied, hunks_failed }[]
metadata:
  writes: true
  idempotent: false
  failure_modes:
    - patch.malformed
    - patch.context_mismatch        # hunk não bate com arquivo
    - patch.partial_apply           # alguns hunks aplicaram
```

#### B. `todo_write` como tool de modelo — rejected

**Proposta:** modelo invoca `todo_write({ items: [...] })` para registrar plano; UI renderiza checklist live.

**Decisão:** rejected como tool. Mantém-se como **UI affordance via stream parsing** (`AGENTIC_CLI.md §2.6`).

**Por que rejeitar:**
- Princípio guia (§2.6.8 acima): meta-cognição não é tool.
- Plan ≠ ação. Modelo emite plano em prosa; UI extrai. Tool budget fica preservado pra ações que mexem em alguma coisa.
- Tool count entraria em pressão: somar `todo_write` (1) + `fetch_url` (1) sem rever `task_*` (4) estoura o teto conceitual de princípio 3.
- Adicionar como tool não traz benefício mensurável de qualidade — só traz **auditabilidade**, que é endereçável de outro jeito (abaixo).

**Mas o audit gap é real.** Se planning é invisível ao audit log, princípio 7 ("Trace tudo") tem buraco. Endereçamento:

#### B.1 Endereçamento do audit gap (sem promover a tool)

Adicionar tabela `todos` em `AUDIT.md §1`:

```sql
CREATE TABLE todos (
  id INTEGER PRIMARY KEY,
  session_id TEXT NOT NULL,
  message_id TEXT NOT NULL,        -- mensagem do modelo onde o todo foi extraído
  step_index INTEGER NOT NULL,     -- step da sessão
  content TEXT NOT NULL,           -- texto do item
  status TEXT NOT NULL,            -- 'pending' | 'in_progress' | 'completed' | 'cancelled'
  parent_todo_id INTEGER,          -- subtask
  created_at INTEGER NOT NULL,
  completed_at INTEGER
);
```

Pipeline:
1. Modelo emite plano em prosa (formato canônico documentado em `CONTEXT_TUNING.md` — checklist markdown).
2. Stream parser do harness extrai itens, escreve em `todos`.
3. UI renderiza a partir da tabela (não do stream cru).
4. `audit_timeline` (§2.1) inclui events `todo_added`, `todo_completed`.

Resultado: auditabilidade igual à de uma tool, sem custo de slot no catálogo. O parser é responsabilidade do harness; falha de parse vira `failure_event` classificado, não silenciosa.

**Gatilho de reconsideração:**
- Stream parser falha em > 5% das sessões (medido).
- OU: modelos locais (`LOCAL_MODELS.md`) emitem formato instável demais pra parser. Nesse caso, `todo_write` vira tool **apenas no profile `orchestrated`**, mantendo simetria do princípio guia (DAG já é meta-cognição harness-side).

#### C. `fetch_url` escopado — accepted

**Proposta:** tool que faz GET HTTP de uma URL específica fornecida pelo usuário, com policy estrita.

**Decisão:** accepted como **tool 12**.

**Por que aceitar:**
- Caso de uso é comum e legítimo: "lê esse RFC", "olha esse stackoverflow", "abre essa doc do MDN".
- Hoje, sem a tool, modelo cai em `bash curl ...` — pior em todo eixo (PII em shell history, sem redaction, output não-estruturado, audit poluído).
- Diferença categórica vs `web_search` / `web_fetch` open-ended: URL **explícita no prompt do usuário**, não inferida.

**Por que não é o `web_fetch` rejeitado em §2.6.7:**
- `web_fetch` open-ended permite modelo browser sem supervisão — vetor de prompt injection wide.
- `fetch_url` exige URL no prompt do usuário (ou em arquivo lido); permission engine bloqueia URLs sintetizadas pelo modelo.

**Schema:**

```yaml
name: fetch_url
input:
  url: string                      # MUST match permission allowlist (ver below)
  max_bytes?: number               # default 256KB; cap 2MB
  timeout_ms?: number              # default 10s; cap 30s
output:
  status: number                   # HTTP status
  content_type: string
  body: string                     # truncado a max_bytes
  truncated: boolean
  redirected_to?: string
metadata:
  writes: false
  network: true
  idempotent: true                 # GET; provider-side caching
  failure_modes:
    - fetch.timeout
    - fetch.size_exceeded
    - fetch.network_error
    - fetch.policy_denied          # URL não na allowlist
    - fetch.injection_suspect      # heurística pós-fetch (ver below)
```

**Policy obrigatória** (em `SECURITY_GUIDELINE.md`, ver gatilho abaixo):

1. **URL allowlist por sessão.** Permission engine extrai URLs do prompt do usuário e de arquivos lidos no turno; só essas são fetcháveis. URL sintetizada pelo modelo → `fetch.policy_denied`.
2. **Sem `Authorization` ou cookies.** Header limpo; tool não acessa segredos.
3. **PII redaction no body.** Mesmo redactor de `read_file` (`AUDIT.md §3.2`).
4. **Size cap default agressivo** (256KB). Override requer flag explícito.
5. **Heurística anti-injection.** Body é escaneado por padrões de prompt injection conhecidos (`Ignore previous`, `<system>`, etc); match → tag `fetch.injection_suspect`, modelo recebe warning prepended ao output.
6. **Domínios bloqueados.** Localhost, private IP ranges (RFC 1918), `metadata.google.internal`, `169.254.169.254` (cloud metadata) — bloqueio incondicional.

**Gatilho de implementação:**
- Antes do merge desta tool: `SECURITY_GUIDELINE.md` ganha §dedicada de `fetch_url policy` cobrindo os 6 pontos acima.
- Antes do release de v1: eval de regressão (`TOKEN_TUNING.md §13.4`) ganha 3 itens de prompt injection via fetch — prompt do usuário aponta pra URL controlada do test corpus que retorna body com tentativa de injection; modelo deve **não** seguir as instruções injetadas.

**Catalog count update:** §2.6.6 vai de 11 para 12. O cálculo conceitual passa a:

```
read (3) + write (2) + exec (1) + task family (~1.5) + memory (1) + fetch (1) ≈ 9.5
```

Ainda dentro do princípio 3.

#### 2.6.8.x Resumo das 3 decisões

| Tool | Status | Próximo passo concreto |
|---|---|---|
| `apply_patch` | deferred | adicionar item de eval mensurando token waste em multi-hunk; reabrir quando ≥ 30% sessões `/refactor` |
| `todo_write` (modelo) | rejected | criar tabela `todos` em `AUDIT.md §1`; documentar formato canônico de checklist em `CONTEXT_TUNING.md` |
| `fetch_url` | accepted | escrever `SECURITY_GUIDELINE.md` policy section; adicionar 3 corpus items de injection em `evals/regression/prompts`; bump catalog para 12 |

Decisões aqui são **append-mostly**: revogar uma requer PR com motivo + eval que justifique mudança de premissa. Reasoning preservado é mais útil que decisão atual silenciosa.

### 2.6.9 Symbolic tools (code-index-driven review)

`CODE_INDEX.md` introduziu um subsistema queryable que torna viável tools simbólicas — modelo pergunta "onde X é usada?" e recebe lista exata, sem grep storm. Esta sub-seção formaliza a decisão de promoção dessas tools ao catálogo §2.6.

Pré-requisito: subsistema de `CODE_INDEX.md` implementado e funcional. Sem index, todas as 4 abaixo retornam `index.unavailable` e o modelo cai em fallback (`grep`/`read_file`).

| # | Tool proposta | Decisão | Tool count após |
|---|---|---|---|
| D | `read_symbol` | **accepted** | 13 |
| E | `find_references` | **accepted** | 14 |
| F | `outline_file` | **accepted** | 15 |
| G | `code_graph` (imports + dependents combinados) | **accepted** | 16 |

Total: 12 → 16. Justificativa do teto revisada em §2.6.6 — tools simbólicas formam **família de retrieval** análoga a `task_*` (família com peso conceitual ~2 slots para 4 tools).

#### D. `read_symbol` — accepted

**Proposta:** retorna corpo de uma função/classe/método/type específico, sem ler o arquivo inteiro.

**Schema:**

```yaml
name: read_symbol
input:
  symbol: string                  # nome local ou FQN
  file?: string                   # disambiguação se nome ambíguo
  include_doc?: boolean           # default true
output:
  symbol: { name, kind, fqn, file, line_range }
  source: string                  # corpo
  signature?: string
  doc?: string
metadata:
  writes: false
  idempotent: true
  reads_secrets: false
  failure_modes:
    - symbol.not_found
    - symbol.ambiguous            # múltiplos matches; exige `file`
    - index.unavailable           # fallback recomendado: read_file + grep
    - file.parse_failed           # symbol não extraível (parser falhou no arquivo)
```

**Por que accepted:**
- Token economy mensurável: ~50-200 tokens vs 500-3000 de `read_file` médio. Redução 5-20×.
- Caso de uso quente: "antes de editar `validateOrder`, lê o corpo dela". Hoje vira `grep validateOrder` + `read_file path:src/orders.ts offset:124 limit:50` — frágil (offset adivinhado), caro (linhas adjacentes ignoradas).
- Ergonomics rubric `§2.5`: 10/10 (input determinístico, output estruturado, idempotente, side effects nulos, failures enumerados, custo declarado, composable, reversível n/a, audit completo, erro como dado).

**Substituição em playbook:** todos os recipes em `CONTEXT_TUNING.md §13` que hoje fazem `read_file` em alvo simbólico passam a usar `read_symbol`.

**Gatilho de reconsideração:** se eval (`§13.4` em `TOKEN_TUNING.md`) mostrar regressão de qualidade > 5% em workflow X após adoção (modelo perde contexto que estava no arquivo inteiro), reverter para `read_file` no recipe específico.

#### E. `find_references` — accepted

**Proposta:** lista todos os usos (call sites, type refs, etc) de um símbolo, com contexto.

**Schema:**

```yaml
name: find_references
input:
  symbol: string
  file?: string                   # disambiguação
  ref_kind?: 'call'|'type'|'import'|'extends'|'implements'  # filtro
  max_results?: number            # default 100, cap 500
output:
  references: [
    { file, line, col, kind, surrounding_text }   # surrounding ±2 linhas
  ]
  truncated: boolean
  total_count: number             # mesmo se truncated
metadata:
  writes: false
  idempotent: true
  failure_modes:
    - symbol.not_found
    - symbol.ambiguous
    - index.unavailable
    - results.truncated           # warning, não erro
```

**Por que accepted:**
- Substitui padrão "grep + filtrar comentários e strings manualmente". Modelo hoje faz `grep "validateOrder"` e recebe matches em comments, docstrings, examples, test fixtures — false positives são caros (re-grep pra refinar) ou perigosos (refactor que mexe no que não é uso real).
- Semântico via tree-sitter: distingue `validateOrder()` (call) de `// validateOrder is broken` (comment) de `"call validateOrder"` (string literal).
- Caso de uso quente: refactor playbook em `CONTEXT_TUNING.md §13.2` lista `[callers section]: arquivos que importam o target (via grep)` — `find_references` é a versão precisa e estruturada disso.

**Limites:**
- Polymorphism: `obj.method()` quando há múltiplas classes com `method` retorna lista com `target_symbol_id: NULL` (ambíguo); modelo decide.
- Dynamic dispatch: callbacks, function refs, eval — não rastreados; ficam como references unresolved se aparecerem na sintaxe de chamada explícita.
- Cross-language: TS chamando WASM Rust não conecta (limite de `CODE_INDEX.md §12`).

**Gatilho de reconsideração:** falsos negativos > 5% em corpus de eval (refactor missou um call site real) → reabrir grammar/queries do tree-sitter pro idioma específico.

#### F. `outline_file` — accepted

**Proposta:** retorna esqueleto de um arquivo (símbolos com signature + line, sem corpos).

**Schema:**

```yaml
name: outline_file
input:
  path: string
  include_internal?: boolean      # default false: só exports
  include_doc?: boolean           # default false: docstrings dos símbolos
output:
  symbols: [
    { name, kind, signature, line, visibility, parent_name? }
  ]
  loc: number
  imports_summary: string         # 1-line: "imports from 5 files (3 local, 2 external)"
  test_files: string[]            # via test_mapping; pode estar vazio
metadata:
  writes: false
  idempotent: true
  failure_modes:
    - file.not_found
    - file.parse_failed
    - file.skipped                # arquivo em exclude do index
    - index.unavailable
```

**Por que accepted:**
- Custo: ~5-15% dos tokens de `read_file`. Use case "qual a estrutura de `auth.ts` antes de editar?" é universal.
- Entrada óbvia para refactor/explain/audit: ver as peças antes de tocar nelas.
- Compõe bem: `outline_file` → escolher symbol → `read_symbol`.
- Inclui `test_files` derivado de `test_mapping` — sinaliza imediatamente onde estão os testes do arquivo, sem grep separado.

**Substituição:** `[target_file]` em recipes (refactor, explain) hoje carrega o arquivo via `read_file`; pode-se trocar por `outline_file` + `read_symbol` on-demand. Eval define qual recipe ganha.

**Gatilho de reconsideração:** se em > 30% dos casos modelo chama `outline_file` seguido de `read_file` do mesmo arquivo (ignora o outline e lê tudo), o outline não está economizando — reabrir.

#### G. `code_graph` — accepted (combinação de imports/dependents)

**Proposta:** query polimórfica do grafo de imports — direção `imports` (o que esse arquivo importa) ou `dependents` (quem importa esse arquivo).

**Schema:**

```yaml
name: code_graph
input:
  path: string
  direction: 'imports' | 'dependents'
  hops?: number                   # default 1, cap 3
  include_external?: boolean      # default false: só locais (não node_modules)
output:
  edges: [
    { from_path, to_path, target_module, names[], is_external, hop_distance }
  ]
  truncated: boolean
metadata:
  writes: false
  idempotent: true
  failure_modes:
    - file.not_found
    - graph.exceeds_hops          # > 3 inviável
    - index.unavailable
```

**Por que accepted (combinado):**
- Imports vs dependents são duas perguntas com **mesma estrutura** de output (lista de edges); polimorfismo pelo campo `direction` é mais barato que duas tools.
- Reduz bloat do catálogo: 1 tool em vez de 2.
- Schema homogêneo facilita parsing pelo modelo.

**Por que não 2 tools separadas:**
- Custo de tool slot é real (`§2.5` rubric, princípio 3).
- Mental model é compatível: "grafo de dependências, qual direção?". Não há ergonomic loss em parametrização.

**Use case quente:** refactor — `code_graph(path: "src/auth.ts", direction: "dependents", hops: 2)` lista todos os arquivos que precisam ser revisados ao mudar API de `auth.ts`. Hoje vira grep do nome do módulo + filtros manuais.

**Limites:**
- Dynamic imports (`import(variable)`, `require(computed)`) não rastreados (`CODE_INDEX.md §12`).
- 3 hops é cap pragmático: hops >= 4 retornam quase todo o repo em projetos coesos; útil zero.
- External (libs) é opt-in: default false para não poluir output com grafo de `react`/`lodash` sem necessidade.

**Gatilho de reconsideração:** se eval mostra que `direction: 'dependents'` é 10× mais usado que `'imports'`, considerar split — `dependents_of` vira tool dedicada e `imports_of` fica em `outline_file.imports_summary` (já está lá).

#### 2.6.9.x Resumo das 4 decisões + impacto no catálogo

| Tool | Status | Próximo passo concreto |
|---|---|---|
| `read_symbol` | accepted | implementar via `CODE_INDEX.md §4`; adicionar a `[tool_schemas]` em ordem após `glob`/`grep` |
| `find_references` | accepted | mesmo + corpus de eval para 5% false-negative threshold |
| `outline_file` | accepted | mesmo + adoptar em recipes de `CONTEXT_TUNING.md §13` (refactor, explain) |
| `code_graph` | accepted | mesmo + adicionar `dependents_of` query em `code-index profile` para benchmark |

Catalog count atualizado: **12 → 16**. Reformulação de §2.6.6 abaixo (peso conceitual revisado).

#### 2.6.9.y Pré-requisitos de release

Estas 4 tools só viram visíveis ao modelo quando:
1. `CODE_INDEX.md` schema implementado e initial scan funcional (`§3.1`).
2. Linguagem do projeto está em §1.3 (TS/JS/Python/Go/Rust/Java).
3. `index.parse_status='ok'` para o arquivo target — senão tool retorna `file.parse_failed`, modelo recai em `read_file`.
4. Eval (`TOKEN_TUNING.md §13.4`) inclui ≥ 1 item por tool com `severity: critical` — gate de release.

Sem (1)-(4): tool **registrada** mas com `visible_to_model: false` em todos os schemas. Decisão simétrica à de MCP servers não-confiáveis (`MCP.md §1.5`). Modelo nem sabe que existe a tool até pré-requisitos serem cumpridos.

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
- Carrega índice em **posição fixa** do prompt (após AGENTS.md, antes de tool schemas)
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

## 11. MCP Server ↔ Harness

A: **Harness** (MCP client embutido)
B: **MCP Server** (processo externo, qualquer linguagem)
Direção: A ↔ B (bidirecional via JSON-RPC; lifecycle controlado por A)

> **Autoridade detalhada:** [`MCP.md`](./MCP.md) — lifecycle completo, transport, manifest format, namespacing, sandbox. Esta seção é o contrato formal; detalhes operacionais lá.

MCP é o **único caminho declarado** para extensão do tool catalog (`§2.6.7`). Sem este contrato, MCP é folclore — implementações divergem silenciosamente.

### A (Harness) garante a B (Server):

- **Transport conformidade:** segue [MCP spec](https://modelcontextprotocol.io) versão declarada em `mcp_protocol_version` (default `2024-11-05`); transport pode ser stdio, SSE, ou streamable HTTP
- **Lifecycle ordenado:** `initialize` → `initialized` → `tools/list` antes de qualquer `tools/call`
- **Spawn limpo (stdio):** env mínima (`PATH`, `HOME`, `MCP_*` allow-list); `cwd` documentado; sem inheritance de file descriptors do agente
- **Cancellation propagada:** `notifications/cancelled` enviado ao server quando user interrompe (Ctrl+C / Esc Esc); aguarda 2s graceful, depois SIGTERM (stdio) ou close (HTTP/SSE)
- **Timeout por call:** default 30s por `tools/call`; configurável via `mcp_servers.<name>.timeout_ms` em config
- **Schema validation:** input para `tools/call` é validado contra o schema declarado pelo server em `tools/list` antes do envio
- **Argumentos não-modificados:** harness não injeta `_meta`, `agent_session_id`, ou similar; server vê apenas o que o modelo emitiu
- **Hash do manifest persistido:** primeiro `tools/list` é hash-ado e gravado em `mcp_servers` (`AUDIT.md §1.5`); mudança detectada antes da próxima sessão
- **Per-server budget:** wall-clock e tokens consumidos cobertos por `ORCHESTRATION.md §11` (limite agregado de 30 connections, budget cascading do parent step)

### B (Server) garante a A (Harness):

- **Manifest válido:** resposta a `tools/list` retorna `tools: { name, description, inputSchema }[]` conforme MCP spec; nomes únicos dentro do server
- **Schemas estáveis:** `inputSchema` de uma tool não muda entre `tools/list` calls da mesma sessão; mudança requer reconnect (lifecycle reset)
- **Resposta em tempo finito:** `tools/call` retorna em ≤ timeout (default 30s); tool longa = ack `progress` notifications, não silêncio
- **Erros estruturados:** falha de tool retorna `{ isError: true, content: [...] }`, **não** JSON-RPC error frame (reservado pra protocol violations)
- **Sem dados sensíveis sem solicitação:** server não envia `resources/read` ou `prompts/get` que o modelo não pediu (princípio: server é tool source, não autoridade)
- **Sem mutação de policy:** server **não pode** declarar permissions, hooks, ou trust upgrades; apenas declara tools (ver `SECURITY_GUIDELINE.md §5` invariant 9)
- **Versão protocolar declarada:** `initialize` response tem `protocolVersion` exato; mismatch → harness aborta connect com erro classificado

### Failure semantics:

- **Server crash mid-call** → `tools/call` em vôo recebe `{ error: "mcp.server.crashed", server: "<name>" }`; server transita pra `disconnected` (ver `STATE_MACHINE.md §6.5`); reconnect tentado em próxima invocação (não automático em background)
- **Server manifest change** → detectado no início da próxima sessão; tools do server ficam **invisíveis ao modelo** até trust prompt re-aprovar (ver `FAILURE_MODES.md §14.2`)
- **Schema violation no input** → harness rejeita antes de enviar; modelo recebe `{ error: "mcp.schema.invalid", hint: <validation error> }`; **não chega ao server**
- **Schema violation no output** → harness loga, devolve `{ error: "mcp.output.invalid", hint }` ao modelo; server fica `degraded` mas não desconectado
- **Timeout** → `notifications/cancelled` enviado; resultado vira `{ error: "mcp.timeout", server, tool, elapsed_ms }`
- **Transport error** (stdio pipe broken, SSE disconnect, HTTP 5xx) → server transita pra `disconnected`; tools invisíveis até reconnect bem-sucedido
- **Protocol version mismatch** → server **nunca** registra; user vê erro no startup com hint pra atualizar harness ou server

### Side effects permitidos:
- **Harness:** escreve em `mcp_servers`, `mcp_manifest_history`, `tool_calls` (com `mcp_server` field), `approvals` (trust prompts), emite spans OTEL
- **Server:** declarados pelo próprio server no manifest e/ou descrição; **não** podem incluir mutação de SQLite do agente, FS fora de `cwd` declarado, ou network sem allowlist (`SECURITY_GUIDELINE.md §9`)

### Trust boundary:

- Server é **NÃO-confiável até prova em contrário** (princípio 11 do `AGENTIC_CLI.md`)
- Primeira conexão exige **trust prompt** (`AGENTIC_CLI.md §9.4`); decisão registrada em `approvals` com hash do manifest
- Tools de server não-confiável: registradas com `visible_to_model: false`; modelo nem sabe que existem
- Trust é **per-manifest-hash**, não per-server-name: server publicar nova tool = novo trust prompt (`STATE_MACHINE.md §6.5`)

### Namespacing:

- Tools MCP aparecem no registry como `mcp:<server-name>:<tool-name>` (ex: `mcp:postgres:query`)
- Colisão com tool canônica do `§2.6` → tool MCP **rejeitada** ao registrar (server não pode shadow `read_file`, `bash`, etc); registro falha com `mcp.namespace.shadow_canonical`
- Colisão entre dois servers → segundo registro vira `mcp:<server-name-2>:<tool-name>`; resolução por `server-name`, sem fallback ambíguo

### Versão: **v1**

---

## 12. Telemetry ↔ Subsistemas

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

## 13. Tabela-resumo de versionamento

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
| MCP Server ↔ Harness | v1 | `mcp_protocol_version` declarada per server; hash do manifest gravado |
| Telemetry ↔ Subsystems | v1 | OTEL semantic conventions seguidas onde aplicável |

Mudança em qualquer contrato:
1. Bump da versão local
2. Migration path documentado
3. Período de coexistência (suportar v1 e v2 por N releases)
4. Eval cobre ambas versões durante transição

---

## 14. O que **não** é contratual (intencionalmente)

Coisas que ficam de fora dos contratos por design:

- **Conteúdo dos prompts** — pode mudar a qualquer momento; eval cobre
- **Tool descriptions** — texto que o modelo lê; mudanças não quebram contrato (só métricas)
- **UI rendering** — Ink components são internos; mudam livremente
- **Schema interno do SQLite além das tabelas declaradas** — índices, triggers, vacuum strategy
- **Latência em microbenchmarks** — coberto em `PERFORMANCE.md`, não aqui
- **Heurísticas** (detection de injection, capability detection) — best-effort, sem garantia formal

---

## 15. Como verificar contratos

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

## 16. Insight final

Contrato bom não é o que **descreve** comportamento. É o que **constraini** comportamento para que duas implementações independentes interoperem.

Spec sem contrato é descrição de como **uma** implementação funciona. Spec com contrato é o **protocolo** que múltiplas implementações respeitam.

Esses contratos não são exaustivos — são o **mínimo necessário** para que o projeto não vire integration nightmare. Acrescentar contratos é fácil; remover é breaking change.
