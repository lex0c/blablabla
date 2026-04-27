# AUDIT

Subsistema de auditoria do `AGENTIC_CLI`. Append-only, tamper-evident, redacted-by-default, queryable.

Este doc é a **autoridade consolidada** sobre audit: tables, timeline, PII redaction, immutability, forensics bundle, query CLI, headless behavior, retention, compliance.

`SECURITY_GUIDELINE.md §10` é overview; este doc é detalhamento operacional. `FAILURE_MODES.md` cataloga falhas; este doc cobre como elas são gravadas e consultadas.

Sem audit consolidado, "compliance" e "debug forense" viram intenção sem entrega. Audit é **função load-bearing** — trust, debug, compliance, billing, todos dependem.

---

## 0. Princípios (não-negociáveis)

1. **Append-only por convenção.** Audit tables não têm UPDATE em código (apenas INSERT); DELETE só via retention cleanup explícito.
2. **Tamper-evident, não tamper-proof.** User com root local pode modificar SQLite; spec declara o limite. Hash chain detecta tampering pós-fato.
3. **Redacted-by-default.** Patterns de secrets + paths com username são redacted **antes** de persistir, não depois.
4. **Timeline unificada disponível.** Cronologia da sessão acessível via uma view, não JOIN ad-hoc.
5. **Queryable via CLI.** `agent audit <subcomando>` cobre forensics comum sem SQL na unha.
6. **Schema versionado.** Mudança de schema preserva audit antigo via reader tolerante.
7. **Cada audit row tem source.** "Quem decidiu/disparou" sempre rastreável.
8. **Honesto sobre o que não defende.** Adversário com root local pode falsificar; documentar limites.

---

## 1. Tables canônicas

Lista canônica das **13 tabelas de audit**, com escopo, retention, sensitivity, e regra de redaction.

| Tabela | Escopo | Retention default | Sensitivity | Redaction |
|---|---|---|---|---|
| `sessions` | metadata de sessão | 90d | low | path → `~/...` |
| `messages` | conversation history | 90d | **high** (PII potential) | full redaction (secrets, paths) |
| `tool_calls` | invocações de tool | 90d | **high** (args + output) | full redaction |
| `approvals` | decisões permission | 365d | medium | nenhuma (decision-level apenas) |
| `hook_runs` | execução de hooks | 90d | medium | redact stdout |
| `failure_events` | falhas classificadas | 365d | medium | redact `details` |
| `memory_events` | memory ops | 365d | low (action+name only; sem body) | nenhuma |
| `recap_runs` | recap executions | 90d | low | nenhuma |
| `checkpoints` | FS snapshots metadata | 30d | low | path → `~/...` |
| `subagent_outputs` | subagent results | 90d | medium | full redaction em `payload` |
| `pending_decisions` | modal state | 7d (transient) | low | nenhuma |
| `recap_cache` | rendered recaps | 1h TTL | medium | nenhuma (output já passou por redaction) |
| `background_processes` | bg processes metadata | 30d | low | redact `cmd` |
| `traces` (NDJSON) | OTEL spans | 90d | medium | redact attrs |
| `redaction_events` | NEW — audit de redactions | 365d | low | nenhuma (sem o conteúdo redacted) |

### 1.1 Sensitivity levels

- **low**: metadata operacional; sem PII esperada
- **medium**: pode conter info sensível em payload; redaction parcial
- **high**: alta probabilidade de PII/secrets; redaction obrigatória completa

### 1.2 Retention configurável

```toml
# ~/.config/agent/config.toml
[audit.retention]
default_days = 90
sessions = 90
messages = 90
approvals = 365         # compliance
failure_events = 365
memory_events = 365
recap_cache = "1h"      # TTL especial
pending_decisions = 7
```

Cleanup via cron user-side (`agent gc`) ou hook `Stop` (configurable).

---

## 2. Timeline unificada

### 2.1 View canônica

```sql
-- View formal (SQLite VIEW ou função TS canônica)
CREATE VIEW audit_timeline AS
  SELECT 'message' AS kind, created_at AS ts, session_id, json_object('role', role, 'message_id', id, 'tokens_in', tokens_in, 'tokens_out', tokens_out) AS payload
  FROM messages
  UNION ALL
  SELECT 'tool_call', t.tc_started_at, m.session_id, json_object('tool_call_id', t.id, 'tool_name', t.tool_name, 'status', t.status, 'duration_ms', t.duration_ms)
  FROM tool_calls t JOIN messages m ON t.message_id = m.id
  UNION ALL
  SELECT 'approval', a.decided_at, m.session_id, json_object('approval_id', a.id, 'decision', a.decision, 'decided_by', a.decided_by, 'tool_call_id', a.tool_call_id)
  FROM approvals a JOIN tool_calls t ON a.tool_call_id = t.id JOIN messages m ON t.message_id = m.id
  UNION ALL
  SELECT 'hook_run', created_at, session_id, json_object('hook_run_id', id, 'event', event, 'hook_id', hook_id, 'exit_code', exit_code, 'blocked', blocked)
  FROM hook_runs
  UNION ALL
  SELECT 'failure', created_at, session_id, json_object('failure_id', id, 'code', code, 'classe', classe, 'recovery_action', recovery_action)
  FROM failure_events
  UNION ALL
  SELECT 'memory_event', created_at, session_id, json_object('memory_event_id', id, 'action', action, 'memory_name', memory_name, 'scope', scope, 'source', source)
  FROM memory_events
  UNION ALL
  SELECT 'checkpoint', created_at, session_id, json_object('checkpoint_id', id, 'step_id', step_id, 'kind', kind, 'files_changed', files_changed)
  FROM checkpoints
  ORDER BY ts;
```

### 2.2 API canônica

```ts
function auditTimeline(sessionId: string, opts?: {
  since?: Date,
  until?: Date,
  kinds?: AuditEventKind[],
  limit?: number
}): AuditEvent[];

interface AuditEvent {
  kind: 'message' | 'tool_call' | 'approval' | 'hook_run' | 'failure' | 'memory_event' | 'checkpoint';
  ts: number;
  sessionId: string;
  payload: object;  // structure depends on kind
}
```

### 2.3 Output canônico

Cronologia formatada:

```
2026-04-26T14:32:01.123Z [message]      user prompt: "refactor queue retry..."
2026-04-26T14:32:02.456Z [hook_run]     UserPromptSubmit (audit.sh) → exit 0 (12ms)
2026-04-26T14:32:02.470Z [message]      assistant: streaming...
2026-04-26T14:32:05.234Z [tool_call]    grep "computeBackoff" → 3 matches (45ms)
2026-04-26T14:32:05.300Z [approval]     bash → allow (policy)
2026-04-26T14:32:08.567Z [tool_call]    bash "npm test src/queue" → done (3.2s)
2026-04-26T14:32:08.789Z [hook_run]     PostToolUse (prettier) → exit 0 (0.2s)
2026-04-26T14:32:09.012Z [checkpoint]   chkpt_abc · 2 files · 4KB
2026-04-26T14:32:11.456Z [failure]      provider.timeout.streaming → retried_3x · user_visible
2026-04-26T14:32:14.789Z [memory_event] proposed: feedback/no-console-log (inferred) → refused (injection_pattern)
```

Acesso via CLI: `agent audit timeline <session_id>`.

---

## 3. PII handling

### 3.1 Redaction antes de persistir

Toda escrita em audit table passa por **redaction layer** antes do `INSERT`:

```ts
function redact(input: any, sensitivity: 'low' | 'medium' | 'high'): any {
  // sensitivity high: full redaction
  if (sensitivity === 'high') {
    // secret patterns (AWS, GitHub, Google, Slack, JWT, generic high-entropy)
    // paths absolutos com username (/home/<u>/... → ~/...)
    // env vars de secrets
  }
  // sensitivity medium: partial redaction
  if (sensitivity === 'medium') {
    // só secret patterns críticos (AWS, GitHub tokens)
  }
  return redacted;
}
```

### 3.2 Patterns canônicos (de SECURITY_GUIDELINE §6.1)

```
- AWS: AKIA[0-9A-Z]{16}, ASIA[0-9A-Z]{16}
- GitHub: gh[psour]_[A-Za-z0-9]{36,255}, github_pat_*
- Google: AIza[0-9A-Za-z_-]{35}
- Slack: xox[baprs]-*
- JWT: eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+
- Generic bearer: bearer\s+[A-Za-z0-9._-]{20,}
- High entropy strings (Shannon > 4.5) em contexto "token"/"key"/"secret"
- Path com username: /home/<user>/, /Users/<user>/, C:\Users\<user>\
```

Match → substitui por `<REDACTED:type>`. Path → `~/`.

### 3.3 `redaction_events` table

Cada redaction registra evento (sem o conteúdo):

```sql
redaction_events(
  id TEXT PRIMARY KEY,
  session_id TEXT,
  source_table TEXT NOT NULL,    -- 'messages' | 'tool_calls' | etc
  source_row_id TEXT NOT NULL,
  pattern_kind TEXT NOT NULL,    -- 'aws_key' | 'github_token' | 'username_path' | ...
  count INTEGER NOT NULL,        -- N matches no row
  created_at INTEGER NOT NULL
);
```

Permite saber **quanta** redaction houve sem expor **o quê**. Forensics pode incluir, audit query pode aggregar.

### 3.4 Output post-processing pipeline canônico

Order-of-operations entre raw output do modelo e dados persistidos. **Ordem importa**; cada etapa pode falhar ou modificar.

```
raw_output (from provider stream)
  ↓
Step 1: Parse
  - Extract text + tool_use blocks
  - Strict format check (JSON parse pra args)
  - Falha: vira ToolError estruturado (CONTRACTS §2)
  ↓
Step 2: Validate (se schema declarado)
  - JSON Schema check pra tool args
  - output_schema check pra step output (em playbooks)
  - Falha: retry com hint OR fail step
  ↓
Step 3: Sanitize
  - Strip ANSI control sequences (CSI escape malicioso)
  - Preserve SGR (cor) seguro
  - Normalize line endings
  - Truncate output > limit (100KB default; pointer)
  ↓
Step 4: Redact
  - Secret patterns (AWS, GitHub, JWT, etc) → <REDACTED:type>
  - Path com username → ~/...
  - Custom patterns (config TOML)
  - Registra em redaction_events (count + pattern_kind)
  ↓
Step 5: Persist
  - SQLite transaction commit (messages, tool_calls, etc)
  - traces NDJSON write
  - Atomic; failure = rollback
  ↓
Step 6: Emit
  - UI render (sanitizado + redacted)
  - Pass pra próximo step (contexto)
  - Hooks PostToolUse fire-and-forget
```

**Idempotência:** mesmo raw_output → mesma persisted output (deterministic). Verifiable em replay.

**Failure handling per etapa:**

| Etapa | Falha | Recovery |
|---|---|---|
| Parse | malformed JSON | Tool error estruturado; modelo decide |
| Validate | schema mismatch | Retry com hint; após N retries: fail step |
| Sanitize | (não falha; sempre retorna string limpa) | n/a |
| Redact | (não falha; aplica patterns) | n/a; eventos registrados |
| Persist | SQLite error | Rollback transação; sessão `error_fatal` |
| Emit | UI render error | Loggado; persist já ok; sessão continua |

**Order rationale:**
- Parse antes de validate (precisa estrutura)
- Validate antes de sanitize (errors em tool args devem ser visíveis ao modelo)
- Sanitize antes de redact (control chars podem mascarar secrets)
- Redact antes de persist (audit table NUNCA recebe raw secrets)
- Persist antes de emit (atomicidade > UX; se persist falha, UI também não recebe)

**Anti-pattern:** redact após persist (window de exposure já em backup).

### 3.5 Limites do redaction

**Honesto:**
- Heurística é trivialmente burlável (atacante com controle do prompt envia secret encoded em base64)
- Detecção de high-entropy é probabilística
- Custom secrets corporativos (não-padrão) não detectados

Mitigação: `[redaction.custom]` em config permite usuário declarar patterns adicionais:

```toml
[[audit.redaction.pattern]]
name = "internal_token"
regex = "^INTERNAL-[A-Z0-9]{32}$"
replacement = "<REDACTED:internal_token>"
```

---

## 4. Audit immutability

### 4.1 Append-only convention

Audit tables seguem regra:
- **INSERT** ✓ (apenas)
- **UPDATE** ✗ (proibido em código de aplicação)
- **DELETE** ✓ apenas via retention cleanup com motivo logged

Enforcement: lint custom em CI verifica que código não tem `UPDATE audit_table` ou `DELETE FROM audit_table` salvo em módulo `retention.ts` permitido.

### 4.2 Hash chain (tamper-evident)

Cada row em audit tables tem coluna `chain_hash`:

```sql
ALTER TABLE approvals ADD COLUMN chain_hash TEXT;
ALTER TABLE failure_events ADD COLUMN chain_hash TEXT;
-- (idem outras tabelas críticas)
```

Computed em INSERT:
```
chain_hash = SHA256(
  prev_chain_hash_for_session ||
  canonical_json(this_row_minus_chain_hash)
)
```

Primeira row da sessão: `prev = SHA256(session_id)`.

### 4.3 Verification

`agent audit verify <session_id>`:
- Walks chain row-by-row em ordem cronológica
- Recomputa cada `chain_hash`
- Reporta primeiro mismatch (= tampering point)
- Output: `{ verified: true } | { broken_at: { kind, row_id, expected, actual } }`

`agent forensics <session_id>` inclui verification result no manifest.

### 4.4 Limites declarados

**O que tamper-evident garante:**
- Modificação de row registrada após write é **detectável**
- Inserção de row fictícia no meio é **detectável**
- Deleção de row no meio é **detectável**

**O que NÃO garante:**
- Adversário com root reescrevendo a chain inteira (consistente)
- Tampering antes do hash ser computed (race condition na escrita)
- Tampering em ferramentas externas (export → modify → reimport)

**Resposta:** tamper-**evidence** não tamper-**proof**. Pra tamper-proof real: external append-only log (CloudWatch, sigstore rekor, blockchain). Out-of-scope pra v1; documentar.

---

## 5. Forensics bundle format

### 5.1 Schema canônico

`agent forensics <session_id>` produz `forensics_<session_id>_<unix_ts>.tar.gz`:

```
forensics_sess_8a3f2b_1714138800.tar.gz
├── manifest.json                   # metadata + integrity
├── session.json                    # row de sessions
├── messages.ndjson
├── tool_calls.ndjson
├── approvals.ndjson
├── hook_runs.ndjson
├── failure_events.ndjson
├── memory_events.ndjson
├── checkpoints.ndjson
├── subagent_outputs.ndjson
├── traces.ndjson                   # OTEL spans
├── redaction_events.ndjson
├── chain_verification.json         # resultado de hash chain check
└── signature.sig                   # cosign signature do manifest.json
```

### 5.2 manifest.json

```json
{
  "schema_version": "v1",
  "session_id": "sess_8a3f2b",
  "generated_at": 1714138800,
  "agent_version": "0.5.0",
  "files": [
    { "name": "session.json",      "sha256": "abc...", "rows": 1 },
    { "name": "messages.ndjson",   "sha256": "def...", "rows": 47 },
    { "name": "tool_calls.ndjson", "sha256": "ghi...", "rows": 23 },
    ...
  ],
  "redaction_summary": {
    "total_events": 12,
    "by_pattern": { "username_path": 8, "github_token": 1, "high_entropy": 3 }
  },
  "chain_verification": { "verified": true }
}
```

### 5.3 Verify

```bash
agent forensics verify <bundle.tar.gz>
```

1. Extract tarball
2. Verify cosign signature de `manifest.json`
3. Verify SHA256 de cada arquivo contra `manifest.files[].sha256`
4. Re-verify hash chain a partir dos NDJSON
5. Output:
   - `✓ verified` (tudo bate)
   - `✗ <reason>` (assinatura inválida, hash mismatch, chain broken at row N)

### 5.4 Cross-version compatibility

`schema_version` em manifest declara qual schema o bundle segue. Reader tolerante:

- v1 reader lê v1 bundles (hoje)
- v2 reader lê v1 e v2 (forward compat)
- v1 reader rejeita v2 com mensagem clara

### 5.5 Privacy

Bundle inclui **conteúdo redacted** das tabelas. NÃO inclui:
- Memory body (usa hash de memory_events apenas)
- Checkpoint blob completo (apenas metadata; user pode exportar separado se quiser)
- Tool output > 100KB (truncated com pointer)

User pode opt-in via flag pra incluir mais (`--include-checkpoints-blob`, etc).

---

## 6. `agent audit` query interface

### 6.1 Sub-commandos

```bash
agent audit timeline <session_id>          # cronologia unificada
agent audit timeline <session_id> --kinds tool_call,approval --since 5min
agent audit timeline --tail                 # streaming de novos eventos

agent audit failures --since 7d --code 'tool.*'
agent audit failures --severity critical    # via FAILURE_MODES classification
agent audit failures --recovery-action retried_3x

agent audit costs --by tool --since 30d
agent audit costs --by session --top 10
agent audit costs --by failure-class

agent audit deny-rate --since 7d            # % de tools negadas
agent audit deny-rate --by tool

agent audit hooks --hook 'audit.sh'
agent audit hooks --event PostToolUse --since 1d

agent audit memory --action promoted --since 30d
agent audit memory --scope shared

agent audit verify <session_id>             # hash chain check
agent audit gc --dry-run                    # mostra o que seria limpo (retention)
agent audit gc --apply                      # aplica retention cleanup
```

### 6.2 Output formats

- Default: tabela compacta colorida (Ink)
- `--json`: NDJSON pra scripting
- `--csv`: pra Excel/sheets
- `--markdown`: pra reports

### 6.3 Cross-session

```bash
agent audit timeline --range 2026-04-01..2026-04-26
agent audit costs --since 30d --all-projects   # opt-in cross-project
```

Cross-project requer flag explícita (privacy, mesma regra de §6.1 do RECAP.md).

---

## 7. Audit em headless / CI

### 7.1 Audit modes

```toml
[audit]
mode = "full" | "minimal" | "none"
```

| Mode | O que escreve | Quando usar |
|---|---|---|
| `full` (default) | tudo | interactive, dev local |
| `minimal` | só `failure_events` + `approvals` | CI heavy (1000+ sessões/dia) |
| `none` | nada | benchmarks, eval (NÃO recomendado em prod) |

### 7.2 Headless cleanup

```bash
agent --json --audit-export=/tmp/audit.tar.gz "..."
```

Comportamento:
1. Roda sessão normal (audit em SQLite local)
2. Ao fim, exporta `forensics` bundle pro path
3. Deleta SQLite local da sessão (mantém sessões pré-existentes)

Útil em CI/scripts: bundle vai pro artifact storage, local fica limpo.

### 7.3 Storage budget em CI

```toml
[audit.storage]
max_size_mb = 5000              # 5GB hard cap
on_full = "rotate" | "abort"    # default rotate (FIFO)
```

Quando hit cap:
- `rotate`: deleta sessões mais antigas até liberar espaço; warning loggado
- `abort`: nova sessão não inicia até cleanup manual

---

## 8. Schema versioning

### 8.1 `audit_schema_version` em cada row

Toda audit table ganha coluna:

```sql
audit_schema_version TEXT NOT NULL DEFAULT '1.0'
```

### 8.2 Reader tolerante

Reader (queries, forensics, timeline) lida com múltiplas versões:

```ts
function readApproval(row: any): Approval {
  switch (row.audit_schema_version) {
    case '1.0': return parseV1(row);
    case '2.0': return parseV2(row);
    default: throw new Error(`Unknown schema version: ${row.audit_schema_version}`);
  }
}
```

### 8.3 Migration tool

```bash
agent audit migrate --from 1.0 --to 2.0 --dry-run
agent audit migrate --from 1.0 --to 2.0 --apply
```

Migrations declarativas em `migrations/audit/v<from>_to_v<to>.ts`. Versionadas em git.

### 8.4 Backwards compatibility

Major bump de audit schema requer:
1. Migration path documentado
2. Reader v(N+1) lê v(N) por **N=2 releases** (period of grace)
3. Bundle forensics v(N+1) inclui downgrade hint pra readers v(N)

---

## 9. Trace sampling

### 9.1 Importance-based default

Default sampling rules:

```toml
[audit.traces.sampling]
default_rate = 1.0                # 100% em interactive

[[audit.traces.sampling.rule]]
when = { kind = "failure_event" }
rate = 1.0                        # sempre captura

[[audit.traces.sampling.rule]]
when = { kind = "approval", decision = "deny" }
rate = 1.0

[[audit.traces.sampling.rule]]
when = { kind = "tool_call", tool_name = "read_file" }
rate = 0.1                        # 10% pra reads frequentes

[[audit.traces.sampling.rule]]
when = { duration_ms_gt = 5000 }
rate = 1.0                        # captura ops lentas sempre
```

### 9.2 Storage budget

```toml
[audit.traces.storage]
max_size_gb = 5
rotation = "fifo" | "importance"
```

`importance`: deleta low-importance primeiro (sucessos rotineiros antes de failures).

### 9.3 Em produção heavy

CI rodando 10k sessões/dia:
- `default_rate = 0.1`
- Importance rules garantem 100% pra failures + denies + slow ops
- Storage budget 50GB com rotation `importance`

---

## 10. Performance & indexes

### 10.1 Indexes obrigatórios

```sql
CREATE INDEX idx_failure_events_code ON failure_events(code, created_at DESC);
CREATE INDEX idx_failure_events_session ON failure_events(session_id, created_at DESC);
CREATE INDEX idx_approvals_decision ON approvals(decision, decided_at DESC);
CREATE INDEX idx_tool_calls_session ON tool_calls(session_id, status);
CREATE INDEX idx_tool_calls_tool_name ON tool_calls(tool_name, status);
CREATE INDEX idx_memory_events_action ON memory_events(action, created_at DESC);
CREATE INDEX idx_memory_events_session ON memory_events(session_id, created_at DESC);
CREATE INDEX idx_hook_runs_event ON hook_runs(event, created_at DESC);
CREATE INDEX idx_hook_runs_session ON hook_runs(session_id, created_at DESC);
CREATE INDEX idx_messages_session ON messages(session_id, created_at DESC);
```

### 10.2 Performance budget

| Query | P99 esperado |
|---|---|
| `audit timeline <session>` (sessão típica ~50 events) | < 100ms |
| `audit failures --since 7d` | < 500ms |
| `audit costs --by tool --since 30d` | < 1s |
| `audit verify <session>` (hash chain walk) | < 200ms |
| `audit gc --dry-run` | < 2s |
| `forensics <session>` (bundle generation) | < 5s |

Violação P99 = regressão; CI cobre.

### 10.3 Vacuum & compact

`agent audit vacuum`:
- `VACUUM` em SQLite (libera espaço fragmentado pós-DELETE)
- Recomendado pós-retention-cleanup grande
- Custo: pode levar minutos em DB grande; documentar warning

---

## 11. Compliance hooks

### 11.1 GDPR right-to-be-forgotten

```bash
agent audit forget --user-id <id>            # apaga refs específicas
agent audit forget --since 2024-01-01        # apaga período
agent audit forget --session <id>            # apaga sessão completa
```

Implementação:
- DELETE em audit tables
- Logged em `failure_events` com `code: compliance.forget` (sem o conteúdo apagado)
- Hash chain após forget tem **hole** documentado (próxima row anota "previous purged")
- Bundle forensics gerado **antes** do forget se quiser preservar

### 11.2 Export pra subject access request

```bash
agent audit export-personal --user-id <id> --out personal.tar.gz
```

Bundle com tudo associado àquele user. Útil pra compliance enterprise.

### 11.3 Compliance presets

```toml
[audit.compliance]
preset = "gdpr" | "soc2" | "hipaa" | "none"
```

Aplica defaults apropriados (retention, redaction patterns extras, sampling, etc). v1 só `gdpr` e `none`; outros em v2.

### 11.4 Limites declarados

`AGENTIC_CLI` é local-first. Compliance é responsabilidade do **operador**:
- Multi-tenant cloud agent: out-of-scope
- Server-side aggregation: opt-in via OTEL export
- Audit central: user configura SIEM próprio

---

## 12. Anti-patterns

| Anti-pattern | Por que ruim |
|---|---|
| UPDATE em audit table fora de retention | Quebra append-only; viola convention |
| DELETE sem `retention_event` row | Audit gap silencioso |
| Redaction post-fato (em vez de pre-persist) | Window de exposure; já gravado em backup |
| Forensics bundle sem signature | Tampering trivial em transport |
| Hash chain quebrado silenciosamente | Tamper-evidence inutilizado |
| Trace 100% em CI heavy sem rotation | Disco cheio; sessões abortam |
| Schema mudando sem `audit_schema_version` bump | Reader explode; data loss |
| `agent audit` que esconde gaps de retention | User assume coverage que não existe |
| GDPR forget sem hole documentation | Hash chain vira inválida sem motivo claro |
| Cross-project query sem opt-in | Privacy violation |
| Audit export sem redaction visible | User exporta secrets sem saber |
| `audit.mode = none` em produção | Trust + compliance perdidos silenciosamente |

---

## 13. Insight final

Audit não é "log que ninguém lê". É **infraestrutura load-bearing** pra trust, debug, compliance, billing.

Spec atual tem peças (13 tabelas, 7 docs com pedaços). Este doc é a **costura**: timeline unificada, redaction antes-de-persistir, append-only com hash chain, forensics bundle formal, CLI de query, headless modes, schema versioning, compliance hooks.

A regra é: **toda decisão tomada em runtime que afeta outro humano, máquina, ou compliance precisa de audit row correspondente.** Se não tem, é bug — não feature.

E como tudo no projeto: **meça duas vezes, corte uma.** Audit é a medição **retrospectiva** que valida (ou desafia) cada corte feito. Sem audit consolidado, "meça duas vezes" vira fé. Com audit, vira fato verificável.
