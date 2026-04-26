# UI

Spec da camada de interface terminal do `AGENTIC_CLI`. Componentes, layout, theme, padrões de interação, microcopy, headless mode.

UI ruim mata adoção mais rápido que arquitetura ruim. Arquitetura ruim você sente em 6 meses. UI ruim você sente em 30 segundos.

---

## 0. Princípios de UX (não-negociáveis)

1. **Latência percebida > latência real.** Streaming + spinner em < 200ms é melhor que resposta completa em 1s sem feedback.
2. **Affordância antes de poder.** Se o user não sabe que pode fazer X, X não existe. Discoverability via `/`, `--help`, `Tab` competition é obrigatório.
3. **Cada estado tem feedback visível.** Pensando, executando, erro, sucesso — sem ambiguidade.
4. **Reversibilidade visível.** Se algo é desfazível, mostre. `Ctrl+Z` / `/undo` são primeiros class citizens, não easter eggs.
5. **Densidade calibrada por contexto.** Sessão longa = high density (econômico em pixels); modal = low density (foco).
6. **Quebra graciosamente.** Sem truecolor, sem Unicode, sem TTY — agente não morre, degrada.
7. **Microcopy importa tanto quanto componente.** "Algo deu errado" é bug. "Tool `bash` excedeu 30s; output abaixo, decisão sua" é UX.
8. **Inputs do humano são sagrados.** Nunca perdê-los. Nunca duplicá-los. Nunca pisotear o cursor durante digitação.
9. **Modal nunca surpreende.** Permission/trust/memory write sempre tem precedente lógico. Se não tem, é bug de arquitetura, não de UI.
10. **Consistência > criatividade.** Atalhos, cores, ícones, microcopy seguem padrões; novidade só onde resolve dor real.

---

## 1. Stack & deps (canônica, versão pinada)

```jsonc
{
  // core
  "ink": "5.x",
  "react": "18.x",

  // ink ecosystem (oficiais ou semi-oficiais)
  "ink-text-input": "6.x",
  "ink-select-input": "6.x",
  "ink-spinner": "5.x",
  "ink-link": "4.x",

  // capability detection
  "supports-color": "9.x",
  "supports-hyperlinks": "3.x",
  "is-unicode-supported": "2.x",

  // text/string
  "string-width": "7.x",
  "wrap-ansi": "9.x",
  "cli-truncate": "4.x",
  "strip-ansi": "7.x",

  // domain
  "diff": "5.x",
  "shikiji": "0.9.x",
  "picocolors": "1.x",

  // testing
  "ink-testing-library": "4.x"
}
```

**Regras de mudança:**
- Adicionar dep nova requer justificativa (substitui o quê? cobre quanto código?).
- Major bump de Ink só com migration guide testado.
- Lib morta (sem release > 12 meses) é candidata a fork interno, não substituição em pânico.

---

## 2. Layout regions (formal)

Tela dividida em **5 regiões fixas**, ordem top-to-bottom:

```
┌─ Header (1 linha, opcional) ─────────────────────┐
│ [profile] · [project] · [model]                   │
├─ History pane (Static, append-only) ─────────────┤
│ user: ...                                         │
│ ─                                                 │
│ ▶ tool call                                       │
│   output                                          │
│ ─                                                 │
│ assistant: ...                                    │
│ ─                                                 │
├─ Live region (dinâmica, opcional) ───────────────┤
│ Plan / TodoList / DAG progress                    │
├─ Input pane (1-3 linhas) ────────────────────────┤
│ > _                                               │
├─ Footer (1 linha, sempre presente) ──────────────┤
│ steps · cost · model · mem · bg                   │
└──────────────────────────────────────────────────┘
```

### 2.1 Breakpoints de largura

| Largura terminal | Comportamento |
|---|---|
| ≥ 120 cols | layout completo, todas as regiões com max content |
| 80-119 cols | layout completo, footer com abreviações (`steps` → `s`, `cost` → `$`) |
| 60-79 cols | esconde header (profile vai pra footer); diff em 1 coluna; tool cards comprimidos |
| < 60 cols | warning único: "terminal estreito (< 60 cols), UX degradada"; segue funcionando |

Detecção via `process.stdout.columns` + listener `SIGWINCH`. Re-layout em < 16ms (1 frame).

### 2.2 Quando cada região aparece

- **Header**: sempre, exceto se largura < 60 cols
- **History**: sempre (mesmo vazia, mostra `(sessão nova)`)
- **Live region**: condicional — só aparece se há TodoList ativo ou DAG em execução
- **Input**: sempre, exceto durante execução headless ou modal sobreposto
- **Footer**: sempre, mesmo em estados terminais (preserva info)

### 2.3 Modal overlay

Modais (permission/trust/memory write/plan approval) **sobrepõem o Input pane**. Nunca overlay sobre History (preserva contexto visual). Footer continua visível durante modal.

```
├─ History pane ───────────────────────────────────┤
│ ...                                               │
├──────────────────────────────────────────────────┤
│ ┌─ Permission required ────────────────────────┐ │
│ │ tool: bash                                    │ │
│ │ command: rm -rf ./build                       │ │
│ │                                               │ │
│ │ [a]ccept  [r]eject  [e]dit  [w]hy?            │ │
│ └───────────────────────────────────────────────┘ │
├─ Footer ─────────────────────────────────────────┤
│ steps 7/50 · $0.04 · ...                          │
└──────────────────────────────────────────────────┘
```

---

## 3. Catálogo de componentes (15 + primitivas)

Para cada: **props**, **estados**, **comportamento**, **fallbacks**.

### 3.1 Primitivas

#### `<Card>`
Box com borda, opcional título e status icon.

```ts
interface CardProps {
  title?: string
  status?: 'pending' | 'running' | 'done' | 'error' | 'denied'
  collapsed?: boolean
  borderStyle?: 'single' | 'round' | 'bold'  // ASCII fallback: '+'/'|'/'─'
  children: ReactNode
}
```

Status icon: `○` pending, `⏳` running, `✓` done, `✗` error, `⊘` denied. ASCII: `[ ]`, `[~]`, `[x]`, `[!]`, `[/]`.

#### `<Modal>`
Box centralizado overlay; foco automático; fecha em Esc ou via `onResolve`.

```ts
interface ModalProps {
  title: string
  variant: 'info' | 'warn' | 'danger' | 'success'
  shortcuts: { key: string; label: string; action: () => void }[]
  children: ReactNode
  onCancel?: () => void   // Esc
}
```

Cores por variant: info azul, warn amarelo, danger vermelho, success verde. NO_COLOR: prefixo `[!]`/`[?]`/`[X]`.

#### `<Bar>`
Barra horizontal de progresso (steps, budget, etc).

```ts
interface BarProps {
  value: number
  max: number
  width?: number
  label?: string
  warningThreshold?: number  // muda cor pra amarelo
  dangerThreshold?: number   // muda cor pra vermelho
}
```

Render: `▆▆▆▆▁▁▁▁` (Unicode) / `####....` (ASCII).

#### `<Badge>`
Pill curta com texto + cor.

```ts
interface BadgeProps {
  text: string
  variant: 'profile' | 'model' | 'count' | 'warning' | 'neutral'
}
```

Render: `[orchestrated]`, `[qwen2.5-coder:14b]`, `mem:12u`.

#### `<Diff>`
Diff colorido inline.

```ts
interface DiffProps {
  before: string
  after: string
  format: 'unified' | 'split'    // unified em ≤ 80 cols, split em ≥ 120
  contextLines?: number          // default 3
  maxLines?: number              // truncate com "[N more lines]"
  language?: string              // pra syntax highlight via shikiji
}
```

Render: linhas `+` em verde, `-` em vermelho, contexto neutro. Hunks separados por `─`.

### 3.2 Domain components

#### `<StreamingMessage>`
Renderiza assistant message com streaming token-by-token. Suporta tool_use blocks intercalados.

```ts
interface StreamingMessageProps {
  steps: Step[]
  currentStream?: AsyncIterable<StreamEvent>
  onToolUse: (toolUse: ToolCall) => void
}
```

**Estados:**
- idle: histórico inteiro static
- streaming: último step dynamic, anterior static
- error: último step com erro mostrado em vermelho discreto

**Performance:** batching de tokens — re-render no máximo 30fps mesmo se stream emite 200 tokens/s. Buffer interno descarrega a cada 33ms.

#### `<ToolCallCard>`
Card colapsável com tool call + output.

```ts
interface ToolCallCardProps {
  toolCall: ToolCall
  collapsed: boolean   // default: true se output > 10 linhas
  onToggle?: () => void
}
```

Render condensado:
```
▶ glob "src/**/*.ts"
  ✓ 14 files (collapsed; press ↵ to expand)
```

Render expandido:
```
▶ glob "src/**/*.ts"
  ✓ 14 files
    src/auth.ts
    src/queue.ts
    ...
```

Status icons + progressive disclosure.

#### `<PermissionPrompt>` (modal)

```ts
interface PermissionPromptProps {
  toolCall: ToolCall
  preview: string         // tool.preview(args)
  policy: 'confirm'       // só aparece em modo confirm
  onResolve: (decision: 'allow' | 'deny' | 'edit') => void
}
```

Microcopy:
```
Tool requires confirmation

  bash
  rm -rf ./build

  Why confirm? Matched rule: "rm *" → confirm

  [a]ccept  [r]eject  [e]dit  [w]hy?
```

Atalhos: `a/y/Enter` accept, `r/n/Esc` reject, `e` edit args (abre input), `w` mostra detalhe da rule matched.

#### `<TrustPrompt>` (modal)

```ts
interface TrustPromptProps {
  cwd: string
  filesToRead: string[]   // CLAUDE.md, .agent/config.toml, etc
  changedSinceLastTrust?: { file: string; lastHash: string; newHash: string }[]
  onResolve: (decision: 'trust' | 'reject' | 'inspect') => void
}
```

Microcopy primeira vez:
```
⚠ Diretório não-confiável detectado

  /home/lex/work/some-repo

  Vou ler:
    CLAUDE.md         (12 KB)
    .agent/config.toml (não existe)

  [t]rust  [r]eject  [i]nspect  [w]hy?
```

Microcopy re-prompt (mudança detectada):
```
⚠ Conteúdo confiável mudou

  /home/lex/work/some-repo

  CLAUDE.md mudou desde último trust:
    + 47 lines added
    - 3 lines removed

  Re-confirmar? [y]es  [n]o  [d]iff  [r]eject
```

#### `<MemoryWritePrompt>` (modal)

```ts
interface MemoryWritePromptProps {
  proposed: ProposedMemory
  source: 'user_explicit' | 'inferred'
  scope: 'user' | 'project'
  onResolve: (decision: 'accept' | 'edit' | 'reject') => void
}
```

Microcopy:
```
📝 Propor nova memória  [feedback / project]

  name: no-console-log
  description: console.log proibido em src/; usar logger
  body:
    Em arquivos `src/**/*.ts`, console.log/warn/error proibidos.
    **Why:** logs estruturados são exportados pra Datadog;
            console.* fura observabilidade.
    **How to apply:** ao editar arquivo em src/, usar
                     `import { logger } from "@/lib/logger"`.

  Source: inferred (de correção do usuário no turn 4)

  [a]ccept  [e]dit  [r]eject  [w]hy?
```

Inferred writes em diretório não-confiável: **prompt extra** "este diretório não-confiável; aceitar mesmo assim? [y/N]".

#### `<TodoListView>`
Checklist live no live region.

```ts
interface TodoListViewProps {
  items: TodoItem[]
  // TodoItem: { content: string, status: 'pending' | 'in_progress' | 'done', activeForm: string }
}
```

Render:
```
Plan:
  ✓ map callers de validateToken
  ▶ extract function pra src/auth/validate.ts
  ○ run tests src/auth/
```

Item em `in_progress` mostra activeForm ("extracting function..." em itálico se suportado).

Atualização **smooth** — adicionar/remover item anima por 1 frame; mudança de status sem scroll do histórico.

#### `<BudgetBar>`
Footer left section.

```ts
interface BudgetBarProps {
  steps: { used: number; max: number }
  cost: { usd: number; max: number }
  showBars?: boolean   // mini-bars em ≥ 100 cols
}
```

Render normal:
```
steps 7/50 · $0.04/$5.00
```

Render warning (steps > 70%):
```
steps 36/50 ⚠ · $0.04/$5.00
```

Render danger (>90%):
```
steps 47/50 ✗ · $4.85/$5.00 ✗
```

#### `<DiffView>`
Diff em mensagens de tool / proposta de edit.

(Já especificado em primitivas como `<Diff>`; este é wrapper com syntax highlight via shikiji.)

#### `<BackgroundProcessTray>`
Footer right section.

```ts
interface BackgroundProcessTrayProps {
  processes: BackgroundProcess[]
}
```

Render:
```
bg: 2 (npm-dev ✓, pytest ⏳)
```

Mouseable em terminais que suportam? Não. Atalho: `/bg` lista detalhada.

#### `<CheckpointBar>`
Indicador de checkpoints disponíveis pra undo.

```ts
interface CheckpointBarProps {
  checkpoints: Checkpoint[]
  currentIndex: number
}
```

Render compacto:
```
checkpoints: 4 · undo: Ctrl+Z
```

#### `<DAGProgress>` (profile orchestrated)
Visualização do DAG em execução.

```ts
interface DAGProgressProps {
  graph: DAGGraph
  nodeStates: Map<NodeId, NodeState>
}
```

Render:
```
DAG: edit_function
  ✓ locate          (file: src/auth.ts:42)
  ✓ read_context
  ▶ propose_edit    (retry 1/2)
  ○ validate_edit
  ○ apply
```

Node em retry mostra contador. Falha persistente em vermelho.

#### `<ValidatorTrace>`
Aparece quando validator falha; mostra hint e progress de retry.

```ts
interface ValidatorTraceProps {
  validator: string
  error: string
  retryHint?: string
  retryCount: number
  maxRetries: number
}
```

Render:
```
✗ JSONSchemaValidator failed (retry 1/2)
  expected: { file: string, line: int }
  got:      { file: string }
  hint:     Adicionar campo "line" com posição da função
```

#### `<ProfileBadge>`
Header indicator.

```ts
interface ProfileBadgeProps {
  profile: 'autonomous' | 'orchestrated' | 'hybrid'
}
```

Render: `[autonomous]` (azul) / `[orchestrated]` (verde) / `[hybrid]` (roxo).

#### `<MemoryBadge>`
Footer right section discreto.

```ts
interface MemoryBadgeProps {
  userCount: number
  projectCount: number
  loaded: boolean
}
```

Render: `mem 12u 4p`. Click/hover não disponível em terminal; `/memory list` pra detalhes.

#### `<SessionPicker>`

Listagem virtualizada de sessões com mini-recap expansível inline. Renderizado em modo full-screen (não modal) quando user invoca `agent --resume` (sem args) ou `/sessions list`.

```ts
interface SessionPickerProps {
  sessions: SessionSummary[]              // ordenado desc por started_at
  initialFilter?: SessionFilter
  onResume: (sessionId: string) => void
  onShow: (sessionId: string) => void     // expand mini-recap inline
  onCancel: () => void
}

interface SessionSummary {
  id: string
  goal: string                            // primeira linha do user prompt
  status: SessionStatus
  startedAt: number
  endedAt?: number
  durationMs: number
  steps: number
  costUsd: number
  cwd: string
  cwdLabel: string                        // ex: "blablabla"
  hasErrors: boolean
  incomplete: boolean                     // status não-terminal
  miniRecap?: string                      // pré-renderizado; carregado lazy se ausente
}

interface SessionFilter {
  project?: string
  since?: Date
  status?: SessionStatus
  search?: string
}
```

**Layout:**

```
┌─ Resumir sessão ──────────────────────────────────────────────────┐
│  [/] filtrar  [s] status  [p] projeto  [d] data                   │
│                                                                    │
│  ▶ sess_8a3f2b · 2 days ago · blablabla                           │
│    ✓ done · $0.04 · 15 steps                                      │
│    "refactor queue retry logic"                                    │
│                                                                    │
│    sess_7c1d09 · 3 days ago · blablabla                           │
│    ⚠ exhausted · $4.92 · 50 steps                                 │
│    "audit auth flow for OWASP top 10"                             │
│                                                                    │
│    sess_6b2e44 · 5 days ago · other-proj                          │
│    ✓ done · $0.18 · 8 steps                                       │
│    "/debug failing test in auth.test.ts"                          │
│                                                                    │
│  [↑↓] navegar  [↵] resume  [r] mini-recap  [Esc] sair              │
└────────────────────────────────────────────────────────────────────┘
```

**Comportamento:**

- `↑↓`: navegação; cursor `▶`
- `↵` (Enter): resume selecionada
- `r`: expande mini-recap **inline** sob a sessão selecionada (não muda de tela)
- `s`/`p`/`d`: filtros via picker secundário
- `/`: fuzzy search em `goal` + `cwdLabel`
- `Esc`: cancela; volta pro shell
- Lista virtualizada (renderiza só visíveis); paginação automática em `↑↓`

**Mini-recap expandido (inline):**

```
│    ▶ sess_8a3f2b · 2 days ago · blablabla                          │
│      ✓ done · $0.04 · 15 steps                                     │
│      "refactor queue retry logic"                                  │
│      ┌──────────────────────────────────────────────────────────┐  │
│      │ Resumo:                                                  │  │
│      │ Extraiu computeBackoff em src/queue/backoff.ts; adicionou│  │
│      │ 5 testes; removeu código morto. 4 etapas, todas verdes.  │  │
│      │                                                          │  │
│      │ Files: src/queue.ts, src/queue/backoff.ts, queue.test.ts │  │
│      │ Decisões: extrair pra função pura (testabilidade)        │  │
│      │ Não feito: src/queue-consumer.ts (fora do escopo)        │  │
│      └──────────────────────────────────────────────────────────┘  │
```

**Performance:**

- Lista de 100 sessões pré-cached: render < 50ms
- Mini-recap pré-renderizado em hook `Stop` (cacheado em `recap_cache`); carregamento lazy se ausente (~500ms-1s pra LLM render)
- Sem mini-recap pré-renderizado: fallback determinístico (`<status>: {steps} steps, {files} files, {goal}`)

**Estados visuais:**

- Sessão `incomplete: true` → indicador `⚠ interrupted by crash` (visualmente distinto)
- Sessão `running` (in another instance, raro) → cinza + label `(active in PID N)`
- Sessão `hasErrors: true` mas `done` → ícone secundário discreto

#### `<Interrupt>`
State machine do Ctrl+C.

```ts
interface InterruptHandlerProps {
  state: 'idle' | 'first_press' | 'tool_running'
  toolInProgress?: string
  onInterrupt: () => void
  onForceExit: () => void
}
```

Comportamento:
- 1× Ctrl+C em idle: cancela input atual, mantém sessão
- 1× Ctrl+C em running (sem tool): cancela geração, volta a idle
- 1× Ctrl+C em tool_running: mostra prompt "tool em execução; press Ctrl+C de novo pra forçar"; 5s timeout volta ao normal
- 2× Ctrl+C em tool_running: SIGTERM no tool; 1× mais Ctrl+C: SIGKILL; mais um: exit do agente

Microcopy do double-press:
```
⚠ tool em execução: bash (rm -rf ./build)
  Press Ctrl+C novamente para abortar tool.
  Aguardando 5s...
```

---

## 4. Theme & capability matrix

### 4.1 Camadas de cor

| Detected | Comportamento |
|---|---|
| Truecolor (24-bit) | Paleta cheia, gradientes |
| 256 colors | Paleta curada (16 base + 32 acentos) |
| 16 colors | Apenas básicas (red/green/yellow/blue/magenta/cyan + bright) |
| `NO_COLOR=1` | Zero cor; ASCII puro com prefixos `[!]`, `[?]`, `[OK]`, `[FAIL]` |
| Pipe (não-TTY) | Idem NO_COLOR; output em formato compatível com grep/awk |

Detecção via `supports-color`. Override via env (`FORCE_COLOR=0|1|2|3`, `NO_COLOR`).

### 4.2 Paleta canônica

```ts
// theme/colors.ts
export const colors = {
  // Status
  pending:   { trueColor: '#888', ansi256: 245, ansi16: 'gray' },
  running:   { trueColor: '#5fafd7', ansi256: 74, ansi16: 'cyan' },
  done:      { trueColor: '#5faf5f', ansi256: 70, ansi16: 'green' },
  error:     { trueColor: '#d75f5f', ansi256: 167, ansi16: 'red' },
  denied:    { trueColor: '#d7af5f', ansi256: 179, ansi16: 'yellow' },

  // Profile
  autonomous:   { trueColor: '#5fafff', ansi256: 75, ansi16: 'blue' },
  orchestrated: { trueColor: '#5faf5f', ansi256: 70, ansi16: 'green' },
  hybrid:       { trueColor: '#af5fd7', ansi256: 134, ansi16: 'magenta' },

  // Modal variants
  info: ..., warn: ..., danger: ..., success: ...,

  // Diff
  diffAdd: ..., diffRemove: ..., diffContext: ...,
};
```

Aplicação via `useTheme()` hook que consulta capabilities + retorna função `color('done')`.

### 4.3 Ícones

```ts
// theme/icons.ts
export const icons = {
  pending:   { unicode: '○', ascii: '[ ]' },
  running:   { unicode: '⏳', ascii: '[~]' },
  done:      { unicode: '✓', ascii: '[x]' },
  error:     { unicode: '✗', ascii: '[!]' },
  denied:    { unicode: '⊘', ascii: '[/]' },
  spinner:   { unicode: ['⠋','⠙','⠹','⠸','⠼','⠴','⠦','⠧','⠇','⠏'], ascii: ['|','/','-','\\'] },
  bullet:    { unicode: '·', ascii: '*' },
  arrow:     { unicode: '▶', ascii: '>' },
  // ...
};
```

### 4.4 Detecção e fallback

```ts
const caps = useCapabilities(); // { color, unicode, hyperlinks, width, isTTY, terminal }
caps.color === 0 ? plainText() : caps.color === 1 ? color16() : truecolor();
caps.unicode ? icons.done.unicode : icons.done.ascii;
```

Centralizado. Componentes não detectam por conta própria.

---

## 5. Padrões de interação

### 5.1 Modal queue

Quando estado é `awaiting_user`, exatamente **1 modal** ativo (invariante de §8 STATE_MACHINE.md). Modal próximo só aparece após anterior resolver.

Implementação:
```ts
const useModalQueue = (): { current: PendingDecision | null; resolve: (...) => void };
```

Render condicional: `{current ? <ModalFor(current) /> : <InputPane />}`.

### 5.2 Focus management

Em modal: foco no modal, input pane congela (mostra `_` mas não capturando tecla).
Sem modal: foco no input pane.

Ink `useFocus` hook + `<FocusManager>` wrapper.

### 5.3 Keybindings (cheat sheet)

| Tecla | Comportamento |
|---|---|
| `Ctrl+C` | Interrupt (state machine acima) |
| `Ctrl+D` | Exit (em idle) ou cancela input (digitando) |
| `Ctrl+L` | Limpa tela (não contexto) |
| `Ctrl+Z` | `/undo` shortcut |
| `Ctrl+R` | Search no histórico de prompts |
| `Esc Esc` | Interrompe modelo (mantém input atual) |
| `Tab` | Completion (slash commands, paths) |
| `↑/↓` | Histórico de prompts |
| `Alt+Enter` | Newline em input |
| `Enter` | Envia |
| `?` em modal | Mostra help do modal |
| `/` | Abre slash command picker |

Conflitos com terminal/shell (ex: `Ctrl+S` em alguns terminais): documentar e oferecer alternativa.

### 5.4 Slash command picker

Ao digitar `/` em input pane, abre picker:

```
/ ▼
  /help
  /resume
  /compact
  /cost
  /model
  /clear
  ...
```

Filtragem fuzzy (`Ctrl+R` style). Tab completa parcial. Esc fecha.

Custom commands de `~/.config/agent/commands/` aparecem misturados, com indicador `[user]`.

### 5.5 Scroll & history

Terminal não tem scroll programático confiável (cada terminal trata diferente). Estratégia:

- **Não tentar fazer scroll lógico interno.** History pane é append-only via `<Static>`; terminal nativo cuida do scroll.
- **Comando `/replay <step_id>`** em vez de scroll — re-renderiza step específico no rodapé com `<Less>`-style nav.
- **Aviso quando historico passa de N tokens**: "(scroll terminal pra cima pra ver early steps)" — explícito.

### 5.6 Copy/paste

Terminal cuida (mouse selection ou keyboard). Não interceptamos.

Exception: tool output muito longo é truncated; "press `c` to copy full output to clipboard" via `pbcopy`/`xclip`/`wl-copy` com detecção. Fallback: "save to file" via `/save <step_id> <path>`.

### 5.7 Mouse

**Não usado.** Princípio #4 da §1 do AGENTIC_CLI.md (keyboard-only). Mouse selection nativa do terminal funciona pra copy; nada mais.

---

## 6. Microcopy (templates versionados)

Mensagens ao usuário são **dados**, não strings hardcoded. Template files em `~/.config/agent/messages/<lang>/<code>.md`.

### 6.1 Estrutura comum

```
<emoji> <título curto, ≤ 50 chars>

<o que aconteceu, 1-2 linhas>

<o que tentar — comando concreto OU ação humana>

(detalhes: <code> | /help <code>)
```

### 6.2 Exemplos canônicos

**Provider lento:**
```
⚠ Provider lento (3 timeouts)

Anthropic API não respondeu em 60s × 3 tentativas.
Continuando com aviso. Próximo step pode degradar.

Tente: /model haiku   (modelo mais rápido)
       /pause          (continuar depois)

(detalhes: provider.timeout.streaming | /help provider.timeout)
```

**Tool denied:**
```
✗ Tool bloqueada

bash: "rm -rf /tmp/build"
Negada por: deny rule "rm -rf /*"

Sugestão: rode manualmente e use /resume.

(detalhes: permission.runtime.deny)
```

**Memory write proposed:**
```
📝 Memória proposta (não salva ainda)

feedback / project-scope:
  "console.log proibido em src/; usar logger"

[a]ccept  [e]dit  [r]eject  [w]hy?
```

**Compaction triggered:**
```
⚙ Compactando contexto...

Histórico atingiu 70% do limite.
Resumindo turns antigos. Goal preservado literal.

(~3s)
```

**Crash recovered:**
```
✓ Sessão recuperada

Crash detectado em step 14 (mid-tool). Tool result perdido;
modelo vai re-tentar. Estado consistente.

(detalhes: recovery.tool_exec | trace: traces/sess_abc.ndjson)
```

### 6.3 Tom e estilo

- **Direto**, sem hedge ("might be possible to" → "tente")
- **Acionável**, sem prosa explicativa demais
- **Honesto** sobre falha — "API down" não "hum, parece que..."
- **Sem emoji excessivo** — 1 por mensagem, opcional via `--no-emoji`
- **Português ou inglês** consistente por sessão (`AGENT_LANG=pt|en`)

### 6.4 Anti-microcopy (exemplos do que **não** fazer)

❌ "Algo deu errado, tente novamente"
✅ "Tool `bash` excedeu timeout (30s). Cancelada. Resultado: erro."

❌ "Iniciando..."
✅ "Carregando 14 memórias do escopo project (~50ms)..."

❌ "Operation completed successfully!"
✅ "✓ 4 etapas aplicadas, testes passando."

❌ "Você quer realmente fazer isso?"
✅ "Confirmar: `rm -rf ./build` (deleta 234 arquivos)? [y/N]"

---

## 7. Information density

Calibragem por contexto:

### 7.1 Sessão ativa (high-density)

- Tool cards condensados por default
- Footer compacto
- Diff inline curto (3 lines context)
- TodoList compacto
- Sem decoração extra

### 7.2 Modal (low-density)

- Padding generoso
- Texto centralizado quando faz sentido
- Atalhos visíveis
- Help text (`[w]hy?`) sempre disponível

### 7.3 First-run (medium-density)

- Mensagem de boas-vindas: 5 linhas máximo
- Tip rotativa por sessão (uma de ~20 tips)
- Pointer pra `/help`

### 7.4 Erro fatal (high-emphasis, low-density)

- Centralizado
- Cor de alerta
- Instruções de recovery
- Path pra logs

### 7.5 Streaming (steady-state)

- Cursor de digitação visível
- Spinner em região que não polui
- Throughput não importa pro user; latência percebida sim

---

## 8. First-run UX (do install ao primeiro prompt)

### 8.1 Primeira invocação

Detecção: `~/.config/agent/` não existe.

Fluxo:
```
$ agent

Bem-vindo ao agent (v0.1.0)

Configuração inicial (uma vez só):

  [1/3] API key da Anthropic? (sk-ant-...)
        > _

  [2/3] Modelo padrão?
        ▶ claude-sonnet-4-6 (recomendado)
          claude-haiku-4-5 (mais rápido, mais barato)
          claude-opus-4-7 (mais inteligente, mais caro)
          ollama/qwen2.5-coder:14b (local; profile orchestrated)

  [3/3] Permitir telemetria local? (NDJSON em ~/.local/share/agent/traces/)
        ▶ sim, local apenas
          sim, exportar pra OTEL collector
          não

✓ Configurado. Run `agent` pra começar.
   Tip: `/help` mostra comandos. Ctrl+C cancela. Ctrl+D sai.
```

Sem nada disso em mode CI / `--json` / non-TTY — config via env vars (`AGENT_API_KEY`, etc). Erro fatal claro se faltam.

### 8.2 Primeira sessão num diretório

`<TrustPrompt>` aparece. Mensagem clara, não-aterrorizante.

### 8.3 Primeira tarefa

Após primeiro prompt do user, tip discreta no footer:
```
tip: Ctrl+Z desfaz o último step · /undo · /help
```

Mostra 3× depois some.

### 8.4 Onboarding por "discovery", não tutorial

Sem tutorial guiado obrigatório. Aprendizado por:
- `/help` sempre disponível
- Tab completion mostra comandos
- Footer mostra atalho relevante por contexto
- Mensagens de erro têm "tente: ..."

Tutorial é fricção; descoberta contextual é UX.

---

## 9. Headless mode (NDJSON contract)

`agent --json "prompt"` ou `cmd | agent --json` bypassa Ink completamente.

### 9.1 Output schema

Cada linha em stdout é um JSON object:

```jsonl
{"type":"session_start","session_id":"sess_abc","model":"sonnet-4-6","ts":1714138800}
{"type":"step_start","step_id":"step_1","ts":1714138801}
{"type":"assistant_text","step_id":"step_1","text":"Vou começar lendo..."}
{"type":"tool_call","step_id":"step_1","tool":"glob","input":{"pattern":"*.ts"}}
{"type":"tool_result","step_id":"step_1","output":["src/a.ts","src/b.ts"],"duration_ms":42}
{"type":"step_end","step_id":"step_1","tokens_in":1234,"tokens_out":56,"cost_usd":0.003}
{"type":"session_end","session_id":"sess_abc","status":"done","total_cost_usd":0.012}
```

Schema versionado em `schema_version: "v1"` no primeiro objeto.

### 9.2 Stderr

`stderr` recebe **logs progressuais** (não estruturados):
```
[14:32:01] starting session...
[14:32:02] streaming response...
[14:32:03] tool: glob (42ms)
```

Pode ser silenciado com `-q`.

### 9.3 Modais em headless

Não existem. Todas decisões viram **erro** ou **comportamento default** definido por flag:

```bash
agent --json --on-permission=allow|deny|fail "prompt"
agent --json --on-trust=fail "prompt"           # nunca silenciosamente confia
agent --json --on-memory-write=skip|fail "prompt"
```

Default: `fail` em tudo. CI explícita o que quer.

### 9.4 Exit codes

- `0` — sucesso (sessão `done`)
- `1` — erro de tarefa (sessão `error_fatal`)
- `2` — budget exausto (`exhausted`)
- `3` — denied por policy ou modal sem flag
- `4` — config inválida
- `130` — interrompido (SIGINT, mesmo em headless via timeout)

---

## 10. Acessibilidade

### 10.1 NO_COLOR (obrigatório)

`NO_COLOR=1` ou `--no-color`: zero cor. Prefixos textuais (`[!]`, `[OK]`, etc).

### 10.2 ASCII-only (obrigatório)

Locale C ou `--ascii`: sem Unicode. Spinners ASCII (`|/-\`), bullets `*`, status `[x]`/`[ ]`.

### 10.3 Screen reader friendly (best-effort)

- Não usar caracteres decorativos sem propósito (sem `═══════` separadores; usa `---` ou whitespace)
- Toda informação importante em texto, não só cor
- Modal com role implícito via texto ("Permission required: ...")
- Sem ANSI escape em stdout em modo `--accessible` (forces fully plain text)

### 10.4 Daltonismo

- Status nunca depende **só** de cor — sempre tem ícone/prefixo
- `done` = ✓ verde, mas `✓` é o sinal primário
- `error` = ✗ vermelho, mas `✗` é o sinal primário

### 10.5 Low-vision

- Suporte a `FORCE_LARGE_FONT` (env var custom): aumenta espaçamento, simplifica layout
- Modo `--simple`: layout linear, sem boxes/cards/tables

### 10.6 Latência cognitiva

- Streaming previsível (tokens aparecendo) reduz ansiedade vs spinner opaco
- Cancel sempre visível (`Ctrl+C` no footer)
- Estado atual sempre visível (`▶ glob "*.ts"` enquanto executa)

---

## 11. Internacionalização (placeholder strategy)

v1 suporta apenas **`pt-BR`** e **`en`** (mistos no código atual). Strings em arquivos `messages/<lang>/<code>.md`.

Detecção: `$LANG`, `$LC_ALL`. Override: `--lang pt|en`.

Padrão em projeto: usuário diz "fala português" → memory `feedback` → próxima sessão respeita.

V2: estrutura aberta pra mais idiomas se houver demanda. Não suportar todas as do `gettext` — escopo grande sem ROI.

---

## 12. Performance budget (slice de UX)

Do `PERFORMANCE.md` §2.6, com explicação UX:

| Métrica | P99 | Por que importa |
|---|---|---|
| Input echo (tecla → tela) | < 10ms | < 16ms é "instantâneo"; > 30ms é "laggy" |
| Streaming token render | < 5ms | Frames perdidos = sensação de travamento |
| Modal abre | < 50ms | Modal lento = perde sincronia com decisão |
| Tab completion | < 100ms | Usuário esquece o que digitou |
| Resize re-layout | < 33ms (1 frame) | Resize visual deve ser smooth |
| First paint após startup | < 200ms | "Demora" começa em 250ms |

Violação = regressão de UX, mesmo se funcional ainda funciona.

---

## 13. Testing strategy

### 13.1 Unit (componente isolado)

`ink-testing-library`:

```ts
import { render } from 'ink-testing-library';
import { ToolCallCard } from './ToolCallCard';

test('collapses long output by default', () => {
  const { lastFrame } = render(
    <ToolCallCard
      toolCall={{
        tool: 'grep',
        status: 'done',
        output: Array(50).fill('match').join('\n'),
      }}
    />
  );
  expect(lastFrame()).not.toContain('match\nmatch\nmatch\nmatch\nmatch');
  expect(lastFrame()).toMatch(/collapsed.*expand/i);
});
```

### 13.2 Integration (interação)

```ts
test('Ctrl+C in tool_running shows abort prompt', async () => {
  const { stdin, lastFrame } = render(<App initialState={...} />);
  stdin.write('\x03'); // Ctrl+C
  await delay(50);
  expect(lastFrame()).toMatch(/press Ctrl\+C novamente/i);
});
```

### 13.3 Visual snapshots

```ts
test('PermissionPrompt matches snapshot', () => {
  const { lastFrame } = render(<PermissionPrompt {...props} />);
  expect(lastFrame()).toMatchSnapshot();
});
```

Snapshots em `__snapshots__/`. PR diff visual mostra mudanças antes de merge.

### 13.4 Capability matrix

Roda mesmos componentes com:
- Truecolor + Unicode + hyperlinks
- 256 + Unicode
- 16 + Unicode
- NO_COLOR + ASCII

Eval automatic confirma que **nenhum componente quebra** em qualquer combo.

### 13.5 Headless contract

Eval roda agent em `--json` mode com prompts canônicos; valida JSON Schema do output line-by-line.

### 13.6 Manual playtest

Antes de release: 30min de uso real com cada profile. Loga friction points em `evals/playtest/<version>/notes.md`. Não substituível por test automatizado.

---

## 14. Anti-patterns (não cometa)

| Anti-pattern | Por quê |
|---|---|
| Animação ASCII-art bonita no startup | Atraso em SSH; some valor real |
| Spinner customizado por componente | Inconsistência visual; usa primitiva |
| Modal sem `[w]hy?` opção | User não entende o que está confirmando |
| Cores sem ícones/prefixos | Daltonismo; NO_COLOR; pipe |
| Tool card sem status visível | "Tá rodando?" "Acabou?" | confusion |
| Footer que muda largura/altura | Salta layout; quebra mental model |
| History scrollable internamente | Conflita com terminal nativo; nunca certo em todos os terminais |
| Mouse interactions | Não-portável; quebra em SSH/tmux |
| Mensagens longas multi-parágrafo em modal | Modal é decisão; texto longo vai pra `--why` |
| Microcopy genérica ("erro", "operação completada") | Inacionável |
| Componente que detecta capability sozinho | Inconsistente; centraliza em `useCapabilities` |
| Re-render de history a cada token | Mata perf; usa `<Static>` |
| Modal stack de 3 níveis | Spec não permite (invariante); se cair aqui, é bug |
| Mostrar caminhos absolutos com username (`/home/lex/...`) | PII em screenshot/share; usa `~` ou relativo |
| "Press any key" | Nunca. Especifique a tecla. |

---

## 15. Insight final

Arquitetura define o que o agente **pode** fazer.
UI define o que o usuário **percebe** que ele faz.
Cada uma sem a outra é metade do produto.

A diferença entre uma CLI agentic que vira hábito e uma que vira screenshot no Twitter é UX.

Não tem novidade técnica aqui — Ink resolve a parte React, terminal resolve a parte de output. O trabalho é nas **decisões pequenas e consistentes**: que cor usa pra status, como modal queue, qual microcopy de erro, quanto detalhe no footer, quando colapsar tool card, quando mostrar tip.

Cada decisão pequena soma. Cinquenta decisões pequenas certas = produto que dá vontade de usar. Cinquenta erradas = wrapper de API com terminal feio.

A regra é: **cada componente é uma decisão feita uma vez bem; replicada perfeita em todo lugar.**

---

## 16. Próximos passos (quando partir pra implementação)

1. Definir tema (paleta + ícones) e `useCapabilities` antes de tudo
2. Construir 5 primitivas (`Card`, `Modal`, `Bar`, `Badge`, `Diff`) — base de tudo
3. Layout regions (`<App>` + `<Header>` + `<HistoryPane>` + `<InputPane>` + `<Footer>`)
4. Componentes domain por ordem de criticidade UX:
   - `<ToolCallCard>` + `<StreamingMessage>` (essencial pro loop)
   - `<PermissionPrompt>` + `<TrustPrompt>` (segurança)
   - `<BudgetBar>` + `<MemoryBadge>` (info constante)
   - `<TodoListView>` (UX killer pra tarefas longas)
   - `<DiffView>` (write-heavy workflows)
   - `<MemoryWritePrompt>` (UX correto previne injection)
   - `<DAGProgress>` + `<ValidatorTrace>` (orchestrated only)
   - `<BackgroundProcessTray>` + `<CheckpointBar>` (qualidade-de-vida)
5. Headless path em paralelo (NDJSON writer, sem Ink)
6. Microcopy templates pra todos os codes em `FAILURE_MODES.md`
7. Snapshot tests + capability matrix tests
8. Playtest manual antes de cada milestone

Sem (1)-(2), o resto fica inconsistente. Sem (5), CI quebra silenciosamente. Sem (8), regride no escuro.
