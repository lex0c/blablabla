# TOOL_ERGONOMICS

Padrões de uso eficiente de comandos shell para o agente. Não é manual de Linux — é catálogo de **padrões onde o modelo default escolhe a versão ineficiente observavelmente**, com a substituição correta.

Referenciado por playbooks que leem código pesadamente (`code-review`, `debug`, `gap-audit`, `explain`, `perf-investigate`) via `references:`.

---

## 0. Critério de inclusão

Padrão só entra aqui se as duas condições valem:

1. **Modelo default escolhe a versão pior** em sessões reais ou eval, não por hipótese.
2. **Substituição economiza tokens, latência ou risco** mensuravelmente — não é só "mais idiomático".

Não entra:
- "Use `cd` antes do comando" — todo modelo sabe; instrução de prompt resolve.
- "Use `awk` em vez de `cut`" — preferência estética sem ganho.
- Padrão específico de uma distro/shell sem evidência de erro recorrente.

Doc cresce por incidente, não por completude. Se ninguém errou, não entra.

---

## 1. Princípios

1. **Token-efficiency primeiro.** Output que não cabe em 1 tela queima contexto sem ganho. Filtrar/slicar antes de imprimir.
2. **Slice cirúrgico.** Target conhecido → ler só ele. Range conhecido → `sed -n 'A,Bp'`. Range desconhecido → `grep -n` primeiro, depois Read com offset.
3. **Combine grep + slice.** `grep -n 'pattern' file` retorna `linha:texto` → use a linha como offset pro Read tool.
4. **Escopo conservador por default.** `find .`, não `find /`. Glob específico, não `**/*` cego.
5. **Prefira tool dedicado quando existe.** Read > cat. Edit > sed -i. Grep tool > bash grep. Bash é fallback, não primeiro recurso.

---

## 2. Tabela "ruim → bom"

### 2.1 Search

| Ruim | Bom | Por quê |
|---|---|---|
| `cat file \| grep X` | `grep X file` | UUOC clássico; dobra processos sem ganho |
| `grep -r X /` | `grep -rn X .` (ou path específico) | escopo + número de linha |
| `find . -name '*.py' -exec grep X {} \;` | `grep -rn X --include='*.py' .` ou `rg X -t py` | 1 processo vs N |
| `ls dir/ \| grep foo` | `ls dir/foo*` ou `find dir -name 'foo*'` | shell glob existe pra isso |
| `grep` em árvore grande sem filtro | `rg` quando disponível (`command -v rg`) | respeita `.gitignore`, ~5-10× mais rápido |
| `grep X file` (sem `-n`) quando vai ler depois | `grep -nH X file` | número de linha = offset pro Read |

### 2.2 Read

| Ruim | Bom | Por quê |
|---|---|---|
| `cat large_file.log` | Read com offset/limit, ou `sed -n 'A,Bp'` | evita explosão de contexto |
| ler arquivo inteiro pra achar 1 função | `grep -n 'def foo' file` → Read offset=linha | 50 tokens vs 5000 |
| `cat a.txt b.txt c.txt` | Read sequencial em cada um | tool dedicado preserva semântica de arquivo |
| `head -n 5000 file` | `sed -n '1,5000p' file` | composável; `head` ok pra N ≤ 50 |

### 2.3 Slice

| Ruim | Bom |
|---|---|
| `cat file \| sed -n '10,20p'` | `sed -n '10,20p' file` |
| `tail -n +100 file \| head -n 50` | `sed -n '100,149p' file` |
| `awk 'NR==42' file` | `sed -n '42p' file` (mesma coisa, menos magia) |
| `head -n 100 \| tail -n 20` | `sed -n '81,100p' file` |

### 2.4 List / navigate

| Ruim | Bom | Por quê |
|---|---|---|
| `ls -R .` em projeto grande | `find . -maxdepth 2 -type d` ou `ls -d */` | recursão sem limite explode |
| `find / -name x` | `find . -name x` (ou path conhecido) | scan de filesystem inteiro queima recurso |
| `tree` sem `-L` | `tree -L 2 -I 'node_modules\|.git\|dist'` | profundidade + ignore |
| `ls -la` quando só quer nomes | `ls` ou `ls -d */` | cada flag custa colunas no output |

### 2.5 Count / stats

| Ruim | Bom |
|---|---|
| `cat file \| wc -l` | `wc -l file` |
| `grep X file \| wc -l` | `grep -c X file` |
| `find . -name '*.py' \| wc -l` | `find . -name '*.py' -printf '.' \| wc -c` (ou aceite o cost) |

### 2.6 Find regex / glob

| Ruim | Bom | Por quê |
|---|---|---|
| `find -regex '.*\.\(ts\|tsx\)'` | `find -regex '.*\.\(tsx\|ts\)'` | longest alternative first; senão pula `.tsx` silenciosamente |
| `find . -name "*.py" 2>/dev/null` em `/` | escopo específico, **sem** `2>/dev/null` mascarando | `2>/dev/null` esconde bug, não conserta |
| `find . -type f -name X -exec ...` | `find . -type f -name X -print0 \| xargs -0 ...` | trata espaços/newlines em paths |

### 2.7 Pipes

| Ruim | Bom |
|---|---|
| `cat \| grep \| awk \| sed` (4 stages) | um único `awk` ou `sed` faz tudo |
| comando \| `less` / `more` em ambiente automático | pager bloqueia o agente; redirecione ou `head` |
| `command \| tee /dev/tty \| ...` | só redirecione output uma vez |

---

## 3. Notas contextuais

- **`rg` vs `grep`:** prefira `rg` quando `command -v rg` retorna 0. Default em sistemas modernos; respeita `.gitignore`. Fallback: `grep -rn --include='<pat>' <path>`.
- **macOS vs Linux:** `sed -i ''` no macOS, `sed -i` no GNU/Linux. Diferenças similares em `find`, `xargs`, `date`. Prefira tool dedicado (Edit) pra evitar a divergência.
- **Encoding / binary:** se arquivo pode ser binário, `file <path>` antes de `cat`. Read tool detecta automaticamente.
- **Paths com espaço ou flag-like:** sempre quote (`"$file"`); use `--` antes de filename pra parar parsing de flags (`grep -- -v file`).
- **Output truncado por terminal:** redirecione pra arquivo se for maior que ~200 linhas, depois Read com offset.

---

## 4. Anti-patterns gerais

| Anti-pattern | Por que é ruim |
|---|---|
| `2>/dev/null` reflexo | Mascara bugs reais. Use só quando o ruído é conhecido e benigno (ex: walk em árvore com permissões mistas). |
| `xargs` sem `-0` em paths possivelmente com espaço | Quebra silenciosamente. Combine com `find -print0`. |
| `sed -i.bak` em automação sem cleanup | Deixa `.bak` lixo. Use Edit. |
| Loop bash quando 1 comando resolve | `for f in *.txt; do grep X "$f"; done` → `grep X *.txt` |
| `command \| head` sem timeout em comando lento | `head` fecha pipe, mas comando upstream pode demorar a notar (SIGPIPE). Use `timeout` se necessário. |
| `ls` parseado por script | `ls` é pra humano. Use `find` ou shell glob. |
| `which` em script | Use `command -v` (POSIX, mais confiável). |
| `cd <dir> && comando` quando comando aceita path | Maioria aceita: `git -C <dir>`, `make -C <dir>`, `tar -C <dir>`. Evita mudar cwd da sessão. |

---

## 5. Quando ignorar este doc

- Comando one-off interativo onde clareza > eficiência.
- Debug exploratório onde você ainda não sabe o que procura — fazer pipeline meio bagunçado é OK até a forma aparecer.
- Output pequeno (< 50 linhas): `cat file` é fine; otimizar é overkill.

A regra é: **otimize quando o custo aparece** (contexto saturado, latência, output truncado). Otimização preventiva em comando trivial é cargo cult.

---

## 6. Como adicionar entrada nova

1. Observou modelo default escolher versão pior em sessão real ou eval.
2. Versão correta economiza tokens, latência ou risco mensurável.
3. Adicionar linha na tabela apropriada (§2.x) com **Por quê** se não óbvio.
4. Se o ganho é grande mas o padrão é exótico, vai pra §3 (notas contextuais), não §2.

Sem (1), o doc vira cargo cult. Sem (2), vira preferência estética. Ambos diluem o sinal das entradas que importam.
