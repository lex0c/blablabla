# CONTEXT_TUNING

Spec operacional sobre **shaping de contexto**: arquitetura do system prompt, layout, few-shot examples, format choices, attention positioning, per-workflow recipes.

`TOKEN_TUNING.md` cobre **HOW** o modelo gera (sampling, output budget). Este doc cobre **WHAT** vai no prompt. Pares conceitualmente complementares: counting + budgeting + tuning de geração + shaping de contexto.

Sem context shaping declarado: cada implementador inventa, eval inconsistente entre versões, attention positioning aleatório custa qualidade silenciosa.

---

## 0. Princípios (não-negociáveis)

1. **Shaping per-workflow.** Review precisa diff + tests; refactor precisa target + callers + tests; cada um difere.
2. **Cache-aware ordering.** Conteúdo estável (system, tool schemas, project ctx) primeiro pra maximizar cache hit.
3. **Format conscious.** JSON minify, prose vs structured — cada decisão custa tokens.
4. **Attention positioning intencional.** Conteúdo crítico em positions atendidas (top/bottom), não meio.
5. **Few-shot deliberado, não decorativo.** Quando incluir, quantos, qual formato — decisão por workflow.
6. **Goal re-injection literal.** Sempre presente, formato canônico, position consciente.
7. **Selective inclusion auditável.** Elidir tool result tem trail; sem "esquecimento silencioso".
8. **Decisão honra premissa raiz.** Cada shape é "meça antes de cortar tokens" — meça relevância, depois corta.

---

## 1. System prompt architecture

### 1.1 Sections canônicas

Layout fixo dentro de `[system]` (cache breakpoint #1):

```
[system]
─ Identity / role marker (1-2 lines)
─ Date/time injection (1 line, atualizado por sessão)
─ Project metadata (cwd, profile, git branch)
─ Output format expectations (formato global)
─ Constraints negativas globais
─ Workflow section (se em playbook)
─ Reference to capabilities (sem listar; tool schemas vêm em breakpoint #2)
[/system]
```

Cada section tem ordem fixa pra cache stability.

### 1.2 Identity / role

**Curto, factual, sem teatro:**

```
Você é o agente do AGENTIC_CLI. Atua sob policy declarativa. Não age sem
verificação. Cada decisão tem audit trail.
```

3-5 linhas. Sem "you are an expert..."; sem "your goal is to please...".

Princípio: **role-as-tool**, não persona.

### 1.3 Date/time injection

```
Current date: 2026-04-26 14:32 UTC
Session started: 2026-04-26 14:30 UTC (2min ago)
```

Permite modelo raciocinar sobre datas relativas em memory/decisions.

### 1.4 Project metadata

```
Working directory: ~/projects/my-app
Profile: orchestrated (local-first)
Git branch: feature/auth-refactor
```

Útil pra contextualização. Sempre disponível.

### 1.5 Output format expectations

Global, aplica a todo step:

```
Output formato: prosa concisa OR tool_use estruturado.
Sem prefácio. Sem disclaimers. Sem "I'll start by...". Aja.
```

Em playbook: substituído por output_schema específico.

### 1.6 Constraints negativas globais

Cross-workflow constraints:

```
NÃO invente arquivos/funções sem ler/grep.
NÃO assuma sucesso sem evidência (tool result, test pass).
NÃO mude semântica observável sem declarar.
```

3-5 itens. Mais que isso vira ruído.

### 1.7 Workflow section (per playbook)

Quando playbook ativo: section adicional injeta playbook prompt completo.

```
[playbook: code-review]
<conteúdo do playbook prompt>
[/playbook]
```

Posição: após constraints globais, antes do final do system.

### 1.8 System prompt de referência (canônico)

Os fragmentos de §1.2–§1.7 montados. Este é o **artefato canônico** que toda implementação deve produzir como baseline antes de tunar. Sem este exemplo, "implemente §1" produz N variações; com ele, divergências são propostas explícitas, não interpretações.

Dois perfis (`autonomous` e `orchestrated`) compartilham 80% do conteúdo; divergem onde a arquitetura diverge.

#### 1.8.1 Profile `autonomous` (frontier, modelo orquestra)

```
[system]
Você é o agente do AGENTIC_CLI. Atua sob policy declarativa. Não age sem
verificação. Cada decisão tem audit trail.

Current date: 2026-04-27
Session started: 2026-04-27 14:30 UTC

Working directory: /home/user/projects/my-app
Profile: autonomous
Git branch: main

Output formato: prosa concisa OR tool_use estruturado.
Sem prefácio. Sem disclaimers. Sem "I'll start by...". Aja.

NÃO invente arquivos/funções sem ler/grep.
NÃO assuma sucesso sem evidência (tool result, test pass).
NÃO mude semântica observável sem declarar.

Você pode usar tools listadas no próximo bloco. Quando usar:
- escolha a tool mínima que resolve o passo;
- entrada estruturada conforme schema;
- aguarde tool_result antes de prosseguir;
- erro de tool é input, não exceção — decida próxima ação.

Cancellation: se receber goal contraditório no próximo turno, descarte
trabalho em andamento e siga o novo goal sem comentário.
[/system]
```

Tamanho-alvo: 25-35 linhas, 500-700 tokens.

#### 1.8.2 Profile `orchestrated` (modelo local, harness orquestra)

Diferenças em **negrito**:

```
[system]
Você é o agente do AGENTIC_CLI no modo orquestrado. **Você executa um step
de cada vez dentro de um DAG controlado pelo harness.** Não age sem
verificação. Cada decisão tem audit trail.

Current date: 2026-04-27
Session started: 2026-04-27 14:30 UTC

Working directory: /home/user/projects/my-app
Profile: **orchestrated**
Git branch: main
**Step atual: <step_id> (<step_kind>)**
**Inputs deste step: <inputs_from refs>**
**Output esperado: <output_schema ref>**

Output formato: **APENAS tool_use OU output estruturado conforme schema.**
**Sem prosa fora do output.** Sem prefácio. Sem disclaimers. Aja.

NÃO invente arquivos/funções sem ler/grep.
NÃO assuma sucesso sem evidência (tool result, test pass).
NÃO mude semântica observável sem declarar.
**NÃO chame tools fora da palette deste step.**
**NÃO planeje próximos steps — o harness decide.**

Tools disponíveis neste step: <tool_palette restrita, 2-4 tools>.
Quando usar: escolha a tool mínima, entrada conforme schema, aguarde
result. Erro vira `{ status: "error", ... }` no output do step.
[/system]
```

Tamanho-alvo: 30-40 linhas, 600-800 tokens. Step-scoped variables (`<step_id>`, `<inputs_from>`, etc.) são preenchidas pelo harness por step — quebram cache breakpoint #1 propositalmente, mas o restante do prefix permanece estável.

#### 1.8.3 Convenções de placeholder

| Placeholder | Quem preenche | Quando |
|---|---|---|
| `<DATE_TOKEN>` | provider adapter (se suportar) ou aceita cache miss | sessão start |
| `<STEP_ID>` | harness | per step (orchestrated) |
| `<TOOL_PALETTE>` | step definition | per step (orchestrated) |
| `<PLAYBOOK_BLOCK>` | playbook loader | quando `task(playbook: …)` ativo |

Placeholders **nunca** ficam vazios em runtime — preenchimento ausente é erro de configuração, não default silencioso. `STATE_MACHINE.md` cobre falha de bind.

#### 1.8.4 Diff vs anti-patterns

Este prompt **deliberadamente não tem**:
- "You are an expert software engineer..." → ver `ANTI_PATTERNS.md §1.2`.
- "Be helpful, harmless, and honest" → role-as-tool, não persona.
- "Take a deep breath and think step by step" → cargo cult; não há eval que sustente em modelos pós-2024.
- Listagem de tools no próprio system prompt → tools vão em cache breakpoint #2 (`§2`), não em #1.

#### 1.8.5 Como evoluir este prompt

Mudança no canônico = breaking change em `prompt_versions` (`AUDIT.md`). Processo:

1. PR muda este §1.8.
2. Eval de regressão (`PERFORMANCE.md` / `TOKEN_TUNING.md`) roda contra corpus fixo.
3. Se regressão > threshold em qualquer playbook, PR é rejeitado.
4. Se aceito: `prompt_versions` ganha novo hash; sessões anteriores referenciam hash antigo.

Sem esse pipeline, drift entre versões fica invisível — exato problema que o leak da Anthropic em 2026 expôs publicamente.

---

## 2. Layout fixo (recap + extensões)

Repete `AGENTIC_CLI.md §6` com detalhamento:

```
┌─ [system] ────────────────────── cache #1 (stable)
│  identity + date + metadata + format + constraints + (playbook?)
├─ [tool_schemas] ───────────────── cache #2 (stable enquanto tools fixas)
│  JSON Schema de cada tool exposta
├─ [project_context] ────────────── cache #3 (stable até AGENTS.md mudar)
│  AGENTS.md content
├─ [memory_index] ───────────────── cache #4 (stable até memory_event)
│  index from MEMORY.md (~150 linhas)
├─ [repo_map] ───────────────────── stable até FS write
│  tree-sitter symbols (~2k tokens) — orchestrated; lazy em autonomous
├─ [compacted_history] ──────────── muda a cada compaction
│  summary de turns antigos
├─ [recent_turns] ───────────────── muda a cada step
│  últimos 3-5 turns na íntegra
└─ [current_turn] ───────────────── novo a cada step
   user prompt OR tool_result + qualquer goal re-injection
```

### 2.1 Tokens estimados por section (sessão típica)

| Section | Tokens |
|---|---|
| system | 500-1000 |
| tool_schemas | 2000-4000 |
| project_context (AGENTS.md) | 1000-3000 |
| memory_index | 1500-2500 |
| repo_map | 1500-3000 |
| compacted_history | 0-3000 |
| recent_turns | varia (até cap) |
| current_turn | varia |

Total estável (cache hit): ~6-13k tokens. Variável: ~5-30k tokens.

---

## 3. Cache breakpoints strategy detalhada

### 3.1 Breakpoints declarados

Anthropic permite até 4 breakpoints. Usar todos:

```
breakpoint #1: após [system]
breakpoint #2: após [tool_schemas]
breakpoint #3: após [project_context]
breakpoint #4: após [memory_index]
```

### 3.2 Por que ordem importa

Cache hit requer **prefix idêntico**. Se algo mudar antes do breakpoint, cache invalida tudo até esse ponto.

Anti-pattern: dynamic content (date) em [system] sem ser último elemento — quebra cache toda chamada.

**Mitigação:** date/time num placeholder fixo:
```
[system]
...
Current date: <DATE_TOKEN>     ← string fixa
```

E substituir pós-cache no provider (se suportado) OR aceitar cache miss (raro mas ok).

### 3.3 Cache TTL awareness

Anthropic: 5min default, 1h extended. Sessão > 5min entre turns: cache invalida.

`extended_cache: true` em config sessão **longa**:

```toml
[providers.anthropic]
extended_cache = true                  # TTL 1h; cobra mais por write mas amortiza em sessão > 30min
```

### 3.4 Anti-pattern

Reordering de sections quebra cache. **Layout é imutável durante implementação**; mudanças = breaking change.

---

## 4. Memory loading strategy

Cross-ref a `MEMORY.md §4`. Resumo do que vai pro contexto:

| Carregado | Quando | Onde | Tokens |
|---|---|---|---|
| Memory index | sempre eager | `[memory_index]` (cache #4) | ~150 linhas / 2k tokens |
| Memory body | lazy via `memory_read` | `[current_turn]` quando solicitado | varies |
| Auto-injected memories | trigger-based | `[memory_index]` extension | ~50-200 tokens cada |

### 4.1 Auto-injection rules

```toml
[[memory.auto_inject]]
trigger = "tool:bash:git commit*"
memories = ["feedback_*git*"]

[[memory.auto_inject]]
trigger = "file:*.env*"
memories = ["feedback_*secrets*"]
```

Em vez de modelo pedir, harness injeta automaticamente quando relevante. Reduz round-trips.

### 4.2 Ordering dentro do index

Memórias no índice ordenadas:

1. **`type: feedback`** primeiro (constraints comportamentais — alta atenção)
2. **`type: project`** (decisões em curso)
3. **`type: user`** (preferences)
4. **`type: reference`** (ponteiros, baixa atenção)

Justificativa: feedback memories são "constraints que devem ser obedecidas"; merecem atenção alta (top do index).

---

## 5. Tool palette shaping

### 5.1 Per-workflow

Cada playbook restringe tools (`PLAYBOOKS.md §1.1`):

```yaml
tools: [read_file, grep, glob, bash]      # whitelist
tool_restrictions:
  bash:
    - "git diff *"                         # subset
    - "rg *"
```

Modelo só vê o que pode usar. Confusão de tool selection cai dramaticamente.

### 5.2 Tool ordering em schemas

Em `[tool_schemas]`, tools listadas em ordem **canônica fixa** (não alfabética; semântica). Catálogo canônico v1 em `CONTRACTS.md §2.6` (21 tools):

1. **Filesystem read** — `read_file`, `grep`, `glob`
2. **Code retrieval simbólica** (gated por `CODE_INDEX.md`; gap when index unavailable) — `read_symbol`, `find_references`, `outline_file`, `code_graph`
3. **Filesystem write** — `edit_file`, `write_file`
4. **Execução** — `bash`
5. **Background coordination** — `bash_background`, `bash_output`, `bash_kill`, `wait_for`, `monitor`
6. **Network escopado** — `fetch_url` (ver `CONTRACTS.md §2.6.5b` + `SECURITY_GUIDELINE.md §9.1`)
7. **Subagent / orquestração** — `task_sync`, `task_async`, `task_await`, `task_cancel`
8. **Memory** — `memory_search`
9. **MCP tools** — sempre por último; ver §5.5 abaixo

Ordem afeta probabilidade de seleção (modelos atendem mais nas primeiras posições). Princípio: tools de **leitura barata** primeiro (read/symbolic), **edits** no meio, **side effects expansivos** (bash, network) depois, **orquestração e coordenação** ao final.

`todo_write` **não está nesta lista** — é UI affordance via stream parsing, não tool de modelo (decisão B em `CONTRACTS.md §2.6.8`).

### 5.3 Tool description cuidadoso

Descrições afetam seleção mais que docs:

```yaml
description: "Lê conteúdo de arquivo. Use sempre antes de editar pra entender o estado atual."
```

vs

```yaml
description: "Read file contents."
```

Primeira encoraja uso; segunda é neutra. Em playbooks especializados, descriptions adaptam.

### 5.4 Anti-pattern

Expor 30 tools em todo playbook = paralisia de decisão + alucinação de tool inexistente. **Restrição de palette é otimização, não limitação.**

### 5.5 MCP tools no cache breakpoint #2

Tools de MCP servers (`MCP.md`) entram no `[tool_schemas]` cache breakpoint #2, **depois** das canônicas e em ordem fixa por server name (alfabética). Razão: canônicas são ordenadas por uso esperado (§5.2 acima); MCP varia por instalação do user, então fixar por nome dá estabilidade de cache cross-session.

#### Eventos que invalidam o cache breakpoint #2

| Evento | Invalidação | Custo típico |
|---|---|---|
| Trust grant para server novo | sim — tools entram no schema | re-cache de ~2-5k tokens |
| Trust revoke ou denied | sim — tools saem do schema | re-cache |
| Manifest hash muda (server atualizou) | sim — schemas das tools podem ter mudado | re-cache |
| Server transita `trusted` → `active` | **não** — schema não muda em activation lazy | zero |
| Server transita `active` → `degraded` | **não** — schema continua | zero |
| Server transita para `disconnected` | **sim, condicional** — se tools previamente visíveis, harness re-renderiza schema sem o server | re-cache |
| Manifest **idêntico** entre sessões | **não** — match de hash skip prompt; schema mantido | zero |

Sessão típica não invalida cache MCP — eventos são raros (trust, manifest change). Sessão patológica (server que muda manifest a cada turn) detectada por taxa de invalidação > 1% em janela de 7d e trata como `ANTI_PATTERNS.md §7.x` (ver MCP anti-patterns).

#### Ordering canônica MCP

```
[tool_schemas]
  # canônicas v1 (ordem em §5.2)
  read_file, grep, glob, edit_file, write_file, bash, ..., memory_search

  # MCP — alfabética por server name, depois por tool name dentro do server
  mcp:github:create_issue
  mcp:github:list_issues
  mcp:postgres:explain_plan
  mcp:postgres:list_tables
  mcp:postgres:query
[/tool_schemas]
```

Ordering por nome do server (não por uso) dá determinismo: dois implementadores com mesmo `mcp.toml` produzem mesmo schema na mesma ordem → hash do prefix idêntico → cache hit cross-implementação possível.

#### Tokens estimados (sessão com MCP)

Adicionar à tabela §2.1:

| Section | Tokens (canônico) | Tokens (com 2-3 MCP servers, ~10 tools) |
|---|---|---|
| tool_schemas | 2000-4000 | 4000-7000 |

Custo aceito: tools MCP são opt-in via config; user que instala servers paga o overhead.

---

## 6. Few-shot examples strategy

### 6.1 Quando incluir

| Workflow | Few-shots? | Por quê |
|---|---|---|
| Modo normal | ⛔ não | Modelo frontier domina; few-shots desperdiçam tokens |
| Playbooks complex (audit, debug) | ✓ 1 exemplo | Format compliance |
| Adapter de tool calling em local | ✓ 2-3 exemplos | Modelo small **precisa** ver formato |
| Playbooks com schema rígido | ✓ 1 exemplo | Force schema compliance |
| Code generation tasks | ⛔ não | Modelo gera melhor sem viés |
| Compaction | ✓ 1 exemplo | Format consistency |

### 6.2 Estrutura canônica

```
Example:
<input>
User: ...
</input>
<output>
<tool>tool_name</tool>
<args>{"key": "value"}</args>
</output>

Counter-example (don't do):
<input>
User: ...
</input>
<output>
I'll call tool_name("value")        ← WRONG: function syntax
</output>
```

**1 positivo + 1 contra-exemplo** geralmente suficiente. Mais positivos = bias.

### 6.3 Position dentro do prompt

Few-shots vão no `[playbook: ...]` section (no system prompt). **Não** misturar com user prompt — modelo pode confundir.

### 6.4 Per provider

| Provider | Few-shot reception | Format preferred |
|---|---|---|
| Anthropic | excelente | XML-like tags |
| OpenAI | bom | Markdown ou XML |
| Llama-3 (Ollama) | precisa | XML tags consistentes |
| Qwen-coder | bom | chat-style turns |
| DeepSeek-Coder | bom | code blocks |

Adapter cuida de re-format por dialect (LOCAL_MODELS §8).

### 6.5 Eval-driven inclusion

Few-shot só **se eval mostra ganho**. Sem evidência: omite (custo de tokens não compensa).

Threshold: few-shot adicionado se ganho > 5% pass rate em eval.

---

## 7. Format choices

### 7.1 JSON minify

Em prompt content (não em input/output schemas):

```
Verbose:                          Minified:
{                                 {"key":"value","x":42,"y":[1,2,3]}
  "key": "value",
  "x": 42,
  "y": [1, 2, 3]
}
```

Economia: ~10-30% tokens. **Trade-off: humano lendo trace tem mais dificuldade.**

```toml
[context.format]
json_minify_in_prompt = true              # default em prod
json_minify_in_traces = false             # human-readable em traces
```

### 7.2 XML tags vs Markdown

Anthropic claramente prefere XML tags. OpenAI funciona com ambos. Llama-3 prefere XML consistente.

Convenção:

```
Estruturado (Anthropic-style):
<file path="src/foo.ts">
... content ...
</file>

Markdown (universal):
**File: src/foo.ts**
```typescript
... content ...
```
```

**Recommendation:**
- Tool input/output: structured (JSON Schema)
- Code blocks: markdown ` ``` ` (universal)
- File contents em prompt: `<file path="...">...</file>` (Anthropic-friendly)
- Diff: unified format em ` ```diff `

### 7.3 File path representation

**Sempre relativo a cwd:**

```
src/queue.ts                              ← canonical
./src/queue.ts                            ← redundante
/home/user/project/src/queue.ts           ← absoluto (PII!)
```

Path absoluto com username = **redaction violation** (SECURITY §6.2). Sempre converter pra `~/...` ou relative em prompts.

### 7.4 Tool args display format

Em assistant messages (não API):

```
✓ Conciso (preferred):
read_file(src/queue.ts:142)

✗ Verbose:
I'll call the read_file tool with path "src/queue.ts" and line 142.
```

Modelo emite tool_use estruturado; UI renderiza conciso.

### 7.5 Diff format

Em prompt (mostrando mudança ao modelo):

```diff
- old line
+ new line
  context line
```

3 lines context default. Em ≥120 cols UI: split format. Em < 80: unified.

### 7.6 Anti-patterns

- Mistura formats sem critério
- Markdown headers em prompts onde XML caberia melhor
- Path absoluto vazando username
- Verbose JSON em prompt (tokens desperdiçados)
- Pretty-printed code em código de exemplo (whitespace que não importa)

---

## 8. Attention positioning ("lost in the middle")

### 8.1 O fenômeno

Modelos têm bias conhecido (Liu et al. 2023):
- Início do prompt: **alta atenção**
- Fim do prompt: **alta atenção** (recente)
- **Meio do prompt: degradação** ("lost in the middle")

Effect mensurável: informação no meio é "lembrada" 30-50% menos que mesma informação no início ou fim.

### 8.2 Implicações pra layout

| Posição | Conteúdo | Por quê |
|---|---|---|
| Top (system) | identity + constraints + format | máxima atenção; estável (cached) |
| Top+1 (tool_schemas) | tools | alta atenção; modelo precisa ver |
| Middle (project_ctx, memory, repo_map) | reference info | degradação aceita; user/modelo busca quando precisa |
| Compacted history | summary | degradação tolerável; goal preservado literal |
| Bottom-1 (recent_turns) | últimos turns | alta atenção; recente |
| Bottom (current_turn) | user prompt + goal | máxima atenção; mais recente |

### 8.3 Goal re-injection

Goal **sempre** em positions atendidas:

```
[recent_turns]
[current_turn]
─ User: <atual prompt>
─ Goal (original): <literal>          ← bottom; máxima atenção
```

Frequência: **a cada step** durante execução longa (> 10 turns), não só após compaction.

### 8.4 Importante info em compacted history

Compaction summary deve preservar **decisões e goal**, não detalhes operacionais. Decisões posicionadas top do summary:

```
[compacted_history]
DECISIONS: extract function X; preserve semantics
GOAL: refactor queue retry preserving behavior
... resto do summary ...
```

### 8.5 Anti-patterns

- Tool schemas no meio (cached mas baixa atenção pra novo modelo)
- Goal apenas no início do prompt (esquecido em sessão longa)
- Critical decisions em meio do compacted summary

---

## 9. Per-step context shaping

### 9.1 O problema

Layout fixo é **bom pra cache** (consistente entre steps), mas **subótimo por step**:
- Step de exploração: precisa repo map completo
- Step de aplicação de edit: precisa repo map mínimo + arquivo focal
- Step de validação (test): precisa apenas test output

Hoje: layout idêntico (cache friendly). Trade-off não-declarado.

### 9.2 Solution: shaping conservativo

V1 — **mantém layout fixo.** Cache hit > tuning per-step na prática.

Otimizações conservadoras:
- Tool palette restrita por playbook (já cobre 80%)
- Memory auto-injection condicional (já existe)
- `repo_map` lazy em autonomous (eager em orchestrated)

V2 considerar: per-step layout override em DAG nodes (orchestrated profile naturalmente cabe).

### 9.3 Quando puxar per-step shaping

Sinal: eval mostra cache hit > 70% mas custo > target. Step-shaping pode otimizar.

Sem sinal: não faça (cache miss custa mais que tokens economizados).

---

## 10. Goal re-injection mechanics

### 10.1 Frequência

| Estado | Re-injection |
|---|---|
| Sessão < 5 turns | apenas no system (parte do playbook) |
| Sessão 5-15 turns | a cada compaction |
| Sessão > 15 turns | a cada 5 steps |
| Após critique abort/redo | sempre (modelo precisa re-orientar) |
| Após interrupt resume | sempre |

### 10.2 Position canônica

Em `[current_turn]`, **bottom** do prompt:

```
[current_turn]
User: <input atual>
─
Goal (original task): <literal goal>
Pinned context (always-include):       ← se houver pins ativos (§12.4)
  - [constraint] <pin.text>
  - [workflow]   <pin.text>
─
```

Marcadores `─` separam visualmente; modelo aprende padrão.

### 10.3 Format

**Literal**, não resumido. Goal vem do primeiro user message da sessão; preservado byte-by-byte.

```
Goal (original task): "refactor queue retry logic into pure function preserving semantics"
```

### 10.4 Multi-goal sessions

> **Cross-ref:** estrutura canônica em `STATE_MACHINE.md §2.3` (goal stack). `todo_write` cobre TODOs; goal stack cobre **objetivos aninhados** com push/pop auditados.

Sessão com aninhamento real (refactor → add tests → update docs) usa goal stack. `active_goal` substitui `goal.text` na injection:

```
Goal (active): "add 5 tests for retry edge cases"
Goal stack:
  - [done]    refactor queue retry logic
  > [active]  add 5 tests for retry edge cases       ← marcador `>` indica active
  - [pending] update docs

Sub-goals (todo_write):
  - [done] arrange test fixtures
  - [in progress] cover backoff edge case
  - [pending] cover concurrent retry case
```

Distinção operacional:
- **Goal stack** = objetivos com push/pop registrados em `goal_stack` table; foco do drift detector (`STATE_MACHINE.md §11`).
- **Sub-goals via `todo_write`** = TODOs informais dentro do active goal; sem audit trail rígido.

Sessão simples (um goal só) usa apenas o raiz; stack tem profundidade 1.

---

## 11. Repo map injection format

### 11.1 Format canônico

```
[repo_map]
src/auth.ts:
  export function login(email: string, password: string): Promise<Session>
  export class AuthError extends Error
  internal: validateCredentials, hashPassword

src/queue.ts:
  export class JobQueue
    method enqueue(job: Job): Promise<JobId>
    method dequeue(): Promise<Job | null>
  internal: drainTimer, retryBackoff

tests/queue.test.ts:
  describe: JobQueue retry logic
[/repo_map]
```

### 11.2 Granularity

| Nível | Inclui | Tokens estimado |
|---|---|---|
| Mínimo (default) | exports públicos + classe names | ~1k pra repo médio |
| Médio | + method signatures + internal markers | ~2k |
| Detalhado | + private methods + types | ~5k |

`agent --repo-map-level <level>` flag.

### 11.3 Atualização incremental

Hook `PostToolUse` em `write_file`/`edit_file`:
- Recompila symbols **só do arquivo afetado**
- Update incremental no map cached
- Custo: ~30ms

Full rebuild: `agent repo-map rebuild` manual ou em `SessionStart` (opt-in).

### 11.4 Filtros

```toml
[repo_map]
include = ["src/**/*.ts", "tests/**/*.ts"]
exclude = ["src/generated/**", "node_modules/**"]
max_tokens = 2000                     # truncate se exceder
max_file_path_depth = 5
```

### 11.5 Profile-aware

| Profile | Repo map |
|---|---|
| `autonomous` | lazy via `repo_map` tool quando solicitado |
| `orchestrated` | eager em `[repo_map]` section (modelo small **precisa** evitar grep storm) |
| `hybrid` | eager em planner; lazy em executor |

### 11.6 Estratégias de leitura sem dump de repo

> **Cross-refs:** subsistema queryable em [`CODE_INDEX.md`](./CODE_INDEX.md); tools simbólicas em `CONTRACTS.md §2.6.5c` e ADRs em §2.6.9.

Repo map (§11.1-11.5) é **uma** estratégia. Compilação canônica das 8 que o `AGENTIC_CLI` reconhece, com quando usar cada uma e custo relativo de tokens.

#### 11.6.1 Catálogo

| # | Estratégia | Custo (tokens) | Cobertura | Ferramenta |
|---|---|---|---|---|
| 1 | **Repo map** (symbol summary) | 1k-5k stable | overview do projeto | `repo_map` (`§11.1-11.5`) |
| 2 | **Symbol-targeted read** | 50-200 per symbol | corpo de função/classe/método | `read_symbol` |
| 3 | **Outline-then-zoom** | 200-500 per file → 50-200 per symbol | estrutura de arquivo + drill-down sob demanda | `outline_file` + `read_symbol` |
| 4 | **Diff-anchored reading** | Σ(arquivos no diff + tests + callers diretos) | mudanças de PR | `git diff` + `code_graph(direction='dependents', hops=1)` |
| 5 | **Import-graph traversal** (BFS) | 1-3 hops, ~50-500 tokens edges | dependências transitivas | `code_graph(direction='imports', hops=2-3)` |
| 6 | **Caller-callee map** | igual ao traversal mas focado em refs simbólicas | "quem chama Y?" | `find_references` |
| 7 | **Test-to-source mapping** | metadata derivado | "bug em test_X.py — qual source?" | `outline_file.test_files` + `tests_for/source_for` API |
| 8 | **Heuristic file ranking** | depende do algoritmo, ~0 (só ranking) | "dada query Q, quais arquivos olhar primeiro" | derivado de filename + path proximity + recency |

Vector search (similarity-based) **não está na lista**: rejeitado em `ANTI_PATTERNS.md §2.2`.

#### 11.6.2 Quando usar cada uma

Decisão é **por workflow**, não global:

| Workflow | Estratégias canônicas |
|---|---|
| **Code review** (PR) | (4) diff-anchored + (6) caller-callee em pontos sensíveis + (3) outline-then-zoom em arquivos novos |
| **Refactor** | (1) repo map + (6) caller-callee no target + (4) diff-anchored para verificar impacto + (5) import-graph para visualizar blast radius |
| **Debug** | (7) test-to-source partindo do teste falhando + (2) symbol-targeted no fluxo suspeito + (6) caller-callee para reverse-engineering |
| **Audit (security)** | (1) repo map + (8) heuristic ranking (filename matches `auth`, `crypto`, etc) + (6) caller-callee em entry points |
| **Explain** | (1) repo map + (3) outline-then-zoom no target |
| **First contact** (sessão exploratória) | (1) repo map + (8) heuristic ranking pela primeira pergunta do user |

Regra geral: **comece com a mais cheap** (repo map, outline) e drill-down apenas quando precisar. Se o modelo está pedindo `read_file` em arquivo > 500 LOC sem ter outlined, é sinal de strategy errada.

#### 11.6.3 Custo comparativo (mesma tarefa)

Tarefa: "qual o impacto de mudar a assinatura de `validateOrder` em `src/orders.ts`?"

| Estratégia ingênua | Custo aproximado |
|---|---|
| Dump do repo via `cat $(find . -name '*.ts')` | ~50k-500k tokens (excede contexto) |
| `glob 'src/**/*.ts'` + `read_file` em cada (8 arquivos) | ~12k tokens, 9 tool calls |

| Estratégia desta seção | Custo aproximado |
|---|---|
| `find_references('validateOrder')` + `read_symbol` em 3 callsites | ~600 tokens, 4 tool calls |
| `code_graph('src/orders.ts', 'dependents', 2)` + outline dos top 3 | ~800 tokens, 4 tool calls |

Diferença: **15-20× redução de tokens, 2-4× redução de tool calls**, mais precisão.

#### 11.6.4 Anti-patterns

- **Estratégia "leia tudo".** `read_file` em arquivos > 500 LOC sem `outline_file` antes. Modelo se perde, contexto trasborda. Eval pega via correlação tokens vs quality.
- **Grep antes de symbol query.** `grep "validateOrder"` para localizar definição quando `read_symbol("validateOrder")` ou `find_references("validateOrder")` cobrem com precisão semântica. Aceitável apenas como fallback (`index.unavailable`).
- **BFS sem hop cap.** `code_graph(hops=999)` retorna o repo todo em projetos coesos. Cap default 1, nunca acima de 3 (§G em `CONTRACTS.md §2.6.9`).
- **Diff-anchored sem callers.** Ler só o diff perde quem chama o que mudou. Workflow review **deve** incluir `code_graph(direction='dependents')` nos arquivos modificados.
- **Outline em todo arquivo do repo.** Outline é per-file pré-edit, não scan exaustivo — para isso é repo map (§1).
- **Heuristic ranking sem evidência.** Se o algoritmo de ranking não tem eval mensurando precision@5, é cargo cult. Sem corpus, ranker mais simples (path proximity + filename match) ganha.

#### 11.6.5 Quando o index não está disponível

Subsistema de `CODE_INDEX.md` pode estar `unavailable` (initial scan em progresso, disk full, schema migration). Estratégias degradam:

| # | Sem index | Custo extra |
|---|---|---|
| 1 (repo map) | re-scan tree-sitter ao vivo (mesma lógica, sem cache); **ou** fallback grep-based em arquivos pequenos | +10-30s no primeiro use |
| 2 (read_symbol) | `grep` definição + `read_file` com offset adivinhado | 5-20× mais tokens, mais frágil |
| 3 (outline) | `read_file` completo, modelo parseia mentalmente | 10-20× mais tokens |
| 4 (diff-anchored) | funciona — só o "callers" parte degrada |
| 5, 6 (graph/refs) | `grep` por nome de módulo/símbolo; risco de false positives |
| 7 (test mapping) | heurística filename: `foo_test.py` ↔ `foo.py` |
| 8 (heuristic ranking) | independente do index |

**Princípio operacional:** modelo recebe warning único por sessão quando index está unavailable; tools simbólicas retornam `index.unavailable` e modelo decide se cai no fallback ou aborta o step.

#### 11.6.6 Eval-driven selection

Workflow A/B testing (em `TOKEN_TUNING.md §13.4`):

```bash
agent eval --suite playbooks/refactor \
  --tune retrieval_strategy=symbol_first,read_file_first \
  --runs 5
```

Métrica primária: **tokens-out** por tarefa (proxy de custo). Métrica secundária: pass rate. Se symbol_first reduz tokens > 30% sem perder pass rate → ganha.

Sem eval-driven selection, escolha de estratégia vira gosto pessoal e regride silenciosa entre versões.

---

## 12. Selective context inclusion

### 12.1 Quando elidir

Heurísticas em compaction:

- **Tool result já consolidado em `decisions[]`** → elidir o tool result, manter decisão
- **Tool result de exploração** (read_file de arquivo não-tocado) → elidir após confirmação que não é referenced
- **Erros recuperados** (failure_event marcado `recovered: true`) → elidir, manter audit em traces
- **Memory bodies já lidas** → manter pointer apenas

### 12.2 Audit trail de elision

Toda elision **registrada**:

```sql
-- adicionar coluna elided_count em sessions
sessions.elided_tool_results_count INTEGER
```

`/recap` mostra: "Compaction elided 12 tool results; full traces available via /forensics".

### 12.3 Anti-patterns

- Elision sem audit (modelo "esquece" silenciosamente)
- Elision agressiva sem eval (perde decisões críticas)
- Elision em sessão crítica (audit/security workflows)

### 12.4 Pinned context primitive

> **Cross-refs:** consumido pelo fallback determinístico em `ORCHESTRATION.md §4.6`; re-injetado em auto-rehydrate em `STATE_MACHINE.md §7.6`; goal re-injection em §10.

Contraints invisíveis matam sessão longa. "Não mudar API pública", "sempre rodar `pnpm fmt` antes de commitar", "evitar import circular em `src/auth/`" — fatos que valem para a sessão inteira mas que ficam soterrados em `decisions[]` no checkpoint, não são re-injetados a cada compaction, e desaparecem em fallback estático.

**Pin** é uma marcação explícita: *este fato sobrevive compaction, é re-injetado junto com o goal, e aparece em auto-rehydrate.*

#### 12.4.1 API

```
/pin "<text>" [--scope session] [--kind <kind>] [--expires-in <duration>]
/pin --list
/pin --remove <id>
```

Flags:

| Flag | Default | Valores |
|---|---|---|
| `--scope` | `session` | apenas `session` em v1 (cross-session pins violariam contrato com memory) |
| `--kind` | `constraint` | `constraint` \| `workflow` \| `invariant` \| `reminder` |
| `--expires-in` | nenhum (vive até fim da sessão) | duration string: `30m`, `2h`, `1d`. Persistido em `expires_at` (epoch ts) |

Exemplos:

```
/pin "API pública de PaymentService não pode mudar"
/pin "rodar pnpm fmt antes de commitar" --kind workflow
/pin "fase de refactor — não tocar em testes ainda" --expires-in 45m
```

Tool equivalente para o modelo: `pin_context(text, kind?, expires_in?)` — modelo pode propor pin (vai a `awaiting_user` para confirmação modal, idêntico a memory writes).

#### 12.4.2 Schema

```sql
context_pins(
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  text TEXT NOT NULL,                  -- ≤ 500 chars
  kind TEXT NOT NULL,                  -- constraint | workflow | invariant | reminder
  created_at INTEGER NOT NULL,
  created_by TEXT NOT NULL,            -- user | model_proposed_user_approved
  expires_at INTEGER,                  -- NULL = sessão; ts = expira
  source_step_id TEXT                  -- step que originou (se model-proposed)
);
```

Cap: **10 pins por sessão**. Pin #11 → erro com sugestão de remover algum primeiro. Limite força priorização.

#### 12.4.3 Onde injeta

Pins são re-injetados em **três pontos**, sempre literais:

1. **Goal re-injection** (§10) — bloco `Pinned context:` logo após `Goal (original task):`, antes dos sub-goals.
2. **Pós-compaction** (LLM ou fallback estático em `ORCHESTRATION.md §4.6`) — primeira coisa após o summary.
3. **Auto-rehydrate no resume** (`STATE_MACHINE.md §7.6`) — bloco dedicado, sempre incluído.

Format:

```
Pinned context (always-include):
  - [constraint] API pública de PaymentService não pode mudar
  - [workflow] rodar pnpm fmt antes de commitar
  - [invariant] sem import circular em src/auth/
```

#### 12.4.4 Garantias

- **Nunca elididos.** §12.1 heurísticas de elision **excluem** pins — eviction policy os pula.
- **Nunca truncados.** Auto-rehydrate (`STATE_MACHINE.md §7.6`) trunca decisions/todos se exceder budget; pins ficam fora do truncamento. Se 10 pins juntos > budget, é erro de configuração — warning explícito ao user.
- **Auditados.** Cada pin entra em `recap_intermediate.pinned_context[]` (extensão a `RECAP.md §3`). `/recap` mostra pins ativos.
- **Persistidos cross-resume.** Pins sobrevivem crash/`/resume`; tabela `context_pins` é durable.

#### 12.4.5 Pins NÃO são memória

Distinção crítica vs `MEMORY.md`:

| Memory (user/feedback/project/reference) | Pin |
|---|---|
| Cross-session, persistente em `~/.config/agent/memory/` | Per-session, em SQLite da sessão |
| Carregada via index condicional | Sempre injetada literal |
| Pode ser body lazy-loaded | Sempre full text |
| Refere-se a fatos sobre projeto/usuário | Refere-se a constraints da tarefa atual |

Regra: se o fato vale para **outras sessões** desse repo, é memory. Se vale só para **esta** tarefa, é pin. Promoção pin → memory é manual: user copia o texto do pin (`/pin --list`) e salva via fluxo canônico de memory (`MEMORY.md`).

#### 12.4.6 Anti-patterns

- **Pin como TODO.** Pin é constraint *para evitar drift*, não uma tarefa. TODO → `todo_write`. Pin → "sempre fazer X" / "nunca fazer Y" / "lembrar de Z".
- **Pin de detalhe.** "Renomear `validateOrder` para `validateOrderInput`" não é pin — é decisão única, vai em `decisions[]`. Pin é coisa que precisa ser lembrada *múltiplas vezes na sessão*.
- **Pin sem expires_at em sessão multi-fase.** Sessão > 50 turns mudando de fase (refactor → test → docs): pins de fase anterior viram ruído. Use `--expires-in 30m` ou remova explicitamente ao mudar de fase.
- **Pinar tudo.** 10 pins = teto duro. Mais que 5 pins ativos é sinal de que a tarefa devia ser quebrada (`PLAYBOOKS.md`) ou que o pin certo é mais geral.

---

## 13. Per-workflow context recipes

Cada playbook tem **shape** distinto. Recipes canônicos:

### 13.1 `code-review`

```
Sections:
  - system + playbook code-review
  - tool_schemas (read-only subset)
  - project_context
  - memory_index (filtered: feedback type)
  - [diff section] git diff vs base branch
  - [tests section] test files relacionados
  - recent_turns

Goal re-injection: a cada 3 steps (review é multi-step)
Few-shot: 1 exemplo de output schema
```

### 13.2 `refactor`

```
Sections:
  - system + playbook refactor
  - tool_schemas (write-enabled)
  - project_context
  - memory_index
  - repo_map (eager; refactor precisa de mapa de callers)
  - [target_file]: arquivo focal
  - [callers section]: arquivos que importam o target (via grep)
  - [tests section]
  - recent_turns
  - goal re-injection (always)

Few-shot: 1 exemplo de plan + applied[] schema
```

### 13.3 `debug`

```
Sections:
  - system + playbook debug
  - tool_schemas (full + bash_background pra repro)
  - project_context
  - memory_index (filtered: debug refs)
  - [symptom section]: bug description literal
  - [repro section]: comandos pra reproduzir (se conhecidos)
  - [logs section]: bg process output relevante
  - recent_turns
  - goal re-injection (sempre — debug é exploratório)

Few-shot: 1 exemplo de hypothesis + verifies_with
```

### 13.4 `security-audit`

```
Sections:
  - system + playbook security-audit
  - tool_schemas (read-only, pesado em busca)
  - project_context
  - memory_index (filtered: security refs)
  - [threat_model section]: STRIDE preliminar
  - [diff section]: changes auditadas
  - [refs section]: SOFTWARE_SECURITY_GUIDELINE excerpts
  - recent_turns
  - goal re-injection

Few-shot: 1 exemplo de finding com exploit_chain
```

### 13.5 `explain`

```
Sections:
  - system + playbook explain (read-only)
  - tool_schemas (read-only)
  - project_context
  - memory_index
  - repo_map (eager — explain precisa visão estrutural)
  - [target section]: o que está sendo explicado
  - recent_turns

Few-shot: 1 exemplo de output estruturado (overview + flow + gotchas)
```

### 13.6 `threat-model`

```
Sections:
  - system + playbook threat-model (read-only, proativo)
  - tool_schemas (read-only + tools simbólicas — code_graph é load-bearing
    pra walk de entry points e trust boundaries)
  - project_context
  - memory_index (filtered: security + architecture + reference)
  - repo_map (eager — STRIDE walk precisa visão completa do design surface)
  - [diff section]: git diff vs base se mudança é sobre PR de feature/design
  - [security_refs]: SECURITY_GUIDELINE.md + THREAT_MODELING.md excerpts injetados
    quando trust boundary identificada (lazy via reference resolution)
  - recent_turns
  - goal re-injection (a cada 4 steps — STRIDE é sistemático e multi-categoria;
    fácil perder track de quais categorias já foram cobertas)

Few-shot: 1 exemplo de threat completo (id + category + target + attack +
          severity + mitigation com residual_risk declarado)

Notas:
- Sem [callers section]: trust boundaries vêm de design + code_graph entry points,
  não de quem chama o quê. Callers seria ruído.
- Memory filter inclui 'reference' pra capturar links pra docs de threat modeling
  externas (OWASP, MITRE, CWE) que o user já registrou.
- Goal canônico injetado: "STRIDE coverage; threats com residual_risk; assumptions
  declaradas" — a cada 4 steps.
```

### 13.7 `perf-investigate`

```
Sections:
  - system + playbook perf-investigate
  - tool_schemas (read tools + bash com profiler whitelist + bash_background +
    wait_for + monitor — coordenação async é load-bearing pra rodar profiler
    sem polling em loop)
  - project_context
  - memory_index (filtered: perf + reference)
  - repo_map (eager — hot path identification precisa estrutura)
  - [callers section]: callers do target hot path (via find_references) —
    contexto de "por que essa função é chamada tanto"
  - [perf_refs]: PROFILING.md + PREMATURE_OPTIMIZATION.md + PERFORMANCE.md
    excerpts (lazy via reference)
  - [profiler_output]: stdout/stderr de bash_background com profiler ativo,
    capturado por monitor; aparece em recent_turns como tool_result
  - recent_turns
  - goal re-injection (a cada 5 steps — espaçado; benchmarks consomem wall-clock,
    re-injection frequente seria ruído)

Few-shot: 1 exemplo com baseline + hot_path com share_pct + hypothesis status
          'confirmed' com delta antes/depois

Notas:
- Sem [diff section]: perf-investigate é sobre estado atual, não sobre mudança.
- include_callers: true é load-bearing — callers contam parte da história de
  por que função é hot (chamada N× por loop, etc).
- Goal canônico injetado: "baseline first; hot path com evidência; hipótese
  validada via medição, não leitura".
```

### 13.8 `git-hygiene`

```
Sections:
  - system + playbook git-hygiene (read-only sobre git state)
  - tool_schemas (read_file + bash com whitelist read-only de git)
  - project_context
  - memory_index (filtered: feedback + reference — capturar feedback_commit_style
    se existir)
  - [diff section]: git diff (working tree + staged) — load-bearing pra propor
    commit msg que reflete o que mudou
  - [git_log_sample]: git log --oneline -50 da branch — pra inferir convenção
    do projeto (Title Case verb / Conventional Commits / etc); injetado uma vez
    no primeiro turn, cacheado
  - recent_turns
  - goal re-injection (a cada 6 steps — sessão curta; raramente precisa)

Few-shot: 1 exemplo com branch_assessment + suggestions[] com commands literais
          + commit_drafts seguindo convenção detectada

Notas:
- Sem repo_map: git-hygiene é sobre history e workflow, não estrutura de código.
  Override: include_repo_map: lazy (sob demanda se user pedir explicação de
  porque commit X tocou arquivos Y).
- include_diff: true é load-bearing — sem diff, commit msg é alucinação.
- memory_filter prioriza 'feedback' pra capturar `feedback_commit_style` se existir
  (ver MEMORY.md exemplo: convenção Title Case do repo blablabla é memória deste
  tipo).
- Goal canônico injetado: "respeite convenção detectada; não execute; diff é
  fonte de verdade".
- Detecção de convenção é cached em `recap_cache` por sessão — não re-detectar
  a cada turn.
```

### 13.9 Per-recipe override

Em playbook frontmatter:

```yaml
context_recipe:
  include_repo_map: eager           # vs lazy
  include_diff: true                # auto-include git diff
  include_callers: true             # auto-grep callers do target
  goal_reinjection_every_n_steps: 3
  fewshot_count: 1
```

---

## 14. Eval-driven context tuning

### 14.1 Quando ajustar

Baseado em evidência:
- Eval regression → context shaping candidate
- Cache hit ratio < 50% → layout problema
- Custo elevado em workflow específico → palette/few-shot tuning
- Quality drop em sessão longa → goal re-injection frequência

Sem evidência: don't tune.

### 14.2 A/B testing

```bash
agent eval --suite playbooks/refactor \
  --tune fewshot_count=0,1,2 \
  --tune goal_reinjection_every=3,5,10 \
  --runs 5
```

Resultado:
```
fewshot=0, reinj=10: 71% pass, $0.18
fewshot=1, reinj=5:  88% pass, $0.20  ← winner
fewshot=2, reinj=5:  87% pass, $0.22
fewshot=1, reinj=3:  88% pass, $0.21
```

### 14.3 Versioning

`context_recipe_version` em playbook frontmatter:

```yaml
context_recipe_version: 2
```

Bump em mudança de recipe. Eval regression isolada por version.

---

## 15. Anti-patterns

| Anti-pattern | Por que ruim |
|---|---|
| Layout dynamic (não-cacheable) | Cache miss em todo turn; cost ×3-5 |
| Date/time no início do prompt | Quebra cache toda call |
| 30 tools sempre expostas | Paralisia de seleção; alucinação de tool |
| Few-shot decorativo (sem eval) | Tokens desperdiçados |
| Path absoluto com username | Redaction violation; PII em logs |
| JSON pretty-printed em prompt | 30%+ tokens desperdiçados |
| Goal apenas no system inicial | Esquecido em sessão > 15 turns |
| Critical info no meio do prompt | Lost-in-the-middle; ignorado |
| Compacted summary sem decisions preservadas | Modelo perde rationale |
| Repo map detalhado em autonomous | Tokens desnecessários (frontier não precisa) |
| Sem repo map em orchestrated | Grep storm; custo absurdo |
| Mistura format (XML + MD + JSON pretty) sem critério | Tokens + confusão |
| Selective elision sem audit | Modelo "esquece" sem rastreio |
| Per-step shaping sem evidência empírica | Cache miss > tokens economizados |

---

## 16. Insight final

Context shaping é **decisão arquitetural**, não detail de implementação. Cada section, ordering, format choice afeta:

- **Cache hit rate** (cost)
- **Attention** (quality)
- **Tokens** (cost + latency)
- **Reproducibility** (eval)

Spec madura **declara** cada decisão; implementador segue. Sem isso, cada implementador inventa, eval inconsistente, regressão silenciosa.

A regra é: **shape antes de tunar generation.** Context errado + sampling perfeito = output ruim. Context correto + sampling default = output usável. Context correto + sampling correto = ótimo.

E como tudo no projeto: **meça duas vezes, corte uma.** Context shaping é a medida de "o que vai pro corte do modelo". Sem shape declarado: cortando às cegas. Com shape: cada token tem propósito declarado.
