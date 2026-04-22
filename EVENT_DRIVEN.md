# Arquiteturas Orientadas a Eventos

Em vez de serviços se chamarem diretamente (request/response), eles publicam **eventos** — fatos imutáveis sobre algo que aconteceu — e outros serviços reagem. O acoplamento cai: o produtor não precisa saber quem consome, e novos consumidores podem ser adicionados sem alterar o produtor.

## Vocabulário

- **Evento**: registro de algo que *ocorreu*, no passado, imutável. Ex.: `OrderPlaced`, `PaymentCaptured`.
- **Comando**: solicitação para que algo *aconteça*, pode ser rejeitado. Ex.: `PlaceOrder`.
- **Tópico / Stream / Canal**: fluxo nomeado onde eventos são publicados.
- **Produtor (Publisher)** e **Consumidor (Subscriber / Consumer)**.
- **Broker**: infraestrutura que recebe, armazena e entrega (Kafka, RabbitMQ, NATS, SQS, Pub/Sub).

## Fila vs Stream

Distinção fundamental:

| | Fila (ex: SQS, RabbitMQ) | Stream (ex: Kafka, Kinesis) |
|---|---|---|
| Modelo | mensagem é "trabalho" a fazer | evento é "fato" registrado |
| Consumo | mensagem some após ack | log persistente; múltiplos consumidores, cada um com seu offset |
| Ordem | frequentemente por fila | por partição |
| Reprocessamento | difícil (dead-letter + replay manual) | natural (rewind de offset) |
| Uso típico | tarefas assíncronas, jobs | event sourcing, pipelines, integração |

A diferença importa: mudar de fila para stream muda a semântica do sistema, não só a tecnologia.

## Garantias de Entrega

- **At-most-once**: rápido, sem dedup. Aceita perda.
- **At-least-once**: padrão da maioria. Consumer precisa ser idempotente (ver `IDEMPOTENCY.md`).
- **Exactly-once**: existe em poucos contextos controlados (Kafka com transactions + consumer idempotente). No geral, o que se consegue é *effectively-once* = at-least-once + consumer idempotente.

## Ordering

Ordenação global em brokers distribuídos é cara ou impossível. Em Kafka, ordem é garantida **por partição**. Se você precisa de ordem por `order_id`, particione pela chave `order_id`. Se tudo precisa ser global → uma partição → throughput de um único consumer.

## Padrões

### Pub/Sub

Produtor publica em um tópico; qualquer número de consumidores recebe. Mais simples, mais usado.

### Event Sourcing

Em vez de guardar o **estado atual** no banco, guarde a sequência de **eventos** que levaram a ele. O estado é derivado reexecutando os eventos.

**Prós**:
- Auditoria completa "de graça": você tem toda a história.
- Time-travel: é possível reconstruir o estado em qualquer ponto.
- Múltiplas projeções a partir do mesmo log.

**Contras**:
- Complexidade alta. Schema evolution é espinhoso (eventos antigos não podem ser reescritos).
- Não é a escolha certa só por ser "elegante". Aplicar onde audit trail é requisito de negócio.
- Snapshots periódicos são necessários para evitar replay longo.

### CQRS (Command Query Responsibility Segregation)

Separa o modelo de **escrita** (commands) do modelo de **leitura** (queries). Frequentemente combinado com event sourcing: commands geram eventos; queries lêem projeções construídas a partir dos eventos.

**Quando vale**: quando os requisitos de leitura e escrita são muito diferentes — ex.: milhares de eventos/seg escritos + leituras complexas e agregadas. Para CRUD normal, CQRS é overengineering.

### Outbox Pattern

Problema: como garantir que "salvar no banco" + "publicar evento" aconteçam juntos? Sem coordenação, tem dois modos de falha: salvou mas não publicou (dado perdido para consumidores) ou publicou mas não salvou (evento fantasma).

Solução: **dual write via tabela outbox**.

1. Na mesma transação do banco, grava o estado + uma linha na tabela `outbox` com o evento.
2. Um processo separado (polling ou CDC) lê `outbox` e publica no broker.
3. Após publicar, marca como enviado (ou apaga).

Garante que evento sai **se e somente se** dado foi persistido. Preço: pequena latência entre commit e publish.

### Saga

Para transações distribuídas em arquitetura sem coordenador (2PC). Em vez de atomicidade, modela a transação como uma **sequência de passos**, cada um com uma **compensação**. Se o passo 3 falha, executa compensações dos passos 1 e 2 na ordem reversa.

**Variantes**:
- **Orquestração**: um serviço coordenador comanda os passos. Mais claro, mais acoplado.
- **Coreografia**: cada serviço reage a eventos dos outros. Mais desacoplado, mais difícil de rastrear.

Não é "rollback" — compensações são operações *de negócio* (reembolsar pagamento ≠ "desfazer" pagamento).

### Change Data Capture (CDC)

Ler o **log de replicação** do banco (ex.: WAL do Postgres, binlog do MySQL) e publicar mudanças como eventos. Ferramentas: Debezium, Maxwell.

**Benefícios**: zero mudança na aplicação; captura *todas* as mudanças (inclusive escritas legadas). **Contras**: schema do banco vaza como schema do evento; exige cuidado com transformação.

Uma alternativa moderna ao outbox — e frequentemente usada em combinação.

## Problemas e Armadilhas

1. **Schema evolution**: eventos são imutáveis, mas seu formato evolui. Use formatos com compatibilidade (Avro, Protobuf). Regra: novos campos opcionais, nunca remover nem renomear. Schema Registry centraliza isso.
2. **Acoplamento via payload**: mesmo sem dependência direta, se produtor muda o formato do evento e consumidor quebra, o acoplamento está lá, só escondido. Schema é contrato.
3. **Eventos como sinal vs eventos como estado**: `UserUpdated` vazio (apenas ID) obriga consumidor a buscar o estado atual. `UserUpdated` com todo o snapshot é pesado mas self-contained. Escolher conscientemente.
4. **Debugging é mais difícil**: uma requisição do usuário pode disparar 5 eventos que disparam 15 eventos. Trace distribuído (correlation ID propagado) é não-opcional.
5. **Ordem é uma miragem**: dois eventos em tópicos diferentes não têm ordem garantida. Design que depende de "A veio antes de B" só funciona se A e B compartilham partição.
6. **Idempotência no consumer não é opcional**: at-least-once + reprocessamento → o mesmo evento chega mais de uma vez.
7. **Poison messages**: uma mensagem que faz o consumer quebrar para sempre. Precisa de dead-letter queue (DLQ) com alerta.
8. **Back-pressure invisível**: consumer lento → lag cresce → eventos atrasam horas. Monitorar lag é crítico.
9. **Eventos falsos são mentiras permanentes**: um evento publicado por engano (ex.: bug) fica no log para sempre. Não se "deleta"; compensa-se publicando outro evento.

## Quando Usar (e Quando Não)

**Usar eventos quando**:
- Múltiplos consumidores precisam reagir à mesma mudança.
- Produtor e consumidor têm ciclos de vida / deploys independentes.
- Audit trail ou event sourcing é valor de negócio.
- Alto volume com processamento assíncrono (pipelines, ingest).

**Não usar quando**:
- Interação é inerentemente síncrona (usuário esperando resposta na UI).
- Um único consumidor, sem necessidade de desacoplamento.
- Equipe não tem maturidade para lidar com consistência eventual, ordenação, dedup. O custo cognitivo é alto.
