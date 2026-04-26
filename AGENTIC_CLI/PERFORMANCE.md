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

---

## 12. UI responsiveness

### 12.1 Frame budget

Ink target 60fps em re-renders → 16ms/frame.

Re-render full screen é evitado. Componentes memoizam aggressively. `<DiffView>` com 1000 linhas é virtualizado.

### 12.2 Input lag

Tecla pressionada → caractere na tela: < 10ms P99.

Stream de modelo + input concorrente: input lag não pode degradar (input loop separado de render loop).

---

## 13. Cost-per-task como métrica primária

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

## 14. O que não medimos (intencionalmente)

- LLM "qualidade" via métricas auto — qualidade é avaliada por eval funcional, não por metric proxy
- Tempo do humano em modal de confirmação — fora de escopo
- Latência de rede ao provider — tracking sim (telemetry), SLO não (não controlamos)
- Performance em hardware abaixo do mínimo (M1 ARM, < 8GB RAM, etc)

---

## 15. Hardware mínimo declarado

Pra cumprir SLOs:

| Profile | CPU | RAM | Disk |
|---|---|---|---|
| `autonomous` | qualquer x86_64/ARM64 dual-core | 1GB livres | 1GB livres |
| `orchestrated` (Ollama 7B) | x86_64/ARM64 quad-core | 8GB livres | 5GB livres |
| `orchestrated` (Ollama 14B+) | + GPU 8GB+ recomendado | 16GB livres | 10GB livres |
| `hybrid` | qualquer (frontier API) + 7B local | 8GB livres | 5GB livres |

Abaixo do mínimo: agente avisa em SessionStart; SLOs não garantidos.

---

## 16. Como mudar um SLO

Mudança em SLO é breaking change de contrato com usuário. Procedimento:

1. Justificativa documentada (por que aumentou? regressão real?)
2. Eval cobrindo o novo budget
3. Mudança em `evals/bench/perf/baseline.json` em PR específico
4. Changelog entry

Reduzir SLO (ficar mais rápido) é livre. Aumentar (degradar) requer aprovação explícita.

---

## 17. Insight final

Performance não é otimização — é **decisão de design**. Cada subsistema tem orçamento porque o orçamento total é fixo, e o orçamento total vem do tempo de atenção do humano.

Quando alguém pergunta "vamos adicionar feature X?", a pergunta correta é "X cabe no budget?". Se não cabe, ou outro subsistema cede orçamento, ou X fica de fora.

A regra é: **se você não tem número, você não tem performance — tem expectativa.**
