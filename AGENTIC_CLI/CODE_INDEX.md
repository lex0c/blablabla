# CODE_INDEX

Subsistema de **indexação semântica de código** do `AGENTIC_CLI`. Fornece a base estruturada que alimenta `repo_map`, tools simbólicas (`read_symbol`, `find_references`, `outline_file`, `imports_of`), e per-workflow recipes (`callers section` em refactor, `target` em explain).

Sem este doc, indexação fica como string que vai no prompt (`CONTEXT_TUNING.md §11`) — sem schema, sem API queryable, sem invalidação formal. Com este doc, vira **subsistema** com contrato, persistência, observabilidade, e budget — paritário com `MEMORY.md`, `MCP.md`, `RECAP.md`.

Premissa raiz: *meça duas vezes, corte uma*. Indexar é medir o repo; cortar tokens é decidir o que ir pro modelo. Sem o índice, o segundo é cego — modelo grep-storma ou recebe arquivos inteiros. Com o índice, cada inclusão de código é decisão informada.

---

## 0. Princípios (não-negociáveis)

1. **Source of truth é o filesystem.** Index é projeção; FS é autoritativo. Mismatch → reindex, não conserto manual.
2. **Incremental por default.** Reindex full é raro; PostToolUse hooks atualizam por arquivo afetado.
3. **Lexical primeiro, semântico opt-in.** Tree-sitter symbols + import edges cobrem 90% dos casos sem custo de embedding (ver `ANTI_PATTERNS.md §2.2`).
4. **Per-language, sem fallback de regex.** Linguagem com grammar tree-sitter ✓ ; sem grammar ⇒ não indexada (não regex-aproximada). Falha visível > correção silenciosa.
5. **Privacy por default.** Não indexar `.env`, `node_modules`, secrets — mesma lista que `SECURITY_GUIDELINE.md §6`.
6. **Index é dado, não cache.** Pode ser deletado e reconstruído, mas tem schema versionado e migrations — não é "regenera quando quebra".
7. **Não substitui `read_file`.** Index responde "onde está X?"; read_file responde "o que está em X?". Linhas distintas.
8. **Sem vector embedding no v1.** Princípio 12 do `AGENTIC_CLI.md` ("sem cargo cult"). Reconsiderar quando dor real existir.
9. **Sem dependência runtime obrigatória.** Index é amenity; sessão funciona (degradada) sem ele. Ausência → `repo_map` vira fallback grep-based em arquivos pequenos, e tools simbólicas retornam `index_unavailable`.
10. **Invalidação é load-bearing.** Index stale é pior que index ausente: modelo recebe info contraditória ao FS. Strategy de invalidação é tão importante quanto schema.

---

## 1. Escopo

### 1.1 O que indexa

| Categoria | Cobertura |
|---|---|
| **Symbols** | funções, classes, métodos, types/interfaces, exports, constantes top-level |
| **References** | call sites, type references, import references |
| **Imports** | `import`/`require`/`from`/`use`/etc — grafo de dependências entre arquivos |
| **File metadata** | path, language, size, last_modified_at, hash do conteúdo |
| **Test mapping** | best-effort: `foo.test.ts` ↔ `foo.ts`, `test_foo.py` ↔ `foo.py` |

### 1.2 O que **não** indexa (escopo declarado)

- **Conteúdo de comentários e docstrings.** Out of scope no v1; modelo lê via `read_file` quando relevante.
- **Linhas de código** (line-by-line). Index é estrutural, não textual. Para busca textual: `grep` (`CONTRACTS.md §2.6.1`).
- **Branches que não são `HEAD`.** Apenas working tree atual; mudança de branch invalida e reindexa o diff.
- **Files não-versionados se em `.gitignore`.** Trust boundary: gitignored = potencialmente sensível.
- **Binaries, mídia, lockfiles.** `package-lock.json`, imagens, etc — heurística por extensão e size.

### 1.3 Linguagens v1

Grammars tree-sitter built-in (compilados no binário):

| Linguagem | Suporte v1 | Razão |
|---|---|---|
| TypeScript / JavaScript | ✓ | stack do harness |
| Python | ✓ | universalidade |
| Go | ✓ | mesma razão |
| Rust | ✓ | hot ecosystem agentic |
| Java | ✓ | enterprise reach |
| C / C++ | parcial | grammar instável; opt-in via flag |
| Markdown | ✓ (headers only) | docs index |
| TOML / YAML / JSON | ✓ (estrutura, não values) | config navigation |

v1.1: Ruby, PHP, C#, Kotlin, Swift. Adições requerem PR contra este doc + benchmark de parse < 100ms p50 em arquivo de 1k linhas.

---

## 2. Storage (SQLite)

### 2.1 Schema canônico

Tabelas residem em `~/.local/share/agent/code_index.db` (separada de `sessions.db` para permitir delete sem afetar audit). Per-project: cada `cwd` tem seu próprio DB.

```sql
CREATE TABLE files (
  path TEXT PRIMARY KEY,                  -- caminho relativo ao project root
  language TEXT NOT NULL,                 -- 'typescript' | 'python' | ...
  content_hash TEXT NOT NULL,             -- SHA256 do conteúdo
  size_bytes INTEGER NOT NULL,
  loc INTEGER NOT NULL,                   -- linhas of code (excluindo blank/comment)
  last_modified_at INTEGER NOT NULL,      -- mtime do FS
  indexed_at INTEGER NOT NULL,            -- quando index processou
  parse_status TEXT NOT NULL              -- 'ok' | 'partial' | 'failed' | 'skipped'
                CHECK (parse_status IN ('ok','partial','failed','skipped')),
  parse_error TEXT,                       -- nullable; classe de erro se failed
  index_schema_version INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE symbols (
  id INTEGER PRIMARY KEY,
  file_path TEXT NOT NULL,
  name TEXT NOT NULL,                     -- nome local (ex: 'login')
  fqn TEXT,                               -- fully-qualified (ex: 'src/auth.ts:Auth.login')
  kind TEXT NOT NULL,                     -- 'function'|'class'|'method'|'type'|'interface'|'const'|'enum'
  visibility TEXT NOT NULL,               -- 'export'|'public'|'private'|'internal'|'unknown'
  signature TEXT,                         -- assinatura formal (params + return type) se inferível
  start_line INTEGER NOT NULL,
  start_col INTEGER NOT NULL,
  end_line INTEGER NOT NULL,
  end_col INTEGER NOT NULL,
  parent_symbol_id INTEGER,               -- método dentro de classe; FK soft a symbols.id
  FOREIGN KEY (file_path) REFERENCES files(path) ON DELETE CASCADE
);

CREATE INDEX idx_symbols_name ON symbols(name);
CREATE INDEX idx_symbols_fqn ON symbols(fqn) WHERE fqn IS NOT NULL;
CREATE INDEX idx_symbols_file ON symbols(file_path);
CREATE INDEX idx_symbols_kind ON symbols(kind);

CREATE TABLE references_ (                -- 'references' é palavra reservada SQLite
  id INTEGER PRIMARY KEY,
  source_file TEXT NOT NULL,              -- arquivo onde aparece o uso
  source_line INTEGER NOT NULL,
  source_col INTEGER NOT NULL,
  target_symbol_name TEXT NOT NULL,       -- nome chamado/usado (sem resolução; pode ser ambíguo)
  target_symbol_id INTEGER,               -- resolvido se único; NULL se ambíguo ou external
  ref_kind TEXT NOT NULL                  -- 'call'|'type'|'import'|'extends'|'implements'
                CHECK (ref_kind IN ('call','type','import','extends','implements')),
  FOREIGN KEY (source_file) REFERENCES files(path) ON DELETE CASCADE,
  FOREIGN KEY (target_symbol_id) REFERENCES symbols(id) ON DELETE SET NULL
);

CREATE INDEX idx_references_target ON references_(target_symbol_id) WHERE target_symbol_id IS NOT NULL;
CREATE INDEX idx_references_target_name ON references_(target_symbol_name);
CREATE INDEX idx_references_source ON references_(source_file);

CREATE TABLE imports (
  id INTEGER PRIMARY KEY,
  source_file TEXT NOT NULL,              -- quem importa
  target_path TEXT,                       -- caminho relativo ao project root, NULL se external
  target_module TEXT,                     -- 'react', './utils', '@/lib/auth' (raw)
  imported_names TEXT,                    -- JSON array: ['login', 'logout'] ou ['*']
  is_external BOOLEAN NOT NULL,           -- true = node_modules, vendor, stdlib
  FOREIGN KEY (source_file) REFERENCES files(path) ON DELETE CASCADE
);

CREATE INDEX idx_imports_source ON imports(source_file);
CREATE INDEX idx_imports_target ON imports(target_path) WHERE target_path IS NOT NULL;

CREATE TABLE test_mapping (
  test_file TEXT NOT NULL,
  source_file TEXT NOT NULL,
  confidence REAL NOT NULL,               -- 0.0-1.0; heurística (filename, import, etc)
  inferred_by TEXT NOT NULL,              -- 'filename'|'import_graph'|'manual'
  PRIMARY KEY (test_file, source_file),
  FOREIGN KEY (test_file) REFERENCES files(path) ON DELETE CASCADE,
  FOREIGN KEY (source_file) REFERENCES files(path) ON DELETE CASCADE
);

CREATE TABLE index_meta (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
-- linhas: 'project_root', 'last_full_scan_at', 'tree_sitter_version',
-- 'index_schema_version', 'partial_reindex_count'
```

### 2.2 Tamanho estimado

| Repo | Files | Symbols | DB size |
|---|---|---|---|
| Pequeno (~300 files) | 300 | ~3k | ~5 MB |
| Médio (~3k files) | 3k | ~30k | ~60 MB |
| Grande (~30k files) | 30k | ~300k | ~600 MB |

Cap soft: `code_index.db_size > 1 GB` → warning em `agent doctor`. Mitigação: ajustar `exclude` filtros.

### 2.3 Por que SQLite e não in-memory

- **Persistência cross-session.** Initial scan é caro (5-30s em repo médio); recomputar a cada session start é desperdício.
- **Queryable durante runtime.** Tools simbólicas viram SELECT, não scan in-memory.
- **Backup trivial.** `code_index.db` é ignorável em git mas deletável sem dor.
- **Migrations.** Schema versioning permite evolução sem reindex full.

In-memory **adicional** é sim: cache LRU de queries quentes (`agent doctor` mostra hit ratio).

---

## 3. Pipeline de indexação

### 3.1 Initial scan (primeira vez)

Trigger: primeira sessão em `cwd` que detecta ausência de `code_index.db`, OU `agent code-index rebuild` explícito.

```
1. Detectar project root (git root ou cwd)
2. Walk FS respeitando .gitignore + filtros de §1.2
3. Para cada arquivo elegível:
     a. Inferir linguagem por extensão (com fallback a shebang)
     b. Tree-sitter parse → CST
     c. Extrair symbols, references, imports via queries pré-definidas
     d. INSERT em batch (transação por 100 arquivos)
4. Resolver references (segundo passe: liga target_symbol_name a target_symbol_id)
5. Inferir test_mapping
6. UPDATE index_meta SET value=now() WHERE key='last_full_scan_at'
```

Executado **em background** com progress bar inicial. Sessão começa funcional sem aguardar conclusão; tools simbólicas retornam `index_warming` até pronto.

Performance budget (`PERFORMANCE.md §X`):

| Fase | Repo médio (3k files) |
|---|---|
| FS walk + filter | ~500ms |
| Tree-sitter parse (paralelo, 4 workers) | ~10-20s |
| Reference resolution | ~3-5s |
| **Total** | **~15-30s** |

### 3.2 Incremental update (caminho quente)

Trigger: `PostToolUse` hook em `write_file` / `edit_file`, OU `bash` que escreve em FS (detectado via tree hash diff).

```
1. Para cada arquivo modificado:
     a. Compute novo content_hash
     b. Match com files.content_hash
        ├── match: skip (touch só last_modified_at)
        └── mismatch: DELETE FROM symbols/references/imports WHERE file_path=?
                      → re-parse → INSERT
2. Re-resolve references que apontavam pra arquivos afetados
3. UPDATE files SET indexed_at=now()
```

Custo: ~30-100ms por arquivo (parse + delete + insert + reference resolution local).

Para `bash` que muda muitos arquivos (ex: `npm install`, `git checkout`): batch detection — se > 50 arquivos mudaram, vira incremental scan ao invés de processar individualmente. Threshold configurável.

### 3.3 Mudança externa (FS watcher)

Trigger: `inotify` (Linux) / `FSEvents` (macOS) / `ReadDirectoryChangesW` (Windows).

- Watcher é **opt-in** via `[code_index] watch = true`. Default: `false`.
- Razão default-off: `inotify` em repos grandes consome FDs; nem todo user precisa de update fora de tool calls.
- Quando ON: mesmo pipeline de §3.2, disparado por evento de FS em vez de hook.
- Debounce 500ms (evita reindex em rajada de saves).

Sem watcher: mudanças externas (git checkout, edit em outro editor) ficam invisíveis ao index até próximo `bash`/`write_file` que dispare detection. `agent doctor` flagga `last_full_scan_at` velho.

### 3.4 Full rebuild (escape hatch)

```
agent code-index rebuild           # sequencial, mostra progress
agent code-index rebuild --clean   # delete code_index.db, reindex from scratch
agent code-index rebuild --since <commit>  # apenas arquivos no diff
```

Quando usar:
- Schema migration (`index_schema_version` bump)
- Index aparenta corruption (queries retornam stale refs)
- Mudança de filtros em `[code_index]` config
- Tree-sitter grammar atualizada (linguagem ganha cobertura nova)

### 3.5 Stale detection

Em `SessionStart`:

```sql
SELECT path, content_hash FROM files
WHERE last_modified_at < (
  SELECT MAX(last_modified_at) FROM files
) - 86400;  -- arquivos não indexados há 24h
```

Se > 5% dos arquivos: warning em UI, sugere `agent code-index rebuild --since <last_session>`.

---

## 4. Query API (interface canônica)

API exposta como funções TypeScript no harness. **Não** é tool exposta ao modelo diretamente — modelo usa via tools simbólicas (`§5`) que envolvem essas calls.

### 4.1 Funções

```ts
type CodeIndex = {
  // Symbol queries
  get_symbol(name: string, opts?: { kind?: SymbolKind, file?: string }): Symbol[];
  get_symbol_by_id(id: number): Symbol | null;
  list_symbols_in_file(path: string): Symbol[];

  // Reference queries
  find_references(symbol_id: number): Reference[];
  find_references_by_name(name: string): Reference[];      // unresolved fallback

  // Import graph
  imports_of(path: string): Import[];                       // o que ESSE arquivo importa
  dependents_of(path: string): string[];                    // quem importa esse arquivo
  bfs_imports(start: string, max_hops?: number): string[];  // walk transitivo

  // Test mapping
  tests_for(source_path: string): { test_file, confidence }[];
  source_for(test_path: string): { source_file, confidence }[];

  // File metadata
  file_meta(path: string): FileMeta | null;
  list_files(opts?: { language?: string, modified_since?: number }): string[];

  // Diagnostics
  index_status(): {
    last_full_scan_at: number;
    files_indexed: number;
    files_failed: number;
    db_size_bytes: number;
    schema_version: number;
  };
};
```

### 4.2 Determinismo

Mesmo `code_index.db` + mesma query = mesmo resultado. Funciona em replay (`AGENTIC_CLI.md` princípio 7) **se** o index for snapshotado junto com a sessão (não é por default; `--snapshot-index` opt-in para sessões críticas, custo ~100MB extra).

### 4.3 Failure modes

| Caso | Comportamento |
|---|---|
| Index não inicializado | Funções retornam `[]` ou `null`; harness loga warning único por sessão |
| Index `parse_status: failed` em arquivo target | Funções operam sobre o que tem; arquivos failed são "ausentes" |
| Schema mismatch (binary atualizou, DB antigo) | Migration auto-executa; falha → reindex --clean recomendado |
| Query timeout (improvável em SQLite local) | Retorna parcial + warning |

---

## 5. Tools simbólicas expostas ao modelo

> **Decisão:** quais tools entram no catálogo §2.6 do `CONTRACTS.md` é discussão separada (próxima rodada). Esta seção lista os candidatos com schema; promoção a tool canônica passa pelo rubric `§2.5` e ADR `§2.6.8`.

### 5.1 `read_symbol` — substitui `read_file` para alvo simbólico

```yaml
input:
  symbol: string                  # nome ou FQN
  file?: string                   # disambiguação se nome ambíguo
  include_doc?: boolean           # default true: docstring/comment imediatamente acima
output:
  symbol: { name, kind, fqn, file, line_range }
  source: string                  # corpo do símbolo
  signature?: string
  doc?: string
metadata:
  writes: false
  idempotent: true
  failure_modes:
    - symbol.not_found
    - symbol.ambiguous            # múltiplos matches, exige `file`
    - index.unavailable
```

Custo típico: **~50-200 tokens** vs `read_file` médio de 500-3000 tokens. Ganho 5-20×.

### 5.2 `find_references` — onde Y é usada

```yaml
input:
  symbol: string
  file?: string                   # disambiguação
  ref_kind?: 'call'|'type'|'import'|'extends'|'implements'   # filtro
output:
  references: [
    { file, line, col, kind, surrounding_text }   # surrounding_text: ±2 linhas
  ]
  truncated: boolean
metadata:
  writes: false
  idempotent: true
```

Cap default 100 references; mais que isso → `truncated: true` e modelo refina query.

### 5.3 `outline_file` — esqueleto de um arquivo

```yaml
input:
  path: string
  include_internal?: boolean      # default false: só exports
output:
  symbols: [
    { name, kind, signature, line, visibility, parent? }
  ]
  loc: number
  imports_summary: string         # 1-line: "imports from 5 files (3 local, 2 external)"
metadata:
  writes: false
  idempotent: true
```

Custo: ~5-15% dos tokens de `read_file`. Use case: "antes de editar `auth.ts`, qual a estrutura?"

### 5.4 `imports_of` / `dependents_of` — grafo

```yaml
# imports_of
input:
  path: string
  hops?: number                   # default 1, cap 3
output:
  imports: [{ from_path, target_path, target_module, names[], is_external }]

# dependents_of
input:
  path: string
  hops?: number
output:
  dependents: [{ path, line, imported_names[] }]
```

Use case canônico: refactor playbook — `dependents_of("src/auth.ts")` lista os arquivos que precisam ser revisados ao mudar API.

### 5.5 Tools deferred / rejeitadas

- **`find_definition`** — coberto por `read_symbol(symbol)` se o output incluir `file + line_range`. Tool separada agrega pouco.
- **`semantic_search`** (busca por similaridade) — rejeitada (`ANTI_PATTERNS.md §2.2`).
- **`call_graph`** — interesting mas custo de implementação alto (resolução de polimorfismo, dynamic dispatch); v2.

---

## 6. Integração com `repo_map`

`repo_map` (`CONTEXT_TUNING.md §11`) é **projeção** sobre o code index. Pré-CODE_INDEX, repo_map era recomputado a cada `SessionStart` via tree-sitter scan; pós-CODE_INDEX vira query SQL.

### 6.1 Pipeline novo

```
[SessionStart - eager (orchestrated profile)]
  ↓
SELECT symbols WHERE visibility = 'export'
  GROUP BY file_path
  ORDER BY file_path
  ↓
Render como string canônica (§11.1 do CONTEXT_TUNING)
  ↓
Inject em [repo_map] section
```

Custo: ~10ms (vs ~10-30s da rebuild full). Cache breakpoint #3 (estável até next FS write).

### 6.2 Granularity revisitada

| Nível | Query |
|---|---|
| Mínimo | `WHERE visibility='export' AND kind IN ('function','class','interface')` |
| Médio | acima + `WHERE kind='method' AND parent.visibility='export'` + `signature` |
| Detalhado | tudo + private methods + types + import edges |

Modelo pode pedir level específico via tool `repo_map(level: "detailed", path_glob: "src/auth/**")`.

### 6.3 Profile-aware (recap do CONTEXT_TUNING §11.5)

| Profile | Repo map injection |
|---|---|
| `autonomous` | Lazy: tool `repo_map` quando modelo solicita |
| `orchestrated` | Eager: section em prompt; modelo small precisa evitar grep storm |
| `hybrid` | Eager no planner step; lazy nos executor steps |

---

## 7. Invalidação

### 7.1 Triggers

| Trigger | Escopo | Frequência típica |
|---|---|---|
| `PostToolUse` em `write_file`/`edit_file` | arquivo único | every edit |
| `PostToolUse` em `bash` com `writes:true` | tree hash diff scan | per bash call |
| FS watcher (opt-in) | arquivo único, debounced 500ms | per save externa |
| `SessionStart` stale check | full scan triggered se >5% files stale | per session |
| Schema migration | full rebuild | per binary upgrade |
| User command `agent code-index rebuild` | user-defined | manual |

### 7.2 Cache invalidation cascade

Quando arquivo `X` é re-indexado:

1. `DELETE FROM symbols WHERE file_path = X` → cascade pra `references_.target_symbol_id` (SET NULL)
2. `DELETE FROM references_ WHERE source_file = X`
3. `DELETE FROM imports WHERE source_file = X`
4. Re-parse → INSERT
5. Re-resolve **somente** references que tinham `target_symbol_name` definido em X (não toda a tabela)
6. Invalidar cache LRU de queries que tocavam X

Custo: O(arquivos que importam X) para passo 5. Em repo médio: ~10-100ms.

### 7.3 Race conditions

| Race | Resolução |
|---|---|
| Tool concorrente lê index enquanto re-index escreve | SQLite WAL mode garante reader/writer concurrent; reader vê snapshot pré-write |
| Dois `write_file` paralelos disparam reindex simultâneo | Lock per-file no harness antes de iniciar reindex; fila |
| `bash` muda 1k arquivos durante session | Batch detection; reindex full enfileirado, tools simbólicas retornam `index_stale` warning |

---

## 8. Privacy

### 8.1 Não-indexados por default

```toml
# Default exclude (não-configurável; só extendível)
exclude_default = [
  "node_modules/**",
  ".git/**",
  ".venv/**",
  "venv/**",
  "__pycache__/**",
  "dist/**",
  "build/**",
  "target/**",
  ".env*",                  # .env, .env.local, .env.production
  "**/*.key",
  "**/*.pem",
  "**/secrets/**",
  "**/.aws/**",
  "**/.ssh/**",
]
```

Lista alinhada com `SECURITY_GUIDELINE.md §6` (secret patterns) + filesystem zones que comumente armazenam credenciais.

### 8.2 Configurável

```toml
[code_index]
include = ["src/**", "tests/**", "lib/**"]
exclude = ["src/generated/**", "vendor/**"]
max_file_size_mb = 5             # arquivos > 5MB não indexados
follow_symlinks = false          # default false
respect_gitignore = true         # default true
```

### 8.3 Audit

Toda mudança em config dispara reindex. Audit em `index_meta`:

```sql
INSERT INTO index_meta VALUES ('config_version_<hash>', '<json>');
```

### 8.4 PII em código

Index armazena símbolos (nomes, paths, signatures) e references — **não** armazena conteúdo de strings literais nem comentários. Risco PII: nome de função/variável que vaza info (`getUserSSN_<numero>`). Aceitável: mesma exposição que repo já tem em git log. Sem redaction extra.

---

## 9. Performance budgets

> **Cross-ref:** [`PERFORMANCE.md §11`](./PERFORMANCE.md). Esta seção declara budgets; PERFORMANCE.md mede e regride.

### 9.1 Budgets canônicos

| Operação | p50 | p99 | Notas |
|---|---|---|---|
| Initial scan (3k files) | 15s | 45s | paralelo 4 workers |
| Incremental update (1 file) | 30ms | 100ms | parse + insert |
| `read_symbol` query | 5ms | 20ms | indexed lookup |
| `find_references` (≤100 refs) | 10ms | 50ms | indexed lookup |
| `outline_file` query | 8ms | 30ms | indexed lookup |
| `dependents_of` (1-hop) | 15ms | 60ms | indexed lookup |
| `dependents_of` (3-hops) | 100ms | 500ms | recursive CTE |
| Repo map render | 10ms | 50ms | aggregate query |
| Stale check (`SessionStart`) | 20ms | 100ms | indexed scan |

### 9.2 SLO violations

PR bloqueado se:
- Initial scan p50 cresce > 30%
- Incremental update p50 cresce > 50%
- Query p99 cresce > 100%

### 9.3 Profiling

`agent code-index profile <op>` roda operação X com tracing detalhado.

---

## 10. CLI

```
agent code-index status                        # estado: files indexed, last scan, db size
agent code-index rebuild [--clean] [--since]   # full rebuild
agent code-index query <sql>                   # SQL ad-hoc (read-only); cuidado!
agent code-index symbol <name>                 # busca simbólica
agent code-index refs <name>                   # find references
agent code-index outline <path>                # outline de arquivo
agent code-index imports <path> [--hops N]     # grafo
agent code-index check                         # valida consistência (FS vs index)
agent code-index profile <op>                  # benchmark de uma operação
```

`--json` flag em todos pra scripting.

---

## 11. Failure modes

| Code | Quando | Recovery |
|---|---|---|
| `index.parse_failed` | tree-sitter falha em arquivo X | `files.parse_status='failed'`; arquivo invisível ao index; warning |
| `index.parse_partial` | parse parcial (sintaxe errada parcial) | `parse_status='partial'`; symbols extraídos até onde foi possível |
| `index.schema_version_mismatch` | binary nova, DB antigo | migration auto; falha → user roda `rebuild --clean` |
| `index.stale_excessive` | > 5% files stale | warning em UI; sugestão de rebuild |
| `index.disk_full` | DB write falha | tools retornam `index_unavailable`; sessão prossegue degradada |
| `index.lock_timeout` | reindex preso > 30s | abort; estado preservado; user roda `agent code-index check` |
| `index.unavailable` | `code_index.db` não existe | tools simbólicas no-op; `repo_map` cai pra fallback grep-based |

Detalhamento operacional em [`FAILURE_MODES.md §16`](./FAILURE_MODES.md) — playbook de recovery por code, mensagens-template, audit footprint, queries de aggregate.

---

## 12. Limites declarados (v1)

- **Sem polymorphism resolution.** `obj.method()` aponta pra `target_symbol_name='method'` sem `target_symbol_id` quando há múltiplas classes com `method`. Modelo recebe lista, decide.
- **Sem cross-language references.** TypeScript chamando WASM em Rust não conecta. Cada linguagem é grafo isolado.
- **Sem dynamic imports.** `import(variableModule)`, `require(computed)` — não rastreados; ficam como references unresolved.
- **Sem macro expansion** (Rust, C/C++). Symbols dentro de macros podem não aparecer.
- **Test mapping é heurístico.** Confidence < 0.7 deveria ser tomado com ceticismo; mapeamento manual via `[code_index.test_mapping]` em config sobreescreve.
- **Sem branch awareness.** Index reflete working tree; mudança rápida de branch (git checkout) → stale até reindex incremental processar.
- **Sem snapshot por sessão.** Replay de sessão em FS modificado pode divergir; `--snapshot-index` é opt-in extra.

---

## 13. Eval e regressão

### 13.1 Eval de qualidade do index

Corpus em `evals/code_index/`:

```yaml
- id: typescript-class-method-001
  fixture: fixtures/sample-ts-app/
  query: { kind: 'find_references', name: 'login' }
  expected:
    refs_count: 7
    files: ['src/api/auth.ts', 'tests/auth.test.ts', ...]
    no_false_positives_in: ['docs/**', 'examples/**']
```

PR que mexe em parser ou schema roda este corpus em CI. Falha = bloqueio.

### 13.2 Eval de impacto em workflows

Métrica chave: **tokens economizados por uso de tool simbólica vs `read_file`/`grep`** em workflows reais.

```sql
-- "Quanto economizamos com read_symbol vs read_file em refactor?"
SELECT
  AVG(CASE WHEN tool_name = 'read_symbol' THEN tokens_out END) AS avg_symbol,
  AVG(CASE WHEN tool_name = 'read_file' THEN tokens_out END) AS avg_file,
  (avg_file - avg_symbol) * 1.0 / avg_file AS reduction_ratio
FROM tool_calls
WHERE session_id IN (
  SELECT id FROM sessions WHERE playbook = 'refactor'
);
```

Threshold para celebrar: redução > 60%. Sem essa métrica, tools simbólicas viram cargo cult — modelo poderia chamar `read_file` e funcionar igual.

---

## 14. Insight final

Code index não é "embedding mais barato". É **a estrutura do repo apresentada como dado queryable**, em vez de string que vai no prompt.

A diferença prática:
- **String-no-prompt** (status atual sem este doc): repo map é renderizado, modelo lê. Qualquer query simbólica vira grep + heurística.
- **Subsistema queryable** (com este doc): modelo pergunta `find_references("login")`, recebe lista exata. Sem grep storm, sem false positives, sem ler arquivos inteiros.

A inversão consistente do `AGENTIC_CLI` aplica aqui: harness faz o trabalho de meta-cognição estrutural (parsing, indexing, resolution); modelo só toma decisões sobre código real. Sem este subsistema, modelo é forçado a fazer trabalho de IDE em prosa — caro e impreciso.

Spec sem este doc: code retrieval é folclore. Com este doc: é **infraestrutura mensurável** com schema, budget, e eval.
