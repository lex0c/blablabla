# PROVIDERS

Catálogo formal de providers do `AGENTIC_CLI`. Interface canônica, capabilities matrix, quirks documentados, recomendações por workflow, eval strategy multi-model.

> **Princípio raiz:** *provider-pluggable, não provider-parity.* Adapters são intercambiáveis no nível da API; qualidade, custo, e features são heterogêneos e ficam declarados.

LangChain promete tudo, entrega bug em providers minoritários. LiteLLM promete API uniforme com quirks documentados; mais útil. Este doc segue LiteLLM.

---

## 0. Princípios

1. **Provider-pluggable, não provider-parity.** Adapters intercambiáveis no nível da chamada; qualidade não.
2. **Capabilities declaradas, não escondidas.** Cada provider expõe o que faz. Sem fingir features ausentes.
3. **Quirks documentados, não escondidos atrás de abstração.** Pretender uniformidade vira leaky abstraction.
4. **Defaults opinionados, não vendor-locked.** Recomendação por workflow vem de eval, não de marketing.
5. **Sem lowest-common-denominator.** Provider forte usa feature forte (cache, vision); provider limitado degrada graciosamente.
6. **Adapter é unidade isolada.** Quirks de um provider não vazam pra outros via core.
7. **Eval multi-model é first-class.** Sem eval em N providers, "agnostic" é declaração sem prova.

---

## 1. Provider interface (formal)

```ts
interface Provider {
  id: string                                  // único; e.g., "anthropic/sonnet-4-6"
  family: ProviderFamily                      // anthropic | openai | ollama | llama_cpp | google | mistral | ...
  generate(req: GenerateRequest): AsyncIterable<StreamEvent>
  generateConstrained(req: ConstrainedRequest): Promise<string>
  countTokens(messages: Message[]): Promise<number>
  capabilities: ProviderCapabilities
}

interface ProviderCapabilities {
  // Core features
  tools: ToolCallingMode | false              // native | adapted | false
  cache: CacheMode | false                    // server_5min | server_persistent | client_only | false
  vision: boolean
  streaming: boolean
  constrained: ConstrainedKind | false        // gbnf | json_mode | tools | regex | false
  sampling: SamplingSupport                   // ver TOKEN_TUNING.md §10
  
  // Limits
  context_window: number
  output_max_tokens: number
  
  // Hints for orchestration (não-bloqueantes)
  recommended_max_tools_per_step?: number     // 12 (frontier) | 3 (small) | 2 (very small)
  prompt_template_dialect?: PromptDialect     // claude | openai_chat | llama3 | qwen | deepseek | ...
  
  // Cost (USD per 1k tokens)
  cost_per_1k_input: number
  cost_per_1k_output: number
  cost_per_1k_cached_input?: number           // se cache server-side
  cost_per_1k_cache_write?: number            // alguns providers cobram pra escrever no cache
  
  // Operational
  max_rps?: number                            // rate limit default
  notes: string[]                             // quirks documentados pra runtime detection
}

type ToolCallingMode = 'native' | 'adapted'   // native: provider tem tools API; adapted: harness simula via prompt
type CacheMode = 'server_5min' | 'server_persistent' | 'client_only'
type ConstrainedKind = 'gbnf' | 'json_mode' | 'tools' | 'regex'
```

### 1.1 Stream events (canonical)

Provider adapter normaliza eventos do SDK pra:

```ts
type StreamEvent =
  | { kind: 'start'; message_id: string }
  | { kind: 'text_delta'; text: string }
  | { kind: 'tool_use_start'; id: string; name: string }
  | { kind: 'tool_use_delta'; id: string; partial_args: string }
  | { kind: 'tool_use_stop'; id: string; final_args: object }
  | { kind: 'thinking_delta'; text: string }   // se provider suporta extended thinking
  | { kind: 'stop'; reason: 'end_turn' | 'tool_use' | 'max_tokens' | 'stop_sequence' }
  | { kind: 'error'; code: string; message: string; retryable: boolean }
```

Adapter responsável por mapear quirks (Anthropic content blocks vs OpenAI choices vs Ollama plain stream) pra esse contrato.

---

## 2. Capability matrix (canônica)

Atualizada por release. Valores são representativos; ver `~/.config/agent/models.toml` ou `agent providers list` pra runtime current.

| Capability | sonnet-4-6 | haiku-4-5 | opus-4-7 | gpt-4o | gpt-4o-mini | qwen-coder-14b (ollama) | llama-3-8b (llama.cpp) |
|---|---|---|---|---|---|---|---|
| tools | native | native | native | native | native | adapted | adapted |
| cache | server_5min | server_5min | server_5min | client_only* | client_only* | none | none |
| vision | yes | yes | yes | yes | yes | no | no |
| streaming | yes | yes | yes | yes | yes | yes | yes |
| constrained | tools | tools | tools | tools+json | tools+json | json_mode | gbnf |
| context (k) | 200 | 200 | 200 | 128 | 128 | 32 | 8 |
| out max | 64k | 64k | 64k | 16k | 16k | 4k | 2k |
| max tools/step | 12+ | 12+ | 12+ | 12+ | 12+ | 3 | 2 |

\* OpenAI tem prefix-cache automático (não controlável; hit é probabilístico).

Provider adicionado depois faz PR atualizando esta tabela.

---

## 3. Quirks per provider (documentação honesta)

### 3.1 Anthropic

**Strengths:**
- Tool calling mais polido em 2026 (formato robusto, baixa alucinação)
- Prompt cache server-side com 5min TTL — economia 70%+ se layout certo
- Vision nativo em todos os models recentes
- Stream events bem-formados (raramente corrompidos)
- Extended thinking disponível em modelos recentes

**Quirks:**
- **Cache breakpoints declarados manualmente** — não é automático. Layout fixo do prompt obrigatório (§6 do AGENTIC_CLI.md).
- **TTL do cache é 5min hard**; cache 1h existe via flag `extended_cache` (preço diferente).
- **Tool use streaming**: emit é `content_block_start` (type=tool_use) → `input_json_delta` x N → `content_block_stop`. Parser precisa reconstruir args incrementais.
- **Rate limit**: `50 req/min` default no tier 1; burst window de 1min.
- **Erros 529 (overloaded)**: retry com backoff agressivo; não é 5xx tradicional.

**Defesas no adapter:**
- Reconstrução de args incrementais com validation no `tool_use_stop`
- Backoff exponencial específico pra 529
- Detecção e relabel de events que não estão no spec (futuro-proof)

### 3.2 OpenAI

**Strengths:**
- Structured outputs **garantido server-side** (JSON Schema enforcement nativo)
- Function calling maduro
- Models cheap (`gpt-4o-mini`) com capability razoável
- Reasoning models (`o1`, `o3`) disponíveis pra tasks deliberadas

**Quirks:**
- **Sem prompt cache controlável** — prefix-cache automático mas não-determinístico
- **Tool calling format ≠ Anthropic**: `tool_calls` array em message vs content blocks
- **Streaming com tool_calls**: pode chegar fragmentado em deltas; parser precisa montar
- **Rate limit varia por tier** drasticamente (tier 1 vs tier 5 muda 100×)
- **Reasoning tokens** (em `o1`/`o3`) não são streamed visíveis; cobra mas user não vê
- **Token counting**: tiktoken local mas precision é "best effort"

**Defesas no adapter:**
- Reconstrução de tool_calls a partir de fragments
- Tokenização via tiktoken cached
- Detecção de reasoning models e tratamento especial (sem stream visible)

### 3.3 Ollama

> **Detalhamento operacional:** [`LOCAL_MODELS.md`](./LOCAL_MODELS.md) — hardware detection, model lifecycle (load/warm-up/keep_alive), tool calling adapter formal, prompt template dialects, setup/bootstrap, remote LAN scenarios, failure modes específicas. Esta seção é overview de quirks.

**Strengths:**
- **Custo zero** (recurso local)
- **Privacidade** (nada sai do host)
- Funciona offline
- JSON mode em versões recentes
- API simples (compatible-mode com OpenAI também)

**Quirks:**
- **Tool calling não-nativo** na maioria dos models — precisa adapter (regex extract / few-shot prompt)
- **JSON mode falha em ~5-10%** dos casos — validation + retry obrigatório
- **Context window limitado** (8k-32k típico) — compaction agressiva
- **Latência alta em CPU**; aceitável em GPU
- **Quality degrada em multi-step reasoning** vs frontier — usar em DAG decomposto, não freeform
- **Prompts model-specific** (llama-3 ≠ qwen ≠ deepseek-coder ≠ codestral) — template dialect importa

**Defesas no adapter:**
- Adapter de tool calling via regex + retry (pattern: `<tool>name</tool><args>{...}</args>`)
- Validation pós-generation com retry-with-hint (≤2 retries)
- Auto-truncation de context se exceder window
- Template dialect detection via model name

### 3.4 llama.cpp

> Mesma autoridade de detalhe operacional em [`LOCAL_MODELS.md`](./LOCAL_MODELS.md). Esta seção é overview.

**Strengths:**
- **GBNF grammar nativo** — constrained generation mais flexível que JSON mode
- Performance excelente em hardware modesto (quantização)
- Total controle (server-side flags, sampling params)

**Quirks:**
- **HTTP server config separado** (não plug-and-play como Ollama)
- **API menos polished** — tool calling 100% via grammar, não nativo
- **Quirks de quantização** afetam qualidade (Q4_K_M ≠ Q8_0 ≠ FP16)
- **Streaming via SSE** com formato próprio

**Defesas no adapter:**
- GBNF templates pre-built pra schemas comuns (JSON, tool call format)
- Quantization detection via model metadata
- Documentação de Q levels recomendados por use case

### 3.5 Google Gemini (futuro, não em v1)

A documentar quando adapter for implementado. Notable: 1M context window; pricing model peculiar; cache different from Anthropic.

---

## 4. Recommended combos by workflow

**Importante:** essas recomendações vêm de **eval empírico**, não marketing. Atualizar quando eval mudar.

| Workflow | Primary | Fallback | Local-acceptable | Por quê |
|---|---|---|---|---|
| `code-review` | sonnet-4-6 | gpt-4o | ⚠ qwen-coder-14b com schema rígido | Nuance importa; modelo pequeno perde detalhe |
| `security-audit` | opus-4-7 | sonnet-4-6 | ✗ não recomendado | Mundo de conhecimento crítico; risco de falso negativo alto |
| `debug` | haiku-4-5 | sonnet-4-6 | ⚠ qwen-coder-14b para repro simples | Latência importa; iteração frequente; haiku é sweet spot |
| `refactor` (1-3 arquivos) | sonnet-4-6 | qwen-coder-14b (orchestrated) | ✓ qwen-coder-14b | Scope-bounded é local-friendly |
| `refactor` (multi-arquivo) | sonnet-4-6 | opus-4-7 | ✗ context window curto | Precisa de janela grande |
| Compaction | haiku-4-5 | gpt-4o-mini | qwen-coder-14b | Pequena task, alta frequência; barato e rápido |
| Format/lint fix | qwen-coder-14b | haiku-4-5 | ✓ ideal | Trivial; local resolve em 100% dos casos |
| Single rename | qwen-coder-14b | haiku-4-5 | ✓ ideal | Scope mínimo |
| One-off prompt (CLI) | haiku-4-5 | qwen-coder-14b | ✓ ideal | Optimize pra latência |

### 4.1 Profile defaults

| Profile | Provider default | Compaction default |
|---|---|---|
| `autonomous` | anthropic/sonnet-4-6 (configurable) | anthropic/haiku-4-5 |
| `orchestrated` | ollama/qwen2.5-coder:14b (configurable) | mesmo backend (ollama) |
| `hybrid` | planner: anthropic/haiku-4-5 · executor: ollama/qwen2.5-coder:14b · validator: deterministic · fallback: anthropic/sonnet-4-6 | anthropic/haiku-4-5 |

User override em `~/.config/agent/config.toml`.

---

## 5. Cost matrix

(USD por 1k tokens; pricing real fica em config dinâmico — atualizado por release)

| Provider/Model | Input | Output | Cached input |
|---|---|---|---|
| anthropic/opus-4-7 | $15.00 | $75.00 | $1.50 |
| anthropic/sonnet-4-6 | $3.00 | $15.00 | $0.30 |
| anthropic/haiku-4-5 | $0.25 | $1.25 | $0.025 |
| openai/gpt-4o | $2.50 | $10.00 | n/a |
| openai/gpt-4o-mini | $0.15 | $0.60 | n/a |
| ollama/* | $0 | $0 | n/a |
| llama.cpp/* | $0 | $0 | n/a |

### 5.1 Custo típico por sessão (estimativa)

| Tarefa | Provider | Custo estimado |
|---|---|---|
| `/review` (200 LOC) | sonnet-4-6 | $0.05 – $0.20 |
| `/audit` (200 LOC) | opus-4-7 | $0.30 – $1.00 |
| `/debug` (1 hipótese) | haiku-4-5 | $0.01 – $0.05 |
| Refactor 3 arquivos | sonnet-4-6 | $0.20 – $0.80 |
| Sessão completa 45min | sonnet-4-6 | $1 – $8 |
| Mesma sessão | qwen-coder-14b (orchestrated) | $0 (recursos locais) |
| Mesma sessão | hybrid | $0.30 – $2 |

Cache amortiza custo em sessão longa (>3 turns) em ~70% no input.

---

## 6. Adicionando provider novo (procedure)

PR-checklist:

1. **Implementar `Provider` interface** em `src/providers/<family>/`
   - `index.ts` (adapter)
   - `capabilities.ts` (declarações)
   - `quirks.md` (notas internas)
   - `prompts/` (templates dialect)

2. **Declarar capabilities honestamente** — sem ufanismo. Se `tools: 'adapted'`, dizer.

3. **Documentar quirks** em §3 deste doc.

4. **Adicionar templates de prompt** se dialect novo (`prompts/<dialect>/`).

5. **Implementar tool calling adapter** se `tools: 'adapted'`. Pattern padrão:
   - regex extraction
   - retry com hint em malformed args
   - validation contra `inputSchema` antes de invocar

6. **Implementar constrained generation backend** se aplicável.

7. **Rodar eval canônica** (§7). Publicar resultados em `evals/providers/<family>/results.json`.

8. **Atualizar matrix** em §2 e cost em §5.

9. **Recomendações em §4** se há workflow onde provider novo bate o atual.

10. **Submit PR** com:
    - Code do adapter (com testes)
    - Doc atualizado
    - Eval results vs current baselines
    - Justificativa: qual gap o adapter preenche?

Sem (3), (7), (8), (10): PR rejeitado.

---

## 7. Eval multi-model strategy

### 7.1 Canonical model set

Eval canônica roda em **3 tier representatives**:

| Tier | Modelo padrão | Substituível? |
|---|---|---|
| Frontier | anthropic/sonnet-4-6 | sim, qualquer modelo classe Sonnet/GPT-4 |
| Mid-tier | anthropic/haiku-4-5 | sim, qualquer modelo classe Haiku/gpt-4o-mini |
| Local | ollama/qwen2.5-coder:14b | sim, qualquer modelo local com tool adapter |

Provider novo entra na canonical set se passar **threshold do tier correspondente** (tabela 7.3) em smoke + regression eval.

### 7.2 Suite breakdown

| Suite | Casos | Frequência | Roda em |
|---|---|---|---|
| smoke | ~30 | todo PR | só frontier |
| regression | ~150 | diário em CI | 3 tiers |
| bench | ~500 | semanal | 3 tiers |
| playbook (review/audit/debug/refactor) | ~50 cada | weekly | 3 tiers |

### 7.3 Threshold por tier

```
                          | Frontier | Mid  | Local |
smoke                     |   ≥85%   | ≥75% |  ≥60% |
regression                |   ≥75%   | ≥65% |  ≥45% |
playbook (review/refactor)|   ≥80%   | ≥70% |  ≥50% |
playbook (audit)          |   ≥75%   | ≥60% | exclude |
playbook (debug)          |   ≥80%   | ≥75% |  ≥60% |
```

Falha em qualquer tier = release bloqueado. Se houver fix por provider, adapter ganha note `quirks.notes` e segue (eval cobre).

### 7.4 Reading the matrix

Output de eval em CI gera tabela públicada em `evals/results/<release>/`:

```
Release: v0.5.0
  smoke (autonomous):
    sonnet-4-6:        96% (29/30)  ✓
    haiku-4-5:         80% (24/30)  ✓
    qwen-coder-14b:    63% (19/30)  ✓
  regression:
    sonnet-4-6:        78%
    haiku-4-5:         68%
    qwen-coder-14b:    52%
  refactor playbook:
    sonnet-4-6:        82%
    haiku-4-5:         71%
    qwen-coder-14b:    54%
```

User vê: "se eu trocar pra qwen-coder-14b, refactor cai de 82% pra 54% — vale o custo zero?". **Decisão informada > marketing.**

### 7.5 Custo de rodar eval

Por suite completa:
- Frontier: ~$5-15 (sonnet) / ~$30-60 (opus)
- Mid: ~$1-3 (haiku/gpt-4o-mini)
- Local: $0 (mas tempo de máquina)

Total estimado por release: **$10-30** + tempo. Roda em CI noturno (regression + bench), smoke a cada PR.

### 7.6 Quem mantém

- Curadoria de fixtures: time mantém
- Revisão de regression visíveis: PR-by-PR
- Atualização de canonical set quando modelo melhor sai: cada quarter

Sem owner: matriz envelhece, "agnostic" vira marketing.

---

## 8. Anti-patterns

| Anti-pattern | Por que ruim |
|---|---|
| Pretender paridade ("works with any model") | LangChain disease; entrega lowest-common-denominator |
| Esconder quirks atrás de abstração leaky | Bug em provider minoritário fica invisível |
| Lowest-common-denominator features (sem cache, sem vision) | Perde valor de provider forte |
| Single-provider testing com claim multi-provider | Marketing > realidade |
| Adapter "experimental" em release prod sem flag | Confusão de qualidade |
| Cost matrix desatualizada | User toma decisão errada |
| Recommendations sem eval empírico | Vira gosto, não engenharia |
| Multi-model router automático sem evidência | Complexidade > ganho |
| Trocar default vendor a cada release | Quebra user trust |
| Adicionar provider sem completar §6 procedure | Bloat |

---

## 9. Versionamento

Provider matrix versiona junto com o spec do agente. Capability changes em provider real (ex: cache 5min → 1h) seguem release notes do provider; adapter atualiza ASAP, e doc reflete.

Major mudanças (provider sai do canonical set, novo entra) entram em changelog. Eval baseline re-publicado.

---

## 10. Insight final

Provider abstraction honesta declara **o que** é uniforme (interface, capabilities exposure) e **o que não** é (qualidade, custo, quirks operacionais).

A trap clássica:
- "Multi-provider" como **marketing** = "supports any LLM" + bug em todos exceto OpenAI
- "Multi-provider" como **arquitetura** = adapters isolados + capabilities honestas + eval em N modelos + quirks documentados

Esse projeto escolhe arquitetura. O custo é maior (eval multi-model não é grátis); o ganho é credibilidade técnica.

A regra é: **agnostic onde dá sem mentir; opinionado onde mentir custa caro.**
