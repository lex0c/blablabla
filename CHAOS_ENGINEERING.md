# Chaos Engineering

*Chaos engineering* é a disciplina de **injetar falhas intencionalmente** em sistemas em produção (ou perto dela) para descobrir fraquezas antes que elas descubram a si mesmas. Originada na Netflix (Chaos Monkey, 2010), formalizada por Casey Rosenthal e equipe em *Principles of Chaos Engineering*.

Não confundir com `CHAOS_CREATIVITY.md` (caos como fonte conceitual de criatividade). Aqui, caos é ferramenta de engenharia.

## Premissa

Sistemas distribuídos falham de maneiras imprevistas. Testes unitários e de integração cobrem cenários conhecidos; chaos engineering expõe os **desconhecidos**: interações entre dependências, timeouts mal configurados, retry em cascata, falsos-positivos de health check, dependências ocultas.

> "Hope is not a strategy." — Google SRE book

A metáfora frequente é a vacina: uma dose pequena de falha controlada imuniza contra falhas reais.

## Os Princípios (princlplesofchaos.org)

1. **Construir hipótese sobre estado estacionário**: defina o que "sistema saudável" significa em métricas (RED/USE, SLO). Sem baseline, nada é anômalo.
2. **Variar eventos do mundo real**: simule falhas que acontecem de fato — VM morta, latência de rede, disco cheio, região indisponível, dependência lenta.
3. **Rodar experimentos em produção**: ambiente de staging é incompleto. Tráfego, dados, interações reais só existem em prod. Rodar lá é controverso mas defendido: é onde o valor está e onde as falhas importam.
4. **Automatizar continuamente**: experimentos manuais rodam uma vez e são esquecidos. Automatize como testes de regressão.
5. **Minimizar o raio de explosão**: comece pequeno. Um nó, um tenant, uma região. Escalar conforme confiança cresce.

## O Ciclo do Experimento

1. **Estabeleça o estado estacionário**: ex.: "taxa de sucesso de checkout > 99.5%, p99 de latência < 300ms".
2. **Formule uma hipótese**: "se o cache Redis ficar indisponível, o estado estacionário se mantém (via fallback ao Postgres)".
3. **Desenhe o experimento**: qual falha, qual blast radius, por quanto tempo, sob qual tráfego, qual critério de abort (*kill switch*).
4. **Execute**: comece em horário comercial, equipe presente, monitoramento afiado. Não rode madrugada de sexta.
5. **Observe**: métricas, alertas, logs, traces. Comportamento confere com hipótese?
6. **Analise e aprenda**: se a hipótese falhou, há um bug/lacuna. Documente, corrija.
7. **Automatize**: experimento vira parte da suíte contínua.

## Tipos de Experimento

### Infraestrutura

- **Shutdown de instância**: matar VM, pod, container.
- **Partição de rede**: bloquear tráfego entre zonas/serviços.
- **Latência / jitter**: adicionar atraso em tráfego de rede.
- **Pacote perdido**: drop X% de pacotes.
- **Disco cheio / I/O lento**.
- **CPU / memória saturadas**.
- **Falha de DNS**.
- **Zona / região inteira fora**.

Ferramentas: **Chaos Mesh**, **LitmusChaos** (ambos Kubernetes), **Gremlin** (comercial), **Toxiproxy** (latência/falha em TCP), `tc netem` (Linux traffic control).

### Aplicação

- **Exceção injetada** em endpoint / função.
- **Slow response** de serviço interno.
- **Retorno inválido** (schema quebrado).
- **Deadlock** simulado.

### Dados / Estado

- **Banco em modo read-only**.
- **Cache invalidado**.
- **Clock skew** entre nós.

### Pessoas / Processo

- **Game day**: cenário coordenado onde a equipe de plantão responde a um incidente fictício. Testa não só o sistema, mas runbooks, comunicação, escalonamento.
- **Wheel of misfortune** (Google): alguém descreve um incidente simulado; on-caller investiga em voz alta.

## Relação com Outras Práticas

- **`RELIABILITY_PATTERNS.md`**: chaos engineering **verifica** que os padrões (circuit breaker, retry, timeout, bulkhead) funcionam de fato. Padrão nunca exercitado é padrão que não existe.
- **`OBSERVABILITY.md`**: sem observabilidade decente, experimento é ruído — não se sabe o que mudou. Chaos pressiona a observabilidade a melhorar.
- **`DISTRIBUTED_SYSTEMS.md`**: chaos é resposta pragmática à complexidade de sistemas distribuídos. Jepsen é chaos para consistência de bancos.
- **`PROD_PROBLEM.md`**: incidentes reais são experimentos de chaos não planejados. Chaos faz antecipado o que senão se aprenderia quebrado.

## Armadilhas

1. **Começar grande**: injetar falha massiva antes de entender o sistema → incidente real. Comece minúsculo.
2. **Sem kill switch**: experimento precisa poder ser abortado em segundos. Sem isso, "vamos tentar" vira "vamos pedir desculpas".
3. **Sem baseline**: "quebrou?" não se responde sem saber como era antes.
4. **Rodar em staging "porque é seguro"**: se staging tem 1% do tráfego, não reproduz condições de concorrência, cache, hot paths. Staging dá confiança inicial; produção dá confiança real.
5. **Chaos como espetáculo**: eventos isolados, muito PR, zero aprendizado incorporado. O valor vem do ciclo: experimento → descoberta → correção → regressão automatizada.
6. **Ignorar o custo humano**: on-caller estressado, equipe sem contexto, experimento fora do horário. Precedido de comunicação + autorização + presença.
7. **Chaos sem correção**: descobrir fraqueza e não corrigir é pior que não descobrir — agora está documentado e ignorado.

## Pré-requisitos Realistas

Não faz sentido rodar chaos em um sistema que ainda está se estabilizando. Checklist mínimo:

- [ ] Observabilidade decente (logs estruturados, métricas, traces).
- [ ] Alertas funcionando (não em excesso, não em falta).
- [ ] Runbook básico para falhas comuns.
- [ ] On-call ou equivalente.
- [ ] Rollback automatizado para deploys.
- [ ] Cultura de blameless postmortem.

Sem isso, chaos vira produção de incidentes — não engenharia de resiliência.

## Maturidade

1. **Nível 0**: só sabe dos problemas quando o usuário reclama.
2. **Nível 1**: ad-hoc — alguém derruba um serviço de teste ocasionalmente.
3. **Nível 2**: game days periódicos, documentados.
4. **Nível 3**: experimentos automatizados em staging, com hipóteses formais.
5. **Nível 4**: chaos contínuo em produção, com blast radius limitado.
6. **Nível 5**: chaos como feedback primário para melhorias arquiteturais.

Poucas organizações passam do nível 3. Está ok — nível 2 já paga os custos.

## Resumo

Chaos engineering é empirismo aplicado à resiliência. O princípio de fundo é humilde: **não sabemos como o sistema se comporta sob falha até provocarmos a falha**. Melhor descobrir em experimento controlado — com hipótese, observação, kill switch — que em incidente de madrugada.
