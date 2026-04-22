# Padrões de Resiliência

Todo sistema não-trivial depende de recursos que podem falhar: bancos, APIs externas, filas, redes. A engenharia de confiabilidade não elimina falhas — ela garante que falhas **localizadas** não virem falhas **globais**. Os padrões abaixo são o vocabulário prático para isso.

## Timeout

O padrão mais elementar e mais negligenciado. Toda chamada a um recurso externo precisa de timeout. Sem timeout, uma dependência lenta vira uma cascata: threads bloqueadas → pool exaurido → o serviço inteiro cai porque um único parceiro ficou 30 segundos sem responder.

**Regras de bolso**:

1. Timeouts precisam existir em **todas as camadas**: conexão, handshake TLS, leitura, escrita, query no banco, HTTP client. O default de muitas libs é "infinito" — verifique.
2. Timeouts devem **diminuir** à medida que se aprofundam. Se seu handler tem budget de 500ms, a chamada ao banco não pode ter timeout de 2s.
3. Timeout não te diz se a operação foi executada. Combine com idempotência (ver `IDEMPOTENCY.md`).
4. Timeouts agressivos demais geram falhas espúrias sob latência natural (GC, warmup). Meça p99 real antes de escolher.

## Retry

Tentar de novo após falha. Parece trivial; é a origem de muitas tempestades em produção.

**Regras obrigatórias**:

1. **Só retente erros transitórios**: timeouts, `503`, `502`, erros de rede. Nunca retente `400`, `401`, `403`, `422` — bug no retry vai se repetir.
2. **Exponential backoff**: espere cada vez mais entre tentativas. Ex.: 100ms, 200ms, 400ms, 800ms. Evita saturar o serviço que está se recuperando.
3. **Jitter**: adicione aleatoriedade ao backoff. Sem jitter, N clientes que falharam ao mesmo tempo vão retentar ao mesmo tempo — sincronizados. "Equal jitter" (`rand(0, backoff)`) ou "full jitter" (`rand(0, cap)`) são implementações comuns (ver post do AWS Architecture Blog).
4. **Cap no número de tentativas**: 3-5 na maioria dos casos. Retry infinito é garantia de cascata.
5. **Cuidado com retry em camadas**: se cliente retenta 3x, gateway retenta 3x, serviço retenta 3x → uma única requisição pode virar 27 no backend. Concentre retry em **uma** camada.

## Circuit Breaker

Quando uma dependência está claramente caída, continuar tentando só piora as coisas. O circuit breaker monitora falhas e, após um limiar, **interrompe** chamadas por um período — "o circuito abre". Três estados:

1. **Closed**: chamadas passam normalmente. Contador de falhas ativo.
2. **Open**: falhas acima do limiar → circuito abre. Chamadas falham imediatamente (fail-fast) sem sobrecarregar a dependência. Dura X segundos.
3. **Half-open**: após o timeout, permite algumas chamadas de teste. Se passam → volta para closed. Se falham → volta para open.

**Benefícios**:
- Falha rápida em vez de travar threads.
- Dá tempo para a dependência se recuperar.
- Evita o anti-padrão "retry tempest".

**Cuidados**:
- Limiar muito baixo → aberturas espúrias em picos normais.
- Limiar muito alto → nunca abre, perde o propósito.
- Estado deve ser **por dependência**, não global.

Implementações: Resilience4j (Java), Polly (.NET), `gobreaker` (Go), `opossum` (Node).

## Bulkhead

Inspirado em compartimentos estanques de navios: um vazamento em uma parte não afunda o todo. Em software: isolar recursos (threads, conexões, memória) por dependência ou classe de tráfego, de forma que a saturação de uma não exaure as outras.

**Exemplos**:
- Pools de thread separados por serviço externo.
- Filas separadas por prioridade.
- Conexões de banco reservadas para health checks.
- Limite de conexões simultâneas por tenant (multi-tenant).

Combina bem com circuit breaker: bulkhead limita o *blast radius*; circuit breaker corta a hemorragia.

## Hedged Requests

Quando latência de cauda importa, envie a mesma requisição para duas (ou mais) réplicas e use a primeira resposta. Cancela as demais. Popularizado pelo paper "The Tail at Scale" (Dean & Barroso, Google).

**Trade-off**: aumenta carga (~2x no pior caso). Vale quando p99 é dominado por "stragglers" aleatórios. Variante econômica: dispare a segunda requisição **após** um timeout curto (ex.: p95 esperado).

## Load Shedding

Quando o sistema está saturado, rejeitar requisições antes de processá-las preserva a capacidade para as que "vão passar". Sem shedding, sob overload, o sistema degrada para um estado onde *nada* funciona (todas respondem lento, todas dão timeout).

**Técnicas**:
- **Queue-depth shedding**: se a fila interna está acima de N, rejeite (retorne `503`).
- **Concurrency limits**: número máximo de requisições em voo. Netflix `concurrency-limits` implementa adaptação dinâmica via TCP Vegas.
- **Priority shedding**: rejeitar tráfego low-priority primeiro (health checks, batch) antes do high-priority (pagamentos).
- **Adaptive throttling**: cliente detecta aumento de 5xx e auto-reduz taxa ("client-side rate limiting").

## Rate Limiting

Controle explícito de quantas requisições por unidade de tempo são aceitas, por cliente/IP/chave. Diferente de load shedding (reativo à saturação), rate limit é **preventivo**.

**Algoritmos**:
- **Token bucket**: tokens adicionados a taxa fixa; cada request consome um. Permite bursts até o tamanho do bucket.
- **Leaky bucket**: requisições processadas a taxa fixa; excesso é descartado ou enfileirado.
- **Fixed window**: N requests por janela. Sofre de *bursts* na virada da janela.
- **Sliding window log**: guarda timestamps; preciso mas caro.
- **Sliding window counter**: aproximação (fixed window + interpolação). Bom custo/precisão.

## Fail-Fast vs Fail-Safe

Duas filosofias opostas:

- **Fail-fast**: na primeira anomalia, pare e sinalize. Bom para desenvolvimento, para dependências críticas.
- **Fail-safe**: continue operando com funcionalidade reduzida. Bom para recursos não-essenciais (ex.: recomendações falham → mostra catálogo sem recomendação).

A escolha é por dependência, não global. Pagamento é fail-fast; telemetria é fail-safe.

## Graceful Degradation

Modo degradado explícito: se uma dependência não-crítica falha, remova a feature em vez de quebrar a requisição inteira. Exige desenhar o produto com essa possibilidade.

Exemplos:
- Sem Redis: vá direto ao Postgres (mais lento, mas funciona).
- Sem serviço de recomendação: mostre lista populares.
- Sem serviço de pricing: use preço cacheado (com badge indicando).

Graceful degradation exige que o produto tolere ausência de funcionalidade — **não é só técnico**. Negociar com produto antes de implementar.

## Backpressure

Mecanismo pelo qual um consumidor lento **freia** o produtor em vez de acumular silenciosamente. Buffers sem backpressure → OOM sob carga.

**Exemplos**:
- TCP: janela de congestionamento.
- Reactive Streams / Project Reactor: `request(n)` explícito.
- Filas bounded: produtor bloqueia quando cheio (ou rejeita).
- Go channels sem buffer.

Buffers não resolvem lentidão, só adiam a crise. Backpressure obriga a enfrentar o desbalanço.

## Padrões Combinados

Em produção, esses padrões se combinam. Uma chamada típica a um serviço externo:

1. **Timeout** agressivo.
2. **Retry** com backoff + jitter em erro transitório.
3. **Circuit breaker** por trás, cortando quando ruim.
4. **Bulkhead**: pool de conexões isolado.
5. **Idempotency key** no header, porque o retry pode duplicar.
6. **Fallback** (cache stale, valor default, feature desligada) quando tudo falha.
7. **Observabilidade**: log + métricas + trace em cada camada.

## Princípios

1. **Falhas não são exceção; são parte do contrato**. Código que ignora falha é código incompleto.
2. **Tempestades retentativas matam**. Mais retry não é mais resiliência; é amplificação.
3. **Degradação progressiva > falha total**. Desenhe para o modo degradado desde o começo.
4. **Teste as falhas**. Ver `CHAOS_ENGINEERING.md`. Padrão nunca exercido é padrão que não funciona.
