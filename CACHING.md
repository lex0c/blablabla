# Cache

Cache é uma camada intermediária que guarda resultados caros para reutilização. Troca **memória/storage extra** por **menos latência e menos carga na origem**. Como toda otimização, tem custos próprios: complexidade, staleness, inconsistência, e — notoriamente — bugs que só aparecem sob carga.

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

## Camadas Típicas

Do mais próximo ao mais distante do código:

1. **CPU cache** (L1/L2/L3): gerido por hardware, invisível ao app. Afeta *memory access patterns*.
2. **Page cache** do SO: leituras de disco já em memória. `/proc/meminfo` mostra o uso.
3. **In-process cache**: estrutura na própria heap do processo (Caffeine, Guava Cache, `lru_cache` do Python). Zero latência de rede, mas não compartilhado entre instâncias.
4. **Cache distribuído**: Redis, Memcached. Compartilhado entre instâncias; latência de rede (<1ms em rede local).
5. **CDN**: Cloudflare, Fastly, CloudFront. Cacheia respostas HTTP próximas ao usuário. Crítico para assets estáticos e, cada vez mais, respostas dinâmicas.
6. **HTTP cache do cliente**: browser, app mobile. Controlado por headers (`Cache-Control`, `ETag`, `Last-Modified`).

## Padrões de Escrita

### Cache-Aside (Lazy Loading)

Padrão mais comum. A aplicação gerencia o cache explicitamente:

```
read:
  val = cache.get(k)
  if miss:
    val = db.get(k)
    cache.set(k, val, ttl)
  return val

write:
  db.set(k, val)
  cache.invalidate(k)  # ou cache.set(k, val)
```

**Prós**: simples, resiliente (cache down não quebra reads). **Contras**: primeira leitura sempre paga miss; janela entre `db.set` e `cache.invalidate` pode servir dado velho.

### Write-Through

A aplicação escreve no cache, que escreve no banco antes de retornar. Cache sempre coerente com banco. **Contras**: latência de escrita maior; cache precisa saber como escrever no banco.

### Write-Behind (Write-Back)

Aplicação escreve no cache, que responde imediatamente. Cache persiste no banco de forma assíncrona (batch). **Prós**: latência baixíssima. **Contras**: risco de perda de dados se o cache cair antes do flush; complexidade alta.

### Read-Through

Similar a cache-aside, mas o cache é responsável por carregar do banco no miss (via callback/provider). Aplicação só fala com cache.

### Refresh-Ahead

Antes do TTL expirar, o cache pré-carrega em background os itens "quentes". Evita o miss que causaria latência ao usuário. Útil quando se sabe que certos itens vão ser requisitados.

## Invalidação

Existem basicamente três estratégias, e todas têm custos:

1. **TTL (Time To Live)**: item expira após N segundos. Simples. Trade-off: TTL curto = menos staleness, mais misses; TTL longo = melhor hit rate, mais staleness.
2. **Invalidação explícita**: no write, remover/atualizar a chave. Exige saber todas as chaves afetadas por uma mutação — não-trivial com queries complexas, agregações, listagens.
3. **Event-driven**: o sistema publica eventos de mudança; caches consomem e invalidam. Escala bem, mas adiciona infra.

Estratégias híbridas são comuns: TTL curto + invalidação explícita, ou *stale-while-revalidate* (serve stale enquanto refresca em background).

## Problemas Clássicos

### Cache Stampede (Thundering Herd)

Muitos clientes batem simultaneamente em uma chave que acabou de expirar. Todos dão miss, todos vão ao banco ao mesmo tempo → banco cai.

Mitigações:
- **Lock / single-flight**: apenas uma requisição busca na origem; as demais esperam e consomem o resultado. `golang.org/x/sync/singleflight` implementa isso.
- **TTL com jitter**: em vez de TTL fixo de 60s, use `60s ± 10s`. Espalha expirações.
- **Recomputação preemptiva**: refrescar item com probabilidade crescente conforme TTL se aproxima do fim (XFetch algorithm).
- **Stale-while-revalidate**: serve a resposta stale enquanto busca a nova em background.

### Cache Penetration

Requisições para chaves que **não existem** na origem (ex.: ID inválido). Sempre dão miss, sempre vão ao banco. Mitigações:

- Cachear valores negativos (`null`) com TTL curto.
- **Bloom filter** antes do banco: rejeita chaves que definitivamente não existem.

### Cache Avalanche

Muitas chaves expiram ao mesmo tempo (ex.: deploy invalidou tudo; TTL fixo foi setado em massa). Banco recebe tsunami. Mitigações: jitter no TTL, warm-up gradual, *circuit breaker* entre app e banco (ver `RELIABILITY_PATTERNS.md`).

### Hot Key

Uma chave recebe 10x mais tráfego que as outras. Satura a shard/nó do cache que a guarda. Mitigações:

- **In-process cache** na frente (L1) para chaves hot: reduz tráfego no cache distribuído.
- **Sharding da chave**: `hot_key:0`..`hot_key:N`, clientes leem uma shard aleatória. Só funciona se o valor é o mesmo em todas.

### Stale Data Visível

Cache TTL + replicação assíncrona = usuário pode ler dado de 1 minuto atrás depois de uma escrita. Trade-off consciente. Se inaceitável, há dois caminhos: *read-your-writes* (cachear por usuário com invalidação imediata no próprio cliente) ou ler direto do primário.

## HTTP Caching

Headers chave:

- `Cache-Control: max-age=3600, public`: cacheável por qualquer cache intermediário por 1h.
- `Cache-Control: private, max-age=60`: apenas browser, 1min.
- `Cache-Control: no-store`: nunca cachear (dados sensíveis).
- `ETag: "abc123"`: identificador do conteúdo. Cliente manda `If-None-Match` na próxima; servidor responde `304 Not Modified` se inalterado.
- `Last-Modified` / `If-Modified-Since`: versão temporal do acima.
- `Vary: Accept-Encoding, Authorization`: cache deve considerar esses headers como parte da chave.

`stale-while-revalidate` e `stale-if-error` (RFC 5861) permitem servir cache vencido brevemente enquanto refresca ou quando a origem falha — excelentes para resiliência em CDN.

## Políticas de Evicção

Quando o cache enche, o que jogar fora:

- **LRU (Least Recently Used)**: o que não foi acessado há mais tempo. Padrão mais comum.
- **LFU (Least Frequently Used)**: o menos acessado. Melhor para padrões estáveis; ruim para *bursts*.
- **FIFO**: o mais antigo a entrar. Simples; raramente melhor que LRU.
- **ARC, 2Q, W-TinyLFU**: algoritmos adaptativos usados por caches modernos (Caffeine usa W-TinyLFU).
- **Random**: surpreendentemente competitivo em alguns benchmarks; elimina worst cases adversariais.

## Princípios Práticos

1. **Meça antes de cachear**: cache cedo demais adiciona complexidade sem ganho mensurável. Sem métrica de *hit rate* e *latência*, você está adivinhando.
2. **Cache é otimização, não fonte da verdade**: a aplicação tem que funcionar com cache vazio (cold start, flush, queda). Se não funciona sem cache, o "cache" virou um banco sem durabilidade.
3. **TTL é a invalidação mais barata**: explícita é mais correta mas mais complexa. Comece com TTL; adicione invalidação só onde o custo da staleness justifica.
4. **Chave de cache como contrato**: documente o que forma a chave. Inclua versão do schema (`user:v2:{id}`) para invalidar tudo num deploy quebrado.
5. **Observe o cache como qualquer outra dependência**: hit rate, latência p99, erros, evictions. Ver `OBSERVABILITY.md`.
