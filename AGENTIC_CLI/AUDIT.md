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

Lista canônica das **17 tabelas de audit**, com escopo, retention, sensitivity, e regra de redaction.

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
| `prompt_versions` | versões de system prompt e playbooks | **forever** (no auto-cleanup) | low | nenhuma (conteúdo é o ponto) |
| `todos` | plano stream-parsed do modelo (ver §1.4) | 90d | low | nenhuma |
| `mcp_servers` | estado e config de MCP servers (ver §1.5) | mantido enquanto server em config | low | redact env values |
| `mcp_manifest_history` | versões de manifest MCP (ver §1.5) | **forever** (no auto-cleanup) | low | nenhuma (manifest é spec) |
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
prompt_versions = "forever"   # nunca limpar; rastreabilidade load-bearing
```

Cleanup via cron user-side (`agent gc`) ou hook `Stop` (configurable).

### 1.3 Prompt versioning (`prompt_versions`)

Princípio 8 ("permissões e hooks como dado") se estende a prompts. Sem versionamento de system prompt e playbooks, regressão de qualidade não é rastreável a um commit — exato problema que o leak da Anthropic em 2026 expôs publicamente.

#### 1.3.1 Schema

```sql
CREATE TABLE prompt_versions (
  hash TEXT PRIMARY KEY,                  -- SHA256(canonical(content))
  kind TEXT NOT NULL,                     -- 'system' | 'playbook' | 'workflow_section'
  name TEXT NOT NULL,                     -- 'system.autonomous' | 'playbook.code-review' | ...
  content TEXT NOT NULL,                  -- prompt literal
  parent_hash TEXT,                       -- hash da versão imediatamente anterior (mesmo `name`)
  author TEXT NOT NULL,                   -- git user que criou (best-effort)
  created_at INTEGER NOT NULL,            -- unix ts
  source_commit TEXT,                     -- git sha onde apareceu (nullable se ad-hoc)
  eval_run_id TEXT,                       -- referência a eval que validou (FK soft)
  notes TEXT                              -- changelog 1-line
);

CREATE INDEX idx_prompt_versions_name ON prompt_versions(name, created_at);
```

Conteúdo é o ponto da tabela; **sem redaction** (princípio: prompt é spec, não dado de usuário).

#### 1.3.2 Referência em `tool_calls` e `messages`

Adicionar coluna `prompt_hash` em ambas:

```sql
ALTER TABLE messages ADD COLUMN prompt_hash TEXT;       -- hash do system prompt ativo
ALTER TABLE tool_calls ADD COLUMN prompt_hash TEXT;     -- mesmo, redundante por joins rápidos
```

Toda invocação ao modelo registra o hash do system prompt usado. Replay sem hash = replay sem garantia de fidelidade.

#### 1.3.3 Fluxo de write

1. Sessão inicia; harness materializa system prompt final (CONTEXT_TUNING §1.8).
2. `hash = SHA256(canonical(content))`.
3. `INSERT OR IGNORE INTO prompt_versions (...)` — idempotente por hash.
4. `messages.prompt_hash = hash` para todo turn da sessão.
5. Se prompt mudar mid-session (raro: troca de playbook), novo hash, novo registro, mensagens subsequentes referenciam o novo.

#### 1.3.4 Append-only e immutability

`prompt_versions` segue regras de §4.1: INSERT-only, sem UPDATE, sem DELETE de retention (`forever`). Tampering = quebra de chain (§4.2).

#### 1.3.5 Queries canônicas

```sql
-- "Quais versões de system prompt rodaram nos últimos 30 dias?"
SELECT pv.name, pv.hash, pv.created_at, COUNT(DISTINCT m.session_id) AS sessions
FROM prompt_versions pv
JOIN messages m ON m.prompt_hash = pv.hash
WHERE m.created_at > unixepoch('now', '-30 days')
GROUP BY pv.hash
ORDER BY sessions DESC;

-- "Regressão começou em qual versão de prompt?"
-- (assume failure_events com classe='quality_regression' tagged manualmente)
SELECT pv.name, pv.hash, pv.notes, COUNT(*) AS failures
FROM failure_events f
JOIN messages m ON f.session_id = m.session_id
JOIN prompt_versions pv ON m.prompt_hash = pv.hash
WHERE f.classe = 'quality_regression'
GROUP BY pv.hash
ORDER BY failures DESC;

-- "Diff entre duas versões"
SELECT a.content, b.content
FROM prompt_versions a, prompt_versions b
WHERE a.hash = ? AND b.hash = ?;
```

#### 1.3.6 CLI

```
agent audit prompts list                  # versões registradas, ordenado por uso
agent audit prompts show <hash>           # conteúdo de uma versão
agent audit prompts diff <hash1> <hash2>  # diff entre versões
agent audit prompts trace <hash>          # sessões/turns que usaram esta versão
```

#### 1.3.7 Integração com eval

Eval de regressão (`PERFORMANCE.md` / `TOKEN_TUNING.md`) toma `prompt_hash` como input dimension. Resultado: matriz `(prompt_hash × workflow × metric)`. Mudança em `prompt_versions` sem eval correspondente = warning em CI.

#### 1.3.8 Limites

- **Não captura provider-side prompts.** Anthropic adiciona instructions internas fora do nosso controle (`CONTRACTS.md §4`). Documentado, fora de escopo.
- **Não captura mid-stream injection do harness.** Goal re-injection (`CONTEXT_TUNING.md §6`) é registrado em `messages` separadamente, não em `prompt_versions`.
- **Best-effort em author.** Se sessão roda em CI sem git user configurado, `author = 'ci'`.

### 1.4 Todos (`todos`)

Endereça audit gap declarado em `CONTRACTS.md §2.6.8` decisão B. `todo_write` foi **rejeitado** como tool de modelo (princípio "meta-cognição não é tool"); o plano emitido em prosa pelo modelo é capturado via stream parser e persistido aqui — auditabilidade igual à de uma tool, sem custo de slot no catálogo.

#### 1.4.1 Schema

```sql
CREATE TABLE todos (
  id INTEGER PRIMARY KEY,
  session_id TEXT NOT NULL,
  message_id TEXT NOT NULL,        -- mensagem do modelo onde o item foi extraído
  step_index INTEGER NOT NULL,     -- step da sessão (ordering)
  content TEXT NOT NULL,           -- texto canônico do item
  status TEXT NOT NULL CHECK (status IN ('pending','in_progress','completed','cancelled')),
  parent_todo_id INTEGER,          -- subtask (FK soft pra todos.id)
  created_at INTEGER NOT NULL,
  completed_at INTEGER,
  source_offset_start INTEGER,     -- offset no message body onde apareceu
  source_offset_end INTEGER,
  parse_confidence REAL NOT NULL,  -- 0.0-1.0 do parser
  prompt_hash TEXT,                -- system prompt ativo (FK soft pra prompt_versions)
  audit_schema_version INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_todos_session ON todos(session_id, step_index);
CREATE INDEX idx_todos_status ON todos(status) WHERE status IN ('pending','in_progress');
```

Sensitivity: **low** (texto curto, sem PII esperada — itens são "rodar testes", "ler src/auth.ts"). Retention: 90d (segue default).

#### 1.4.2 Pipeline canônico

```
modelo emite plano em prosa
    │
    ▼
stream parser (harness)
    │ regex/state machine procura formato canônico
    │ (definido em CONTEXT_TUNING.md §6.x — checklist markdown)
    ▼
extrai itens + status
    │
    ▼ INSERT/UPDATE em todos
SQLite todos
    │
    ▼ UI lê via SELECT
TodoList live em UI
```

Fonte de verdade é a tabela, **não o stream**. UI não parse de novo — lê de SQLite. Garante consistência entre o que audit registra e o que usuário vê.

#### 1.4.3 Formato canônico de checklist

Documentado em `CONTEXT_TUNING.md` (constraint do system prompt). Modelo deve emitir:

```
- [ ] item pendente
- [/] item em progresso
- [x] item completo
- [-] item cancelado
```

Outras formas (`* [ ]`, `1. [ ]`, emoji, prosa) são tolerantes mas geram `parse_confidence < 0.8` e warning em audit. Confidence < 0.5 → item **não persiste**, `failure_event` classe `parse.todo_low_confidence` é registrado.

#### 1.4.4 Append + update

Excepcional dentro de `AUDIT.md §4.1` (regra geral: append-only). `todos` permite UPDATE em `status` e `completed_at` apenas. Razão: estado do plano evolui durante sessão; replay precisa do estado final, não da progressão. Progressão fica em `audit_timeline` via events `todo_added`, `todo_status_changed`.

Restrição de UPDATE enforced via trigger:

```sql
CREATE TRIGGER todos_no_content_update BEFORE UPDATE ON todos
WHEN OLD.content != NEW.content OR OLD.session_id != NEW.session_id
BEGIN
  SELECT RAISE(ABORT, 'todos: content/session_id immutable');
END;
```

#### 1.4.5 Integração com `audit_timeline`

Adicionar UNION na view §2.1:

```sql
UNION ALL
SELECT 'todo_added' AS kind, created_at AS ts, session_id,
       json_object('todo_id', id, 'content', content, 'parent_todo_id', parent_todo_id)
FROM todos
UNION ALL
SELECT 'todo_status_changed', completed_at, session_id,
       json_object('todo_id', id, 'status', status)
FROM todos WHERE completed_at IS NOT NULL
```

#### 1.4.6 Queries canônicas

```sql
-- "Plano final da sessão"
SELECT step_index, content, status FROM todos
WHERE session_id = ? ORDER BY step_index;

-- "Sessões com plano abandonado (todos pending no fim)"
SELECT session_id, COUNT(*) AS pending
FROM todos WHERE status = 'pending'
GROUP BY session_id HAVING pending > 0;

-- "Failure rate do parser"
SELECT
  AVG(CASE WHEN parse_confidence < 0.8 THEN 1.0 ELSE 0.0 END) AS warn_rate,
  AVG(CASE WHEN parse_confidence < 0.5 THEN 1.0 ELSE 0.0 END) AS fail_rate
FROM todos
WHERE created_at > unixepoch('now', '-7 days');
```

A última query é o **gatilho de reconsideração** declarado em `CONTRACTS.md §2.6.8`: se `fail_rate > 5%` em janela móvel de 7d, abrir issue pra reconsiderar promoção a tool.

#### 1.4.7 Limites

- **Não captura "intent" do modelo, só o que ele declarou.** Plano implícito (sem checklist) fica invisível. Aceitável: mesma limitação de Claude Code com `todo_write` opcional.
- **Parser confidence é heurística.** False positives (UI list markdown que não é todo) são possíveis; mitigado por contexto (parser só ativa em respostas pós-prompt do tipo "plan", "list steps", ou explicit slash command).
- **Modelos locais podem ter alta `fail_rate`.** Se inviável em `LOCAL_MODELS.md`, profile `orchestrated` pode promover `todo_write` a tool **só nesse profile**. Decisão deferida pra v1.1.

### 1.5 MCP servers (`mcp_servers`, `mcp_manifest_history`)

Cobre o ponto cego de auditabilidade de MCP — antes deste schema, trust history vivia espalhada em `approvals` genérica, sem rastreio de manifest. Spec completa em [`MCP.md`](./MCP.md); state machine em [`STATE_MACHINE.md §6.5`](./STATE_MACHINE.md); contrato em [`CONTRACTS.md §11`](./CONTRACTS.md).

#### 1.5.1 Schema — `mcp_servers`

Estado **atual** de cada server em config (1 row per `<name>`).

```sql
CREATE TABLE mcp_servers (
  name TEXT PRIMARY KEY,                 -- ex: "postgres"
  transport TEXT NOT NULL CHECK (transport IN ('stdio','sse','http')),
  command TEXT,                          -- JSON array, stdio only; redacted env values
  url TEXT,                              -- SSE/HTTP only
  source TEXT NOT NULL,                  -- 'user' | 'project_shared' | 'project_local'
  state TEXT NOT NULL CHECK (state IN (
    'disconnected','handshaking','trust_pending',
    'trusted','active','degraded','denied','error'
  )),
  current_manifest_hash TEXT,            -- hash atualmente trusted (NULL se nunca aprovado)
  protocol_version TEXT,                 -- ex: "2024-11-05"
  server_version TEXT,                   -- declarado em initialize
  last_connected_at INTEGER,
  last_error TEXT,                       -- last failure reason, classed
  total_calls INTEGER NOT NULL DEFAULT 0,
  total_tokens_in INTEGER NOT NULL DEFAULT 0,
  audit_schema_version INTEGER NOT NULL DEFAULT 1
);
```

Sensitivity: **low**. Redaction: env values em `command` substituídos por `${VAR_NAME}` antes de persistir; URL em `url` é literal (esperado público). Retention: enquanto server estiver em config; remoção de config → row removida no próximo `agent gc` (excepcional pra audit table; razão: estado, não history).

UPDATE permitido em: `state`, `current_manifest_hash`, `last_connected_at`, `last_error`, contadores. **Imutável**: `name`, `transport`, `command`, `url`, `source`. Mudança de transport ou command = remove + insert.

#### 1.5.2 Schema — `mcp_manifest_history`

History de manifests. **Append-only, forever retention** (igual a `prompt_versions §1.3`).

```sql
CREATE TABLE mcp_manifest_history (
  id INTEGER PRIMARY KEY,
  server_name TEXT NOT NULL,
  hash TEXT NOT NULL,
  previous_hash TEXT,                    -- hash imediatamente anterior pra esse server
  manifest_json TEXT NOT NULL,           -- conteúdo canonical
  protocol_version TEXT NOT NULL,
  server_version TEXT,
  decision TEXT NOT NULL CHECK (decision IN ('granted','denied','revoked','superseded')),
  decided_by TEXT NOT NULL,              -- 'user' | 'auto-approve-flag' | 'ci'
  decided_at INTEGER NOT NULL,
  approval_id INTEGER,                   -- FK soft pra approvals
  audit_schema_version INTEGER NOT NULL DEFAULT 1
);

CREATE UNIQUE INDEX idx_mcp_manifest_unique
  ON mcp_manifest_history(server_name, hash);

CREATE INDEX idx_mcp_manifest_decided
  ON mcp_manifest_history(server_name, decided_at DESC);
```

Sensitivity: **low**. Redaction: nenhuma (manifest é spec, não dado de usuário; mesmo princípio de `prompt_versions`).

`decision = 'superseded'` é registrado quando manifest atual é trocado por um novo (não revogado, apenas substituído).

#### 1.5.3 Pipeline canônico

```
SessionStart
   │
   ▼
para cada server em config (mcp.toml):
   │
   ▼ initialize + tools/list
   ▼ hash = SHA256(canonical(manifest))
   │
   ▼ SELECT current_manifest_hash FROM mcp_servers WHERE name=?
   │
   ├─── match: state ← 'trusted' (skip prompt)
   │
   └─── mismatch ou NULL:
        ▼ INSERT INTO mcp_manifest_history (..., decision='granted'|'denied')
        ▼ UPDATE mcp_servers SET current_manifest_hash, state
```

Em `tools/call`:
- `tool_calls.mcp_server` populado com nome do server (NULL para canônicos)
- `mcp_servers.total_calls += 1`, `total_tokens_in += output_tokens`
- Em erro: `mcp_servers.last_error` updated; `failure_events` criado

#### 1.5.4 Append-only com exceção declarada

Conforme `§4.1` (regra geral), audit é INSERT-only. `mcp_servers` é a **segunda exceção** (primeira: `todos`). Razão: representa **estado**, não history; history vive em `mcp_manifest_history` que é append-only puro.

Trigger restritivo:

```sql
CREATE TRIGGER mcp_servers_immutable_keys BEFORE UPDATE ON mcp_servers
WHEN OLD.name != NEW.name
  OR OLD.transport != NEW.transport
  OR (OLD.command IS NOT NULL AND OLD.command != NEW.command)
  OR (OLD.url IS NOT NULL AND OLD.url != NEW.url)
  OR OLD.source != NEW.source
BEGIN
  SELECT RAISE(ABORT, 'mcp_servers: immutable keys changed; remove + insert');
END;
```

#### 1.5.5 Integração com `audit_timeline`

Adicionar UNION na view §2.1:

```sql
UNION ALL
SELECT 'mcp_trust_decision' AS kind, decided_at AS ts, NULL AS session_id,
       json_object('server', server_name, 'hash', hash, 'decision', decision,
                   'previous_hash', previous_hash, 'decided_by', decided_by)
FROM mcp_manifest_history
UNION ALL
SELECT 'mcp_state_change', last_connected_at, NULL,
       json_object('server', name, 'state', state, 'last_error', last_error)
FROM mcp_servers WHERE last_connected_at IS NOT NULL
```

Note: `session_id` NULL — eventos de MCP são cross-session.

#### 1.5.6 Queries canônicas

```sql
-- "Quais servers MCP rodaram nos últimos 30 dias?"
SELECT s.name, s.state, s.total_calls, s.last_connected_at
FROM mcp_servers s
WHERE s.last_connected_at > unixepoch('now', '-30 days')
ORDER BY s.total_calls DESC;

-- "Manifest history de um server específico"
SELECT decided_at, decision, hash, previous_hash, decided_by
FROM mcp_manifest_history
WHERE server_name = 'postgres'
ORDER BY decided_at DESC;

-- "Servers com falha recente"
SELECT name, state, last_error, last_connected_at
FROM mcp_servers
WHERE state IN ('error','degraded','disconnected')
  AND last_connected_at > unixepoch('now', '-7 days');

-- "Tools MCP mais chamadas (últimos 7d)"
SELECT json_extract(tc.tool_name, '$') AS tool,
       tc.mcp_server, COUNT(*) AS calls
FROM tool_calls tc
WHERE tc.mcp_server IS NOT NULL
  AND tc.tc_started_at > unixepoch('now', '-7 days')
GROUP BY tool, tc.mcp_server
ORDER BY calls DESC;

-- "Manifest changes que viraram denied (red flag)"
SELECT server_name, decided_at, hash, previous_hash
FROM mcp_manifest_history
WHERE decision = 'denied' AND previous_hash IS NOT NULL
ORDER BY decided_at DESC;
```

#### 1.5.7 CLI

```
agent audit mcp list                       # servers + estados + total calls
agent audit mcp show <name>                # config + state atual + history compacta
agent audit mcp history <name>             # full mcp_manifest_history
agent audit mcp diff <hash1> <hash2>       # diff entre versões de manifest
agent audit mcp calls <name>               # tool_calls filtrados por server
```

#### 1.5.8 Interação com `tool_calls`

Adicionar coluna em `tool_calls`:

```sql
ALTER TABLE tool_calls ADD COLUMN mcp_server TEXT;  -- NULL para canônicos
ALTER TABLE tool_calls ADD COLUMN mcp_manifest_hash TEXT;  -- hash ativo na chamada
```

Permite query: "quando esse server estava nesse manifest, quais tools foram chamadas?". Integra com `prompt_versions §1.3` para análise cross-dimensional (prompt_hash × mcp_manifest_hash × outcome).

#### 1.5.9 Limites

- **Server-side audit não captura.** O que o server faz internamente (queries SQL, FS reads) é problema do server, não do harness. Audit cobre interface, não implementação.
- **Hash de manifest, não de comportamento.** Server pode ter mesmo manifest mas comportamento diferente (mudou implementação interna). Trust é em manifest, não em binary; documented.
- **`source` do server pode mudar entre sessões.** Mesmo `<name>` aparecendo em fontes diferentes (user → project → local) não é detectado como event; rastrear via git/config diff.
- **Sem audit de `prompts/get` ou `resources/read`.** v1 só consome `tools/*` (ver `MCP.md §11`).

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
