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
├─ [project_context] ────────────── cache #3 (stable até CLAUDE.md mudar)
│  CLAUDE.md content
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
| project_context (CLAUDE.md) | 1000-3000 |
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

Em `[tool_schemas]`, tools listadas em ordem **canônica fixa** (não alfabética; semântica). Catálogo canônico v1 em `CONTRACTS.md §2.6` (12 tools):

1. `read_file` (mais usada)
2. `grep`, `glob`
3. `edit_file`, `write_file`
4. `bash`, `bash_background`
5. `fetch_url` (escopado; ver `CONTRACTS.md §2.6.5b` + `SECURITY_GUIDELINE.md §9.1`)
6. `task_sync`, `task_async`, `task_await`, `task_cancel`
7. `memory_search`
8. `wait_for`, `monitor` (background process coordination — `AGENTIC_CLI.md §7.3.1`)
9. **MCP tools** — sempre por último; ver §5.4 abaixo

Ordem afeta probabilidade de seleção (modelos atendem mais nas primeiras posições). `todo_write` **não está nesta lista** — é UI affordance via stream parsing, não tool de modelo (decisão B em `CONTRACTS.md §2.6.8`).

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
─
```

Marcadores `─` separam visualmente; modelo aprende padrão.

### 10.3 Format

**Literal**, não resumido. Goal vem do primeiro user message da sessão; preservado byte-by-byte.

```
Goal (original task): "refactor queue retry logic into pure function preserving semantics"
```

### 10.4 Multi-goal sessions

Sessão com sub-goals (refactor + tests + docs):

```
Goal (original task): "...refactor + add tests + update docs..."
Sub-goals (declared):
  - [done] extract pure function
  - [in progress] add 5 tests
  - [pending] update docs
```

Sub-goals trackeados via `todo_write` tool; injetados se ativos.

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

### 13.6 Per-recipe override

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
