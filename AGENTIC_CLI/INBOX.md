# INBOX

Subsistema de **enfileiramento de mensagens do usuário** no `AGENTIC_CLI`. Cobre como mensagens digitadas enquanto o loop executa um turn são recebidas, persistidas, exibidas, editáveis, e finalmente drenadas como próxima user turn. Estratégia canônica: **fila FIFO simples drenada em turn boundary** (estilo Claude Code). Outras estratégias (checkpoint mid-turn, steering, cancel-and-replay) ficam fora deste doc — coexistem mas são owners separados.

Não é "fila de eventos". Eventos são `audit_timeline`. Não é "buffer de prompt". Buffer é estado da TUI. Inbox é **mailbox tipado, persistido, auditável**, com semântica de delivery declarada. Sem isso, "envia enquanto trabalha" vira UX que come mensagem em crash, duplica em race, ou drena fora de ordem.

Premissa raiz herdada: *meça duas vezes, corte uma*. Aqui a medição é **delivery lag** — quanto tempo entre `received_at` e `drained_at`. Sem essa métrica, "responsividade do agent" é folclore.

---

## 0. Princípios (não-negociáveis)

1. **Inbox é estado externo ao modelo.** Modelo nunca consulta inbox, nunca tem tool `inbox_read`. Vê só o efeito: msg drenada vira próxima user turn. Mesma fronteira herdada de [`FEEDBACK_ADAPTATION §0.3`](./FEEDBACK_ADAPTATION.md) e [`CONTRACTS §2.6.8`](./CONTRACTS.md): meta-cognição harness-only.
2. **Turn boundary determinístico.** Drain dispara em condição precisa de [`STATE_MACHINE`](./STATE_MACHINE.md), não em heurística temporal. Sem critério declarado, drain vira não-determinístico e replays divergem.
3. **Auditabilidade total.** Toda transição (received, edited, cancelled, drained) em `inbox_events`. Lag medido. Sem registro = bug.
4. **Visibilidade obrigatória.** Contagem de pendentes sempre exibida na TUI. Sem indicador, usuário duplica msgs ou crê que sumiram — pior UX possível.
5. **Persistência via WAL.** Inbox sobrevive a crash do processo. Resume oferece drenar. Inbox in-memory-only é receita pra perder input em momento ruim.
6. **Sem priority queue.** Estratégia (1) é FIFO puro. Prioridade/interrupção é problema da estratégia (2) ([`STATE_MACHINE §X`](./STATE_MACHINE.md)), não desta. Misturar = bug semântico difícil.
7. **Drain semantics declarada.** Múltiplas msgs antes do drain: **concat** com separador explícito (§5). Não inferida pelo modelo, não decidida em runtime.
8. **ESC é escape hatch separado.** Cancel hard do turn em curso não é função da inbox. Coexistem (§12). Sobrecarregar inbox com cancel hard polui semântica.
9. **Subagent não herda inbox do parent.** Handoff começa com inbox vazia; msgs pro parent ficam pendentes até subagent retornar. Caso contrário, msg destinada ao parent é entregue ao filho — vetor de confusão e de injection.
10. **Headless é fail-closed.** Em `--json` non-interactive, inbox **desativada** por default. Opt-in explícito via `--allow-inbox` pra CI específicas que sabem o que fazem.

---

## 1. Escopo

### 1.1 O que resolve

- "Como aceitar input do usuário enquanto loop está em tool call ou model gen?"
- "Onde exatamente a msg entra no fluxo sem corromper turn em curso?"
- "O que acontece com msgs pendentes se processo crasha?"
- "Como o usuário sabe que a msg foi recebida, e edita/cancela antes do drain?"
- "Por que essa msg foi drenada agora e não antes?" (auditabilidade)

### 1.2 O que **não** resolve

- **Cancelamento hard do turn em curso.** Owner: orchestrator + cancellation token. ESC ≠ inbox (§12.1).
- **Steering / side-channel mid-stream.** Owner: estratégia (3), fora deste doc. Modelo receber "hint" durante geração é problema diferente.
- **Interrupção em checkpoint mid-turn.** Owner: estratégia (2), candidata a doc próprio. Requer flag de interrupt + checks em step terminator com semântica diferente.
- **Compaction / context shaping.** Owner: [`CONTEXT_TUNING`](./CONTEXT_TUNING.md). Inbox drenada vira user turn normal; compaction aplica como em qualquer turn.
- **Histórico de mensagens.** Owner: [`STATE_MACHINE`](./STATE_MACHINE.md) (transcript). Inbox é entrada; transcript é histórico pós-drain.
- **Comandos slash.** `/inbox list` etc. são UI deste doc, mas o roteador é o CLI geral.

### 1.3 Não-objetivos explícitos

- **Responsividade < 1s em loops longos.** Estratégia (1) tem lag = duração do turn restante. Se você precisa disso, é estratégia (2) ou (3), não esta.
- **Reordenação de msgs.** Drain é FIFO. Sem reorder.
- **Cross-session inbox.** Inbox é per-session. Msg em sessão A não cruza pra sessão B; rope de confusion e race.

---

## 2. Turn boundary

Definição precisa — load-bearing pra determinismo do drain.

### 2.1 Critério

Turn boundary = **último step terminou e nenhuma tool call está pendente**, registrado em [`STATE_MACHINE`](./STATE_MACHINE.md).

```
step end hook:
  if state.tool_calls_pending > 0:    # ainda no meio do turn
    return CONTINUE
  if state.last_step.kind == 'model_text_only':
    return TURN_BOUNDARY               # drain inbox aqui
  if state.last_step.kind == 'error' or 'abort':
    return TURN_BOUNDARY               # estado degradado, drena assim mesmo
  return CONTINUE                      # default conservador
```

### 2.2 O que **não** é boundary

- Entre dois tool calls dentro do mesmo turn: **não**. Modelo ainda está raciocinando, drenar aqui é estratégia (2).
- Streaming chunk recebido sem tool call: **não**. Modelo ainda pode emitir tool call no chunk seguinte.
- Compaction event: **não**. Compaction roda dentro do turn; não é fronteira de input.
- Handoff pra subagent: **não pro parent**. Boundary do subagent é independente; inbox do parent espera o subagent retornar.

### 2.3 Boundary em estados degradados

| Estado | Drena? | Razão |
|---|---|---|
| Turn termina com erro de tool | sim | user provavelmente quer corrigir; segurar piora UX |
| Turn termina com timeout do modelo | sim | mesma lógica |
| Turn termina com abort manual (ESC) | sim | drena após cancel hard (§12.1) |
| Turn termina com permission denied | sim | user quer rever decisão |
| Crash do processo | n/a | drain acontece em resume (§4.5) |

Inbox nunca fica retida por estado degradado. Segurar msg pendente porque turn falhou ⇒ usuário desiste e duplica.

---

## 3. Estados do item

| Estado | Significado | Reversível? |
|---|---|---|
| `pending` | recebida, aguardando boundary | sim (edit, cancel) |
| `editing` | item aberto em $EDITOR ou prompt expand | sim |
| `cancelled` | cancelada antes do drain; fica em audit | não auto |
| `drained` | entregue como user turn; terminal | terminal |
| `expired` | sessão acabou antes do drain (raro) | terminal |

Transições legais:

```
pending ─► editing ─► pending
pending ─► cancelled                (terminal)
pending ─► drained                  (terminal)
pending ─► expired                  (terminal, session end)
editing ─► cancelled                (terminal)
```

`drained` não volta. Mensagem drenada virou turn; está no transcript, governado por [`STATE_MACHINE`](./STATE_MACHINE.md).

---

## 4. Pipeline canônico

```
[user digita + enter]
        │
        ▼
┌─ receive ─────────────────────────────────┐
│ allocate msg_id                           │
│ persist em inbox_items (status=pending)   │
│ inbox_events(received)                    │
│ TUI atualiza contador                     │
└──────────────────┬────────────────────────┘
                   │
                   │  (loop continua sem ser afetado)
                   │
                   ▼
┌─ awaiting boundary ───────────────────────┐
│ usuário pode:                             │
│   - editar (↑ ou /inbox edit)             │
│   - cancelar (ESC msg ou /inbox cancel)   │
│   - adicionar novas msgs                  │
└──────────────────┬────────────────────────┘
                   │
                   ▼
┌─ STATE_MACHINE step end ──────────────────┐
│ §2.1 critério satisfeito?                 │
└──────────────────┬────────────────────────┘
                   │ sim
                   ▼
┌─ drain ───────────────────────────────────┐
│ load all pending in FIFO order            │
│ concat com separador (§5)                 │
│ submit como próxima user turn             │
│ mark cada item como drained               │
│ inbox_events(drained) + lag_ms            │
│ TUI limpa contador                        │
└───────────────────────────────────────────┘
```

### 4.1 Receive

```
on user_input_committed(body):
  if profile == 'headless' and not flag('--allow-inbox'):
    reject("inbox disabled in headless mode")
    return
  if size(body) > MAX_MSG_BYTES:
    reject("msg too large")
    return
  msg_id = uuid()
  insert inbox_items(msg_id, body, status='pending', received_at=now())
  emit inbox_events(received, msg_id, hash(body))
  tui.update_counter(inbox.pending_count())
```

WAL fsync **antes** de confirmar receipt na TUI. UX vê `📥` só quando msg está durável.

### 4.2 Edit

User aperta ↑ ou roda `/inbox edit <n>`:

```
mark item.status = editing
open $EDITOR with item.body
on save:
  diff = compute_diff(old, new)
  emit inbox_events(edited, msg_id, hash_before, hash_after)
  item.body = new
  item.status = pending
on cancel:
  item.status = pending  (sem mudança)
```

Edit não muda `received_at` — ordem FIFO preservada. Editar a msg #2 não a faz pular pra frente da #1.

### 4.3 Cancel

```
on /inbox cancel <n> or ESC-on-msg:
  if item.status not in [pending, editing]:
    error("can't cancel %s msg", status)
  item.status = cancelled
  item.cancelled_at = now()
  emit inbox_events(cancelled, msg_id)
  tui.update_counter()
```

`/inbox clear` cancela todas em uma operação atômica.

### 4.4 Drain

```
on turn_boundary:
  items = select * from inbox_items
          where session_id = current
            and status = 'pending'
          order by received_at asc
  if items is empty:
    return  # boundary sem inbox = prompt síncrono normal
  body = concat_with_separator(items)        # §5
  user_turn_id = state_machine.start_turn(body, source='inbox_drain')
  for item in items:
    item.status = 'drained'
    item.drained_at = now()
    item.drained_into_turn = user_turn_id
    emit inbox_events(drained, msg_id, lag_ms = drained_at - received_at)
  tui.clear_counter()
```

### 4.5 Resume após crash

```
on session_resume:
  pending = inbox_items where status in [pending, editing]
  if pending is empty:
    return
  prompt user: "tinha N msgs pendentes da sessão anterior. Drenar agora?"
  user choice:
    drain  ─► comporta como §4.4 imediatamente
    cancel ─► todas viram cancelled com motivo='resume_declined'
    review ─► abre /inbox list pra decisão item-a-item
```

WAL garante que nenhuma msg confirmada some. Msgs em `editing` ao crash voltam pra `pending` com flag `was_editing`.

---

## 5. Drain semantics: concat com separador

Decisão de UX declarada, não inferida.

### 5.1 Regra

Múltiplas msgs pendentes ⇒ **uma user turn** com corpo:

```
<msg 1>

---

<msg 2>

---

<msg 3>
```

Separador é `\n\n---\n\n` (linha de regra horizontal markdown). Modelo trata como turn com pontos múltiplos.

### 5.2 Por que concat (não sequential turns)

- **N turns sucessivos** drenando 1 msg cada significa: modelo gera, executa, drena próxima, gera de novo. Cada turn tem custo (model call, possivelmente tool calls). Em fila de 3 msgs, custo 3×.
- Drains sucessivos passam por compaction independente; contexto fica fragmentado.
- User intent prático é "também faz X" — semanticamente um turn.

### 5.3 Por que separador visível

- Modelo precisa ver fronteira pra não fundir "obrigado" + "mas erra Y" em um pensamento só.
- User no transcript posterior consegue distinguir o que veio junto.
- Audit em `inbox_events` registra `drained_into_turn` por msg_id; transcript do turn mostra cada msg como bloco.

### 5.4 Override

User pode forçar drain sequencial via `/inbox drain --sequential` antes do boundary. Caso raro; default é concat.

### 5.5 Limite

Concat tem limite de tamanho (default 32KB no body agregado). Estouro ⇒ drena os primeiros N que cabem, mantém resto como `pending` pro próximo boundary. Avisa user.

---

## 6. TUI / input dual-channel

### 6.1 Layout

```
┌──────────────────────────────────────────────┐
│ [stream do loop principal]                   │
│  - model output                              │
│  - tool calls                                │
│  - status updates                            │
│  ...                                         │
│                                              │
├──────────────────────────────────────────────┤
│ 📥 2 queued · ↑ edit · ESC cancel last       │  ← status line
│ > user types here_                           │  ← input line
└──────────────────────────────────────────────┘
```

- Output do loop **acima**, scroll independente
- Input line **abaixo**, fixa, nunca sobrescrita
- Status line entre as duas, atualiza com contador

### 6.2 Modo de input

- stdin em raw mode + poll/select async
- Linha de input gerenciada por TUI lib (prompt_toolkit, rustyline, etc.)
- Enter committa pra inbox (chama §4.1)
- ↑/↓ navega histórico de inbox **da sessão atual** (não histórico geral do shell)

### 6.3 Indicadores

| Estado | Display |
|---|---|
| 0 pending | (nada; só prompt normal) |
| 1+ pending | `📥 N queued` + atalhos |
| editando | `📝 editing msg <n>` + `:wq` hint |
| drain em curso | `📤 draining N msgs...` (breve, < 100ms) |
| crash recovery | `⚠ N msgs pendentes da sessão anterior · /inbox review` |

### 6.4 Sem indicador = bug

Health check de UX: contagem sempre visível quando `pending > 0`. Violação ⇒ bug fatal; sem indicador, usuário duplica msg e pipeline FIFO drena duas vezes.

---

## 7. Comandos slash

```
/inbox list                  # lista pendentes com cursor + preview
/inbox show <n>              # mostra msg n completa
/inbox edit <n>              # abre $EDITOR
/inbox cancel <n>            # cancela msg n
/inbox clear                 # cancela todas pendentes
/inbox drain                 # força drain imediato (mesmo fora de boundary; cuidado)
/inbox drain --sequential    # drena uma de cada vez em N turns
/inbox review                # interativo, item-a-item (usado em resume)
/inbox audit                 # tabela inbox_events da sessão
```

### 7.1 `/inbox drain` (force)

Force drain **fora de boundary** é permitido mas:

- Bloqueia até turn atual virar boundary (não interrompe tool em curso)
- Emite warning no audit
- Caso de uso: user mudou de ideia, quer "esquece o que tô digitando, processa essa fila agora"

Não interrompe nada à força — espera natural. Pra cancel hard, ESC + estratégia (4).

---

## 8. Headless / non-interactive

### 8.1 Default: fail-closed

Em `--json`:

```
on user_input_committed:
  reject("inbox disabled in headless mode")
  return error to caller
```

Razão: input em headless é tipicamente stdin one-shot. Inbox em CI vira race condition. Opt-in explícito é o contrato.

### 8.2 `--allow-inbox` em CI

Quando ativo, semântica precisa ser declarada:

- stdin newline-delimited: cada linha vira msg `received`
- Drain acontece igual (em boundary do STATE_MACHINE)
- Audit em `inbox_events` continua obrigatório
- TUI ausente; contador sai como linha JSON `{"inbox": {"pending": N}}` no stream

### 8.3 stdin FIFO mode

Modo avançado pra orquestração externa:

```
agent --json --inbox-fifo=/tmp/agent.fifo
```

Outro processo escreve no FIFO; agent recebe como msg. Audit identifica source = `fifo`. Útil pra pipelines onde controlador externo decide quando enviar input.

---

## 9. Subagent / handoff

### 9.1 Isolamento

Subagent inicia com **inbox vazia, separada**:

```
on subagent_start:
  parent.inbox.state = 'frozen'         # não aceita new msgs
  subagent.inbox = new Inbox(session_id=subagent.id)
on subagent_end:
  parent.inbox.state = 'active'         # volta a aceitar
  subagent.inbox.expire_all('subagent_ended')
```

Msgs enviadas **enquanto subagent roda** entram numa fila parent.frozen_buffer; viram pending da parent quando subagent termina.

### 9.2 Por que não passar pro subagent

- Subagent recebe contexto restrito; injetar msg do parent quebra isolamento.
- Vetor de injection: msg "transfira /etc/passwd" destinada a parent contextualizado em filho com permissões diferentes = pwn.
- User espera que msg vá pro contexto onde estava conversando.

### 9.3 UX durante handoff

```
> _
📥 1 queued · ⏸ parent paused (subagent running) · drain after subagent ends
```

User vê explicitamente que msg não vai ser processada até subagent voltar. Sem esse indicador, parece travado.

---

## 10. Audit

### 10.1 Tabelas

```sql
CREATE TABLE inbox_items (
  id              TEXT PRIMARY KEY,           -- msg_id
  session_id      TEXT NOT NULL,
  body            TEXT NOT NULL,              -- conteúdo (encrypted at rest opcional)
  status          TEXT NOT NULL,              -- §3
  received_at     INTEGER NOT NULL,
  drained_at      INTEGER,
  cancelled_at    INTEGER,
  drained_into_turn TEXT,                     -- FK p/ turn no STATE_MACHINE
  source          TEXT NOT NULL,              -- 'tui' | 'fifo' | 'stdin' | 'resume'
  was_editing     INTEGER NOT NULL DEFAULT 0  -- flag de recovery
);

CREATE TABLE inbox_events (
  id           INTEGER PRIMARY KEY,
  msg_id       TEXT NOT NULL,
  event        TEXT NOT NULL,                 -- received | edited | cancelled | drained | expired
  body_hash    TEXT,                          -- só hash; conteúdo em inbox_items
  details_json TEXT,                          -- lag_ms, edit diff hash, motivo, etc
  recorded_at  INTEGER NOT NULL
);

CREATE INDEX idx_inbox_session ON inbox_items(session_id, status);
CREATE INDEX idx_inbox_events_msg ON inbox_events(msg_id, recorded_at);
```

### 10.2 Queries canônicas

```sql
-- "Por que a msg X demorou tanto pra drenar?"
SELECT received_at, drained_at, drained_at - received_at AS lag_ms
FROM inbox_items WHERE id = ?;

-- "Quantas msgs canceladas vs drenadas nesta sessão?"
SELECT status, COUNT(*) FROM inbox_items
WHERE session_id = ? GROUP BY status;

-- "Qual o lag p95 nesta semana?"
SELECT percentile_cont(0.95) WITHIN GROUP (ORDER BY drained_at - received_at)
FROM inbox_items
WHERE status = 'drained' AND drained_at > ?;
```

### 10.3 Retention

- Items `drained`: mantidos enquanto session existe; arquivados com a session.
- Items `cancelled`/`expired`: mantidos 30d, depois purgados (conteúdo) deixando metadata em `inbox_events`.
- Privacy purge: hook em `inbox_items` permite redact por session_id (compliance).

### 10.4 Hash do body

Em `inbox_events`, registra-se só `body_hash`. Conteúdo fica em `inbox_items`. Razão: events tem retention longo; bodies podem conter sensível. Separar permite redactar conteúdo sem perder timeline.

---

## 11. Métricas

| Métrica | O que mede | Threshold |
|---|---|---|
| `inbox.lag_ms_p50` | mediana entre received e drained | informativo |
| `inbox.lag_ms_p95` | p95 do lag | > 30s ⇒ considerar estratégia (2) checkpoint |
| `inbox.cancel_rate` | % cancelled / received | > 20% ⇒ UX ruim; user desiste de mensagens |
| `inbox.edit_rate` | % editadas antes do drain | informativo (alto = user pensando muito tempo) |
| `inbox.drain_size_avg` | msgs por drain | > 3 consistente ⇒ turns muito longos |
| `inbox.expired_count` | msgs pendentes em session end | > 0 sem crash ⇒ bug de drain |
| `inbox.resume_drain_rate` | % crash-recovered que dreram | < 50% ⇒ UX de resume confusa |
| `inbox.headless_rejected` | tentativas em --json sem flag | informativo |

`lag_p95 > 30s` é o sinal-chave: estratégia (1) começa a doer. Se persistir, upgrade pra (2).

---

## 12. Coexistência com outras estratégias

### 12.1 ESC (cancel hard do turn)

ESC **não envolve inbox**:

```
ESC pressionado durante turn:
  cancellation_token.fire()
  turn em curso aborta (tool em execução tenta graceful stop)
  transcript marca turn como aborted
  STATE_MACHINE chega em boundary degradado
  inbox drena normalmente (§2.3)
```

Sequência típica:

```
user digita msg 1 ─► inbox pending
loop continua tool call lento
user pressiona ESC ─► cancel hard do turn
boundary alcançado (degradado)
inbox drena msg 1 como próxima user turn
```

Inbox não causa o cancel; ESC causa. Inbox só drena depois. Sobrecarregar inbox com semântica de cancel quebra (§0.8).

### 12.2 Upgrade pra estratégia (2) checkpoint

Quando lag p95 fica inaceitável, estratégia (2) pode coexistir:

- Inbox adiciona campo `priority` (default `normal`)
- Msg com `priority=interrupt` checa em step terminator, não em turn boundary
- Demais campos/audit/UI ficam iguais

Esse caminho é **extensão** deste doc, não substituição. Default permanece (1) FIFO simples.

### 12.3 Steering (estratégia 3)

Não usa inbox. Steering é canal separado:

- Msg vai pra `steering_buffer`, não pra `inbox_items`
- Modelo recebe como side-channel no próximo tool_result
- Audit em tabela `steering_events`, separada

Coexistência: user escolhe UI explicitamente (`/say` pra inbox normal, `/hint` pra steering). Sem distinção, semântica vira loteria.

---

## 13. Anti-patterns

- ❌ **Modelo consulta inbox via tool.** Vira meta-cognição, racionaliza em cima, contamina turn. §0.1.
- ❌ **Drain entre tool calls do mesmo turn.** Isso é estratégia (2), não (1). §0.6, §2.2.
- ❌ **Sem contador visível.** User duplica msg; FIFO drena 2× a mesma intenção. §6.4.
- ❌ **Inbox in-memory only.** Crash come msg confirmada. §0.5.
- ❌ **Priority queue na (1).** "Só essa msg passa na frente" subverte FIFO + audit. Se precisar, é (2). §0.6.
- ❌ **Drain individual (sequential default).** N turns separados, contexto fragmentado, custo N×. §5.2.
- ❌ **ESC dispara drain.** ESC é cancel hard; drain é orthogonal. §12.1.
- ❌ **Subagent vê inbox do parent.** Quebra isolamento + vetor de injection. §9.2.
- ❌ **Headless aceita inbox por default.** CI race. §8.1.
- ❌ **Body em `inbox_events`.** Retention longo + dados sensíveis. Hash, sempre. §10.4.
- ❌ **Drained item volta pra pending.** `drained` é terminal; "undrain" não existe. Re-envio é msg nova. §3.
- ❌ **Force drain interrompe tool em curso.** Force drain espera turn boundary; só dispensa o "esperar usuário digitar". §7.1.

---

## 14. Cross-refs

- [`STATE_MACHINE.md`](./STATE_MACHINE.md) — owner de turn boundary; step terminator chama §2.1.
- [`CONTEXT_TUNING.md`](./CONTEXT_TUNING.md) — drain vira user turn normal; compaction aplica depois.
- [`CONTRACTS.md §2.6.8`](./CONTRACTS.md) — meta-cognição harness-only (herdado).
- [`FEEDBACK_ADAPTATION.md §0.3`](./FEEDBACK_ADAPTATION.md) — princípio "harness-only, invisível ao modelo" (herdado).
- [`PERMISSION_ENGINE.md`](./PERMISSION_ENGINE.md) — turn drenado passa por permission engine normal; inbox não bypassa.
- [`AUDIT.md`](./AUDIT.md) — `inbox_events` indexado aqui pra audit.
- [`SECURITY_GUIDELINE.md`](./SECURITY_GUIDELINE.md) — body em `inbox_items` pode ser candidato a encryption at rest; redact flow.
- [`EVICTION.md`](./EVICTION.md) — `expired` items em session end podem ser purged via mecanismo geral; cancelled/drained têm retention própria (§10.3).
- [`ORCHESTRATION.md`](./ORCHESTRATION.md) — subagent handoff freeze do parent (§9.1) integra aqui.
- [`UI.md`](./UI.md) — layout dual-channel, indicadores, atalhos (§6).
- [`RECAP.md`](./RECAP.md) — lag p95, cancel rate aparecem em recap semanal.

---

## 15. Insight final (não-instrução, contexto)

Inbox parece feature de UX, mas é spec de **fronteira de input**. Sem ela declarada, "envia enquanto trabalha" é qualidade emergente do TUI — quebra em crash, race em handoff, drena fora de ordem, come msg em CI. Com ela declarada, vira contrato: msg confirmada é durável, drain é determinístico, audit é completo, e usuário sempre vê o que está pendente.

O que separa inbox boa de inbox ruim não é a estrutura de dados — é **onde drena**. Step terminator do `STATE_MACHINE` é o único ponto que garante coerência. Drenar antes corrompe turn; drenar depois é tarde demais; drenar em timer é não-determinístico. Boundary preciso é toda a engenharia que importa.
