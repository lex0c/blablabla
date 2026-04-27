# MCP

Spec operacional do **Model Context Protocol** dentro do `AGENTIC_CLI`. MCP é o **único caminho declarado** de extensão do tool catalog v1 (`CONTRACTS.md §2.6.7`); este doc cobre lifecycle, transport, manifest, namespacing, sandbox, budget e cache impact.

`CONTRACTS.md §11` é o contrato formal A/B. `STATE_MACHINE.md §6.5` é a máquina de estado. `SECURITY_GUIDELINE.md §3, §5` é o threat model. Este doc consolida tudo em vista única para implementador.

Sem este doc, MCP fica como folclore espalhado por 7 docs — cada implementador adivinha o resto. Princípio 8 ("permissões e hooks como dado, não como `if`") aplica também à integração MCP.

---

## 0. Princípios (não-negociáveis)

1. **Server é não-confiável até prova em contrário.** Princípio 11 do `AGENTIC_CLI.md`. Trust é per-manifest-hash, não per-name.
2. **Server declara tools, não policy.** MCP fornece capacidades; harness decide políticas (`SECURITY_GUIDELINE.md §5` invariant 9).
3. **Namespacing explícito.** `mcp:<server>:<tool>` em todo audit, output, e UI. Sem ambiguidade com canônicos.
4. **Failure visível.** Server caído ≠ tool ausente silenciosa; UI sinaliza, audit registra.
5. **Hash do manifest é load-bearing.** Mudança de manifest = re-trust mandatório.
6. **Lazy activation.** Server só conecta quando modelo efetivamente vai chamar uma tool; conexões pré-emptivas são waste.
7. **Sem MCP no critical path.** Tools canônicas (read_file, bash, etc.) **não** podem ser substituídas por MCP. Razão: replay e portabilidade.
8. **Configuração é dado, não código.** Server config em `~/.config/agent/mcp.toml` — versionável, diff-able, scriptable.

---

## 1. Lifecycle

### 1.1 Visão geral

```
discover → handshake → trust_prompt → register → activate → call → degrade? → disconnect
   │           │             │            │          │        │        │           │
config     initialize    user input  tools/list  first    tools/  schema     transport
file       /protocol     + hash      registered  use      call     err        broken
```

Estados formais em `STATE_MACHINE.md §6.5`. Esta seção é narrativa.

### 1.2 Discovery

MCP servers são declarados em **3 fontes**, em ordem de precedência:

```
1. ~/.config/agent/mcp.toml          (per-user global)
2. .agent/mcp.toml                   (per-project shared, committed)
3. .agent/mcp.local.toml             (per-project local, gitignored)
```

Mesmo `<name>` em fontes diferentes: precedência local > shared > global. Conflito é warning, não erro.

Format de `mcp.toml`:

```toml
[servers.postgres]
transport = "stdio"
command = ["mcp-server-postgres", "--dsn", "$DATABASE_URL"]
env = { LOG_LEVEL = "info" }                  # passa pro server
cwd = "."                                      # default cwd da sessão
timeout_ms = 30000
parallel_safe = false                          # default false
disabled = false

[servers.github]
transport = "sse"
url = "https://mcp.github.com/v1"
auth = { kind = "bearer", env = "GITHUB_MCP_TOKEN" }   # token em env, não em config
```

Variáveis `$VAR` resolvem do env do agent (não do user shell genérico — env de sessão definida em `STATE_MACHINE.md §2`).

### 1.3 Handshake

Conexão lazy: harness não conecta até modelo efetivamente chamar `mcp:<name>:*`. Em `SessionStart`, harness apenas:

1. Lê `mcp.toml`
2. Para cada server **ativo** (não `disabled`): consulta `mcp_servers` em SQLite pra estado anterior
3. Se `trusted` e `manifest_hash` recente (< 7 dias): tools **viram visíveis** ao modelo no `tool_schemas` (cache breakpoint #2)
4. Senão: tools **invisíveis** até o user aprovar
5. **Não conecta** ainda

Conexão ocorre no primeiro `tools/call` da sessão pra um server. Custo amortizado: handshake (~50-200ms stdio, ~100-500ms HTTP) acontece uma vez por sessão por server.

### 1.4 Initialize

Conforme MCP spec:

```jsonc
// → server
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "roots": { "listChanged": false },
      "sampling": {}
    },
    "clientInfo": {
      "name": "agentic-cli",
      "version": "0.x.y"
    }
  }
}
```

```jsonc
// ← server
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": { "tools": {}, "resources": {} },
    "serverInfo": { "name": "postgres", "version": "1.2.3" }
  }
}
```

Mismatch em `protocolVersion` exato → `mcp.initialize_protocol_mismatch` → `error` state. Harness suporta uma versão por release.

### 1.5 Trust prompt

Após `initialize`, harness chama `tools/list`, computa hash, e compara com `mcp_manifest_history`.

| Caso | Comportamento |
|---|---|
| Hash desconhecido (novo server) | UI modal: "Server `<name>` declara N tools. Aprovar?" + listagem das tools (nome + descrição) + hash |
| Hash conhecido e aprovado | Skip prompt, vira `trusted` |
| Hash mudou | Modal: "Server `<name>` mudou. Diff abaixo:" + tools adicionadas/removidas/modificadas |
| Hash conhecido mas previamente recusado | Modal pergunta de novo (decisão pode mudar) |

Decisão registrada em `approvals` com `kind = "mcp_trust"`, plus `mcp_manifest_history` com `decision` e `decided_by`.

Headless / CI: trust prompt é **fail-closed** — se `--auto-approve-mcp` não foi passado explicitamente, server fica `denied` em CI. Em headless interativo (TTY), modal funciona normal.

### 1.6 Activate & call

`tools/list` resposta vira `register` no Tool Registry com namespacing:

```
mcp:postgres:query
mcp:postgres:list_tables
mcp:github:create_issue
```

Tools registradas com `visible_to_model: true` (se `trusted`) entram no `tool_schemas` cache breakpoint #2. Modelo as vê com nome completo namespaced.

Quando modelo emite `tool_use` com `name: "mcp:postgres:query"`:
1. Harness valida input contra `inputSchema`
2. Permission engine aplica policy (mesmas regras de tools canônicas)
3. Se `disconnected`: handshake on-demand (transição para `active`)
4. Envia `tools/call` JSON-RPC
5. Aguarda response (≤ timeout); aplica `notifications/cancelled` se user interrompeu
6. Output validado contra `outputSchema` (se declarado), redactor aplicado, retorna ao modelo

Per-server timeout default 30s; per-call override via input não é permitido (decisão de policy, não de tool).

### 1.7 Disconnect

`SessionStop` (não `Stop` hook — fim normal de sessão) envia `notifications/cancelled` pendentes e fecha transport. Stdio: SIGTERM com 2s grace, depois SIGKILL. SSE/HTTP: close connection.

Server crash mid-session: tudo que estava em vôo recebe error; server transita para `disconnected`. Reconnect só na próxima invocação (não automático em background).

---

## 2. Transport

| Transport | Spec | v1? | Quando usar |
|---|---|---|---|
| **stdio** | MCP padrão; processo local com pipes | ✓ | tools que precisam de FS local, processos curtos |
| **SSE** | Server-Sent Events sobre HTTP | ✓ | servers remotos persistentes, streaming de progress |
| **streamable HTTP** | MCP 2024-11+ | v1.1 | desejável mas não bloqueante |
| **WebSocket** | não-padrão | ✗ | rejeitado: extra complexity sem ganho mensurável |

### 2.1 Stdio

- Spawn: `spawn(command[0], command.slice(1), { env: cleanEnv, cwd, stdio: ["pipe", "pipe", "pipe"] })`
- Env limpa: apenas `PATH`, `HOME`, `USER`, `MCP_*` vars do user, vars declaradas em `[servers.<name>.env]`
- `setsid` para evitar processo órfão (Linux/macOS)
- stderr capturado, redirected para `traces/mcp-<name>.log` (rotacionado em 10MB)
- Detected dead via `kill(pid, 0)` ou `EPIPE` em writes

### 2.2 SSE

- HTTP GET com `Accept: text/event-stream`
- POST para mesmo endpoint para enviar JSON-RPC
- Heartbeat detectado por gap > `heartbeat_max_age` (default 60s)
- Retry exponencial em conexão initial; **sem retry** após `trusted` (transient failure → `disconnected`)
- Auth: `Authorization: Bearer $TOKEN` se `auth.kind = "bearer"`; nunca em config inline

### 2.3 Sandbox (stdio)

Mesmo profile que `bash` sandboxing (`SECURITY_GUIDELINE.md §8.1`):
- Mount: `cwd` declarado read-write; `~/.config/agent` read; `/tmp` read-write isolado; resto read-only
- Network: per-server allowlist (`[servers.<name>.network.allow_hosts]`); default DENY
- Ulimits: CPU 30s soft, memory 512MB soft, file size 100MB
- Linux: `bwrap` opt-in; macOS: `sandbox-exec` opt-in; opt-in via `[servers.<name>.sandbox = true]`

Sandbox em SSE/HTTP servers: N/A (server roda fora; agente só faz HTTP). Trust boundary é a rede.

---

## 3. Manifest

### 3.1 Schema canônico

`tools/list` response:

```jsonc
{
  "tools": [
    {
      "name": "query",                     // sem namespace; harness adiciona "mcp:postgres:"
      "description": "Run a read-only SELECT against the configured database.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "sql": { "type": "string", "description": "SELECT only" }
        },
        "required": ["sql"]
      },
      "outputSchema": {                    // optional but encouraged
        "type": "object",
        "properties": {
          "rows": { "type": "array" },
          "columns": { "type": "array" }
        }
      },
      "_meta": {                           // extension namespace
        "agentic_cli": {
          "parallel_safe": true,
          "writes": false,
          "idempotent": true
        }
      }
    }
  ]
}
```

Campos `_meta.agentic_cli.*` são **opt-in** mas considerados pelo harness:

| Campo | Default se ausente | Usado em |
|---|---|---|
| `parallel_safe` | `false` | `ORCHESTRATION.md §1` (parallel tool calls) |
| `writes` | `true` (pessimista) | `CONTRACTS.md §2` (checkpoint pré-call) |
| `idempotent` | `false` | retry policy em `FAILURE_MODES.md` |
| `reads_secrets` | `false` | redaction extra em audit |

### 3.2 Hash canonical

```
manifest_hash = SHA256(
  canonical_json({
    tools: sorted_by_name(tools),
    serverInfo: { name, version }
  })
)
```

`canonical_json` é JSON com chaves ordenadas, sem whitespace, encoding UTF-8 NFC. Mudança em qualquer campo de qualquer tool → hash diferente.

`description` muda mas comportamento não → ainda é breaking change (modelo lê descrição). Trust é re-prompted.

### 3.3 Diff em re-trust

Quando hash muda, modal mostra diff estruturado:

```
Server postgres mudou:
  + tool: list_indexes
  ~ tool: query
      inputSchema: adicionou parâmetro `timeout_ms`
  - tool: explain_plan

Aprovar nova versão? [y/N]
```

Diff computed comparando manifest cacheado vs novo. Display em `UI.md` (TBD: componente `<MCPDiffModal>`).

---

## 4. Namespacing

### 4.1 Regras

- Tools MCP **sempre** aparecem como `mcp:<server>:<tool>` no registry, audit, slash commands, output
- `<server>` vem de `[servers.<name>]` em config; `<tool>` vem de `tools/list`
- Colisão com tool canônica (`§2.6` em `CONTRACTS.md`): registro **rejeitado** com `mcp.namespace.shadow_canonical`
- Colisão entre dois MCP servers: ambos sobrevivem (namespacing por server resolve); sem fallback

### 4.2 Tools canônicas reservadas

Os seguintes nomes são reservados; servers não podem usar (mesmo sem namespace):

```
read_file, write_file, edit_file, glob, grep, bash,
task_sync, task_async, task_await, task_cancel,
memory_search, fetch_url
```

Razão: replay determinístico. Se `read_file` pode vir tanto de canônico quanto de MCP, replay sem o MCP server diverge silenciosamente.

### 4.3 Slash commands

Tools MCP **não** ganham slash command automático. Razão: slash commands são UX curados (`AGENTIC_CLI.md §2.5`); auto-promotion poluiria. Se user quer atalho, define manualmente em `~/.config/agent/commands/`.

---

## 5. Per-server budget

Limites operacionais (`PERFORMANCE.md §8`):

| Variável | Default | Cap absoluto | Onde |
|---|---|---|---|
| `max_concurrent_servers` | 10 | 30 | global |
| `max_calls_per_session` | 200 | 1000 | per-server |
| `max_tokens_in_per_session` | 50k | 500k | per-server (output que volta ao modelo) |
| `timeout_ms` | 30000 | 60000 | per-call |
| `heartbeat_max_age_ms` | 60000 | 300000 | SSE/HTTP only |

Excedeu cap absoluto → server transita para `disconnected` com `failure_event` `mcp.budget.exceeded`. Soft cap → warning em audit, sem ação.

Budget herda de step parent em `orchestrated` profile (cascading via `ORCHESTRATION.md §11`).

---

## 6. Cache impact

Tools MCP entram no `tool_schemas` cache breakpoint #2 (`CONTEXT_TUNING.md §2`). Implicações:

- **Trust grant** invalida cache (tools novas no schema)
- **Manifest change** invalida cache
- **Server `disconnected`** com tools antes visíveis → harness re-renderiza `tool_schemas` removendo o server, invalida cache
- **Lazy activation** não invalida cache (server passa de `trusted` para `active` sem mudar schema)

Cache invalidation custo: ~5-30k tokens em next call. Aceitável: events que invalidam cache são raros (trust, manifest change). Sessão típica: zero invalidations após primeira prompt.

Anti-pattern: server que muda manifest a cada minuto (ex: server "dynamic" que adiciona tools baseado em FS state). Documentado em `ANTI_PATTERNS.md §6.2`.

---

## 7. Slash commands

```
/mcp list                          # servers ativos + estado
/mcp show <name>                   # detalhe + manifest hash + tools
/mcp trust <name>                  # forçar trust prompt (ex: re-aprovar após config change)
/mcp revoke <name>                 # transita para denied; tools invisíveis
/mcp reconnect <name>              # força reconnect (em caso de degraded)
/mcp doctor                        # diagnóstico de cada server (latência, last error, status)
/mcp logs <name>                   # tail dos logs (stderr capturado)
```

Implementação compartilha pipeline com slash commands canônicos (`AGENTIC_CLI.md §2.5`). Output JSON via `--json` flag.

---

## 8. Failure modes (cross-ref)

> **Catálogo operacional completo:** [`FAILURE_MODES.md §15`](./FAILURE_MODES.md) — playbook de recovery por code, mensagens-template, audit footprint, queries de aggregate. Tabela abaixo é só índice rápido.

| Code | Estado | Recovery | Detalhe |
|---|---|---|---|
| `mcp.protocol.version_mismatch` | `error` | user atualiza harness/server | `FAILURE_MODES.md §15.1` |
| `mcp.transport.broken` | `disconnected` | reconnect lazy; threshold 3/60s | `§15.2` |
| `mcp.timeout` | `active` | tool result com error; modelo decide | `§15.3` |
| `mcp.schema.invalid` (input) | `active` | rejeição pre-send; modelo retry | `§15.4` |
| `mcp.output.invalid` | `degraded` | warning; recover em 3 outputs válidos | `§15.5` |
| `mcp.budget.exceeded` | `disconnected` | bloqueio até próxima sessão | `§15.6` |
| `mcp.manifest.changed` | `trust_pending` | trust prompt | `§14.2` |
| `mcp.namespace.shadow_canonical` | server não registra | fix config (fail-fast em SessionStart) | `§15.7` |
| `mcp.metadata.writes_lied` | `degraded` + flag persistente | checkpoint forçado em calls subsequentes | `§15.8` |

---

## 9. Observabilidade

### 9.1 Tabelas (ver `AUDIT.md §1.5`)

- `mcp_servers` — config + estado atual (1 row per server name)
- `mcp_manifest_history` — versions de manifest (append-only, forever retention)
- `tool_calls` ganha coluna `mcp_server` para distinguir canônico vs MCP

### 9.2 Spans OTEL

```
mcp.handshake          attrs: { server, protocol_version, duration_ms }
mcp.trust_prompt       attrs: { server, hash, decision }
mcp.tools_list         attrs: { server, tool_count }
mcp.tools_call         attrs: { server, tool, status, duration_ms, tokens_out }
mcp.transport_event    attrs: { server, kind: "connect"|"close"|"error" }
```

### 9.3 `agent doctor`

`agent doctor` (`AGENTIC_CLI.md §2.1`) inclui section MCP com:
- Servers em config
- Estado atual de cada
- Last error (se houver)
- Latência média (últimas 7 dias)
- Manifest hash

---

## 10. Anti-patterns (ver [`ANTI_PATTERNS.md §6`](./ANTI_PATTERNS.md))

Resumo do que **não** fazer com MCP. Detalhamento, motivo, e substituição em `ANTI_PATTERNS.md §6`:

- ❌ Re-implementar tools canônicas via MCP (`§6.1`)
- ❌ Server que muda manifest a cada sessão (`§6.2`)
- ❌ Auth via env var hardcoded em config (`§6.3`)
- ❌ Server que ignora `network.allow_hosts` (`§6.4`)
- ❌ Server que escreve em diretórios do agente (`§6.5`)
- ❌ `--auto-approve-mcp` por default em CI (`§6.6`)
- ❌ Tool MCP com `_meta.agentic_cli.writes: false` que escreve (`§6.7`)
- ❌ MCP-over-MCP / chain de servers (`§6.8`)
- ❌ Slash command auto-promotion para tools MCP (`§6.9`)

---

## 11. Limites declarados (v1)

- **Sem MCP no critical path.** Tools canônicas não podem ser substituídas. Reconsiderar se eval mostrar > 30% de uso vindo de MCP em workflow específico (sinaliza que canônico tem gap real).
- **Sem `prompts/get` / `resources/read` no v1.** MCP suporta esses primitives, mas v1 só consome `tools/*`. Razão: `prompts/get` colide com playbooks (`PLAYBOOKS.md`); `resources/read` colide com FS tools. v2 reconsidera.
- **Sem `roots/list` no v1.** MCP permite server pedir lista de roots (workspaces). Harness não responde no v1; servers que dependem disso degradam graciosamente ou erro.
- **Sem `sampling/createMessage` no v1.** MCP permite server pedir LLM call ao cliente. Harness recusa: server não dirige o modelo. v2 reconsidera com policy estrita.
- **Sem MCP-over-MCP.** Server não pode declarar dependência em outro MCP server; chain é responsabilidade do user na config.

---

## 12. Insight final

MCP é **superfície de extensão**, não **API de integração**. Diferença prática:

- API: você espera comportamento estável, contratos fortes, breaking changes raros.
- Extensão: você espera variabilidade, contratos best-effort, breaking changes esperados.

Trust prompt + manifest hash + namespacing + budget + sandbox são as 5 defesas que tornam extensão viável **sem** tratar como API. Sem elas, MCP é vetor de bug, não recurso.

Spec sem este doc: MCP é conceito. Com este doc: MCP é **protocolo operável**.
