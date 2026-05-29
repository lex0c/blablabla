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

### 5.7 Seed memory (bootstrap)

Memória que o agente **já vem carregando** no primeiro start, antes de qualquer interação. Existe pra evitar que todo agente novo seja "burro day-zero" e precise ser ensinado por correção sobre coisas que todo agente bom deveria saber.

#### 5.7.1 O que é (e não é)

Seed é **meta-comportamento do próprio agente** — como ele se relaciona com edits, commits, confirmação, leitura antes de escrita. É *self-conhecimento operacional*, não conhecimento técnico.

**Vai pra seed:**
- "Confirme ações destrutivas antes de executar"
- "Prefira `Edit` a `Write` em arquivo existente"
- "Não commit sem pedido explícito"
- "Leia arquivo antes de editar"
- "Convenção de commit vem do `git log` do repo, não de default genérico"

**NÃO vai pra seed (é skill/playbook/project memory):**
- "Como debugar query lenta em Postgres" → `skills/pg-heavy-queries.md`
- "Não fazer N+1" → `PLAYBOOKS.md`
- "React 18 usa createRoot" → envelhece; doc do projeto
- "Use Bun em vez de Node" → preferência específica do user, vai pra `user_role`
- "Use kebab-case" → convenção de projeto, vai pra project shared

Heurística: se a regra mudaria entre dois agentes bons de equipes diferentes, **não é seed**.

#### 5.7.2 Frontmatter

Quarta `source` no enum (§5.2 adiciona):

```yaml
source: seed
seed_origin: vendor    # vendor | team | install
seed_version: "1.0"    # versão do catálogo (pra upgrades sem stomp)
```

- `vendor` — veio com o binário do agente
- `team` — catálogo compartilhado de uma org (opt-in)
- `install` — opt-in interativo durante `agent init`

#### 5.7.3 Tratamento como qualquer outra memória

Seed **não é especial em runtime**. Mesma estrutura `Why/How to apply` (§1.2), mesmo lazy load via índice (§4), mesma capacidade de `/memory edit`, `/memory delete`, aparecer em `/memory list`.

`source: seed` serve só pra:
- Auditoria ("essa memória veio do vendor ou eu salvei?")
- Upgrade do catálogo sem stompar customizações (§5.7.5)
- UI mostrar `[seed]` discreto na lista — transparência

#### 5.7.4 Escopo

Seeds vivem em **user scope**, sub-pasta dedicada:

```
~/.config/agent/memory/
  MEMORY.md
  seeds/                       # catálogo seed
    MEMORY.md                  # índice seed
    seed_no_auto_commit.md
    seed_read_before_edit.md
    seed_confirm_destructive.md
  user_role.md                 # memórias normais do user
  feedback_*.md
```

Time pode ter `team seeds` em `.agent/memory/shared/seeds/` via `agent init --team-seeds=<repo>` — opt-in, raro.

#### 5.7.5 Upgrade do catálogo

Seeds são versionadas. Em update:

1. Compara `seed_version` instalada vs. nova
2. Pra cada seed nova/modificada:
   - User não editou (hash bate): atualiza silenciosamente
   - User editou (hash diverge): prompt `[k]eep mine / [v]iew diff / [a]ccept new / [m]erge`
3. Seeds removidas no novo catálogo viram `seeds/archived/`, não delete (reversível)

Default: preserva o que o user mexeu. Nunca sobrescreve customização sem confirmação.

#### 5.7.6 Disable / opt-out

```
agent init --no-seeds              # bootstrap sem seed pack
/memory delete <seed-name>         # remove individual
/memory seeds disable              # mantém arquivos, ignora no load
/memory seeds enable
```

Seed nunca é mandatório.

#### 5.7.7 Limites

- **Hard cap: 10 entradas no pack default vendor.** Se crescer, virou doc — move pra skill ou playbook.
- Body < 30 linhas. Maior que isso, é conhecimento estruturado, não memória.
- Sem `expires` — seed é semântica estável; se envelheceu, sobe versão.
- `trust: trusted` sempre. Seed `untrusted` é contradição.

#### 5.7.8 Catálogo default sugerido (vendor)

Esboço calibrado contra doctrine do KB do user (fontes na última coluna). Hard cap §5.7.7 respeitado.

| Name | Conteúdo essencial | Fonte |
|---|---|---|
| `safe-edit-discipline` | Ler arquivo antes de propor Edit (mesmo se leu em sessão anterior). Em arquivo existente: Edit. Write só pra arquivo novo ou rewrite completo | core |
| `prefer-specialized-navigation` | Usar tool dedicada (Read/Edit/Grep/Glob) em vez de Bash quando existir. Em arquivo > 200 linhas, **grep + read targeted** (offset/limit ao redor do match) em vez de ler arquivo inteiro | `TOOL_ERGONOMICS.md` + `NAVIGATION.md §2.1-2.2` |
| `git-first-orientation` | Sessão fresca em repo git começa com `git status` + `git log --oneline -10` antes de explorar FS. Investigação de bug/mudança recente: git é primitiva de navegação temporal (`git log -S`, `git blame`, `git diff <ref>`), não só de versionamento | `NAVIGATION.md §2.5` |
| `confirm-blast-radius` | Antes de ação irreversível **ou** de raio amplo (rm -rf, force push, drop table, branch -D, mass-rename), mapear o que afeta além do alvo imediato e confirmar | core + `HOLISTIC_VIEW.md` |
| `no-auto-commit` | Não criar commit sem pedido explícito; sugerir mensagem, executar só se user pedir | core (peer de `feedback_no_auto_commit`) |
| `respect-repo-conventions` | Convenção de commit/lint/style vem do repo (`git log`, configs existentes), nunca de default genérico | `COMMIT.md` |
| `scope-discipline` | Não adicionar feature/refactor/abstração além do pedido; bugfix ≠ cleanup; sem abstração antes da terceira repetição | `REFACTORING.md` + `ANTI_PATTERNS_AND_CODE_ENTROPY.md` |
| `failure-root-cause` | Em erro/teste falhando: **reproduzir antes de propor fix**; investigar causa raiz; **red-then-green** ao adicionar teste de regressão (ver teste falhar pelo motivo certo antes de aplicar fix); nunca contornar com `--no-verify`, `except: pass`, skip silencioso, mock que sempre passa | `BRUTE_FORCE_VS_ROOT_CAUSE.md` + `TESTS.md` + skill `add-regression-test` |
| `no-fabrication` | Não inventar URL, path, símbolo, flag, API. Antes de afirmar/agir em "X existe", verificar (grep, ls, request). Memória factual = hipótese até validar. **Premissa do user também** — output que o user mostrou pode estar stale; confirmar baseline antes de tarefa não-trivial. Quando ferramenta lexical não resolve (polymorphism, dynamic dispatch, call graph transitivo, macros), **declarar limite explicitamente** em vez de afirmar precisão falsa | `SCIENTIFIC_METHOD.md` + `NAVIGATION.md §6, §7.2, §13` (absorve `verify-before-act-on-memory` + `no-fabricated-urls`) |
| `secret-handling` | Nunca propor commit de credencial; nunca salvar credencial em memória; redact em logs/output | core (reforça hard rule §10) |

Cada um com `Why/How to apply` no formato da §1.2.

**Consolidações aplicadas:**
- `read-before-edit` + `prefer-edit-over-write` → fundidos em `safe-edit-discipline` (mesma família: higiene de edit)
- `prefer-specialized-tool` ampliado pra `prefer-specialized-navigation` (incorpora grep-first + targeted-read de `NAVIGATION.md §2.1-2.2`)
- `verify-before-act-on-memory` + `no-fabricated-urls` → fundidos em `no-fabrication` (mesmo princípio: não afirmar sem evidência); ampliado com cláusula de "declarar incerteza" pra limites semânticos (`NAVIGATION.md §6`)
- `confirm-destructive` ampliado pra `confirm-blast-radius` (cobre também ações não-destrutivas mas wide-reaching)
- `respect-repo-conventions` absorve commit style (já redundante com `feedback_commit_style` do user)
- `scope-discipline` absorve "no refactor in bugfix" + "entropy budget" (mesma família)
- `failure-root-cause` absorve "no test for fake coverage" (mesma raiz: não fingir que problema sumiu)
- **Novo:** `git-first-orientation` (de `NAVIGATION.md §2.5`) — git como primitiva de orientação inicial, não só versionamento

**Notas do KB promovidas pra `skills/` ou `PLAYBOOKS.md` em vez de seed** (envelhecem ou são situacionais demais):
- `PREMATURE_OPTIMIZATION.md` → playbook (não comportamental day-zero; vira útil sob trigger "performance")
- `DESIGN_CONTRACT.md`, `IDEMPOTENCY.md` → playbook de code-writing (doctrine de código, não comportamento do agente)
- Restante (runbooks, security, ML, RE) → skills já existentes ou a criar

#### 5.7.9 Interação com §7 (Trust)

- `seed_origin: vendor` — trusted por construção (veio com binário assinado)
- `seed_origin: install` — trusted (user aceitou no init)
- `seed_origin: team` — **dispara trust prompt** no primeiro carregamento (catálogo externo = superfície de ataque, mesma lógica de §7.2.8)

#### 5.7.10 Anti-patterns específicos de seed

| Anti-pattern | Por que é ruim |
|---|---|
| Seed com conhecimento técnico versionado ("React 18…", "Postgres 16…") | Envelhece silenciosamente; pertence a skill |
| Seed com convenção de projeto ("use kebab-case") | Não é universal; pertence a project memory |
| Seed > 30 linhas | Virou playbook; move pra `PLAYBOOKS.md` ou `skills/` |
| Seed pra "personalidade" do agente | Cargo cult; sem efeito comportamental concreto |
| Pack default > 10 entradas | Bloat; cada seed custa contexto em toda sessão pra sempre |
| Seed que duplica conteúdo de AGENTIC_CLI.md | Redundância; doc já tá no contexto base |
| Seed sem `seed_version` | Upgrade futuro vai stompar customizações silenciosamente |
| Mandar inferred write virar seed ("isso é tão bom que vira default") | Quebra modelo de origem; promove ruído de sessão a default global |

#### 5.7.11 Templates completos (seeds-chave)

Exemplos canônicos dos 4 seeds que mais mudaram em forma ou são novos. Cada um seria um arquivo real em `~/.config/agent/memory/seeds/`.

##### `seed_safe_edit_discipline.md`

```markdown
---
name: safe-edit-discipline
description: ler antes de Edit; Edit em existente, Write só pra novo ou rewrite completo
type: feedback
source: seed
seed_origin: vendor
seed_version: "1.0"
trust: trusted
---

Ler arquivo antes de propor Edit, mesmo se já leu em sessão anterior.
Arquivo existente: usar Edit. Write só pra arquivo novo ou rewrite
quase total (>70% do conteúdo).

**Why:** Edit em arquivo não lido recentemente pode colidir com estado
real (símbolo renomeado, import movido, contexto diferente) — bug
silencioso: diff parece OK, problema aparece depois. Write em arquivo
existente apaga mudanças unrelated que estavam lá (do user em outra
ferramenta, de outro processo, de merge recente) sem aviso. Ambos os
erros corroem confiança rápido.

**How to apply:**
- Antes de Edit: Read no arquivo (FS pode ter mudado entre sessões)
- Mudança pequena/cirúrgica em arquivo existente → Edit
- Arquivo novo → Write
- Rewrite >70% de arquivo existente → Write é aceitável, mas confirmar
  com user que a substituição completa é intencional
- Nunca usar `sed`/`awk` via Bash pra editar — Edit dedicado mostra
  range na UI e valida match único
```

##### `seed_prefer_specialized_navigation.md`

```markdown
---
name: prefer-specialized-navigation
description: tool dedicada > Bash; grep + read targeted > read inteiro em arquivo grande
type: feedback
source: seed
seed_origin: vendor
seed_version: "1.0"
trust: trusted
---

Usar tool dedicada (Read/Edit/Grep/Glob) em vez de Bash sempre que
existir. Em arquivo > 200 linhas, preferir grep + leitura targeted
(`offset`/`limit` ao redor do match) em vez de ler arquivo inteiro.

**Why:** tool dedicada tem schema validado, output estruturado
(path/line/text), cap automático, sem shell escaping bugs, UI mostra
range/diff. Bash equivalente perde tudo isso. Read de arquivo grande
inteiro gasta tokens em conteúdo irrelevante; trecho ao redor do
match + estrutura inferida do path bastam pra ~90% dos casos
(redução típica de 10-13× em arquivos > 500 linhas).

**How to apply:**
- Procurar símbolo/string → `Grep` (não `bash("grep ...")`)
- Listar arquivos por pattern → `Glob` (não `bash("find ...")`)
- Editar conteúdo → `Edit` (não `sed`/`awk`/HEREDOC via Bash)
- Ler trecho de arquivo > 200 linhas → `Grep symbol file` seguido de
  `Read file offset=N limit=80` ao redor do match
- Resistir ao instinto "ler arquivo todo pra ter contexto" — contexto
  vem do trecho relevante + filename + diretório
- Exceção: arquivo < 200 linhas pode ser lido inteiro; overhead de
  targeted read não compensa
```

##### `seed_git_first_orientation.md`

```markdown
---
name: git-first-orientation
description: sessão fresca em repo git começa por git status + git log -10
type: feedback
source: seed
seed_origin: vendor
seed_version: "1.0"
trust: trusted
---

Em sessão fresca em repo git, começar com `git status` +
`git log --oneline -10` antes de explorar o FS. Tratar git como
primitiva de navegação temporal/causal, não só de versionamento.

**Why:** `ls`/`glob` mostra estrutura mas não direção. `git status` +
`git log -10` revelam em ~200ms: o que está em curso, o que mudou
recentemente, qual o eixo de trabalho atual. Sem essa orientação, o
agente gasta tokens explorando código que pode ter sido refatorado ou
removido na semana passada. Bugs frequentemente vêm de mudança
recente; `git log -S`, `git bisect`, `git blame -L` cortam espaço de
busca exponencialmente vs. grep cego.

**How to apply:**
- Sessão fresca em repo git: rodar `git status` + `git log --oneline -10`
  antes de qualquer exploração de estrutura
- "Onde está X?" muitas vezes é "X foi tocado recentemente —
  `git log -p --follow path/X`"
- Investigação de bug: começar por `git log --since=<range>` ou
  `git log -S "fragmento da mensagem de erro"`
- "Por que essa linha existe?" → `git blame -L start,end file`, não
  tentar inferir do contexto sozinho
- "Quem entende essa área?" → `git log --format=%an path/ | sort -u`
- Fallback: repo sem história significativa (squash extremo, fresh
  shallow clone, monorepo recém-importado) perde essa primitiva —
  cair pra navegação puramente espacial
```

##### `seed_no_fabrication.md`

```markdown
---
name: no-fabrication
description: não inventar fato/URL/path/símbolo; verificar antes de afirmar; declarar incerteza em limite semântico
type: feedback
source: seed
seed_origin: vendor
seed_version: "1.0"
trust: trusted
---

Não inventar URL, path, símbolo, flag, API, conteúdo de arquivo, ou
fato. Antes de afirmar ou agir em "X existe", verificar (`grep`, `ls`,
`Read`, request). Memória factual = hipótese até validar contra estado
atual. **Premissa do user também é hipótese** — output que o user
mostrou pode estar stale; antes de tarefa não-trivial, confirmar o
baseline atual (build verde? lint error realmente presente? arquivo
no estado descrito?). Quando ferramenta lexical não resolve
(polymorphism, dynamic dispatch, call graph transitivo, macros,
metaprogramação), **declarar o limite explicitamente** em vez de
afirmar precisão falsa.

**Why:** fabricação é o erro mais corrosivo de LLM — sai com
confiança alta, junto com conteúdo correto, user só descobre quando
aplica e quebra. Custo de verify é mínimo (1 grep, 1 ls); custo de
fabricação descoberta tarde é alto (debug + erosão de confiança).
False confidence em call graph dinâmico é pior que best-effort
honesto: user trabalha com "minha melhor hipótese, falta verificar X";
não consegue trabalhar com "achei todos os callers" se não achou.
Memória persistente amplifica: fato fabricado e salvo vira "verdade"
em sessões futuras.

**How to apply:**
- URLs: só usar URL fornecida pelo user, presente no código, ou
  obtida via WebSearch/WebFetch. Nunca compor URL plausível
- "Função X em arquivo Y" → `Grep "X" Y` ou `Read` no range antes de
  afirmar
- "Esse caminho existe" → `Glob` ou `Bash ls` antes de afirmar
- Memória diz "src/auth.ts exporta validateToken" → grep antes de
  basear ação; se não bate, atualizar/descartar a memória
- **Premissa do user**: antes de tarefa não-trivial baseada em output
  que o user citou ("o teste X tá falhando", "lint reclama de Y"),
  rodar o comando localmente e confirmar — output pode ser de antes
  de uma mudança, de outra branch, ou de outro ambiente
- Pergunta exige resolução semântica que ferramenta lexical não
  cobre: declarar explicitamente — "best-effort; posso ter perdido
  callers via dynamic dispatch / WASM bridge / metaprogramação".
  Nunca afirmar completude que não tem
- Em dúvida: omitir ou marcar incerteza explícita; nunca preencher
  gap com plausibilidade ("provavelmente é X" sem evidência)
```

##### `seed_confirm_blast_radius.md`

```markdown
---
name: confirm-blast-radius
description: ações irreversíveis ou de raio amplo exigem mapeamento de impacto + confirmação
type: feedback
source: seed
seed_origin: vendor
seed_version: "1.0"
trust: trusted
---

Antes de ação irreversível **ou** de raio amplo (rm -rf, force push,
drop table, branch -D, mass-rename, mass-delete, kill em PID com
filhos), mapear o que afeta além do alvo imediato e confirmar
explicitamente com o user.

**Why:** ação irreversível com blast radius não previsto destrói
trabalho — mudanças uncommitted, branches com WIP, código dependente
que referenciava símbolo renomeado, dados sem backup. Custo de
confirmar é segundos; custo de recuperar de erro destrutivo varia
de horas a impossível. Autorização para uma ação não implica
autorização para próxima ação similar.

**How to apply:**
- Antes de `rm -rf`: listar (sample de) o que vai ser deletado
- Antes de `git push --force`: `git log` do que vai ser sobrescrito;
  warning extra se a branch é main/master
- Antes de `drop table`/`truncate`/`delete from` sem WHERE: confirmar
  backup; nunca em prod sem aprovação explícita
- Antes de `git branch -D`: checar commits unmerged
- Mass-rename/mass-delete: mostrar prévia do diff; aplicar em 2-3
  amostras antes de aplicar a todos
- Permissão pra uma ação destrutiva **não** vale pra próxima similar
  no mesmo session — confirmar de novo
- Se o user pediu "faz X" e X tem efeito colateral não mencionado
  (cascade delete, hook que dispara, lock holder afetado), listar o
  efeito colateral **antes** de executar
```

##### `seed_no_auto_commit.md`

```markdown
---
name: no-auto-commit
description: nunca criar commit sem pedido explícito do user
type: feedback
source: seed
seed_origin: vendor
seed_version: "1.0"
trust: trusted
---

Nunca criar git commit sem pedido explícito do user. Sugerir mensagem
de commit; executar `git commit` só se user pedir.

**Why:** user controla o histórico do repo manualmente; auto-commits
poluem `git log`, baguncam batching de mudanças relacionadas, podem
incluir arquivos não intencionais (`.env`, credenciais, build
artifacts), e tiram do user a decisão de quando/como agrupar
mudanças. "Acabei de editar 3 arquivos, vou commitar" parece útil,
mas remove controle.

**How to apply:**
- Após editar arquivo(s): **não** rodar `git commit` automaticamente
- Ao terminar série de mudanças, sugerir mensagem de commit no
  formato do repo (vem de `respect-repo-conventions`)
- Executar commit só com pedido explícito do user: "commita isso",
  "faz o commit", "git commit -am '...'"
- Mesmo após série longa de edits relacionados, esperar pedido
- Não perguntar "posso commitar?" a cada edit — apenas sugerir
  mensagem e parar
- Em modo headless/CI: nunca commitar, mesmo com flag genérica de
  "auto"
```

##### `seed_respect_repo_conventions.md`

```markdown
---
name: respect-repo-conventions
description: convenções vêm do repo (git log, configs), nunca de defaults genéricos
type: feedback
source: seed
seed_origin: vendor
seed_version: "1.0"
trust: trusted
---

Convenção de commit message, lint, format, naming e estrutura de
arquivo vem do repo — `git log`, configs existentes, arquivos
adjacentes — nunca de default genérico do agente.

**Why:** convenções genéricas (Conventional Commits, Prettier
defaults, "use kebab-case porque AI") frequentemente conflitam com a
convenção real do repo. Aplicar convenção errada cria churn (diff
cheio de mudanças cosméticas não pedidas), atrapalha code review,
e sinaliza falta de atenção ao contexto. `git log` revela o estilo
do repo em segundos.

**How to apply:**
- Antes de propor mensagem de commit: `git log --oneline -20` pra
  inferir formato (Conventional Commits? Title Case verbo? lowercase?
  com scope? sem?)
- Lint/format: respeitar configs presentes (`.eslintrc`, `.prettierrc`,
  `.editorconfig`, `ruff.toml`, `rustfmt.toml`) — não impor
  formatação não configurada
- Naming: ler nomes próximos no diretório, não impor convenção por
  linguagem ("snake_case porque Python" se o módulo usa camelCase
  por motivo qualquer)
- Estrutura de arquivo novo: casar com pares do mesmo diretório
  (padrão de imports, ordem de exports, header convencional)
- `CLAUDE.md`/`AGENTS.md`/`CONTRIBUTING.md` no repo: ler antes de
  propor qualquer mudança que toque convenção
- Se convenção do repo é ruim por critério externo: apontar isso ao
  user separadamente, **não** corrigir de surpresa no PR
```

##### `seed_scope_discipline.md`

```markdown
---
name: scope-discipline
description: ficar no escopo pedido; bugfix ≠ cleanup; sem abstração antes da terceira repetição
type: feedback
source: seed
seed_origin: vendor
seed_version: "1.0"
trust: trusted
---

Não adicionar feature, refactor ou abstração além do pedido. Bugfix
não inclui cleanup. Sem abstração antes da terceira repetição. Sem
error-handling, fallback ou validação pra cenário que não pode
acontecer — confiar em invariantes internas e garantias de framework;
validar só em boundary (input do user, API externa).

**Why:** mudança expandida além do pedido infla o PR, dificulta
review, adiciona risco não autorizado, e mistura concerns. User
pediu bugfix queria bugfix; refactor surpresa no mesmo PR mascara
regressão e quebra bisect. Abstração prematura (DRY antes da hora)
é fonte primária de complexidade acidental — três linhas similares
são quase sempre mais legíveis que helper compartilhado mal cortado.
Defensive code pra cenário impossível adiciona ruído + branches
não-testadas.

**How to apply:**
- Pedido foi bugfix: corrigir o bug. **Não** renomear variável
  adjacente, **não** reformatar, **não** extrair helper, **não**
  atualizar dependência
- Code smell adjacente ao trabalho: marcar pra depois (sugerir como
  follow-up ao user), **não** corrigir junto
- Duas ocorrências do mesmo pattern: copiar está OK. Três: aí avaliar
  se abstração compensa
- Não adicionar `try/except`, validação, ou log "preventivo" pra
  cenário que invariante interna garante não acontecer
- Half-finished implementation: terminar agora ou não começar —
  nunca deixar abstração parcial pro próximo
- Sem feature flag/backwards-compat shim "por garantia" se basta
  mudar o código direto
```

##### `seed_failure_root_cause.md`

```markdown
---
name: failure-root-cause
description: erro/teste falhando exige causa raiz; nunca bypass silencioso
type: feedback
source: seed
seed_origin: vendor
seed_version: "1.0"
trust: trusted
---

Diante de erro, teste falhando, hook bloqueando, ou build quebrado:
**reproduzir primeiro**, depois investigar causa raiz. Ao adicionar
teste de regressão: **red-then-green** (ver o teste falhar pelo
motivo certo antes de aplicar o fix). **Nunca** contornar com
`--no-verify`, `except: pass`, `# noqa` cego, skip de teste sem
motivo escrito, mock que sempre passa, ou retry blind.

**Why:** bypass torna sintoma invisível, não corrige problema. Teste
skipped que cobriria bug real = bug em produção depois. `except: pass`
mascara erro que ia revelar incompatibilidade. Mock que sempre passa
cria CI verde sobre sistema quebrado. Teste que você nunca viu
falhar é suspeito — pode ser tautológico ou estar testando o mock,
não o código. Cada bypass acumula débito invisível: a "rapidez" de
hoje é o incidente de amanhã.

**How to apply:**
- **Reproduzir antes de propor fix**: rodar o comando que falhou,
  ver a mensagem real (não a que o user citou — pode estar stale),
  confirmar que falha de forma consistente. "Fix" sem repro é chute
- Teste falhando: ler mensagem, reproduzir local, formar hipótese,
  validar — nunca pular
- **Red-then-green** ao adicionar regression test pra um bug:
  1. Escrever o teste primeiro
  2. Rodar e ver falhar **pelo motivo certo** (mensagem bate com o
     bug, não com erro de setup)
  3. Aplicar o fix
  4. Rodar e ver passar
  Test escrito depois do fix nunca viu falhar — não há garantia de
  que captura o bug
- Pre-commit hook falhando: investigar e corrigir. `--no-verify` só
  se o user pediu explicitamente, com motivo
- Exception em código: catch específico com handling real (log,
  retry com critério, fallback documentado); nunca `except: pass`
  ou `catch (e) {}` como atalho
- Mock: se sempre passa, não está testando nada — mockar
  comportamento real ou remover o teste
- "Funciona local, falha em CI": investigar diferença de ambiente
  (versões, env vars, ordem de teste, paralelismo), não desabilitar
  o job
- Lint warning: corrigir. Suprimir com `# noqa: <code>` ou equivalente
  só com comentário explicando o porquê
- Build intermitente: investigar como flaky (race, ordem de teste,
  recurso externo); retry blind transforma em problema crônico
- Se o root cause está fora do escopo da tarefa: apontar ao user e
  pedir decisão, não esconder
```

##### `seed_secret_handling.md`

```markdown
---
name: secret-handling
description: nunca commitar/salvar credenciais; redact em output
type: feedback
source: seed
seed_origin: vendor
seed_version: "1.0"
trust: trusted
---

Nunca propor commit de credencial (API key, token, password,
private key, connection string com auth). Nunca salvar credencial
em memória persistente. Redact em logs e output mostrado ao user.

**Why:** credencial commitada fica pública até deletada — e mesmo
depois persiste em `git history` sem rewrite (e em forks/clones e
GitHub event cache). Salvar credential em memória persistente
contamina todas as sessões futuras + pode vazar em exports/sync.
Output sem redact aparece em transcripts compartilhados, screenshots,
audit logs, suporte. É o vetor primário de incidente de segurança
operacional.

**How to apply:**
- Antes de propor commit: scan dos staged files por padrões
  (`AKIA[0-9A-Z]{16}`, `ghp_*`, `ghs_*`, `xoxb-*`, `-----BEGIN.*PRIVATE KEY-----`,
  JWT 3-segmentos, `.env*`, `*.key`, `*.pem`, `credentials.json`,
  `id_rsa*`) — avisar e bloquear
- User pede "salva essa API key na memória" → **recusa explícita**
  com sugestão de password manager / env file / secret manager
- Credential aparece em output (response de curl, log file): redact
  antes de exibir (`***REDACTED***` ou últimos 4 chars apenas);
  nunca eco completo
- Arquivos sensíveis (`.env`, `secrets/`, `credentials.json`,
  `~/.aws/`, `~/.ssh/`): nunca em `git add`; warning se aparecem
  em diff
- Env var contendo secret: imprimir só o nome, nunca o valor
- Em dúvida sobre se algo é secret: tratar como secret (fail safe)
- Se already commited: orientar processo de rotation + history rewrite,
  não fingir que `git rm` resolveu
```

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
