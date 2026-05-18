# TREE_SITTER_SHELL

Uso prático de **`tree-sitter-bash`** pra parsing de shell em contexto de ferramentas — com foco honesto em onde funciona, onde falha, e como usar em **decisão de segurança** (capability resolver, policy gate, command lint) sem entregar bypass de graça.

Companion de `TREE_SITTER.md` (visão geral) e `AGENTIC_CLI/PERMISSION_ENGINE.md` §5.2 (uso concreto em capability resolver).

> **Tese:** parsear bash é menos sobre "compreender bash" e mais sobre "reconhecer rapidamente o que você não vai conseguir compreender com segurança". Tree-sitter-bash é excelente pra primeira metade; o resto exige disciplina.

---

## 1. Por que bash é uniquely awful pra parsear

Bash não é hostil por design, é hostil por **acumulação histórica**. Cada construção foi adicionada com o lexer que existia, e nenhuma quebra de compatibilidade aconteceu.

| Característica | Por que dói |
|---|---|
| Lexing context-sensitive | `[[` em contexto condicional ≠ `[[` em contexto comum; mesmo token, gramática diferente |
| Quoting com 4 variantes | `'...'`, `"..."`, `$'...'` (ANSI-C), `$"..."` (locale); cada uma com regras de escape distintas |
| Word splitting + glob | Mesmo `*` é wildcard de glob, multiplicação aritmética em `(( ))`, ou repetição em regex `=~` |
| Expansion recursivo | `$(cmd)` recursa pro top-level shell; `${var}` tem ~12 sub-variantes; brace expansion `{a,b}` é resolvida pré-glob |
| Heredocs com lookahead | `cat <<EOF\n...\nEOF` exige scanner com lookahead arbitrário |
| Process substitution | `<(...)`, `>(...)` cria FIFO/FD não-determinístico |
| Comandos sem keyword | `function` é opcional pra definir funções; mesmo símbolo `()` define grupo, subshell ou função |
| Arrays | indexed (`a[0]`) e associative (`a[key]`); expansion `${a[@]}` vs `${a[*]}` muda semântica de quoting |
| Aliases | runtime macro expansion — texto pre-lexer; **tree-sitter não vê** |
| `eval` | texto interpretado runtime — **tree-sitter não vê** |
| POSIX sh subset vs bash extensions | `[[ ]]`, `(( ))`, arrays, `function` keyword — só bash. dash/ash quebram |

A consequência: qualquer parser que afirma "entender bash" entende **bash em estado parcial**. Tree-sitter-bash não é exceção — só é honesto sobre isso.

---

## 2. tree-sitter-bash: o que é

Repo: `github.com/tree-sitter/tree-sitter-bash`
Maintainers: comunidade tree-sitter; commits estáveis.
Estado: maduro; usado em GitHub, Neovim, Helix, Zed.

Cobre:
- Sintaxe POSIX sh completa
- Extensões bash 5.x (associative arrays, `[[ ]]`, `(( ))`, `${var@operator}`, etc.)
- Heredocs / here-strings (via external scanner em C)
- Process substitution
- Command substitution `$(...)` e backtick legacy
- Brace expansion (parcial — sintaxe reconhecida, não expandida)
- Funções (com e sem keyword)
- Pipelines, sequences (`;`, `&`, `&&`, `||`)
- Redirections completas (FD numbers, `&>`, `<<<`)

Não cobre:
- ksh-specific (`[[ -ot ]]`, coprocesses ksh-style)
- zsh-specific (parameter flags, globbing extensions, `=>`)
- fish (completamente diferente)
- bash 5.x **alias expansion** (resolvido pré-parse pelo shell; tree-sitter vê o texto literal)
- `eval` runtime (texto, não AST)

---

## 3. Taxonomia de nodes (os que importam)

A grammar gera ~80 node kinds. Os relevantes pra capability resolver:

### 3.1 Estruturais

| Node | O que é |
|---|---|
| `program` | root |
| `command` | comando simples: name + args + redirections |
| `pipeline` | `cmd1 \| cmd2 \| cmd3` |
| `list` | `cmd1 ; cmd2`, `cmd1 && cmd2`, `cmd1 \|\| cmd2` |
| `subshell` | `(cmd)` |
| `compound_statement` | `{ cmd; }` |
| `function_definition` | `func() { ... }` |
| `if_statement`, `while_statement`, `for_statement`, `case_statement` | controle de fluxo |

### 3.2 Comando

```
(command
  name: (command_name (word))
  argument: (word)
  argument: (string)
  ...)
```

`command.name` é sempre o primeiro filho; restantes são `argument`. Importante: `command_name` é wrapper sobre `word` — extrair valor exige descer.

### 3.3 Quoting e expansion

| Node | Exemplo |
|---|---|
| `word` | literal sem aspas: `foo` |
| `string` | `"texto $var"` — pode conter `string_expansion` e `expansion` |
| `raw_string` | `'texto literal'` — sem expansion |
| `ansi_c_string` | `$'texto\n'` — ANSI-C escapes |
| `expansion` | `${var}`, `${var:-default}`, `${var/pat/rep}` |
| `simple_expansion` | `$var` (sem chaves) |
| `command_substitution` | `$(cmd)` ou backtick |
| `arithmetic_expansion` | `$(( expr ))` |
| `process_substitution` | `<(cmd)` ou `>(cmd)` |

### 3.4 Redirections

```
(file_redirect
  descriptor: (file_descriptor)?
  destination: (word))

(heredoc_redirect
  (heredoc_start)
  (heredoc_body))
```

### 3.5 Constructs adversariais (red-flag nodes)

| Node | Por que vermelho |
|---|---|
| `command_substitution` | recursão em shell context arbitrário |
| `process_substitution` | side-effect process fora do AST principal |
| `expansion` com `operator` complexo | `${var@P}`, `${!prefix*}` — runtime expansion |
| `arithmetic_expansion` | aritmética pode ler/atribuir variáveis |
| `heredoc_body` com `expansion` interno | string com substituição = não-literal |
| Função sendo definida | scope dinâmico; redefinição de built-in |
| Pipeline com `eval` em qualquer posição | game over pra análise estática |

---

## 4. External scanner (heredocs)

tree-sitter-bash usa external scanner em C pra heredocs. Funções:

- Scanner detecta `<<DELIM` e captura body até linha que é exatamente `DELIM` (ou `\tDELIM` se `<<-`).
- Body é exposto como nó `heredoc_body`.
- Substituição de `$var` dentro do body é parseada **se** delim não-quotado (`<<EOF`); body fica literal **se** delim quotado (`<<'EOF'`).

Isso é a parte mais frágil do parser. Em arquivo grande com heredocs aninhados o scanner pode estourar buffer interno (já corrigido em versões recentes, mas histórico de bug).

Implicação prática: **fuzz com heredocs adversariais** se for usar pra security gate.

---

## 5. O que tree-sitter NÃO te diz sobre o comando

Mesmo com AST perfeito, há informação que só existe em runtime:

| Coisa | Por que |
|---|---|
| Valor de `$VAR` | depende do env no momento do exec |
| Resultado de `$(cmd)` | depende do que o sub-cmd retorna |
| Path resolvido de `~user` | depende do passwd |
| Glob expandido | depende do FS no momento |
| Alias expandido | depende do shell rc |
| `command_not_found_handle` hook | função user-definida |
| Function call resolution | função pode shadowar built-in (`function rm() { ... }`) |
| `PATH` lookup | qual binário é `curl` depende de `$PATH` |

Isso é a fronteira. Tudo que depende de runtime → seu resolver tem que tratar como **não-determinístico** (capability conservadora ou Refuse).

---

## 6. Pattern: extração de comandos de pipeline

Use case mais comum em capability resolver: dado `cmd1 | cmd2 && cmd3 > file`, extrair `[cmd1, cmd2, cmd3]` + redirections.

Pseudo-código (linguagem-agnóstico):

```
fn extract_commands(node) -> list[CmdInfo]:
    result = []
    walk(node):
        case "command":
            name = first_child_with_field("name").text()
            args = [arg.text() for arg in children_with_field("argument")]
            redirects = collect_redirects(node)
            result.append(CmdInfo(name, args, redirects))
        case "command_substitution" | "process_substitution":
            mark_dynamic_context()
            recurse_into(node)
        case "expansion" with non-literal operator:
            mark_dynamic_context()
        case "function_definition":
            register_local_function(name)
        case "eval" as command_name:
            return [REFUSE]
    return result
```

Saída tipada:

```rust
struct CmdInfo {
    name: String,                // "rm"
    args: Vec<ArgInfo>,
    redirects: Vec<RedirectInfo>,
    confidence: Confidence,
    dynamic_context: bool,       // dentro de $(...) ou expansion não-literal
}

enum ArgInfo {
    Literal(String),             // "foo.txt"
    Quoted(String),              // "'foo bar'"
    DynamicExpansion,            // $var, $(...) — não resolvível estaticamente
    UnknownExpansion,            // ${var@P} etc
}
```

Aí o resolver olha `CmdInfo` e decide capabilities:
- `name="rm"` + `args=[Literal("/tmp/x")]` → `delete-fs(/tmp/x)` com `confidence=high`
- `name="rm"` + `args=[DynamicExpansion]` → `delete-fs(*)` com `confidence=low`
- `name="eval"` → `Refuse`

---

## 7. Pattern: detecção de adversarial constructs

Conjunto de checks que devem rodar **sempre** sobre o AST antes de aceitar qualquer resolução:

```
fn is_safe_shape(node) -> bool | Refuse:
    if contains(node, kind="command_name", text in BLOCKLIST_COMMANDS):
        return Refuse("blocklist_command")
    
    if depth_of("command_substitution") > 1:
        return Refuse("nested_command_substitution")
    
    if any("expansion".operator in DYNAMIC_OPERATORS):
        return Refuse("dynamic_expansion")
    
    if any("process_substitution"):
        return Refuse("process_substitution")
    
    if any("function_definition"):
        return Refuse("function_redefinition")
    
    if any("heredoc_body" contains "expansion"):
        confidence = low
    
    if pipeline_contains(node, "tee", "sh", "bash") with stdin redirect:
        return Refuse("pipe_to_shell")
    
    return ok
```

`BLOCKLIST_COMMANDS` mínimo: `eval`, `exec`, `source`, `.`, `trap`, `alias`, `shopt`, `set` (com flags perigosos).

`DYNAMIC_OPERATORS` em `${var@OP}`: `@P` (prompt expansion), `@A` (assign), `@K` (key), `!prefix*` (indirect), etc.

---

## 8. Performance pra security gate

| Operação | Latência típica |
|---|---|
| Parse de comando de 100 chars | < 1ms |
| Parse de script de 1k linhas | 5–20ms |
| Query simples sobre AST de comando curto | < 1ms |
| Cold start: load grammar | 10–50ms (uma vez) |

Aceitável pra hot path de tool call. Cold start amortizado.

Cuidado: parser carrega ~500KB de grammar compilada. Em CLI tool com binary size apertado, considerar `tree-sitter-wasm` ou hand-rolled lexer pra cases simples.

---

## 9. O pattern "shape recognition + Refuse"

Este é o pattern correto pra security gate. Não é "compreenda bash"; é "**reconheça uma forma simples ou recuse**".

### 9.1 Definir whitelist de shapes

Shapes que valem a pena suportar:

```
SHAPE_SIMPLE_COMMAND    := command_name (word | string | raw_string)*
SHAPE_SIMPLE_PIPELINE   := SHAPE_SIMPLE_COMMAND ('|' SHAPE_SIMPLE_COMMAND)*
SHAPE_SIMPLE_SEQUENCE   := SHAPE_SIMPLE_PIPELINE (('&&' | '||' | ';') SHAPE_SIMPLE_PIPELINE)*
SHAPE_WITH_REDIRECT     := SHAPE_SIMPLE_SEQUENCE (file_redirect_with_literal_target)?
```

### 9.2 Walk o AST, valide cada nó

```
fn validate_shape(node):
    match node.kind:
        "program" | "list" | "pipeline" | "command" | "word" 
            | "string" | "raw_string" | "file_redirect" | "&&" | "||" | ";"
            → recurse_children
        any other kind
            → Refuse(f"unsupported_shape: {node.kind}")
```

Strings com `expansion` interno → confidence=medium, mas ok. Strings com `command_substitution` interno → Refuse.

### 9.3 O ganho

Você cobre ~70% dos comandos que LLM gera (a maioria é `npm test`, `git status`, `bun run x`, `grep foo file`). Os 30% restantes — heredocs, eval, process substitution, função inline — viram confirm humano. Que é exatamente onde deveriam.

Em troca:
- Parser de uso é simples e auditável
- Tree-sitter faz heavy lifting de tokenização correta
- Você não reimplementa bash
- Adversário não escapa por construção exótica — escapa = Refuse

---

## 10. Limitações específicas pra security

### 10.1 Aliases

`alias rm='echo deletado'` → user define alias; tree-sitter vê `rm` literal, não sabe que vai expandir. **Sandbox + scrub_env de PS1/PROMPT_COMMAND** ajuda; aliases em `.bashrc` requerem hide_paths em §9 do PERMISSION_ENGINE.

### 10.2 `command_not_found_handle`

Bash chama função user-definida quando comando não existe. Tree-sitter não vê. Sandbox que não execute `.bashrc` mitiga (`bwrap` com `--unsetenv BASH_ENV` + readonly `/etc`).

### 10.3 PATH manipulation

`PATH=/tmp:$PATH rm foo` → `rm` é `/tmp/rm` se existir. Tree-sitter vê assignment + command, mas não resolve qual binário roda. Resolver deve: detectar PATH em assignment prefix → Refuse ou degradar pra capability conservadora.

### 10.4 Bash version drift

Bash 5.2 adicionou `BASH_REMATCH` formato novo, `${var@K}`, etc. Grammar atualiza, mas seu resolver pode não. Pin versão + teste.

### 10.5 Shell != bash

Se policy permite `sh`, `dash`, `ash` (Alpine), `ksh`, `zsh`, tree-sitter-bash **pode parsear errado**. POSIX sh subset funciona; extensions diferem.

Mitigação: identificar shell pelo shebang ou pelo command name; parsear com grammar apropriada (`tree-sitter-fish` existe; `tree-sitter-zsh` parcial).

---

## 11. Linux vs macOS: shell landscape

A parte que parece detalhe e morde forte. Mesmo binário (`bash`), mesma sintaxe geral, comportamentos diferentes por plataforma. Capability resolver que ignora isso ganha bypass de graça.

### 11.1 Shells default por plataforma

| Plataforma | `/bin/sh` é | Shell interativo default | Bash version |
|---|---|---|---|
| Ubuntu/Debian | `dash` | `bash` | 5.x |
| Arch | `bash` | `bash` (ou zsh, fish — user) | 5.x |
| Fedora/RHEL | `bash` | `bash` | 5.x |
| Alpine | `ash` (BusyBox) | `ash` ou `bash` se instalado | bash 5.x se instalado |
| macOS ≤ 10.14 (Mojave) | `bash` 3.2 | `bash` 3.2 | **3.2.57** (GPLv3 avoidance) |
| macOS ≥ 10.15 (Catalina) | `bash` 3.2 | `zsh` 5.x | 3.2.57 default; Homebrew traz 5.x em path separado |
| WSL Ubuntu | `dash` | `bash` | 5.x |
| FreeBSD | `sh` (Almquist-derived) | `csh` ou `sh` | bash 5.x se instalado |

Implicações:

- **macOS default bash é 3.2.57.** Sem `[[ ]]` com `=~` em alguns casos, sem associative arrays (`declare -A`), sem `${var^^}` (case conversion), sem `mapfile`/`readarray`. Tree-sitter-bash 5.x parser parseia 3.2 sem erro, mas seu resolver pode **assumir features 5.x ausentes**.
- **Ubuntu `/bin/sh` é dash.** Script com shebang `#!/bin/sh` em Ubuntu **não** roda como bash. `[[ ]]` falha; `function foo` falha; arrays falham. Se você parseia com tree-sitter-bash, **AST diz "OK" mas runtime falha** — capability resolver pode autorizar comando que nem executa, levando user a tentar variantes.
- **WSL é Linux nativo dentro de Windows.** Behavior é Linux puro; não confunda com Windows native shell.

### 11.2 Coreutils: GNU vs BSD

A diferença mais explorável. **Mesmo nome de comando, flags diferentes.**

| Comando | GNU (Linux) | BSD (macOS default) |
|---|---|---|
| `ls -A` | hidden exceto `.`/`..` | idem |
| `ls --color` | suportado | **não suportado** (`-G` em vez) |
| `sed -i` | `sed -i 's/.../.../' file` | `sed -i '' 's/.../.../' file` (backup ext obrigatória) |
| `grep -P` | PCRE suportado | **não suportado** |
| `grep -r --include` | suportado | suportado mas semantics diferentes |
| `xargs -r` | "run if non-empty" | **não suportado** |
| `find . -printf` | suportado | **não suportado** |
| `readlink -f` | resolve symlinks recursivamente | **não suportado**; `-f` é "force overwrite stdout flush" |
| `stat -c '%n'` | format string | **não suportado**; `stat -f '%N'` |
| `date -d '...'` | parse string | **não suportado**; `date -j -f` |
| `tar --owner=root` | suportado | **não suportado** (BSD tar) |
| `cp --parents` | mantém dir tree | **não suportado** |
| `cut -d' ' -f1` | igual | igual |
| `mktemp -t prefix` | prefix no nome | template obrigatório (`mktemp -t prefix.XXXXXX`) |

macOS tem alternativas via Homebrew (`brew install coreutils gnu-sed`) com prefix `g`: `gsed`, `gls`, `greadlink`. Mas isso é opt-in do user.

Impacto no resolver:
- `rm -rf /tmp/foo` → idêntico GNU/BSD, ok.
- `sed -i 's/x/y/' file` → **sucesso em Linux, erro em macOS default**. Resolver detecta `sed -i` sem backup-ext → flag `bsd_incompatible` ou degrade.
- `readlink -f` → no resolver, capability `read-fs` em algum path; mas em macOS o `-f` significa outra coisa. Pode resolver pra capability errada.

### 11.3 PATH e binários

Linux:
- GNU coreutils em `/usr/bin/`
- Homebrew em `/home/linuxbrew/.linuxbrew/bin/` (raro)
- Nix em `/nix/store/...`

macOS:
- Sistema (BSD): `/bin/`, `/usr/bin/`
- Homebrew (Intel): `/usr/local/bin/`
- Homebrew (Apple Silicon): `/opt/homebrew/bin/`
- MacPorts: `/opt/local/bin/`

Capability resolver precisa **resolver $PATH** pra saber qual binário será executado. `which curl` no macOS pode dar `/usr/bin/curl` (BSD, sistema) ou `/opt/homebrew/bin/curl` (Homebrew, GNU-ish), com flags diferentes. Em Linux, geralmente `/usr/bin/curl` (GNU).

Mitigação: resolver não tenta resolver $PATH (não-determinístico em runtime); registra `cmd_name` literal e aceita ambiguidade. Sandbox profile fixa PATH se necessário.

### 11.4 Sandbox: bwrap vs sandbox-exec

| Aspecto | Linux (bwrap) | macOS (sandbox-exec) |
|---|---|---|
| Tecnologia | user namespaces + bind mounts | sbpl (Scheme-like policy) |
| FS isolation | granular (bind ro/rw, hide) | granular mas via sbpl rules |
| Net isolation | `--unshare-net` (zero net) | regras em sbpl (`deny network*`) |
| Process isolation | `--unshare-pid` | `process-fork` deny |
| Status oficial | first-class no Linux | `sandbox-exec` é **deprecated** desde macOS 10.10 (ainda funciona, sem garantia futura) |
| Maturidade | battle-tested em flatpak | menos documentado; sbpl é Apple-internal API |
| Performance | spawn ~10ms | spawn ~20ms |

Implicação direta: **suporte de macOS pra sandbox é fragile**. Apple não documenta sbpl publicamente; reverse-engineered por projetos como `firefox-sandbox`. Atualizações de macOS podem quebrar profiles sem aviso.

Recomendação prática (alinhado com `AGENTIC_CLI/PERMISSION_ENGINE.md` §13.2):

- Linux: bwrap é tier-1.
- macOS: sandbox-exec é tier "partial". Esperar bugs; degradar gracefully.
- macOS regulado/enterprise: considerar **rodar em VM Linux** (UTM, Parallels) pra ter sandbox confiável. Sandbox-exec não é mais audit-grade.

### 11.5 Heredocs e parsing platform-specific

Tree-sitter-bash parser é o mesmo em Linux e macOS — sem diferença. Mas:

- Line endings: heredocs com CRLF (vindo de Windows/WSL crossover) podem confundir o scanner em versões antigas do parser.
- Locale: `LC_ALL=C` vs `en_US.UTF-8` afeta range de chars aceitos em identifiers (tree-sitter trata como bytes, então geralmente OK; mas grammars com classes regex podem variar).

### 11.6 Aliases e RC files

| Arquivo | Linux | macOS |
|---|---|---|
| `~/.bashrc` | carregado em interactive bash | idem (se shell é bash) |
| `~/.bash_profile` | login bash | idem |
| `~/.profile` | login fallback (POSIX) | idem |
| `~/.zshrc` | só se shell é zsh | **default em Catalina+** |
| `/etc/profile.d/*.sh` | comum | menos comum |

Resolver lendo aliases user → vetor de bypass. Sandbox que **não** monta `~/.bashrc` e `~/.zshrc` (hide_paths) elimina. Em macOS Catalina+ a coisa default a proteger é `~/.zshrc`, não `.bashrc`.

### 11.7 Resumo da matriz Linux/macOS pra security gate

| Decisão | Linux | macOS |
|---|---|---|
| Tree-sitter-bash parseia idêntico? | sim | sim |
| Comando `cmd` tem mesma semântica? | **frequentemente não** (GNU vs BSD) | — |
| Sandbox confiável disponível? | sim (bwrap) | parcial (sandbox-exec, deprecated) |
| Default shell pra interactive? | bash 5.x | zsh 5.x (Catalina+) |
| `/bin/sh` é bash? | quase nunca (dash) | bash 3.2 (legacy) |
| Coreutils com mesmas flags? | GNU consistente | BSD; GNU via Homebrew opt-in |
| RC file principal | `.bashrc` | `.zshrc` (Catalina+) |
| Path de bwrap | `/usr/bin/bwrap` | N/A; usar sandbox-exec |
| Recomendação pra Forja tier | first-class | partial; considerar VM |

### 11.8 Implementação prática

Resolver deve:

1. **Detectar plataforma** no startup (`uname -s`).
2. **Marcar AST node com platform flag** se comando tem flags BSD/GNU-divergentes.
3. **Capability ceiling diferenciado** por plataforma (macOS: sem `net-egress` filtering granular se sandbox-exec não suporta na versão atual).
4. **Doctor command (§13.3 do PERMISSION_ENGINE)** reporta versão do shell, version do bash, BSD vs GNU coreutils.
5. **Fail clearly em features platform-specific.** Resolver que vê `sed -i ''` em Linux → flag `macos_specific_pattern` → não confunde com erro de sintaxe.

### 11.9 Windows nativo

Out of scope pra shell parsing. PowerShell tem AST próprio (não bash). Cmd.exe tem sintaxe completamente distinta. WSL roda bash Linux normal (parsing idêntico ao Linux).

Recomendação: **agentic CLI em Windows usa WSL2 ou Docker**. Não tente parsear cmd.exe ou PowerShell com tree-sitter-bash.

---

## 12. Alternativas pra shell parsing

| Alternativa | Quando ganha |
|---|---|
| **tree-sitter-bash** | Cobertura ampla, error recovery, ecossistema |
| **`shfmt` parser** (Go, da mvdan) | Formatter; AST limpo; embed em Go é trivial; cobertura POSIX/bash/mksh excelente |
| **`bashlex`** (Python) | Pure Python; sem dep C; lento mas auditável |
| **`oil shell` parser** | Mais rigoroso semanticamente; AST mais "real" mas dep pesada |
| **Hand-written lexer + whitelist** | Quando você só aceita pipelines simples literais; ~300 linhas |
| **POSIX sh subset hand-parser** | Quando você consegue restringir input a sh estrito |

Pra security gate em produto que precisa ship: **`shfmt` parser embarcado** é frequentemente melhor que tree-sitter-bash, se Go é viável. Vantagens:
- Sem dep C
- AST mais limpo
- Cobre POSIX sh + bash + mksh
- Manutenção ativa do mvdan
- Mesmo author do `gosh` (shell parser/printer)

Tree-sitter ganha se você já tem ecossistema tree-sitter no produto (highlighting + análise).

---

## 13. Recomendações pra capability resolver

Síntese das partes anteriores.

### 12.1 Use tree-sitter como entry-point, não como autoridade

Parse → walk → validate shape → reject cedo. **AST não autoriza nada**; autoriza o que sobreviver à whitelist.

### 12.2 Mantenha registry hardcoded de comandos resolvíveis

Não dependa de tree-sitter pra "entender" o que `git`, `npm`, `curl` fazem. Registry estático mapeia `cmd → resolver(args) → capabilities`. Tree-sitter só te diz que o nó é `git` e os args são `[clone, https://...]`.

### 12.3 Reject early, reject loud

Construção exótica detectada → `Refuse(reason="...")` com motivo legível. Não tente resolver. Confirm humano cobre.

### 12.4 Fuzz adversarial

Conformance suite com tentativas de bypass:
- `$(rm$IFS-rf$IFS/)`
- `r''m -rf /` (alias expansion)
- `; }rm -rf /; { :`
- `cat <<EOF ; rm -rf /\nEOF`
- `: > /etc/passwd`
- `bash<<<"rm -rf /"`

Cada um deve cair em Refuse ou em capability resolver conservador.

### 12.5 Threat model honesto

Documente: parser detecta forma sintática. Não detecta **intenção semântica**. `cat /etc/passwd` é lido como `read-fs:/etc/passwd`, capability resolve, policy decide. Tree-sitter não decide se ler `/etc/passwd` é benigno ou exfil — só identifica que é a operação.

---

## 14. Setup prático

### 13.1 Linguagens de produto

| Lang | Crate / package |
|---|---|
| Rust | `tree-sitter` + `tree-sitter-bash` |
| Go | `github.com/smacker/go-tree-sitter` + bash binding |
| Python | `tree_sitter` + `tree_sitter_bash` |
| Node | `tree-sitter` + `tree-sitter-bash` |
| WASM | `web-tree-sitter` + bash wasm parser |

### 13.2 Boilerplate Rust

```rust
use tree_sitter::{Parser, Query, QueryCursor};

let mut parser = Parser::new();
parser.set_language(&tree_sitter_bash::LANGUAGE.into())?;

let source = b"npm test && git status";
let tree = parser.parse(source, None).ok_or("parse failed")?;
let root = tree.root_node();

// Query: encontrar todos os command_name
let query = Query::new(
    &tree_sitter_bash::LANGUAGE.into(),
    "(command name: (command_name (word) @name))",
)?;

let mut cursor = QueryCursor::new();
for match_ in cursor.matches(&query, root, source.as_slice()) {
    for capture in match_.captures {
        let text = capture.node.utf8_text(source)?;
        println!("command: {}", text);
    }
}
```

### 13.3 Testing

Use `tree-sitter-bash` test corpus como baseline + adicione seus adversariais:

```
tree-sitter parse "rm -rf /tmp/foo"        # AST limpo
tree-sitter parse '$(rm$IFS-rf$IFS/)'      # AST esquisito; deve cair em Refuse
tree-sitter parse 'cat <<EOF\nfoo\nEOF'    # heredoc; AST com heredoc_body
```

---

## 15. Resumo brutal

- Tree-sitter-bash é a **melhor escolha pública** pra parsing de shell quando você precisa de cobertura + error recovery.
- Não confunda "parseou" com "entendeu". Parser te dá estrutura; semântica é problema seu.
- Pra security gate, **whitelist + Refuse** é o pattern correto, com tree-sitter como entry-point.
- Considere alternativas leves (`shfmt` em Go, hand-rolled lexer) quando você não precisa da cobertura toda.
- Fuzz adversarial é obrigatório. Bash parser bug = bypass de policy.

Quem usa pra capability resolver: veja `AGENTIC_CLI/PERMISSION_ENGINE.md` §5.2 pra integração concreta.
