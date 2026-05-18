# EVICTION

Subsistema de **lifecycle e despejo** do `AGENTIC_CLI`. Consolida governance de objetos persistentes ou semi-persistentes que competem por contexto, peso de ranking ou autoridade comportamental: memory entries, policies adaptadas, candidatos de retrieval, slots de contexto. Não é "GC de memória" — é **decisão tipada de despejo**, com motivo, escopo, reversibilidade e auditoria.

Não é decay. Decay é função do tempo; eviction é função de **evidência**. Tempo entra como input em alguns triggers (`expires`, half-life de session), nunca como mecanismo único. Sem essa separação declarada, "esquecer" vira folclore — coisa some, ninguém sabe por quê, ninguém consegue reverter.

Premissa raiz herdada: *meça duas vezes, corte uma*. Aqui o que se mede é **utilidade operacional residual** de cada objeto sob governance; o corte é a transição de estado, sempre commit-like.

---

## 0. Princípios (não-negociáveis)

1. **Eviction é evento, não função.** Toda saída de estado terminal (`evicted`, `invalidated`, `purged`) tem `id`, `parent`, `motivo`, `evidence`, `actor`. Sem isso, vira "sumiu do disco" e debugging é impossível.
2. **Lifecycle tipado.** Cada substrato tem estados declarados e transições legais. Não há "qualquer coisa pode virar qualquer coisa". State machine explícita em `§4`.
3. **Reversibilidade default.** Estado terminal não significa delete físico. `evicted` ≠ `purged`. Tombstone permite restore enquanto janela de retenção válida (`§7`).
4. **Motivo declarado, não inferido.** Eviction sem `motivo ∈ {irrelevant, conflict, shift, low_roi, quota, expired, user_purge, security}` é bug. Cada motivo tem evidence schema próprio.
5. **Não-decay.** Half-life sobre objeto persistente é proibido. Decay aplicável só onde escopo é `session` e tier é 4-5 (herdado de [`FEEDBACK_ADAPTATION.md §7.2`](./FEEDBACK_ADAPTATION.md) e [`RETRIEVAL.md §0.10`](./RETRIEVAL.md)). Detalhamento em `§8`.
6. **Quarantine antes de eviction.** Failure burst ou conflito não vai direto pra `evicted`. Passa por `quarantined` — visível, logado, re-avaliável no loop frio. Eviction só após segunda evidência ou TTL de quarentena estourado.
7. **Ownership preservado.** EVICTION **propõe** transições; substrato dono executa. Memory entry vira `evicted` via fluxo de [`MEMORY.md §6`](./MEMORY.md); policy vira `quarantined` via [`FEEDBACK_ADAPTATION.md §3.2`](./FEEDBACK_ADAPTATION.md). Este doc é o **contrato de transição**, não o executor.
8. **Auditabilidade total.** Toda transição em `eviction_events` (`§9`). Modelo pode consultar "essa memory entry foi evictada quando, por quê, com que evidência?" — resposta determinística.
9. **Sem GC silencioso.** Despejo automático sem registro em `eviction_events` é violação. Hooks bloqueáveis em `Eviction` event permitem audit externo (compliance, forensics).
10. **Eviction é meta-cognição harness-side.** Modelo nunca decide o que evictar e nunca é notificado da eviction em si. Vê só o efeito (memory entry sumiu do índice; candidate sumiu do slot). Herdado de [`CONTRACTS.md §2.6.8`](./CONTRACTS.md): meta-cognição não é tool.

---

## 1. Escopo

### 1.1 O que resolve

- "Quando um objeto persistente deixa de merecer existir?"
- "Qual a diferença entre 'cortar do contexto agora' e 'apagar para sempre'?"
- "Como reverter uma eviction que se mostrou prematura?"
- "Quem decide despejo: memory engine, retrieval, compaction, ou um árbitro acima?"
- "Por que esse objeto sumiu?" (auditabilidade)

### 1.2 O que **não** resolve

- **Geração/escrita de memória.** Owner: [`MEMORY.md §5`](./MEMORY.md). Admissão tem critério próprio; este doc cobre só saída.
- **Geração/promoção de policy.** Owner: [`FEEDBACK_ADAPTATION.md §3.2`](./FEEDBACK_ADAPTATION.md). Eviction recebe `quarantined` como input, não decide promoção.
- **Ranking de candidatos.** Owner: [`RETRIEVAL.md §5`](./RETRIEVAL.md). Eviction define **quando** candidate ranqueado é cortado do slot final; ranking decide ordem.
- **Compaction de contexto in-session.** Owner: [`CONTEXT_TUNING.md §12`](./CONTEXT_TUNING.md). Compaction é eviction efêmera de turn-context; este doc cobre persistente. Compaction consulta EVICTION pra critério (`§5.3`).
- **Backup/snapshot.** Owner: ops/infra. Tombstone aqui ≠ snapshot. Restore via `eviction_events` é local; recovery de disco corrompido é fora de escopo.

### 1.3 Não-objetivos explícitos

- **"Esquecimento total".** Mesmo `purged` deixa metadata em `eviction_events`. Privacy purge real (GDPR-style) é flow separado documentado em [`SECURITY_GUIDELINE.md`](./SECURITY_GUIDELINE.md) — usa este doc como hook, não como mecanismo.
- **Eviction proativa "preventiva".** Nada é evictado por suspeita. Eviction sem evidência registrada é proibida (§0.4).
- **Cross-substrate eviction cascading automática.** Evictar policy não auto-evicta memory entries que a citam. Cascading é proposta no loop frio, decisão explícita.

---

## 2. Substratos sob governance

Quatro substratos com lifecycle declarado. Cada um tem owner próprio; EVICTION define **contrato comum** de transições e auditoria.

| Substrato | Owner | Persistência | Granularidade |
|---|---|---|---|
| **Memory entry** | [`MEMORY.md`](./MEMORY.md) | cross-session (disk) | arquivo `.md` |
| **Adaptation policy** | [`FEEDBACK_ADAPTATION.md`](./FEEDBACK_ADAPTATION.md) | cross-session (SQLite) | row em `policies` |
| **Retrieval candidate** | [`RETRIEVAL.md`](./RETRIEVAL.md) | per-query (efêmero) | nó no slot proposto |
| **Context slot item** | [`CONTEXT_TUNING.md`](./CONTEXT_TUNING.md) | per-turn (efêmero) | bloco no prompt |

### 2.1 Persistente vs efêmero

- **Persistente (memory, policy):** eviction tem peso. Tombstone obrigatório (`§7`). Reversível por janela de retenção declarada.
- **Efêmero (candidate, slot item):** eviction é "não entrou nesse turn". Trail registrado (`§5.3`), mas não persistido em `eviction_events` por default — geraria volume sem valor. Persiste só se `motivo ∈ {conflict, security}` ou se `--trace-eviction` ativo.

Distinção load-bearing: tratar candidate como memory entry inflaria storage; tratar memory entry como candidate causaria perda silenciosa.

### 2.2 Por que não unificar storage

Tentação: tabela única `evictable` com discriminator. Razão pra **não**:

- Cada owner já tem schema otimizado pro próprio acesso (memory = filesystem + index, policy = SQLite indexed, candidate = in-memory).
- Cross-substrate query é raro (debug); cross-substrate write é antipattern (acopla owners).
- `eviction_events` é a **vista unificada** pra audit/debug. Storage continua dividido.

---

## 3. Estados canônicos

Mesmos nomes em todos os substratos. Owner mapeia pra coluna/frontmatter próprio.

| Estado | Significado | Visível ao modelo? | Conta pra ranking? |
|---|---|---|---|
| `proposed` | criado mas não admitido (gate pendente) | não | não |
| `active` | em uso normal | sim | sim |
| `shadow` | aplicado mas não vinculante (modo dry-run) | não direto; efeito logado | não |
| `quarantined` | suspeito; evidência conflitante ou failure burst | sim (com flag) | não |
| `invalidated` | invariante quebrado (shift, source removido) | não | não |
| `evicted` | despejado; reversível dentro de janela | não | não |
| `purged` | removido permanentemente; só metadata em `eviction_events` | não | não |

### 3.1 Por que `shadow` é estado, não flag

Tentação: tratar shadow como `active + shadow=true`. Razão pra **não**: shadow muda semântica de ranking (entra mas não conta) e de promoção (não dispara accumulation trigger). Modelar como estado próprio evita propagação de bug em pontos que esquecem do flag.

### 3.2 Por que `evicted` ≠ `purged`

`evicted` significa "fora do circuito ativo". `purged` significa "metadata-only, conteúdo foi". Separação permite:

- **Restore barato** dentro de janela de retenção (§7.1)
- **Compliance purge** com prova de irrecuperabilidade quando exigido (§7.4)
- **Forensics** sobre o que existiu (sem expor conteúdo) mesmo após `purged`

---

## 4. State machine

```
                            ┌─────────┐
                            │proposed │
                            └────┬────┘
                                 │ admission gate (owner-specific)
                                 ▼
                    ┌────────► active ◄────────┐
                    │            │             │
        promote/    │            │ accumulate  │ re-promote
        validate    │            │ outcome     │ (loop frio)
                    │            ▼             │
                    │        shadow ───────────┘
                    │            │
                    │            │ ci_low > threshold (FEEDBACK §5.3)
                    │            ▼
                    │         active
                    │
                    │  failure burst | user override 3× | conflict detect
                    ▼
              ┌──────────────┐
              │ quarantined  │ ◄── re-quarantine
              └──────┬───────┘
                     │
       ┌─────────────┼─────────────┐
       │             │             │
   evidence       TTL stale    shift detected
   restored       (quarantine)  (FEEDBACK §7.3)
       │             │             │
       ▼             ▼             ▼
   active        evicted     invalidated
                     │             │
                     │             │ source removed |
                     │             │ trust revoked
                     │             ▼
                     │         evicted
                     │             │
                     │             │
                     ▼             ▼
                  retention window ticking
                     │             │
                     │             │ window expired |
                     │             │ user_purge | security
                     ▼             ▼
                          purged
```

### 4.1 Transições legais (tabela canônica)

| De → Para | Trigger | Motivo permitido | Reversível? |
|---|---|---|---|
| `proposed → active` | admission gate passou | — | sim (rollback) |
| `proposed → evicted` | admission gate falhou | `irrelevant`, `low_roi` | sim |
| `active → shadow` | distribution shift `unstable` | `shift` | sim |
| `active → quarantined` | failure burst, user override 3×, conflict | `conflict`, `low_roi` | sim |
| `active → invalidated` | invariante quebrado (source ausente, schema mudou) | `shift`, `security` | não auto |
| `shadow → active` | scope estabilizou + posterior reconfirmado | — | sim |
| `shadow → quarantined` | shadow divergiu de default em N runs | `conflict` | sim |
| `quarantined → active` | evidência nova restaura confiança (loop frio) | — | sim |
| `quarantined → evicted` | TTL de quarentena estourou sem restauração | `low_roi`, `conflict` | sim (janela) |
| `quarantined → invalidated` | shift confirmado durante quarentena | `shift` | não auto |
| `invalidated → evicted` | retention de invalidados estourou | `shift` | sim (curta janela) |
| `evicted → active` | restore explícito (user ou loop frio) | — | — |
| `evicted → purged` | retention window estourou OU `user_purge`/`security` | `expired`, `user_purge`, `security` | **não** |
| `* → purged` (skip evicted) | só com `user_purge` ou `security` + confirmação dupla | `user_purge`, `security` | **não** |

Transições não listadas são proibidas. Owner que tentar move ilegal recebe erro fatal + audit event.

### 4.2 Por que não auto-revert de `invalidated`

`invalidated` significa invariante externo quebrado (binário removido, deps mudaram, schema alterou). Re-promoção exige re-medição completa, não auto-flip por "ah, voltou". Loop frio pode re-promover, mas começa de `proposed` novo, com nova evidence — não reaproveita posterior antigo. Reaproveitar contaminaria com sample de stack diferente.

---

## 5. Triggers

Triggers são **detectados por owner** mas **avaliados contra critério comum** declarado aqui.

### 5.1 Triggers de transição pra `quarantined` ou `evicted`

| Trigger | Substratos aplicáveis | Owner detector | Motivo emitido |
|---|---|---|---|
| `failure_burst` | policy | failure pipeline (`FAILURE_MODES.md`) | `conflict` |
| `user_override_repeated` | policy, memory | permission engine + memory events | `conflict` |
| `verify_failed` | memory | memory verify-before-act (`MEMORY §6.1`) | `shift` |
| `distribution_shift` | policy, memory (project scope) | shift detector (`FEEDBACK §7.3`) | `shift` |
| `source_removed` | memory (reference type), policy (L1 aliases) | startup probe | `shift` |
| `trust_revoked` | memory (`untrusted`) | trust prompt re-fires | `security` |
| `quota_pressure` | candidate, slot item | context budgeter | `quota` |
| `roi_below_threshold` | memory, policy | usage telemetry (loop frio) | `low_roi` |
| `expired_at` | memory (com `expires`) | session start hook | `expired` |
| `conflict_detected` | memory (memory × memory), policy (policy × policy) | conflict resolver (`§6.3`) | `conflict` |
| `user_purge` | qualquer | `/memory delete`, `/agent policy purge` | `user_purge` |
| `security_purge` | qualquer | hook `Eviction` com `reason=security` | `security` |

### 5.2 Trigger ≠ ação

Trigger marca **candidato a transição**. Ação só dispara após passar:

- **Gate de evidência** (§6.1): N de amostras + intervalo de credibilidade
- **Gate de proteção** (§6.2): item pinned, item `user_explicit` recente, item dentro de cooldown
- **Gate de cascading** (§6.4): se transição afeta outros objetos, propor cascade explícito

### 5.3 Compaction como trigger especial

Compaction in-session ([`CONTEXT_TUNING §12`](./CONTEXT_TUNING.md)) é o consumidor de maior frequência do contrato de eviction — roda por turn, sobre slot items efêmeros. Não paga gate bayesiano (§6.1); usa proxy "fora do top-K na re-rank" como sinal de `quota`. Lifecycle paralelo, divergências de gate, interação com pins/quarentena e fallbacks em **§9**.

---

## 6. Decisão

### 6.1 Gate de evidência (per motivo)

| Motivo | Evidence schema mínimo | N mínimo | Threshold |
|---|---|---|---|
| `irrelevant` | usage = 0 em N consultas onde escopo bate | N=20 | usage_rate = 0 |
| `conflict` | (memory_id_a, memory_id_b, contradição_detectada) OU N falhas consecutivas | N=3 (failures) | — |
| `shift` | fingerprint(scope_atual) ≠ fingerprint(scope_admissão) acima de threshold | N=1 (shift é binário) | shift_score > 0.3 |
| `low_roi` | tokens_consumidos / vezes_load_bearing, calibrado por substrato | N=30 (consultas) | ROI < threshold (§6.5) |
| `quota` | budget(slot) < cost(item) com item fora do top-K | N=1 | — |
| `expired` | `expires < now()` no frontmatter | N=1 | — |
| `user_purge` | comando explícito do usuário | N=1 | — |
| `security` | hook bloqueou OU pattern match em `§7.3` MEMORY | N=1 | — |

Evidência abaixo do gate ⇒ trigger registrado em `eviction_events` como `trigger_fired_no_action`, sem transição. Útil pra detectar trigger thrashing.

### 6.2 Gate de proteção (proteções absolutas)

Mesmo com evidence suficiente, transição é bloqueada se:

- **Pinned** ([`CONTEXT_TUNING §12.4`](./CONTEXT_TUNING.md)): item explicitamente pinned não é evictado de contexto. Eviction de objeto persistente subjacente é permitido, mas slot item pinned no turn fica.
- **`source: user_explicit` em cooldown:** memory criada manualmente nas últimas 72h não é evictada por `low_roi` ou `irrelevant`. Usuário acabou de criar — sample insuficiente.
- **Active policy em escopo `session`:** policy de session só morre com a session. Eviction durante sessão é bug.
- **Em quarentena há menos de TTL mínimo:** quarentena tem TTL mínimo (default 7d). Eviction antes disso é bypass do gate de re-promoção.

Bloqueio ⇒ `eviction_events` com `outcome: blocked_by_protection` + motivo.

### 6.3 Conflict resolution

Quando dois objetos do mesmo substrato afirmam coisas incompatíveis:

```
detector emite (item_a, item_b, conflict_kind)
  │
  ▼
resolver ranqueia por:
  1. tier de proveniência     (user_explicit > inferred > imported)
  2. recência                  (mais novo vence em empate)
  3. escopo                    (mais específico vence)
  4. evidence base size        (mais N vence em empate)
  │
  ▼
perdedor → quarantined  (motivo: conflict, evidence: (winner_id, conflict_kind))
vencedor mantém estado
  │
  ▼
ambos visíveis em `/memory list --conflicts` ou `/agent policy conflicts`
```

Conflict **nunca** auto-purga. Quarentena permite usuário inspecionar e decidir. Auto-purge de conflito é receita pra perder o item correto silenciosamente quando heurística erra.

### 6.4 Cascading

Eviction de objeto X pode ter dependentes:

- Policy P referencia memory M (raro, mas possível em recipes L3)
- Memory M cita symbol S do CODE_INDEX
- Slot item I derivou de memory M

Regra: EVICTION **não cascateia automaticamente**. Quando detecta dependentes:

1. Registra `eviction_events` com `dependents: [...]`
2. Loop frio (próxima execução) re-avalia cada dependente com evidence nova
3. Dependente pode passar gate sozinho ou não — decisão própria

Cascade auto contaminaria: evictar policy errada poderia derrubar memories válidas em sequência.

### 6.5 Memory ROI

ROI = tokens_consumidos_residentes / vezes_load_bearing_em_decisão.

- **`tokens_consumidos_residentes`:** custo agregado em prompts onde a memória esteve presente (carregada eager via índice, ou lazy via `memory_read`).
- **`vezes_load_bearing`:** consultas onde decisão registrada em `audit_timeline` cita o objeto via `policy_id` ou `memory_ref`.

Threshold default: ROI > 100 tokens/load-bearing-event ⇒ candidato a `low_roi` quando N ≥ 30. Threshold ajustável per substrato.

ROI **não é gate único**. É um dos motivos possíveis; precisa passar gate de evidência (§6.1) e gate de proteção (§6.2).

---

## 7. Reversibilidade

### 7.1 Janela de retenção

| Substrato | Estado terminal | Retention window default | Restore via |
|---|---|---|---|
| Memory entry | `evicted` | 30d | `/memory restore <name>` |
| Memory entry | `invalidated` | 7d | re-admit via flow normal |
| Policy | `evicted` | 14d | `/agent policy restore <id>` |
| Policy | `invalidated` | 0d (não recupera; re-promove de novo) | re-accumulation |
| Candidate | n/a (efêmero) | — | re-query |
| Slot item | n/a (efêmero) | — | retrieval expansion request |

Window estourada ⇒ transição automática pra `purged`. Conteúdo deletado; metadata fica.

### 7.2 Tombstone

Tombstone = registro em `eviction_events` + (opcional) blob de conteúdo cifrado no storage do owner.

- Memory: arquivo movido pra `~/.config/agent/memory/.tombstones/<name>.<ts>.md` (ou equivalente per-project).
- Policy: row mantida com `state = evicted`, `purge_at` setado.
- Restore: copia conteúdo de volta, novo `id`, parent aponta pro tombstone. Audit trail preserva linhagem.

### 7.3 Restore

```
/memory restore <name>            # tenta restore do tombstone mais recente
/agent policy restore <policy_id> # similar
```

Restore re-cria o objeto em estado `proposed`, não `active`. Re-precisa passar admission gate do owner. Razão: condição que causou eviction pode ainda valer; restore não é bypass.

### 7.4 Purge irreversível

Caminhos pra `purged` sem retention window:

- `user_purge` explícito com `--force`: confirmação dupla, registrada com hash de identidade do usuário (se disponível).
- `security` trigger: hook externo determinou que conteúdo precisa sumir agora (chave vazada, PII detectado tarde). Bypass do window com flag `--security-purge`. Hook obrigatório registra reason em `eviction_events`.
- Compliance flow (`SECURITY_GUIDELINE.md`): GDPR/DSR style; usa este mecanismo como hook, valida prova de irrecuperabilidade.

Purge irreversível **não apaga `eviction_events`**. Só apaga conteúdo. Metadata (id, ts, motivo, hash do conteúdo deletado) fica — pra prova de despejo, compliance, forensics.

---

## 8. Decay (uso restrito, não confundir com eviction)

Decay e eviction são **mecanismos diferentes** que muitas vezes se confundem. Spec separa explicitamente.

| | Decay | Eviction |
|---|---|---|
| Aplica em | weight de ranking, score temporal | estado do objeto |
| Função de | tempo (half-life) | evidência (motivo + gate) |
| Reversível? | sim (recência sobe quando reusa) | sim (dentro de window) |
| Persiste? | não muta storage | muta estado |
| Auditável? | observável em score breakdown | obrigatório em `eviction_events` |
| Onde vive | `RETRIEVAL §4.3` | aqui |

### 8.1 Onde decay é legítimo

Herdado de [`RETRIEVAL §4.3`](./RETRIEVAL.md) e [`FEEDBACK_ADAPTATION §7.2`](./FEEDBACK_ADAPTATION.md):

- **Session view (half-life 1h):** goal antigo da sessão atual perdeu relevância. Não evicta nó; rebaixa score.
- **Memory view (half-life 30d):** precedent envelhecido recebe peso menor no fusion. Não evicta entry.
- **Tier 4-5 outcomes (humano implícito, long horizon):** recência importa pra peso bayesiano; sample mais novo vale mais. Não evicta policy.

### 8.2 Onde decay é proibido

- **Workspace view:** FS é estado, não evento. Symbol não fica "menos verdadeiro" com tempo.
- **L1-L2 policies em scope persistente:** "use ripgrep" não decai. Invalida quando binário some.
- **Memory `user_explicit`:** preferência declarada explicitamente não decai. Sai por `user_purge` ou conflito explícito.
- **Pinned items:** ([`CONTEXT_TUNING §12.4`](./CONTEXT_TUNING.md)) pin desabilita decay no slot.

### 8.3 Por que separar é load-bearing

Confundir é o erro do "ensaio biomimético": tratar decay como solução de governance leva a:

- Memória ruim que esmaece passivamente em vez de ser quarentenada (cobre menos rápido um erro real)
- Memória boa em código estável que perde peso por inatividade (ranking degrada silenciosamente)
- Half-life cego como única defesa contra reinforced hallucinations (a defesa real é tier de evidência + quarantine, não passar do tempo)

Quem confunde decay com eviction acaba com sistema "que esquece" sem nunca **decidir** o que esquecer.

---

## 9. Compaction (eviction efêmera de turn-context)

Compaction é o caso especial onde **eviction roda em loop curto, sobre objetos efêmeros, dentro do mesmo turn**. Owner: [`CONTEXT_TUNING §12`](./CONTEXT_TUNING.md). Este doc declara o **contrato**: estados aplicáveis, gates herdados, e onde compaction **diverge** da eviction persistente.

### 9.1 Contraste com eviction persistente

| | Eviction persistente | Compaction |
|---|---|---|
| Substrato | memory, policy | slot item, candidate |
| Cadência | per-trigger (raro) | per-turn (frequente) |
| Persistência do registro | obrigatória em `eviction_events` | trail efêmero em [`RETRIEVAL §6.4`](./RETRIEVAL.md) |
| Retention window | sim (§7.1) | não (decisão expira com o turn) |
| Restore | via tombstone | via re-retrieval no próximo turn |
| Gates aplicáveis | §6.1 evidência + §6.2 proteção | só §6.2 proteção (evidência = "fora do top-K") |
| Cascading | considerado (§6.4) | n/a (efêmero) |
| Hooks bloqueáveis | sim (§10.3) | não (latência incompatível) |

Razão da divergência: compaction roda em hot path do turn (latência ms). Não pode pagar gate bayesiano, escrita em SQLite, hook externo síncrono. Garantia que sobrevive: **usa a mesma função de relevância** que admitiu o item no slot — coerência sem custo.

### 9.2 Estados aplicáveis ao slot item

Subset de §3, mapeado pra ciclo de vida do turn:

- `active` — item está no prompt do turn corrente
- `evicted` — item foi removido do turn corrente; trail registra (`RETRIEVAL §6.4`)
- `purged` — fim do turn ⇒ slot inteiro vira garbage; `purged` é trivial

Slot item nunca passa por `proposed`/`shadow`/`quarantined`/`invalidated`. Lifecycle reduzido reflete o escopo. Quarentena não se aplica porque efêmero não sobrevive ao turn — não há janela pra re-avaliação.

### 9.3 Triggers de compaction

Distintos dos triggers persistentes em §5.1:

| Trigger | Detector | Motivo emitido |
|---|---|---|
| `token_pressure` | budget watcher: prompt > N% do limite | `quota` |
| `turn_threshold` | turns_since_last_compact > config | `quota` |
| `manual` | `/compact` ou orchestrator step | `quota` |
| `summary_request` | modelo pediu "summarize older context" | `quota` |
| `goal_shift` | goal stack mudou; contexto do goal antigo perde peso | `irrelevant` |

Trigger marca **candidatura a re-rank**. Decisão de cortar é a re-rank em si (§9.4).

### 9.4 Fluxo canônico

```
compaction trigger fires
  │
  ▼
re-query retrieval com budget reduzido (B' < B)
  │
  ▼
candidate set re-ranked
  │
  ▼
para cada item do contexto atual:
  ├─ item pinned? ─► mantém (skip gate; §9.5)
  ├─ source em quarantined/untrusted? ─► penalty (§9.7)
  ├─ item em top-K do novo ranking? ─► mantém
  └─ caso contrário ─► evict (motivo: quota, trail em RETRIEVAL §6.4)
```

### 9.5 Pinned items como proteção absoluta

Pin desabilita compaction sobre o item, mesmo fora do top-K na re-rank. Herdado de [`CONTEXT_TUNING §12.4`](./CONTEXT_TUNING.md).

Pin é gate **absoluto**: não há override por pressão de token. Estouro de budget com tudo pinned ⇒ erro de configuração, não silent drop. Operador resolve depinning explícito ou aumentando budget. Drop silencioso de pinned item seria violação de invariante — pin existe pra significar "isso fica".

### 9.6 Re-entrada de items compactados

Item evictado por compaction no turn N pode voltar no turn N+k. Não é "restore" (§7.3) — é **re-retrieval**:

| | Restore (persistente) | Re-retrieval (compaction) |
|---|---|---|
| Origem | tombstone explícito | re-query no pipeline normal |
| Custo | barato (cópia local) | normal (RETRIEVAL completo) |
| Latência | imediata | um turn de delay |
| Versão garantida | snapshot do tombstone | atual (workspace pode ter mudado) |
| Trigger | comando explícito | mudança de relevância natural |

Slot item não tem tombstone; re-retrieval é o único caminho. Modelo pode pedir expansão explícita ("expandir `UserRepo` que foi cortado"; ver [`RETRIEVAL §6.4`](./RETRIEVAL.md)) — vira hint pro próximo turn, **não restore in-place no turn corrente**.

### 9.7 Compaction × quarentena (cross-lifecycle)

Caso sutil: policy `quarantined` (ou memory `untrusted`) pode aparecer em slot por inércia — entrou antes da transição de estado. Regra:

- **Penalty no re-rank, não evict imediato.** `score *= 0.3` para items derivados de source em `quarantined`. Mesma penalty pra memory `untrusted` ([`MEMORY §7.2`](./MEMORY.md)).
- **Se ainda passar top-K, fica.** Quarentena ≠ proibido em contexto; só "menos confiável".
- **Sem override automático de pin.** Item pinned cujo source virou quarantined permanece (operador pinou; operador despinha). Apenas exibe flag visual no trace.

Razão pra penalty (não evict forçado): às vezes o item quarantined é exatamente o mais relevante — debug de por que foi pra quarentena, decisão sobre re-promoção. Rebaixar sem bloquear preserva esse caso.

### 9.8 Coerência com retrieval

Garantia load-bearing: compaction usa **a mesma função de relevância** que selecionou o slot inicial, só com budget menor.

```
slot inicial:   rank(candidates, budget=B)        → slot_t
compaction:     rank(slot_t ∪ delta, budget=B')   → slot_t+1     (B' < B)
```

Implica:

- O que entrou e o que fica seguem mesmo critério (não há "regra de admissão" vs "regra de despejo" divergentes).
- Mudanças de slot são auditáveis com as mesmas métricas de ranking ([`RETRIEVAL §5.3`](./RETRIEVAL.md) score breakdown).
- Bug em ranking polui um lado, não dois.

Detalhamento em [`RETRIEVAL §15.5`](./RETRIEVAL.md).

### 9.9 Fallback determinístico

Retrieval indisponível ⇒ compaction degrada pra heurística antiga ([`ORCHESTRATION §4.6`](./ORCHESTRATION.md)):

```
drop_oldest(tool_results) until prompt < budget
```

Warning no trace, métrica `compaction.fallback_used` incrementada. Sessão funciona pior; não trava. Paritário com degradação geral declarada em [`RETRIEVAL §15.7`](./RETRIEVAL.md). Pinned items continuam protegidos no fallback (drop_oldest pula pins).

### 9.10 O que compaction **não** faz

- ❌ **Não evicta objeto persistente.** Cortar item do slot **não** muda estado de memory entry ou policy. Persistente só muda via §3-§5. Pressão de contexto não é motivo válido pra deletar memória subjacente.
- ❌ **Não escreve em `eviction_events`.** Volume incompatível com persistência. Exceção: `motivo ∈ {conflict, security}` ou flag `--trace-eviction` ativa.
- ❌ **Não dispara cascade.** Slot item efêmero não tem dependentes registrados em §6.4.
- ❌ **Não respeita cooldown de `user_explicit`.** Cooldown (§6.2) protege memory entry, não slot item derivado dela. Memory `user_explicit` recente que cair fora do top-K perde o slot; memory subjacente fica intocada.
- ❌ **Não pula re-rank "pra economizar latência".** Compaction sem re-rank vira `drop_oldest`, que é o fallback degradado (§9.9). Sem re-rank, perde coerência com retrieval (§9.8) — e aí compaction sumarizando contexto vira loteria.
- ❌ **Não compacta dentro de um único turn em loop.** Um trigger ⇒ uma re-rank ⇒ um slot novo. Re-compactar o mesmo turn é sinal de budget mal calibrado, não de compaction agressiva.

---

## 10. Audit

### 10.1 Tabela `eviction_events`

Canônica neste doc; referenciada por [`AUDIT.md`](./AUDIT.md).

```sql
CREATE TABLE eviction_events (
  id              INTEGER PRIMARY KEY,
  parent_id       INTEGER,                  -- evento que originou (ex: trigger detection)
  substrate       TEXT NOT NULL,            -- 'memory' | 'policy' | 'candidate' | 'slot_item'
  object_id       TEXT NOT NULL,            -- name do memory, policy_id, node_id, slot ref
  object_scope    TEXT NOT NULL,            -- escopo (session, repo, user, language, global)
  from_state      TEXT NOT NULL,            -- §3 estados
  to_state        TEXT NOT NULL,            -- §3 estados
  trigger         TEXT NOT NULL,            -- §5.1 trigger kind
  motivo          TEXT NOT NULL,            -- §0.4 motivo enum
  evidence_json   TEXT NOT NULL,            -- payload tipado por motivo (§6.1)
  outcome         TEXT NOT NULL,            -- 'applied' | 'blocked_by_protection' | 'trigger_fired_no_action'
  blocked_by      TEXT,                     -- se outcome='blocked_by_protection'
  actor           TEXT NOT NULL,            -- 'loop_cold' | 'compaction' | 'user' | 'hook' | 'startup_probe'
  session_id      TEXT,
  dependents_json TEXT,                     -- ids dos objetos potencialmente afetados (cascade detection)
  recorded_at     INTEGER NOT NULL,
  purge_at        INTEGER                   -- se to_state='evicted', quando vira purged
);

CREATE INDEX idx_evict_obj   ON eviction_events(substrate, object_id);
CREATE INDEX idx_evict_state ON eviction_events(to_state, recorded_at);
CREATE INDEX idx_evict_purge ON eviction_events(purge_at) WHERE purge_at IS NOT NULL;
```

### 10.2 Queries canônicas

```sql
-- "Por que essa memory entry foi evictada?"
SELECT * FROM eviction_events
WHERE substrate='memory' AND object_id=? AND to_state='evicted'
ORDER BY recorded_at DESC LIMIT 1;

-- "O que foi evictado mas ainda pode ser restaurado?"
SELECT substrate, object_id, motivo, recorded_at, purge_at
FROM eviction_events
WHERE to_state='evicted' AND purge_at > strftime('%s','now') * 1000;

-- "Trigger thrashing: trigger disparou N vezes sem ação?"
SELECT substrate, object_id, trigger, COUNT(*) c
FROM eviction_events
WHERE outcome='trigger_fired_no_action' AND recorded_at > ?
GROUP BY substrate, object_id, trigger
HAVING c >= 5;
```

### 10.3 Hook `Eviction`

Bloqueável. Empresa pode forçar audit externo antes de transição terminal.

```toml
[[hooks]]
event = "Eviction"
matcher = { motivo = "security" }
command = "~/.config/agent/hooks/eviction_audit.sh"
# bloqueia em exit 1
```

Matcher suporta `substrate`, `motivo`, `from_state`, `to_state`, `actor`. Bloqueio gera `outcome: blocked_by_hook` em `eviction_events`.

### 10.4 Sem GC silencioso (auditoria de invariante)

Health check: contar objetos que mudaram de estado **sem** evento correspondente em `eviction_events`. Esperado: zero. Não-zero ⇒ bug em owner (não chamou contrato de transição).

```sql
-- conceptual: depende de owners exporem snapshot de estado
SELECT * FROM owner_state_snapshot s
WHERE NOT EXISTS (
  SELECT 1 FROM eviction_events e
  WHERE e.substrate = s.substrate AND e.object_id = s.object_id
    AND e.to_state = s.current_state
    AND e.recorded_at = s.last_transition_at
);
```

---

## 11. Métricas

| Métrica | O que mede | Trigger | Threshold |
|---|---|---|---|
| `eviction.rate_by_motivo` | distribuição de motivos em janela | dashboard | revisão se `low_roi` > 40% (priors miscalibrated) |
| `eviction.restore_rate` | % de evicted que voltaram pra active dentro de window | review semanal | > 20% ⇒ gate de eviction muito agressivo |
| `eviction.purge_irreversible_count` | quantos foram purged sem passar por window | alerta | > 0 sem `security`/`user_purge` ⇒ bypass detectado |
| `quarantine.dwell_time` | tempo médio em `quarantined` antes de transição | review | > 30d ⇒ loop frio não está re-avaliando |
| `quarantine.escape_rate` | % de quarantined que voltam pra active | review | < 5% ⇒ quarentena vira purgatório |
| `cascade.dependents_orphaned` | dependents sem re-avaliação após eviction de parent | alerta | > 0 ⇒ loop frio backlog |
| `roi.bottom_decile_residency` | tokens consumidos por items no decil inferior de ROI | review | alto ⇒ candidatos a eviction se passar gate |
| `protection.cooldown_blocks` | quantas evictions bloqueadas por proteção (§6.2) | health check | spike ⇒ trigger ruim ou cooldown mal calibrado |
| `hook.eviction_blocks` | quantas evictions bloqueadas por hook externo | review | persistente ⇒ revisar matcher |
| `decay.misfire` | objetos persistentes onde decay foi aplicado (proibido §8.2) | alerta | > 0 ⇒ bug fatal |

`restore_rate > 20%` ou `quarantine.escape_rate < 5%` indicam calibração ruim — sistema descarta cedo demais ou prende em quarentena pra sempre. Ambos pioram inteligência efetiva.

---

## 12. Anti-patterns

- ❌ **Decay como única defesa contra reinforced hallucinations.** Defesa real é tier de evidência + quarantine. §8.3.
- ❌ **Eviction sem motivo declarado.** "Sumiu por inatividade" não conta. §0.4.
- ❌ **Cascading automático.** Evictar parent não auto-purga dependentes. §6.4.
- ❌ **Auto-purge de conflito.** Conflict vai pra `quarantined`, decisão humana decide. §6.3.
- ❌ **Half-life sobre objeto persistente.** Decay é peso de ranking, não estado. §8.
- ❌ **Tombstone sem retention window.** Sem janela, restore vira incerto e storage cresce sem GC. §7.1.
- ❌ **Modelo decide eviction.** Meta-cognição harness-side. §0.10.
- ❌ **Confundir compaction (in-session) com eviction (persistente).** Compaction usa este doc como contrato, não substitui. §9.
- ❌ **Pular `quarantined` em failure burst.** Direto pra `evicted` ⇒ perde sinal de re-promoção. §0.6, §4.1.
- ❌ **Restore reusando posterior antigo.** Restore re-entra como `proposed`, re-precisa passar gate. §7.3.
- ❌ **`purged` sem registro em `eviction_events`.** Conteúdo some, metadata fica. Sem metadata, audit é impossível. §7.4.
- ❌ **"Limpar memória pra liberar contexto" sem critério.** Espremer storage não é motivo válido. Sem `low_roi` calibrado ou `quota` real, não evicta. §6.1.

---

## 13. Cross-refs

- [`MEMORY.md §6`](./MEMORY.md) — owner de lifecycle de memory entries; consome contrato de transição daqui.
- [`FEEDBACK_ADAPTATION.md §7`](./FEEDBACK_ADAPTATION.md) — owner de policy invalidation/quarantine; emite triggers consumidos aqui.
- [`RETRIEVAL.md §6`](./RETRIEVAL.md) — compression e trail de skipped (efêmero); consulta EVICTION pra compaction (`§5.3`).
- [`CONTEXT_TUNING.md §12`](./CONTEXT_TUNING.md) — compaction delega critério aqui; pinned items são gate de proteção (`§6.2`).
- [`AUDIT.md`](./AUDIT.md) — `eviction_events` é canônico aqui, indexado por AUDIT.
- [`FAILURE_MODES.md`](./FAILURE_MODES.md) — `failure_burst` trigger origina daqui, consumido por `§5.1`.
- [`PERMISSION_ENGINE.md`](./PERMISSION_ENGINE.md) — `user_override_repeated` trigger; permission events alimentam evidence.
- [`SECURITY_GUIDELINE.md`](./SECURITY_GUIDELINE.md) — purge compliance flow usa `security` motivo + hook `Eviction`.
- [`CONTRACTS.md §2.6.8`](./CONTRACTS.md) — meta-cognição harness-only herdado.
- [`RECAP.md`](./RECAP.md) — restore_rate, quarantine dwell time aparecem em recap semanal.

---

## 14. Insight final (não-instrução, contexto)

Construir memória é fácil; construir **esquecimento tipado, com motivo e reversível** é o problema real. A diferença entre agent que acumula entropia e agent que mantém densidade informacional ao longo de meses é exatamente esta: cada saída de cada objeto é decisão consciente, com evidence registrada, reversível enquanto a window dura. Decay esmaece; eviction decide. Sistemas maduros precisam dos dois — mas precisam saber qual está atuando, em que objeto, por quê.
