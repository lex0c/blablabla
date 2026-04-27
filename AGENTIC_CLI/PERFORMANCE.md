# PERFORMANCE

SLOs, budgets de latência, e estratégia de regressão para o `AGENTIC_CLI`.

Sem números, "rápido" é palavra. Com números, é compromisso testável. Esses budgets não são arbitrários — vêm de **orçamento de atenção do usuário**: terminal-first significa que pausas humanamente perceptíveis (>200ms em UI síncrona, >2s em loop) destroem o fluxo.

---

## 0. Princípios

1. **Budgets são compromisso, não aspiração.** SLO violado = regressão = PR bloqueado.
2. **Orçamento total é fixo**, alocado entre subsistemas. Aumentar orçamento de A obriga a tirar de B.
3. **LLM latency é externa** — não controlamos. Mas controlamos *overhead* em volta dela.
4. **Streaming first.** Time-to-first-token é mais importante que time-to-completion.
5. **P50 e P99 separados.** P50 é o "típico"; P99 é o "pior aceitável". Ambos importam.
6. **Cold vs warm startup.** Métricas distinguem — cache de prompt e fs cache mudam comportamento.
7. **Profiling é eval.** Performance regression é tipo de regressão; CI cobre.
8. **Custo é dimensão de performance.** $/task é métrica primária junto com latência.

---

## 1. Orçamento de atenção (motivação dos números)

| Faixa | Percepção humana | Ação típica do agente |
|---|---|---|
| < 16ms | imperceptível | render frame |
| < 100ms | "instantâneo" | input echo, autocomplete |
| < 200ms | rápido | tool dispatch, schema check |
| < 1s | "tá pensando" | startup, leitura de arquivo grande |
| < 3s | "tolerável" | compaction, repo map build |
| < 10s | "demorado mas OK" | tool de bash, request a provider |
| > 10s | "tá travado" | só com progress visível |

Nada do que controlamos pode passar de **200ms** em path crítico (UI síncrona). Operações longas mostram progress.

---

## 2. SLO table — operações principais

Métricas em `latency_ms`, exceto onde indicado. P50/P99 sob carga normal (1 sessão, repo médio ~10k arquivos).

### 2.1 Startup

| Operação | P50 | P99 | Nota |
|---|---|---|---|
| Binary launch (cold) | 80ms | 200ms | Bun startup + módulos |
| Config load (warm fs cache) | 15ms | 40ms | TOML/YAML parse + validate |
| Trust check (already trusted) | 5ms | 15ms | hash compare |
| Trust prompt (first time) | — | — | bloqueia em humano; n/a |
| SQLite open + WAL ready | 20ms | 60ms | inclui PRAGMA setup |
| Memory index load | 10ms | 30ms | até 200 entries |
| Hooks `SessionStart` chain | 50ms | 200ms | depende dos hooks (cap 5s) |
| **Total: ready for prompt (cold)** | **180ms** | **500ms** | budget rígido |
| **Total: ready for prompt (warm)** | **80ms** | **200ms** | warm fs cache |

### 2.2 Loop interno (autonomous)

| Operação | P50 | P99 |
|---|---|---|
| Context assembly | 30ms | 80ms |
| Token count (pré-request) | 10ms | 30ms |
| Provider call: time-to-first-token | 400ms | 2s |
| Provider call: full stream (small step) | 2s | 8s |
| Provider call: full stream (large step) | 8s | 30s |
| Stream parse + emit | < 1ms/event | 5ms/event |
| Permission check | 5ms | 15ms |
| Hook `PreToolUse` (single fast hook) | 20ms | 100ms |
| Tool dispatch overhead | 5ms | 15ms |
| Step persist (SQLite) | 5ms | 15ms |
| Hook `PostToolUse` | 20ms | 100ms |

**Overhead total não-LLM por step:** P50 ~100ms, P99 ~300ms.
Provider domina; nosso overhead ≤ 5% de step LLM típico.

### 2.3 Tools

| Tool | P50 | P99 | Nota |
|---|---|---|---|
| `read_file` (1MB) | 10ms | 40ms | mmap-friendly |
| `glob` (10k arquivos) | 30ms | 100ms | ripgrep walk |
| `grep` (10k arquivos, padrão simples) | 80ms | 300ms | ripgrep |
| `grep` (10k arquivos, regex complexo) | 300ms | 1.5s | ripgrep |
| `write_file` (1MB) + checkpoint | 50ms | 200ms | git stash + write |
| `edit_file` (1MB) + checkpoint | 60ms | 250ms | leitura + replace + checkpoint |
| `bash` (echo) | 30ms | 80ms | overhead de spawn |
| `bash_background` (spawn) | 40ms | 120ms | + setup de pipes |
| `web_fetch` (cached) | 5ms | 20ms | hit local cache |
| `web_fetch` (uncached) | 200ms | 3s | depende de rede |

### 2.4 Subsystems

| Operação | P50 | P99 |
|---|---|---|
| Compaction (Haiku) | 1.5s | 4s |
| Checkpoint snapshot (git stash) | 30ms | 150ms |
| Checkpoint snapshot (reflink, 1k arquivos) | 50ms | 300ms |
| Checkpoint snapshot (cp -r, 1k arquivos) | 200ms | 2s |
| Checkpoint restore (`/undo`) | 50ms | 300ms |
| Memory write (with confirmation UI bypassed) | 20ms | 60ms |
| Subagent spawn | 100ms | 300ms |
| Subagent first heartbeat | 200ms | 500ms |

### 2.5 Profile `orchestrated`

| Operação | P50 | P99 |
|---|---|---|
| DAG load + validate | 20ms | 60ms |
| DAG node dispatch | 5ms | 15ms |
| Validator built-in (fast) | 10ms | 50ms |
| Validator AST (tree-sitter parse) | 50ms | 200ms |
| Step LLM (Ollama qwen-coder-14b, GPU) | 800ms | 3s |
| Step LLM (Ollama qwen-coder-14b, CPU) | 5s | 20s |
| Repo map update (1 file changed) | 30ms | 100ms |
| Repo map full rebuild (10k arquivos) | 5s | 15s |

### 2.6 UI rendering (Ink)

| Operação | P50 | P99 |
|---|---|---|
| First paint após layout change | 16ms | 33ms (= 1 frame) |
| Stream token render | 1ms | 5ms |
| Tool card update | 5ms | 20ms |
| Diff render (100 hunks) | 30ms | 100ms |
| TodoList update | 5ms | 20ms |

---

## 3. Budget allocation por step

Step LLM típico de **5s** (provider) tem budget total de **~200ms** de overhead nosso (4%):

```
[step total: ~5.2s]
├─ context assembly         30ms
├─ token count               10ms
├─ provider HTTP setup       20ms
├─ [PROVIDER STREAM]      5000ms   ← não controlamos
├─ stream parse/emit         50ms (acumulado)
├─ permission check          5ms
├─ pre-tool hook            20ms
├─ tool execute (varia)
├─ post-tool hook           20ms
├─ checkpoint (se write)    50ms
├─ persist                  10ms
└─ ui render                10ms
```

Se overhead total > 5% do step LLM por > 24h: **alerta**. Indicativo de algo mal-otimizado.

---

## 4. Custo (USD)

Latência sem custo é metade da história. SLOs de custo:

| Tarefa típica | Profile autonomous (Sonnet 4.6) | Profile orchestrated (Ollama local) | Profile hybrid |
|---|---|---|---|
| `/review` (200 LOC diff) | $0.05–0.20 | $0 | $0–$0.05 |
| `/audit` (200 LOC diff) | $0.10–0.40 | $0 (qualidade menor) | $0.02–0.10 |
| `/debug` (1 hipótese) | $0.05–0.30 | $0 | $0–$0.10 |
| Refactor multi-arquivo | $0.50–3.00 | n/a (qualidade insuficiente) | $0.20–1.00 |
| Sessão completa típica (45min) | $1–8 | $0 | $0.30–2 |

**Hard cap padrão:** `$5/sessão`. Configurável; nunca opt-out.

---

## 5. Performance regression strategy

### 5.1 Bench dataset

`evals/bench/perf/` contém:
- 30 fixtures de tarefa real
- Cada com input + estado fs esperado + budget
- Roda em CI noturno
- Compara contra baseline em `evals/bench/perf/baseline.json`

### 5.2 Thresholds de regressão

PR é bloqueado se:
- P50 de qualquer operação principal aumenta > 20%
- P99 aumenta > 50%
- Custo médio de eval suite aumenta > 15%
- Throughput (steps/min sob carga) cai > 15%

### 5.3 Profiling

`agent --profile` ativa instrumentação detalhada:
- Per-span CPU time
- Per-span allocations
- Output: NDJSON consumível por `chrome://tracing`

Em CI: roda em 3 fixtures longos; compara flame graph com baseline (visualmente em PR via comment).

### 5.4 Specific budgets como assert

Em código de hot path:

```ts
const t0 = performance.now();
const result = await contextEngine.assemble(...);
const elapsed = performance.now() - t0;
if (elapsed > 100 && process.env.AGENT_BUDGET_ENFORCE) {
  throw new BudgetViolation('contextEngine.assemble', elapsed, 100);
}
```

Em `--budget-enforce`: violations viram exceptions. Em prod: warning em telemetry.

---

## 6. Memory footprint

Limites operacionais:

| Cenário | RSS esperado | Hard cap |
|---|---|---|
| Sessão idle | 80MB | 200MB |
| Sessão ativa, 10 turns | 150MB | 400MB |
| Sessão longa (100 turns, sem compaction) | 300MB | 600MB |
| Profile orchestrated com Ollama (out-of-process) | 150MB | 400MB (agente apenas) |
| 5 subagents paralelos | 600MB total | 1.2GB total |

Hard cap atingido: agente avisa user, pausa, oferece compaction.

Memory leak detection: `bench/perf/leak/` roda 1000 turns sintéticos; RSS deve estabilizar em < 400MB. Crescimento monotônico = regressão.

---

## 7. Disk usage

| Categoria | Tamanho típico | Cleanup |
|---|---|---|
| `sessions.db` (após 100 sessões médias) | 50–200MB | `agent db vacuum` manual |
| `traces/` NDJSON (90 dias) | 100MB–1GB | retention via cron, default 90d |
| `checkpoints/` (fallback non-git) | 100MB–10GB | `agent checkpoint gc` ao final de sessão |
| `~/.config/agent/memory/` | < 1MB | manual via `/memory delete` |
| Worktrees abandonados | varia | `agent worktree gc` |

Total típico: 500MB–2GB para usuário normal.

---

## 8. Concurrency

### 8.1 Limites concorrentes

| Recurso | Default | Hard cap |
|---|---|---|
| Sessões ativas (mesmo processo) | 1 | 1 (multi-instance via lockfile) |
| Subagents paralelos por sessão | 3 | 8 |
| Background processes por sessão | 5 | 20 |
| Hooks paralelos por evento | 10 | 50 |
| MCP server connections | 10 | 30 |
| Tool calls em flight (no DAG executor) | 5 | 16 |

### 8.2 Lock strategy

- `cwd` lockado via lockfile (`.agent/lock`) com PID + start_ts
- Lock stale (PID não existe) é roubado com warning
- SQLite em modo WAL (concorrência de leituras OK; uma escrita por vez)

---

## 9. Cold start otimizações (concretas)

Pra atingir os 80ms P50 cold:

1. **Bun runtime** (~30ms cold vs Node ~100ms)
2. **Lazy imports** — só carrega módulos quando usados (Ink só se interactive, OTEL só se enabled, etc)
3. **Config cached** — `~/.config/agent/.cache/` com config compilada (TTL by mtime)
4. **SQLite prepared statements** cached entre runs (via cached statements file)
5. **Repo map skipped on startup** — só constrói se requested ou em DAG (incremental updates após)

---

## 10. Streaming guarantees

Time-to-first-token (TTFT) é métrica de UX primária:

| Provider | TTFT P50 | TTFT P99 |
|---|---|---|
| Anthropic Sonnet | 400ms | 1.5s |
| Anthropic Haiku | 200ms | 800ms |
| Ollama qwen-coder-14b (local GPU) | 80ms | 300ms |
| Ollama qwen-coder-14b (local CPU) | 1s | 5s |

**Nosso overhead entre HTTP recv → tela:** < 5ms P99.

UI mostra spinner se nada renderizou em 500ms — usuário sabe que está rolando.

---

## 11. Provider request optimization

### 11.1 Prompt cache (Anthropic)

- Layout fixo do prompt (§6 do AGENTIC_CLI) com 3 cache breakpoints
- TTL 5 min — cada turn dentro de uma sessão hits cache
- **Esperado:** > 70% de input tokens cached em sessão de > 3 turns
- Eval mede `cache_hit_ratio`; abaixo de 50% = bug de layout

### 11.2 Token-aware truncation

Antes de enviar request, hard cap em `context_window - 2000` (reserva pra output).

### 11.3 Concurrent provider calls

Em DAG executor: nodes sem dependência entre si **rodam em paralelo** (até `max_concurrent_llm_calls=5`).

### 11.4 Provider-specific deep dive

Cada provider tem otimizações específicas além de §11.1-3.

#### Anthropic

- **Cache breakpoints declarados** — sempre 3 (system + tools + project_ctx); hit ≥ 3 turns
- **Extended cache**: usar `extended_cache: true` em sessões com gaps > 5min entre turns (TTL 1h; preço diferente, vale se sessão > 30min)
- **Reasoning tokens** (extended thinking): não persistem em messages; cobrados separados em `reasoning.cost_usd`
- **Tool_use streaming parser**: incremental — `content_block_start (tool_use)` → N× `input_json_delta` → `content_block_stop`; harness reconstrói args antes de invoke
- **529 (overloaded)**: backoff agressivo; não tratar como 5xx tradicional

#### OpenAI

- **Prefix-cache automático**: layout estável early no prompt maximiza hit (não controlable, é probabilístico)
- **Structured outputs**: ~100-200ms overhead; usar quando schema é crítico, evitar quando opcional
- **Reasoning models (o1/o3)**: reasoning tokens ocultos mas cobrados; UI mostra `🧠 reasoning... (Xs)`
- **Function calling fragmented stream**: reconstruir tool_calls a partir de deltas antes de validate
- **Tier rate limit**: tier 1 vs tier 5 muda 100×; backoff agressivo em tier baixo

#### Ollama

- **`keep_alive` parameter**: default 5min; em sessão ativa, considerar `keep_alive: -1` (modelo persiste em GPU/RAM)
- **Context shift**: modelos com window pequeno fazem shift automático perdendo contexto inicial — compaction agressivo (50% trigger)
- **JSON mode retry**: validate + ≤ 2 retries com hint específico em malformed
- **Prompts model-specific**: detection via model name prefix (llama-3 ≠ qwen ≠ deepseek-coder)

#### llama.cpp

- **GBNF compilation cache**: compilar grammar uma vez por session, cachear em RAM
- **Batch size**: tunable; default 512 mas hardware-dependent (Q4 vs Q8 muda)
- **Quantization sweet spot**: Q4_K_M para qualidade aceitável + speed; FP16 quando precisão importa
- **`mlock`**: pin model em RAM evita swap em low-RAM systems

---

## 12. SQLite tuning

PRAGMAs canônicas aplicadas em boot da conexão.

### 12.1 PRAGMAs canônicas

```toml
# config aplicado pelo agent em SessionStart
[storage.sqlite.pragmas]
journal_mode = "WAL"             # Write-Ahead Log; concurrency reads
synchronous = "FULL"             # prod (FULL); dev/CI (NORMAL — 10× faster, risk em power loss)
busy_timeout = 5000              # 5s wait em lock contention (multi-instance)
cache_size = -65536              # negativo = kB; -65536 = 64MB cache
mmap_size = 268435456            # 256MB mmap pra reads
temp_store = "MEMORY"            # temp tables em RAM
foreign_keys = "ON"              # integridade referencial
auto_vacuum = "INCREMENTAL"      # libera espaço pós-DELETE sem full vacuum
page_size = 4096                 # default; 8192 em datasets > 1GB
```

### 12.2 Modes (por contexto)

| Mode | synchronous | auto_vacuum | cache_size | Quando usar |
|---|---|---|---|---|
| `prod` (default interactive) | FULL | INCREMENTAL | 64MB | dev local, prod |
| `dev` | NORMAL | INCREMENTAL | 32MB | speedup em testes; risk power loss |
| `ci` | NORMAL | NONE | 16MB | espaço volta após retention global |
| `forensics` (read-only) | n/a (read) | n/a | 256MB | inspeção de bundle grande |

Override via `~/.config/agent/config.toml [storage.sqlite] mode = "dev"`.

### 12.3 VACUUM strategy

- `auto_vacuum = INCREMENTAL` roda automatic ao deletar
- Manual: `agent audit vacuum` (recomendado semanal em sessões grandes)
- Custo: pode levar minutos em DB > 1GB; CI agenda noturno
- WAL checkpoint: automatic; manual `PRAGMA wal_checkpoint(TRUNCATE)` em retention cleanup grande

### 12.4 Indexes

Já listados em `AUDIT.md §10.1`. Verificação periódica via `EXPLAIN QUERY PLAN` em queries de hot path em CI.

---

## 13. Caching strategy consolidada

Múltiplas camadas com TTL + invalidation + hit rate target explícitos.

| Cache | Layer | TTL | Hit rate target | Invalidation |
|---|---|---|---|---|
| Prompt cache (Anthropic) | provider server-side | 5min (extended: 1h) | > 70% em sessão > 3 turns | layout change; gap > TTL |
| Prefix cache (OpenAI) | provider server-side | opaco (probabilístico) | varies | indeterminate; layout estável ajuda |
| `recap_cache` table | local SQLite | 1h | > 80% em re-listings | sessão `ended_at` change; manual `/recap regenerate` |
| Memory index | RAM (per process) | enquanto sessão ativa | 100% | `memory_event` action ∈ {create,edit,delete} |
| File content cache | RAM (per session) | enquanto sessão ativa | > 60% em refactor | `write_file`/`edit_file` em path; FS mtime change |
| Schema validators | RAM (per process) | enquanto processo vivo | 100% | reload em config change |
| GBNF grammars (llama.cpp) | RAM (per process) | enquanto processo vivo | 100% | nunca (recompila apenas em version bump) |
| Repo map | SQLite + RAM | até FS change | > 90% | `PostToolUse` em write tools dispara incremental update |
| DNS resolution | OS-level | TTL do registro | varies | TTL expire |
| HTTP keepalive | per-provider connection pool | 60s idle | depends on traffic | connection close ou recycle |

### 13.1 Invalidation rules formalizadas

Sem TTL implícito. Cada cache documenta:
- **Quando invalida** (event-based ou time-based)
- **Granularidade** (per file, per session, global)
- **Cost de miss** (em ms ou $)

### 13.2 Cache observability

`agent stats --cache`:

```
Cache             Hit rate   Size      Misses (last 1h)
prompt (anthropic)   78%       n/a       12
recap_cache          82%       4.2 MB    8
memory_index        100%       12 KB     0
file_content         63%       18 MB     45
repo_map             94%       2.1 MB    3
```

Hit rate < target sustained = warning em `agent doctor`.

---

## 14. Network optimizations

```toml
[network]
http_keepalive = true                    # reaproveita TCP entre requests
keepalive_idle_ms = 60000                # 60s default; SSH high-latency: 300000 (5min)
connection_pool_size = 10                # por provider
connection_pool_recycle_ms = 600000      # 10min; previne connection rot
compression = "gzip"                     # request body > 1KB
dns_cache_ttl_ms = 300000                # 5min override de OS
streaming_chunk_size_bytes = 4096
provider_request_timeout_ms = 60000      # default 60s; configurável
```

### 14.1 Por provider

| Provider | Keepalive | Compression | Notes |
|---|---|---|---|
| Anthropic | crítico (sessão longa = N reqs ao mesmo host) | sim em prompts grandes | TLS handshake é caro; reuse essencial |
| OpenAI | sim | sim | tier 1 tem rate limit baixo; pool > 3 quase nunca usado |
| Ollama (localhost) | irrelevante (loopback) | desnecessário | keep_alive do model importa mais |
| llama.cpp HTTP server | sim | opcional | localhost típico |
| MCP servers (stdio) | n/a | n/a | comunicação via pipe |

### 14.2 Em SSH com alta latência

```toml
[network.ssh_high_latency]
keepalive_idle_ms = 300000               # 5min
connection_pool_size = 5                 # menos é melhor (cada socket caro)
streaming_chunk_size_bytes = 16384       # chunks maiores reduzem round-trips
```

Detection: heuristic via medir RTT do primeiro request; se > 200ms → aplica preset.

---

## 15. Memory optimization techniques

Pra cumprir RSS hard cap (600MB em sessão longa de 100+ turns).

### 15.1 Técnicas

1. **Stream parser sem buffer completo** — consome `text_delta` chunk-by-chunk; descarta após persist
2. **Tool result truncation antes de buffer** — output > 100KB salvo em arquivo + pointer; só pointer vai pro contexto
3. **Object pool pra Step/Message** — reuso após persist; reduz GC pressure
4. **Compaction agressivo em RSS warning** — se RSS > 80% do hard cap, força compaction antes do trigger normal de tokens
5. **Bun `--smol` flag** em subagents (less memory mode)
6. **Lazy require** de tool implementations (carrega em first invoke)
7. **Repo map compression** — symbols deduplicados, paths shared via interning

### 15.2 Memory leak detection

```
bench/perf/leak/
  long_session_1000_turns.test.ts        # roda 1000 turns sintéticos; RSS deve estabilizar < 400MB
  parallel_subagents_stress.test.ts      # 8 subagents em rotação; RSS < 800MB total
```

CI roda noturno; regressão de + 50MB em RSS pico = bloqueia merge.

### 15.3 Heap snapshot em diagnostics

`agent doctor --heap-snapshot=/tmp/heap.heapsnapshot`:
- Gera snapshot Bun/V8
- Inspecionável em Chrome DevTools
- Útil em sessão com RSS anormal pra debug

---

## 16. Lazy loading rules

**Default: lazy.** Carrega apenas em first use.

### 16.1 Eager (sempre carregado)

| O quê | Por quê |
|---|---|
| Memory index (`MEMORY.md` + frontmatters) | Modelo precisa ver pra decidir relevance; ~2k tokens, cheap |
| `CLAUDE.md` (project context) | Modelo precisa pra orientar comportamento |
| Tool schemas | Modelo precisa pra decidir invocação |
| Permission policy | Cada tool call consulta |
| System prompt | Sempre |
| Hooks declarações | Listadas eager (comandos executam lazy) |

### 16.2 Lazy (load em first use)

| O quê | Trigger |
|---|---|
| Memory content (body do `.md`) | `memory_read` tool invoke |
| Tool implementations | first invoke do tool |
| Playbook prompts | first slash command ou `task(playbook=...)` |
| Subagent definitions | first `task_async`/`task_sync` |
| Compaction prompt | first compaction trigger |
| Critique prompt | first critique pass (se opt-in) |
| Repo map | first `repo_map` tool ou first `grep`/`glob` em sessão `orchestrated` |
| Slash command custom (`.md` file) | first invocação |

### 16.3 Pré-warming opt-in

Hook `SessionStart` pode pré-carregar artefatos:

```toml
[[hooks]]
event = "SessionStart"
command = "agent prewarm --tools edit_file,grep --playbooks code-review"
```

Útil em workflows que sempre usam mesmas tools (cold start ~0).

---

## 17. Workload-specific tuning

Diferentes workloads têm perfis distintos. Tunings recomendados:

### 17.1 Tabela

| Workload | Tunings |
|---|---|
| **Read-heavy** (review, audit, explain) | tool palette restrita; SQLite `cache_size = -32768`; sem checkpoints; compaction trigger 80% |
| **Write-heavy** (refactor, fix-bug) | checkpoint reflink onde possível; FS cache aggressive (`mmap_size` 512MB); compaction trigger 60%; `auto_vacuum INCREMENTAL` |
| **Long-context** (sessão > 30 turns) | compaction trigger 50%; goal re-injection a cada 10 steps; max_steps 100; trace sampling 50% |
| **CI / batch** | `audit.mode=minimal`; trace sampling 10%; storage budget 50GB; sessions retention 7d; sandbox obrigatório |
| **Local-first orchestrated** | `keep_alive=-1` (modelo pinned em GPU); memory index pré-warmed; tool palette mínima por DAG node; per-node max_retries 1 |
| **Hybrid** | planner cache muito quente (extended TTL); aggressive prompt caching no frontier; rotation pra fallback frequente |
| **Watch** (build watching, log tailing, dev servers) | `wait_for` / `monitor` agressivo; `bash_background` keep_alive longo; HTTP keepalive 5min (SSH-friendly); `monitor` max_events alto (50+); reduzir polling LLM |

### 17.2 Aplicação via flag

```bash
agent --workload read-heavy "..."
agent --workload long-context --resume <id>
agent --workload ci --json "..."
```

Aplica preset; user pode override individual setting via flag adicional.

---

## 18. Distribution optimizations

### 18.1 Build

```toml
[build]
runtime = "bun"                          # bun build --compile
tree_shaking = true
minify = true                            # em release
sourcemap = "external"                   # .map separado
target = "bun"                           # AOT pra cada plataforma
```

### 18.2 Binary size targets

| Plataforma | Target |
|---|---|
| Linux x86_64 (glibc) | < 50 MB |
| Linux x86_64 (musl) | < 50 MB |
| Linux ARM64 | < 50 MB |
| macOS x86_64 | < 50 MB |
| macOS ARM64 | < 45 MB |
| Windows x86_64 | < 60 MB |

Hit do target = warning; +20% = bloqueia release.

### 18.3 Compression opcional

```bash
agent build --compress              # UPX-compressed; -40% size, +30ms startup
agent build                         # default; perf > size
```

Default release: descompactado.

### 18.4 Native deps

Não bundlados (disponíveis via PATH):
- `bwrap` (sandbox Linux)
- `ripgrep` (rg)
- `git`
- `tree-sitter` (compilado uma vez)

`agent doctor` checa disponibilidade.

### 18.5 Cross-platform CI

```yaml
# .github/workflows/build.yaml
strategy:
  matrix:
    target: [linux-x64, linux-arm64, darwin-x64, darwin-arm64, windows-x64]
```

Cada release:
- SBOM por target
- SHA256 publicado
- Cosign signature
- Reproducible build (mesmo source = mesmo binary)

---

## 19. UI responsiveness

### 12.1 Frame budget

Ink target 60fps em re-renders → 16ms/frame.

Re-render full screen é evitado. Componentes memoizam aggressively. `<DiffView>` com 1000 linhas é virtualizado.

### 12.2 Input lag

Tecla pressionada → caractere na tela: < 10ms P99.

Stream de modelo + input concorrente: input lag não pode degradar (input loop separado de render loop).

---

## 20. Cost-per-task como métrica primária

Em CI, eval registra `$/task` por suite:

```
Suite                      Avg cost    P95 cost    Trend
smoke (autonomous)         $0.08       $0.15       →
regression (autonomous)    $0.12       $0.30       ↑ +5% (alerta)
smoke (orchestrated)       $0          $0          —
hybrid (planner+executor)  $0.04       $0.10       →
```

Aumento > 15% em qualquer suite: **PR bloqueado** mesmo com testes passando. Cost regression é regressão.

---

## 21. O que não medimos (intencionalmente)

- LLM "qualidade" via métricas auto — qualidade é avaliada por eval funcional, não por metric proxy
- Tempo do humano em modal de confirmação — fora de escopo
- Latência de rede ao provider — tracking sim (telemetry), SLO não (não controlamos)
- Performance em hardware abaixo do mínimo (M1 ARM, < 8GB RAM, etc)

---

## 22. Hardware mínimo declarado

Pra cumprir SLOs:

| Profile | CPU | RAM | Disk |
|---|---|---|---|
| `autonomous` | qualquer x86_64/ARM64 dual-core | 1GB livres | 1GB livres |
| `orchestrated` (Ollama 7B) | x86_64/ARM64 quad-core | 8GB livres | 5GB livres |
| `orchestrated` (Ollama 14B+) | + GPU 8GB+ recomendado | 16GB livres | 10GB livres |
| `hybrid` | qualquer (frontier API) + 7B local | 8GB livres | 5GB livres |

Abaixo do mínimo: agente avisa em SessionStart; SLOs não garantidos.

---

## 23. Como mudar um SLO

Mudança em SLO é breaking change de contrato com usuário. Procedimento:

1. Justificativa documentada (por que aumentou? regressão real?)
2. Eval cobrindo o novo budget
3. Mudança em `evals/bench/perf/baseline.json` em PR específico
4. Changelog entry

Reduzir SLO (ficar mais rápido) é livre. Aumentar (degradar) requer aprovação explícita.

---

## 24. Insight final

Performance não é otimização — é **decisão de design**. Cada subsistema tem orçamento porque o orçamento total é fixo, e o orçamento total vem do tempo de atenção do humano.

Quando alguém pergunta "vamos adicionar feature X?", a pergunta correta é "X cabe no budget?". Se não cabe, ou outro subsistema cede orçamento, ou X fica de fora.

A regra é: **se você não tem número, você não tem performance — tem expectativa.**
