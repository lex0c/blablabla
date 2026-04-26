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

## 15. Recovery hierarchy (decisão por classe)

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

## 16. Mensagens ao usuário (templates)

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

## 17. Audit trail (o que **sempre** é registrado)

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

## 18. Como se preparar pra falhas que **não** estão aqui

Princípio: **catálogo é vivo**, não exaustivo. Procedimento quando uma nova falha aparece em produção:

1. Reproduzir em test/eval
2. Atribuir código (`<classe>.<subtipo>.<detalhe>`)
3. Documentar aqui (PR no doc)
4. Implementar recovery
5. Adicionar regression test
6. Atualizar `failure_events` se schema precisa mudar

Mensagem ao usuário **sempre antes** de implementação — força clareza.

---

## 19. Insight final

Catálogo de falhas é o doc que ninguém lê quando tudo funciona — e o único que importa quando para de funcionar.

Não é exaustivo (impossível). É o **piso**: classes principais, recovery padrão, audit consistente. O que não está aqui ainda vai chegar; quando chegar, ganha entrada.

A regra é simples: **se uma falha não tem código, não tem audit, e não tem mensagem-template — ela vai te morder em produção, em volume, em horário ruim.**
