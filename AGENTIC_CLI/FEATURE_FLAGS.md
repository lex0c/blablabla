# FEATURE_FLAGS

Subsistema **mínimo** de governance de flags no `AGENTIC_CLI`. Resolve o gap operacional onde ~25 flags ad hoc se acumularam sem inventário, sem lifecycle, sem audit. **Não** é uma plataforma de feature management — é o **mínimo necessário** pra que flags não viram folclore.

Premissa raiz: *meça duas vezes, corte uma*. Flags são o ponto onde o projeto mais frequentemente "corta sem medir" — adiciona toggle pra postergar decisão, esquece, e o toggle vira default acidental. Este doc é a especificação de pontos de medição mínimos: **inventário, lifecycle, audit, discovery**.

---

## 0. Princípios (não-negociáveis)

1. **Flag é último recurso, não default.** Mudança direta de código vence flag toda vez que possível. Princípio 12 do `AGENTIC_CLI.md` ("sem cargo cult") aplica aqui — feature flag service não é o que falta.
2. **Inventário canônico ou não existe.** Flag não listada em `/flags` ou `agent --list-flags` é **bug**, não comportamento. Source of truth é o code (registry); este doc é o **lifecycle e governance**.
3. **Lifecycle declarado.** Cada flag declara `stage`: `experimental`, `staged`, `stable`, `deprecated`. Sem stage = flag indisciplinada.
4. **Audit captura o que estava ON.** Sessão grava flags ativas; replay sem essa info diverge silenciosamente.
5. **TTL em experimental.** Flag `experimental` por > 6 meses sem promoção é débito — graduar ou remover, não deixar derivar.
6. **Sem flag-as-config.** Se algo deveria viver em `~/.config/agent/*.toml` (estável, raramente mexido), não vire flag. Inverso também: se é toggle de feature em desenvolvimento, não polui config estável.
7. **Sem rollout percentual.** CLI single-user; "10% dos sessions" não faz sentido. Flag é binário (off/on) por user, sessão, ou cwd.
8. **Cleanup tem dono.** Toda flag `deprecated` tem `cleanup_target` (data ou release); sem dono é candidata a remoção imediata.

---

## 1. Categorias canônicas

Quatro mecanismos de toggle existem na spec; **cada um tem use case distinto**. Confundir é anti-pattern (`§6`).

### 1.1 CLI flags

Flags passadas no invocation: `agent --strict --plan "prompt"`.

**Quando usar:**
- Toggle por sessão, sem persistência.
- Modo de execução (strict, plan, no-pipeline).
- Override pontual de policy de segurança (`--allow-large-fetch`).
- Output format (`--json`).

**Não usar para:**
- Configuração estável (vai em `config.toml`).
- Toggle dentro de uma sessão (vai em slash command).
- Estado de subsystem (vai em DB).

**Exemplo:**
```
agent --strict --plan "refactor src/auth"
```

### 1.2 Config flags (TOML)

Booleans em `~/.config/agent/*.toml`, `.agent/*.toml`. Lidos em `SessionStart`.

**Quando usar:**
- Configuração persistente per-user ou per-project.
- Defaults que raramente mudam.
- Setup-time decisions (sandbox on/off, watcher on/off).

**Não usar para:**
- Toggles de feature em desenvolvimento (vai em CLI flag enquanto experimental).
- Override per-sessão (vai em CLI flag).

**Exemplo:**
```toml
[code_index]
watch = false           # config flag
[generation.test]
disabled = true         # config flag (per-stage default)
```

### 1.3 Slash command toggles

Mutação runtime mid-sessão: `/strict on`, `/thinking on`.

**Quando usar:**
- Toggle precisa mudar **durante** a sessão sem reinicializar.
- Modelo ou usuário decide ajustar comportamento dinamicamente.
- Override temporário pro próximo turn.

**Não usar para:**
- Mudanças que requerem reinicializar subsystems (memory loading, MCP handshake).
- Decisões que afetam audit hash chain (precisariam invalidar prompt_hash).

**Exemplo:**
```
/strict on              # próximo edit roda gates strict
/thinking off           # esconde extended thinking dali pra frente
```

### 1.4 Subsystem state

Flag implícita via estado de subsystem. **Não** é flag user-controlled; é resultado de operação (trust prompt, parse failure, etc).

**Exemplos:**
- `mcp_servers.state` (`trusted`/`active`/`degraded`/...) — funciona como flag por server
- `files.parse_status` (`ok`/`partial`/`failed`/`skipped`) — flag por arquivo
- Profile (`autonomous`/`orchestrated`/`hybrid`) — flag por sessão (selected at start)

**Quando aparece como flag:**
Em audit/replay, esses estados precisam ser capturados como contexto da sessão (mesma razão que `prompt_hash`).

---

## 2. Lifecycle canônico

Toda flag passa por estágios bem-definidos. Transições requerem critério, não vontade.

```
[introduced] → experimental → staged → stable → deprecated → removed
                    ↓             ↓
                 (TTL 6m)     (eval gate)
```

### 2.1 `experimental`

- Default **off**, sempre.
- Opt-in via CLI flag ou config.
- Documentado em `FEATURE_FLAGS.md §3` com `stage: experimental`.
- TTL de **6 meses** desde introdução; após isso, decisão forçada: graduar (`staged`) ou remover.
- **Não** há promessa de estabilidade — pode mudar shape entre releases.
- Coberto por eval (corpus dedicado se modificar comportamento de modelo).

**Critério para graduar:** ≥ 100 sessões em `feature_flags_active` com flag ativa, sem regressão em eval.

### 2.2 `staged`

- Default **off**, mas considerado pronto para graduar.
- Documentação completa, eval estável.
- Período de observação: **3 meses** mínimo.
- Pode ser promovido a `stable` antes se eval mostra adoção sem regressão.

**Critério para promover:** ≥ 1000 sessões com flag ativa, eval pass-rate ≥ 95%, sem failure_event recorrente atribuído a ela.

### 2.3 `stable`

- Default **on** (na maioria dos casos).
- Opt-out via flag/config se preciso.
- Audit captura quando opt-out aparece (sinal de friction).
- Lifecycle não-encerrado: stable é estado terminal por default.

**Promoção a default-on:** mudança em `defaults built-in`; flag continua existindo mas com inversão de polaridade (passa a ser `--disable-X` em vez de `--enable-X`).

### 2.4 `deprecated`

- Default **off**, e usar dispara warning.
- Documentação aponta substituto.
- `cleanup_target`: data ou release explícito.
- Em `agent --list-flags`: aparece com `[deprecated since X, remove in Y]`.

**Critério para remover:** após `cleanup_target` + 1 release de buffer.

### 2.5 Transições proibidas

| De → Para | Motivo |
|---|---|
| `experimental` → `stable` direto | Pula `staged`; sem janela de observação. Anti-pattern. |
| `stable` → `experimental` | Regressão de promessa; quebra trust em estabilidade. Em vez disso: `deprecated` + nova flag `experimental`. |
| Qualquer → `removed` sem `deprecated` | Quebra usuários sem aviso. Exceção: `experimental` que excedeu TTL pode ser removida diretamente. |

---

## 3. Inventário (representativo, não exaustivo)

> **Source of truth:** registry no código + `agent --list-flags --json`. Esta tabela mostra flags representativas pra cada categoria/stage; lista completa em runtime.

### 3.1 CLI flags representativas

| Flag | Category | Stage | Default | Doc |
|---|---|---|---|---|
| `--strict` | mode | stable | off | `CODE_GENERATION.md §2.2` |
| `--plan` | mode | stable | off | `AGENTIC_CLI.md §2.1` |
| `--no-pipeline` | mode (escape hatch) | stable | off | `CODE_GENERATION.md §2.3` |
| `--auto-approve-mcp <list>` | CI bypass | stable | (none) | `MCP.md §1.5` |
| `--budget-enforce` | mode | stable | off | `PERFORMANCE.md §5.4` |
| `--snapshot-index` | dev | experimental | off | `CODE_INDEX.md §4.2` |
| `--allow-large-fetch` | policy override | stable | off | `SECURITY_GUIDELINE.md §9.1.4` |
| `--show-thinking` | UI | stable | off | `UI.md §3.2` |
| `--dangerous` | escape hatch | **deprecated** | off | sem expiry — anti-pattern; remover em v1.1 |
| `--check-mcp-writes` | dev | experimental | off | `FAILURE_MODES.md §15.8` |

### 3.2 Config flags representativas

| Path TOML | Category | Stage | Default | Doc |
|---|---|---|---|---|
| `[code_index] watch` | subsystem | staged | false | `CODE_INDEX.md §3.3` |
| `[providers.anthropic] extended_cache` | provider | stable | false | `PROVIDERS.md` |
| `[generation.test] disabled` | pipeline | stable | true | `CODE_GENERATION.md §3.3` |
| `[servers.<name>] sandbox` | MCP | stable | false | `MCP.md §2.3` |
| `[hooks] fail_closed_on_dispatcher_down` | hooks | staged | false | `CONTRACTS.md §10` |

### 3.3 Slash toggles representativos

| Slash | Stage | Effect |
|---|---|---|
| `/strict on\|off` | stable | toggle modo strict mid-session |
| `/thinking on\|off` | stable | mostra/esconde extended thinking |
| `/mcp revoke <name>` | stable | revoga trust de server |

### 3.4 Subsystem state (não-toggles, mas auditados como flags)

| Estado | Onde | Audit |
|---|---|---|
| `profile` (autonomous/orchestrated/hybrid) | `sessions.profile` | sim, per session |
| `mcp_servers.state` | `mcp_servers.state` | sim, per server |
| `files.parse_status` | `files.parse_status` | sim, per file |

---

## 4. Storage e audit

### 4.1 Tabela `feature_flags_active`

Captura flags ativas per sessão. **Não** é a definição da flag (essa vive no registry de código); só registro de **o que estava ON**.

> **Schema completo em** [`AUDIT.md §1.6`](./AUDIT.md). Resumo aqui:

```sql
CREATE TABLE feature_flags_active (
  session_id TEXT NOT NULL,
  flag_name TEXT NOT NULL,
  flag_kind TEXT NOT NULL,        -- 'cli' | 'config' | 'slash' | 'state'
  flag_value TEXT NOT NULL,       -- 'on' | 'off' | string serializado
  set_at INTEGER NOT NULL,        -- unix ts
  set_by TEXT NOT NULL,           -- 'user' | 'config' | 'slash:/strict' | 'init'
  PRIMARY KEY (session_id, flag_name),
  FOREIGN KEY (session_id) REFERENCES sessions(id)
);
```

Flag mid-session change (via slash): `UPDATE` permitido, com new `set_at`. Histórico de mudanças vai em `audit_timeline` como events `feature_flag_changed`.

### 4.2 Queries canônicas

```sql
-- "Sessões com --no-pipeline ativo nos últimos 30d (red flag)"
SELECT session_id, set_at, set_by
FROM feature_flags_active
WHERE flag_name = 'no_pipeline' AND flag_value = 'on'
  AND set_at > unixepoch('now', '-30 days');

-- "Regressão correlata a flag específica"
SELECT pv.hash, ffa.flag_name,
       AVG(CASE WHEN fe.classe = 'quality_regression' THEN 1.0 ELSE 0.0 END) AS regression_rate
FROM messages m
JOIN prompt_versions pv ON m.prompt_hash = pv.hash
JOIN feature_flags_active ffa ON m.session_id = ffa.session_id
LEFT JOIN failure_events fe ON fe.session_id = m.session_id
WHERE ffa.flag_value = 'on'
GROUP BY pv.hash, ffa.flag_name
HAVING regression_rate > 0.05;

-- "Flag adoption rate (pra promoção experimental → staged)"
SELECT flag_name, COUNT(DISTINCT session_id) AS sessions_with_flag_on
FROM feature_flags_active
WHERE flag_value = 'on'
  AND set_at > unixepoch('now', '-90 days')
GROUP BY flag_name
ORDER BY sessions_with_flag_on DESC;
```

### 4.3 Append + update

`feature_flags_active` é **exceção** a `AUDIT.md §4.1` (regra geral append-only): UPDATE permitido em `flag_value` e `set_at` quando flag muda mid-session. Histórico vai em `audit_timeline`. Razão: o que importa pra forensics é o estado **final** da sessão, não cada toggle intermediário (que fica no timeline).

---

## 5. Discovery

### 5.1 CLI

```
agent --list-flags                       # human-friendly table
agent --list-flags --json                # NDJSON pra scripting
agent --list-flags --stage experimental  # filtro
agent --list-flags --deprecated          # only deprecated
agent --list-flags --active              # só as ativas nesta sessão (cli + config + slash)
```

Output (table — usa `<Table>` de `UI.md §3.1`):

```
Flag                       Kind     Stage         Default  Active?  Doc
─────────────────────────  ───────  ────────────  ───────  ───────  ─────────────────────
--strict                   cli      stable        off      yes      CODE_GENERATION.md §2.2
--plan                     cli      stable        off      no       AGENTIC_CLI.md §2.1
--snapshot-index           cli      experimental  off      no       CODE_INDEX.md §4.2
[code_index] watch         config   staged        false    no       CODE_INDEX.md §3.3
/strict                    slash    stable        —        yes      UI.md §5.3
                                                                    [10 more flags]
```

### 5.2 Slash command

```
/flags                       # tabela de flags ativas
/flags all                   # tabela completa
/flags show <name>           # detalhe de flag específica (stage, doc, history)
/flags experimental          # filtro
```

### 5.3 Doctor integration

`agent doctor` (`AGENTIC_CLI.md §2.1`) inclui section flags:

```
Feature flags:
  ✓ 14 flags em stable (default-set ou opt-in com TTL ok)
  ⚠ 3 flags em experimental por > 6 meses (cleanup overdue)
       --snapshot-index    introduced 2025-09 (~7m)
       --check-mcp-writes  introduced 2025-08 (~8m)
  ✗ 1 flag deprecated sem cleanup_target
       --dangerous         no expiry — anti-pattern
```

---

## 6. Anti-patterns

> **Cross-ref:** [`ANTI_PATTERNS.md §7`](./ANTI_PATTERNS.md) — esta seção lista; lá tem motivo + substituição + gatilho de reconsideração.

| Anti-pattern | Sintoma | Mitigação |
|---|---|---|
| **Forever-flag** | flag em `experimental` > 6m sem promoção/remoção | TTL hard, doctor warning, force decision |
| **Flag-as-config** | toggle estável virou CLI flag em vez de config | revisão na promoção `staged` → `stable`: vai pra config se default mudou |
| **Flag-as-shim** | flag escondendo decisão postergada ("decide depois") | review em `stable`: ainda escondendo decisão? Se sim, decidir e remover |
| **Flag duplicada** | mesma intent em CLI + config + slash | reconciliação: cada toggle tem **um** mecanismo canônico |
| **Flag não-auditada** | flag muda comportamento mas não está em `feature_flags_active` | checagem em CI: flag registry vs `feature_flags_active` schema |
| **Flag sem dono** | introduzida; ninguém mantém; ninguém remove | registry exige `owner` field |
| **Default invisível** | flag default-on que ninguém sabe que existe | `agent doctor` lista flags non-trivial defaults |
| **Flag bypassando audit** | `--dangerous` que silencia segurança sem registrar | flags de override **sempre** geram `failure_event` informacional + são gravadas com peso visual no recap |

---

## 7. Eval integration

Flag é **dimensão** de eval junto com `prompt_hash` (`AUDIT.md §1.3`):

```sql
-- "Performance comparativa: --strict on vs off"
SELECT
  CASE WHEN ffa.flag_value = 'on' THEN 'strict_on' ELSE 'strict_off' END AS variant,
  AVG(t.duration_ms) AS avg_duration,
  AVG(CASE WHEN t.status = 'ok' THEN 1.0 ELSE 0.0 END) AS pass_rate,
  AVG(json_extract(t.pipeline_result, '$.test.passed')) AS avg_tests_passing
FROM tool_calls t
JOIN messages m ON m.id = t.message_id
LEFT JOIN feature_flags_active ffa
  ON ffa.session_id = m.session_id AND ffa.flag_name = 'strict'
WHERE t.tool_name IN ('write_file', 'edit_file')
GROUP BY variant;
```

`TOKEN_TUNING.md §13.4` (system prompt regression eval) ganha cláusula extra: corpus precisa **declarar** quais flags são exigidas. Eval rodado com flag set diferente do declarado → falha de configuração, não de qualidade.

```yaml
- id: refactor-strict-001
  required_flags:
    - strict: on
  prompt: "..."
  asserts: [...]
```

---

## 8. Limites declarados (v1)

- **Sem rollout percentual.** CLI single-user; "10% dos sessions" não tem semântica útil.
- **Sem flag service externa.** Sem LaunchDarkly, GrowthBook, ConfigCat. Princípio 6 (`AGENTIC_CLI.md`): local-first, sem daemon obrigatório.
- **Sem A/B testing per-user automatic.** Comparação cross-flag é eval-driven (`§7`), não product-grade A/B.
- **Sem flag inheritance entre sessões.** Cada sessão começa com defaults + flags explícitas; sem "modelo aprendeu que você gosta de strict".
- **Sem flag scope per-cwd.** Flag é per-session, per-config, ou per-user. Querer "flag X só em cwd Y" é ruído desnecessário; usuário controla via `.agent/config.toml` per-projeto.
- **Sem versionamento dentro de flag.** Flag é binária ou enum estável; "v1 vs v2 desta flag" significa duas flags diferentes.

---

## 9. Como evoluir esta spec

Adicionar flag nova requer PR cobrindo:

1. **Nome** (snake_case CLI ou TOML path).
2. **Category** (cli/config/slash/state).
3. **Stage** inicial (sempre `experimental` para flags novas).
4. **Default**.
5. **Owner** (quem mantém).
6. **Doc reference** (link pra spec section).
7. **Cleanup criterion** (gate pra graduar OU remover).
8. **Eval coverage** (corpus item ou justificativa de não-precisar).

Sem (1)-(8): PR rejeitado. Flag entrar sem governance é exatamente o que este doc evita.

Renumerar/promover requer:
- `experimental` → `staged`: ≥ 100 sessões em audit, eval estável, owner ativo.
- `staged` → `stable`: ≥ 1000 sessões, ≥ 3 meses, sem regression atribuída.
- `stable` → `deprecated`: substituto declarado + cleanup_target.
- `deprecated` → `removed`: cleanup_target + 1 release.

Cada transição é PR explícito. Audit do PR: incluir métricas de adoção da query `§4.2`.

---

## 10. Insight final

Feature flag platform é overengineering em 95% dos casos. Mas **inventário e lifecycle de flags** são gaps reais quando o projeto cresce — o que acontece sem este doc:

- Flags acumulam sem cleanup.
- Audit incompleto (regressão não-atribuível).
- Discovery ruim (usuário não sabe o que existe).
- Cross-implementação divergente.

Este doc é o **mínimo** que resolve os 4 pontos sem virar SaaS de feature management. Spec sem flag governance: ad hoc. Com este doc: **flags como dado, não como `if`** (princípio 8 do `AGENTIC_CLI.md` aplicado).

Inversão consistente: harness gerencia o estado das flags (registry, audit, lifecycle); modelo só vê **comportamento**. Modelo nunca pergunta "está em strict?", o pipeline simplesmente roda strict. Sem essa fronteira, flags viram meta-cognição — exatamente o que `CONTRACTS.md §2.6.8` (princípio "meta-cognição não é tool") rejeita.
