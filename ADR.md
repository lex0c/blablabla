# Architecture Decision Records (ADR)

Decisões arquiteturais moldam sistemas por anos. Código mostra **o que** foi escolhido, mas raramente **por que** — e sem o "por que", revisões futuras são cegas: não se sabe se a decisão foi deliberada, quais alternativas foram descartadas, sob quais premissas. ADRs são o registro dessas decisões.

## Definição

Um **Architecture Decision Record** é um documento curto, imutável por convenção, que captura uma decisão arquitetural, seu contexto e suas consequências. Popularizado por Michael Nygard em 2011.

## Quando Escrever

Um ADR vale quando a decisão:

1. **É significativa** — afeta estrutura, performance, segurança, escalabilidade ou custos.
2. **Tem alternativas plausíveis** — "usar HTTPS" não precisa de ADR; "gRPC vs REST para comunicação interna" precisa.
3. **É de difícil reversão** — trocar depois exige esforço real.
4. **Estabelece convenção duradoura** — vai guiar decisões futuras.

Não escreva ADR para decisões triviais, de escopo local (uma função), ou táticas temporárias. O sinal de que está se escrevendo demais: ADRs virando ruído, ninguém lê.

## Formato Clássico (Nygard)

Arquivo curto (1-2 páginas), em Markdown, com seções:

```markdown
# ADR-0007: Adotar PostgreSQL como banco primário

Data: 2026-04-21
Status: Accepted

## Contexto

Precisamos escolher um banco relacional para o novo serviço de pedidos.
Os requisitos incluem transações ACID, suporte a JSONB, extensões para
geolocalização e experiência da equipe.

## Decisão

Adotaremos PostgreSQL 16.

## Alternativas consideradas

- MySQL: familiar para parte da equipe, mas suporte a JSON e tipos
  compostos é mais limitado.
- CockroachDB: escalabilidade horizontal atraente, mas overhead
  operacional não justifica neste estágio.
- MongoDB: descartado — cargas relacionais são majoritárias.

## Consequências

Positivas:
- Stack operacional simplificada (já rodamos Postgres em outros serviços).
- Ferramental maduro (pgBouncer, Debezium, PgBench).

Negativas:
- Escala vertical tem limite; caso tráfego cresça acima do projetado,
  precisaremos particionar manualmente (ver SHARDING.md).
- Extensões específicas (PostGIS) criam dependência que dificulta
  migração futura.
```

## Status

Ciclo de vida típico:

- **Proposed**: em discussão.
- **Accepted**: decisão tomada.
- **Deprecated**: não aplicamos a novos casos, mas ainda vive em código legado.
- **Superseded by ADR-NNNN**: substituído por outro.

Regra de ouro: **ADRs aceitos não se editam**. Mudou de ideia? Escreva um novo ADR que supersede o anterior, referenciando-o. O histórico é o valor.

## Numeração e Organização

- Numeração sequencial zero-padded: `0001`, `0002`, ..., `0042`.
- Arquivo por ADR: `docs/adr/0042-nome-curto.md`.
- Índice (`README.md` ou `index.md`) com listagem e status.
- Ferramenta útil: [adr-tools](https://github.com/npryce/adr-tools) gera arquivos a partir de template.

## Variantes

- **MADR (Markdown Any Decision Records)**: template mais detalhado, bom para times maiores.
- **Y-Statements** (Rebecca Wirfs-Brock): formato ultra-curto: "In the context of X, facing concern Y, we decided Z to achieve W, accepting downside V." Excelente para decisões menores.
- **RFCs internos**: documento maior antes do ADR, para decisões que merecem discussão extensa. ADR captura o resultado final; RFC captura o debate.

## ADR vs Documentação

| | ADR | Documentação técnica |
|---|---|---|
| Foco | **por que** foi escolhido | **como** funciona |
| Mutabilidade | imutável (supersede com novo ADR) | mutável conforme evolui |
| Escopo | uma decisão | um sistema/módulo |
| Audiência | futuros engenheiros da equipe | usuários / operadores |

Ambos são necessários, e nenhum substitui o outro.

## Benefícios

1. **Onboarding**: novo engenheiro entende as premissas do sistema sem precisar reconstruir por arqueologia.
2. **Revisão de decisão**: ao descobrir que uma premissa mudou (ex.: carga dobrou), sabe-se exatamente qual decisão revisitar.
3. **Fim do "we have always done it this way"**: a razão está escrita. Se não faz mais sentido, o ADR é supersedido.
4. **Código de revisão**: um PR grande com mudança arquitetural frequentemente merece um ADR anexo. Dá contexto ao reviewer.
5. **Registro da evolução da equipe**: ler ADRs em ordem cronológica conta a história técnica do produto.

## Armadilhas

1. **ADR defensivo**: escrito só para "cobrir a bunda" em reunião. Sem alternativas consideradas honestamente, vira teatro.
2. **ADR burocrático**: exigir ADR para tudo paralisa. Mantenha a barra alta — só decisões com as características listadas.
3. **ADR sem consequências**: a seção mais útil costuma ser a de consequências (o que ganhamos, o que perdemos). Omiti-la mata o valor.
4. **Falta de linking**: ADRs isolados perdem tração. Linkar para ADRs relacionados, PRs, issues, documentação.
5. **ADR como compromisso eterno**: decisões podem e devem ser revistas. Supersede quando necessário. Imutabilidade é do **registro**, não da **decisão**.

## Integração com Outras Práticas

- Combina com `CONCEPTUAL_INTEGRITY.md`: ADRs acumulados revelam (ou traem) a coerência do design ao longo do tempo.
- Complementa `SOFTWARE_ARCHITECTURE.md`: a arquitetura descrita é *o resultado*; ADRs são *o caminho*.
- Apoia `REFACTORING.md`: saber a premissa original ajuda a avaliar se a refatoração ainda respeita as intenções ou se está contradizendo-as.
