# Idempotência

Uma operação é **idempotente** quando aplicá-la múltiplas vezes produz o mesmo efeito que aplicá-la uma única vez. `f(x) == f(f(x))`. Em sistemas distribuídos, idempotência não é uma escolha estética — é uma necessidade: redes reentregam mensagens, clientes fazem retry, jobs são reprocessados. Sem idempotência, reentregas viram duplicatas, cobranças duplas, e-mails repetidos, contadores inflados.

## Por Que Importa

Em qualquer sistema não-trivial, uma requisição pode ter três resultados:

1. **Sucesso conhecido**: servidor respondeu `200`.
2. **Falha conhecida**: servidor respondeu `4xx`/`5xx`.
3. **Resultado desconhecido**: timeout, conexão caiu, cliente não soube se a operação aconteceu.

O caso (3) é o difícil. O cliente *precisa* tentar de novo — não tentar pode perder a operação, tentar sem idempotência pode duplicar. Idempotência torna o retry seguro.

## HTTP e Idempotência

A RFC 9110 define semântica dos métodos:

| Método | Seguro? | Idempotente? |
|---|---|---|
| GET | sim | sim |
| HEAD | sim | sim |
| PUT | não | **sim** |
| DELETE | não | **sim** |
| POST | não | não (por padrão) |
| PATCH | não | não (depende) |

`PUT /users/42` com o mesmo payload duas vezes → mesmo estado final. `POST /users` duas vezes → dois usuários. Por isso, operações que criam recursos via `POST` precisam de mecanismos explícitos de idempotência.

## Técnicas

### 1. Idempotency Key

Cliente gera um identificador único (UUID v4, ULID) para a requisição e envia em header (`Idempotency-Key`, convenção popularizada pela Stripe). Servidor:

1. Verifica se a chave já foi processada.
2. Se sim, retorna a resposta anterior armazenada.
3. Se não, processa, persiste resposta + chave atomicamente, retorna.

**Detalhes críticos**:
- A chave precisa ter **TTL** (ex.: 24h). Guardar para sempre explode storage.
- A verificação e a execução precisam ser **atômicas** (transação, lock, ou `INSERT ... ON CONFLICT`). Senão duas requisições concorrentes com a mesma chave podem ambas passar pela checagem.
- A resposta cacheada deve incluir status e corpo. Diferentes requisições com a mesma chave mas *payloads diferentes* devem ser rejeitadas (sinal de bug no cliente).

### 2. Chave Natural

Em vez de um ID separado, use um identificador de negócio. Ex.: `POST /payments` com `external_ref = "invoice_123"`. Se já existe pagamento com esse `external_ref`, retorna o existente. Simples quando o domínio já tem uma chave adequada.

### 3. Upsert

`INSERT ... ON CONFLICT DO UPDATE` (PostgreSQL) ou equivalente. Torna a escrita inerentemente idempotente. Ideal quando o estado final é determinado pelo payload.

### 4. Máquina de Estados

Cada transição só é válida a partir de um estado específico. Se a transição já aconteceu, repeti-la é no-op (não erro). Ex.: um pedido em estado `paid` não pode ser pago de novo, mas o endpoint retorna `200` com o estado atual em vez de `400`.

### 5. Token de Uso Único

Antes de operar, cliente obtém um token. O servidor consome o token na operação. Retry com o mesmo token é rejeitado — cliente precisa de novo token.

## At-Least-Once vs Exactly-Once

Garantias de entrega em sistemas de mensagens:

- **At-most-once**: mensagem pode ser perdida, nunca duplicada. Fácil, mas inaceitável para dados importantes.
- **At-least-once**: mensagem pode ser duplicada, nunca perdida. É o padrão realista.
- **Exactly-once**: aparece em marketing, raramente na prática. Requer coordenação forte (transações distribuídas) ou é na verdade *effectively-once* via idempotência no consumidor.

**A regra prática**: *exactly-once delivery* não existe; *exactly-once processing* se consegue combinando at-least-once + consumidor idempotente.

## Armadilhas

1. **Efeitos colaterais não-idempotentes dentro de handlers idempotentes**: o endpoint pode ter idempotency key, mas se ele dispara um e-mail sem checar, reprocessamento envia e-mail duplicado. Toda chamada externa com efeito precisa estar sob o mesmo guarda-chuva.
2. **Race condition na checagem**: `SELECT` + `INSERT` sem lock/constraint pode passar duas requisições simultâneas. Use `INSERT ... ON CONFLICT` ou `UNIQUE` constraint que falha a segunda.
3. **Payload divergente com mesma chave**: cliente com bug reusa chave com dados diferentes. Servidor deve detectar (hash do payload) e retornar erro claro.
4. **Chave insuficientemente única**: usar `user_id + timestamp-em-segundos` parece ok até duas requisições chegarem no mesmo segundo.
5. **Retrys sem backoff**: um cliente que tenta 10x em 100ms quando o servidor está caído é um ataque DoS contra o próprio sistema. Sempre combinar retry com backoff exponencial + jitter (ver `RELIABILITY_PATTERNS.md`).
6. **Esquecer DELETE**: deletar algo deletado deve ser no-op (retornar `204` ou `200`), não `404`. Cliente que deletou e sofreu timeout vai reenviar.

## Idempotência em Filas

Consumidor de fila deve ser idempotente por construção — quase todas as filas garantem at-least-once, não exactly-once.

Padrões:
- **Dedup table**: persistir `message_id` processado; ignorar reprocesso. Combinar com TTL.
- **Transaction outbox**: escrever na fila como parte da transação do banco (via tabela outbox + worker). Garante que a mensagem sai se e somente se o dado foi persistido. Ver `EVENT_DRIVEN.md`.
- **Idempotent sink**: escrever no destino de forma que reprocesso seja seguro (upsert, set, contadores via delta + dedup).

## Resumo Operacional

Se a sua operação pode ser retentada (spoiler: pode), a pergunta não é *se* você precisa de idempotência, mas *onde* você vai colocá-la: no cliente, no servidor, na fila, no banco. Escolher explicitamente > descobrir sob incidente.
