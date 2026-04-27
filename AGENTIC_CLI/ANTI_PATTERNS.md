# ANTI_PATTERNS

Lista canônica de padrões que o `AGENTIC_CLI` **decidiu não fazer**, com motivo. Toda decisão aqui é uma **âncora contra scope creep**: se aparecer PR sugerindo um destes, este doc é o link de fechamento.

Anti-patterns são tão load-bearing quanto princípios. Princípios definem o que perseguir; anti-patterns definem o que rejeitar mesmo quando parece útil. Sem este doc, cada implementador re-discute do zero, e o protocolo erode.

Cada item: **o que é → por que rejeita → quando reconsiderar**.

---

## 1. Identity / persona

### 1.1 Undercover mode (não-se-apresentar-como-IA)

**O que é:** modo onde o agente se passa por humano em interações (chat, email, comentários).

**Por que rejeita:**
- Conflita com princípio 11 ("Confiança explícita, nunca implícita").
- Quebra `[system] identity / role marker` (`CONTEXT_TUNING.md §1.2`) — modelo recebe instrução contraditória, qualidade cai silenciosamente.
- Audit trail fica inverificável: se o agente mente sobre o que é, nada que ele declara é confiável.
- Risco regulatório (EU AI Act art. 52, similar em outras jurisdições).

**Quando reconsiderar:** nunca em escopo do projeto. Casos legítimos (red team, pentest) são tools dedicadas com policy explícita, não modo do agente.

### 1.2 Persona tuning ("you are an expert X who...")

**O que é:** prompts longos descrevendo personalidade, expertise, tom emocional do agente.

**Por que rejeita:**
- `CONTEXT_TUNING.md §1.2` declara **role-as-tool, não persona**. Persona consome tokens estáveis (cache breakpoint #1) sem retorno mensurável.
- Eval mostra que constraints **negativas** (§1.6) afetam mais comportamento do que afirmações de identidade.
- Persona vira teatro: dois implementadores escrevem dois textos diferentes, ambos passam eval, regressão fica invisível.

**Quando reconsiderar:** se eval comparativo (`PERFORMANCE.md`) mostrar que persona melhora ≥ 5% em workflow específico, com p < 0.05. Até lá, role marker curto (3-5 linhas) é o teto.

### 1.3 Verbosity drift como tuning

**O que é:** ajustar verbosity do output via instruções vagas no system prompt ("be concise", "explain thoroughly").

**Por que rejeita:**
- Drift silencioso entre versões de modelo (Anthropic admitiu publicamente em 2026 que Claude Code regressou em verbosity sem mudança intencional).
- Não-mensurável sem eval dedicado.
- Sintoma de não saber qual é o output esperado.

**Substituição:** `CONTEXT_TUNING.md §1.5` (output format expectations literais) + `TOKEN_TUNING.md` (output budget hard cap). Verbosity é consequência de schema + budget, não de adjetivo.

**Quando reconsiderar:** nunca como adjetivo. Sempre como schema ou budget.

---

## 2. Prompts e knowledge

### 2.1 Prompt-as-IP (system prompt secreto)

**O que é:** tratar system prompts como propriedade intelectual fechada, não versionada, não auditável.

**Por que rejeita:**
- Princípio 8 ("permissões e hooks como dado, não como `if`") se estende a prompts.
- Sem versionamento (`AUDIT.md` §prompt_versions), regressão de qualidade não é rastreável a um commit.
- Open-source: prompts ficam em `~/.config/agent/playbooks/*.md` versionáveis e diff-áveis.

**Inversão consciente:** o valor migra de **prompt-quality** para **prompt-curation-workflow** (eval-driven tuning, ver `TOKEN_TUNING.md`).

**Quando reconsiderar:** nunca em escopo do projeto. Se um usuário quer manter playbook próprio privado, isso é um arquivo em `.agent/` gitignored — escolha do usuário, não default.

### 2.2 Vector DB no v1

**O que é:** indexar code/memory/docs em embedding store (Pinecone, Chroma, etc) e fazer similarity search.

**Por que rejeita:**
- Princípio 12 ("sem cargo cult"). `MEMORY.md §0.9` declara: markdown auditável, diff-able, grep-able vence embedding pra os tamanhos típicos (~1500 linhas de memória).
- Embedding = solução pra problema errado quando dataset é < 100k docs ou estrutura é navegável (filesystem, repo).
- Custo: dependência runtime + reindex + drift entre embedding e fonte.

**Quando reconsiderar:** quando dor real existir (memória > 50k entries, busca lexical falha em > 20% dos casos medidos). Não antes.

### 2.3 Multi-model router por task

**O que é:** routing automático entre modelos por tipo de task ("usa Haiku pra X, Opus pra Y").

**Por que rejeita:**
- Princípio 12. Eval-driven choice por playbook (`PROVIDERS.md`) é mais simples e auditável.
- Router dinâmico vira black box: por que escolheu Haiku pra esse turno? Reproduzível?
- Otimização prematura: a heurística é sempre pior que a escolha consciente do usuário no playbook.

**Quando reconsiderar:** quando ≥ 4 playbooks tiverem o mesmo padrão de routing manual e eval mostrar economia ≥ 30% em custo sem regressão de qualidade.

---

## 3. Loop e orquestração

### 3.1 Auto-retry infinito

**O que é:** loop que re-tenta tool failure sem teto até "dar certo".

**Por que rejeita:**
- `ORCHESTRATION.md §1.6` (loop degenerado detection) e `maxToolErrors=5` existem porque retry sem teto é como o orçamento de tokens evapora silenciosamente.
- Mascara bugs: se `bash` falha 50x com mesmo erro, o problema é a invocação, não a flakiness.

**Substituição:** retry com backoff e teto explícito; após teto, falha vira input pro modelo decidir.

### 3.2 Auto-commit pós-edit

**O que é:** agente commita automaticamente após cada edit "porque parece útil".

**Por que rejeita:**
- Princípio 11. Commit é ação compartilhada, não ação do worktree do usuário.
- Memória do projeto: feedback registrado é "só commitar quando pedido explícito; nunca após criar/editar".
- Reverter commits feitos sem permissão é caro; não-commitar é trivialmente reversível.

**Quando reconsiderar:** modo `--auto-commit` opt-in explícito, com policy declarada. Nunca como default.

### 3.3 Stateful tools (tool com memória entre invocações)

**O que é:** tool que mantém estado interno entre chamadas (conexão aberta, cursor, contador).

**Por que rejeita:**
- `CONTRACTS.md §2`: "sem state global mutável entre invocações (tool é função, não classe com estado)".
- Quebra replay (`AGENTIC_CLI.md` princípio 7): mesmo input → mesmo output só vale se tool é pura.
- Side effects implícitos viram bugs intermitentes.

**Substituição:** estado externalizado em SQLite ou FS, tool lê/escreve explicitamente.

---

## 4. UX e output

### 4.1 Spinner sem informação

**O que é:** "Thinking..." / "Working..." sem dizer **o quê**.

**Por que rejeita:**
- `UI.md` e `DESIGN_SYSTEM.md` exigem progress label informativo. Spinner mudo é UI placebo.
- Em sessão > 10s, usuário precisa saber se vale interromper.

**Substituição:** label do step atual ("running tests...", "reading src/auth.ts...").

### 4.2 Markdown rico em --json

**O que é:** modo `--json` retornando strings com markdown embedado (asteriscos, backticks).

**Por que rejeita:**
- `AGENTIC_CLI.md §2.2`: "em modo `--json`, **nada** vai pra stdout que não seja JSONL válido". Markdown vira lixo pro consumidor.
- Quebra pipe-ability (princípio 9).

**Substituição:** campos estruturados (`code: string`, `language: string`); cliente renderiza.

### 4.3 Disclaimers, prefácios, "I'll start by..."

**O que é:** prosa decorativa antes/depois da ação real ("Sure! I'll help you with that. Let me start by...").

**Por que rejeita:**
- `CONTEXT_TUNING.md §1.5`: "Sem prefácio. Sem disclaimers. Sem 'I'll start by...'. Aja."
- Custa tokens estáveis no output, polui scrollback, atrasa primeira ação útil.
- Sintoma de RLHF mal calibrado herdado do modelo base.

**Substituição:** uma frase de status quando há mudança de direção; senão, ferramenta direto.

---

## 5. Audit e segurança

### 5.1 Logging silencioso de PII

**O que é:** gravar tool inputs/outputs crus em audit sem redação.

**Por que rejeita:**
- `AUDIT.md` exige PII redaction **antes-de-persistir**, não no read-time.
- Vazamento via backup, replay, ou recap é vetor real.

**Substituição:** redactor declarativo por tabela (já especificado em `AUDIT.md`).

### 5.2 Trust transitivo

**O que é:** "este diretório é confiável → tudo dentro dele é confiável".

**Por que rejeita:**
- `AGENTS.md` é input não-confiável (princípio 11) mesmo dentro de diretório confiado — pode vir de fork, paste, supply chain.
- Trust em `AGENTIC_CLI` é por **superfície** (FS read, FS write, network, exec), não por path.

**Substituição:** policy YAML com globs explícitos (`SECURITY_GUIDELINE.md`).

---

## 6. MCP

> **Spec consolidada:** [`MCP.md`](./MCP.md). Esta seção é **só** o que **não** fazer.

### 6.1 Re-implementar tools canônicas via MCP

**O que é:** server MCP que expõe `read_file`, `bash`, `edit_file`, ou outras tools do catálogo §2.6.

**Por que rejeita:**
- `MCP.md §4.2` lista nomes reservados; registro é **rejeitado** automaticamente com `mcp.namespace.shadow_canonical`.
- Quebra replay determinístico: se `read_file` pode vir de canônico OU de MCP, sessão sem o server diverge silenciosa.
- Quebra portabilidade: outra instalação do `agentic-cli` sem o server não roda a mesma sessão.

**Substituição:** se canônica tem gap real, abrir issue contra catálogo §2.6 (tools deferred em §2.6.8 viram pull request, não MCP workaround).

**Quando reconsiderar:** nunca para tools canônicas. Para extensões legítimas (databases, services, browsers), MCP é o caminho.

### 6.2 Server que muda manifest a cada sessão

**O que é:** server cujo `tools/list` retorna tools diferentes a cada handshake (ex: tools "dinâmicas" baseadas em FS state, env, ou hora do dia).

**Por que rejeita:**
- Cada manifest change → trust prompt re-disparado → ruído de UX brutal.
- Cache breakpoint #2 invalida toda sessão (`CONTEXT_TUNING.md §5.5`); custo de re-cache de 2-7k tokens em todo turn.
- Trust history (`AUDIT.md §1.5`) infla com hashes superseded sem valor analítico.

**Substituição:** server tem **manifest fixo**; tools recebem parâmetros que selecionam comportamento dinâmico. Ex: em vez de tool `query_postgres_v1` aparecer/sumir, tool `query` recebe `database` como input.

**Quando reconsiderar:** nunca como design pattern. Casos legítimos (server detecta capability X disponível ou não, decide tool por isso): aceitável **se** mudança ocorre apenas em install/upgrade do server, não em runtime.

### 6.3 Auth via env var hardcoded em config

**O que é:**

```toml
[servers.github]
env = { GITHUB_TOKEN = "ghp_actualSecretInsideConfig..." }
```

**Por que rejeita:**
- Config é versionado (`AGENTIC_CLI.md §6` — diretório `.agent/` committed). Secret no git = vazamento.
- Memória do projeto: feedback "só commitar quando pedido explícito" não impede commit acidental de secret se ele estiver em `.agent/mcp.toml`.
- Padrão correto existe: `auth.env` aponta pra var; var não fica em config.

**Substituição:**

```toml
[servers.github]
auth = { kind = "bearer", env = "GITHUB_MCP_TOKEN" }
```

`GITHUB_MCP_TOKEN` vem do shell do user (`~/.bashrc`, `direnv`, etc.), não do config versionado.

### 6.4 Server que ignora `network.allow_hosts`

**O que é:** server stdio que faz HTTP requests internos sem respeitar a allowlist declarada em `[servers.<name>.network]`.

**Por que rejeita:**
- Bypass do `SECURITY_GUIDELINE.md §9` (network egress control).
- Trust prompt aprovou tools, **não** acesso de rede arbitrário.
- Sandbox em Linux/macOS pode pegar (bwrap/sandbox-exec); sem sandbox, é convenção honrada apenas pelo server.

**Substituição:** server bem-comportado **respeita** declaração; servers de terceiros são tratados como hostis até prova em contrário (sandbox opt-in via `[servers.<name>.sandbox = true]`).

**Quando reconsiderar:** nunca. Server que precisa de network = declarar explicitamente.

### 6.5 Server que escreve em diretórios do agente

**O que é:** server que escreve em `~/.config/agent/`, `~/.local/share/agent/sessions.db`, ou paths internos do harness.

**Por que rejeita:**
- Out of contract (`CONTRACTS.md §11` — server não muta state do agente).
- Corrupção de SQLite vira data loss silenciosa.
- Backdoor potencial: server compromete sessões futuras editando memória ou prompts.

**Substituição:** server escreve em `cwd` declarado ou em paths fora do agente. Estado próprio do server fica em diretório do server.

### 6.6 `--auto-approve-mcp` por default em CI

**O que é:** pipeline de CI passa `--auto-approve-mcp` mesmo sem motivo claro, "porque é mais fácil que configurar trust".

**Por que rejeita:**
- Vira backdoor: PR malicioso adiciona server hostil em `mcp.toml`; CI auto-aprova; tools maliciosas viram visíveis.
- Anula camada de trust prompt inteira.
- Princípio 11 ("confiança explícita") evaporado.

**Substituição:**
- CI roda **sem** servers MCP por default.
- Se CI precisa de MCP server específico, trust **pre-gravado** no estado SQLite (commitable em fixture controlada).
- `--auto-approve-mcp` só aceita lista explícita: `--auto-approve-mcp postgres,github` — nunca `--auto-approve-mcp *`.

### 6.7 MCP tool com `_meta.agentic_cli.writes: false` que escreve

**O que é:** server declara `writes: false` no manifest extension, mas tool efetivamente muta FS, DB, ou state externo.

**Por que rejeita:**
- Quebra checkpoint pre-call (`CONTRACTS.md §11`); harness não cria snapshot, `/undo` não cobre.
- Replay diverge: re-execução tem efeito que a primeira não previu.
- Bug de implementação do server, não graceful degradation.

**Substituição:** server **conservador na declaração**: `writes: true` é default seguro; declarar `false` só quando tool é provadamente read-only (sem network mutável, sem cache externo, sem rate-limited counters).

**Detecção:** modo dev hash do `cwd` antes/depois de tool com `writes: false`; mismatch → bug do server, classificado como `mcp.metadata.writes_lied`.

### 6.8 MCP-over-MCP (servers que dependem de outros servers)

**O que é:** server A declara que precisa do server B estar ativo; chain de dependências.

**Por que rejeita:**
- Lifecycle vira complexo: B disconnecta → A degraded em cascata?
- Trust composition virá quem-confia-em-quem-recursivamente.
- v1 explicitamente fora de escopo (`MCP.md §11`).

**Substituição:** user compõe na config — ambos servers em `mcp.toml`, decisão arquitetural do user. Chain implícita = não.

### 6.9 Slash command auto-promotion para tools MCP

**O que é:** harness cria `/<server>:<tool>` automaticamente como slash command pra cada tool MCP.

**Por que rejeita:**
- Slash commands são **UX curados** (`AGENTIC_CLI.md §2.5`).
- 5 servers × 10 tools cada = 50 slash commands poluindo `/help`.
- User que quer atalho cria manualmente em `~/.config/agent/commands/`.

**Substituição:** auto-promotion **never**; manual mapping **always**.

---

## 7. Feature flags

> **Spec consolidada:** [`FEATURE_FLAGS.md`](./FEATURE_FLAGS.md). Esta seção é **só** o que **não** fazer.

### 7.1 Forever-flag

**O que é:** flag em `experimental` por > 6 meses sem decisão de promover (`staged`) ou remover.

**Por que rejeita:**
- Vira default acidental: usuários adotam, dependem, mas spec diz "experimental, sem promessa de estabilidade".
- Audit poluído: `feature_flags_active` cresce sem ganho.
- Bloqueia decisões reais — flag-as-shim disfarçada (ver §7.3).

**Substituição:** TTL hard de 6 meses; `agent doctor` flagga; PR forçando decisão. Excedeu TTL sem PR de promoção → remoção é default.

**Quando reconsiderar:** flag complexa cuja avaliação de eval-driven leva > 6 meses (raro). Extensão **explícita** via PR com justificativa, não derive silenciosa.

### 7.2 Flag-as-config

**O que é:** toggle estável (não muda há 12+ meses) virou CLI flag em vez de config TOML.

**Por que rejeita:**
- CLI flag implica mudança per-sessão; toggle estável não muda per-sessão.
- Force user a relembrar `--no-color --no-llm-render --simple` toda invocação.
- Categoria errada: config é pra estável, flag é pra ephemeral (`FEATURE_FLAGS.md §1`).

**Substituição:** quando flag promove a `stable` no `§2`, revisar se default mudou — se sim, vai pra config; CLI flag fica como override pontual ou some.

**Detecção:** flag em `feature_flags_active` com `set_by = 'init'` (=== "user passou na CLI") em > 80% das sessões → promover a config default.

### 7.3 Flag-as-shim

**O que é:** flag escondendo decisão de design adiada — "decidiremos quando virar problema".

**Por que rejeita:**
- Decisão postergada é decisão tomada (default vence).
- Flag fica como artefato de indecisão que ninguém limpa.
- Acumula com o tempo: 5 flags-shim viram 5 caminhos de código não-testados.

**Sintoma:** flag introduzida sem `cleanup_target`; sem eval que diferencie ON vs OFF; sem owner ativo.

**Substituição:** decidir o default agora; remover ou completar a feature. Se decisão real exige tempo, criar issue de design **antes** de flag.

**Quando reconsiderar:** se eval mostra trade-off real (flag X melhora Y mas piora Z, decisão depende de workload do user), flag legítima — mas precisa lifecycle declarado.

### 7.4 Flag duplicada

**O que é:** mesma intent expressa em CLI **e** config **e** slash. Ex: `--strict` + `[generation] strict = true` + `/strict on`.

**Por que rejeita:**
- Precedência fica confusa: qual ganha em conflito?
- Audit triplica rows em `feature_flags_active`.
- Mantenedor precisa atualizar 3 lugares ao mudar comportamento.

**Substituição:** cada flag tem **um** mecanismo canônico. Se precisa de override pontual + persistência + mid-session toggle, defina precedência canônica (CLI > slash > config) e documente.

**Excepção legítima:** `--strict` (CLI sticky) + `/strict on|off` (slash mid-session) — diferentes use cases. Config TOML não-redundante seria errado: já é feature stable, default-on em alguns playbooks.

### 7.5 Flag não-auditada

**O que é:** flag muda comportamento mas não está em registry → não aparece em `feature_flags_active`.

**Por que rejeita:**
- Replay diverge: sessão A com flag ON ≠ sessão B sem flag, sem rastro de qual era qual.
- Forensics impossível: bug atribuído a "comportamento esquisito" sem ângulo de flag.
- Princípio 7 (`AGENTIC_CLI.md`, "trace tudo") fere.

**Substituição:** CI check — diff entre flag registry e `feature_flags_active` schema. Flag em código ausente em registry → erro de build.

### 7.6 Flag sem dono

**O que é:** flag introduzida no PR; PR autor saiu do projeto; ninguém mantém.

**Por que rejeita:**
- Ninguém promove (fica forever-flag).
- Ninguém remove (custo invisível).
- Decisões dependentes da flag ficam reféns.

**Substituição:** registry exige `owner` field. Quando owner sai, flag entra automaticamente em `deprecated` com warning até novo owner ou remoção.

### 7.7 Default invisível

**O que é:** flag default-on que muda comportamento materialmente, mas usuário não sabe que existe.

**Por que rejeita:**
- Comportamento "mágico" sem affordance.
- Debug fica caro: "por que minha sessão se comporta de X?".
- Princípio 11 (`AGENTIC_CLI.md`, "confiança explícita") fere.

**Sintoma:** `agent --list-flags --active` mostra flag default-on que usuário nunca tocou.

**Substituição:** flags default-on aparecem em `agent doctor` como **sumário** com link pra doc; primeira sessão de um cwd novo lista as relevantes (similar ao trust prompt).

### 7.8 Escape hatch sem audit visual

**O que é:** flag de bypass de segurança (`--dangerous`, `--auto-approve-mcp`, `--no-pipeline`, `--allow-large-fetch`) ativa sem que `<ModeBadges>` (`UI.md §3.2`) mostre, ou sem evento em `failure_events` informacional.

**Por que rejeita:**
- Bypass silencioso = vulnerabilidade UX. Usuário esquece que ativou; vira default acidental em sessões CI.
- Princípio 11 fere (confiança implícita).
- Scenario real: PR malicioso em CI passa `--auto-approve-mcp *` → sem warning visual, vira backdoor.

**Substituição:**
1. Toda flag de bypass aparece como **badge alarmante** no header (`<ModeBadges>` cor vermelha).
2. Toda sessão com flag de bypass gera `failure_event` informacional `flag.bypass_active`.
3. `agent doctor` lista bypass flags ativos por session_id como linha de risco.
4. Em CI, output de qualquer comando do agente inclui linha "⚠ session ran with: --auto-approve-mcp postgres" no fim.

**Detecção:** flag classificada como `bypass: true` no registry; pipeline de validação verifica que cada bypass tem todos os 4 sinais acima.

---

## 8. SQL e acesso a banco

### 8.1 Tool SQL exposta ao modelo (`sql_query`, `db_exec`, ou similar)

**O que é:** tool no catálogo §2.6 que aceita string SQL e executa contra alguma DB (`sessions.db`, `code_index.db`, etc).

**Por que rejeita:**

1. **Meta-cognição não é tool** (`CONTRACTS.md §2.6.8` princípio guia). Consultar audit do próprio modelo (`SELECT * FROM tool_calls WHERE session_id = ?`) é autoanálise — modelo decidindo sobre o modelo. Spec rejeita esse padrão consistentemente (`todo_write` rejeitado pelo mesmo motivo).
2. **Replay quebra.** SQL com `now()`, `random()`, ou query plans dependentes de tamanho de DB não replay determinístico. Princípio 7 (`AGENTIC_CLI.md`, "trace tudo") exige reprodutibilidade.
3. **Tool budget**. SQL é tool **genérica** — opõe princípio 3 ("10 bem desenhadas vencem 40 genéricas"). Cobriria várias capabilities mas com superfície enorme: schema de 23+ tabelas, dialects SQLite, error handling de cada classe.
4. **PII e security.** Read-only não é suficiente: cross-session join (`SELECT * FROM messages WHERE session_id != ?`) expõe contexto de outras sessões. Redactor é antes-de-persistir (`AUDIT.md §3.1`) — SQL bruto bypassa output redaction operando sobre raw rows.
5. **Audit recursivo.** `tool_call` executando SQL sobre `tool_calls` é recursão circular: query é uma tool call gravada em `tool_calls`, que pode ser questionada por outra query. Cleanup, retention, e replay ficam loop-prone.
6. **Confiança explícita** (princípio 11 do `AGENTIC_CLI.md`). SQL é input não-confiável vindo do modelo: prompt injection via doc/tool output → modelo emite SQL malicioso → harness executa. Tools tipadas com schema fechado eliminam esse vetor inteiramente.

**Substituição:** tools tipadas por domínio cobrem todos os casos de uso reais:
- "ler código" → `read_symbol` / `find_references` / `outline_file` / `code_graph` (`CONTRACTS.md §2.6.5c`)
- "lembrar fato" → `memory_search` (`§2.6.5`)
- "investigar histórico" → `agent audit *` (CLI human-driven, **não** tool de modelo)
- "ver inventários" → slash commands (`/mcp list`, `/memory list`, `/flags`, etc.)

**Quando reconsiderar:**

Caso emergente: eval-driven self-improvement do modelo (modelo roda métricas sobre suas próprias sessões pra ajustar comportamento). Se isso virar requisito real, adicionar `analytics_query` com:
- Schema fixo declarativo (DSL com métricas e dimensões enumeradas), **não SQL livre**
- Apenas tabelas de aggregate (counts, averages, percentiles) — nunca rows individuais
- Output sempre agregado, nunca conteúdo de message/tool args
- Cobertura por eval (corpus de queries esperadas)

Esse caminho é paralelo a tools simbólicas (`§2.6.5c`): alta precisão, baixa superfície de ataque. **Hoje:** deferido. Eval continua human-driven (`agent eval --suite`); modelo não precisa rodar suas próprias regressões.

### 8.2 SQL ad-hoc humano sem PII redaction no output

**O que é:** `agent code-index query <sql>` (`CODE_INDEX.md §10`) ou similar, sem aplicar redactor no output.

**Por que rejeita:**
- Tabelas de audit contêm conteúdo de `tool_calls.args`, `messages.body`, `failure_events.payload` — todos `sensitivity: high` em `AUDIT.md §1`.
- SQL ad-hoc é destinado a **troubleshooting humano**, não export — mas output via `--json` pode acabar em log, paste, screenshot, gist.
- Sem redaction, ferramenta de debug vira vazamento de PII.

**Substituição:** SQL ad-hoc passa pelo mesmo pipeline de redaction que `read_file` (`AUDIT.md §3.4`): paths → `~/...`, secrets → `[REDACTED]`, rows que matcham padrões PII flagged.

**Quando reconsiderar:** nunca. Forensics raw exige bundle gerado por `agent forensics <session_id>` (`AUDIT.md §5`) — esse bundle é deliberadamente fechado, com checksum e trust prompt.

### 8.3 SQL ad-hoc em headless/CI sem flag explícita

**O que é:** `agent code-index query <sql>` rodando em CI sem opt-in declarado.

**Por que rejeita:**
- CI tem credenciais elevated (publish, deploy, push) — SQL ad-hoc nesse contexto é vetor de exfil escalado.
- Output de CI vai pra logs públicos (GitHub Actions, etc) — mesmo redacted, expõe estrutura interna.
- Sem flag, comportamento silencioso: PR malicioso adiciona `agent code-index query "SELECT ..."` em workflow → vaza schema interno sem warning.

**Substituição:** flag `--allow-adhoc-sql` exigida em `--json` / non-TTY mode; ausente → comando recusa com erro claro. Em interactive: comando funciona normal (assume troubleshooting humano).

### 8.4 ORM / query builder no modelo

**O que é:** tool que aceita objeto JSON tipo `{ table: 'tool_calls', where: { session_id: '...' }, select: ['id', 'tool_name'] }` e roda como SQL internamente.

**Por que rejeita:**
- Mesmos motivos que §8.1, com "verniz" de schema fixo que não é fixo (selects arbitrários permitem bypass).
- Cross-table join via `joins: [...]` reintroduz superfície completa.
- Aparência de tipagem dá falsa segurança ao implementador.

**Substituição:** se tipagem é o objetivo, faça **uma tool por intenção** (read_symbol, find_references, etc). Cobertura via 4 tools tipadas é mais simples que 1 tool query builder.

---

## 9. Quando este doc muda

Este doc é deliberadamente **append-mostly**. Remover anti-pattern requer:
1. PR com motivo (eval, mudança de premissa, ou bug do anti-pattern original).
2. Atualização do princípio relacionado em `AGENTIC_CLI.md` se aplicável.
3. Migração documentada se houver implementação afetada.

Adicionar anti-pattern requer apenas: nome, motivo, substituição. Mais fácil rejeitar que reconsiderar — by design.
