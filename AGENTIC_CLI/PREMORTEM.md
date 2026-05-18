# PREMORTEM

Técnica de Gary Klein aplicada ao ciclo do agente: **antes** de executar um plano, assumir que ele já fracassou e enumerar as causas. Inverte o framing do "o que pode dar errado?" — pedir hipóteses sobre uma falha *já ocorrida* ativa raciocínio causal específico em vez de preocupação genérica.

Premortem é a forma operacional canônica de "meça duas vezes, corte uma" (premissa raiz do `AGENTIC_CLI`, §0) quando o corte é caro, irreversível ou observável por terceiros. Não substitui execução incremental; é o último filtro antes de atos que não dá pra desfazer.

---

## 0. Por que funciona (e por que LLM precisa de scaffolding pra fazer bem)

Em humanos, prospective hindsight ("imagine que deu errado, por quê?") gera ~30% mais hipóteses concretas que brainstorm padrão (Mitchell et al., 1989). O framing força commitment com um cenário e raciocínio backward a partir dele.

Em LLM o efeito existe mas degrada rápido sem constraint:

- Sem schema, o modelo lista riscos genéricos (`"pode haver um bug"`, `"edge case não tratado"`).
- Sem orçamento, ele empilha riscos plausíveis-mas-irrelevantes até o token limit.
- Sem ranking por blast radius, o output é uma lista chapada onde "log message errada" pesa igual a "drop de tabela em prod".
- Sem vínculo a mitigação, vira teatro — gera lista, ignora lista.

O resto desta nota é como evitar essas quatro patologias.

---

## 1. Princípios

1. **Gate, não ritual.** Premortem roda em ações que satisfazem critério explícito (§2), não em todo turno. Premortem universal vira ruído e é silenciosamente ignorado.
2. **Especificidade obrigatória.** Cada risco nomeia arquivo, comando, flag, linha, ou recurso concreto. "Race condition no handler" é insuficiente; "race entre `flush()` em `writer.go:142` e shutdown signal" é aceitável.
3. **Backward, não forward.** Prompt fixa o cenário ("o plano falhou, o usuário está furioso, são 6 meses depois") e pede *causas*. Pedir "o que pode dar errado" gera lista forward enviesada por otimismo.
4. **Ranking por blast radius × reversibilidade**, não por probabilidade. Em segurança/ações irreversíveis, low-prob+high-impact > high-prob+low-impact.
5. **Cada risco → mitigação concreta OU aceitação explícita.** Sem `accepted_without_mitigation`, agente não pode prosseguir; força o usuário a ver o que está deixando passar.
6. **Diversidade independente.** Multi-agente: cada premortem-agent gera lista *antes* de ver as outras. Concatenar agentes que se influenciam degenera em groupthink — mesmo padrão do estudo original.
7. **Output schema versionado.** Premortem sem schema vira free-form e some no audit trail.

---

## 2. Quando rodar (gates concretos)

Não rodar em: leitura, grep, plan-mode, edit reversível em working tree limpo, dry-run.

Rodar em uma das condições:

| Condição                                                                 | Exemplo                                                       |
|--------------------------------------------------------------------------|---------------------------------------------------------------|
| Ação irreversível local                                                  | `rm -rf`, `git reset --hard`, drop table, truncate            |
| Ação observável por terceiros                                            | `git push`, deploy, abrir PR, enviar mensagem, criar issue    |
| Ação em recurso compartilhado                                            | migration prod, mudança de CI, rotação de secret              |
| Plano com >N steps de side-effect (default N=5)                          | refactor multi-file, série de comandos encadeados             |
| Confidence baixa declarada pelo planner                                  | `assumptions` não-vazio em campos críticos                    |
| Reuso de plano que falhou anteriormente (lookup em audit)                | retry após `subagent.*.failed`                                |

Gates implementados em `STATE_MACHINE.md` §12 (`clarify_mode: on_high_blast`) e em `PERMISSION_ENGINE.md` (cada permission de tier ≥ 2 dispara premortem antes do prompt de confirmação).

---

## 3. Estrutura do prompt

Template canônico (constraints negativas explícitas, conforme `PLAYBOOKS.md` §0):

```
São 6 meses no futuro. O plano abaixo foi executado e falhou de forma
embaraçosa. O usuário está pedindo um post-mortem.

PLANO:
<plano completo, sem omissão>

CONTEXTO RELEVANTE:
<diff, arquivos tocados, comandos previstos, recursos afetados>

TAREFA:
Liste de 5 a 10 causas plausíveis da falha. Para cada causa:
- referência concreta (arquivo:linha, comando exato, nome de recurso)
- mecanismo (a cadeia causal, não só o sintoma)
- blast radius (local | shared | external | irreversible)
- mitigação proposta OU "accepted_without_mitigation" com justificativa

NÃO:
- liste riscos sem referência concreta
- inclua causas que já estão mitigadas no plano (releia antes)
- pondere por probabilidade — pondere por blast × reversibilidade
- proponha mitigação que exija reescrever o plano inteiro (sinalize "plan_invalid" e pare)
```

Notas:

- Cenário no passado, não no futuro. "Vai falhar" gera otimismo; "falhou" gera causalidade.
- "Embaraçosa" + "post-mortem" ancora em consequência social, melhora a busca por causas não-óbvias.
- Limite explícito (5–10) impede o modelo de empilhar riscos genéricos até o token limit.
- "Releia antes" reduz double-counting de risco já mitigado.

---

## 4. Output schema

```yaml
premortem_version: 1
plan_ref: <hash do plano avaliado>
findings:
  - id: <kebab-case>
    cause: string                     # 1 linha; o que aconteceu
    reference: string                 # file:line | comando | recurso
    mechanism: string                 # cadeia causal
    blast: enum [local, shared, external, irreversible]
    mitigation:
      kind: enum [patch_plan, add_check, abort, accepted]
      detail: string                  # se accepted: justificativa
verdict: enum [proceed, proceed_with_patch, plan_invalid]
assumptions: [string]                 # premissas do premortem
not_checked: [string]                 # o que ficou fora do escopo
```

`verdict: plan_invalid` é terminal — agente principal recebe o premortem e tem que re-planejar, não pode "executar com cuidado".

Campos `assumptions` e `not_checked` são obrigatórios (`PLAYBOOKS.md` §1.2) — premortem também faz suposições; declarar quais é parte de não ser teatro.

---

## 5. Integração no ciclo do agente

Três pontos de entrada, todos no `STATE_MACHINE.md`:

**5.1. Planner → premortem → executor.** Após `plan.ready`, se algum gate de §2 disparou, emite `premortem.requested`. Bloqueia executor até `premortem.ready`. Verdict `plan_invalid` força volta ao planner com findings injetados no context.

**5.2. Pré-tool em ação irreversível.** Permission engine (`PERMISSION_ENGINE.md` §tier-2) intercepta tool call de tier alto, dispara premortem inline (subagente curto, ~1 turno), apresenta findings + ação ao usuário no prompt de confirmação. Usuário vê *o que pode quebrar*, não só "permitir? s/n".

**5.3. Red-team paralelo em orchestration.** `ORCHESTRATION.md` §multi-agent: para tarefas de alto blast, um agente "red" roda premortem em paralelo ao planner. Sai junto com o plano, não depois. Custo: 1 turno extra de inferência. Ganho: crítica adversarial que não foi ancorada pelo plano.

Em todos os três casos, findings vão pro audit trail (`AUDIT.md`) com `premortem_version` + `plan_ref`. Permite eval longitudinal: quais classes de findings se materializaram em falha real (true positive), quais foram ignoradas e nada aconteceu (false positive).

---

## 6. Anti-patterns

- **Premortem decorativo.** Roda, gera lista, executor ignora. Mitigação: verdict é input bloqueante do executor, não anexo opcional.
- **Lista genérica.** "Bugs podem ocorrer", "talvez não trate edge cases". Mitigação: schema exige `reference` concreta; sem ela, item é descartado pelo validator.
- **Premortem universal.** Rodar em todo turno ensina o agente a tratar como overhead. Mitigação: gate em §2.
- **Concordância com o planner.** Premortem feito pelo mesmo agente que escreveu o plano tende a defender o plano. Mitigação: subagente separado, contexto restrito (só plano + diff, sem histórico do planner), temperature ligeiramente maior pra diversidade.
- **Riscos só forward.** Modelo entrega "pode acontecer X" em vez de "aconteceu X, porque Y". Mitigação: prompt fixa cenário no passado e exige `mechanism` (cadeia causal), não só sintoma.
- **Mitigação que vira refactor.** Premortem retorna 7 findings, cada um pedindo redesign. Mitigação: regra explícita no prompt — se a única correção é reescrever o plano, retorne `plan_invalid` em vez de empilhar mitigações.
- **Premortem como bloqueio infinito.** Cada execução gera novo premortem, cada premortem gera novos findings, agente nunca executa. Mitigação: max 1 premortem por `plan_ref`; se verdict é `plan_invalid` e replan, novo `plan_ref` → novo premortem permitido. Loop detection no orchestrator.

---

## 7. Eval

Como saber se o premortem está funcionando, não só rodando:

- **True-positive rate.** Findings cuja causa se materializou em falha real (cruzar `premortem.findings[].id` com `audit.failures` no mesmo `plan_ref`).
- **False-positive rate.** Findings com `mitigation.kind = patch_plan` aplicados sem que a causa apareceria — sinaliza overcaution.
- **Coverage.** Falhas reais sem finding correspondente no premortem (miss). Pior métrica das três; é o sinal de que o gate ou o prompt está fraco.
- **Latência adicionada.** P50/P95 de tempo entre `plan.ready` e `executor.start` com vs sem premortem. Se P95 > N segundos sem ganho de coverage, gate está disparando demais.

Eval acoplada ao playbook (`PLAYBOOKS.md` §0.6), versionada por `premortem_version`. Sem eval, premortem apodrece — vira ritual cuja remoção ninguém defende nem ataca.
