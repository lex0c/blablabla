# RECAP

Subsistema de **vista projetada** sobre sessões do `AGENTIC_CLI`. Renderiza o que aconteceu — para humano, PR, changelog, ou Slack — a partir do SQLite + traces, com renderização opcional via LLM.

Recap **não é compaction**. Compaction comprime contexto pra cabe na próxima chamada. Recap apresenta o que aconteceu pra humano.

Recap é instância direta da premissa raiz: *meça duas vezes, corte uma*. Aqui a medição é **retrospectiva** — o que de fato cortamos? Sem isso, o que aconteceu vira folclore. Com isso, vira artefato.

---

## 0. Princípios

1. **Source of truth é SQLite, não LLM.** Decisões e tool calls foram gravados; recap projeta. LLM só renderiza.
2. **Schema-bound em todos os formatos.** Cada renderer tem schema fixo. Sem campos opcionais "extras" inventados pelo modelo.
3. **Determinismo na projeção.** Mesmo input SQLite ⇒ mesmo intermediate JSON. Reprodutível sem LLM.
4. **LLM apenas no renderer.** Opt-out trivial (`--no-llm-render` retorna intermediate cru).
5. **Privacy by default.** Cross-session/cross-project recap **opt-in explícito**, nunca agregação automática.
6. **Honestidade epistêmica.** Sessão incompleta (crash, em-progresso) marca `incomplete: true`; renderer mostra explicitamente.
7. **Não substitui audit log.** `failure_events`, `approvals`, `hook_runs` continuam fonte forense. Recap é narrativa, não forense.
8. **Sem chatbot summarization.** Recap não responde "explica o que aconteceu" como prosa livre — sempre via schema.

---

## 1. Sub-modos (slash commands)

```
/recap                         # sessão atual, últimos 10 steps (default)
/recap last <N>                # sessão atual, últimos N steps
/recap session <id>            # sessão específica completa
/recap day [YYYY-MM-DD]        # cross-session no dia (default: hoje)
/recap range <from> <to>       # cross-session em range
/recap pr                      # current session formatted como PR description
/recap changelog               # current session formatted como entry user-facing
/recap slack                   # markdown Slack-compatible
/recap pre-compact             # mostra o que vai ser compactado ANTES
/recap json                    # intermediate cru (sem LLM)
```

Flags universais:
- `--out <path>` — escreve em arquivo (default: stdout)
- `--no-llm-render` — força modo determinístico (sem LLM)
- `--include-tool-output` — inclui output completo de tools (default: omite, só metadata)
- `--limit <N>` — corta a N steps mais relevantes
- `--lang pt|en` — idioma de renderização

---

## 2. Pipeline

```
┌─ SQLite (source of truth) ────────────────────────┐
│ sessions, messages, tool_calls, approvals,        │
│ checkpoints, failure_events, hooks, traces        │
└─────────────────┬─────────────────────────────────┘
                  │
                  ▼
┌─ Projection (TS puro, determinístico) ────────────┐
│ - Carrega tabelas relevantes                      │
│ - Agrupa por categoria (files_read, files_written,│
│   commands, decisions, costs, errors)             │
│ - Calcula deltas, custos agregados, durações      │
│ - Detecta incompleteness (status não-terminal)    │
│ - Output: RecapIntermediate (JSON)                │
└─────────────────┬─────────────────────────────────┘
                  │
        ┌─────────┴────────┐
        ▼                  ▼
┌─ no-LLM mode ─┐   ┌─ Renderer (LLM) ──┐
│ output JSON   │   │ Prompt versionado │
│ ou markdown   │   │ Schema fixo       │
│ template-     │   │ Validators        │
│ based         │   └─────────┬─────────┘
└───────────────┘             │
                              ▼
                    ┌─ Format outputs ─┐
                    │ human | pr |     │
                    │ changelog | slack│
                    │ json             │
                    └──────────────────┘
```

**Garantia:** se LLM falha (offline, rate limit, schema violation), modo determinístico responde com template-based markdown. **Recap nunca quebra**.

---

## 3. Schema intermediate (canonical)

```yaml
recap_intermediate:
  schema_version: "v1"
  generated_at: int                  # unix timestamp
  scope:
    kind: enum [session_current, session_specific, day, range, pre_compact]
    session_ids: [string]            # 1 para session_*; N para day/range
    range: { start: int, end: int }
  completeness:
    incomplete: bool                 # alguma sessão em estado não-terminal
    incomplete_sessions: [string]
    incomplete_reason: string
  goal:                              # objetivo original literal (não resumido)
    text: string
    source_step_id: string
  decisions:                         # decisões tomadas (extraídas determinísticamente de approvals + step outputs)
    - { step_id: string, what: string, why: string, decided_by: enum [user, policy, hook] }
  actions:
    files_read: [{ path: string, count: int }]
    files_written:                   # com diff summary
      - { path: string, lines_added: int, lines_removed: int, semantic_summary: string }
    commands_run: [{ command: string, exit_code: int, duration_ms: int }]
    web_fetches: [{ url: string, cached: bool }]
    subagents_spawned: [{ name: string, status: string, output_summary: string }]
  outcomes:
    tests_run: [{ command: string, passed: bool, duration_ms: int }]
    checkpoints: [{ id: string, step_id: string, files_affected: int }]
    artifacts: [{ kind: string, path_or_ref: string }]
  timeline:                          # eventos chave em ordem, pra narrativa
    - { ts: int, event: string, detail: string }
  costs:
    tokens: { in: int, out: int, cached: int }
    usd: float
    duration_ms: int
    model: string
    cache_hit_ratio: float
  errors:                            # falhas tratadas que valem mencionar
    - { code: string, recovered: bool, summary: string }
  not_done:                          # honestidade epistêmica (de subagent_outputs.not_done, schema de playbooks, etc)
    - { what: string, reason: string }
  unresolved_questions: [string]     # extraídas de assistant messages com explicit "?"
  memory_proposed:                   # memórias propostas durante sessão
    - { name: string, scope: string, accepted: bool }
```

Campos **sempre presentes** mesmo se vazios (array `[]`, string `""`). Ausência viola schema.

---

## 4. Renderers

Cada renderer consome `RecapIntermediate` e produz markdown formatado. Prompt versionado, schema esperado, eval acoplado.

### 4.1 `human` (default)

Markdown comum, lê-se em terminal. Estrutura:

```markdown
# Recap — {session_id} ({duration})

**Goal:** {goal.text}

## Resumo
{2-3 linhas, gerado por LLM, schema-bound}

## O que mudou
- 3 arquivos editados (+47 / -12 linhas)
- 5 comandos rodados
- 1 subagent (`refactor` em src/queue/)
- Checkpoints: 4

## Decisões
- step 7: extrair `computeBackoff` em arquivo próprio (motivo: testabilidade isolada)
- step 12: NÃO renomear `validateToken` (callers em 3 outros repos não rastreáveis)

## Não feito
- src/queue-consumer.ts: fora do escopo declarado, padrão similar precisa plan próprio

## Custo
$0.04 · 12.4k tokens · 78% cached · sonnet-4-6

(15 steps · 4m32s)
```

### 4.2 `pr` (PR description)

Otimizado para `gh pr create --body` ou colagem em GitHub:

```markdown
## Summary

- Extrai `computeBackoff` em função pura (`src/queue/backoff.ts`)
- Adiciona testes para 5 casos de retry
- Remove código morto (`computeBackoffOld`)

## Changes

### `src/queue.ts`
- Substitui inline backoff logic por import
- -11 lines (dead code removed)

### `src/queue/backoff.ts` (new)
- Pure function `computeBackoff(retry, baseMs)` 
- No external dependencies

### `tests/queue/backoff.test.ts` (new)
- 5 test cases covering retry edge cases
- All passing

## Test plan

- [x] Unit: `pnpm test src/queue/` → all green
- [ ] Integration: needs review against staging
- [ ] Manual: trigger retry path in dev

## Notes

- `computeBackoff` is now reusable; consider extracting to shared lib if used elsewhere
- Did not touch `src/queue-consumer.ts` (similar pattern but out of scope)
```

### 4.3 `changelog` (user-facing entry)

Curto, sem detalhe técnico:

```markdown
### Improved

- Retry backoff is now testable in isolation, fixing intermittent flakes in CI
- Removed dead retry code paths
```

### 4.4 `slack` (markdown Slack-compatible)

```
*Refactor: queue retry logic* (4m32s, $0.04)

✓ Extracted `computeBackoff` to pure function
✓ Added 5 unit tests (all passing)
✓ Removed dead code

Files: `src/queue.ts`, `src/queue/backoff.ts` (new), `tests/queue/backoff.test.ts` (new)

Decisions:
• Did NOT rename `validateToken` (3 untracked callers)
• Did NOT touch `queue-consumer.ts` (out of scope)
```

### 4.5 `json` (intermediate cru)

Output completo do schema intermediate. Consumível por scripts; sem LLM, sem rendering.

```bash
agent /recap json --out recap.json
jq '.actions.files_written' recap.json
```

### 4.6 `terse` (alternativa pra `human`)

Uma frase, máximo 200 chars:

```
Refactored queue retry logic — extracted `computeBackoff` to pure function with 5 new tests, removed dead code. 3 files, 4m32s, $0.04.
```

Útil pra footer / status line / commit body.

---

## 5. Source of truth (o que é carregado de onde)

| Campo do schema | Fonte |
|---|---|
| `goal.text` | primeiro `messages.role='user'` da sessão |
| `decisions` | `approvals` (with `decided_by`) + assistant messages com explicit decision markers |
| `actions.files_read` | `tool_calls` com `tool_name='read_file'`, agregado |
| `actions.files_written` | `tool_calls` com `writes:true` + diff via `checkpoints` |
| `actions.commands_run` | `tool_calls` com `tool_name='bash'` ou `bash_*` |
| `actions.subagents_spawned` | tabela `subagent_outputs` |
| `outcomes.tests_run` | heurística sobre `bash` commands matching test runners |
| `outcomes.checkpoints` | tabela `checkpoints` |
| `costs` | agregado de `messages.tokens_*` + custo por modelo |
| `errors` | `failure_events` (apenas user-visible, recovered) |
| `not_done` | extraído de subagent outputs + playbook reports + assistant messages com explicit "not done" markers |
| `memory_proposed` | tabela `memory_events` (action='proposed') |

**Tudo determinístico.** Projeção é função pura `(SQLite, scope) → RecapIntermediate`.

LLM **só** preenche:
- `summary` (campo de prosa em renderers human/pr/changelog/slack)
- `actions.files_written[].semantic_summary` (1 linha por arquivo)
- `decisions[].why` (quando não está literal nas approvals — caso raro)

Esses 3 campos têm **eval específico** garantindo que não inventam.

---

## 6. Cross-session & privacy

### 6.1 Boundaries

| Modo | Cruza projeto? | Cruza workspace? | Precisa flag? |
|---|---|---|---|
| `/recap` (current) | n/a | n/a | não |
| `/recap session <id>` | n/a (1 sessão) | n/a | não |
| `/recap day` | apenas dentro do mesmo `cwd`/projeto | **não** automático | `--all-projects` opt-in |
| `/recap range` | mesma regra | **não** automático | `--all-projects` opt-in |

### 6.2 Privacy guarantees

- Cross-project recap **nunca** roda sem flag explícita
- Recap **nunca** lê `~/.config/agent/memory/` (memory user-scope) — só `messages` e `tool_calls`
- Path absoluto com username é **anonimizado** em renderers (substitui `/home/lex/...` por `~/...`)
- Tool output completo é **omitido** por default (`--include-tool-output` opt-in)
- Conteúdo de `.env`, secrets detectados via heurística → **redacted** no output

### 6.3 Audit do próprio recap

Cada execução de `/recap` é gravada em tabela `recap_runs`:

```sql
recap_runs(
  id TEXT PRIMARY KEY,
  scope_kind TEXT NOT NULL,
  session_ids TEXT NOT NULL,        -- JSON array
  renderer TEXT NOT NULL,
  used_llm BOOLEAN NOT NULL,
  output_path TEXT,                  -- se --out usado
  created_at INTEGER NOT NULL
);
```

Útil pra: detectar uso anormal (script gerando recaps em loop), debugar regressões, prove compliance.

---

## 7. LLM renderer constraints

### 7.1 Prompt versionado

```
prompts/recap/
  human-v1.j2
  pr-v1.j2
  changelog-v1.j2
  slack-v1.j2
  terse-v1.j2
```

Cada um tem versão; mudança bumpa versão + entra em eval.

### 7.2 Schema enforcement

- Em providers com tool calling / structured outputs (Anthropic, OpenAI): use schema enforcement nativo
- Em providers sem (Ollama, llama.cpp): GBNF grammar
- Output validado contra schema **antes de mostrar** ao usuário; falha → fallback determinístico

### 7.3 Constraints negativas no prompt

```
NÃO invente decisões que não estão em decisions[].
NÃO crie arquivos em files_written que não estão no input.
NÃO adicione campos fora do schema.
NÃO use "we", "I", ou "the agent" — use voz passiva ou imperativa.
NÃO inclua ANSI escapes ou emoji excessivo.
NÃO especule motivações além do que está em why.
```

### 7.4 Eval do renderer

Fixtures: ~15 sessões reais (sintetizadas) com `RecapIntermediate` conhecido + golden `human`/`pr`/`changelog` outputs.

Métricas:
- **Fidelity** — campos do output batem com input (sem alucinar arquivo, decisão, custo)
- **Coverage** — campos importantes do input aparecem (não omitir decisões críticas)
- **Concision** — within size limits por renderer (terse ≤ 200 chars, etc)
- **Consistency** — mesmo input ⇒ output similar (não 5 renderings diferentes)

Threshold: fidelity 100% (zero alucinação tolerada), coverage ≥ 90%, concision 100%.

---

## 8. Performance & cost

### 8.1 Latência target

| Modo | Modo determinístico | Com LLM render |
|---|---|---|
| `/recap` (current, 10 steps) | < 100ms | < 1.5s |
| `/recap session <id>` (50 steps) | < 300ms | < 3s |
| `/recap day` (5 sessões) | < 800ms | < 5s |
| `/recap pre-compact` | < 200ms | n/a (sempre LLM-rendered) |

Modo determinístico é instantâneo (apenas SQL + template).

### 8.2 Custo

LLM render usa **Haiku** por default (barato, suficiente). Override via `--model`.

Custo típico:
- Recap session ~50 steps → ~3k tokens input → **$0.001 com Haiku**, $0.005 com Sonnet
- Recap day ~5 sessões → ~10k tokens → **$0.003 Haiku**, $0.015 Sonnet

`/recap json` é zero custo (sem LLM).

### 8.3 Tabela em SQLite vs cache RAM

Recaps recentes ficam em `recap_cache` (tabela com TTL 1h):

```sql
recap_cache(
  scope_hash TEXT PRIMARY KEY,       -- hash do scope_kind + session_ids
  renderer TEXT,
  output TEXT,
  generated_at INTEGER,
  expires_at INTEGER
);
```

Re-executar `/recap` em sessão ativa nos últimos 1h: hit do cache, ~10ms.

---

## 9. Headless mode

`agent --json /recap session <id>` retorna NDJSON estruturado:

```jsonl
{"type":"recap_start","scope":{...},"ts":1714138800}
{"type":"recap_intermediate","data":{...full schema...}}
{"type":"recap_render","renderer":"human","output":"# Recap...\n..."}
{"type":"recap_end","duration_ms":1234,"used_llm":true,"cost_usd":0.001}
```

Útil em CI/scripts: `pnpm postcommit` chama `agent --json /recap pr --out PR_DESCRIPTION.md`.

---

## 10. Anti-patterns

| Anti-pattern | Por que ruim |
|---|---|
| LLM "summarizer" sem schema | Vira chatbot que alucina decisões |
| Recap como tool (não slash) | Confunde — recap é vista, não ação que modelo decide |
| Cross-project sem opt-in | Vetor de leak entre workspaces |
| Recap que mostra tool output completo por default | Bloat, vaza secrets |
| Recap que roda em loop por hook | Custo runaway, sem valor |
| Renderer que escreve em FS sem `--out` | Surpresa; viola "stdout é puro" |
| Recap sem fallback determinístico | Quebra quando provider está down |
| LLM call em recap pre-compact | Pre-compact é critical path; não pode esperar Haiku |
| Recap em sessão `incomplete` sem flag visível | User toma decisão errada baseado em estado parcial |
| Memorizar texto de recap (auto-saving via memory) | Bloat de memória; recap não é fato persistente |

---

## 11. Como verificar (testes)

### 11.1 Unit

- Projection: input SQLite fixture → output schema (deterministic; snapshot test)
- Renderer (no-LLM): intermediate → markdown via template
- Renderer (LLM): mock provider, fixture intermediate, validate output schema

### 11.2 Integration

- Run agent em fixture de tarefa → execute `/recap` → validate intermediate against expected
- Crash mid-session → `/recap` em resume mostra `incomplete: true`

### 11.3 Eval

`evals/recap/` com 15 fixtures:
- 5 sessões puras (read-only)
- 5 sessões write (refactor, fix, etc)
- 3 sessões com erro recovered
- 2 sessões cross-day

Cada uma com golden output em `human`, `pr`, `changelog`. PR-bloqueante: fidelity < 100%.

---

## 12. Roadmap

**M4.1 — Recap básico**
- Projection function determinística
- Renderer `human` + `json`
- Slash commands `/recap`, `/recap session`, `/recap json`
- Eval smoke (5 fixtures)

**M4.2 — Renderers especializados**
- Renderer `pr`, `changelog`, `slack`, `terse`
- LLM render com Haiku + schema enforcement
- Cache em `recap_cache`
- Headless mode (`agent --json /recap`)

**M4.3 — Cross-session**
- `/recap day`, `/recap range`
- `--all-projects` flag com guards
- Eval regression (15 fixtures)
- `/recap pre-compact` integrado com Context Engine

**Total esforço: ~1.5 semanas** (já estimado).

---

## 13. Insight final

Sem recap, o que o agente fez vira folclore. Com recap, vira artefato — PR description, changelog entry, audit trail, status update.

Recap **não inventa nada**. Tudo que aparece veio do SQLite, projetado deterministicamente. LLM apenas dá voz ao dado, sob schema rígido. Quando LLM falha, fallback determinístico mantém o subsistema funcional — recap **nunca quebra**, no máximo fica menos bonito.

A regra é simples: **recap é a medição retrospectiva** que valida (ou desafia) o que o agente disse que fez.

Sem recap, "meça duas vezes, corte uma" cobre só o futuro. Com recap, cobre também o passado.
