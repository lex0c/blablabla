# TREE_SITTER

Parser generator + runtime de parsing **incremental, robusto e generalizado** projetado pra ferramentas que rodam em editor enquanto o usuário digita. Hoje é a base de syntax highlighting, structural code intel e static analysis em Atom, Neovim, Helix, GitHub web, Zed e em compiladores/linters modernos.

> **Tese central:** parser não precisa ser perfeito; precisa ser **rápido, recuperável de erro, e re-utilizar trabalho** entre edições. Tree-sitter foi desenhado pra esse trade-off, não pro de compilador.

---

## 1. O que é (e o que não é)

Tree-sitter é:
- **Parser generator** — você escreve grammar em DSL (`grammar.js` em JavaScript), ele gera parser em C.
- **Runtime de parsing** — biblioteca C com bindings em Rust, Go, Python, Node, Ruby, Swift, Java, Kotlin, WASM.
- **Query engine** — DSL S-expression pra navegar AST sem código imperativo.

Tree-sitter **não é**:
- Compilador (não faz type checking, name resolution, codegen).
- Lexer puro (token + AST integrados; não há separação clássica).
- LL/LR clássico (é GLR; aceita ambiguidade local resolvida por precedência declarada).
- Validador semântico (sintaxe correta ≠ programa correto).

---

## 2. Modelo de parsing

### 2.1 GLR generalizado

Tree-sitter usa **GLR** (Generalized LR) com lookahead arbitrário via lexer dinâmico. Trade-off:

| Característica | Implicação |
|---|---|
| Aceita ambiguidade local | linguagens context-sensitive (C++ template `<`, bash quoting) parseiam |
| Backtracking limitado | resolve com `prec`, `prec.left`, `prec.right`, `prec.dynamic` na grammar |
| Lexer integrado | tokens reconhecidos no contexto sintático (não há fase léxica separada que erra) |
| External scanners em C | quando regex não basta (heredocs Python, indentação significativa) escreve scanner manual |

Resultado: linguagens "impossíveis" de parsear classicamente (Python indent, Ruby heredoc, JSX dentro de TS) são tratáveis.

### 2.2 Error recovery

Pilar do design. Editor está sempre vendo código sintaticamente inválido (usuário no meio de digitar). Tree-sitter:

- **Nunca falha o parse.** Sempre produz árvore.
- Nodes inválidos viram `ERROR` ou `MISSING` no AST.
- Resto da árvore continua válido.
- Ferramentas (highlighter, linter) seguem funcionando.

Compare com parser clássico que aborta no primeiro erro: tree-sitter mantém o editor utilizável.

### 2.3 Incremental

Parse inicial é O(n). Edit subsequente é **O(log n + d)** onde `d` é o tamanho da edit. Implementação:

- Cada subtree tem hash + range
- Edit invalida apenas subtrees overlapping
- Re-parse só do range afetado, reusa o resto
- Bom o suficiente pra rodar a cada keystroke em arquivos de 100k LOC

---

## 3. Arquitetura

```
grammar.js (DSL JS)
    │
    │  tree-sitter generate
    ↓
parser.c (gerado, ~100k linhas típico)
    │
    │  cc compila
    ↓
libparser.so / libparser.a
    │
    │  linka via FFI
    ↓
Aplicação (Rust / Go / Node / Python / ...)
    │
    ↓
TSTree (AST) + TSQueryCursor (queries)
```

### 3.1 grammar.js (a DSL)

JavaScript DSL declarativo. Exemplo simplificado:

```js
module.exports = grammar({
  name: 'mylang',

  rules: {
    source_file: $ => repeat($._statement),

    _statement: $ => choice(
      $.assignment,
      $.if_statement,
      $.expression_statement,
    ),

    assignment: $ => seq(
      field('left', $.identifier),
      '=',
      field('right', $._expression),
      ';'
    ),

    _expression: $ => choice(
      $.identifier,
      $.number,
      $.binary_expression,
    ),

    binary_expression: $ => prec.left(1, seq(
      $._expression,
      choice('+', '-', '*', '/'),
      $._expression
    )),

    identifier: $ => /[a-zA-Z_][a-zA-Z0-9_]*/,
    number: $ => /\d+/,
  }
});
```

Convenções:
- `_name` (underline prefix) = node "supressed" (não aparece no AST, só usado pra factoring)
- `field('name', ...)` = acessor nomeado (ergonomia em queries)
- `prec`, `prec.left`, `prec.right` = resolver ambiguidade
- `token(...)` = força tratamento como token atômico
- `seq`, `choice`, `repeat`, `optional` = combinators padrão

### 3.2 External scanners

Quando regex não dá conta (heredocs, indentação Python, end-of-input tokens), escreve scanner em C exposto via 4 funções:

```c
void *tree_sitter_<lang>_external_scanner_create(void);
void  tree_sitter_<lang>_external_scanner_destroy(void *);
unsigned tree_sitter_<lang>_external_scanner_serialize(void *, char *);
void  tree_sitter_<lang>_external_scanner_deserialize(void *, const char *, unsigned);
bool  tree_sitter_<lang>_external_scanner_scan(void *, TSLexer *, const bool *valid_symbols);
```

É onde as linguagens reais ficam complicadas. tree-sitter-python tem scanner pra indent/dedent; tree-sitter-bash tem pra heredocs.

---

## 4. Query language (S-expression DSL)

Em vez de visitor patterns imperativos, tree-sitter expõe **queries declarativas** em S-expression.

Exemplo: encontrar todas as chamadas de função em JavaScript:

```
(call_expression
  function: (identifier) @function-name
  arguments: (arguments) @args)
```

Predicates:

```
((identifier) @const
 (#match? @const "^[A-Z_]+$"))
```

Captura todo identifier que matches regex (constantes ALL_CAPS).

Use cases:
- Syntax highlighting (`highlights.scm`)
- Indent rules (`indents.scm`)
- Folding (`folds.scm`)
- Textobjects pra Neovim (`textobjects.scm`)
- Refactoring / linting (queries customizadas em código)

Performance: query runner é otimizado; matches em arquivos grandes saem em ms.

---

## 5. Performance

### 5.1 Latência

| Operação | Típico |
|---|---|
| Parse inicial (10k LOC JS) | 5–20ms |
| Re-parse incremental após edit | < 1ms |
| Query em arquivo de 10k LOC | 1–5ms |
| Highlight de arquivo inteiro | 5–10ms |

Suficiente pra rodar em toda keystroke sem perceptível.

### 5.2 Custo binário

| Componente | Tamanho |
|---|---|
| Runtime libtree-sitter | ~100KB |
| Parser por linguagem | 200KB–2MB compilado |
| Total típico pra editor com 20 langs | 10–40MB |

Não é leve. Pra CLI tool pequena, pode ser percebido. Há esforço em WASM-based deploy (`tree-sitter-wasm`) com size menor.

### 5.3 Memory

AST é mantido em memória. Arquivo de 100k LOC = AST de ~10–50MB típico. Aceitável pra editor, pesado pra ambiente embarcado.

---

## 6. Casos de uso

### 6.1 Casos canônicos

| Caso | Por quê tree-sitter |
|---|---|
| Syntax highlighting estrutural | Em vez de regex, AST real → highlight de `function` vs `class` é trivial |
| Code folding inteligente | folds.scm declara o que é foldable |
| Refactoring (rename, extract) | AST permite mutação consciente |
| Semantic search (grep estrutural) | "encontre todas chamadas de `foo()` no contexto X" via query |
| Linters / static analysis lightweight | parse barato, query expressiva |
| Diff estrutural | comparação AST > comparação linha (ex: difftastic) |
| LSP fallback / hybrid | LSP completo + tree-sitter pra editor responsivo enquanto LSP carrega |

### 6.2 Casos limítrofes

| Caso | Avaliação |
|---|---|
| Compilador full | Não. Falta semântica, type check, codegen. Use só pra front-end frágil. |
| Security parsing (bash, SQL) pra policy gates | Funciona, mas considere parser whitelist (mais restrito) — ver §10 |
| Format-preserving rewrites | Funciona com cuidado; ranges precisam ser mantidos exatos |
| Linguagens com macro expansion (Rust, Scheme) | Tree-sitter vê pré-expansão; análise pós-expansão precisa de outra camada |

---

## 7. Limitações

### 7.1 GLR não é mágica

Ambiguidade global persistente (não resolvível por precedência local) → você reescreve grammar. C++ `<` (template vs less-than) exige scanner external com lookahead profundo.

### 7.2 Erro de grammar bug é silencioso

Se a grammar tem ambiguidade não declarada, parser ainda produz árvore — só que pode ser a árvore errada. Testes de grammar são essenciais.

### 7.3 Versionamento de grammar

Grammars de linguagens populares (JS, TS, Rust, Go) mudam ao longo do tempo: node names renomeiam, hierarquia muda, novos nodes aparecem. Ferramentas que dependem de node names específicos quebram silenciosamente em upgrade.

Mitigação: pin a versão da grammar; teste contra fixtures conhecidos antes de atualizar.

### 7.4 Não substitui análise semântica

Tree-sitter te dá **estrutura**. Saber que `x` em `x + 1` é variável local vs global é semantic analysis. Pra isso: LSP, compilador, ou tooling adicional (scope queries são possíveis mas limitadas).

### 7.5 Performance em arquivos enormes

Parse de arquivo de 10MB+ é segundos, não ms. Pra esses casos, parsing on-demand por região é necessário.

---

## 8. Quando NÃO usar

| Cenário | Alternativa |
|---|---|
| Você precisa de validação semântica completa | Compilador ou LSP real |
| Você está parsing DSL pequena e estável que você controla | Hand-written recursive descent é mais simples |
| Parsing **adversarial** (input controlável por atacante, decisão de segurança) | Parser whitelist com Refuse agressivo (§10) |
| Embedded device com KB de memória | Lex/yacc clássico, mais leve |
| Você só precisa extrair info simples (regex resolve) | Regex, com testes |
| Linguagem completamente irregular (TeX, esoteric) | Hand-written ou specialized parser |

---

## 9. Alternativas e trade-offs

| Alternativa | Quando ganha |
|---|---|
| **Hand-written recursive descent** | DSL pequena, controle total, sem dep |
| **ANTLR4** | Grammars formais com codegen pra muitas linguagens, mas runtime mais pesado e error recovery inferior |
| **lex/yacc, bison/flex** | Quando você precisa lookahead clássico LR(1) e perf máxima em batch (compilador) |
| **lark (Python), nom (Rust), chumsky (Rust)** | Parser combinators; control fino; sem geração externa |
| **PEG (pest, peg.js)** | Ambiguidade resolvida por ordem; menos potente que GLR; mais previsível |
| **Tree-sitter** | Editor / IDE / refactoring tools / linters interativos |

Heurística: se a ferramenta roda **durante edição interativa**, tree-sitter ganha. Se roda **uma vez em batch** (build, CI), ANTLR ou hand-written podem ser melhores.

---

## 10. Considerações de segurança

Tree-sitter foi desenhado pra ambiente confiável (editor do próprio usuário). Em contexto adversarial (parsing de input não-confiável pra decisão de segurança), atente:

### 10.1 Parsing ≠ validação

Tree-sitter te diz "isso é sintaxe válida de bash". **Não** te diz "isso é seguro". `rm -rf /` é bash perfeitamente válido.

Quem usa tree-sitter pra capability resolver (ver `AGENTIC_CLI/PERMISSION_ENGINE.md` §5.2) precisa de **camada de policy em cima do AST**, não trustar o parse por si.

### 10.2 Cobertura é tentação

"Tree-sitter parseia tudo de bash" vira "vou suportar tudo". Mas suportar full bash em decisão de segurança = suportar full bash em superfície de ataque.

**Alternativa preferida pra security gates:** parsear com tree-sitter, mas **rejeitar ativamente** qualquer construção fora de whitelist pequena. AST serve pra reconhecer shape, não pra dar permissão.

```
[shape reconhecida ∧ comandos em registry conhecido] → resolve capabilities
[qualquer outra coisa] → Refuse → confirma com humano
```

### 10.3 Parser bugs

Tree-sitter runtime tem histórico bom; grammars específicas ocasionalmente têm bugs em edge cases. Fuzz a integração com input adversarial. Crash do parser não pode escalar pra crash do sistema.

### 10.4 External scanner é C não-sandboxed

External scanner é C compilado, linked no seu processo. Bug → memory corruption → comprometimento. Auditar scanners de grammars third-party antes de usar em contexto de segurança.

---

## 11. Pegadinhas práticas

### 11.1 Node names "óbvios" não são estáveis

Grammar de JS mudou `arrow_function` → `arrow_function_expression` em algum point release. Pin versão ou aceite refactor recorrente.

### 11.2 Anonymous nodes vs named nodes

Em `(if_statement "if" (condition) (block))`, o `"if"` é anonymous node (terminal literal). Queries diferentes pra cada categoria; trips beginners.

### 11.3 Children iteration

API tem `child(i)`, `named_child(i)`, e `field("name")`. Misturar produz bugs sutis. Prefira `field` quando grammar declara field (mais robusto a refactoring).

### 11.4 Cursor é leaky

`TSTreeCursor` é stateful e mantém ponteiros internos. Não compartilhe entre threads. Recreate em vez de reset se cruzar boundary.

### 11.5 Bindings têm latência de update

Tree-sitter core release → bindings em N linguagens precisam atualizar. Bindings em Python e Go costumam ficar atrás. Pra produção, pin tudo (runtime + binding + grammars).

### 11.6 WASM tem limitações

WASM build do parser não suporta external scanners em C arbitrário. Linguagens com scanner complexo (Python indent, Ruby) podem não ter parity 100% no WASM.

---

## 12. Ecossistema

### 12.1 Grammars consolidadas

JavaScript, TypeScript, TSX, Python, Ruby, Go, Rust, C, C++, Java, Kotlin, Swift, Bash, JSON, YAML, TOML, Markdown, HTML, CSS, SCSS, GraphQL, SQL (parcial), Lua, PHP, OCaml, Haskell, Scala, Elixir, Erlang, Zig, Nix, Dockerfile, regex.

Lista canônica: `github.com/tree-sitter` org + topic `tree-sitter-parser` em GitHub.

### 12.2 Consumidores principais

- **GitHub web** — highlight em arquivos
- **Atom** (origem) — descontinuado mas onde nasceu
- **Neovim** (built-in desde 0.5) — highlight, textobjects, folding
- **Helix editor** — parsing nativo
- **Zed editor** — heavy user
- **Difftastic** — diff estrutural
- **GitHub Semantic** (descontinuado) — code intel
- **Linters / formatters** — várias dependências indiretas

### 12.3 Ferramentas auxiliares

- `tree-sitter` CLI — gerar, testar, debug grammar
- `tree-sitter-cli playground` — REPL interativo pra grammar dev
- `tree-sitter-bash --highlight file.sh` — uso direto pra debug

---

## 13. Quando tree-sitter é a escolha certa: resumo

Sim, use tree-sitter se:
- Ferramenta roda interativamente em editor / IDE
- Você precisa de parsing **robusto a erro** (input parcial, em-edição)
- Múltiplas linguagens com superfície similar (highlight, refactor)
- Performance importa por keystroke
- Existe grammar madura pra sua linguagem

Não use tree-sitter se:
- Validação semântica é o ponto (use compilador / LSP)
- Contexto adversarial sem camada de policy em cima (use whitelist)
- DSL pequena/estável sob seu controle (hand-write)
- Constraints de tamanho/memória apertados (embed)

A regra prática: **tree-sitter é frontend de ferramentas que rodam em editor**. Quando o use case não é esse, há quase sempre alternativa melhor.

---

## 14. Referências

- Repo: `github.com/tree-sitter/tree-sitter`
- Paper original (Max Brunsfeld, 2018): "Tree-sitter — a new parsing system for programming tools"
- Docs: `tree-sitter.github.io`
- Ecosystem: `github.com/tree-sitter` (grammars oficiais)
- Neovim tree-sitter docs: `:help treesitter`
