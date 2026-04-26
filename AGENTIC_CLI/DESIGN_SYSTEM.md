# DESIGN_SYSTEM

Foundations visuais do `AGENTIC_CLI`. Tokens, glyphs, vocabulário, hierarquia. As regras que governam consistência **antes** de qualquer componente existir.

`UI.md` cataloga **componentes**. Este doc define **as decisões atômicas** que componentes consomem. Decisão tomada uma vez aqui, replicada perfeitamente em todo lugar.

Design system bom é invisível — ninguém percebe consistência, todo mundo percebe inconsistência.

---

## 0. Princípios visuais (não-negociáveis)

1. **Densidade de informação primeiro; decoração por último.** Cada caractere ocupa espaço; cada espaço deve carregar valor.
2. **Semântica antes de estética.** Cor/glyph existe pra comunicar estado, nunca pra "ficar bonito".
3. **Capability-aware sempre degrada.** Truecolor → 256 → 16 → NO_COLOR; Unicode → ASCII. Toda decisão visual tem 4 fallbacks declarados.
4. **Unicode é decoração; ASCII é conteúdo.** Glyph Unicode realça, não carrega. Mensagem deve fazer sentido em pure ASCII.
5. **Cor carrega *hint*, nunca *load-bearing meaning*.** Daltonismo + NO_COLOR + pipe — informação crítica nunca depende só de cor.
6. **Whitespace é ferramenta, não ausência.** Linha em branco separa; indent agrupa. Sem entropia visual gratuita.
7. **Consistência > criatividade.** Glyph com 1 significado. Tokens com 1 nome. Componente novo adapta padrão antigo, não inventa.
8. **Sem motion no critical path.** Animação custa atenção. Spinner em pending OP, ok. Animar elemento informativo, não.

---

## 1. Vocabulário visual (semantics atomics)

Cada elemento visual tem **uma** definição. Reuso = bug.

### 1.1 Glyphs de status

| Unicode | ASCII | Semântica | Cor token |
|---|---|---|---|
| `✓` | `[x]` | Concluído com sucesso | `status.success` |
| `✗` | `[!]` | Falhou (erro) | `status.error` |
| `⊘` | `[/]` | Negado por policy/user | `status.denied` |
| `⚠` | `[?]` | Warning, atenção | `status.warning` |
| `⏳` | `[~]` | Em execução | `status.running` |
| `○` | `[ ]` | Pendente, não iniciado | `status.pending` |
| `↻` | `[r]` | Retry em andamento | `status.running` |

Sem ambiguidade. `✓` é sucesso terminal — nunca pra "checked checkbox" ou "good idea".

### 1.2 Glyphs de ação

| Unicode | ASCII | Semântica |
|---|---|---|
| `▶` | `>` | Apontador / item ativo / prompt |
| `↵` | `Enter` | Confirmação / submit |
| `␛` | `Esc` | Cancelar / sair |
| `⇥` | `Tab` | Completion |
| `⌃C` | `Ctrl+C` | Interrupt |
| `↑↓` | `up/down` | Navegação |

Inputs hint sempre visíveis no rodapé do modal/prompt onde se aplicam.

### 1.3 Glyphs estruturais

| Unicode | ASCII | Uso |
|---|---|---|
| `─` | `-` | Separador horizontal leve |
| `═` | `=` | Separador horizontal pesado (raro; só em error fatal) |
| `│` | `|` | Borda vertical |
| `┌┐└┘` | `+` | Cantos de box |
| `├┤` | `+` | Junções |
| `·` | `*` | Bullet de lista |
| `…` | `...` | Truncate |
| `▆▁` | `#.` | Progress bar (filled/empty) |

### 1.4 Glyphs proibidos em conteúdo funcional

| Categoria | Exemplos | Por que |
|---|---|---|
| Emoji decorativo | 🚀 🎉 🔥 ⭐ | Inconsistente entre terminais; ruído |
| ASCII art | `╔══╗`, `▓▒░` | Decoração sem informação |
| Tipografia fancy | `★`, `❖`, `◈` | Substituível por glyph estrutural |
| Setas direcionais decorativas | `➜ ⇒ ➤` | `→` ou `>` resolvem |

Regra: se o glyph não está nas tabelas 1.1–1.3, **não usa**. Adicionar novo glyph = PR específica com justificativa.

---

## 2. Color tokens (semânticos, não literais)

Tokens são **semânticos** (`status.error`), nunca literais (`red-500`). Componentes consultam tokens; tokens resolvem em runtime baseado em capabilities.

### 2.1 Status

| Token | Truecolor | ANSI 256 | ANSI 16 | NO_COLOR (prefixo) |
|---|---|---|---|---|
| `status.pending` | `#888888` | 245 | gray | (sem prefixo) |
| `status.running` | `#5fafd7` | 74 | cyan | (sem prefixo) |
| `status.success` | `#5faf5f` | 70 | green | `[OK]` |
| `status.error` | `#d75f5f` | 167 | red | `[FAIL]` |
| `status.warning` | `#d7af5f` | 179 | yellow | `[WARN]` |
| `status.denied` | `#af5f87` | 132 | magenta | `[DENIED]` |
| `status.info` | `#87afd7` | 110 | blue | `[INFO]` |
| `status.disabled` | `#555555` | 240 | dim | (suffix `(disabled)`) |

### 2.2 Text

| Token | Truecolor | ANSI 256 | ANSI 16 | NO_COLOR |
|---|---|---|---|---|
| `text.primary` | `#e4e4e4` | 254 | default | default |
| `text.secondary` | `#9e9e9e` | 247 | dim | dim |
| `text.muted` | `#6c6c6c` | 242 | dim | dim |
| `text.inverse` | `#1c1c1c` on bg | — | bg-default | — |
| `text.code` | `#d7d787` | 186 | yellow | default |
| `text.link` | `#5fafd7` | 74 | cyan + underline | underline |

### 2.3 Border / Surface

| Token | Truecolor | ANSI 256 | ANSI 16 | NO_COLOR |
|---|---|---|---|---|
| `border.default` | `#5e5e5e` | 240 | dim | (sem) |
| `border.focus` | `#5fafd7` | 74 | cyan | (label `[focus]`) |
| `border.modal.info` | `#5fafd7` | 74 | cyan | (label `[modal]`) |
| `border.modal.warn` | `#d7af5f` | 179 | yellow | (label `[!]`) |
| `border.modal.danger` | `#d75f5f` | 167 | red | (label `[!!]`) |
| `surface.modal` | (none — bg sempre default) | — | — | — |

### 2.4 Profile / Brand

| Token | Truecolor | ANSI 256 | ANSI 16 | NO_COLOR |
|---|---|---|---|---|
| `profile.autonomous` | `#5fafff` | 75 | blue | `[autonomous]` |
| `profile.orchestrated` | `#5faf5f` | 70 | green | `[orchestrated]` |
| `profile.hybrid` | `#af5fd7` | 134 | magenta | `[hybrid]` |

### 2.5 Diff

| Token | Truecolor | ANSI 256 | ANSI 16 | NO_COLOR |
|---|---|---|---|---|
| `diff.add` | `#5faf5f` | 70 | green | prefix `+ ` |
| `diff.remove` | `#d75f5f` | 167 | red | prefix `- ` |
| `diff.context` | `#9e9e9e` | 247 | dim | prefix `  ` |
| `diff.hunk_header` | `#af87d7` | 140 | magenta | prefix `@@ ` |

### 2.6 Resolução de token (algoritmo)

```ts
function resolve(token: string, caps: Capabilities): string {
  if (caps.color === 'truecolor') return tokens[token].truecolor;
  if (caps.color === '256')        return ansi256(tokens[token].ansi256);
  if (caps.color === '16')         return ansi16(tokens[token].ansi16);
  /* NO_COLOR */                   return ''; // prefix aplicado em outro lugar
}
```

Componente nunca conhece RGB. Só conhece token.

---

## 3. Typography (no terminal)

Limitado mas existe. Cada estilo tem uso definido.

| Estilo | Uso semântico | Suporte |
|---|---|---|
| Normal | Conteúdo padrão | universal |
| **Bold** | Labels, títulos, ênfase em decisão | universal |
| Dim | Secondary, hint, disabled | universal |
| _Italic_ | Estado efêmero ("running...", "extracting...") | maioria |
| ~~Strikethrough~~ | Item denied/cancelled | maioria |
| Underline | Links (em terminais que suportam) | hyperlink-supported |

### 3.1 Combinações proibidas

- **Bold + Dim** — conflito visual; bold quer ênfase, dim quer recuo
- _Italic + Dim_ — legibilidade ruim em fontes monospace estreitas
- Underline + Italic — sobrecarga; nunca duas decorações simultâneas

### 3.2 Hierarquia de texto

```
**Section Header**           ← bold
Item content                 ← normal
  hint or detail             ← dim
  _running state_            ← italic
  ~~cancelled action~~       ← strikethrough
```

Apenas 5 níveis. Mais que isso: simplifique ou refatore estrutura.

---

## 4. Spacing system

Em terminal, spacing é **vertical (linhas)** ou **horizontal (caracteres)**. Sem pixels.

### 4.1 Vertical rhythm

| Token | Linhas | Uso |
|---|---|---|
| `space.adjacent` | 0 | items inline ou conectados |
| `space.tight` | 1 | items relacionados em lista |
| `space.section` | 2 | entre seções diferentes |
| `space.heavy` | 3 + `─` | pré/pós erro fatal (raro) |

Default vertical entre tool cards: `space.tight` (1 linha).
Default entre user prompt e response: `space.section` (2 linhas).

### 4.2 Horizontal indent

| Token | Caracteres | Uso |
|---|---|---|
| `indent.0` | 0 | nível root |
| `indent.1` | 2 | primeiro aninhamento (tool output dentro de card) |
| `indent.2` | 4 | segundo aninhamento (sub-detalhe) |
| `indent.3` | 6 | máximo permitido |

**indent ≥ 8 é proibido.** Refatore estrutura.

### 4.3 Padding em modal

```
┌──────────────────────────────┐
│                              │   ← 1 linha de padding
│  Title                       │
│                              │   ← 1 linha
│  Body content goes here.     │
│                              │
│  [a]ccept  [r]eject          │
│                              │   ← 1 linha
└──────────────────────────────┘
```

Modal sempre 1 linha de padding top/bottom, 2 chars left/right.

### 4.4 Largura

| Contexto | Largura mínima | Largura máxima |
|---|---|---|
| Modal | 40 chars | 60 chars (legibilidade) |
| Tool card | full width | full width |
| Diff | full width | full width |
| Footer | 1 line | 1 line |
| Header | 1 line | 1 line |

---

## 5. Iconography catalog (referência canônica)

Lista única, autoridade. Componentes consultam, não inventam.

### 5.1 Operacionais (já em §1.1-1.3)

Ver tabelas em §1. **Esses são os únicos glyphs permitidos** em conteúdo funcional.

### 5.2 Reservados (futuro)

Glyphs reservados pra uso futuro, **não usar agora** em outro contexto:

| Unicode | Reservado pra |
|---|---|
| `🔒` | Secrets/encrypted (v2) |
| `📝` | Memory write proposed |
| `⚙` | Compaction running |
| `⏸` | Paused state |
| `▶▶` | Replay mode |

### 5.3 Composições válidas

```
✓ done                          ← glyph + space + label
[ a ] accept                    ← bracket + space + key + space + bracket + space + action
▶ tool_name                     ← arrow + space + identifier (tool/file/etc)
$ 0.04 / $5.00                  ← prefix + space + value + sep + value
```

---

## 6. Affordance language

Como user sabe que algo é interativo / acionável.

### 6.1 Affordances explícitas

| Visual | Significado |
|---|---|
| `[a]ccept [r]eject` | Atalhos de teclado disponíveis (letra entre brackets) |
| `>` cursor | Input field ativo |
| `▶ option` | Item selecionável atual em lista |
| `(focus)` em NO_COLOR | Elemento focado |
| Bold border | Modal ativo |
| Dim text | Disabled / unavailable |
| Strikethrough | Cancelled / denied |

### 6.2 Affordances proibidas

- **Underline pra "clicável"** — terminal não suporta click consistentemente
- **Hover state** — não existe sem mouse
- **Tooltip** — não-portável em terminal
- **Drag-and-drop indicators** — não aplicável

### 6.3 Discoverability

Toda ação interativa **declara seu atalho** explicitamente:

```
[a]ccept  [e]dit  [r]eject  [w]hy?
```

Não:
```
Press a key to continue.
```

Atalho sem display = feature invisível.

---

## 7. Motion & feedback

Terminal tem motion limitada. Catálogo do permitido.

### 7.1 Motions permitidas

| Tipo | Frame rate | Quando |
|---|---|---|
| Spinner | 80ms/frame | Pending op > 200ms |
| Progress bar | ≤ 60fps (16ms) | Op com progress conhecido |
| Token streaming | batched 30fps (33ms) | Modelo gerando |
| Cursor blink | terminal-managed | Input ativo (terminal nativo cuida) |

### 7.2 Spinner frames

```
Unicode (10): ⠋ ⠙ ⠹ ⠸ ⠼ ⠴ ⠦ ⠧ ⠇ ⠏
ASCII (4):    | / - \
```

Único spinner. Sem variantes "fancy".

### 7.3 Progress bar render

```
Unicode: ▆▆▆▆▆▁▁▁▁▁  60%
ASCII:   #####.....  60%
```

Largura fixa: 10 chars. Sempre acompanhado de percent label.

### 7.4 Motions proibidas

- Animação ASCII art ("loading rocket 🚀")
- Fades / transições suaves
- Easing curves
- Bounce / elastic
- Snowflakes / festividade
- Cursor-trail effects

Princípio: motion **só comunica progresso**, nunca diverte.

---

## 8. Density principles

Calibragem por contexto, com regras hard.

### 8.1 Tabela de densidade

| Contexto | Padding | Detail level | Whitespace |
|---|---|---|---|
| History pane | min | low (collapsed by default) | minimal |
| Footer | none | minimal (counts only) | none |
| Header | none | minimal | none |
| Modal | medium (1 line) | medium | regular |
| First-run | generous | full | regular |
| Erro fatal | medium | full + path to logs | high |

### 8.2 Tool card collapse threshold

- ≤ 10 lines de output: expandido por default
- 11-50 lines: colapsado, expand on demand
- > 50 lines: colapsado obrigatório, sem expand inline (`/replay <step>` pra ver completo)

### 8.3 Modal width

- Mínimo: 40 chars
- Máximo: 60 chars
- Width adapta a conteúdo dentro do range; nunca extrapola

### 8.4 Footer limits

- 1 linha **sempre**
- Em width < 80 cols: usa abreviações (`steps` → `s`, `cost` → `$`)
- Em width < 60 cols: footer fica reduzido a `s7/50 $0.04`

---

## 9. Naming conventions

### 9.1 Tokens

```
<category>.<subcategory>.<modifier?>

status.success           ← simple
status.success.bg        ← com modifier (raro; usado em diff)
text.primary
border.modal.warn
profile.orchestrated
diff.add
```

Convenção: lowercase, `.`-separated, sem underscore.

### 9.2 Componentes

```
<PascalCase>             ← Card, Modal, ToolCallCard
```

### 9.3 Props

```
<camelCase>              ← onResolve, isCollapsed
variant: 'info' | 'warn' ← kebab-case em strings de variant
```

### 9.4 CSS-equivalent classes (em Ink, são props no Box)

Não temos classes em Ink. Convenção é **prop direta** com nome do token aplicado:

```tsx
<Box borderStyle="round" borderColor={tokens.border.modal.warn}>
```

---

## 10. Composition rules

### 10.1 Hierarquia permitida

```
App
├── Header
├── HistoryPane (Static)
│   ├── UserMessage
│   ├── StreamingMessage
│   └── ToolCallCard
│       ├── Card (primitive)
│       └── DiffView (sometimes)
├── LiveRegion
│   ├── TodoListView
│   └── DAGProgress (orchestrated)
├── InputPane
│   └── (TextInput | Modal)
│       └── PermissionPrompt | TrustPrompt | MemoryWritePrompt
└── Footer
    ├── BudgetBar
    ├── ProfileBadge
    ├── MemoryBadge
    ├── BackgroundProcessTray
    └── CheckpointBar
```

### 10.2 Regras hard

- **Modal nunca contém Modal.** Stack é proibido (invariante de §8 do `STATE_MACHINE.md`).
- **Footer nunca contém Modal.** Modal sobrepõe Input, não Footer.
- **HistoryPane não contém Modal.** Modal vai entre History e Footer.
- **StreamingMessage é leaf.** Não recebe children.
- **Card pode aninhar Card** apenas pra collapsibility (1 nível de profundidade).
- **DiffView é leaf.** Não aninha outros componentes.

### 10.3 Slots vs children

- `<Card>{children}</Card>` — content
- `<Modal title="..." shortcuts={[...]}>{children}</Modal>` — slot via prop quando estruturado
- Children genérico só pra body solto

---

## 11. Theming hooks (extensibilidade futura)

### 11.1 v1: paleta única

Sem theme switcher na v1. Uma paleta canônica nesta spec.

### 11.2 v2: theme override (deferred)

Plano:

```toml
# ~/.config/agent/theme.toml
[colors]
status_success = "#5faf5f"   # override truecolor
status_error = "#d75f5f"

[icons]
status_success_unicode = "✓"
status_success_ascii = "[x]"
```

Tokens semânticos resolvem via merge: defaults → user override.

### 11.3 Light/dark detection

Detecção via `COLORFGBG` env var (alguns terminais expõem). Default: dark theme. Fallback: tabela canônica em §2 (que assume bg dark).

Light theme custom é responsabilidade do usuário (override via `theme.toml` em v2). Sem suporte automático em v1 — incidente raro vs custo de manter 2 paletas.

---

## 12. Capability degradation matrix

```
┌─ Detected ─────────────────┬─ Color ─┬─ Glyph ─┬─ Render strategy ─────────┐
│ Truecolor + Unicode        │ truecolor│ Unicode │ Full fidelity              │
│ 256 + Unicode              │ 256     │ Unicode │ Curated palette            │
│ 16 + Unicode               │ 16      │ Unicode │ ANSI base + glyphs         │
│ 16 + ASCII (locale C)      │ 16      │ ASCII   │ ANSI + ASCII fallback      │
│ NO_COLOR + Unicode         │ none    │ Unicode │ Glyphs + text prefixes     │
│ NO_COLOR + ASCII           │ none    │ ASCII   │ Pure text + prefixes       │
│ Pipe (não-TTY)             │ none    │ ASCII   │ → headless mode (NDJSON)   │
└────────────────────────────┴─────────┴─────────┴────────────────────────────┘
```

Detecção via `useCapabilities()` (uma vez no startup; reavalia em SIGWINCH apenas pra width).

---

## 13. Acessibilidade no design system

Reforça princípios já em UI.md §10:

### 13.1 Cor + redundância

Toda informação de status tem **3 sinais**:
1. Cor (token)
2. Glyph
3. Label textual

Daltonismo + NO_COLOR + pipe — 2 dos 3 ainda funcionam.

### 13.2 Contraste

ANSI 16 é referência. Combinações testadas:
- `text.primary` em bg default: passa
- `status.error` em bg default: passa
- `text.muted` em bg default: marginal — usar com cuidado

Truecolor segue paleta acessível por design (testes visuais antes de adicionar token).

### 13.3 ASCII fallback é first-class

Cada glyph Unicode tem ASCII obrigatório em §1. Eval cobre que componentes funcionam em `LOCALE=C`.

### 13.4 NO_COLOR é first-class

Cada token tem fallback NO_COLOR em §2. Eval cobre que prefixos textuais aparecem corretamente.

### 13.5 Width mínimo

60 cols é piso suportado. Abaixo: warning único + degradação graciosa.

### 13.6 Sem dependência de mouse

Princípio. Re-afirma em design system pq UX-design tradicional assume mouse; aqui não.

---

## 14. Versionamento

| Tipo de mudança | Bump | Eval requerido |
|---|---|---|
| Adicionar token novo | minor | smoke pass |
| Adicionar glyph novo | minor | a11y eval (ASCII fallback OK) |
| Mudar valor de fallback (ex: `#5faf5f` → `#5fcf5f`) | patch | snapshot diff visual |
| Mudar significado de token semântico | **major** | full a11y + visual eval |
| Mudar glyph semântico (ex: `✓` → `✔`) | **major** | full a11y eval |
| Remover token / glyph | **major** | migration path documentado |

Major bump em design system = breaking change pra implementadores que copiam tokens — comunicar claramente.

---

## 15. Anti-patterns visuais (não cometa)

| Anti-pattern | Por que ruim |
|---|---|
| Cor sem ícone redundante | Daltonismo / NO_COLOR / pipe quebram |
| Glyph hardcoded no componente (sem token) | Inconsistência futura inevitável |
| RGB hardcoded no componente | Quebra ANSI 16 / NO_COLOR |
| Animação no critical path | Custa atenção; quebra em SSH lento |
| ASCII art decorativo | Sem informação; bloat visual |
| Emoji em mensagem funcional | Inconsistente entre terminais |
| Modal stack | Invariante quebrado; user perdido |
| Indent > 6 levels | Refatore estrutura |
| Bold + dim simultâneo | Conflito visual |
| Underline pra "clicável" | Não funciona consistentemente |
| Cores customizadas fora dos tokens | Branding em UI = bloat |
| "Press any key" | Especifique a tecla |
| Footer multi-linha | Salta layout |
| Width fixed > 60 chars em modal | Legibilidade ruim |
| Spinner único por componente | Inconsistência; usa o canônico |

---

## 16. Tokens canônicos (referência rápida)

### 16.1 Tabela completa

```ts
// tokens.ts (canonical)
export const tokens = {
  // Status
  'status.pending':   { trueColor: '#888888', ansi256: 245, ansi16: 'gray',    noColor: '' },
  'status.running':   { trueColor: '#5fafd7', ansi256: 74,  ansi16: 'cyan',    noColor: '' },
  'status.success':   { trueColor: '#5faf5f', ansi256: 70,  ansi16: 'green',   noColor: '[OK]' },
  'status.error':     { trueColor: '#d75f5f', ansi256: 167, ansi16: 'red',     noColor: '[FAIL]' },
  'status.warning':   { trueColor: '#d7af5f', ansi256: 179, ansi16: 'yellow',  noColor: '[WARN]' },
  'status.denied':    { trueColor: '#af5f87', ansi256: 132, ansi16: 'magenta', noColor: '[DENIED]' },
  'status.info':      { trueColor: '#87afd7', ansi256: 110, ansi16: 'blue',    noColor: '[INFO]' },
  'status.disabled':  { trueColor: '#555555', ansi256: 240, ansi16: 'gray',    noColor: '(disabled)' },

  // Text
  'text.primary':     { trueColor: '#e4e4e4', ansi256: 254, ansi16: 'default', noColor: '' },
  'text.secondary':   { trueColor: '#9e9e9e', ansi256: 247, ansi16: 'gray',    noColor: '' },
  'text.muted':       { trueColor: '#6c6c6c', ansi256: 242, ansi16: 'gray',    noColor: '' },
  'text.code':        { trueColor: '#d7d787', ansi256: 186, ansi16: 'yellow',  noColor: '' },
  'text.link':        { trueColor: '#5fafd7', ansi256: 74,  ansi16: 'cyan',    noColor: '' },

  // Border
  'border.default':   { trueColor: '#5e5e5e', ansi256: 240, ansi16: 'gray',    noColor: '' },
  'border.focus':     { trueColor: '#5fafd7', ansi256: 74,  ansi16: 'cyan',    noColor: '[focus]' },
  'border.modal.info':{ trueColor: '#5fafd7', ansi256: 74,  ansi16: 'cyan',    noColor: '[modal]' },
  'border.modal.warn':{ trueColor: '#d7af5f', ansi256: 179, ansi16: 'yellow',  noColor: '[!]' },
  'border.modal.danger':{ trueColor:'#d75f5f', ansi256: 167, ansi16: 'red',    noColor: '[!!]' },

  // Profile
  'profile.autonomous':   { trueColor: '#5fafff', ansi256: 75,  ansi16: 'blue',    noColor: '[autonomous]' },
  'profile.orchestrated': { trueColor: '#5faf5f', ansi256: 70,  ansi16: 'green',   noColor: '[orchestrated]' },
  'profile.hybrid':       { trueColor: '#af5fd7', ansi256: 134, ansi16: 'magenta', noColor: '[hybrid]' },

  // Diff
  'diff.add':         { trueColor: '#5faf5f', ansi256: 70,  ansi16: 'green', noColor: '+ ' },
  'diff.remove':      { trueColor: '#d75f5f', ansi256: 167, ansi16: 'red',   noColor: '- ' },
  'diff.context':     { trueColor: '#9e9e9e', ansi256: 247, ansi16: 'gray',  noColor: '  ' },
  'diff.hunk_header': { trueColor: '#af87d7', ansi256: 140, ansi16: 'magenta', noColor: '@@ ' },
};
```

### 16.2 Glyphs canônicos

```ts
// glyphs.ts
export const glyphs = {
  // Status (§1.1)
  'status.success':  { unicode: '✓', ascii: '[x]' },
  'status.error':    { unicode: '✗', ascii: '[!]' },
  'status.denied':   { unicode: '⊘', ascii: '[/]' },
  'status.warning':  { unicode: '⚠', ascii: '[?]' },
  'status.running':  { unicode: '⏳', ascii: '[~]' },
  'status.pending':  { unicode: '○', ascii: '[ ]' },
  'status.retry':    { unicode: '↻', ascii: '[r]' },

  // Action (§1.2)
  'action.pointer':  { unicode: '▶', ascii: '>' },
  'action.confirm':  { unicode: '↵', ascii: 'Enter' },
  'action.cancel':   { unicode: '␛', ascii: 'Esc' },
  'action.complete': { unicode: '⇥', ascii: 'Tab' },
  'action.interrupt':{ unicode: '⌃C', ascii: 'Ctrl+C' },

  // Structural (§1.3)
  'sep.light':       { unicode: '─', ascii: '-' },
  'sep.heavy':       { unicode: '═', ascii: '=' },
  'border.v':        { unicode: '│', ascii: '|' },
  'border.tl':       { unicode: '┌', ascii: '+' },
  'border.tr':       { unicode: '┐', ascii: '+' },
  'border.bl':       { unicode: '└', ascii: '+' },
  'border.br':       { unicode: '┘', ascii: '+' },
  'bullet':          { unicode: '·', ascii: '*' },
  'truncate':        { unicode: '…', ascii: '...' },
  'progress.fill':   { unicode: '▆', ascii: '#' },
  'progress.empty':  { unicode: '▁', ascii: '.' },

  // Spinner (§7.2)
  'spinner.unicode': ['⠋', '⠙', '⠹', '⠸', '⠼', '⠴', '⠦', '⠧', '⠇', '⠏'],
  'spinner.ascii':   ['|', '/', '-', '\\'],
};
```

---

## 17. Como verificar conformidade

### 17.1 Static check

Lint custom em CI:
- Componentes não importam strings de cor literal (`#xxxxxx`, `red`, etc) — só `tokens.X`
- Componentes não usam glyph literal — só `glyphs.X`
- Indent ≤ 6 levels (AST check em JSX)
- Sem combinações proibidas (bold+dim) — detecta via análise de `<Text bold dim>`

### 17.2 Visual snapshot

Cada componente renderizado em **6 capability profiles** (§12). Snapshot por profile. Mudança visual = PR review.

### 17.3 A11y eval

- Componente renderiza corretamente em `NO_COLOR=1`
- Componente renderiza corretamente em `LOCALE=C` (ASCII)
- Componente renderiza corretamente em `COLUMNS=60`
- Toda informação de status tem ícone + label além de cor

---

## 18. Insight final

Design system não é feito pra ser admirado. É feito pra ser **pressuposto**. Componente bem desenhado **não tem decisão a fazer** sobre cor ou glyph; consulta token e segue. Quando todo componente faz isso, consistência é grátis.

A regra é: **cada decisão visual tomada uma vez aqui, replicada perfeita em todo lugar.** Nada de "esse caso é diferente". Casos diferentes viram tokens novos, não exceções inline.

Sem design system, cada componente reinventa cor, glyph, spacing — e o produto vira colcha de retalhos visuais. Com ele, terminal feio vira terminal **coerente**, que é o mais perto de "bonito" que terminal precisa chegar.
