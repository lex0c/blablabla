# CODE_GENERATION

Subsistema que rege como o modelo **escreve código** no `AGENTIC_CLI`. Cobre o pipeline canônico **generate → format → lint → test → checkpoint → accept**, integração com hooks, modos de strictness, anti-patterns.

Sem este doc, geração de código é só "modelo chama `edit_file`, modelo torce". Com ele, vira pipeline com gates verificáveis e fallback declarado por estágio.

Princípio raiz: *meça duas vezes, corte uma*. Geração de código é o ponto onde o agente mais frequentemente "corta sem medir" — escreve, parece bom, mergeia. Este doc é a especificação dos pontos de medição obrigatórios entre o `edit_file` e a aceitação humana.

---

## 0. Princípios (não-negociáveis)

1. **Toda escrita é checkpointed.** `write_file`/`edit_file` cria checkpoint pré-call (`CONTRACTS.md §2.6.2`). Sem exceção.
2. **Format é gate, não suggestion.** Código não-formatado **não atravessa** `accept` em modo `--strict`.
3. **Lint feedback vai pro modelo no mesmo turno.** Erros de lint após edit são input pro próximo step, não tarefa do humano descobrir.
4. **Test gating é opt-in mas declarado.** `--strict` corre testes relevantes; sem flag, testes são responsabilidade do user. Falta de teste **não bloqueia** edit por default.
5. **Falha de qualquer estágio não rola back automaticamente.** Modelo decide: tentar de novo, ajustar, ou pedir ajuda. Rollback explícito via `/undo` ou `/checkpoint restore`.
6. **Pipeline é transparente.** Cada estágio aparece em `tool_calls` audit; `/recap` mostra a sequência (edit → format → lint → test).
7. **Sem auto-commit.** Pipeline termina em `accept`, não em commit. Commit é decisão humana (memória deste projeto: `feedback_no_auto_commit`).
8. **Generation é per-language.** Cada linguagem tem seu formatter/linter/test runner declarado em config; harness não adivinha.
9. **Qualidade local primeiro.** Format/lint locais (rodam em ms) antes de test (roda em s). Falha cheap → não custa o caro.
10. **Sem geração de código secreta.** Modelo não pode "escrever em background" — toda mudança de FS passa pelo pipeline visível.

---

## 1. Pipeline canônico

### 1.1 Visão geral

```
modelo decide editar
   │
   ▼ tool_use: write_file | edit_file
   │
   ▼ harness cria checkpoint (CONTRACTS.md §2.6.2)
   │
   ▼ FS write
   │
   ▼ PostToolUse hook chain:
       │
       ├─ format    (rápido, ~50-300ms)
       │
       ├─ lint      (médio, ~100ms-2s)
       │
       └─ test      (lento, ~1s-30s)  [opt-in: --strict ou playbook --strict]
   │
   ▼ resultados agregados em tool_result
   │
   ▼ modelo recebe resultado: { edit_ok, format, lint, test }
   │
   ▼ modelo decide:
       ├─ accept (continua)
       ├─ retry (novo edit)
       └─ rollback (/checkpoint restore)
```

Nenhum estágio é "silencioso": tool_result inclui o resultado de cada gate, mesmo quando passa.

### 1.2 Estágio: format

**Quando:** após cada `write_file`/`edit_file` que toca arquivo cuja linguagem tem formatter configurado.

**Como:** `PostToolUse` hook canônico:

```toml
[generation.format.typescript]
command = ["prettier", "--write", "$FILE"]
timeout_ms = 5000

[generation.format.python]
command = ["black", "$FILE"]
timeout_ms = 5000

[generation.format.go]
command = ["gofmt", "-w", "$FILE"]
timeout_ms = 5000

[generation.format.rust]
command = ["rustfmt", "$FILE"]
timeout_ms = 5000
```

**Comportamento:**
- Formatter **modifica o arquivo in-place**. Diff entre pré-format e pós-format aparece em `tool_calls.payload`.
- Falha de formatter (sintaxe inválida, comando não encontrado): vira `format.failed` no tool_result; modelo recebe stderr; **não bloqueia** o pipeline (fica como warning).
- Custo registrado em audit (`tool_calls.duration_ms` inclui format time).

**Modo `--strict`:** falha de format **bloqueia** accept. Modelo precisa corrigir sintaxe antes de prosseguir.

### 1.3 Estágio: lint

**Quando:** após format. Roda em arquivo afetado + (em `--strict`) arquivos transitivamente afetados via `code_graph(direction='dependents')`.

**Como:**

```toml
[generation.lint.typescript]
command = ["eslint", "--max-warnings=0", "$FILES"]
timeout_ms = 30000
parse_output = "eslint-json"           # parser canônico

[generation.lint.python]
command = ["ruff", "check", "$FILES"]
timeout_ms = 15000
parse_output = "ruff-json"

[generation.lint.go]
command = ["go", "vet", "$FILES"]
timeout_ms = 15000

[generation.lint.rust]
command = ["cargo", "clippy", "--", "-D", "warnings"]
timeout_ms = 60000
```

**Output canônico** (após parse):

```json
{
  "diagnostics": [
    {
      "file": "src/auth.ts",
      "line": 42,
      "col": 10,
      "severity": "error",
      "rule": "no-unused-vars",
      "message": "'temp' is defined but never used"
    }
  ],
  "errors": 1,
  "warnings": 0
}
```

Vai para o modelo como **structured input no tool_result**, não como stderr cru. Modelo decide se corrige todos, alguns, ou nenhum.

**Comportamento por severity:**

| Severity | Modo normal | Modo `--strict` |
|---|---|---|
| `error` | warning visível | bloqueia accept |
| `warning` | aparece em recap, não bloqueia | aparece, não bloqueia |
| `info` / `hint` | suprimido por default | suprimido |

**Custo declarado:** lint do arquivo afetado: ~100-500ms. Lint transitivo (callers): ~500ms-2s.

### 1.4 Estágio: test

**Quando:** **opt-in**. Trigger:
- Flag `--strict` na sessão
- `playbook.frontmatter.strict_tests = true`
- Slash command `/test` explícito

**Como:** test runner declarado por linguagem; resolve testes relevantes via `test_mapping` do `CODE_INDEX.md §1.1`.

```toml
[generation.test.typescript]
command = ["npm", "test", "--", "$TEST_FILES"]
timeout_ms = 60000
parse_output = "jest-json"

[generation.test.python]
command = ["pytest", "$TEST_FILES", "-v", "--tb=short"]
timeout_ms = 120000

[generation.test.go]
command = ["go", "test", "$PACKAGES", "-count=1"]
timeout_ms = 120000

[generation.test.rust]
command = ["cargo", "test", "--", "--nocapture"]
timeout_ms = 180000
```

**Test resolution:**

```
edited file: src/auth.ts
  ↓ test_mapping query (CODE_INDEX §4.1)
relevant tests: [tests/auth.test.ts, tests/integration/login.test.ts]
  ↓
runner com $TEST_FILES expandido
```

Sem `test_mapping` (linguagem não suportada ou index unavailable): roda **suite inteira** com warning de custo, OR usa pattern do user (`[generation.test].fallback_command`).

**Output canônico:**

```json
{
  "passed": 12,
  "failed": 0,
  "skipped": 1,
  "failures": [],
  "duration_ms": 2340
}
```

**Comportamento:**

| Resultado | Modo normal | Modo `--strict` |
|---|---|---|
| Tudo verde | accept; warning silencioso | accept |
| Falha existente (já estava failing pré-edit) | accept com warning explícito | accept se delta=0 (não introduziu falha nova) |
| Falha **introduzida pelo edit** | warning em UI; modelo recebe failures | bloqueia accept |
| Test runner crash/timeout | warning; fallback para "tests indeterminate" | bloqueia accept |

### 1.5 Estágio: checkpoint + accept

**Checkpoint** já foi criado no início (§1.1). Pós-pipeline, harness:

1. Marca o checkpoint como `pipeline_completed: { format: ok, lint: ok|warn, test: ok|warn|skipped }` no metadata.
2. Persiste o tool_call com payload completo.
3. Modelo recebe tool_result e decide próxima ação.

`accept` é implícito: modelo continua o turno; nada bloqueou. Diferente de `reject` (modelo emite `tool_use` de `/checkpoint restore`).

---

## 2. Modos de strictness

### 2.1 Default (`normal`)

Pipeline roda **format + lint** automaticamente. Test é opt-in (slash command). Erros de lint não bloqueiam — vão como input pro modelo, que decide.

Use case: desenvolvimento iterativo, exploração, prototipação. Velocidade > rigor absoluto.

### 2.2 `--strict`

Pipeline roda **format + lint + test** automaticamente. Erros de qualquer estágio bloqueiam accept; modelo precisa corrigir antes de continuar.

Use case: PR final, refactor de código de produção, audit-related changes.

### 2.3 `--no-pipeline` (escape hatch)

Pipeline desabilitado. Apenas `write_file`/`edit_file` + checkpoint. Sem format, sem lint, sem test.

Use case: edição de configs, docs, fixtures de teste — onde formatter/linter não se aplicam ou são inconsistentes.

Audit registra `pipeline: skipped` no tool_call. Aceitável; sem aviso por default.

### 2.4 Per-playbook override

```yaml
# playbooks/refactor.md frontmatter
strict_tests: true
strict_lint: true
auto_format: true
```

Frontmatter sobrescreve flag de sessão. Razão: refactor sem teste é loteria — playbook força o gate independentemente de flag.

---

## 3. Configuração

### 3.1 Resolução de config

Ordem (precedência decrescente):

```
1. ~/.config/agent/generation.toml         # per-user global
2. .agent/generation.toml                  # per-project shared (committed)
3. .agent/generation.local.toml            # per-project local (gitignored)
4. defaults built-in                       # fallback para linguagens conhecidas
```

Mesmo schema TOML que MCP — coerência cross-subsystem.

### 3.2 Defaults por linguagem

```toml
# defaults built-in (§1.2-§1.4)
[generation.format.typescript]   command = ["prettier", "--write", "$FILE"]
[generation.format.python]       command = ["black", "$FILE"]
[generation.format.go]           command = ["gofmt", "-w", "$FILE"]
[generation.format.rust]         command = ["rustfmt", "$FILE"]
[generation.format.markdown]     command = ["prettier", "--write", "$FILE"]

[generation.lint.typescript]     command = ["eslint", "$FILES"]
[generation.lint.python]         command = ["ruff", "check", "$FILES"]
[generation.lint.go]             command = ["go", "vet", "$FILES"]
[generation.lint.rust]           command = ["cargo", "clippy"]

[generation.test.typescript]     command = ["npm", "test", "--", "$TEST_FILES"]
[generation.test.python]         command = ["pytest", "$TEST_FILES"]
[generation.test.go]             command = ["go", "test", "$PACKAGES"]
[generation.test.rust]           command = ["cargo", "test"]
```

Linguagem sem entry: estágio é `skipped`, não erro. `agent doctor` lista linguagens detectadas no projeto + cobertura de pipeline.

### 3.3 Disabled per-stage

```toml
[generation.format]
disabled = false                  # global; default false

[generation.lint]
disabled = false

[generation.test]
disabled = true                   # default true (opt-in)
```

Disable per-language:

```toml
[generation.lint.markdown]
disabled = true                   # markdown lint costuma ser ruidoso
```

### 3.4 Variables expandidas

| Var | Significado |
|---|---|
| `$FILE` | arquivo único modificado (format stage) |
| `$FILES` | espaço-separados, todos arquivos no batch (lint stage) |
| `$TEST_FILES` | testes resolvidos via test_mapping (test stage) |
| `$PACKAGES` | Go/Rust: packages contendo arquivos modificados |
| `$CWD` | project root |

Vars não-expandíveis viram literal — server bem-comportado falha com erro claro, não silencioso.

---

## 4. Audit footprint

### 4.1 `tool_calls` extension

Coluna nova em `tool_calls`:

```sql
ALTER TABLE tool_calls ADD COLUMN pipeline_result TEXT;  -- JSON
```

Conteúdo:

```json
{
  "format": { "status": "ok", "duration_ms": 145, "diff_lines": 3 },
  "lint": { "status": "warn", "duration_ms": 480, "errors": 0, "warnings": 2 },
  "test": { "status": "ok", "duration_ms": 2340, "passed": 12, "failed": 0 }
}
```

`status`: `ok` | `warn` | `failed` | `skipped` | `disabled` | `crashed` | `timeout`.

### 4.2 Spans OTEL

```
generation.pipeline       attrs: { tool, file_count, mode (normal|strict) }
generation.format         attrs: { language, files, duration_ms, status }
generation.lint           attrs: { language, errors, warnings, duration_ms }
generation.test           attrs: { language, passed, failed, duration_ms }
```

### 4.3 Recap integration

`/recap` (`RECAP.md`) inclui sumário de pipeline:

```
Edits this session:
  - src/auth.ts (3 edits) — format ✓, lint ✓ (1 warn), test ✓ (8/8)
  - src/queue.ts (1 edit) — format ✓, lint ✗ (2 errors), test skipped
```

### 4.4 Queries canônicas

```sql
-- "Quantos edits dessa sessão passaram em todos os gates?"
SELECT COUNT(*) FROM tool_calls
WHERE session_id = ?
  AND tool_name IN ('write_file', 'edit_file')
  AND json_extract(pipeline_result, '$.format.status') = 'ok'
  AND json_extract(pipeline_result, '$.lint.status') IN ('ok','warn')
  AND json_extract(pipeline_result, '$.test.status') IN ('ok','skipped','disabled');

-- "Tools mais usadas com falha de lint introduzida"
SELECT tool_name,
       SUM(json_extract(pipeline_result, '$.lint.errors')) AS lint_errors_introduced,
       COUNT(*) AS edits
FROM tool_calls
WHERE created_at > unixepoch('now', '-30 days')
  AND tool_name IN ('write_file', 'edit_file')
GROUP BY tool_name;

-- "Edits que pularam testes em modo strict (red flag)"
SELECT id, session_id, file_path
FROM tool_calls
WHERE json_extract(pipeline_result, '$.test.status') = 'skipped'
  AND session_id IN (SELECT id FROM sessions WHERE mode = 'strict');
```

---

## 5. Slash commands

```
/format                     # roda format em todos os arquivos modificados na sessão
/lint                       # roda lint
/test [pattern]             # roda test (default: arquivos da sessão; pattern: glob)
/strict on|off              # toggle de strictness para a sessão
/pipeline status            # último resultado por estágio
/pipeline reset             # limpa estado, próximo edit roda pipeline fresh
/checkpoint restore <id>    # rollback (cross-ref AGENTIC_CLI.md §2.5)
```

`--json` em todos.

---

## 6. Integração com playbooks

### 6.1 Playbooks que escrevem código

| Playbook | Strict por default? | Razão |
|---|---|---|
| `code-review` | n/a (read-only) | review não escreve |
| `refactor` | **sim** | refactor sem testes regrede silencioso |
| `debug` | não | iteração rápida; user roda `/test` quando convém |
| `audit` | n/a (read-only) | — |
| `explain` | n/a (read-only) | — |

Override no frontmatter (§2.4).

### 6.2 Test-driven refactor

Recipe canônico em `refactor` playbook (cross-ref `CONTEXT_TUNING.md §13.2`):

```
Passo 1: outline_file no target
Passo 2: find_references no symbol a refatorar
Passo 3: identify tests via test_mapping
Passo 4: rodar tests pré-edit (baseline) via /test
Passo 5: edit_file com pipeline strict
Passo 6: tests pós-edit devem ter delta=0 vs baseline
Passo 7: commit (manual, user-aprovado)
```

Sem passo 4, "delta=0 vs baseline" é mito — não dá pra saber se o teste já estava quebrado.

---

## 7. Anti-patterns

### 7.1 Skip pipeline silencioso

**O que é:** modelo chama `bash` com `cat > arquivo.ts <<EOF ... EOF` para evitar `edit_file` e seu pipeline.

**Por que é ruim:**
- Quebra checkpoint (`bash` com `writes:true` cria checkpoint, mas pipeline não roda).
- Sem format/lint, código entra non-conforming.
- Audit perde estrutura (`pipeline_result` fica NULL).

**Detecção:** modo dev — comparar diffs de `bash`-generated vs `edit_file`-generated; sinalizar quando padrão emergir.

**Substituição:** se `edit_file` é insuficiente para a operação (ex: criar arquivo grande), use `write_file` — pipeline ainda roda.

### 7.2 Aceitar diff sem testar (modo strict)

**O que é:** ignorar `test.failed` em modo `--strict` via `bash` que ignora exit code, ou via `--no-pipeline` ad hoc.

**Por que é ruim:**
- Modo strict é o gate. Bypassar = sessão deixa de ser strict de fato.
- Audit não tem como distinguir "test falhou e foi ignorado" de "test passou".

**Detecção:** `pipeline_result.test.status = 'failed'` em sessão com `mode = 'strict'` é alerta em `agent doctor`.

**Substituição:** se test não pode passar agora (em progresso, depende de outro PR), desligue strict explicitamente: `/strict off` registrado em audit.

### 7.3 Format/lint divergente entre dev e CI

**O que é:** pipeline local usa `prettier@2.x`, CI usa `prettier@3.x`. Edit passa local, falha em CI.

**Por que é ruim:**
- Modelo "aprendeu" que código está OK; PR é rejeitado por lint divergente.
- Custa ciclos de retrabalho.

**Substituição:** versionar formatter/linter em `package.json`/`pyproject.toml`/etc; pipeline usa exatamente o que o projeto declara, não o globalmente instalado. Default do `agent` é tentar `npx prettier`, `python -m black`, `cargo fmt` (resolução project-local).

### 7.4 Pipeline em arquivos não-editáveis

**O que é:** rodar `eslint` em `dist/`, `node_modules/`, ou arquivos generated.

**Por que é ruim:**
- Custo zero retorno: lint vai falhar (esperadamente) em código generated.
- Output ruidoso polui contexto do modelo.

**Substituição:** respeitar `.eslintignore`, `.prettierignore` etc. Em config:

```toml
[generation]
exclude = ["dist/**", "build/**", "**/*.generated.*"]
```

### 7.5 Long-running test em modo strict bloqueando UX

**O que é:** suite de testes E2E de 5min rodando em todo edit em `--strict`.

**Por que é ruim:**
- Cada `edit_file` espera 5min — sessão fica inviável.
- Modelo perde contexto entre invocações longas.

**Substituição:**
- `[generation.test].command` aponta para subset rápido (unit tests).
- E2E vai pra slash command separado (`/test e2e`) ou pre-commit hook.
- Frontmatter de playbook restritivo: `test_pattern: "tests/unit/**"`.

### 7.6 Auto-fix sem confirmação

**O que é:** usar `eslint --fix` ou `ruff check --fix` no pipeline auto.

**Por que é ruim:**
- Auto-fix pode mudar semântica (ex: ruff convertendo loop em comprehension introduz bug sutil).
- Modelo nem viu o que mudou; audit perde rastro.

**Substituição:** lint roda `--no-fix` por default. Auto-fix vira slash command explícito (`/lint --fix`) que registra diff resultante em audit.

### 7.7 Formatter como tool de modelo

**O que é:** registrar `format_file` como tool exposta ao modelo.

**Por que é ruim:**
- Format é gate, não decisão do modelo. Promover a tool dá ao modelo a opção de **não** formatar.
- Estoura tool budget (princípio 3, `CONTRACTS.md §2.5`).

**Substituição:** format roda em `PostToolUse` automaticamente. Usuário força via slash command (`/format`), nunca o modelo.

---

## 8. Failure modes (cross-ref)

| Code | Detection | Recovery |
|---|---|---|
| `generation.format.failed` | exit code != 0 | warning; pipeline prossegue (ou bloqueia em strict) |
| `generation.format.timeout` | timeout_ms exceeded | warning; pipeline prossegue |
| `generation.lint.errors` | erros parsed > 0 | input pro modelo (modo normal) ou bloqueio (strict) |
| `generation.lint.parse_failed` | linter output não parseou contra `parse_output` schema | warning; modelo recebe stderr cru com flag |
| `generation.test.failed` | testes failed > 0 | warning (normal) ou bloqueio (strict); modelo recebe failures |
| `generation.test.indeterminate` | runner crash/timeout | bloqueio em strict; warning em normal |
| `generation.test.no_mapping` | test_mapping sem matches | usa `fallback_command` se configurado, senão skip com warning |
| `generation.config.invalid` | TOML schema violation | sessão não inicia em strict; warning em normal |
| `generation.tool_not_found` | command não está no PATH | warning único por sessão; estágio skip |

Detalhamento operacional em `FAILURE_MODES.md` (TODO: §17 dedicado a generation failures, paralelo a §15 MCP e §16 code index).

---

## 9. Limites declarados (v1)

- **Sem refatorações automatizadas** (rename, extract method via tooling). Modelo gera diff manualmente; LSP integration é v2.
- **Sem code formatters em-runtime** (rustfmt API, prettier API). Sempre via subprocess — custo ~50-300ms por chamada.
- **Sem incremental compile/check.** Cada lint é cold; tools como `tsc --watch` mode são separadas (slash command).
- **Sem AI-assisted lint fix.** Auto-fix é determinístico (linter próprio); pedir ao modelo para "consertar warnings" é workflow normal, não pipeline.
- **Sem coverage gating.** "Tests pass" é binário; coverage delta é métrica de eval (`PERFORMANCE.md`), não gate.
- **Sem dependency check** (npm audit, cargo audit) no pipeline. Vai como slash command separado.

---

## 10. Eval e regressão

### 10.1 Eval do pipeline em si

Corpus em `evals/code_generation/`:

```yaml
- id: refactor-rename-symbol-001
  fixture: fixtures/sample-ts-app/
  prompt: "Renomeie validateOrder para checkOrder em todo o projeto"
  asserts:
    - kind: pipeline_status
      stage: format
      expected: ok
    - kind: pipeline_status
      stage: lint
      expected: ok
    - kind: pipeline_status
      stage: test
      expected: ok
    - kind: tool_called
      tool: edit_file
      min_calls: 3
    - kind: tool_called
      tool: find_references
      min_calls: 1                  # deve usar tool simbólica antes de editar
  weight: 1.0
```

### 10.2 Métricas chave

- **Pass rate por estágio** — em quantos % das sessões cada estágio passou.
- **Time to first failure** — quanto tempo até primeiro warning bloquear sessão em strict.
- **Tokens spent debugging post-edit** — proxy de "modelo gerou código pior que precisou consertar".

```sql
-- "Pass rate por estágio (últimos 30d)"
SELECT
  AVG(CASE WHEN json_extract(pipeline_result, '$.format.status') = 'ok' THEN 1.0 ELSE 0.0 END) AS format_pass,
  AVG(CASE WHEN json_extract(pipeline_result, '$.lint.status') IN ('ok','warn') THEN 1.0 ELSE 0.0 END) AS lint_pass,
  AVG(CASE WHEN json_extract(pipeline_result, '$.test.status') IN ('ok','skipped','disabled') THEN 1.0 ELSE 0.0 END) AS test_pass
FROM tool_calls
WHERE tool_name IN ('write_file', 'edit_file')
  AND created_at > unixepoch('now', '-30 days');
```

### 10.3 CI gate

PR que muda `defaults built-in` ou schema de pipeline_result triggera eval `code_generation/regression`. Falha = bloqueio de PR.

---

## 11. Insight final

Geração de código sem pipeline canônico é "torcer pelo modelo". Com pipeline, é **infraestrutura mensurável**: quem escreveu o quê, passou em quais gates, falhou em qual estágio, com que custo.

Diferença operacional:
- **Sem este doc:** modelo gera, user inspeciona, eventual CI pega o que escapou.
- **Com este doc:** modelo gera, harness valida em 3 estágios, modelo aprende com lint feedback no mesmo turno, user vê resultado consolidado.

A inversão recorrente do `AGENTIC_CLI` se aplica: harness faz o trabalho mecânico (format, lint, test orchestration); modelo faz o trabalho semântico (decidir o que mudar). Sem essa divisão, modelo gasta tokens em meta-trabalho de qualidade — mais caro e menos confiável que ferramenta dedicada.

Spec sem este doc: geração de código é arte. Com este doc: é **engenharia com gates declarados**.
