# FAILURE_MODES

Catálogo de modos de falha do `AGENTIC_CLI` com playbook de recovery por classe. Detecção, diagnóstico, ação automática, mensagem ao usuário, audit trail.

Sistema de produção é definido pelo que faz quando dá errado, não quando dá certo. Spec sem catálogo de falhas é receita pra "funcionou na minha máquina".

---

## 0. Princípios

1. **Falha é estado de primeira classe.** Toda falha tem classe, código, payload, audit.
2. **Fail closed por default em segurança.** Em dúvida, nega. Em ambiguidade, bloqueia.
3. **Fail forward em UX.** Em falha não-crítica, agente continua com aviso, não morre.
4. **Recovery hierarchy:** retry transparente → retry com backoff → escalar → preservar estado → abortar com instruction → erro fatal.
5. **Mensagem ao usuário é template versionado.** Não improvisada. Acionável: o que aconteceu + o que tentar.
6. **Toda falha vai pra audit trail.** SQLite + NDJSON. Sem exceção.
7. **Mata-mata silencioso é bug.** Falha que não chega ao usuário nem ao log = bug crítico.

---

## 1. Taxonomia

```
┌─ External (não controlamos) ──────────────────────────────┐
│  Provider (LLM API)                                        │
│  Network                                                   │
│  Filesystem (disk full, perms, corruption)                 │
│  Process signals (SIGKILL externo)                         │
└────────────────────────────────────────────────────────────┘

┌─ Internal (controlamos, devemos prevenir) ────────────────┐
│  Storage (SQLite ops)                                      │
│  Checkpoint                                                │
│  Hook                                                      │
│  Subagent                                                  │
│  Compaction                                                │
│  Validator (orchestrated)                                  │
│  Permission/Config                                         │
└────────────────────────────────────────────────────────────┘

┌─ User-induced (esperado) ─────────────────────────────────┐
│  Cancel (Ctrl+C, kill)                                     │
│  Bad input (malformed prompt, invalid command)             │
│  Trust violation (acesso a dir não-confiado)               │
└────────────────────────────────────────────────────────────┘

┌─ Adversarial (segurança) ─────────────────────────────────┐
│  Prompt injection                                          │
│  Memory poisoning                                          │
│  MCP server hostil                                         │
│  Path traversal                                            │
└────────────────────────────────────────────────────────────┘
```

Cada falha tem **código** no formato `<classe>.<subtipo>.<detalhe>`, ex: `provider.timeout.streaming`, `storage.sqlite.disk_full`, `checkpoint.git.corrupt_index`.

---

## 2. Provider failures

### 2.1 `provider.timeout.request` — request não respondeu em N segundos

**Detecção:** request HTTP excede `request_timeout_ms` (default 60s).
**Diagnóstico:** rede lenta, provider sobrecarregado, modelo travou.
**Recovery:**
1. Backoff exponencial (200ms, 800ms, 3.2s)
2. 3 tentativas
3. Após terceira falha: vira `error` no step (não na sessão)
4. Modelo recebe tool_result `{ error: "provider timeout, retried 3x" }` no próximo step **ou** sessão escala pra fallback (em profile `hybrid`)

**Mensagem ao usuário:**
> Provider lento (3 timeouts consecutivos). Continuando com aviso. `/cost` mostra impacto.

**Audit:** `traces` span `provider.request` com `status=timeout`, `retries=3`.

### 2.2 `provider.rate_limit.429` — rate limit hit

**Detecção:** HTTP 429 ou erro estruturado do SDK.
**Recovery:**
1. Respeita `Retry-After` header se presente
2. Senão, backoff começando em 5s (mais agressivo)
3. Após 3 tentativas com backoff: erro de step
4. Eventual: pode mudar pra `error_fatal` se rate limit persistente em `> 5min`

**Mensagem:**
> Rate limit do provider. Aguardando {retry_after}s antes de tentar novamente. Considere `/model` pra trocar.

### 2.3 `provider.5xx.persistent` — provider down

**Detecção:** HTTP 5xx repetido após retries.
**Recovery:**
1. Em profile `autonomous`: `error_fatal`. Sessão preservada para resume manual.
2. Em profile `hybrid`: escala pra `fallback` model automaticamente; emite warning.

**Mensagem:**
> Provider {name} indisponível. Sessão pausada. `/resume` quando voltar, ou `/model` pra trocar.

### 2.4 `provider.malformed.tool_use` — modelo emitiu tool_use que não parseia

**Detecção:** stream interrompido com tool_use incompleto OU args não-parseáveis após stream completar.
**Recovery:**
1. Tool_use é descartada (não persiste em `tool_calls`)
2. Modelo recebe mensagem do sistema: "Sua última tool call estava malformada. Tente de novo com args válidos."
3. Re-gera step
4. Após 2 falhas consecutivas: erro de step

**Mensagem:** silenciosa por default; visível em `--verbose`.

### 2.5 `provider.stream.cut_mid_response` — conexão caiu mid-stream

**Detecção:** EOF ou network error durante leitura de stream.
**Recovery:**
1. Output parcial **descartado** (nem persistido como step incompleto)
2. Re-emit do mesmo request (provider deve ser idempotente para isso)
3. Custo do request abortado: contado como spent (provider cobra mesmo assim)

---

## 3. Tool failures

### 3.1 `tool.timeout` — tool excedeu timeout

**Detecção:** `setTimeout` no harness; tool não retornou em ≤ `tool_timeout_ms` (default 30s, override por tool).
**Recovery:**
1. AbortSignal disparado
2. Aguarda 2s graceful
3. SIGKILL no subprocess (se aplicável)
4. tool_call.status = 'error', output = `{ error: "tool timeout", code: "tool.timeout" }`
5. Modelo recebe error como tool_result; decide se tenta de novo ou outra tool

**Mensagem:**
> Tool {name} excedeu timeout ({n}s). Resultado: erro. Próximo step decide.

### 3.2 `tool.exception` — tool lançou exception não-capturada

**Detecção:** try/catch ao redor do execute().
**Recovery:**
1. Exception convertida em `{ error, code: "tool.exception", details }`
2. tool_call.status = 'error'
3. Modelo decide

**Mensagem:** silenciosa salvo `--verbose`. Audit completo no NDJSON.

### 3.3 `tool.schema_violation.input` — args do modelo não satisfazem schema

**Detecção:** validação JSON Schema antes da invocação.
**Recovery:**
1. Tool **não é invocada**
2. tool_call.status = 'denied' com motivo `schema_violation`
3. Modelo recebe `{ error: "args invalid", validation: <details>, hint: <fix>}`
4. Eval específico: schema violations > 5% → tool description ruim

### 3.4 `tool.schema_violation.output` — tool retornou output fora do schema

**Detecção:** validação JSON Schema do output (se `outputSchema` declarado).
**Recovery:**
1. Resultado é loggado como bug (audit)
2. Modelo recebe output cru com warning
3. Tool em quarentena se 3 violações em 24h (eval)

### 3.5 `tool.writes_without_checkpoint` — tool com `writes: true` falhou em criar checkpoint

**Detecção:** Checkpoint Manager retornou erro pré-invocação.
**Recovery:**
1. Tool **não é invocada**
2. tool_call.status = 'denied'
3. Erro escalado pra usuário (não é falha "soft")

**Mensagem:**
> Não consegui criar checkpoint antes de {tool}. Tool abortada. Verifique espaço em disco e estado do git. Detalhes: {checkpoint_error}.

### 3.6 `tool.bash.killed_external` — bash subprocess morto fora do controle do harness

**Detecção:** exit code = 137 (SIGKILL) sem signal interno.
**Recovery:**
1. Result = `{ error: "bash killed externally", exit_code: 137 }`
2. Modelo decide
3. Audit avisa que pode ter side effect parcial (DB modificado, arquivo escrito parcial, etc)

---

## 4. Storage failures (SQLite)

### 4.1 `storage.sqlite.disk_full` — espaço em disco insuficiente

**Detecção:** SQLITE_FULL retornado em INSERT/UPDATE.
**Recovery:**
1. Step **abortado** (não pode persistir)
2. Sessão entra em `error_fatal`
3. State em RAM dumpado para `~/.local/share/agent/recovery/<session>.json` (best-effort)
4. Mensagem fatal ao usuário com instrução de cleanup

**Mensagem:**
> ⚠ Disco cheio. Sessão preservada em recovery file. Libere espaço e rode `agent recover <id>`.

### 4.2 `storage.sqlite.locked` — DB lockado por outro processo

**Detecção:** SQLITE_BUSY após retry interno.
**Recovery:**
1. Detecta se outro `agent` está rodando no mesmo cwd (lockfile)
2. Se sim: erro `multi_instance_detected`, oferece anexar ou abortar
3. Se não: assume corruption ou crash anterior; tenta `PRAGMA integrity_check`
4. Se OK: aguarda 5s, retry; se persistente: erro fatal

### 4.3 `storage.sqlite.corruption` — `PRAGMA integrity_check` retorna não-OK

**Detecção:** verificação periódica ou em SessionStart.
**Recovery:**
1. Sessão **não é resumível**
2. DB renomeada para `sessions.db.corrupt-<timestamp>`
3. Nova DB criada vazia
4. Usuário avisado; oferece `agent db salvage <corrupt-file>` (best-effort recovery)

### 4.4 `storage.write_atomic_failure` — INSERT/UPDATE em transação atômica falhou

**Detecção:** rollback em transação SQLite.
**Recovery:**
1. Estado da sessão **não muda** (transação rolled back)
2. Step retentado uma vez
3. Falha persistente: erro fatal

---

## 5. Hook failures

### 5.1 `hook.timeout` — hook não terminou em tempo

**Detecção:** `setTimeout` no dispatcher.
**Recovery:**
1. SIGTERM no processo do hook; após 1s SIGKILL
2. Em evento bloqueável: assume `allow` (default), warning
3. Em evento não-bloqueável: registra `timeout` em `hook_runs`
4. Se hook tem `fail_closed: true`: assume `block` em vez de allow

**Mensagem:** warning em UI rodapé; visível em `/hooks audit`.

### 5.2 `hook.command_not_found` — comando do hook não existe (exit 127)

**Detecção:** exit code 127.
**Recovery:**
1. Loga em `hook_runs` com status='error'
2. **Não bloqueia** ação alvo (default; configurável por hook com `fail_closed`)
3. Eval cobre: hooks "broken" estão em `/hooks` UI

### 5.3 `hook.subprocess_orphan` — hook deixou subprocess órfão

**Detecção:** PID detectado em pgrep após dispatcher considerar terminado.
**Recovery:**
1. Documentado como **limitação** — hook é responsabilidade do usuário
2. Recomendação: usar `setsid`, trap no script, ou hooks que respeitam SIGTERM
3. Audit: warning em `/hooks audit`; sem cleanup automático

---

## 6. Checkpoint failures

### 6.1 `checkpoint.git.no_repo` — `cwd` não é git repo

**Detecção:** `git rev-parse --git-dir` falha.
**Recovery:**
1. Fallback automático para reflink (btrfs, xfs, apfs)
2. Se reflink indisponível: fallback `cp -r` (não atômico, warning)
3. Se até cp falha: erro fatal pra tool com `writes: true`

**Mensagem inicial (1ª vez):**
> Não é um repo git. Usando snapshot via {reflink|cp}. `/undo` pode não cobrir tudo. Considere `git init`.

### 6.2 `checkpoint.git.corrupt_index` — git tem index corrompido

**Detecção:** `git stash create` falha com erro de index.
**Recovery:**
1. Tenta reset do index (sem perder working tree): `git reset HEAD`
2. Re-tenta stash
3. Se ainda falha: fallback reflink/cp
4. Audit indica fallback foi usado; `/undo` pode falhar

### 6.3 `checkpoint.disk_full` — sem espaço para snapshot

**Detecção:** ENOSPC em escrita do snapshot.
**Recovery:**
1. Tool com `writes: true` **abortada** (sem checkpoint = sem prosseguir)
2. Mensagem fatal ao usuário com cleanup

### 6.4 `checkpoint.restore.conflict` — `/undo` não consegue aplicar (conflito)

**Detecção:** `git stash apply` retorna conflito.
**Recovery:**
1. Estado do FS preservado (sem aplicar parcial)
2. Mensagem ao usuário com instruções (lista de arquivos em conflito)
3. Manual fixup recomendado

---

## 7. Subagent failures

### 7.1 `subagent.crash` — processo morreu sem output

**Detecção:** exit code != 0 e `subagent_outputs.payload` vazio.
**Recovery:**
1. Pai recebe output sintético: `{ status: "error", output: "subagent crashed", errors: ["<exit_code>", "<stderr_tail>"] }`
2. Worktree (se isolation) é cleanup'd: `git worktree remove --force` + branch delete
3. Pai decide: retry, fallback, ou continuar

### 7.2 `subagent.worktree.conflicts_at_end` — subagent terminou mas worktree tem mudanças não-commitadas conflitantes

**Detecção:** `git status` no worktree mostra mudanças após "done".
**Recovery:**
1. Se subagent declarou `commit_on_done: true`: tenta `git commit -am "agent: <task>"`
2. Se commit falha (ex: hook rejeitou): worktree preservado; pai recebe output com `artifacts: [{ kind: "worktree", path }]`
3. Cleanup: usuário manualmente ou via `agent worktree gc`

### 7.3 `subagent.timeout` — excedeu wall_clock

**Detecção:** `subagent_outputs.last_heartbeat` parou de atualizar por > `wall_clock_timeout`.
**Recovery:**
1. SIGTERM; 5s grace; SIGKILL
2. Pai recebe `{ status: "interrupted", output: "subagent timed out", ... }`
3. Worktree cleanup

---

## 8. Configuration failures

### 8.1 `config.yaml.parse_error` — config inválido

**Detecção:** YAML parser erro em load.
**Recovery:**
1. **Sessão não inicia.** Fail closed.
2. Mensagem com path, linha, coluna, motivo
3. Aponta arquivo de exemplo conhecido bom

**Mensagem:**
> ⚠ Config inválido em {path}:{line}. {error}. Veja `~/.config/agent/permissions.yaml.example` ou rode `agent config validate`.

### 8.2 `config.permission.conflicting_rules` — allow + deny match no mesmo input

**Detecção:** validação no load de permission engine.
**Recovery:**
1. Warning em load (não bloqueia)
2. Em runtime: **deny vence** (least privilege)
3. `/perms` mostra conflito explicitamente

### 8.3 `config.locked_override_attempt` — usuário tentou override de regra `locked` (enterprise)

**Detecção:** merge de hierarquia detecta override em key `locked`.
**Recovery:**
1. Override **ignorado** silenciosamente (com log)
2. Audit registra tentativa
3. Em modo `--verbose`: warning visível

---

## 9. Memory failures

### 9.1 `memory.frontmatter.invalid` — `.md` de memória com frontmatter quebrado

**Detecção:** parse de YAML frontmatter falha.
**Recovery:**
1. Memória **não é carregada** no contexto
2. Aparece em `/memory list` com flag `[corrupt]`
3. Suggestão: `/memory edit <name>` ou delete manual

### 9.2 `memory.index.missing` — `MEMORY.md` ausente

**Detecção:** índice esperado não existe.
**Recovery:**
1. Reconstrói do diretório (escaneia todos os `.md`, gera índice)
2. Avisa via log
3. Operação one-time (se reconstrução também falhar: erro fatal sob aquele scope)

### 9.3 `memory.injection.detected` — scanner heurístico match

**Detecção:** body contém pattern (`ignore previous instructions`, etc) na operação de write.
**Recovery:**
1. Write **bloqueado** (não só warning)
2. `memory_events` action='refused', details com pattern matched
3. Mensagem ao usuário

**Mensagem:**
> ⚠ Memória bloqueada — pattern de injection detectado: "{pattern}". Origem: {source}. Use `/memory audit` pra revisar.

### 9.4 `memory.path_traversal` — tentativa de escape do diretório

**Detecção:** `name` contém `..`, `/`, ou path absoluto.
**Recovery:**
1. **Erro fatal** (não bug recoverable; é tentativa adversarial ou bug grave)
2. Audit completo
3. Sessão pode continuar; tool de memory marcada como `compromised` no scope da sessão

---

## 10. Compaction failures

### 10.1 `compaction.llm.malformed_output` — Haiku retornou texto inválido

**Detecção:** parse falha no schema esperado.
**Recovery:**
1. Retry uma vez com prompt mais estrito
2. Falha persistente: **mantém contexto não-compactado**, aborta sessão se contexto > limit hard
3. Em profile `orchestrated`: usa modelo do backend pra compaction

### 10.2 `compaction.preserves_loss` — eval detecta que compaction perdeu goal

**Detecção:** eval específico (offline) compara antes/depois.
**Recovery:**
1. Não detectado em runtime; só em CI
2. Compaction prompt versionado; rollback de prompt como fix
3. Em produção: feature flag pra desativar compaction temporariamente

---

## 11. Validator failures (profile `orchestrated`)

### 11.1 `validator.persistent_failure` — esgotou retries

**Detecção:** retry_count >= max_retries em DAG node.
**Recovery:**
1. Node em estado `failed`
2. DAG aplica `on_failure` policy:
   - `abort` — DAG inteiro falha
   - `skip` — node marcado como skipped, dependentes recebem `null`
   - `escalate` — em hybrid: re-roda node com fallback model
3. Subagent (se DAG é de subagent) reporta error

### 11.2 `validator.timeout` — validator demorou > 200ms (built-in fast)

**Detecção:** wall-clock no validator.
**Recovery:**
1. Considerado `{ ok: false, fatal: true }` (não retry)
2. Bug do validator; eval cobre
3. Step falha imediato (sem retry)

---

## 12. Permission failures

### 12.1 `permission.policy.parse_error` — policy file inválido

**Detecção:** load de policy falha.
**Recovery:**
1. **Sessão não inicia.** Fail closed.
2. Mensagem com erro

### 12.2 `permission.runtime.deny` — policy negou tool call (não é "falha", é "esperado")

**Detecção:** decisão `deny` em policy match.
**Recovery:**
1. tool_call.status = 'denied'
2. Modelo recebe `{ error: "denied by policy", reason }`
3. Modelo decide

Não é falha de fato; está aqui pra completude do catálogo.

---

## 13. User-induced failures

### 13.1 `user.interrupt` — Ctrl+C

**Detecção:** SIGINT.
**Recovery:** ver §2.6 do STATE_MACHINE.md (interrupted state).

### 13.2 `user.terminal_closed` — terminal fechou (SIGHUP)

**Detecção:** SIGHUP.
**Recovery:**
1. Comportamento configurável (`on_sighup: pause | exit | ignore`)
2. Default: **pause**. Sessão preservada; resume manual.
3. Em headless mode: exit imediato.

### 13.3 `user.disk_full_user_dir` — `~/.config` ou `~/.local/share` cheios

**Detecção:** ENOSPC em ops de config/state.
**Recovery:** mesmo de `storage.sqlite.disk_full`.

### 13.4 `user.invalid_slash_command` — comando inválido

**Detecção:** parse de `/<cmd>`.
**Recovery:**
1. Mensagem com sugestões (Levenshtein distance < 3)
2. Sessão continua

---

## 14. Adversarial failures (segurança)

### 14.1 `adversarial.prompt_injection.in_tool_output`

**Detecção:** scanner heurístico em tool output (`ignore previous`, etc).
**Recovery:**
1. Tool output marcado com `<injection-suspected>` tag
2. Modelo é avisado via system prompt addendum
3. Audit registra

### 14.2 `adversarial.mcp_server.changed_manifest`

**Detecção:** hash do manifest do MCP server diferente do gravado em `trusted_dirs`.
**Recovery:**
1. Tools do server **invisíveis** ao modelo
2. Trust prompt re-mostrado
3. Audit completo

### 14.3 `adversarial.path_traversal.in_tool_args`

**Detecção:** validação de paths (resolve real path; checa se está dentro de `cwd` allowed).
**Recovery:**
1. Tool **denied** com motivo `path_traversal`
2. Audit
3. Tool fica em quarentena no scope da sessão (3 violações = block permanent)

---

## 15. MCP failures

> **Cross-refs:** [`MCP.md §8`](./MCP.md) catálogo curto; [`STATE_MACHINE.md §6.5`](./STATE_MACHINE.md) máquina de estado; [`CONTRACTS.md §11`](./CONTRACTS.md) contrato A/B; [`AUDIT.md §1.5`](./AUDIT.md) tabelas `mcp_servers` e `mcp_manifest_history`. Esta seção é o **catálogo operacional** com playbook de recovery por code.

Classificação (§1 Taxonomia):

| Code | Classe | User-visible | Estado pós-falha |
|---|---|---|---|
| `mcp.protocol.version_mismatch` | external | sim | `error` |
| `mcp.transport.broken` | external (transient) | depende | `disconnected` |
| `mcp.timeout` | external (transient) | sim | `active` (tool result com error) |
| `mcp.schema.invalid` (input) | internal | não (modelo retry) | `active` |
| `mcp.output.invalid` | external | não (warning audit) | `degraded` |
| `mcp.budget.exceeded` | configuration | sim | `disconnected` (até próxima sessão) |
| `mcp.namespace.shadow_canonical` | configuration | sim (fail closed em startup) | server não registra |
| `mcp.metadata.writes_lied` | adversarial-ish (server bug) | sim | `degraded` + flag |
| `mcp.manifest.changed` | adversarial | sim | `trust_pending` (ver §14.2) |

`mcp.manifest.changed` já está em `§14.2`. Os 8 abaixo são novos.

### 15.1 `mcp.protocol.version_mismatch`

**Detecção:** após `initialize`, server retorna `protocolVersion` ≠ versão suportada pelo harness (default `2024-11-05`).

**Recovery:**
1. Server transita pra `error`; tools **nunca** registradas.
2. UI modal: "Server `<name>` requer protocolo X; harness suporta Y."
3. Sessão prossegue **sem** o server; tools dele invisíveis.
4. `failure_event` registrado em SessionStart, não bloqueia sessão.

**Mensagem-template:**

```
⚠ MCP server "{name}" incompatível

Server declarou protocolo {server_protocol}, mas harness suporta {harness_protocol}.
Tools deste server ficam indisponíveis nesta sessão.

Tente: atualizar o server ou o agentic-cli (`agent doctor` mostra versões)
       /mcp logs {name}    (ver erro do server)

(detalhes: mcp.protocol.version_mismatch | /help mcp.protocol)
```

**Audit footprint:** `failure_events.payload = { server, server_protocol, harness_protocol }`. Persiste enquanto config tiver o server.

### 15.2 `mcp.transport.broken`

**Detecção:**
- Stdio: `EPIPE` em write, ou `kill(pid, 0)` retorna `ESRCH`.
- SSE: gap > `heartbeat_max_age_ms` (default 60s) sem evento.
- HTTP: response 5xx em `tools/call` ou connection reset.

**Recovery:**
1. Server transita `*` → `disconnected`.
2. Tool em vôo (se houver) recebe `{ error: "mcp.server.crashed", server }` (cobre `STATE_MACHINE.md §7.1`).
3. **Reconnect lazy**: nada acontece em background; próxima invocação tenta reconnect.
4. Tools do server permanecem **registradas e visíveis** ao modelo (não invisibilizadas em transient failure — design intencional).
5. Após 3 falhas consecutivas em janela de 60s → tools invisibilizadas até `/mcp reconnect <name>` explícito.

**Mensagem-template (apenas após threshold):**

```
⚠ MCP server "{name}" instável

3 falhas de transporte em 60s. Tools deste server ficaram ocultas.

Tente: /mcp reconnect {name}
       /mcp logs {name}

(detalhes: mcp.transport.broken | /help mcp.transport)
```

**Audit footprint:** cada falha vira `failure_event`; threshold de 3 vira `mcp_servers.last_error = "transport.broken"` + state `disconnected`.

### 15.3 `mcp.timeout`

**Detecção:** `tools/call` não retornou em `mcp_servers.<name>.timeout_ms` (default 30s, cap 60s).

**Recovery:**
1. Harness envia `notifications/cancelled` com matching id.
2. Aguarda 2s graceful para server reconhecer.
3. Resultado vira `{ error: "mcp.timeout", server, tool, elapsed_ms }`.
4. Server **permanece** `active` (timeout não derruba); contador `mcp_servers.timeout_count_session += 1`.
5. Após 3 timeouts consecutivos do mesmo tool → tool flag `degraded_in_session` (modelo recebe warning quando escolher).

**Mensagem ao usuário:** apenas se ≥ 3 timeouts no mesmo tool em 1 sessão (senão é ruído).

**Audit footprint:** `failure_events.payload = { server, tool, elapsed_ms, request_id }`. Permite query "qual tool MCP é mais lenta?".

### 15.4 `mcp.schema.invalid` — input

**Detecção:** input emitido pelo modelo **não valida** contra `inputSchema` declarado pelo server em `tools/list`.

**Recovery:**
1. Harness **não envia** ao server (rejeição pre-send).
2. Modelo recebe `{ error: "mcp.schema.invalid", hint: "<validation_error_path>: <msg>" }` como tool result.
3. Modelo decide retry; harness não auto-retry (decisão do modelo, não do harness).
4. Server permanece `active`; nada muda no estado dele.

**Mensagem ao usuário:** silenciosa. Modelo lida.

**Audit footprint:** `failure_events.payload = { server, tool, validation_error, schema_hash }`. Trend útil: "esse modelo erra mais em qual schema?".

### 15.5 `mcp.output.invalid`

**Detecção:** server retornou output que **não valida** contra `outputSchema` (se declarado). Se schema ausente: heurística mínima — não-string em campo `text`, content array vazio, etc.

**Recovery:**
1. Harness loga; devolve ao modelo `{ error: "mcp.output.invalid", hint, raw_output_truncated }` (raw com truncate em 1KB).
2. Server transita `active` → `degraded`.
3. Após 3 outputs válidos consecutivos do mesmo server → recover para `active` (`STATE_MACHINE.md §6.5` evento `mcp.recover_ok`).
4. Modelo decide retry, fallback, ou abandono — não automatic.

**Mensagem ao usuário:** apenas se persistir > 5 minutos em `degraded`.

**Audit footprint:** `mcp_servers.state` reflete; `failure_events.payload = { server, tool, validation_error, raw_truncated }`.

### 15.6 `mcp.budget.exceeded`

**Detecção:** algum dos limites em `MCP.md §5`:
- `total_calls > max_calls_per_session` (default 200, cap 1000)
- `total_tokens_in > max_tokens_in_per_session` (default 50k, cap 500k)
- `max_concurrent_servers` excedido (default 10, cap 30) no startup

**Recovery (per-session limits):**
1. Server transita `active` → `disconnected` na sessão atual.
2. Tools do server invisíveis até **próxima sessão** (não reconectável dentro da sessão atual).
3. Modal informativa ao user: "Server `<name>` excedeu budget; tools indisponíveis até /reset ou nova sessão."

**Recovery (`max_concurrent_servers`):**
1. Servers **excedentes** ficam `disabled` automatically (ordem: alfabética por nome, last in disabled first).
2. Erro em startup; user vê quais foram desabilitados.

**Mensagem-template:**

```
⚠ MCP server "{name}" budget exhausted

Limite atingido: {limit_kind} = {value} (cap {cap}).
Tools deste server indisponíveis até próxima sessão.

Tente: ajustar [servers.{name}] em mcp.toml
       /sessions switch  (nova sessão começa zerada)

(detalhes: mcp.budget.exceeded | /help mcp.budget)
```

**Audit footprint:** `failure_events.payload = { server, limit_kind, value, cap }`. Sinal útil para tunar caps por workflow.

### 15.7 `mcp.namespace.shadow_canonical`

**Detecção:** em `tools/list`, server declara tool com nome **reservado** pelo catálogo canônico (`MCP.md §4.2`): `read_file`, `write_file`, `edit_file`, `glob`, `grep`, `bash`, `task_*`, `memory_search`, `fetch_url`.

**Recovery:**
1. Server transita pra `error` em SessionStart; **nenhuma** tool do server é registrada (não só a conflituosa — bloqueia o server inteiro, fail-fast).
2. Sessão prossegue sem o server.
3. Erro fica visível em `agent doctor` e `/mcp show <name>`.

**Razão para fail-fast (vs. registrar as outras):** server que tenta shadow canônica é sinal de design ruim ou tentativa adversarial; melhor errar visível que silenciosamente registrar metade.

**Mensagem-template:**

```
✖ MCP server "{name}" rejeitado

Server declara tool "{tool_name}" que colide com o catálogo canônico.
Nomes reservados não podem ser sombreados via MCP.

Tente: pedir ao maintainer do server pra renomear a tool
       remover server de mcp.toml se não for crítico

(detalhes: mcp.namespace.shadow_canonical | /help mcp.namespace)
```

**Audit footprint:** `failure_events` em SessionStart; persiste enquanto config tiver o server. Não vira `mcp_servers` row (server nunca foi registrado).

### 15.8 `mcp.metadata.writes_lied`

**Detecção (modo dev):**
1. Antes de `tools/call` com declarado `_meta.agentic_cli.writes: false`: harness computa hash de tree do `cwd` declarado do server.
2. Após response: re-computa hash.
3. Mismatch → bug do server (escreveu apesar de declarar `writes: false`).

**Detecção (modo prod):** desabilitado por default (custo de hash de tree). Habilitável via `agent doctor --check-mcp-writes`.

**Recovery:**
1. Server transita `active` → `degraded` com flag persistente `writes_lied = true` em `mcp_servers`.
2. Próximas calls do mesmo tool: harness força tratamento como `writes: true` (cria checkpoint pré-call) ignorando o manifest.
3. Tool result do call original: vai pro modelo normal (já aconteceu o side effect; aviso a posteriori não desfaz).
4. UI mostra warning persistente: "Server X tem tool Y mal-comportada; checkpoints habilitados."

**Mensagem-template (uma vez por server por sessão):**

```
⚠ MCP server "{name}" comportamento inconsistente

Tool "{tool}" declarou writes:false mas modificou {cwd}.
Checkpoints automáticos habilitados pra essa tool nesta sessão.

Tente: reportar ao maintainer do server (link em /mcp show {name})
       /mcp revoke {name}    (se tool não for confiável)

(detalhes: mcp.metadata.writes_lied | /help mcp.metadata)
```

**Audit footprint:**
- `mcp_servers.last_error = "metadata.writes_lied:<tool>"`
- `failure_events.payload = { server, tool, hash_before, hash_after, files_changed_count }`
- Flag persiste cross-session: server fica em "modo paranoico" até user explicitamente fazer `/mcp trust <name>` novamente.

**Cross-ref:** `ANTI_PATTERNS.md §6.7` (anti-pattern do lado do server).

### 15.9 Audit aggregate queries

```sql
-- "Servers MCP com falha nos últimos 7d"
SELECT s.name,
       SUM(CASE WHEN fe.code LIKE 'mcp.transport%' THEN 1 ELSE 0 END) AS transport_failures,
       SUM(CASE WHEN fe.code = 'mcp.timeout' THEN 1 ELSE 0 END) AS timeouts,
       SUM(CASE WHEN fe.code = 'mcp.output.invalid' THEN 1 ELSE 0 END) AS output_failures
FROM mcp_servers s
JOIN failure_events fe ON json_extract(fe.payload, '$.server') = s.name
WHERE fe.created_at > unixepoch('now', '-7 days')
GROUP BY s.name;

-- "Tools MCP que excedem budget mais frequentemente"
SELECT json_extract(payload, '$.server') AS server,
       json_extract(payload, '$.limit_kind') AS limit_kind,
       COUNT(*) AS hits
FROM failure_events
WHERE code = 'mcp.budget.exceeded'
  AND created_at > unixepoch('now', '-30 days')
GROUP BY server, limit_kind
ORDER BY hits DESC;

-- "Servers com writes_lied flag (red flag persistente)"
SELECT name, last_error, last_connected_at
FROM mcp_servers
WHERE last_error LIKE 'metadata.writes_lied%';
```

### 15.10 O que **não** está coberto

- **Server-side bugs internos** que não violam contrato (ex: tool retorna resultado errado mas com schema válido). Fora do contrato — eval do user pega.
- **Latência crônica** (não-timeout, mas lento). Coberto por `PERFORMANCE.md §5` SLOs, não falha classificada.
- **Manifest válido mas tools mentirosas** (ex: tool "read_only_query" que faz INSERT). Cobertura parcial via §15.8 (`writes_lied`); resto exige user reportar e revogar trust.
- **MCP-over-network specifically** (SSE/HTTP) com TLS broken: cobre como `mcp.transport.broken` mas user pode querer telemetria mais fina (TLS handshake fail vs DNS fail vs cert mismatch). Deferido para v1.1.

---

## 16. Code index failures

> **Cross-refs:** subsistema em [`CODE_INDEX.md`](./CODE_INDEX.md); tools simbólicas em [`CONTRACTS.md §2.6.5c, §2.6.9`](./CONTRACTS.md). Esta seção é o catálogo operacional.

Classificação (§1 Taxonomia):

| Code | Classe | User-visible | Recovery |
|---|---|---|---|
| `index.parse_failed` | internal | não (warning em doctor) | arquivo invisível ao index; recai em `read_file` |
| `index.parse_partial` | internal | não | symbols extraídos até onde possível; flag `parse_status='partial'` |
| `index.schema_version_mismatch` | configuration | sim (em SessionStart) | migration auto; falha → `rebuild --clean` |
| `index.stale_excessive` | internal | sim (warning) | sugestão de rebuild; sessão prossegue com stale |
| `index.disk_full` | external | sim | tools simbólicas degradam; fallback para `read_file`/`grep` |
| `index.lock_timeout` | internal (transient) | sim | abort do reindex; `agent code-index check` |
| `index.unavailable` | configuration | sim (warning único) | tools simbólicas no-op; modelo cai em fallback |

7 codes. Comparado com MCP (8 codes em §15) é menor — code index é subsistema mais auto-contido.

### 16.1 `index.parse_failed`

**Detecção:** tree-sitter parse retorna erro fatal (sintaxe que o parser não consegue recuperar) em arquivo `X`.

**Recovery:**
1. `files.parse_status = 'failed'`, `parse_error` preenchido com classe (`syntax_error`, `unsupported_feature`, `parser_panic`).
2. **Nenhum** symbol/reference/import extraído desse arquivo.
3. Arquivo continua existindo no FS; só **invisível** ao index.
4. Tools simbólicas que tentam alvejar o arquivo retornam `file.parse_failed`; modelo recebe e decide (recai em `read_file`).
5. Próximo edit no arquivo dispara re-parse incremental (`CODE_INDEX.md §3.2`); se sintaxe foi corrigida, status volta para `ok`.

**Mensagem ao usuário:** silenciosa por default. `agent doctor` mostra contagem de `parse_failed`; > 5% do repo dispara warning.

**Audit footprint:** `failure_events.payload = { file, parse_error_class, line, col }`. Útil pra: "quais arquivos do repo o parser não digere?".

### 16.2 `index.parse_partial`

**Detecção:** tree-sitter retorna CST mas com nodos `ERROR` ou `MISSING` — sintaxe parcialmente válida.

**Recovery:**
1. `files.parse_status = 'partial'`.
2. Symbols extraídos das **regiões válidas** apenas; regiões com erro são puladas.
3. Tools simbólicas funcionam normalmente, mas com warning implícito (resultado pode estar incompleto).
4. `outline_file` retorna campo extra `parse_warnings: number` para sinalizar.

**Mensagem:** silenciosa. `outline_file` carrega o sinal pro modelo via `parse_warnings`.

**Audit:** `failure_events.payload = { file, error_node_count, recovered_symbols }`. Sinal para tunar grammar selection (linguagem com `partial` rate alto pode estar sub-suportada).

### 16.3 `index.schema_version_mismatch`

**Detecção:** em `SessionStart`, `index_meta.value WHERE key='index_schema_version'` ≠ versão atual do binary.

**Recovery:**
1. **Migration auto-tentada** com pipeline declarado em `migrations/code_index/`:
   - 1.x → 1.y: ALTER TABLE adicionando colunas com default
   - 1.x → 2.x: rebuild parcial; symbols/references mantidos, schemas dinâmicos refeitos
2. Migration succeeded → sessão prossegue normal.
3. Migration failed → `index.unavailable` é o estado de fallback; user prompt sugerindo `agent code-index rebuild --clean`.

**Mensagem-template (em failure de migration):**

```
⚠ Code index incompatível com esta versão

Schema da DB local é v{old}, binary atual usa v{new}.
Migration automática falhou: {error_class}.

Tente: agent code-index rebuild --clean
       (~15-30s; reindex from scratch)

Tools simbólicas (read_symbol, find_references, ...) ficam
indisponíveis até o rebuild. Sessão prossegue com fallback
a read_file/grep.

(detalhes: index.schema_version_mismatch | /help index.schema)
```

**Audit:** `failure_events.payload = { old_version, new_version, migration_step_failed }`.

### 16.4 `index.stale_excessive`

**Detecção:** em `SessionStart`, query do §3.5:

```sql
SELECT COUNT(*) FROM files
WHERE last_modified_at > indexed_at + 60;  -- arquivos modificados há > 60s sem reindex
```

Se contagem > 5% do total: stale excessivo.

**Recovery:**
1. **Não bloqueia** sessão.
2. Warning em UI: "Code index stale em N% dos arquivos. Tools simbólicas podem retornar info desatualizada."
3. Sugestão: `agent code-index rebuild --since <last_session_end>` (incremental restrito ao diff).
4. Tools simbólicas continuam funcionando, mas com flag `index_stale: true` em metadata do tool result.

**Mensagem:**

```
⚠ Code index parcialmente stale

{N} arquivos foram modificados fora do harness e ainda não foram
reindexados (~{percent}% do repo).

Tente: agent code-index rebuild --since {last_session}
       (incremental; ~5-15s)

Tools simbólicas ainda funcionam, com warning de staleness.

(detalhes: index.stale_excessive | /help index.stale)
```

**Audit:** `failure_events.payload = { stale_count, total_files, threshold_pct }`. Trend útil: "user trabalhando em outro editor (vim/vscode) entre sessões com FS watcher off?".

### 16.5 `index.disk_full`

**Detecção:** SQLite write retorna `SQLITE_FULL` ou `SQLITE_IOERR_WRITE`.

**Recovery:**
1. Re-index abortado in-flight; estado SQLite preservado consistente (transação rolled back).
2. Subsistema transita para `index.unavailable` em-memory; tools simbólicas no-op até fim da sessão.
3. Próxima sessão tenta reconectar; se disco ainda cheio, mantém unavailable.

**Mensagem:**

```
✖ Code index sem espaço em disco

Falha ao escrever em ~/.local/share/agent/code_index.db.
Espaço disponível: {free_mb} MB.

Tente: agent gc                   (limpar audit/sessions antigas)
       agent code-index rebuild --clean   (após liberar espaço)

(detalhes: index.disk_full | /help index.disk_full)
```

**Audit:** `failure_events.payload = { db_path, free_mb, attempted_write_kb }`. `redaction`: nenhuma (paths já são `~/...`).

### 16.6 `index.lock_timeout`

**Detecção:** SQLite write/read aguardando lock por > 30s (SQLITE_BUSY repetido).

**Recovery:**
1. Operação atual aborta; estado preservado.
2. Possível causa: outra instância do `agent` está reindexando o mesmo `cwd`.
3. Tool simbólica que disparou retorna `index.lock_timeout`; modelo decide retry (provavelmente passa após alguns segundos) ou fallback.
4. `agent code-index check` é sugestão: detecta locks órfãos (de processos crashed) e tenta recovery.

**Mensagem (apenas se persistir > 60s):**

```
⚠ Code index com lock prolongado

Tools simbólicas estão aguardando ~{seconds}s.
Possível causa: outra instância do agent reindexando.

Tente: agent code-index check   (detecta locks órfãos)
       /sessions list           (ver outras sessões ativas)

(detalhes: index.lock_timeout | /help index.lock)
```

**Audit:** `failure_events.payload = { operation, wait_ms, holding_pid_if_known }`.

### 16.7 `index.unavailable`

**Detecção:** `code_index.db` ausente, ou em estado `error` cumulativo.

**Recovery:**
1. **Modo esperado** em primeira sessão de um repo (initial scan ainda não rodou).
2. Tools simbólicas (`read_symbol`, etc.) retornam `index.unavailable` no tool_result.
3. Modelo recebe **fallback hint** no error: "use `read_file`/`grep` enquanto index não está pronto".
4. Initial scan dispara em background; `index_status()` informa progress.
5. Quando scan termina: tools simbólicas viram disponíveis silenciosamente (sem interrupção da sessão).

**Mensagem (apenas em primeiro encontro da sessão):**

```
ℹ Code index aquecendo

Initial scan em progresso (~{remaining_files} arquivos).
Estimativa: ~{seconds}s.

Tools simbólicas (read_symbol, find_references, outline_file,
code_graph) ficam indisponíveis até conclusão. Use read_file/grep
neste meio tempo.

(detalhes: index.unavailable | /help index.unavailable)
```

**Audit:** `failure_events.payload = { reason: 'initial_scan'|'corrupted'|'error', files_indexed_so_far }`.

### 16.8 Audit aggregate queries

```sql
-- "Saúde do code index nos últimos 30d"
SELECT code, COUNT(*) AS occurrences
FROM failure_events
WHERE code LIKE 'index.%'
  AND created_at > unixepoch('now', '-30 days')
GROUP BY code
ORDER BY occurrences DESC;

-- "Arquivos com parse failure recorrente (eval candidates)"
SELECT json_extract(payload, '$.file') AS file,
       COUNT(*) AS failures
FROM failure_events
WHERE code = 'index.parse_failed'
  AND created_at > unixepoch('now', '-30 days')
GROUP BY file
HAVING failures > 3
ORDER BY failures DESC;

-- "Sessões com index unavailable + tools simbólicas tentadas"
SELECT s.id, s.created_at,
       COUNT(DISTINCT tc.id) AS symbolic_tool_attempts
FROM sessions s
JOIN failure_events fe ON fe.session_id = s.id AND fe.code = 'index.unavailable'
LEFT JOIN tool_calls tc ON tc.session_id = s.id
  AND tc.tool_name IN ('read_symbol', 'find_references', 'outline_file', 'code_graph')
GROUP BY s.id;
```

### 16.9 O que **não** está coberto

- **Symbol resolution falha cross-language.** TS chamando WASM Rust = ref `target_symbol_id IS NULL`. Não é "falha" classificada — é limite documentado em `CODE_INDEX.md §12`.
- **Polymorphism resolution.** `obj.method()` ambíguo retorna lista; modelo decide. Sem código de erro.
- **Performance regression em initial scan.** Cresce > 30% vs baseline → coberto por `PERFORMANCE.md §5` (SLO violation), não por failure code.
- **Index corruption parcial silenciosa** (queries retornam stale refs sem warning). `agent code-index check` detecta sob demanda; sem failure event automático.
- **Race entre dois agents reindexando paralelamente** com `inotify` agressivo. Aceitável: `index.lock_timeout` cobre o sintoma. Documentado em `CODE_INDEX.md §7.3`.

---

## 17. Recovery hierarchy (decisão por classe)

Quando uma falha ocorre, a árvore de decisão é:

```
falha detectada
  ├─ é transiente e retry é seguro?
  │    └─ retry transparente (sem aviso ao user)
  ├─ é transiente mas pode ter side effect?
  │    └─ retry com aviso (warning em UI)
  ├─ é determinística mas recoverable?
  │    └─ recovery automatic com mensagem clara
  ├─ pode-se preservar estado e continuar parcial?
  │    └─ continua com warning + audit
  ├─ requer ação humana?
  │    └─ pause, modal com instruções acionáveis
  └─ é catastrófica?
       └─ error_fatal, dump de state, sessão preservada offline
```

---

## 18. Mensagens ao usuário (templates)

Toda mensagem segue:

```
<emoji> <título curto>

<o que aconteceu, 1-2 linhas>

<o que tentar — comando ou ação concreta>

(detalhes: <código> | /help <código>)
```

Exemplo:

```
⚠ Provider lento (3 timeouts)

Anthropic API não respondeu em 60s × 3 tentativas.
Continuando com aviso. Próximo step pode degradar.

Tente: /model haiku   (modelo mais rápido)
       /pause          (continuar depois)

(detalhes: provider.timeout.streaming | /help provider.timeout)
```

Templates ficam em `~/.config/agent/messages/<lang>/<code>.md`. Versionados, traduzíveis.

---

## 19. Audit trail (o que **sempre** é registrado)

Tabela `failure_events`:

```sql
failure_events(
  id TEXT PRIMARY KEY,
  session_id TEXT,
  step_id TEXT,
  code TEXT NOT NULL,           -- e.g. "provider.timeout.streaming"
  classe TEXT NOT NULL,          -- e.g. "external"
  recovery_action TEXT NOT NULL, -- e.g. "retried_3x", "fallback_to_model_X", "fatal"
  user_visible BOOLEAN NOT NULL,
  payload JSONB,                 -- detalhes específicos
  created_at INTEGER NOT NULL
);
```

Toda falha **classificada** vai pra `failure_events`. Sem exceção. Auditoria mensal escaneia trends.

---

## 20. Como se preparar pra falhas que **não** estão aqui

Princípio: **catálogo é vivo**, não exaustivo. Procedimento quando uma nova falha aparece em produção:

1. Reproduzir em test/eval
2. Atribuir código (`<classe>.<subtipo>.<detalhe>`)
3. Documentar aqui (PR no doc)
4. Implementar recovery
5. Adicionar regression test
6. Atualizar `failure_events` se schema precisa mudar

Mensagem ao usuário **sempre antes** de implementação — força clareza.

---

## 21. Insight final

Catálogo de falhas é o doc que ninguém lê quando tudo funciona — e o único que importa quando para de funcionar.

Não é exaustivo (impossível). É o **piso**: classes principais, recovery padrão, audit consistente. O que não está aqui ainda vai chegar; quando chegar, ganha entrada.

A regra é simples: **se uma falha não tem código, não tem audit, e não tem mensagem-template — ela vai te morder em produção, em volume, em horário ruim.**
