# TOKEN_TUNING

Spec operacional sobre **tuning de geração**: sampling parameters, output limits, reasoning effort, stop sequences, truncation strategies, tokenizer accuracy.

`PERFORMANCE.md` cobre **token counting + budgeting** (custo, cache, compaction). Este doc cobre **token tuning** — como o modelo gera, com que parâmetros, com que controle.

Sem tuning declarado: eval não-reprodutível, cost não-otimizado, quality variável entre runs.

---

## 0. Princípios (não-negociáveis)

1. **Tuning é per-workflow, não global.** Review precisa baixa temperatura; brainstorm precisa alta. Default global é compromisso ruim.
2. **Defaults conservadores.** Quando em dúvida, escolher menor variância (temperature lower, top_p lower).
3. **Trade-offs documentados.** Cada parâmetro tem trade-off declarado (qualidade vs custo vs latência vs determinismo).
4. **Reprodutibilidade declarada.** `seed` quando disponível; `temperature: 0` em eval.
5. **Reasoning effort é cost-conscious.** Extended thinking custa real; default conservador.
6. **Output budget per call**, não global. `max_tokens` ajustado por step.
7. **Tokenizer accuracy é honesta.** Margem de erro declarada; sem cap rígido sem buffer.
8. **Cada decisão honra a premissa raiz** — sampling configura "antes do corte"; eval mede o corte; tuning ajusta com evidência.

---

## 1. Sampling parameters

### 1.1 Tabela canônica

```ts
interface SamplingParams {
  temperature?: number              // 0.0 - 2.0; default per-workflow (§9)
  top_p?: number                    // 0.0 - 1.0; default 1.0 (sem cutoff)
  top_k?: number                    // 1 - inf; default null (provider-specific)
  repetition_penalty?: number       // 1.0 = neutral; >1 penaliza repetição (Ollama, llama.cpp)
  frequency_penalty?: number        // -2.0 - 2.0 (OpenAI); default 0
  presence_penalty?: number         // -2.0 - 2.0 (OpenAI); default 0
  min_p?: number                    // probabilidade mínima cutoff (alguns models); default null
  seed?: number                     // reprodutibilidade onde provider suporta
}
```

### 1.2 Significado e trade-offs

| Param | Efeito | Range típico | Trade-off |
|---|---|---|---|
| `temperature` | controla "criatividade" — sampling distribution | 0.0 (greedy) - 2.0 (chaos) | baixa = determinístico, repetitivo; alta = variado, alucinatório |
| `top_p` | nucleus sampling — corta cauda de probabilidade | 0.9 - 1.0 típico | baixo = conservador; alto = mais opções (~ temperature) |
| `top_k` | top-k sampling — N candidatos | 40 (llama) | redundante com top_p; usado em alguns local models |
| `repetition_penalty` | desencoraja repetição literal | 1.0 - 1.3 | >1.2 distorce gramática |
| `frequency_penalty` | penaliza tokens já usados (OpenAI) | -2.0 - 2.0 | positivo evita repetir; negativo encoraja |
| `presence_penalty` | penaliza tokens presentes (OpenAI) | -2.0 - 2.0 | positivo encoraja diversidade |
| `min_p` | cutoff probabilístico | 0.05 típico | substituto de top_p mais robusto em alguns models |
| `seed` | reprodutibilidade | int | nem todo provider honra; eval crítico |

### 1.3 Defaults conservadores agent-wide

Quando **não declarado** em workflow:

```toml
[sampling.defaults]
temperature = 0.2          # baixa — agente é factual por default
top_p = 0.95
top_k = null               # null = provider default
repetition_penalty = 1.0   # neutral
seed = null                # não-determinístico por default
```

**Override hierárquico:**
```
session flag (volátil)
  → playbook frontmatter
  → workflow-specific config
  → agent global
```

### 1.4 Anti-pattern: sampling implícito

Não declarar sampling = cada provider usa seu default (Anthropic temp 1.0, OpenAI 1.0). **Resultado: agent comporta-se diferente entre providers** com mesmo prompt — eval inconsistente.

Toda chamada provider explicita sampling. Sem fallback "use o default do provider".

---

## 2. Output budget per call

### 2.1 `max_tokens` por step

Default era 4096 (Anthropic) / variável (OpenAI). Tuning mais inteligente:

```toml
[sampling.max_tokens]
default = 4096                     # backstop
critique = 512                     # critique é curto
compaction = 8192                  # compaction precisa
tool_call_only = 1024              # se step só emite tool_use
```

Per playbook override:
- `code-review`: max_tokens 16384 (relatório longo)
- `refactor`: 8192 (plan + diffs)
- `debug`: 4096 (hipóteses)
- `audit`: 16384 (findings completos)
- `explain`: 8192 (explicação estruturada)

### 2.2 Auto-tune por contexto

Heurísticas no harness:

```ts
function recommendMaxTokens(step: Step): number {
  if (step.expectsToolCallOnly) return 1024;
  if (step.isCritique) return 512;
  if (step.isCompaction) return 8192;
  if (step.playbook) return playbookConfig[step.playbook].maxTokens;
  return defaults.maxTokens;
}
```

User pode override via flag: `agent --max-tokens 16384 ...`.

### 2.3 Por que importa

Custo: tokens de output são 5× input em Anthropic. `max_tokens` baixo onde aplicável corta custo significativo.

Latência: tempo de geração ∝ output tokens. `max_tokens: 1024` em vez de 4096 reduz latência ~3-4× se modelo encheria mesmo.

### 2.4 Hit do limite

Se output truncado por `max_tokens`:
- `stop_reason: 'max_tokens'`
- Não é erro — modelo cortou abruptamente
- Próximo step pode ou continuar com "continue" prompt ou aceitar

Em playbooks: se limite inadequado, eval cobre (output incompleto deve falhar validation).

---

## 3. Stop sequences

### 3.1 Uso canônico

```toml
[sampling.stop_sequences]
default = []
# adapter de tool calling em local: força respeitar boundary
local_tool_adapter = ["</args>"]
# anti-hallucination de tool fechamento
all = ["</invoke>"]
```

### 3.2 Em LOCAL_MODELS adapter

Pattern XML-style adapter (LOCAL_MODELS §3):
```
<tool>name</tool>
<args>{...}</args>
```

Stop sequence `</args>` força modelo parar imediatamente após fechar args, evitando lixo após tool call. Reduz parse failures em ~30%.

### 3.3 Per provider

| Provider | Stop sequence support | Notes |
|---|---|---|
| Anthropic | `stop_sequences: string[]` (até 4) | tools nativas raramente precisam |
| OpenAI | `stop: string[]` (até 4) | similar |
| Ollama | `stop: string[]` | depende de model |
| llama.cpp | `stop: string[]` | flexível |

### 3.4 Anti-patterns

- Stop sequence muito comum (ex: `\n`) — corta output cedo
- Stop sequence em tool name (ex: `bash`) — modelo não consegue mencionar a tool

Stop sequences cuidadosamente curadas; eval cobre.

---

## 4. Reasoning effort / thinking

### 4.1 Anthropic extended thinking

```toml
[sampling.thinking]
mode = "off" | "enabled"           # default off
budget_tokens = 4096               # default conservador
```

Levels recomendados (PERFORMANCE.md §11.4):

| Level | budget_tokens | Quando usar |
|---|---|---|
| `off` | 0 | maioria dos workflows; default |
| `conservative` | 4096 | tasks complexas mas não-deliberativas |
| `balanced` | 16384 | playbooks `audit`, `debug` complex, `refactor` multi-arquivo |
| `maximum` | 64000 | tasks de research/architecture (raras) |

### 4.2 OpenAI reasoning_effort

Em `o1`/`o3` models:

```toml
[sampling.openai_reasoning]
effort = "low" | "medium" | "high"   # default low
```

| Level | Cost multiplier | Quando |
|---|---|---|
| `low` | 1× | maioria; default |
| `medium` | 2-3× | tasks deliberativas |
| `high` | 5-10× | research/architecture |

### 4.3 Cost honesty

Reasoning tokens são **cobrados** mesmo se ocultos:
- Anthropic: `reasoning.cost_usd` em traces
- OpenAI: aparece em billing como tokens normais

UI mostra `🧠 thinking... (Xs, $Y)` durante reasoning ativo (LOCAL_MODELS via UI.md `<ThinkingIndicator>`).

### 4.4 Per playbook

| Playbook | Anthropic thinking | OpenAI reasoning |
|---|---|---|
| `code-review` | conservative | low |
| `security-audit` | balanced | medium |
| `debug` (complex hypotheses) | balanced | medium |
| `refactor` (multi-arquivo) | balanced | medium |
| `explain` | conservative | low |
| `compaction` | off | low |
| `critique` | off | low |

---

## 5. Multi-sample / `n` parameter

### 5.1 V1: single sample

Default: `n: 1` (uma resposta). Suficiente pra maioria.

### 5.2 V2 deferred: best-of-N com voting

Strategy quando puxado:
- Gera N samples (cost N×)
- Validator determinístico vota (test pass, schema valid, etc)
- Best-of-N pra code generation crítico

Útil em:
- DAG nodes com tests gate (gera 3 fixes; aceita primeiro que passa)
- High-stakes refactor (3 propostas; user escolhe)

Trade-off: 3× custo por 30-50% melhor qualidade em tasks específicas.

### 5.3 Quando NÃO usar

- Conversational steps (resposta única faz sentido)
- Workflows simples (cost não compensa)
- Low-stakes (overhead inútil)

---

## 6. Token-aware truncation strategies

### 6.1 Onde cortar quando contexto excede limite

PERFORMANCE.md §11.2 declara reserve. Estratégias por tipo de conteúdo:

| Conteúdo | Estratégia | Exemplo |
|---|---|---|
| Tool output longo | head + tail com pointer | `[150 lines truncated; re-read with offset=N]` |
| Long file read | head + tail (50 + 50 lines) | `[middle 800 lines truncated]` |
| Multi-tool result | drop least-recent | `[tool_result do step 3 truncated]` |
| Conversation history | compaction (§4 ORCHESTRATION) | summary + recent literal |
| Memory body grande | summary + reference | `(see ./.agent/memory/...)` |
| Repo map gigante | namespace fold | `[truncated; expand with repo_map(namespace="src")]` |

### 6.2 Tool output truncation rules

```toml
[truncation.tool_output]
inline_max_kb = 100              # > 100KB salvo em arquivo + pointer
head_lines = 50                  # mostra primeiras 50 linhas
tail_lines = 50                  # mostra últimas 50 linhas
preserve_errors = true           # error lines (warning/error) sempre preservadas
```

Em `bash` output: errors detectados via heuristic (`error:`, `panic:`, etc) sempre incluídos.

### 6.3 Importance-weighted truncation

Em compaction: tool results menos importantes (read_file de paths fora do escopo) cortam primeiro; decisões e findings preservam.

### 6.4 User override

Flag `--include-tool-output` em recap (RECAP.md §1) opta por full output em forensics. Default omite.

---

## 7. Context compression beyond compaction

### 7.1 Strategies além de compaction LLM

#### Reordering

Mover conteúdo importante early no prompt, descartável tail:
- System prompt (top, cached)
- Tool schemas (cached)
- Recent decisions (importante)
- Old tool outputs (descartável tail)
- Current turn (top novamente)

Ordering hint: cache breakpoints + content priority.

#### Selective inclusion

Modelo não precisa **todo** tool result em todo step. Heurísticas:
- Step N+5 raramente precisa de tool_result do step N — refactor cuts
- Tool result que virou decisão (registrada em `decisions`) pode ser elidido
- Files já editados (em `applied[]`) podem ter content elidido

#### Format compression

JSON em prompt: minify (sem indentação) economiza ~10-30% tokens.
```
Verbose:                          Compact:
{                                 {"key":"value","x":42}
  "key": "value",
  "x": 42
}
```

Trade-off: humano lendo trace tem mais dificuldade. Em prod ok; em dev opt-out.

### 7.2 Anti-patterns

- "Compactar agressivo" sem eval — perde decisão crítica
- Reorder rompendo cache breakpoints — cache miss
- Selective elision sem audit log — agent "esquece" silenciosamente

---

## 8. Tokenizer accuracy

### 8.1 Por provider

| Provider | Tokenizer | Accuracy |
|---|---|---|
| Anthropic | não publicado | estimativa via heurística (~5% error) |
| OpenAI | `tiktoken` (público) | preciso (~ 0.5% error) |
| Ollama | per-model (sentencepiece, BPE) | preciso quando tokenizer correto |
| llama.cpp | per-model | preciso |

### 8.2 Margem de erro

Quando agent calcula `tokens_in` antes de send:
- Anthropic: assume +5% buffer (ie, count 95% do real cap)
- OpenAI: assume +1% buffer
- Local: count exato (mesma tokenizer)

### 8.3 Discrepância handling

Quando count local divergir do server-side billed:
- Log discrepância em `failure_events.code: 'tokenizer.discrepancy'`
- `agent stats --tokens` mostra ratio billed/counted
- Threshold > 10%: warning (tokenizer pode estar desatualizado)

### 8.4 Custom tokenizer override

Modelo recente sem suporte tiktoken: usuário pode declarar:

```toml
[providers.openai]
tokenizer_override = "cl100k_base"   # se não-detectado automatic
```

---

## 9. Per-workflow defaults (canonical)

Tabela canônica de tuning per workflow. Aplicada em playbook frontmatter.

| Workflow | Temperature | top_p | max_tokens | thinking | seed em eval |
|---|---|---|---|---|---|
| `code-review` | 0.1 | 0.95 | 16384 | conservative (Anthropic) | yes (eval determinístico) |
| `security-audit` | 0.0 | 0.9 | 16384 | balanced | yes |
| `debug` (initial) | 0.5 | 0.95 | 4096 | balanced | no (variance é feature) |
| `debug` (verify) | 0.0 | 0.9 | 2048 | conservative | yes |
| `refactor` (plan) | 0.3 | 0.95 | 4096 | balanced | yes |
| `refactor` (apply) | 0.1 | 0.95 | 8192 | conservative | yes |
| `explain` | 0.1 | 0.95 | 8192 | conservative | yes |
| `compaction` | 0.0 | 0.9 | 8192 | off | yes |
| `critique` | 0.0 | 0.9 | 1024 | off | yes |
| `recap` (LLM render) | 0.2 | 0.95 | 4096 | off | yes |
| Modo normal (sem playbook) | 0.2 | 0.95 | 4096 | conservative | no |

### 9.1 Justificativas

- **`code-review` temp 0.1**: factual; queremos consistência; mesma lógica em re-runs
- **`security-audit` temp 0.0**: determinístico; `confidence` honesta sobre incerteza, não variance no output
- **`debug` initial temp 0.5**: queremos hipóteses **diversas** pra evitar tunnel vision
- **`debug` verify temp 0.0**: confirmação precisa ser estável
- **`refactor` plan temp 0.3**: balanço entre criatividade e consistência
- **`compaction` temp 0.0**: deve ser determinístico (mesma history → mesmo summary)
- **`critique` temp 0.0**: factual; warnings consistentes

### 9.2 Override em playbook frontmatter

```yaml
---
name: code-review
description: ...
sampling:
  temperature: 0.1
  top_p: 0.95
  max_tokens: 16384
  thinking_budget: 4096
  seed_in_eval: true
---
```

Aplicado pelo harness em chamadas provider deste playbook.

---

## 10. Per-provider sampling matrix

Capability matrix por provider:

| Param | Anthropic | OpenAI | Ollama | llama.cpp |
|---|---|---|---|---|
| `temperature` | 0-1 | 0-2 | 0-2 | 0-2 |
| `top_p` | sim | sim | sim | sim |
| `top_k` | n/a | n/a | sim | sim |
| `repetition_penalty` | n/a | n/a | sim | sim |
| `frequency_penalty` | n/a | -2 a 2 | n/a | n/a |
| `presence_penalty` | n/a | -2 a 2 | n/a | n/a |
| `min_p` | n/a | n/a | sim (recente) | sim |
| `seed` | sim (best-effort) | sim | sim | sim (preciso) |
| `stop_sequences` | até 4 | até 4 | array | array |
| `n` (multi-sample) | 1 only | sim | sim | n/a |
| Reasoning/thinking | sim (Opus 4.x) | sim (o1/o3) | n/a | n/a |

### 10.1 Adapter responsibility

Provider adapter normaliza:
- `repetition_penalty` em Anthropic/OpenAI: ignored com warning
- `frequency_penalty` em Ollama/llama.cpp: aproximado via `repetition_penalty`
- `top_k` em Anthropic/OpenAI: ignored

Sem error em parâmetro não-suportado; warning em audit.

---

## 11. Per-model adjustments

### 11.1 Local models (especialmente)

Local models são **mais sensíveis** a sampling. Recommended specifics:

#### Llama-3 family
```toml
[models."ollama/llama3.1:8b".sampling]
temperature = 0.3
top_p = 0.9
top_k = 40                        # default llama3
repetition_penalty = 1.1          # Llama tende a repetir; corrige
```

#### Qwen 2.5 family
```toml
[models."ollama/qwen2.5-coder:14b".sampling]
temperature = 0.2
top_p = 0.95
top_k = null                      # qwen funciona melhor sem top_k
repetition_penalty = 1.0
```

#### DeepSeek-Coder
```toml
[models."ollama/deepseek-coder:6.7b".sampling]
temperature = 0.1                 # code precisa low temp
top_p = 0.95
top_k = 50
repetition_penalty = 1.0
```

#### Mistral / Codestral
```toml
[models."ollama/codestral:22b".sampling]
temperature = 0.2
top_p = 0.95
top_k = null
repetition_penalty = 1.05
```

### 11.2 Frontier model adjustments

Anthropic/OpenAI menos sensíveis a top_k/repetition_penalty. Tuning principalmente via temperature.

### 11.3 Sampling em validation

Em validators (PERFORMANCE.md §15) que chamam LLM (raro mas possível): temperature **sempre 0.0** + seed se disponível. Reprodutibilidade crítica.

---

## 12. Logit bias (deferred v2)

### 12.1 V1: não suportado

Logit bias permite favorecer/penalizar tokens específicos. Útil mas niche:
- Encourage tool name tokens (boost)
- Discourage profanity / unsafe outputs (suppress)
- Workflow-specific vocabulary

V1 não suporta. Documentar como hook hook futuro.

### 12.2 V2 quando puxar

Sinal:
- Eval mostra que adapter parse failure correlaciona com não-emitir tokens específicos
- Usuários querem suppress padrões (`<|im_end|>` em llama por exemplo)

V2 expor via:

```toml
[[sampling.logit_bias]]
when = { provider = "openai" }
biases = { "100264" = 50 }       # boost token id 100264
```

---

## 13. Eval-driven tuning

### 13.1 Quando ajustar

Tuning **não é arbitrário**. Baseado em:

1. **Eval regression** mostra quality drop em workflow X — tuning candidato
2. **Cost spike** em workflow Y — `max_tokens` ou `thinking_budget` candidato
3. **Reproducibility issue** — `seed` adequado?
4. **User reports** — variance percebida em playbook

Sem **uma destas evidências**: don't tune.

### 13.2 A/B testing tuning

```bash
agent eval --suite playbooks/code-review --tune temperature=0.0,0.1,0.2,0.5 --runs 5
```

Resultado:
```
temperature=0.0: 88% pass, $0.05 avg, 2.3s p50
temperature=0.1: 92% pass, $0.05 avg, 2.4s p50  ← winner
temperature=0.2: 87% pass, $0.05 avg, 2.5s p50
temperature=0.5: 71% pass, $0.06 avg, 3.1s p50
```

Aplicar winner; documentar decisão em audit.

### 13.3 Versioning

Sampling params são parte da `prompt_version`. Mudança em sampling = bump.

```yaml
# playbooks/code-review.md frontmatter
prompt_version: 3
sampling:
  temperature: 0.1
  ...
```

---

## 14. Anti-patterns

| Anti-pattern | Por que ruim |
|---|---|
| Sampling implícito (provider default) | Eval inconsistente entre providers |
| `temperature: 1.0` em workflow factual | Output varia 30%+; quality cai |
| `temperature: 0.0` em brainstorm | Sem diversidade; tunnel vision |
| `max_tokens: 4096` em todas chamadas | Cost desnecessário em chamadas curtas |
| Stop sequence muito genérica (ex: `\n`) | Output cortado prematuramente |
| Reasoning effort `high` por default | Custo 5-10× sem benefício na maioria |
| `n > 1` (multi-sample) sem validator | Custo N× sem ganho mensurável |
| Truncation arbitrária no meio de tool output | Modelo perde contexto; alucina |
| Tokenizer assumption sem buffer | Surprise overrun em production |
| Sampling change sem eval | Regressão silenciosa |
| Per-step override sem audit | Behavior emerge sem rastreio |
| Logit bias custom em v1 | Complexidade prematura |
| Compaction temperature alta | Summary varia entre runs |
| Critique temperature alta | Critic inventa issues |

---

## 15. Insight final

Tuning não é "mexer em parâmetros até funcionar". É **decisão informada** por workflow, evidência empírica em eval, declaração explícita pra reprodutibilidade.

Sem tuning declarado:
- Eval é ruído
- Cost é desperdício
- Quality é loteria
- Reprodutibilidade é mito

Com tuning declarado:
- Cada workflow tem perfil testado
- Cost é otimizado per call
- Quality é mensurável
- Eval reproduz exatamente

A regra é: **toda chamada provider tem sampling explícito; cada parâmetro tem justificativa empírica; cada mudança passa por eval.**

E como tudo no projeto: **meça duas vezes, corte uma**. Sampling é a medida de "como cortar tokens". Tuning sem eval é cortar sem medir.
