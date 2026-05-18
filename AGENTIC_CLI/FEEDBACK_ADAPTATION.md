# FEEDBACK_ADAPTATION

Subsistema de **aprendizado operacional** do `AGENTIC_CLI`. Cobre como o harness mede resultado real de ações, como agrega esses resultados em sinal confiável, e como muda comportamento futuro sem contaminar o modelo. Não é RLHF, não é fine-tuning, não é "memória esperta". É policy update baseado em outcome.

Não é "agent que aprende". O objeto de primeira classe é a **separação entre cognição estática (modelo) e adaptação dinâmica (harness)**. Modelo é commodity; harness é onde acumula vantagem composta. Sem isso declarado, "agent que melhora" vira marketing — na prática, cada sessão começa do zero e os mesmos erros reaparecem.

Premissa raiz herdada: *meça duas vezes, corte uma*. Aqui a medição é **operacional** — o que funcionou de fato, em quantas amostras, com que validação externa? Sem isso, adaptação é astrologia probabilística com temperatura baixa.

---

## 0. Princípios (não-negociáveis)

1. **LLM não avalia LLM.** Auto-avaliação é o mesmo loop estatístico que gerou a ação — correlação 100% com o erro original. Feedback obrigatoriamente externo: exit code, test outcome, build status, user action, métrica observável.
2. **Dois loops, cadências diferentes.** *Loop quente* registra outcome a cada ação (latência ms, custo zero, sem aprendizado). *Loop frio* agrega evidência multi-sessão e edita policy (latência horas/dias, trigger explícito). Misturar os dois é onde nasce o colapso por feedback ruim.
3. **Adaptação é harness-only, invisível ao modelo.** Modelo nunca "sabe" próprias heurísticas — senão racionaliza em cima e o sinal vira ruído. Adaptação se manifesta como tool registry rewrite, permission policy update, retrieval ranking update. Modelo vê só o efeito (`grep` virou `ripgrep` no registry), não a causa.
4. **Princípio guia herdado (`CONTRACTS.md §2.6.8`):** meta-cognição não é tool. Adaptação é meta-cognição máxima — fica integralmente fora do catálogo exposto ao modelo.
5. **Escopo declarado, sempre.** Heurística sem escopo (per-repo, per-user, per-language, global) contamina cross-context. Escopo é parte do schema, não annotation opcional.
6. **Confidence calibrada, não pontual.** N=1 não justifica `confidence: 0.91`. Posterior bayesiano com prior explícito + intervalo de credibilidade. Sem N suficiente, política fica como `proposed`, não `active`.
7. **Invalidação > decay.** Decay é resposta preguiçosa. Heurística boa em código estável não fica menos verdade com tempo; heurística pra stack que mudou precisa **invalidar imediatamente**. Trigger de invalidação > função de decay.
8. **Default importa mais que adaptação.** Cold-start não tem sinal. Agent ruim no dia 1 em repo novo é ruim sempre — usuário forma opinião antes do loop frio rodar. Adaptação melhora marginalmente; defaults bons resolvem 80%.
9. **Auditabilidade total.** Toda mudança de policy é evento. Toda decisão informada por policy carrega referência ao policy_id. Sem isso, debugging vira folclore ("o agent começou a fazer X, ninguém sabe por quê").
10. **Reversibilidade.** Policy update é commit-like — tem id, parent, autor (loop frio que disparou), motivo, e diff. Rollback é primeira classe.

---

## 1. Escopo

### 1.1 O que resolve

- "Como o harness sabe se uma ação funcionou de verdade (não 'o modelo achou que funcionou')?"
- "Quando promover um sinal de outcome a mudança de comportamento?"
- "Como evitar o ciclo erro → memória errada → adaptação ruim → mais erro?"
- "Onde fica o limite entre per-repo, per-user, e global?"
- "Como reverter uma policy que degradou métrica?"

### 1.2 O que **não** resolve

- **Aprendizado de pesos do modelo.** Fora de escopo. Spec é harness-side.
- **Memória do usuário.** Owner: `MEMORY.md`. Memória armazena fatos; adaptação muda comportamento. Confundir os dois é §11 anti-patterns.
- **Retrieval ranking.** Owner: `RETRIEVAL.md`. Adaptação **alimenta** weights de ranking; não decide pipeline.
- **Tool catalog.** Owner: `CONTRACTS.md`. Adaptação não adiciona tools; só rewrita preferences/aliases.
- **Permission engine.** Owner: `PERMISSION_ENGINE.md`. Adaptação propõe regras (ex.: auto-allow `npm test`); engine valida e aplica.

---

## 2. Tipos de feedback

Tier ordenado por confiabilidade. Maior tier substitui menor; menor nunca contradiz maior.

| Tier | Tipo | Exemplos | Latência | Custo | Confiabilidade |
|---|---|---|---|---|---|
| 1 | **Determinístico** | exit code, test pass/fail, build status, lint result, type check, benchmark delta | ms–s | zero | alta |
| 2 | **Estrutural** | diff size, files touched, deps changed, API surface change | ms | zero | média |
| 3 | **Humano explícito** | aprovação/rejeição em permission prompt, `/approve`, `/reject`, edit manual do output | s–min | alto (user-time) | alta |
| 4 | **Humano implícito** | user reverteu commit, user re-rodou tool, user editou logo após output | min–h | zero | média-baixa |
| 5 | **Long horizon** | feature em produção sem incidente após N dias, métrica de PR aceito sem rework | dias | zero | baixa (atribuição difícil) |

### 2.1 Regras de tier

- Tier 1 disponível ⇒ usa tier 1. Tier 1 fala "fail" ⇒ tier 2/3 não conseguem promover pra "success".
- Tier 5 (long horizon) **nunca** vira gatilho sozinho. Só refina confidence de policy já existente em tier 1-3.
- Tier 4 (humano implícito) requer **par** de sinais pra contar: "user reverteu" sem corroboração é ruído (pode ser revert por outro motivo).
- Auto-avaliação (modelo julgando output próprio) **não é feedback**. É registro de intenção. Pode aparecer em `audit_timeline` como prosa, nunca em policy decision.

### 2.2 Por que não tier "score de qualidade do modelo"

LLM-as-judge sobre próprio output (ou output de modelo do mesmo provider) tem correlação alta com erro do modelo julgado. Vira eco-câmara estatística. Spec rejeita explicitamente — mesma razão que `todo_write` é rejected como tool (`CONTRACTS.md §2.6.8 B`).

Exceção tolerada: judge de **modelo diferente** (Opus julgando Haiku, ou vice-versa) **e** ancorado em rubric estruturado **e** com calibração documentada contra ground truth. Custo alto; usado só onde tier 1-3 não cobrem (raro).

---

## 3. Dois loops

### 3.1 Loop quente (per-action)

Cadência: a cada tool call. Latência tolerada: < 100ms. Custo: zero (pure SQLite write).

```
tool_call termina
  │
  ▼
outcome detector
  │ (lê exit code, stderr, mudança de FS, etc)
  ▼
INSERT em outcomes
  │
  ▼
audit_timeline event (outcome_recorded)
```

Loop quente **não muda comportamento**. Só registra. Erro comum: ler 1 outcome e já editar policy — degrada com ruído de amostra única.

Tabela `outcomes` (canônica em `AUDIT.md §1`):

```sql
CREATE TABLE outcomes (
  id INTEGER PRIMARY KEY,
  session_id TEXT NOT NULL,
  tool_call_id INTEGER NOT NULL,   -- FK a tool_calls.id
  action_signature TEXT NOT NULL,  -- unidade adaptável; ver §4
  tier INTEGER NOT NULL,           -- 1-5 (§2)
  result TEXT NOT NULL,            -- 'success' | 'failure' | 'partial' | 'ambiguous'
  evidence_json TEXT,              -- payload tipado por tier
  scope_kind TEXT NOT NULL,        -- 'global' | 'language' | 'repo' | 'user' | 'session'
  scope_id TEXT NOT NULL,          -- ex.: repo path hash, language id, user id
  recorded_at INTEGER NOT NULL
);

CREATE INDEX idx_outcomes_action_scope ON outcomes(action_signature, scope_kind, scope_id);
```

### 3.2 Loop frio (per-trigger)

Cadência: dispara em **trigger explícito**, não em timer.

Triggers válidos:

| Trigger | Condição | Owner |
|---|---|---|
| `accumulation` | ≥ N outcomes novos pra mesma `action_signature` desde última análise (N default 10) | scheduler |
| `incident` | `failure_event` crítico relacionado a policy ativa | failure pipeline |
| `distribution_shift_detected` | mudança em stack/deps/branch policy do repo (§7) | shift detector |
| `manual` | user roda `/agent policy review` | CLI |
| `rollback_requested` | métrica derivada degradou ≥ X% pós-policy | metrics watcher |

Pipeline:

```
trigger
  │
  ▼
load outcomes pra action_signature + scope
  │
  ▼
bayesian update (prior + evidence) ─► posterior + credibility interval
  │
  ▼
policy proposer ─► diff candidato
  │
  ▼
validation gate
  ├─ confidence > threshold?
  ├─ contradiz policy ativa?
  ├─ shift recente invalida amostra?
  └─ requer aprovação humana? (§9.3)
  │
  ▼
policy commit (id, parent, autor, motivo, diff)
  │
  ▼
audit_timeline event (policy_changed)
```

Loop frio **é** o aprendizado. Roda raramente, com evidência agregada, e é commit-like.

### 3.3 Por que separar

Misturar = adaptar baseado em N=1. É o ciclo de colapso descrito no §0.6:

```
ação ruim → outcome ruim → policy edit imediato → próxima ação enviesada pela policy → outcome confirma viés → policy se entrincheira
```

Loop frio com agregação multi-sessão quebra esse ciclo: N=1 não passa do gate de confidence.

---

## 4. Unidade adaptável (`action_signature`)

A pergunta central: **o que ganha preferência?**

Granularidade declarada em quatro níveis. Mais concreto > mais abstrato.

| Nível | Exemplo | Detecção | Risco |
|---|---|---|---|
| **L1 Binário/alias** | `grep` → `ripgrep` | trivial (string match) | baixo |
| **L2 Tool flag** | `bash` com `cd` evitado em favor de `cwd` arg | regex/lint | baixo |
| **L3 Recipe** | "ao alterar SQL, rodar migration dry-run antes" | template match + dependency graph | médio |
| **L4 Strategy** | "refactor > 5 arquivos: dividir em batches" | classificador (LLM ou heurística) | alto |

### 4.1 Regra de promoção

Adaptação só age em L1-L2 por default. L3 requer N alto e validação humana. L4 requer **opt-in explícito** — classificador erra, e classificador errando contamina o sinal de adaptação.

Texto que celebra `postgres_slow_join → composite_index` está em L4. É exatamente onde quase tudo falha na prática: detectar que o problema atual **é** desse padrão. Spec mantém L4 como experimental, gated.

### 4.2 Schema de signature

`action_signature` é string opaca pro storage, estruturada pro emitter:

```
L1:  alias:<from>:<to>                    ex: alias:grep:ripgrep
L2:  flag:<tool>:<flag>:<value>           ex: flag:bash:cwd_arg:preferred
L3:  recipe:<id>                          ex: recipe:sql_migration_dry_run
L4:  strategy:<id>:<scope>                ex: strategy:refactor_batching:js
```

Hash do prefixo permite query eficiente. Naming convention é load-bearing — sem ela, agregação cruza signatures não relacionadas e o sinal degrada.

---

## 5. Calibração de confidence

### 5.1 Por que pontual está errado

`confidence: 0.91` depois de 9/10 sucessos parece bom. É enganoso:

- 9/10 com prior uniforme: posterior Beta(10, 2), média 0.83, 95% CI [0.55, 0.97].
- 90/100: posterior Beta(91, 11), média 0.89, 95% CI [0.82, 0.94].

Mesma média de sucesso, **incertezas radicalmente diferentes**. Spec exige par `(mean, ci_low, ci_high)` em todo policy.

### 5.2 Prior

Cada `action_signature` tem prior declarado em config:

```yaml
priors:
  alias:*:*: Beta(2, 1)              # otimista: aliases simples costumam funcionar
  recipe:*: Beta(1, 1)               # uniforme: sem prior
  strategy:*:*: Beta(1, 2)            # pessimista: estratégias falham mais do que parecem
```

Prior pessimista pra L4 é deliberado — protege contra entusiasmo prematuro.

### 5.3 Gate de promoção

Policy só vira `active` se:

- `ci_low > 0.7` (margem inferior, não média) **AND**
- `n >= 10` (mínimo de amostras) **AND**
- `scope.distribution_stable == true` (§7) **AND**
- Não contradiz policy ativa em tier superior.

Senão fica em `proposed` — visível em `/agent policy list`, não aplicada automaticamente.

---

## 6. Escopo

Hierarquia ordenada por prioridade (mais específico vence):

```
session (per-conversation, ephemeral)
  ▲
repo (per-working-directory)
  ▲
user (per-user, cross-repo)
  ▲
language (per-language, cross-user)
  ▲
global (shipped defaults)
```

### 6.1 Regras

- Policy é declarada em **um único** escopo. Não há "policy global com override per-repo" — duplica e fica em escopo independente.
- Resolução em decision time: percorre de session pra global, primeira match vence.
- Outcome registrado no escopo da ação. Outcome de `bash` em repo X **não** alimenta policy global; alimenta policy per-repo X.
- Promoção entre escopos é **manual ou trigger-driven**, nunca automática. "9/10 sucessos em repo X" não vira policy user-wide sem `/agent policy promote`.

### 6.2 Por que não é per-repo automático

Tentação: detectar que mesma policy funciona em N repos do mesmo user, promover pra user-scope. Razão pra **não**:

- Atribuição difícil (repos compartilham stack? mesma língua não significa mesma convention).
- Risco de contaminar repo novo com policy aprendida em repo legacy.
- Custo de manter graph de "policies relacionadas cross-repo" supera ganho marginal.

Promoção fica como ação consciente do user, com diff visível.

---

## 7. Invalidação e distribution shift

### 7.1 Quando invalidar (não decay)

Invalidação imediata em:

| Evento | Detecção | Ação |
|---|---|---|
| Stack change | `package.json`/`pyproject.toml`/`Cargo.toml`/`go.mod` mudou substancialmente | invalidar policies `scope_kind = repo` em L3-L4 |
| Branch policy change | `.github/`, CODEOWNERS, lint config mudou | invalidar L2-L3 ligadas a essa policy |
| Tool removed | binário ausente no PATH | invalidar aliases dependentes |
| Failure burst | N falhas consecutivas em policy `active` | move pra `quarantined`; loop frio re-avalia |
| User override repetido | usuário rejeita 3× ação derivada da policy | move pra `quarantined` |

Invalidação ≠ delete. Policy invalidada vai pra status `invalidated` com motivo registrado; loop frio pode re-promover se evidência nova confirmar.

### 7.2 Decay (uso restrito)

Decay aplicável só onde:

- Escopo é `session` (heurística da sessão atual, irrelevante depois).
- Tier é 4-5 (humano implícito / long horizon — sinais ruidosos, recency importa).

Decay **não** aplicado em L1-L2 com escopo persistente — heurística "use ripgpref" não fica menos verdade com tempo. Half-life cego degrada o sistema.

### 7.3 Distribution shift detector

Roda no init de cada sessão:

```
load fingerprint do repo (deps, lockfile hashes, config hashes)
compare contra fingerprint da última sessão registrada
  ├─ delta > threshold ⇒ marca scope_id como 'unstable'
  └─ delta ≤ threshold ⇒ marca scope_id como 'stable'
```

`scope.distribution_stable` (§5.3 gate) sai daqui. Sessão em scope unstable não promove policies novas e usa policies existentes em modo `shadow` (logged, not applied) até estabilizar.

---

## 8. Cold start

### 8.1 Defaults importam mais que adaptação

Spec assume **shipped defaults bons** como base. Adaptação refina; não substitui.

- Catálogo de tools (`CONTRACTS.md`) já incorpora preferências aprendidas via curadoria humana (ex.: `ripgrep` default, não `grep`).
- Permission engine (`PERMISSION_ENGINE.md`) já tem allowlists razoáveis pra workflows comuns.
- Retrieval (`RETRIEVAL.md`) já tem ranking weights ajustados em eval suite.

Adaptação per-user/per-repo é **delta** em cima desses defaults, não substituição.

### 8.2 Bootstrap em repo novo

Primeiro uso em repo novo:

- `scope_kind = repo` está vazio — zero policies.
- Fallback hierarchy: language → user → global. Sempre tem alguma policy.
- Loop frio só dispara após N outcomes (§3.2 trigger `accumulation`). N=10 default; primeiros uses do agent são **defaults puros**.

Não tenta "adivinhar" policies pra repo novo a partir de repos similares. Adivinhar é contaminação cross-context — proibido por §0.5.

---

## 9. Onde adaptação se manifesta

Adaptação muda **estado externo ao modelo**. Nunca prompt, nunca system message.

### 9.1 Tool registry

- Aliases (L1): `grep → ripgrep` aplicado em runtime; modelo chama `grep`, harness substitui antes do dispatch.
- Tool ordering em catálogo: tools preferidas aparecem primeiro no schema enviado ao modelo (heurística de "primeira opção" no LLM).
- Default flags: tool A com flag X aplicado por default; modelo pode override explícito.

### 9.2 Permission policy

Owner: `PERMISSION_ENGINE.md`. Adaptação **propõe**:

- "user aprovou `npm test` 12× ⇒ propor auto-allow per-repo".
- Proposta vai pro permission engine que decide aplicar (pode rejeitar — engine tem regras de safety próprias).
- Auto-allow nunca cross-repo automático (§6.1).

### 9.3 Retrieval ranking

Owner: `RETRIEVAL.md §6`. Adaptação alimenta weights:

- Pattern X foi load-bearing em N sessões ⇒ sobe weight em ranking signal "historical relevance".
- Memory item Y nunca foi citado em decision ⇒ desce weight em ranking signal "recall score".

Mudança de weight é policy commit (§3.2), não edit silencioso.

### 9.4 Context strategy

Owner: `CONTEXT_TUNING.md`. Adaptação ajusta thresholds:

- Sessões > N tokens consistentemente falham em compaction ⇒ propor threshold mais agressivo.
- Summaries muito comprimidas geram re-pergunta do modelo ⇒ propor menos compressão.

### 9.5 Onde **não** se manifesta

- ❌ System prompt rewrite. Modelo nunca vê heurística aprendida.
- ❌ Few-shot examples auto-gerados. Geraria eco-câmara estatística.
- ❌ Tool description rewrite baseado em uso. Description é contrato (`CONTRACTS.md`); contratos não auto-editam.

---

## 10. Pipeline canônico

```
┌─ ação ────────────────────────────────────────────┐
│ tool_call dispatch (já reescrito por policy ativa)│
└─────────────────┬─────────────────────────────────┘
                  │
                  ▼
┌─ loop quente (per-action) ────────────────────────┐
│ outcome detector (tier 1-5)                       │
│ INSERT outcomes                                   │
│ audit_timeline event (outcome_recorded)           │
└─────────────────┬─────────────────────────────────┘
                  │
                  │   (acumulação assíncrona;
                  │    não bloqueia próxima ação)
                  ▼
┌─ triggers (§3.2) ─────────────────────────────────┐
│ accumulation | incident | shift | manual | rollback│
└─────────────────┬─────────────────────────────────┘
                  │
                  ▼
┌─ loop frio (per-trigger) ─────────────────────────┐
│ load outcomes + prior                             │
│ bayesian update ─► (mean, ci_low, ci_high, n)     │
│ policy proposer ─► diff candidato                 │
│ validation gate (§5.3, §9.3)                      │
│ policy commit (id, parent, motivo, diff)          │
│ audit_timeline event (policy_changed)             │
└─────────────────┬─────────────────────────────────┘
                  │
                  ▼
┌─ aplicação ───────────────────────────────────────┐
│ tool registry / permission / retrieval / context  │
│ atualizados pra próximas ações                    │
│ MODELO NÃO É NOTIFICADO                           │
└───────────────────────────────────────────────────┘
```

---

## 11. Anti-patterns

- ❌ **LLM-as-judge sobre output do mesmo modelo.** Eco-câmara. §2.2.
- ❌ **Adaptação como prompt injection.** "Lembrar ao modelo que ele já errou isso antes." Modelo racionaliza em cima e ruim degrada. §0.3.
- ❌ **Decay como invalidação.** Heurística em código estável não fica menos verdade. §7.2.
- ❌ **Confidence pontual sem CI.** `0.91` sem N e sem intervalo é UI que mente. §5.1.
- ❌ **Promover policy cross-scope automaticamente.** Repo-scoped policy aprendida em projeto legacy contamina projeto novo. §6.2.
- ❌ **Adaptar em L4 sem opt-in.** Strategy classification erra; classificador errado contamina sinal. §4.1.
- ❌ **N=1 promove policy.** Loop quente editando policy. §3.3 ciclo de colapso.
- ❌ **Confundir memória com adaptação.** Memória armazena fato; adaptação muda comportamento. Mesmo fato pode existir sem mudar nada — e mesma adaptação pode rodar sem memória explícita (só estatística). §1.2.
- ❌ **"Agent que aprende" como marketing.** Sem schema declarado de unidade adaptável + escopo + invalidação, "aprendizado" é folclore.

---

## 12. Métricas operacionais

| Métrica | O que mede | Trigger |
|---|---|---|
| `policy.active_count` | Quantas policies ativas por escopo | dashboard |
| `policy.proposed_pending` | Policies aguardando N suficiente | review periódico |
| `policy.quarantined` | Policies que falharam em produção | alerta |
| `policy.rollback_rate` | % de policies revertidas em N dias | review semanal |
| `outcome.tier1_coverage` | % de tool calls com outcome tier 1 disponível | health check |
| `shift.unstable_sessions` | % sessões em scope unstable | health check |
| `shadow.divergence` | Quantas policies em shadow divergiram do default | review |

`rollback_rate > 10%` ⇒ sistema de adaptação está pior que defaults; revisar priors e thresholds.

---

## 13. Cross-refs

- `CONTRACTS.md §2.6.8` — princípio "meta-cognição não é tool" (herdado).
- `AUDIT.md §1` — tabelas `outcomes`, `policies` (canônicas).
- `MEMORY.md` — memória ≠ adaptação (anti-pattern §11).
- `PERMISSION_ENGINE.md` — onde policies de auto-allow são aplicadas.
- `RETRIEVAL.md §6` — onde ranking weights vivem.
- `CONTEXT_TUNING.md` — onde thresholds de compaction vivem.
- `CONTRACTS.md` — tool registry (alvo de rewrite por aliases L1).
- `STATE_MACHINE.md` — loop quente roda no step terminator.
- `FAILURE_MODES.md` — `incident` trigger vem daqui.
- `RECAP.md` — policy changes aparecem em recap semanal/mensal.
- `LOCAL_MODELS.md` — modelos locais podem ter priors diferentes (calibração separada).
