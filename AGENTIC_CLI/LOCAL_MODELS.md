# LOCAL_MODELS

Operação detalhada pra rodar `AGENTIC_CLI` com modelos locais (Ollama, llama.cpp, e backends similares). Hardware detection, model lifecycle, tool calling adapters, constrained generation, failure modes, privacy verifiable.

`PROVIDERS.md §3.3-§3.4` cobre quirks de cada backend; este doc é o **detalhamento operacional** consolidado. `PERFORMANCE.md §11.4` lista otimizações; este doc costura tudo em fluxo.

Local-first é **killer use case declarado** (§0 Visão do AGENTIC_CLI.md). Spec sem este detalhamento entrega "você pode usar local"; spec com ele entrega "você usa local bem".

---

## 0. Princípios (não-negociáveis)

1. **Custo zero não é grátis.** Time-cost real (latência alta em CPU; warm-up 5-30s). Documentar honestamente.
2. **Privacy verifiable, não declared.** Local-first significa nada sai do host — checável via `agent privacy verify`, não promessa.
3. **Hardware-aware.** Agent detecta GPU/VRAM/RAM e recomenda modelo apropriado, não força user a adivinhar quantization.
4. **Lifecycle explícito.** Load, warm-up, keep_alive, unload — todos documentados; sem assumption "Ollama cuida".
5. **Tool calling adapter é parte do contrato.** Modelos sem tool nativo precisam adapter formal — não improvisar.
6. **Quality declared per model.** Tabela honesta de "X funciona bem em workflow Y; não funciona em Z".
7. **Fallback graceful (hybrid).** Quando local não dá conta, hybrid escala pra frontier — sem surpresa.

---

## 1. Hardware detection & requirements

### 1.1 Hardware probe

`agent doctor --hardware`:

```
Hardware probe:
  GPU:        NVIDIA RTX 4090 (24GB VRAM, CUDA 12.4)
  CPU:        AMD Ryzen 9 5950X (32 threads)
  RAM:        64GB
  Disk:       1.2TB free in ~/.ollama/models
  OS:         Linux 6.18 (kernel)

Recommendations (model + quantization):
  best:       qwen2.5-coder:32b @ Q4_K_M (~18GB VRAM, fits comfortably)
  balanced:   qwen2.5-coder:14b @ Q4_K_M (~8GB VRAM, leaves room)
  fast:       qwen2.5-coder:7b @ Q4_K_M (~4GB VRAM, low latency)
  fallback:   none needed (frontier optional)
```

### 1.2 Hardware → modelo matrix

| Hardware | Quantization | Recommended models | Profile típico |
|---|---|---|---|
| GPU 24GB+ | Q4_K_M | 32B-70B class (Qwen-32B, Llama-70B, DeepSeek-33B) | orchestrated standalone |
| GPU 16GB | Q4_K_M | 14B-32B class | orchestrated standalone |
| GPU 8-12GB | Q4_K_M | 14B class (Qwen-coder-14B sweet spot) | orchestrated standalone |
| GPU 4-6GB | Q4_K_M | 7B class | orchestrated com fallback hybrid |
| GPU 0 (CPU only) | Q4_K_M | 7B class (paciência) | orchestrated com fallback ou hybrid |
| RAM ≥ 16GB | (CPU inference) | 7B-13B com `mlock` | acceptable em workloads não-latency-critical |
| RAM < 8GB | n/a | hybrid only (planner remote, exec mínima local) | hybrid |

### 1.3 Disk space

Cada modelo Q4 ocupa ~0.6-1× tamanho em GB:
- 7B Q4: ~4GB
- 14B Q4: ~8GB
- 32B Q4: ~18GB
- 70B Q4: ~40GB

`agent setup local` checa disk antes de pull.

### 1.4 OS support

| OS | GPU support | Notes |
|---|---|---|
| Linux | CUDA, ROCm, Vulkan | mais maduro |
| macOS | Metal (MLX, Ollama nativo) | M-series excelente |
| Windows | CUDA via WSL2 ou nativo | Ollama Windows beta |

CPU inference funciona em todos.

---

## 2. Model lifecycle

### 2.1 Estados

| Estado | Significado | Latência subsequente |
|---|---|---|
| `not_loaded` | modelo não está em memória | + load time (5-30s) |
| `loading` | primeira request iniciou load | aguardando |
| `loaded` (idle) | em VRAM/RAM, ready | < 100ms (TTFT) |
| `serving` | atendendo request | streaming |
| `unloading` | sendo removido | n/a |

### 2.2 Lifecycle policies

```toml
[providers.ollama.lifecycle]
keep_alive_active_session = "-1"     # sessão ativa: pinned em VRAM
keep_alive_idle = "5m"               # sessão idle: libera após 5min
preload_on_session_start = true      # pre-warm em SessionStart
preload_timeout_ms = 60000           # max 60s pra primeira load

[providers.ollama.lifecycle.background]
unload_after_session_end = true      # libera VRAM ao fim da sessão
```

**Comportamento:**
- Sessão ativa → modelo `keep_alive: -1` (pinned em VRAM até comando)
- Sessão termina (`done`/`exhausted`/`interrupted`) → libera (default Ollama 5min)
- Resume de sessão → reload (custo cold start aceito)

### 2.3 Pre-warm

Hook `SessionStart` opcional:

```toml
[[hooks]]
event = "SessionStart"
command = "curl -s http://localhost:11434/api/generate -d '{\"model\":\"qwen2.5-coder:14b\",\"prompt\":\"ready\",\"stream\":false}' > /dev/null"
```

Roda em paralelo com trust prompt; user mal vê esperar.

### 2.4 Cold start UX

Primeira request com `not_loaded`:

```
[step 1] modelo: qwen2.5-coder:14b
  ⏳ loading model into VRAM (24s)... (esperado primeira vez)
  ✓ ready
  → generating...
```

Componente UI: `<ModelLoadIndicator>` durante load.

Timeout: 60s default. Falha → `local.model.load_timeout` (§11).

### 2.5 Unload semantics

Em `Stop` hook (fim de sessão):
- `keep_alive_active_session: -1` revertido
- Default Ollama timeout (5min) toma conta
- Em multi-instance: respeitar lock (não unload se outra sessão usa)

User explícito: `agent local unload <model>` força unload imediato.

---

## 3. Tool calling adapter formal

Modelos sem `tools: 'native'` (maioria local) precisam **adapter**. Spec canônica:

### 3.1 Pattern XML-style

Modelo é instruído a emitir tool calls como:

```
<tool>tool_name</tool>
<args>
{
  "key": "value"
}
</args>
```

Pattern simples, robusto, parseável com regex confiável.

### 3.2 Few-shot template (canônico)

```
You have access to these tools:
- read_file(path: string)
- grep(pattern: string, path?: string)
- bash(command: string)

To call a tool, emit:
<tool>tool_name</tool>
<args>{"key": "value"}</args>

Example:
User: list TS files in src/
Assistant: I'll search.
<tool>grep</tool>
<args>{"pattern": "*.ts", "path": "src/"}</args>

Example bad (don't do):
User: ...
Assistant: I'll call grep("*.ts", "src/")    ← WRONG: function syntax
```

2 positivos + 1 contra-exemplo. Validação empírica em eval.

### 3.3 Parser

```ts
function parseToolCall(output: string): ToolCall | null {
  const match = output.match(/<tool>([^<]+)<\/tool>\s*<args>([\s\S]*?)<\/args>/);
  if (!match) return null;
  const [_, name, argsRaw] = match;
  try {
    const args = JSON.parse(argsRaw);
    return { name: name.trim(), args };
  } catch {
    return null;
  }
}
```

### 3.4 Validation pipeline

1. Parse output
2. Se parse falha: retry com hint `Sua tool call estava malformada. Use formato: <tool>name</tool><args>{...}</args>`
3. Schema validate args contra tool's `inputSchema`
4. Schema falha: retry com hint específico contendo schema esperado
5. Após 2 retries: tool_use abandonado; modelo recebe `tool_error` estruturado (§11.1.4)

### 3.5 Multi-tool no mesmo step

Adapter parser detecta múltiplos `<tool>...</tool><args>...</args>` em sequência. Sequencial por default (`parallel_safe: false`).

### 3.6 Quando adapter NÃO funciona

Modelos < 7B parameters geralmente **não conseguem** seguir o adapter consistentemente. Recomendação:
- 7B: marginal; 70%+ accuracy
- 14B: usável; 85%+ accuracy
- 32B+: confiável; > 95%

Eval por modelo declara accuracy. Abaixo de 70%: modelo não-recomendado pra workflows que dependem de tools.

---

## 4. Constrained generation backends

### 4.1 GBNF (llama.cpp)

Grammar canônica + flexível:

```bnf
root ::= tool-call | text-response

tool-call ::= "<tool>" tool-name "</tool>\n<args>\n" json-object "\n</args>"

tool-name ::= "read_file" | "grep" | "bash" | ...

json-object ::= "{" ws members? ws "}"
members ::= member (ws "," ws member)*
member ::= ws string ws ":" ws value

# (resto da grammar JSON)
```

**Compilation:**
- Compilada uma vez por session
- Cached em RAM (~10ms compile)
- Near-zero overhead per token (rejection sampling)

**Failures:** raras (grammar enforced em runtime). Quando ocorre: bug de grammar generator.

### 4.2 JSON mode (Ollama)

API:
```bash
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5-coder:14b",
  "prompt": "...",
  "format": "json",
  "stream": false
}'
```

**Comportamento:**
- Modelo treinado a respeitar; não enforced runtime
- Validation rate observada: 90-95% (5-10% malformed)
- Common failures: trailing comma, missing quote, truncation

**Retry strategy:**
1. First fail: re-prompt com `Output deve ser JSON válido. Erro: <details>`
2. Second fail: fallback pra adapter XML-style ou abort

### 4.3 Tools native (Anthropic, OpenAI)

Trust the provider; sem adapter. Stream parsing per provider quirks (PROVIDERS.md §3.1-3.2).

### 4.4 Regex/parse-and-retry

Last resort. Útil em modelos que nem adapter XML seguem direito (raro, e merece sair do registry).

### 4.5 Schema → backend matrix

| Schema declarado | llama.cpp | Ollama | Anthropic | OpenAI |
|---|---|---|---|---|
| Output schema simples | GBNF | JSON mode | tools | tools+json |
| Output schema complexo (nested, enums) | GBNF | JSON mode + retry | tools | structured outputs |
| Tool call format | GBNF + adapter | JSON mode + adapter | tools native | tools native |

`force_constrained: true` em DAG node falha-imediato se backend não suporta.

---

## 5. Embeddings strategy

### 5.1 V1: sem embeddings

`memory_search`: grep apenas. Repo map: tree-sitter symbols.

Justificativa:
- grep + repo map cobrem 90% dos casos em código
- Embeddings adicionam complexidade (modelo extra, dimension, similarity ops)
- Privacy concern em embeddings cloud

### 5.2 V2 deferred

Quando puxar:
- Demanda real de semantic search além de literal grep
- Sinal: usuários reclamando "não acha conceito X"

Backends candidatos:
- **Ollama embeddings** (recente; `nomic-embed-text`, `mxbai-embed-large`)
- **sentence-transformers** local (Python deps; subprocess)
- **OpenAI ada** (cloud; **viola privacy** em orchestrated; só hybrid)

### 5.3 Privacy em embeddings

Em profile `orchestrated` declarado:
- Embeddings **devem** ser local (Ollama embedding model ou sentence-transformers)
- Cloud embedding (`OpenAI ada`) **proibido** sem warning explícito
- `agent privacy verify` checa que nenhum embedding sai do host

Em profile `hybrid`:
- Embedding pode ser cloud (planner usa)
- Documentado como cross-boundary

### 5.4 Storage

SQLite + `sqlite-vec` extension (mencionado em arquitetura geral; deferred). Sem servidor extra.

---

## 6. Concurrent inference

### 6.1 Ollama setting

`OLLAMA_NUM_PARALLEL` env (default 1 ou 4 por versão).

Agent override:

```toml
[providers.ollama]
num_parallel = 4                       # default conservador
```

Recommendation hardware-aware:
- VRAM ≥ 24GB: `num_parallel = 4`
- VRAM 12-16GB: `num_parallel = 2`
- VRAM ≤ 8GB: `num_parallel = 1`
- CPU only: `num_parallel = 1`

### 6.2 Coordination com agent concurrency

Limites de ORCHESTRATION §11:
- `max_concurrent_subagents = 3` (default)
- `max_concurrent_llm_calls = 5` (DAG)

Se `num_parallel = 1` no Ollama, ainda funciona (agent enfileira). Hit do limite Ollama: requests aguardam slot.

### 6.3 llama.cpp HTTP server

```bash
llama-server -m model.gguf --n-parallel 4 --ctx-size 8192
```

`n_parallel`: max concurrent inferences (compartilham KV cache).

### 6.4 Effective parallelism

`min(agent.max_concurrent_llm_calls, ollama.num_parallel)` é o real cap.

UI mostra `<ConcurrencyIndicator>` quando hitting limit (raro em dev local).

---

## 7. Context shift behavior

### 7.1 O problema

Ollama (e alguns llama.cpp configs) fazem **context shift** automático quando context > window:
- Descarta tokens do **começo** (sliding window)
- Modelo perde system prompt, project context
- Comportamento degrada silenciosamente

### 7.2 Agent strategy (preventiva)

Token count antes de cada send:

```ts
const tokens = await provider.countTokens(messages);
if (tokens > window * 0.9) {
  // dispara compaction agressivo
  await compact(messages, 'aggressive');
}
if (tokens > window * 0.95) {
  // ainda não cabe — abort
  throw new FailureEvent('local.context.exceeded', ...);
}
```

### 7.3 Compaction trigger ajustado

PERFORMANCE.md §17 workload "Long-context": trigger 50% (vs 70% default em frontier).

Local com window 8k: compaction muito mais frequente que frontier 200k. Performance budget reflexa isso.

### 7.4 Detection ex-post

Se context shift ocorrer (não preventiva): detection via:
- Modelo ignora system prompt (sintoma comum)
- Modelo perde memória de turn anterior
- Eval cobre

Aviso ao user com hint explícito de compactar.

---

## 8. Prompt template dialects

Cada family de modelo tem template próprio. Tabela canônica:

### 8.1 Llama-3 family

```
<|begin_of_text|><|start_header_id|>system<|end_header_id|>
{system}
<|eot_id|><|start_header_id|>user<|end_header_id|>
{user}
<|eot_id|><|start_header_id|>assistant<|end_header_id|>
```

Tool calling: via XML adapter (§3); não-native.

Identifier: model name contains `llama-3` or `llama3`.

### 8.2 Qwen 2.5 family

```
<|im_start|>system
{system}<|im_end|>
<|im_start|>user
{user}<|im_end|>
<|im_start|>assistant
```

Tool calling: native em `qwen-coder`; adapter em `qwen` base.

Identifier: model name contains `qwen` or `qwen2`.

### 8.3 DeepSeek-Coder family

Conversational:
```
<|im_start|>system
{system}<|im_end|>
<|im_start|>user
{user}<|im_end|>
<|im_start|>assistant
```

FIM (Fill-In-Middle) tokens (útil em completion):
```
<|fim_begin|>{prefix}<|fim_hole|>{suffix}<|fim_end|>
```

Identifier: model name contains `deepseek-coder`.

### 8.4 Mistral family

```
<s>[INST] {user} [/INST] {assistant}</s>
```

Tool calling: native em recentes; adapter em older.

Identifier: model name contains `mistral` or `mixtral`.

### 8.5 Codestral

Mistral-based mas fine-tuned pra code. Mesmo template Mistral.

### 8.6 Granite (IBM)

```
<|start_of_role|>system<|end_of_role|>{system}<|end_of_text|>
<|start_of_role|>user<|end_of_role|>{user}<|end_of_text|>
<|start_of_role|>assistant<|end_of_role|>
```

### 8.7 Auto-detection

Agent detecta dialect via model name prefix:
- `llama-3*` / `llama3*` → llama3
- `qwen*` / `qwen2*` → qwen
- `deepseek-coder*` → deepseek
- `mistral*` / `mixtral*` → mistral
- `codestral*` → mistral (Codestral é Mistral-based)
- `granite*` → granite

Override via config:

```toml
[models."ollama/custom-model"]
prompt_template = "llama3"
```

### 8.8 Tabela de quirks per family

| Family | Tool calling | Multi-turn quality | Context typical | FIM |
|---|---|---|---|---|
| Llama-3 | adapter | bom | 8k-128k | não |
| Qwen 2.5 | native (coder) / adapter | excelente em code | 32k | sim (coder) |
| DeepSeek-Coder | adapter | bom em code | 16k | sim |
| Mistral | native (recente) / adapter | bom | 32k | não |
| Codestral | adapter | excelente em code | 32k | sim |
| Granite | adapter | bom | 8k | não |

---

## 9. Setup / bootstrap

### 9.1 `agent setup local`

Comando interativo:

```bash
$ agent setup local

Setup local model environment

  [1/5] Hardware probe...
        GPU:  NVIDIA RTX 4090 (24GB)
        CPU:  32 threads, 64GB RAM

  [2/5] Recommended model:
        ▶ qwen2.5-coder:14b @ Q4_K_M (8GB VRAM, balanced)
          qwen2.5-coder:7b @ Q4_K_M (4GB, faster)
          qwen2.5-coder:32b @ Q4_K_M (18GB, best quality)

  [3/5] Ollama daemon... ✓ running
        Disk space: 1.2TB free, 8.2GB needed

  [4/5] Pull model? [Y/n] y
        Downloading qwen2.5-coder:14b... ████████████ 100% (8.2GB / 4m23s)

  [5/5] Warm-up + smoke test...
        Loading model... ✓ (12s)
        Smoke prompt → response... ✓

✓ Local setup complete. Run `agent --profile orchestrated` to start.
```

### 9.2 `agent doctor` runtime check

Já em spec (§19 v1 promovido). Adiciona checks pra local:

```
$ agent doctor

✓ Ollama daemon: running (localhost:11434)
✓ Model qwen2.5-coder:14b: available, last used 2h ago
✓ GPU: 22.5/24 GB available
✓ num_parallel: 4 (configurado)
✓ Disk: 1.1TB free
⚠ Model qwen2.5-coder:32b: configured but not pulled
   Fix: `ollama pull qwen2.5-coder:32b` or remove from config
```

### 9.3 Auto-pull comportamento

Default: **NÃO** auto-pull. Pulls são bandwidth + disk caro; user decide.

Exceção opt-in:

```toml
[providers.ollama]
auto_pull = true
```

Em sessão onde modelo não-baixado: erro fatal `local.model.not_loaded` com hint `ollama pull <model>`.

---

## 10. Remote Ollama / LAN

### 10.1 Configuração

Workstation potente em casa, laptop pra coding:

```toml
[providers.ollama]
host = "http://192.168.1.10:11434"
```

Ou via env: `OLLAMA_HOST=http://192.168.1.10:11434`.

### 10.2 Trust prompt para remote host

LAN host **não é localhost**. Trust prompt aplica:

```
⚠ Provider Ollama remoto detectado: 192.168.1.10:11434

Sua sessão vai enviar contextos pra esse host.
Você confia neste host?

  - host name: workstation.lan
  - última verificação: nunca

[t]rust  [r]eject  [i]nspect
```

Hash do `model + host + version` cached em `~/.config/agent/trusted_hosts`.

### 10.3 Latência implications

- localhost: sub-ms TCP overhead
- LAN gigabit: 1-3ms RTT
- Wi-Fi: 5-30ms RTT, jitter
- VPN: + 50-200ms

SLOs ajustam: `network.keepalive_idle_ms = 300000` em LAN setup.

### 10.4 Privacy posture

Remote LAN **não é** local-only:
- Contextos saem do laptop
- LAN traffic interceptable se hostile
- Documentar implication em §12

---

## 11. Failure modes específicas

Adicionado a `FAILURE_MODES.md` como classe `local.*`:

### 11.1 `local.model.not_loaded`

**Detecção:** API retorna erro "model not found" ou agent verifica `ollama list` antes.

**Recovery:**
- Mensagem com path pra fix: `Run: ollama pull <model>`
- Em `auto_pull: true`: pull automático com progress bar
- Em headless: erro fatal exit 4

### 11.2 `local.model.load_timeout`

**Detecção:** primeira request excede `preload_timeout_ms` (default 60s).

**Recovery:**
- Aborta load
- Mensagem: "Modelo demorou > 60s pra carregar. Hardware suficiente? `agent doctor --hardware`"
- Sessão entra em error_fatal

### 11.3 `local.model.oom`

**Detecção:** API retorna OOM ou GPU CUDA error mid-inference.

**Recovery:**
- Mensagem: "GPU sem memória durante inference. Tente quantization menor (Q4_K_M → Q3) ou modelo menor."
- Sessão salva, marca step como failed
- Hint pra `agent local recommend --hardware-strict`

### 11.4 `local.gpu.unavailable`

**Detecção:** GPU detection mid-session falha (driver crash, GPU removed, etc).

**Recovery:**
- Fallback pra CPU (com warning de latência)
- Sessão pode continuar mas eval cai dramaticamente

### 11.5 `local.context.exceeded`

**Detecção:** Token count pré-send excede 95% do window mesmo após compaction.

**Recovery:**
- Aborta step
- Mensagem: "Contexto excedeu janela do modelo (8k tokens). Considere `/compact` ou modelo com janela maior."
- User pode `/model qwen2.5-coder:32b` (32k window) e resumir

### 11.6 `local.adapter.parse_failure`

**Detecção:** Tool calling adapter parse falhou após 2 retries.

**Recovery:**
- Step marca tool_call como `error: tool.adapter.parse`
- Modelo recebe ToolError estruturado (CONTRACTS §2)
- Modelo decide próxima ação
- Audit log: hint pra evaluar adapter

### 11.7 `local.constrained.malformed`

**Detecção:** GBNF/JSON mode retornou malformed após 2 retries.

**Recovery:**
- Igual `local.adapter.parse_failure`
- Eval offline detecta padrão; modelo entra em quarentena se taxa > 10%

---

## 12. Privacy guarantees verificáveis

### 12.1 Em profile `orchestrated`

Garantias declaradas:
- Modelo executa local (Ollama localhost ou llama.cpp localhost)
- Telemetry exports **disabled by default**
- DNS resolve **apenas localhost** + hosts em allowlist (`web_fetch.allow_hosts`)
- Memory body **nunca sai** do disco local
- Crash dumps **redact paths**
- Embedding (se v2) usa modelo local

### 12.2 `agent privacy verify`

Comando que checa configuração runtime:

```bash
$ agent privacy verify

Profile: orchestrated
  ✓ Provider host: localhost:11434 (not remote)
  ✓ Telemetry export: disabled
  ✓ web_fetch deny_hosts: includes external IPs
  ✓ Memory writes: local only (.agent/memory/)
  ✓ No embeddings configured

Profile guarantees: HONORED

To verify network behavior:
  Run with: tcpdump -i any port 11434 (sniff localhost only)
  Or: strace -f -e network agent --profile orchestrated ...
```

### 12.3 Em profile `hybrid`

**Privacy não é mesma garantia.** Hybrid usa frontier model (cloud) pra planner/fallback. Documentado como cross-boundary:

```bash
$ agent privacy verify --profile hybrid

⚠ Profile hybrid: contextos saem do host

  - Planner model: anthropic/haiku-4-5 (cloud)
  - Executor model: ollama/qwen2.5-coder:14b (local)
  - Fallback model: anthropic/sonnet-4-6 (cloud)

  Contextos enviados pra cloud em:
    - Initial planning step
    - Fallback steps quando local falha 2x

  Isso é INTENCIONAL em hybrid. Use orchestrated para privacy total.
```

### 12.4 LAN remote

Remote LAN host (não localhost) **quebra** garantia local-only. Trust prompt avisa explícito.

### 12.5 Limites declarados

Mesmo em orchestrated com localhost-only:
- OS pode logar (kernel-level tracing)
- Telemetria do Ollama em si (não-controlada pelo agent)
- Modelo carregado em VRAM compartilhado (não isolado)

Honesto: privacy contra **rede** + **disco do agent**, não contra adversário com root local.

---

## 13. Performance tuning consolidated

Cross-refs a docs existentes:

- **PERFORMANCE.md §11.4** — Ollama keep_alive, llama.cpp GBNF compilation cache
- **PERFORMANCE.md §13** — caching strategy (memory index, file content, repo map)
- **PERFORMANCE.md §15** — memory optimization techniques (object pool, lazy require)
- **PERFORMANCE.md §17** — workload tunings (local-first orchestrated)

### 13.1 Tunings rápidos

```toml
# ~/.config/agent/config.toml — preset orchestrated otimizado

[profile]
default = "orchestrated"

[providers.ollama]
host = "http://localhost:11434"
num_parallel = 4
timeout_ms = 60000

[providers.ollama.lifecycle]
keep_alive_active_session = "-1"
preload_on_session_start = true

[compaction]
trigger_threshold = 0.5             # mais agressivo em window pequeno
model = "ollama/qwen2.5-coder:7b"   # cheap pra compact

[orchestrated]
max_concurrent_llm_calls = 4        # bate em num_parallel

[network]
keepalive_idle_ms = 60000
```

---

## 14. Eval específico para local

Cross-ref a `PROVIDERS.md §7`. Tier "Local" com canonical model `ollama/qwen2.5-coder:14b`.

### 14.1 Smoke pass rate threshold

- Frontier (Sonnet): ≥ 85%
- Mid (Haiku): ≥ 75%
- **Local (qwen-coder-14b): ≥ 60%**

### 14.2 Eval pra adapter

Eval específico em `evals/local/adapter/`:
- Tool calling format compliance (parse rate)
- Schema adherence (validation rate)
- Multi-turn coherence
- Context shift detection

Threshold: parse ≥ 90%, validation ≥ 85%. Modelo abaixo: marcado `recommended: false`.

### 14.3 Hardware variation

Eval roda em 3 hardware tiers (CI):
- High: GPU 24GB (Q4)
- Medium: GPU 8GB (Q4)
- Low: CPU only (16GB RAM)

Resultados por tier publicados; user vê o que esperar.

---

## 15. Small Model Strategy (defense in depth)

Modelo pequeno (7B-14B) **alucina mais**, **esquece mais**, **erra format mais** que frontier. Spec tem mecanismos espalhados pra mitigar; esta seção é o **mapa de como compõem** em sistema de defesa em profundidade.

A regra raiz: **substitua confiança no modelo por verificação externa**. Cada layer transforma "modelo decidiu X" em "modelo propôs X, externamente confirmamos viabilidade".

### 15.1 Defense in depth — visão estratégica

```
Modelo gera output
  ↓
[Layer 1] Constrained generation (token-level)
   GBNF / JSON mode / tools / adapter regex
  ↓
[Layer 2] Schema validation (output-level)
   JSON Schema check via validators
  ↓
[Layer 3] Fact-checking (semantic-level)
   FileExists, GrepValidator, ASTValidator
  ↓
[Layer 4] Permission (policy-level)
   Allow/deny/confirm — bloqueia ações fora do esperado
  ↓
[Layer 5] Test gate (behavior-level, em refactor)
   Roda testes; falha = revert
  ↓
[Layer 6] Self-critique opt-in (peer-review level)
   Segundo modelo crítica antes de persist
  ↓
[Layer 7] Hybrid fallback (model-level)
   Escala pra frontier em N falhas consecutivas
  ↓
Output validado → persist
```

Layer 1 é o mais barato (zero LLM call extra). Layers 5-7 são caros mas alta confiança. Composição = qualidade aceitável em modelo small.

### 15.2 State maintenance — 5 mecanismos

Modelo small **esquece** rápido. Spec mantém estado externalizado em 5 lugares:

| Mecanismo | Onde mora | Re-injetado quando |
|---|---|---|
| **Goal re-injection** | `current_turn` bottom (`CONTEXT_TUNING §10`) | A cada 5 steps em sessão > 15 turns; sempre após critique abort/interrupt resume; literal byte-by-byte |
| **TodoList tool state** | tabela `todo_items` + UI live (`AGENTIC_CLI §7.4`) | Visível no contexto sempre; modelo lê/escreve via `todo_write` |
| **DAG step_outputs** | tabela `step_outputs` em SQLite (`ORCHESTRATION §2`) | Cada node consome via `inputs_from`; estado entre nodes nunca em memória do modelo |
| **Memory eager index** | `memory_index` section (cache breakpoint #4, `MEMORY §4`) | Sempre presente; preferências não dependem de "lembrar" |
| **Repo map eager** | `repo_map` section em orchestrated (`AGENTIC_CLI §6`) | Sempre presente; estrutura do código externalizada |

**Princípio:** o que o modelo "precisa lembrar" mora no prompt, não na sua atenção.

### 15.3 Anti-hallucination — layers concretas

Tabela de defesas por tipo de alucinação:

| Tipo de alucinação | Defesa primária | Defesa secundária |
|---|---|---|
| Inventa arquivo/função inexistente | Repo map eager + `FileExistsValidator` + `GrepValidator` (§15.6) | Permission engine bloqueia path traversal |
| Inventa tool name | Tool palette restrita + tool schemas eager | Adapter regex parser detecta + retry com hint |
| Args malformados (JSON) | Constrained generation (GBNF/JSON mode) | Validator parse + retry com hint |
| Esquece goal mid-session | Goal re-injection literal | Compaction preserva goal sem resumir |
| Esquece constraint mid-step (multi-tool) | Re-injection no tool result observation (§15.5) | Constraints negativas em system prompt |
| Output format drift | Output schema obrigatório com `not_checked` + `assumptions` | Self-critique pass detecta |
| "Cargo cult fix" (consertar sem entender) | Test gate em refactor | `confidence: speculation` exposto |
| Renomeia exportado sem listar callers | Constraint negativa + `GrepValidator` antes do edit | Checkpoint permite revert |
| Inventa import path | `ImportValidator` (§15.6) | Test gate pega em runtime |
| Recomenda baseado em memória stale | `verify-before-act` em memory factual | Memory expires + re-prompt |

### 15.4 Output quality preservation

Pra modelo small entregar qualidade frontier-acceptable:

1. **Constraints negativas explícitas** (não positivas) — "NÃO faça X" mais robusto que "Faça Y"
2. **Output schema com `confidence`** — força modelo a admitir incerteza em vez de chutar
3. **Output schema com `not_checked`** — força modelo a declarar limites do que viu
4. **Retry com hint específico** — falha vira input pro próximo (não generic "try again")
5. **Test gate** em workflows write-heavy — comportamento observável > opinião do modelo
6. **Few-shot por workflow** — formato consistente reduz drift
7. **Per-model sampling** (`TOKEN_TUNING §11`) — top_k 40 em llama, repetition_penalty 1.1 em base llama
8. **Per-step max_tokens** — output budget enxuto reduz divagação

### 15.5 State within multi-tool step (gap-fill)

Spec original: goal re-injection é **entre steps**. Mas modelo small em step com múltiplos `tool_use` esquece **dentro do step**:

```
Step N:
  tool_use_A → tool_result_A → ✓
  tool_use_B → ??? (modelo já esqueceu constraints?)
```

**Estratégia:** re-state goal/constraint compacto no início de **cada tool result observation**:

```
[tool_result do tool_use_A]
─
Goal (reminder): {goal_compact}
Constraints in scope: NÃO {top 3 constraints}
─
[próximo turn do modelo]
```

Custo: ~50-100 tokens por observação. Ganho: drift mid-step cai dramaticamente.

Configurável:

```toml
[orchestrated.multi_tool_step]
inject_reminder_per_tool_result = true     # default true em local
reminder_format = "compact"                 # compact | full
```

Anti-pattern: re-state goal **completo** a cada observação — bloat absurdo. Use `goal_compact` (1-line summary).

### 15.6 Fact-checking validators (gap-fill)

Validators atuais (`§7.5` AGENTIC_CLI) cobrem schema/file/AST. Pra modelo small, precisa **fact-checking semântico** específico de código:

```ts
// Novos validators built-in:

GrepValidator(pattern, path?): {
  ok: bool,
  matches: string[],
  hint?: string                  // ex: "símbolo `validateToken` não existe em src/auth.ts"
}

TypeValidator(typeName, file): {
  ok: bool,
  found_in?: string,
  hint?: string                  // ex: "type `Session` não exportado de @/types"
}

ImportValidator(importPath, fromFile): {
  ok: bool,
  resolved_to?: string,
  hint?: string                  // ex: "import path `@/lib/queue` não resolve"
}

SchemaReferenceValidator(ref, schemaFile): {
  ok: bool,
  found?: object,
  hint?: string
}
```

Aplicação:
- Em qualquer tool com `writes: true`: validators rodam **antes** do checkpoint
- Falha → retry com hint específico (não generic "try again")
- Confirmed via grep/AST = modelo não pode inventar

Custo: < 100ms por validation (deterministic, sem LLM).

### 15.7 Slot-filling output forcing (gap-fill)

Pra schema rígido, modelo small ainda alucina campos. Strategy: **slot-filling** — gera campo a campo, schema enforced em cada.

Em vez de:

```
Generate output matching this schema: {...}
[modelo gera tudo de uma vez; alucina campo]
```

Usa:

```
Step 1: Generate `summary` field (string, ≤ 200 chars):
[gera apenas summary; validate]

Step 2: Generate `findings` array:
[gera findings; validate cada item]

Step 3: Generate `not_checked` array:
[gera; validate]

Step 4: Compose final output:
[montagem deterministic em código TS, não no modelo]
```

3-4× chamadas LLM, mas:
- Cada chamada é **muito mais simples** (modelo small acerta facilmente)
- Schema enforced em cada slot via constrained generation
- Composição final é deterministic (não pode alucinar)

**Quando usar:**
- Modelo < 14B
- Schema com 5+ campos
- Workflow crítico (audit, refactor plan)
- Eval mostra schema violation > 10% sem slot-filling

Configurável:

```toml
[orchestrated.slot_filling]
enabled = true                              # default em modelo small
threshold_field_count = 5                   # slot-fill se schema tem ≥ 5 campos
```

Não usado em frontier (caro sem ganho).

### 15.8 Decision tree — quando usar qual estratégia

```
Modelo escolhido?
  ├─ Frontier (Sonnet+ class) → autonomous profile; layers 1-2 suficientes
  ├─ Mid-tier (Haiku, gpt-4o-mini) → autonomous + layer 3 (validators)
  ├─ Local 30B+ (Qwen-32B) → orchestrated; layers 1-4 + DAG
  ├─ Local 14B (Qwen-coder-14B) → orchestrated; layers 1-5 + slot-filling em workflows críticos
  ├─ Local 7B → orchestrated; layers 1-7 obrigatórios + hybrid fallback
  └─ Local < 7B → hybrid only (planner frontier, executor local em tasks triviais)

Workflow crítico (security, refactor multi-arquivo)?
  ├─ Sempre layer 5 (test gate) onde aplicável
  ├─ Layer 6 (self-critique) opt-in fortemente recomendado
  └─ Layer 7 (hybrid fallback) com threshold baixo (1 falha = escala)

Eval mostra hallucination rate alta em workflow X?
  ├─ Adicionar validator específico (GrepValidator se inventando funções)
  ├─ Aumentar threshold de retry
  ├─ Considerar slot-filling se schema-related
  └─ Considerar hybrid fallback se persistente
```

### 15.9 Hybrid escalation — last-resort declarado

Quando layers 1-6 falham, layer 7 (hybrid) escala pra frontier.

Triggers default:

```toml
[hybrid.escalation]
on_validator_failure_count = 2              # 2 retries falhados em layer 2/3
on_format_drift = true                      # output schema violation persistente
on_stale_state = true                       # modelo emitiu mesma tool 3× (degenerate loop)
on_explicit_request = true                  # user pode forçar via /escalate
```

Comportamento:
- Step atual aborta no executor local
- Frontier model pega o **mesmo input** + hint do que falhou
- Output do frontier vira novo step
- Audit registra `failure_events.escalation` com motivo

**Não é falha; é design.** Modelo small + frontier supervisão > frontier puro em alguns workflows (custo).

### 15.10 Métrica de saúde

`agent stats --small-model-health`:

```
Small model strategy health (last 7d):
  Sessions in orchestrated:        47
  Layer 1 (constrained gen):       100% applied
  Layer 2 (validators):            failures 12% → retry success 89%
  Layer 3 (fact-checking):         caught 8 hallucinations (file/function inventados)
  Layer 4 (permission denies):     3 (path traversal attempts)
  Layer 5 (test gate):             24 runs, 19 pass first try, 5 reverted
  Layer 6 (self-critique):         disabled in 38, enabled in 9 (caught 2 issues)
  Layer 7 (hybrid escalation):     fired 4× (8.5% of sessions)

Hallucination rate: 4.2% (target < 5%)
Cost vs autonomous-frontier: -67%
```

Threshold > 5% hallucination rate sustained = warning; revisar layers ou modelo.

---

## 16. Anti-patterns

| Anti-pattern | Por quê ruim |
|---|---|
| Pretender qualidade frontier em modelo 7B | Eval mostra; user frustrado |
| Sem keep_alive: reload em cada step | Latência cumulativa absurda |
| Sem warm-up: primeira request lenta sem feedback | UX horrível em cold start |
| Tool calling adapter ad-hoc por implementador | Qualidade variável; eval não cobre |
| Context shift silencioso | Bug em produção; modelo "esquece" sem aviso |
| Embeddings cloud em "local-first" | Quebra privacy claim |
| Hardware-blind recommendation | User com 4GB GPU recebe sugestão de modelo 70B |
| Ignorar OOM como erro graceful | Sessão silenciosamente errada |
| Auto-pull default (bandwidth surpresa) | User em mobile data: GB consumidos sem aviso |
| Trust automático em LAN host | Vetor de ataque |
| Privacy claim sem verify | Promessa não-checável |

---

## 17. Insight final

Local não é "frontier mais barato". É **trade-off explícito** com perfil distinto:

- **Quality**: degrada com tamanho do modelo; medido em eval
- **Latency**: latency-zero rede mas warm-up 5-30s; depende de hardware
- **Cost**: $0 mas time-cost real
- **Privacy**: garantia verificável, não declarada
- **Setup**: hardware-aware obrigatório
- **Workflows**: cobertura honesta (read-heavy ✓, complex multi-step refactor ✗)

Spec madura **declara** o trade-off. Spec imatura **esconde** atrás de "supportamos local".

A regra é: **honestidade epistêmica > marketing**. Local funciona excellently em workflows certos com hardware certo. Em outros: hybrid escala graciosa, ou frontier honesto.

E como tudo no projeto: **meça duas vezes, corte uma**. Hardware probe mede antes de pull. Adapter mede output antes de invoke. Compaction mede tokens antes de exceder window. Privacy verify mede config antes de declarar local-only. Cada decisão é instância da premissa raiz aplicada ao espaço local.
