# NAVIGATION

Subsistema de **navegação de repositório em runtime** do `AGENTIC_CLI`. Substitui indexação semântica pré-computada por exploração iterativa altamente otimizada — `grep`, `glob`, targeted reads, repo walking — orquestrados pelo próprio modelo.

Sem este doc, fica implícito que o harness deve "construir representação global do repo" (SQLite, embeddings, symbol graph) e injetá-la no contexto. Com este doc, fica explícito o contrário: o harness oferece **ferramentas baratas e instantâneas** de navegação; o modelo decide onde olhar.

Premissa raiz: *code muda o tempo todo; freshness importa mais que pré-computação*. Index pré-computado é sempre uma fotografia desatualizada do FS. `grep` é a verdade no momento do call.

> **Inversão consciente.** Versões anteriores deste spec descreviam um subsistema `CODE_INDEX` com schema SQLite, parsers tree-sitter, pipeline de invalidação, tools simbólicas (`read_symbol`, `find_references`, `outline_file`, `dependents_of`). Essa direção foi revertida — ver §11 (Decisão arquitetural). Este doc é o substituto.

---

## 0. Princípios (não-negociáveis)

1. **Source of truth é o filesystem.** Sempre. Não há projeção intermediária a invalidar, porque não há projeção.
2. **Grep-first.** Busca lexical resolve a vasta maioria dos casos de navegação em código. Símbolos, referências, imports — todos visíveis com `grep` + filenames.
3. **Targeted reads, nunca scans cegos.** `read_file` é uma operação cara; usar com `offset`/`limit` ao redor de um match, não o arquivo inteiro.
4. **Exploração iterativa, não retrieval em massa.** Modelo formula hipótese → inspeciona → refina hipótese → inspeciona mais fundo. Não "joga 50 chunks no contexto e reza".
5. **Estrutura implícita do FS é o symbol graph.** Filenames, diretórios, imports, naming conventions. O repo já é navegável; não precisa de um grafo paralelo.
6. **Sem stale index problem.** Rename funciona imediatamente. Refactor não quebra retrieval. Generated code não desincroniza.
7. **Sem embedding no v1.** Princípio 12 do `AGENTIC_CLI.md` ("sem cargo cult") + `ANTI_PATTERNS.md §2.2`. Vector store entra quando dor for medida.
8. **Sem tree-sitter como infra obrigatória.** Tree-sitter pode aparecer em ferramentas específicas (linter, formatter), nunca como pré-requisito de navegação. Sessão funciona em qualquer linguagem suportada por `grep`.
9. **Cache-aware.** Tools de navegação são determinísticas dado FS estável; output é cacheável por hash do input + tree hash do path. Mas o cache vive no harness, não em DB persistente.
10. **Custo de navegação migra pro modelo.** Inferência dinâmica substitui pré-computação. Trade-off aceito: paga-se em tokens de raciocínio o que se economiza em infra, freshness, e bug surface.
11. **Budget explícito de exploração.** Sem cap por tarefa, exploração iterativa degenera em loop. Cada playbook declara um teto; harness sinaliza quando próximo (ver §2.8).

---

## 1. Escopo

### 1.1 O que este subsistema oferece

Quatro famílias de tool, todas operando direto no FS:

| Família | Tools | Custo típico |
|---|---|---|
| **Lexical search** | `grep`, `glob` | ~10-100ms; resultado sempre fresco |
| **Targeted read** | `read_file` com `offset`/`limit` | ~5ms + tokens do trecho lido |
| **Temporal/causal** | `git log`, `git diff`, `git blame` via `bash` | ~50-200ms; sinal complementar (ver §2.5) |
| **Repo overview** | `repo_map` (renderização de filenames + estrutura) | ~50-200ms |

Estas tools são as canônicas em `CONTRACTS.md §2.6`. Nenhuma depende de DB local persistente.

### 1.2 O que este subsistema **não** faz

- **Não pré-computa nada.** Sem initial scan, sem walk de boot, sem indexação background.
- **Não mantém grafo de símbolos.** Modelo infere call graph via `grep` por nome + leitura targeted.
- **Não mantém grafo de imports.** Mesmo método: `grep "from foo"` ou `grep "import.*foo"`.
- **Não resolve polimorfismo, dynamic dispatch, ou macro expansion.** Esses problemas continuam difíceis — a diferença é que agora nem fingimos resolvê-los.
- **Não armazena test mapping.** `grep` no diretório de testes pelo nome do símbolo cobre 90% dos casos; o resto exige leitura humana de qualquer forma.
- **Não snapshota estado.** Replay de sessão em FS modificado pode divergir — é uma propriedade aceita, não um defeito.

### 1.3 Linguagens

**Todas.** `grep` é language-agnostic. A navegação funciona igual em TS, Python, Go, Rust, Java, Ruby, Elixir, OCaml, COBOL.

Linguagens com convenções fortes (Python: `def foo`, Rust: `fn foo`, Go: `func foo`) ganham searches mais precisos via patterns. Esses patterns vivem em `PLAYBOOKS.md` ou skills, **não** num parser embutido no harness.

---

## 2. Estratégias canônicas

### 2.1 Grep-first navigation

Padrão dominante.

```
task: "onde AuthService é usado?"
  ↓
grep -rn "AuthService" --include="*.ts"
  ↓
ler 80 linhas ao redor de matches relevantes
  ↓
inferir architecture
```

vs. abordagem rejeitada:

```
task: "onde AuthService é usado?"
  ↓
SELECT * FROM references_ WHERE target='AuthService'
  ↓
(precisa de schema, parser, invalidation, watcher, …)
```

Grep ganha em: barato, instantâneo, sempre fresco, deterministicamente correto, zero infra.

### 2.2 Targeted file reads

`read_file` sem `offset`/`limit` é um anti-padrão para arquivos > 200 linhas.

| Padrão | Custo típico | Notas |
|---|---|---|
| `cat massive_file.ts` | 8k tokens | desperdício |
| `grep "symbol" file.ts` + `read_file file.ts offset=120 limit=80` | 600 tokens | reduce 13× |

Modelo deve resistir ao instinto de "ler o arquivo inteiro pra ter contexto". O contexto vem do trecho relevante + estrutura inferida do nome/path.

### 2.3 Repository exploration loops

Loop característico — "humano fazendo investigação real":

```
hypothesis
  ↓
inspect (grep / read narrow)
  ↓
refine hypothesis
  ↓
inspect deeper (grep / read narrow)
  ↓
…
```

Em vez de: "vector search → top-k chunks → resumir tudo no contexto".

A iteratividade não é fraqueza — é **a feature**. Cada step refina o que está no contexto; o que não foi necessário nunca entra.

### 2.4 Symbolic navigation implícita

Mesmo sem index, o modelo extrai estrutura simbólica de:

- **Filenames.** `auth.ts`, `auth.test.ts`, `auth.types.ts` revelam módulo + papel.
- **Diretórios.** `src/services/`, `src/components/`, `tests/integration/` revelam camadas.
- **Imports.** `import { foo } from './bar'` revela dependência ponta-a-ponta.
- **Naming conventions.** `UserController`, `useAuth`, `IConfig` revelam kind por convenção.

Esse substrato simbólico é **lido em runtime**, não pré-computado. O custo é uma `glob` ou `ls` ocasional.

### 2.5 Git como sinal de navegação

`git log`, `git diff`, `git blame` não são só ferramentas de versionamento — são primitivas de navegação **temporal e causal**, complementares à navegação espacial (filename, símbolo).

| Pergunta | Comando | Por que melhor que grep puro |
|---|---|---|
| "Onde mexeram em X recentemente?" | `git log -p --follow path/X` | filtra ruído histórico; mostra contexto da mudança |
| "Por que essa linha existe?" | `git blame -L 120,180 file` | conecta linha → commit → autor → mensagem |
| "O que mudou desde minha última leitura?" | `git diff <ref>` | foca em delta, não em estado |
| "Qual a forma da branch atual?" | `git log --oneline -20` | orientação imediata sobre direção do trabalho |
| "Quem entende essa área?" | `git log --format=%an path/ \| sort -u` | aponta humanos pra perguntar |
| "Quando esse bug apareceu?" | `git bisect` ou `git log -S "string"` | corta espaço de busca exponencialmente |

Tratar git como primeira classe muda comportamento padrão:
- Sessão fresca começa com `git status` + `git log --oneline -10`, não só com `ls`.
- "Onde está X?" frequentemente é resolvido por "X foi tocado no último commit — veja o diff".
- Investigação de bug começa por `git log --since` do range onde o bug apareceu.

Cross-ref: `AUDIT.md §3` (timeline unificada) e `RECAP.md` (artefato a partir de sessão) usam git como sinal análogo.

**Limite:** repo sem história significativa (squash extremo, monorepo recém-importado, fresh clone shallow) perde essa primitiva. Fallback é navegação puramente espacial.

### 2.6 Cache-aware prompting

Sem indexing pesado:

- Prompts ficam mais estáveis (tools idênticas turn a turn).
- `[repo_map]` injection vira string canônica de filenames, com cache breakpoint estável (`CONTEXT_TUNING.md §11`).
- Sem injection dinâmico de "chunks relevantes" que quebra cache locality.

Cache hit alto é consequência direta de **não injetar conteúdo dinâmico no system prompt**. Embedding systems falham aqui: cada query troca os chunks injetados, invalidando cache.

### 2.7 Large context, selectively used

Janelas grandes (200k+) permitem manter muita informação viva durante uma exploração. Mas:

- **Não** é justificativa pra "jogar o repo inteiro lá".
- **É** justificativa pra "manter os 10 arquivos relevantes da exploração atual sem precisar evict prematuro".

A skill central é **seleção eficiente do que entra na janela**, não janela grande por si só. Ver `EVICTION.md` e `CONTEXT_TUNING.md §11–13`.

### 2.8 Exploration budget

Sem budget explícito, exploração iterativa degenera em loop ("só mais um grep"). Cap por tipo de tarefa, com sinal de "achei o suficiente":

| Tipo de tarefa | Budget típico (navegações) | Sinal de parada |
|---|---|---|
| Debug pontual | 5-8 | reproduziu causa OU tem 2 hipóteses concretas pra testar |
| Refactor | 10-15 | mapeou call sites + identificou risco crítico |
| Explain module | 3-5 | sabe entry points + 2-3 dependências chave |
| Locate-and-fix | 4-7 | encontrou linha + entendeu contexto imediato |
| Architectural review | 15-25 | cobriu camadas principais |

Heurística de detecção de loop:

- **3 grep consecutivos sem novo símbolo descoberto** → recuar, refinar hipótese.
- **2 leituras de arquivo sem novo trecho relevante** → entry point errado; revisitar plano.
- **Query próxima a anterior** (Levenshtein < 5 do pattern, ou mesmo file/range) → reformular, não repetir.

Implementação no harness:

- Conta navegações por tarefa em `tool_calls` (`AUDIT.md`).
- PostToolUse hook injeta warning não-bloqueante ao aproximar do cap: `"⚠ 12/15 navegações nesta tarefa — considere consolidar"`.
- Hard cap = soft × 1.5; após hard, modelo recebe failure event que força sumarização do que aprendeu até ali (ou delegação pra subagent — ver §3).

Anti-padrão: budget como gate hard sem warning. Modelo precisa do sinal antes de bater, não depois.

**Conexão com subagent (§3):** tarefa que estoura budget consistentemente é candidato natural a delegar pra subagent com contexto isolado e escopo mais largo.

---

## 3. Subagent delegation

Exploração ampla polui contexto do main loop com excertos intermediários que perdem valor após síntese. Subagent com contexto isolado resolve: explora N arquivos, devolve sumário denso, descarta o resto.

### 3.1 Critério de delegação

Pergunta canônica: **"o output bruto da exploração vai poluir mais que o resultado final vai ajudar?"**

| Tipo de query | Decisão | Razão |
|---|---|---|
| "Onde X é definido?" | direto (1 grep) | output curto; main loop quer ver |
| "Quais arquivos implementam padrão Y no repo?" | subagent | output longo; main loop só quer lista sintetizada |
| "Como funciona módulo Z ponta a ponta?" | subagent (`very_thorough`) | exige ler 10+ arquivos; síntese de 200 tokens vale mais que 8k de excertos |
| "Esse import é usado em algum lugar?" | direto | resposta binária |
| "Auditar consistência cross-arquivo de um pattern" | subagent | requer leitura completa de cada match, não excerto |
| "Estourei budget de exploração (§2.8) e ainda não achei" | subagent | re-explorar com contexto fresco e escopo redefinido |

### 3.2 Schema canônico

```yaml
input:
  description: string                   # 3-5 palavras, label visível em UI/audit
  prompt: string                        # task self-contained: contexto + pergunta + form de resposta
  subagent_type: 'explore'|'general'|'plan'|...   # capability set
  breadth?: 'quick'|'medium'|'very_thorough'      # signal de profundidade pro Explore
output:
  summary: string                       # síntese curta (200-2k tokens típico)
  trail?: { tool_calls_count, files_read, queries }   # opcional, pra audit
metadata:
  isolation: 'contextual'               # contexto separado do main; não compartilha histórico
  writes: depende_do_type               # explore=false; general=true
  cost: ~2x_baseline                    # trade-off explícito
```

### 3.3 Trade-offs aceitos

- **Latência 2× pior** que main loop fazendo direto (subagent precisa parsear prompt + bootstrap contexto).
- **Perda de controle fino:** main loop não vê os greps individuais, só o sumário. Se subagent errou no meio do caminho, descoberta é tardia.
- **Custo em tokens 1.5-3×** dependendo de breadth.

**Por que vale:** main loop processa N tarefas em série; subagent processa 1 com profundidade. Especialização > generalização em context budget.

### 3.4 Anti-padrões

- **Delegar tudo.** Main loop perde sense do código; cada decisão depende de prompt + retorno de subagent. Latência cumulativa explode.
- **Não delegar nunca.** Main loop polui com excertos; síntese final sai pobre porque contexto está saturado de raw data.
- **Subagent recursivo profundo.** Subagent que spawneia subagent que spawneia subagent — debug vira impossível, audit vira labirinto. Cap em 2 níveis.
- **Subagent sem prompt self-contained.** "Continue de onde parei" não funciona — subagent não tem o contexto do main. Prompt precisa carregar tudo que importa.
- **Subagent pra query binária.** "Esse símbolo existe?" não justifica o overhead.

### 3.5 Quando subagent perde

- Tarefa requer múltiplas iterações curtas com decisão humana no meio.
- Output esperado é o próprio trail (ex: "me mostre os 5 arquivos pra eu ler"), não síntese.
- Repo pequeno onde main loop cabe tudo sem evict — overhead do subagent não compensa.

Cross-ref: `ORCHESTRATION.md` (subagent semantics), `PLAYBOOKS.md` (templates de subagent especializados).

---

## 4. Tool specialization

`bash` universal é insuficiente para navegação eficiente. Tools especializadas reduzem tokens, ambiguity, exploration chaos.

| Tool | Substitui | Ganho |
|---|---|---|
| `grep` (tool dedicada) | `bash("grep ...")` | output estruturado (path/line/text); cap automático; sem shell escaping |
| `glob` | `bash("find ...")` | pattern matching previsível cross-platform |
| `read_file` (`offset`/`limit`) | `bash("sed -n 100,180p file")` | sem risco de comando errado; UI mostra range |
| `repo_map` | grep ad-hoc para overview | rendering canônico cacheável |

Detalhes de schema em `CONTRACTS.md §2.6.2` (`grep`), `§2.6.3` (`glob`), `§2.6.1` (`read_file`).

---

## 5. `repo_map` sem index

`repo_map` (`CONTEXT_TUNING.md §11`) **não** depende de um subsistema separado de indexação. Ele é computado on-demand a partir do FS:

```
[SessionStart - eager (orchestrated profile)]
  ↓
walk FS respeitando .gitignore
  ↓
collect filenames + 1-line summary heurístico (header da primeira linha não-vazia, ou shebang, ou nada)
  ↓
render como string canônica
  ↓
inject em [repo_map] section
```

Custo: ~100-500ms em repo médio. Cacheável até next FS write (tree hash invalida).

### 5.1 Granularity

| Nível | Conteúdo |
|---|---|
| Mínimo | filenames + diretórios |
| Médio | acima + 1ª linha de cada arquivo (header inference) |
| Detalhado | acima + grep automático de exports por linguagem comum (`export function`, `def`, `func`, `pub fn`) |

Modelo escolhe nível via tool `repo_map(level: "detailed", path_glob: "src/auth/**")`. Default é mínimo em sessão fresca.

### 5.2 Profile-aware

Idêntico ao layout anterior:

| Profile | Repo map injection |
|---|---|
| `autonomous` | Lazy: tool quando modelo pedir |
| `orchestrated` | Eager: section em prompt |
| `hybrid` | Eager no planner step; lazy nos executors |

---

## 6. Quando o modelo deveria pedir mais ferramenta — e quando deveria só usar grep

Frame canônico para decisões:

| Pergunta | Tool ideal | Por quê |
|---|---|---|
| "Onde X é definida?" | `grep "def X\|function X\|class X\|fn X"` | match definicional simples |
| "Onde X é usada?" | `grep "X"` + filtros por kind via inspeção | call sites visíveis lexicalmente |
| "Quais arquivos importam de Y?" | `grep "from.*Y\|import.*Y\|require.*Y"` | imports são lexicalmente uniformes |
| "Qual a estrutura desse arquivo?" | `grep "^(export\|def\|class\|func)" file` ou `read_file` com `limit=60` | header é suficiente; eviting full read |
| "Tudo que esse symbol toca transitivamente" | exploração iterativa multi-step OU subagent (§3) | não há atalho honesto; index pré-computado fingiria precisão que não tem |
| "O que mudou nessa área esta semana?" | `git log --since` + `git diff` (§2.5) | grep não responde "quando" |

Se a pergunta exige resolução semântica (polymorphism, dynamic dispatch, macros) — não há ferramenta barata. Modelo precisa ler código, inferir, e declarar incerteza. **Aceitar essa incerteza é melhor que ter um index que mente com confiança.**

---

## 7. Trade-offs aceitos

Esta arquitetura paga preço em algumas dimensões. Lista explícita pra evitar "redescoberta surpresa":

### 7.1 Custo em tokens de raciocínio

Modelo gasta turns formulando queries, lendo trechos, refinando hipóteses. Em repos grandes, navegação iterativa custa mais tokens que um `read_symbol(name)` retornando direto.

**Por que aceitamos:** tokens de raciocínio são mensuráveis e otimizáveis (`TOKEN_TUNING.md`); infraestrutura de index é custo opaco e crescente.

### 7.2 Imprecisão em casos exóticos

`grep "AuthService"` traz menções em comentários, strings, docs. Modelo precisa filtrar.

**Por que aceitamos:** false positives lexicais são óbvios pro modelo (e pro user lendo o trace); false negatives semânticos de um index quebrado são silenciosos e tóxicos.

### 7.3 Sem call graph determinístico

Pergunta "todos os callers transitivos de X" não tem resposta exata sem index. Modelo dá best-effort.

**Por que aceitamos:** call graph determinístico em linguagens dinâmicas (Python, JS, Ruby) é mentira mesmo com tree-sitter. Confidence falsa é pior que best-effort honesto.

### 7.4 Latência de exploração em repos enormes

Em monorepos > 100k arquivos, `grep` global pode levar segundos. `glob` ajuda restringir escopo; subagent (§3) com `breadth=very_thorough` absorve o custo fora do main loop.

**Por que aceitamos:** monorepos têm scoping natural (workspace, package, módulo); modelo aprende a restringir. Pré-computar o monorepo inteiro tem cost stationary muito maior.

---

## 8. Performance budgets

> **Cross-ref:** [`PERFORMANCE.md`](./PERFORMANCE.md). Esta seção declara budgets; PERFORMANCE.md mede e regride.

### 8.1 Budgets canônicos

| Operação | p50 | p99 | Notas |
|---|---|---|---|
| `grep` em repo médio (3k files) | 80ms | 400ms | `ripgrep` por baixo |
| `glob` em repo médio | 30ms | 150ms | walk com .gitignore |
| `read_file` (com offset/limit, 80 linhas) | 5ms | 20ms | FS direto |
| `read_file` (arquivo inteiro, 500 linhas) | 15ms | 60ms | a evitar |
| `repo_map` render (level=minimal, 3k files) | 100ms | 400ms | cacheado até FS write |
| `repo_map` render (level=detailed, 3k files) | 300ms | 1200ms | grep por exports |

### 8.2 SLO violations

PR bloqueado se:
- `grep` p50 cresce > 30% (regressão na implementação interna).
- `repo_map` minimal p50 cresce > 50%.
- `read_file` p99 cresce > 100%.

---

## 9. Privacy

### 9.1 Filtros default

Mesmo set que seria usado por um indexador — mas aplicado a `grep`/`glob`/`repo_map` direto:

```toml
exclude_default = [
  "node_modules/**",
  ".git/**",
  ".venv/**", "venv/**",
  "__pycache__/**",
  "dist/**", "build/**", "target/**",
  ".env*",
  "**/*.key", "**/*.pem",
  "**/secrets/**",
  "**/.aws/**", "**/.ssh/**",
]
```

Lista alinhada com `SECURITY_GUIDELINE.md §6`.

### 9.2 Configurável

```toml
[navigation]
include = ["src/**", "tests/**", "lib/**"]
exclude = ["src/generated/**", "vendor/**"]
respect_gitignore = true              # default true
follow_symlinks = false               # default false
max_file_size_mb = 5                  # read_file refuses larger
```

### 9.3 PII

Sem DB persistente, não há superfície de leak adicional pra defender. `grep` output passa pelo mesmo redactor que `read_file` (`AUDIT.md §3.4`).

---

## 10. CLI

Nada novo precisa existir como CLI — as tools são exercitáveis via shell padrão:

```
rg "pattern" path/                            # ripgrep (preferido)
fd -e ts "pattern"                            # find por extensão
sed -n '120,200p' file                        # read range
```

Para debugging do harness:

```
agent doctor --check-navigation               # confere que ripgrep está disponível
agent profile grep "pattern"                  # benchmark de uma query
```

Sem `agent code-index status/rebuild/query` — comandos extintos porque não há index.

---

## 11. Decisão arquitetural (ADR)

### 11.1 Contexto

Versão anterior do spec (até `CODE_INDEX.md` de 2026-05-xx) descrevia subsistema completo de indexação:

- SQLite local com schema (`files`, `symbols`, `references_`, `imports`, `test_mapping`).
- Pipeline tree-sitter multi-language com initial scan + incremental + FS watcher.
- Tools simbólicas (`read_symbol`, `find_references`, `outline_file`, `dependents_of`).
- ~750 linhas de spec, ~10 cross-refs em outros docs.

### 11.2 Por que reverter

Análise pública de traces/papers do Claude Code (anti-padrão favorito de inspiração: ele aposta em runtime navigation, não em indexing) somada a observações práticas:

1. **Stale index é fonte silenciosa de bugs cognitivos.** Index é projeção; FS é verdade. Toda divergência leva modelo a recomendações erradas sem trigger de warning.
2. **Refactor/rename quebra retrieval.** Mesmo com invalidation perfeita, há janela de inconsistência.
3. **Generated code desincroniza.** Codebase moderno gera código em build; index sempre persegue.
4. **Custo de infra cresce monotonicamente** (SQLite, parsers, watchers, migrations), enquanto custo de inferência cai a cada geração de modelo.
5. **Símbolos importam mais que semântica textual em código.** `grep` por nome resolve o que vector embeddings tentam aproximar.
6. **Cargo cult.** Embedding/index virou default sem dor real medida — viola `ANTI_PATTERNS.md §2.2` e princípio 12 do `AGENTIC_CLI.md`.

### 11.3 Trade-off explícito

Trocamos:

- **Custo de indexação** (CPU/disk/RAM/manutenção/bug surface)

por:

- **Custo de runtime reasoning** (tokens gastos em exploração iterativa)

Modelos modernos têm janela e raciocínio suficientes pra fazer esse trabalho de IDE em prosa, com a vantagem de freshness garantida.

### 11.4 Quando reconsiderar

Voltar a discussão de index pré-computado se **alguma destas dores for medida**:

| Dor | Threshold | Métrica |
|---|---|---|
| Exploração custa caro demais | tokens/turn de navegação > 30% do total em playbook típico | trace dashboard |
| Repo enorme degrada `grep` | `grep` p99 > 2s em repo de uso real | `PERFORMANCE.md` |
| Modelo erra em call graph crítico | > 15% de tarefas refactor falham por símbolo ausente | eval corpus |
| Custo de modelo > custo de infra estimado | ROI cruza ponto | `PERFORMANCE.md` |

Até lá: sem index.

### 11.5 Inspiração explícita

Esta direção converge com:

- **Unix philosophy.** Pequenas ferramentas compostas em runtime.
- **Incremental discovery.** Cada query refina próxima query.
- **Programador humano real.** Que faz `grep → jump → inspect → infer`, não `vector_search → cosine_similarity → top_k`.

---

## 12. Failure modes

Sem subsistema de index, o catálogo encolhe drasticamente:

| Code | Quando | Recovery |
|---|---|---|
| `navigation.grep_unavailable` | binário `rg`/`grep` ausente | `agent doctor` instrui instalação; sessão prossegue com fallback `bash("grep ...")` lento |
| `navigation.fs_walk_excessive` | `repo_map` em repo > 100k files demora > 5s | sugere `[navigation] exclude` mais agressivo |
| `navigation.read_oversize` | `read_file` em arquivo > `max_file_size_mb` | recusa com sugestão de `grep` + offset/limit |
| `navigation.glob_no_matches` | pattern não casa nada | retorna lista vazia; não é erro (modelo decide) |
| `navigation.budget_soft_exceeded` | tarefa passou cap soft de §2.8 | warning não-bloqueante; modelo é nudged a consolidar ou delegar |
| `navigation.budget_hard_exceeded` | tarefa passou cap hard (soft × 1.5) | force-summarize; sugere subagent delegation |

Comparado com 10+ codes de `index.*` antes: navegação direta é menor superficie de falha.

Detalhamento operacional em [`FAILURE_MODES.md`](./FAILURE_MODES.md) — seção dedicada deixará de existir; failure modes ficam inline em `read_file`/`grep`/`glob` (`CONTRACTS.md §2.6`).

---

## 13. Limites declarados (v1)

- **Sem precisão semântica.** Polymorphism, dynamic dispatch, macros: modelo aproxima; aceita incerteza.
- **Sem call graph transitivo determinístico.** Exploração iterativa cobre, com cost em turns.
- **Sem cross-language linking.** TS chamando WASM Rust requer leitura humana / iterativa do modelo.
- **Sem snapshot por sessão.** Replay em FS modificado pode divergir.
- **`grep` é lexical.** Renomear variável local que casa por acidente com outro símbolo pode poluir output.
- **Repo enormes (> 100k files) requerem disciplina de escopo.** `glob` sempre, `grep` global raramente.

---

## 14. Eval

### 14.1 Métrica chave

**Tokens gastos por tarefa de navegação** (refactor, explain, debug) — medido em `tool_calls.tokens_out` por playbook.

```sql
SELECT
  playbook,
  AVG(tokens_out) FILTER (WHERE tool_name IN ('grep','read_file','glob','repo_map')) AS avg_navigation_tokens,
  AVG(tokens_out) AS avg_total_tokens
FROM tool_calls
JOIN sessions ON tool_calls.session_id = sessions.id
GROUP BY playbook;
```

Threshold pra alarme: `avg_navigation_tokens / avg_total_tokens > 0.30`. Indica que exploração está cara demais — gatilho de §11.4.

### 14.2 Corpus

`evals/navigation/`:

```yaml
- id: find-callers-001
  fixture: fixtures/sample-ts-app/
  task: "list all callers of AuthService.login"
  expected:
    callers_found_min: 5
    false_positives_max: 1
    turns_max: 6
```

PR que mexe em `grep`/`read_file`/`repo_map` roda este corpus em CI.

---

## 15. Insight final

Navigation não é "índice mais barato". É **a inversão consciente** sobre se vale pré-computar a estrutura do repo de antemão.

A diferença prática:

- **Index pré-computado** (rejeitado): harness mantém grafo paralelo do repo; promete precisão semântica; entrega freshness duvidosa e bug surface crescente.
- **Runtime navigation** (escolhido): harness oferece ferramentas baratas; modelo navega como humano navegaria; FS é a verdade no momento do call.

O princípio análogo do `AGENTIC_CLI` aplica em direção oposta ao que `CODE_INDEX` aplicava: o harness faz **menos** trabalho de meta-cognição estrutural — apenas dá as primitivas. Modelo faz **mais** raciocínio em runtime sobre código real, com **budget explícito** (§2.8) e **delegação pra subagent** (§3) quando exploração ampla ameaça poluir o contexto principal.

A aposta: `model reasoning + smart exploration + bounded budget + subagent fan-out > massive precomputed semantic indexing` para qualquer repo que de fato muda.

Spec sem este doc: harness é tentado a re-construir um indexador a cada release. Com este doc: a porta fica fechada com motivo registrado.
