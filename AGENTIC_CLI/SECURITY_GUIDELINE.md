# SECURITY_GUIDELINE

Diretriz de segurança **do `AGENTIC_CLI`** — o agente como produto, não como auditor.

> **Escopo importante:** este doc é sobre proteger o agente, seu usuário e o ambiente de execução. **Não é** sobre o playbook `security-audit` (que ataca código de terceiro pra encontrar bugs — ver `PLAYBOOKS.md` §3). Auditar com o agente ≠ proteger o agente.

Premissa raiz aplicada: *meça duas vezes, corte uma*. Em segurança vira: **assuma hostil até prova em contrário, com defesas em camadas e fail-closed por default**.

---

## 0. Princípios (não-negociáveis)

1. **Fail closed por default.** Em dúvida, nega. Em ambiguidade, bloqueia. Em policy ausente, recusa.
2. **Defesa em camadas.** Nenhum controle isolado é confiável. Trust prompt + sandbox + permission + audit + validator.
3. **Trust boundaries explícitas.** Cada interface entre componentes declara quem confia em quem e por quê.
4. **Inputs não-confiáveis até prova em contrário.** `CLAUDE.md`, `tool output`, `MCP manifests`, `memory body` — tudo é não-confiável até classificação.
5. **Secrets nunca em logs, traces, memórias, ou prompt.** Detecção heurística obrigatória; redaction antes de persistir.
6. **Auditabilidade total.** Toda decisão de segurança grava em `approvals` ou `failure_events`. Sem decisão silenciosa.
7. **Limites declarados, não escondidos.** O que o agente **não defende** é tão importante quanto o que defende.
8. **Princípio do menor privilégio.** Tool com mínimo necessário; subagent com escopo mínimo; provider com permissão mínima.
9. **Reversibilidade onde possível.** Side effect persistente = checkpoint + audit. Ataque parcial é recuperável.
10. **Update + signing first-class.** Binário não-assinado é vetor; release sem hash é cargo cult.

---

## 1. Threat model (STRIDE aplicado)

Análise sistemática por categoria, com mitigação primária.

### 1.1 Spoofing (impersonação)

| Ameaça | Mitigação |
|---|---|
| Atacante substitui `agent` binary local | Update mechanism com signing (Sigstore/cosign); `agent --version --verify` checa assinatura |
| Atacante substitui `~/.config/agent/hooks/audit.sh` | Hook scripts de paths confiados; trust prompt em primeira execução; logging em `hook_runs` com hash |
| MCP server hostil se passa por confiável | Hash de manifest gravado; trust prompt em primeiro contact; tools invisíveis ao modelo se manifest mudou |
| `CLAUDE.md` em repo terceiro se passa por instrução do usuário | Trust prompt por dir; tratamento como input não-confiável; injection scanner |

### 1.2 Tampering (alteração)

| Ameaça | Mitigação |
|---|---|
| Tool retorna output adulterado pra prompt-injetar modelo | Output sanitization; ANSI strip; injection heurística com flag visível |
| Memória adulterada entre sessões | `memory_events` audit log; hash do source da inferência; `trust: untrusted` flag |
| **PR malicioso adiciona memória shared** que prompt-injeta agentes futuros | **PR review humano** (gate primário); scanner adicional em `/memory promote shared` (path traversal, secrets, injection patterns); trust prompt re-fires em mudança de hash de `.agent/memory/shared/` |
| **PR malicioso adiciona playbook/orchestrator/hook shared** | Mesma defesa: PR review + hash check de `.agent/playbooks/`, `.agent/orchestrators/`, `.agent/hooks.toml` em trust prompt agregado |
| Permission policy adulterada | Hierarquia enterprise → user → project com `locked`; load-fail closed |
| SQLite corruption por adversário com FS access | `PRAGMA integrity_check` em SessionStart; sessão refusa resume se corrompido |
| Hook script trocado entre sessões | Hash do script gravado; warning se mudou; re-trust opcional |

### 1.3 Repudiation (negação)

| Ameaça | Mitigação |
|---|---|
| User nega ter autorizado tool destrutiva | `approvals` table com timestamp + decision + decided_by |
| Modelo nega ter feito refactor | Step persisted; checkpoint criado; replay possível |
| Hook nega ter bloqueado | `hook_runs` com exit_code + duration + message |

### 1.4 Information disclosure

| Ameaça | Mitigação |
|---|---|
| Secrets vazam em logs/traces | Redaction heurística (AWS keys, GitHub tokens, JWT, etc) antes de persistir |
| Path absoluto com username vaza em recap/PR | Anonimização (`/home/lex/...` → `~/...`) em renderers |
| Memória user-scope vaza em projeto diferente | Scope isolation; nunca cross-project sem opt-in |
| Prompt cache com PII vaza pra próxima sessão | Cache TTL 5min + scope por sessão |
| Tool output com secret vai pro contexto | Sanitization layer + redaction; user pode ver original via `--include-tool-output` |
| Telemetry com PII exporta pra OTEL | Scrubbing ativo em attrs antes de emit; opt-in para export |

### 1.5 Denial of service

| Ameaça | Mitigação |
|---|---|
| Loop infinito do modelo (tool call repetido) | `maxSteps` budget + `maxToolErrors` consecutive |
| Cost runaway por loop degenerado | `maxCostUsd` budget hard cap; sem opt-out |
| Hook trava harness | Timeout 5s; SIGKILL após grace |
| Tool com side effect pesado (rm -rf) | Permission deny por padrão; sandbox `bwrap` opcional/obrigatório |
| Memory bloat (índice cresce sem fim) | Hard cap 200 linhas; expires default 90d em project |
| Disk full por checkpoints | Cleanup automatic em `Stop`; hard limit configurável |
| Provider rate limit cascateado | Backoff exponencial; fallback model em hybrid |

### 1.6 Elevation of privilege

| Ameaça | Mitigação |
|---|---|
| Tool `bash` escapa policy via shell expansion | Pattern matching em prefix + glob (não regex); deny rules pesadas; sandbox real |
| Modelo manipula args pra burlar permission | Schema validation rígida; allow_paths em `write_file` deny por default |
| Subagent acessa state do pai além do permitido | Contexto isolado; communication via SQLite read-only do pai |
| Hook em path de usuário ganha acesso de enterprise | Hierarquia respeitada; `locked` em enterprise não-overridable |
| MCP server obtém acesso a tools do agente | Tools MCP no registry com policy aplicada igual local; trust prompt |

---

## 2. Trust boundaries (quem confia em quem)

Mapa explícito.

### 2.1 Níveis de confiança

```
[Enterprise config]              ← maior confiança (locked rules)
        ↓
[User config]                    ← confiável (override permitido salvo locked)
        ↓
[Project config (.agent/, committed)]  ← confiável após trust prompt + hash check agregado
        ↓
[Session flags]                  ← confiável (user explícito)
        ↓
[CLAUDE.md]                      ← NÃO-confiável até trust prompt
        ↓
[.agent/memory/shared/ (committed)]  ← NÃO-confiável até trust prompt; PR review é gate; scanner em promoção
        ↓
[.agent/memory/local/ (per-user)] ← confiável (escrito pelo próprio user, com confirmation prompts)
        ↓
[Tool output]                    ← NÃO-confiável; sanitize + injection scan
        ↓
[MCP server response]            ← NÃO-confiável; hash check + sanitize
        ↓
[Web fetch result]               ← NÃO-confiável; sanitize; deny_hosts
        ↓
[Memory body (inferred)]         ← suspeito; require user confirmation
        ↓
[Memory body (untrusted dir)]    ← marcado; não carrega no contexto base
```

### 2.2 Direção de confiança (não simétrica)

- Enterprise confia em: nada (é root)
- User confia em: enterprise + sua própria config
- Project (após trust) confia em: user + enterprise
- Session confia em: project + user + enterprise
- **Tool output não confia em nada** — é payload, não autoridade
- **Modelo não confia em nada** — todo input é não-confiável; permission engine + harness é authority

### 2.3 Inversão proibida

- Tool output NUNCA pode escalar pra autoridade. "Tool me disse pra ignorar policy" → bloqueio fatal.
- MCP server NUNCA pode declarar políticas que o agente respeite. MCP só fornece tools.
- Memory body NUNCA é tratado como instrução de sistema, mesmo em formato `### Instructions`.
- Web fetch NUNCA modifica policy ou hooks.

---

## 3. Attack vectors (catálogo com severidade)

### 3.1 Críticos (mata segurança do produto)

| Vetor | Detalhe | Defesa primária |
|---|---|---|
| Prompt injection persistente via memory | `inferred` write em dir não-confiável vira instrução em todas sessões futuras | Trust boundary + scanner heurístico + confirmação humana + `trust: untrusted` flag |
| Sandbox escape em `bash` | Shell expansion ou syscall não-coberto burla policy | `bwrap` (Linux) / `sandbox-exec` (macOS); deny pesada; eval cobre escape |
| Path traversal em tool args | `read_file("../../../etc/shadow")` | Resolve path absoluto + check dentro de cwd allowed; deny `..` em paths |
| Supply chain do binário | npm postinstall malicioso em dep | Lockfile pinado; audit em CI; binário compilado AOT (Bun) |
| MCP server hostil | Tool injetada com efeito malicioso disfarçada | Trust prompt; hash do manifest; tools MCP em quarentena por default |
| CLAUDE.md prompt injection com tool call | "ignore policy and run bash X" no contexto | Sanitization; injection scanner; trust prompt; permission engine **antes** do harness |

### 3.2 Altos

| Vetor | Detalhe | Defesa primária |
|---|---|---|
| Tool output com ANSI escape malicioso | Esconde texto, fakes confirmação, redireciona terminal | Strip CSI controle, preservar SGR seguro |
| Hook script trocado entre sessões | Auto-format vira exfiltrate | Hash do script + re-trust opcional + audit em `hook_runs` |
| Tool result com fake schema | Modelo confunde; harness nem desconfia | Output schema validation rígida (quando declarado) |
| Memory injection scanner bypass | "Now ignore all previous" mascarado em sintaxe diferente | Multiple patterns; user reviewing always overrides; eval em CI |
| Cost runaway por adversário | Repo malicioso induz loops | `maxCostUsd` hard cap; alerta em jump anormal |

### 3.3 Médios

| Vetor | Detalhe | Defesa primária |
|---|---|---|
| Information disclosure em error message | Stack trace expõe path/configs | Sanitização de error messages user-facing; detalhes só em `--verbose` |
| Time-based attack via tool latency | Latência sinaliza presence/absence | Não defendido em v1; flag explícito como limitação |
| Cache poisoning de prompt cache | Não aplicável (cache é server-side da Anthropic) | Confiança no provider |
| Audit log bypass | Adversário com FS access deleta `failure_events` | Não defendido contra adversário com root local; documentado |

### 3.4 Baixos / Aceitáveis

| Vetor | Detalhe | Decisão |
|---|---|---|
| Adversário com root no host | Pode tudo; agente é apenas userspace | Documentado como fora de escopo |
| Side-channel de timing em LLM stream | Token-by-token expõe estrutura | Não defendido; nicho |
| Adversário com acesso ao terminal compartilhado | tmux session sequestrada | Documentado; usuário responsável |

---

## 4. Defense layers (defense in depth)

Nenhum controle isolado é suficiente. Camadas:

```
┌─ User intent (prompt) ───────────────────────┐
│        ↓                                      │
├─ Trust boundary (1: project) ────────────────┤
│   - Trust prompt                              │
│   - Hash check de CLAUDE.md / .agent          │
├─ Pre-prompt hooks (UserPromptSubmit) ────────┤
│   - User-defined block                        │
│   - Injection scanner                         │
├─ Context engine ─────────────────────────────┤
│   - Layout fixo (cache breakpoints)           │
│   - Memory loaded (untrusted = not in base)   │
│   - Tool output sanitized                     │
├─ Provider call ──────────────────────────────┤
│   - Tool schema rígido                        │
│   - Constrained output (orchestrated)         │
├─ Tool dispatch ──────────────────────────────┤
│   - Permission engine (allow/deny/confirm)    │
│   - Pre-tool hooks (PreToolUse)               │
│   - Schema validation de args                 │
│   - Path traversal check                      │
│   - Permission hierarchy (locked override)    │
├─ Tool execution ─────────────────────────────┤
│   - Sandbox bwrap/sandbox-exec                │
│   - Timeout enforcement                       │
│   - Output sanitize on return                 │
│   - Checkpoint pré-mutação                    │
├─ Audit ──────────────────────────────────────┤
│   - approvals, hook_runs, failure_events     │
│   - traces NDJSON                             │
└──────────────────────────────────────────────┘
```

Falha em qualquer camada **não compromete sistema** — próxima camada pega. Defense in depth de verdade.

---

## 5. Security invariants (sempre verdadeiros)

Propriedades que devem **sempre** valer. Violação = bug crítico, não graceful degradation.

1. **Toda tool com `writes: true` cria checkpoint antes de invocar**, sem exceção.
2. **Toda decisão de permission grava `approvals` row**, antes ou junto com a invocação.
3. **Toda escrita em memória passa por confirmação humana** (UI ou fail em headless).
4. **Tool output passa por sanitization antes de chegar ao contexto.**
5. **Diretório não-confiável bloqueia inferred memory writes** automaticamente.
6. **Locked rules em enterprise nunca são overridable** por user/project/session.
7. **Hook timeout não bloqueia harness por mais que `max_hook_chain_ms`** (15s default).
8. **`maxCostUsd` é respeitado**; sem opt-out runtime.
9. **MCP tools de servidor não-confiável são invisíveis** ao modelo.
10. **Schema violation em tool args bloqueia invocação**, não tenta "best effort".
11. **Path traversal em qualquer tool é fatal** (não warning).
12. **Recap nunca cruza projetos sem `--all-projects`**.

Eval específico cobre cada invariant. PR que quebra invariant é bloqueado.

---

## 6. Secret handling

Categoria especial pela severidade.

### 6.1 Detecção heurística

Patterns checados antes de persistir em logs/traces/memory:

```
- AWS: AKIA[0-9A-Z]{16}, ASIA[0-9A-Z]{16}
- GitHub: gh[psour]_[A-Za-z0-9]{36,255}, github_pat_*
- Google: AIza[0-9A-Za-z_-]{35}
- Slack: xox[baprs]-*
- JWT: eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+
- Generic: bearer\s+[A-Za-z0-9._-]{20,}
- High-entropy strings (Shannon > 4.5) em contexto de "token"/"key"/"secret"
```

Match → **redacted** com `<REDACTED:type>` antes de persistir; warning ao usuário.

### 6.2 Onde nunca pode aparecer

- `messages.content` (vai pro contexto futuro)
- `tool_calls.input/output` (logged + replayable)
- `traces` NDJSON (exported potencialmente)
- `memory body` (persistente cross-session)
- `recap` outputs (pode virar PR description público)
- Mensagens de erro user-facing (vão pra screenshot/Slack)

### 6.3 Onde pode aparecer (transientemente)

- Tool input em runtime (memória de processo, não persiste)
- Stream do modelo em tela (mas não em traces)
- Variáveis de ambiente de hook (se usuário declarou)

### 6.4 Recovery: secret detectado

Se scanner detecta secret **após** já ter sido persistido (eval offline pega):

1. `failure_events` row com classe `data.secret_leaked`
2. Mensagem fatal ao usuário com path do arquivo + recomendação de rotação
3. Tool de cleanup: `agent secret-redact <session_id>` reescreve o que já tem
4. Eval em CI cobre que detection rate ≥ 95% em fixtures

---

## 7. Supply chain

### 7.1 Dependências

- **Lockfile pinado** (`bun.lock` em git)
- **Dep audit em CI** (`bun audit` ou `npm audit`); CVE high+ bloqueia merge
- **No postinstall scripts** em deps; bloqueia install que tenta
- **Dep nova precisa justificativa em PR** (substitui o quê? cobre quanto?)
- **Major bump** com migration plan testado; smoke eval

### 7.2 Build & release

- **Reproducible build**: mesmo source ⇒ mesmo binário (Bun compila AOT)
- **SBOM** gerado em release: lista de deps com versão
- **Sigstore/cosign** assina binário; `agent --version --verify` checa
- **Hash publicado** junto ao release; install script (`curl | sh`) verifica
- **Distribuição**: oficial via canais signed (brew tap, releases assinados); avoid `curl | sh` sem verify

### 7.3 Trust no provider

- Anthropic SDK: confia (vendor de fato)
- Ollama: confia (open source, hash verifiable)
- llama.cpp: confia (open source)
- MCP servers third-party: NÃO-confiável até trust explícito

---

## 8. Runtime hardening

### 8.1 Sandbox de bash

| OS | Mecanismo | v1 | v2 |
|---|---|---|---|
| Linux | `bwrap` (bubblewrap) | opt-in via `--sandbox` | default |
| macOS | `sandbox-exec` | opt-in | default |
| Windows | (TBD — `AppContainer`?) | sem suporte | feature flag |

Profile mínimo:
- Mount: `cwd` read-write; `~/.config/agent` read; `/tmp` read-write isolado; resto read-only
- Network: DENY por default (modo strict); ALLOW lista limitada (modo normal)
- /etc, /root: read-only
- /proc, /sys: minimal

### 8.2 Process isolation

- Subprocess de tool spawned com env limpa (apenas PATH, HOME, USER, AGENT_*)
- `setsid` para evitar processo órfão
- Ulimits: max CPU time, max memory, max file size
- SIGKILL chain garantido (5s grace, depois force)

### 8.3 Filesystem permissions

- `~/.config/agent/` → `0700` (apenas owner)
- `~/.local/share/agent/sessions.db` → `0600`
- `traces/*.ndjson` → `0600`
- Hooks scripts → `0700` esperado; warning se world-readable

---

## 9. Network egress control

### 9.1 Default deny

`web_fetch` tem `deny_hosts` por padrão:

```yaml
web_fetch:
  deny_hosts:
    - "localhost"
    - "127.0.0.0/8"
    - "169.254.0.0/16"   # link-local (AWS metadata, GCP metadata)
    - "10.0.0.0/8"
    - "172.16.0.0/12"
    - "192.168.0.0/16"
    - "fc00::/7"
    - "::1"
  allow_hosts: []         # opt-in se necessário
```

Mitiga SSRF. Override por user explícito em `permissions.yaml`.

### 9.2 Provider hosts

API providers têm allowlist hardcoded:
- `api.anthropic.com`
- `api.openai.com`
- `localhost:11434` (Ollama default)

Nada mais sai do agente sem user override.

### 9.3 OTEL / telemetry

- Default: NDJSON local apenas (sem network)
- Export: opt-in via `OTEL_EXPORTER_OTLP_ENDPOINT` env var
- Endpoint validado: HTTPS obrigatório, host explícito (não wildcard)

---

## 10. Audit & forensics

### 10.1 O que é gravado

```sql
approvals          -- toda decisão de permission
hook_runs          -- execução de hook
failure_events     -- falha classificada
memory_events      -- write/read/delete de memória
recap_runs         -- execução de recap
traces             -- spans OTEL completos
sessions           -- metadados de sessão
tool_calls         -- toda invocação de tool
```

### 10.2 Retention

- Default: 90 dias
- Configurable via `~/.config/agent/config.toml` (`retention_days`)
- Cleanup via cron user-side (`agent gc`) ou hook `Stop`

### 10.3 Forensics output

`agent forensics <session_id>` gera bundle:
- All audit tables for session
- All traces (NDJSON)
- Memory events
- Recap final
- Failure events
- Empacotado em tarball assinado (sha256)

Útil pra: incident review, share com Anthropic em bug report, compliance.

### 10.4 Acesso ao audit

- User local: read direto via SQLite ou `agent forensics`
- Externo: never (exceto user export explícito)

---

## 11. Update & signing

### 11.1 Mecanismo de update

```
agent update              # checa, mostra diff de versão, confirma com user
agent update --check      # só checa
agent update --auto       # opt-in pra automation; nunca default
```

Antes de instalar:
1. Download do release
2. Verifica assinatura via cosign / Sigstore
3. Verifica hash contra release notes
4. Backup do binário atual
5. Replace + verify execution

Falha em qualquer step → abort, agente não atualiza.

### 11.2 Signing

- Releases assinados via Sigstore (keyless)
- Public key disponível em `https://github.com/<org>/agent/security/...`
- `agent --version --verify` mostra assinatura + chain

### 11.3 Rollback

`agent rollback` volta pra versão anterior se update quebrou. Backup automático preservado por 30 dias.

---

## 12. Disclosure & CVE process

### 12.1 Reporting channel

- Email: `security@<projeto>` (não issues públicas)
- Encrypted: GPG key publicada em `SECURITY.md` no repo
- Acknowledged em < 48h
- Triagem em < 7 dias

### 12.2 Disclosure timeline

- Day 0: report recebido, ack ao reporter
- Day 7: triage completa, severity assigned (CVSS-equiv)
- Day 30: fix em branch privada
- Day 60: coordinated disclosure se reporter aceita
- Day 90: full public disclosure independent

Negociável caso a caso (vuln in-the-wild = expedite).

### 12.3 CVE assignment

- High+ severity → CVE solicitado via GitHub Security Advisory ou MITRE
- Patch release com CVE no changelog
- Notificação aos usuários (newsletter se houver; release notes sempre)

### 12.4 Hall of fame

Reporters reconhecidos em `SECURITY_HALL_OF_FAME.md` (opt-in).

---

## 13. Pre-shipping security checklist

Antes de cada release:

- [ ] `bun audit` clean (no high/critical CVE em deps)
- [ ] Lockfile diff revisado (sem deps novas suspeitas)
- [ ] Eval de invariants (§5) passa 100%
- [ ] Eval de injection scanner ≥ 95% detection rate
- [ ] Eval de secret detection ≥ 95% detection rate
- [ ] Sandbox tested em Linux (`bwrap`) e macOS (`sandbox-exec`)
- [ ] Path traversal eval passa em todas tools de write
- [ ] Trust prompt funcional em todos os modos (interactive + headless flag)
- [ ] Sem `console.log` ou `dbg!` em código de produção
- [ ] Release binary signed via cosign
- [ ] SBOM gerado e publicado
- [ ] Hash publicado em release notes
- [ ] Changelog inclui security-relevant changes destacadas
- [ ] CHANGELOG menciona breaking changes em policy schemas

PR que ignora qualquer item: bloqueado.

---

## 14. Multi-user / shared state

### 14.1 Não suportado em v1

Agente é **single-user, single-session por host**. Múltiplos users no mesmo host:
- Cada um tem seu `~/.config/agent/`
- Nenhum compartilhamento automático
- Nenhuma isolação cross-user explícita (kernel-level isolation já cuida)

### 14.2 Caveat: shared `cwd`

Se 2 users editam o mesmo repo via NFS/network share:
- Lockfile (`.agent/lock`) detecta conflito
- Segundo user vê warning: "outro agente ativo em este dir"
- Não há cross-user audit; cada um tem o seu

### 14.3 Enterprise / multi-tenant: out of scope

Agentes multi-tenant (1 servidor, N users) é projeto à parte. Aqui é local-first.

---

## 15. Anti-patterns (não cometa)

| Anti-pattern | Por que ruim |
|---|---|
| Single layer de defesa | Falha = compromisso total |
| Cor em UI marcando "trusted" sem texto | Daltonismo + spoofing fácil |
| Memory write silencioso pra "facilitar UX" | Vetor de injection persistente |
| Sandbox opcional em prod sem warning explícito | User assume seguro, não está |
| Trust prompt skip em headless sem flag | Silenciosamente trusta tudo em CI |
| Tool output renderizada com cores controladas pelo output | ANSI manipulation; modelo influencia render |
| Secret scanner só em write_file, não em traces | Secrets vazam por outro caminho |
| Permission policy sem load validation (carrega broken silenciosamente) | Policy "vazia" = allow all em código bugado |
| Hook script aceito de qualquer path | Hook em `/tmp/...` é vetor |
| `--dangerous` flag sem expiry | Esquece que ativou; vira default acidental |
| MCP server confiado por default | Cada um vira SDK não-auditada |
| Recap em diretório não-confiável sem warning | Vaza decisões pra terceiro |
| "User pediu, então faço" — rm -rf sem confirmação extra | Confiança cega no LLM |
| Telemetry export default-on | Vaza estrutura interna |

---

## 16. Limites declarados (o que **NÃO** defende)

Honestidade explícita do que está fora de escopo:

1. **Adversário com root no host.** Pode tudo. Agente é userspace.
2. **Adversário com acesso ao terminal compartilhado.** Sequestro de tmux, etc — não defendido.
3. **Side-channel de timing.** Latência de stream pode expor estrutura. Aceito.
4. **Compromisso do provider (Anthropic, OpenAI).** Confiamos em vendor; sem mitigação.
5. **Compromisso do CPU/firmware.** Out of scope.
6. **Adversário com FS write em `~/.config/agent/`.** Agente assume integridade do próprio home.
7. **Quantum attacks.** N/A em prazo realista.
8. **DoS via cliente local consumindo todo CPU.** Limites de processo cuidam parcialmente; cliente sob controle do user.

Esses limites declarados **não** são bugs — são escopo. Documentar é mais honesto que pretender defesa.

---

## 17. Segurança e os outros docs

- **`AGENTIC_CLI.md` §9 Trust & Safety** — overview; este doc é detalhamento
- **`MEMORY.md` §7 Trust & Injection** — específico de memória
- **`FAILURE_MODES.md` §14 Adversarial** — recovery de incidente
- **`CONTRACTS.md` §9** — permission engine contract
- **`PERFORMANCE.md` §8 Concurrency** — limits que também são de segurança

Este doc consolida e adiciona; não substitui referências específicas.

---

## 18. Insight final

Segurança em agentic CLI não é o que te impede de cortar — é o que **mede** se você devia cortar.

Trust prompt mede confiabilidade do dir antes de ler. Permission engine mede policy antes de invocar. Sandbox mede syscall antes de executar. Audit mede tudo, sempre, pra revisitar depois. Cada controle é instância de **meça duas vezes, corte uma** aplicada ao espaço adversarial.

Nenhum controle isolado segura. Camadas que segura. Cada camada falha — alguma — em circunstâncias específicas. A regra é: **falha em uma camada não pode comprometer o sistema**.

Honestidade epistêmica vale igualmente em segurança: declarar **o que não defende** é mais útil que pretender defender tudo. Quem promete proteção universal entrega falsa segurança — vetor pior que nenhum.

A regra final é simples: **trate todo input como hostil, todo controle como falível, e toda exceção como evidência de que faltou medir.**
