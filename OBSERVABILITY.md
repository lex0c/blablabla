# Observabilidade

Observabilidade é a capacidade de inferir o estado interno de um sistema a partir das saídas que ele expõe. Enquanto **monitoramento** responde perguntas previamente formuladas ("o CPU está acima de 80%?"), **observabilidade** permite formular *novas* perguntas sem precisar alterar o código — essencial quando bugs em produção são imprevistos, não sintomas conhecidos.

## Os Três Pilares

### 1. Logs

Registros discretos de eventos, geralmente textuais ou estruturados (JSON). Bons logs são:

- **Estruturados**: campos explícitos (`user_id`, `latency_ms`, `error_code`) em vez de strings livres. Facilita filtrar e agregar em ferramentas como Elasticsearch, Loki, Splunk.
- **Contextuais**: carregam *trace id*, *span id*, *request id*, *tenant id* — permitindo correlação com traces e métricas.
- **Com nível apropriado**: `DEBUG`/`INFO`/`WARN`/`ERROR`. Não logar em `ERROR` o que é apenas recuperável.
- **Sem PII sensível**: senhas, tokens, CPFs completos nunca devem aparecer em logs.

**Custo**: logs são os sinais mais caros em volume. Amostragem em alto volume (ex.: só 1% dos sucessos, 100% dos erros) é uma prática comum.

### 2. Métricas

Agregações numéricas ao longo do tempo. Baratas de armazenar (séries temporais comprimidas) e rápidas de consultar. Quatro tipos principais:

- **Counter**: valor monotonicamente crescente (ex.: `http_requests_total`).
- **Gauge**: valor que sobe e desce (ex.: `memory_used_bytes`, `queue_depth`).
- **Histogram**: distribuição de valores em buckets (ex.: latência p50/p95/p99).
- **Summary**: similar a histogram, mas percentis são calculados no cliente.

Ferramentas típicas: Prometheus, Datadog, Graphite, CloudWatch, OpenTelemetry Metrics.

### 3. Traces

Registro do caminho de uma requisição através de múltiplos serviços. Um *trace* é uma árvore de *spans*; cada span representa uma operação (chamada HTTP, query SQL, job de fila) com início, fim e atributos. Essencial em arquiteturas distribuídas — sem traces, uma latência p99 alta é impossível de localizar.

Padrão atual: **OpenTelemetry** (OTel), que unificou OpenTracing e OpenCensus. Backends: Jaeger, Tempo, Honeycomb, Zipkin.

## SLI, SLO e Error Budget

Popularizados pelo SRE Book do Google:

1. **SLI (Service Level Indicator)**: uma métrica que mede aspecto do serviço relevante ao usuário. Exemplos: % de requisições com status 2xx, p99 de latência, taxa de entrega de mensagens.
2. **SLO (Service Level Objective)**: um alvo numérico para um SLI em uma janela de tempo. Ex.: "99.9% das requisições /checkout respondem em menos de 300ms, medido em janela rolante de 30 dias".
3. **SLA (Service Level Agreement)**: compromisso contratual com consequências (geralmente financeiras). SLA é sobre contrato; SLO é sobre engenharia.
4. **Error Budget**: `1 - SLO`. Com SLO de 99.9%, o orçamento é 0.1% de falhas no período. Se o budget está sendo consumido rápido → congelar releases, priorizar estabilidade. Se sobra budget → pode-se investir em velocidade/experimentos.

Error budget transforma discussões subjetivas ("está estável?") em decisões baseadas em dado.

## Métodos de Análise

### USE Method (Brendan Gregg) — para recursos

Para cada recurso (CPU, memória, disco, rede), monitore três dimensões:

- **Utilization** — tempo ocupado.
- **Saturation** — quanto trabalho está enfileirado além da capacidade.
- **Errors** — contagem de falhas.

Útil para diagnosticar gargalos em infraestrutura.

### RED Method (Tom Wilkie) — para serviços

Para cada serviço ou endpoint:

- **Rate** — requisições por segundo.
- **Errors** — taxa de erros.
- **Duration** — distribuição de latência.

Mais focado em experiência do usuário que USE.

### Golden Signals (Google SRE)

Combinação prática: *latency*, *traffic*, *errors*, *saturation*.

## Boas Práticas

1. **Instrumente nas bordas e nos pontos de decisão**: entrada/saída de serviços, queries ao banco, chamadas externas, erros tratados. Instrumentação excessiva polui; insuficiente cega.
2. **Use cardinality com cuidado**: métricas com muitas dimensões (ex.: `user_id` como label) explodem em custo. Alta cardinalidade pertence a logs/traces, não a métricas.
3. **Correlacione os pilares**: um alerta em métrica → busca nos logs do período → trace da requisição problemática. Ferramentas modernas (Grafana, Datadog) já fazem essa ponte quando os IDs estão propagados.
4. **Alerte em sintomas, não em causas**: alerte em "p99 de /checkout > 1s" (sintoma visto pelo usuário), não em "CPU > 80%" (causa possível). Alertas em causa geram ruído e perdem gravidade.
5. **Dashboards como ferramenta de narrativa**: um bom dashboard conta uma história — do topo (SLOs, tráfego) para o detalhe (latência por endpoint, erros por tipo). Dashboards desorganizados viram ruído visual.

## Armadilhas Comuns

- **Observabilidade como afterthought**: adicionar depois que o incidente aconteceu. Deve ser parte do design, não remendo.
- **Over-alerting**: muitos alertas → fadiga → alerta real ignorado. Se um alerta não exige ação humana, não é alerta — é log.
- **Logs como métrica**: contar ocorrências de uma string em log é 10-100x mais caro que ler um counter. Se você vai contar, emita métrica.
- **Métricas sem contexto**: p99 de latência global é inútil se você tem 50 endpoints com perfis diferentes. Segmente.
- **Falta de trace propagation**: serviços que não repassam headers de trace (`traceparent`, `b3`) criam buracos na árvore, e o trace distribuído vira uma colcha de retalhos.
