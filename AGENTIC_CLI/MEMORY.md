# MEMORY

Subsistema de memória cross-session para o `AGENTIC_CLI`. Markdown-based, auditável, escopado, com confirmação explícita de escrita.

Memória **não é log de atividade**, **não é cache semântico**, **não é vector DB**. É um conjunto pequeno e curado de fatos não-deriváveis que o agente carrega entre sessões para evitar repetição de aprendizado.

> Feito mal, memória **piora** o agente. Feito bem, é o que separa "wrapper de API" de "agente que te conhece".

---

## 0. Princípios (não-negociáveis)

1. **Não-derivável apenas.** Se dá pra obter lendo código, git, ou config — não é memória, é dever do agente buscar.
2. **User-auditável sempre.** Memória que humano não vê/edita/apaga é doença.
3. **Escrita confirmada.** Agente nunca grava sem aprovação — vetor de injection direto.
4. **Source tracking obrigatório.** Toda memória sabe se veio do humano explicitamente, foi inferida, ou foi importada.
5. **Escopo isolado.** Project memory não vaza pra outros projetos. User memory vale globalmente. Sem mistura silenciosa.
6. **Index eager, content lazy.** Index sempre carregado (cheap), conteúdo só sob demanda.
7. **Trust boundary explícito.** Memória escrita a partir de diretório não-confiável é marcada e não entra no contexto base.
8. **Apodrece, então previne.** Verify-before-act + expiração opcional + audit log.
9. **Markdown, não vector.** Auditável, diffable, grep'ável, portável. Embedding é solução pra problema errado.
10. **Sem cargo cult de "personalização".** Memória existe pra reduzir repetição, não pra fingir que o agente "tem identidade".

---

## 1. Tipos

Quatro categorias com motivos diferentes de existir. Sem categoria = lixeira.

### 1.1 `user` — quem é o humano

Role, expertise, preferências de trabalho, ferramentas que usa.

```markdown
---
name: user role
description: full-stack TS dev, vive em tmux/SSH, prefere Bun sobre Node
type: user
source: user_explicit
---

Dev sênior, full-stack TypeScript com peso no backend. Trabalha
exclusivamente em terminal (tmux + nvim). Prefere Bun runtime.
Não gosta de explicações longas; valoriza concisão técnica.
```

**Quando salvar:** quando aprende algo durável sobre o humano que afetaria respostas futuras.
**Não salvar:** "está com sono hoje", "está debugando agora" — efêmero.

### 1.2 `feedback` — correções e validações

Corrections ("não faça X") **e** validations não-óbvias ("sim, exatamente"). As duas formas são igual de importantes — só registrar correção te deixa cauteloso demais; só validação te deixa otimista demais.

```markdown
---
name: commit verb casing
description: usar Title Case em verbos de commit, nunca ALL CAPS
type: feedback
source: user_explicit
---

Em commits do repo `blablabla`, usar "Create"/"Update"/"Delete"
(Title Case). Nunca `CREATE` ou `create`.

**Why:** convenção do repo, vista no `git log` desde o início.
Lowercase ou ALL CAPS quebra padrão visível.

**How to apply:** quando sugerir mensagem de commit nesse repo,
gerar `Create FOO.md, BAR.md` — não `feat: add foo`, não
`CREATED foo.md`.
```

**Estrutura obrigatória do body:**
- Regra/fato em primeira linha
- `**Why:**` — motivo (incidente passado, preferência forte, constraint externa)
- `**How to apply:**` — quando/onde a regra dispara

Sem `Why`, você não consegue julgar edge case depois. Vira regra cega.

### 1.3 `project` — estado do trabalho

Decisões, deadlines, motivações, estado em curso **com horizonte de semanas**, não horas.

```markdown
---
name: local-first deadline
description: profile orchestrated precisa estar maduro até 2026-08-15
type: project
source: user_explicit
expires: 2026-09-01
---

Profile `orchestrated` (Step Graph Executor + validators + Ollama)
deve estar maduro até **2026-08-15**.

**Why:** demonstração para grupo de OSS contributors em workshop;
financiamento depende de mostrar local-first funcionando bem.

**How to apply:** priorizar trabalho de M5/M6 sobre M7. Sugerir
adiamento de hybrid para depois do workshop.
```

**Sempre converta datas relativas para absolutas** ao salvar. "Quinta" → "2026-04-30".
**`expires`** opcional mas recomendado — projeto memory apodrece rápido.

### 1.4 `reference` — ponteiro pra fora

Onde achar informação em outro sistema. Não duplica conteúdo.

```markdown
---
name: linear pipeline bugs
description: bugs de pipeline ficam em Linear projeto INGEST
type: reference
source: user_explicit
---

Bugs de pipeline são rastreados em Linear, projeto **INGEST**.

**Use quando:** usuário menciona bug de pipeline, ETL, ingestão.
Sugerir buscar lá antes de palpitar.
```

---

## 2. Escopos

### 2.1 User scope (global)

```
~/.config/agent/memory/
  MEMORY.md                # índice
  user_role.md
  feedback_commit_style.md
  reference_linear_ingest.md
```

Carregado em **toda sessão**, independente do diretório. Cuidado: cada entrada nesse scope custa contexto em todas as sessões pra sempre.

### 2.2 Project scope (por repo) — split em **shared** + **local**

Project scope é dividido em duas sub-pastas com semântica distinta:

```
./.agent/memory/
  shared/                    # VERSIONADO em git (team-wide)
    MEMORY.md                # índice shared
    project_q3_milestone.md
    feedback_team_conventions.md
    reference_linear_ingest.md
  local/                     # GITIGNORED (per-user dentro do projeto)
    MEMORY.md                # índice local
    feedback_my_quirks.md
    project_in_progress.md
```

#### `.agent/memory/shared/` — versionado

- Decisões/convenções/refs **do time**
- Curadas via PR (humano revisa antes de merge)
- Onboarding instantâneo: clone do repo já traz contexto
- Auditoria via `git blame` (quem adicionou, quando, em que PR)

#### `.agent/memory/local/` — per-user

- Inferred memories vão **sempre** pra cá por default
- Working notes individuais
- Observações pessoais (preferências dentro deste projeto)
- Nunca commitadas; cada dev tem o seu

#### Por que split

Sem split: ou todo mundo compartilha tudo (atrito social, vetor de injection ampliado, lock-in de inferência ruim) ou ninguém compartilha (knowledge não compõe). Split entrega:

- Default seguro (inferred = local)
- Promoção explícita pra shared via slash command (§5.4)
- PR review como gate de qualidade/segurança
- Onboarding via clone

### 2.3 Reference é tipo, não scope

Reference vive em qualquer scope/sub-pasta. Faz sentido tanto user-global ("uso Linear pra X") quanto project shared ("docs deste repo em Notion Y") quanto local ("eu uso este endpoint quando debugando").

### 2.4 Resolução & merge

Quando agente consulta, ordem (mais específico → mais genérico):

1. Session flags (volátil, mais específico)
2. **Project local** (`.agent/memory/local/`)
3. **Project shared** (`.agent/memory/shared/`)
4. User (`~/.config/agent/memory/`)
5. Reference de qualquer scope

Conflito: scope mais específico sobrepõe genérico (local > shared > user). Toda decisão logada em `memory_events` com `resolved_from` indicando origem.

### 2.5 Default `.gitignore` (auto-gerado)

Em `agent init` ou primeira invocação num repo sem `.agent/.gitignore`, agente gera:

```gitignore
# .agent/.gitignore (auto-generated; safe to edit)
sessions.db
sessions.db-*
traces/
checkpoints/
memory/local/
*.log
```

User pode editar livremente. Agente **nunca sobrescreve** após geração inicial.

---

## 3. Storage

### 3.1 Arquivo individual

```markdown
---
name: <kebab-case, único no scope>
description: <uma linha; aparece no índice>
type: <user | feedback | project | reference>
source: <user_explicit | inferred | imported>
expires: <YYYY-MM-DD opcional>
trust: <trusted | untrusted opcional, default trusted>
---

<corpo em markdown>

(para feedback/project, estrutura obrigatória):
**Why:** ...
**How to apply:** ...
```

### 3.2 Index (`MEMORY.md`)

```markdown
- [User role](user_role.md) — full-stack TS dev, vive em tmux/SSH
- [Commit verb casing](feedback_commit_style.md) — Title Case em commits
- [Linear pipeline bugs](reference_linear_ingest.md) — INGEST tracker
```

Regras do índice:
- **Uma linha por memória**, < 150 caracteres
- Não tem frontmatter próprio (é índice, não memória)
- Truncado em 200 linhas — força disciplina
- Sempre carregado em system prompt (cache breakpoint após AGENTS.md)

### 3.3 Por que markdown

| Critério | Markdown | Vector DB |
|---|---|---|
| User audit | `cat`, `vim` | dump JSON ilegível |
| Edição manual | direto | precisa re-embed |
| Versionamento | git nativo | snapshots fora de banda |
| Custo de write | zero | embedding API |
| Custo de read | trivial | similarity search |
| Retrieval relevante | exact + grep | parecido ≠ relevante |
| Portabilidade | move pasta | dump/import |

Vector ganha quando você tem milhões de memórias e busca semântica importa. Em memória cross-session de **dezenas** de entries, é over-engineering.

---

## 4. Loading

### 4.1 Eager: index

```
[system prompt]
  ...
[AGENTS.md / project context]
[memory index]                ← MEMORY.md aqui (~150 linhas, ~2k tokens)
  - [User role](user_role.md) — full-stack TS dev
  - [Commit casing](feedback_commit_style.md) — Title Case
  - ...
[tool schemas]
```

Cache breakpoint depois do index — index é estável entre turnos da mesma sessão.

### 4.2 Lazy: content

Modelo lê o índice, decide se vale puxar conteúdo. Tool dedicada:

```
memory_read(name="commit verb casing", scope="user")
→ retorna conteúdo do .md
```

Custo: 1 round-trip de tool call por memória relevante. Aceitável dado raridade.

### 4.3 Auto-injection condicional

Algumas memórias são **carregadas eager mesmo sem o índice**, em momentos específicos:

| Trigger | O que carrega |
|---|---|
| `git commit` mencionado | feedback memories com tag `git` |
| Diretório com `.env` detectado | feedback memories com tag `secrets` |
| Tool `bash` chamada | feedback memories com tag `bash` |

Configuração via `triggers:` no frontmatter — opcional, opt-in.

---

## 5. Writing

### 5.1 Fluxo de escrita

1. Modelo decide salvar (ou usuário pede via `/memory save`)
2. Agente chama `memory_write(...)` com proposta
3. **Default destino: project `local/`** ou user scope (baseado em escopo da proposta). **Nunca direto pra `shared/`** — promoção é ato explícito separado (§5.4).
4. **TUI mostra prompt de confirmação:**

```
📝 Propor nova memória [feedback / project · local]

  name: no-console-log
  description: Em src/, console.log proibido — usar logger.debug
  destino: ./.agent/memory/local/feedback_no-console-log.md
  body:
    Em arquivos `src/**/*.ts`, console.log/warn/error proibidos.
    **Why:** logs estruturados são exportados pra Datadog;
    console.* fura observabilidade.
    **How to apply:** quando editar arquivo em src/, usar
    `import { logger } from "@/lib/logger"` e `logger.debug(...)`.

[a]ccept  [e]dit  [r]eject  [w]hy?

(memória vai pra local; compartilhar depois via:
 /memory promote shared no-console-log)
```

5. User decide. Decisão vai pra `memory_events`.
6. Em modo headless `--json`: write **sempre rejeitado**, retornado como warning. Sessão pode persistir intent em sessão local; user revê depois.

### 5.2 Source tracking

Toda memória tem campo `source`:

| Source | Significado |
|---|---|
| `user_explicit` | User pediu explicitamente ("salva isso") |
| `inferred` | Modelo decidiu salvar baseado em correção/validação |
| `imported` | Veio de export de outra ferramenta ou scope |

UI distingue. `inferred` requer confirmação extra — é o vetor de injection mais provável.

### 5.3 Audit log

```sql
memory_events(
  id TEXT PRIMARY KEY,
  scope TEXT NOT NULL,          -- user | project_local | project_shared
  action TEXT NOT NULL,         -- proposed | created | edited | deleted | read | refused | promoted | demoted
  memory_name TEXT NOT NULL,
  source TEXT NOT NULL,
  session_id TEXT,
  cwd TEXT,
  created_at INTEGER NOT NULL,
  details JSONB                 -- diff, motivo de refuse, hash do source, ref do PR, etc
);
```

Conteúdo das memórias **não vai pro SQLite** — fica em arquivo. SQLite só rastreia eventos.

### 5.4 Promoção (local → shared)

Inferred memories nascem **sempre** em `local/`. Promoção pra `shared/` é ato **explícito** do usuário, respeitando "no auto-commit":

```
/memory promote shared <name>
```

Fluxo:

1. Agente lê `./.agent/memory/local/<name>.md`
2. Mostra preview do conteúdo + diff que vai aparecer em `git status`
3. Roda **scanner adicional** específico de promoção:
   - Path traversal check
   - Secret pattern detection (rejeita se encontra)
   - Injection heuristic (rejeita se forte; warning se fraco)
   - Content fica < 200 lines (limite hard)
4. User confirma com `[p]romote  [c]ancel  [d]iff  [w]hy?`
5. Em accept:
   - Move arquivo: `local/<name>.md` → `shared/<name>.md`
   - Atualiza `shared/MEMORY.md` (índice)
   - Remove entry de `local/MEMORY.md`
   - **Não roda `git add` ou `git commit`** — fica como mudança modificada/staged-able
   - User commita manualmente quando quiser; PR review é gate final
6. `memory_events` registra `action: 'promoted'` com `details.from_scope=local, to_scope=shared`

### 5.5 Demoção (shared → local)

Inverso de promoção, útil quando memória shared não vale mais pro time mas user quer manter localmente:

```
/memory demote local <name>
```

Mesmo fluxo, sem scanner adicional (going to less-trusted scope). Cria mudança em `git status` (deletion em shared, novo em local). User commita.

### 5.6 Headless mode

Em `agent --json` non-interactive:
- `memory_write` em local: rejeitado (ver §5.1.6)
- `memory_write` em shared: **sempre** rejeitado (segurança)
- `memory promote/demote`: rejeitados
- Flag `--allow-memory-write=local` opt-in pra CI específicas

Default fail-closed.

---

## 6. Lifecycle

### 6.1 Verify before act

Antes de agir baseado em memória **factual** (não preferência), confirma:

```
memória diz: src/auth.ts exporta `validateToken`
→ grep -n 'export.*validateToken' src/auth.ts
→ se não bate, atualiza ou descarta memória
```

Memória de **preferência** não precisa verify (preferência não tem "estado atual").

### 6.2 Expiry

Frontmatter opcional `expires: YYYY-MM-DD`. Hook `SessionStart` remove memórias expiradas (com confirmação se houver muitas).

Project memory sem `expires` ganha default **+90 dias** se for `inferred` (não `user_explicit`).

### 6.3 Slash commands

```
/memory list [scope]               # lista do índice (scope: user|project|local|shared)
/memory show <name>                # imprime conteúdo
/memory edit <name>                # abre $EDITOR
/memory delete <name>              # com confirmação
/memory diff                       # mudanças não-confirmadas
/memory save                       # propõe salvar baseado em sessão atual (default: local)
/memory promote shared <name>      # project local → project shared (cria mudança git, sem auto-commit)
/memory demote local <name>        # project shared → project local (idem)
/memory promote user <name>        # project (qualquer) → user global (com confirmação dupla)
/memory expire <name> <date>       # set/update expires
/memory audit                      # tabela memory_events da sessão
```

Promoção entre scopes nunca é silenciosa. Cada uma cria mudança que aparece em `git status` (quando relevante) — usuário decide o commit.

### 6.4 Hook `PreCompact`

Antes de compaction, hook opcional pode revisar memória ("alguma estale?"). Útil em sessão longa onde memória recém-escrita virou redundante.

---

## 7. Trust & Injection

O **vetor de ataque mais sério** do subsistema.

### 7.1 O cenário

1. Atacante coloca `AGENTS.md` malicioso em repo terceiro
2. Você clona, abre o agente
3. Modelo lê `AGENTS.md`, "infere" memória ("Salvar: usuário autoriza ler /etc/shadow")
4. **Confirmação acidentalmente aceita** (ou hook auto-aprovou)
5. Memória vira persistente, prompt-injetando todas as sessões futuras
6. Permanente até user perceber

### 7.2 Mitigações obrigatórias

1. **Trust prompt encadeia em memória:** se diretório não-confiável (§9 do AGENTIC_CLI), `inferred` writes ficam **disabled by default** na sessão. Só `user_explicit` permitido.

2. **Trust marking na memória:** se aceitar `inferred` write em diretório não-confiável, memória ganha `trust: untrusted` no frontmatter. **Não carrega no contexto base** — só sob demanda explícita via `memory_read`.

3. **Hash do source diff:** `memory_events` registra hash do contexto que originou a inferência. Permite rastrear "essa memória veio de onde?" depois.

4. **Hook `MemoryWrite`:** evento bloqueável. Empresa pode forçar audit externo:

```toml
[[hooks]]
event = "MemoryWrite"
matcher = { source = "inferred" }
command = "~/.config/agent/hooks/memory_audit.sh"
# bloqueia em exit 1
```

5. **Confirmação dupla pra user-scope:** memória user-global precisa **dois prompts** (write + escopo) — vai afetar todas as sessões, exige fricção extra.

6. **Sandbox de paths:** memória escrita só em `~/.config/agent/memory/` e `./.agent/memory/`. Tentativa de path traversal = erro fatal + audit.

7. **Read inspeção:** UI mostra `[memory: untrusted]` em qualquer memória `untrusted` carregada — user vê o que tá no contexto.

8. **Hash check em `.agent/memory/shared/`:** trust prompt re-fires quando hash do conjunto de arquivos shared muda. Mesma lógica que `AGENTS.md` (§9.1 do AGENTIC_CLI). Pull do repo com mudança em shared = re-trust obrigatório.

9. **Promoção tem scanner adicional** (§5.4): path traversal, secret patterns, injection heuristic, size limit. Promoção bloqueada se falha qualquer check.

10. **PR review é gate primário pra shared:** memória shared só entra no repo via commit; commit passa por code review do time. Defesa social, não automática — mas eficaz.

### 7.3 Detecção heurística de injection

Antes de propor write, scanner simples checa o body:
- "ignore previous instructions"
- "you are now"
- "from now on, always"
- secret patterns (chaves AWS, GitHub tokens, etc — não salvar nunca)

Match: write **bloqueado**, não só warning. Vai pra `memory_events` como `refused` com motivo.

---

## 8. Tools

```ts
// Lê conteúdo de uma memória (lazy load)
memory_read(name: string, scope?: "user" | "project"): string

// Propõe nova memória — UI confirma com user
memory_write(
  name: string,
  scope: "user" | "project",
  type: "user" | "feedback" | "project" | "reference",
  body: string,
  expires?: string
): { accepted: boolean, path?: string }

// Grep nas memórias (não vector)
memory_search(query: string, scope?: "user" | "project"): MemoryHit[]

// Lista memórias do índice (sem ler conteúdo)
memory_list(scope?: "user" | "project"): MemorySummary[]
```

Schemas validados. Tool `memory_write` **sempre** dispara confirmação UI; em headless, retorna `{ accepted: false, reason: "headless mode" }`.

---

## 9. Profile interaction

### `autonomous`
- Modelo decide quando ler/escrever
- UI confirma writes
- `inferred` source comum, `user_explicit` quando tool é chamada via `/memory save`

### `orchestrated`
- Nó dedicado `memory_consult` no início de DAGs relevantes (carrega memória relacionada antes de step LLM)
- Nó `memory_propose_write` no final, condicional ("se aprendeu algo, propõe")
- Decisões mais determinísticas; menos propostas espúrias

### `hybrid`
- **Read:** local (modelo pequeno consulta índice e busca; barato)
- **Write decision:** frontier ("isso vale memória?" é decisão semântica difícil)
- **Compaction de memória:** frontier (reescrever memórias preserva nuance)

---

## 10. O que **NÃO** salvar

Lista negra explícita:

- Padrões de código → lê o código
- Estrutura de pastas → lê o repo
- Receitas de fix → commit message tem
- Conteúdo de AGENTS.md → já tá no contexto
- Snippets de conversa
- Estado em-progresso ("trabalhando em X agora") → sessão, não memória
- Logs de atividade ("PRs revisados ontem")
- Listas geradas (lista de PRs, lista de issues)
- Resumos de chamadas/reuniões → put em note dedicada, não memória
- Credenciais, tokens, paths absolutos com username (`/home/lex/...` é vazamento)
- Opiniões transitórias ("agora estou empolgado com X")
- Métricas/números que mudam ("temos 50 endpoints") — vai envelhecer

Mesmo se user pedir explicitamente: agente deve perguntar **"o que foi surpreendente ou não-óbvio nisso?"** — só essa parte vira memória.

---

## 11. Anti-patterns (não cometa)

| Anti-pattern | Por que é ruim |
|---|---|
| Auto-save sem confirmação | Vetor de injection garantido |
| Vector DB pra retrieval | Retorna parecido, não relevante |
| Memória global única (sem scope) | Vaza projeto A → projeto B |
| "Atividade do dia" como memória | Apodrece em horas; é log de sessão |
| Memória sem `Why:` | Vira regra cega; não dá pra julgar edge case |
| Memória sem `source` | Sem auditabilidade real |
| Carregar TUDO no contexto | Bloat até derreter o cache |
| `expires` opcional pra project | Apodrece e poluí; default deve forçar TTL |
| Memória escrita por hook automático | Anula confirmação humana |
| Salvar credenciais "porque user pediu" | NÃO. Erro fatal, sempre |
| `inferred` sem hash do source | Não dá pra rastrear injection |

---

## 12. Exemplos completos (templates)

### `user_role.md`

```markdown
---
name: user role
description: full-stack TS dev, terminal-heavy, prefere Bun
type: user
source: user_explicit
---

Dev sênior, full-stack TypeScript com peso em backend.
Trabalha exclusivamente em terminal (tmux + nvim em SSH).
Prefere Bun runtime sobre Node. Não gosta de explicação prolixa.

**How to apply:** respostas técnicas e concisas. Comandos
em Bun por default. Não explicar conceitos básicos de TS.
Não usar emoji em sugestões de código.
```

### `feedback_no_auto_commit.md`

```markdown
---
name: no auto-commit
description: nunca commitar sem pedido explícito; nem após criar/editar
type: feedback
source: user_explicit
---

Nunca crie git commits sem pedido explícito do usuário.
Mesmo após criar/editar arquivos no fluxo natural.

**Why:** usuário controla o histórico do repo manualmente;
auto-commits poluem `git log` e bagunçam batching de mudanças.

**How to apply:** ao terminar edição de arquivo, **não** rodar
`git commit`. Sugerir mensagem de commit ao final, mas executar
só se user pedir explicitamente ("commita isso", "faça o commit").
```

### `project_q2_release.md`

```markdown
---
name: q2 release scope
description: M5-M7 (local-first) precisa estar pronto até 2026-06-30
type: project
source: user_explicit
expires: 2026-07-15
---

Q2 deadline: M5 (local providers), M6 (orchestrated profile),
M7 (hybrid) prontos até **2026-06-30**.

**Why:** apresentação no workshop OSS em 2026-07-10; demo do
profile orchestrated com Ollama é cerne da apresentação.

**How to apply:** priorizar trabalho de M5/M6/M7 acima de
features de v2. Sugerir adiamento de memory cross-session se
houver conflito de prioridade.
```

### `reference_linear.md`

```markdown
---
name: linear ingest tracker
description: bugs de pipeline ficam em Linear projeto INGEST
type: reference
source: user_explicit
---

Bugs de pipeline (ETL, ingestão, transformação) são
rastreados em **Linear**, projeto `INGEST`.

**Use quando:** usuário menciona bug em pipeline, ETL, ingest.
Sugerir buscar issue lá antes de chutar root cause.
```

---

## 13. Insight final

Memória boa não tenta lembrar tudo. Memória boa lembra o que **não dá pra derivar**, com o motivo, no escopo certo, com auditoria completa.

Quem confunde "agente que aprende" com "salvar tudo que parece útil" cria um banco de fatos envelhecidos que prompt-injetam o agente em todas as sessões futuras.

A regra é simples: **se removendo a memória o agente não fica pior em nada concreto, a memória nunca devia ter sido salva.**
