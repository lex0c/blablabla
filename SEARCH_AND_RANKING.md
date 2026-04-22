# Busca e Ranking

Busca é, em essência, **ordenar documentos por relevância a uma intenção**. O problema tem 50+ anos de maturidade — inverted index, BM25 — e uma revolução recente (embeddings, vector search, LLMs). Sistemas de busca modernos (Google, Amazon, Elastic, Algolia, RAG em LLMs) combinam o clássico (léxico) com o novo (semântico). Este arquivo organiza os conceitos que você precisa para construir busca, avaliar ranking, ou entender RAG.

## Fundamentos

### Duas tarefas

1. **Retrieval (recall)**: dado query, trazer candidatos relevantes da coleção grande. Importa: não perder nada.
2. **Ranking (precision)**: ordenar candidatos. Importa: topo é muito bom.

Sistemas típicos fazem em pipeline — retrieval gera 100-1000, re-ranker reordena para top 10.

### IR clássico vs IR neural

- **Clássico**: operações lexicais em tokens. BM25, TF-IDF. Rápido, explicável, baseline forte.
- **Neural/dense**: embeddings vetoriais. Captura semântica, parafraseamento, multilíngue.
- **Híbrido**: combina os dois. Hoje é o padrão em sistemas sérios.

## Léxico (Sparse)

### Tokenização

- **Whitespace + lowercase**: baseline.
- **Stemming** (Porter, Snowball): raiz. "running" → "run".
- **Lemmatization**: mais sofisticado, usa morfologia. "better" → "good".
- **Stopwords**: remover "a", "de", "the" — menos útil que parece.
- **N-grams**: "machine learning" como bigram.
- **Language analyzers**: ICU, kuromoji (japonês), nori (coreano), smartcn (chinês), cada idioma com suas quirks.
- **BPE / SentencePiece / WordPiece**: subword tokenizers modernos — bons em idioma nuevo, neologismo, misspell.

### Inverted Index

Estrutura canônica. Para cada **termo**, lista de **documentos** (postings) que contêm. Geralmente com:

- Frequência no doc.
- Posições (para phrase queries).
- Payloads opcionais.

Query "machine learning" = intersecção de postings de "machine" e "learning". Posting list comprimido (variable-byte, PFOR-Delta) para eficiência.

Ferramentas: **Lucene** é a base — Elasticsearch, OpenSearch, Solr derivados. Tantivy (Rust), Quickwit, Vespa, Typesense, Meilisearch.

### TF-IDF

Clássico scoring:

- **TF** (term frequency): quanto mais aparece no doc, maior score.
- **IDF** (inverse document frequency): termos raros valem mais. `log(N / df)`.
- **TF-IDF = TF × IDF**.

Problema: TF satura (doc com 1000 "a's" não é 1000x mais relevante). Entra BM25.

### BM25

Okapi BM25 (Robertson et al., 1994) — variante com **saturação de TF** e normalização por tamanho do doc:

```
score(doc, query) = Σ IDF(term) · (tf · (k+1)) / (tf + k · (1 - b + b · |doc|/avgdl))
```

- `k` (~1.2-2): controla saturação de TF.
- `b` (~0.75): penalidade por tamanho (docs longos tendem a ter mais palavras).

BM25 é **surpreendentemente forte**. Em muitos benchmarks, é baseline difícil de bater sem neural. Sempre comece com BM25.

### Phrase, proximity, field-weighted

- **Phrase query**: "machine learning" exato.
- **Proximity / slop**: termos próximos (ex.: dentro de 3 tokens).
- **Fields**: campos distintos (title, body, tags) com pesos distintos. Score é combinação.
- **BM25F**: BM25 com fields.

### Boolean

Query com AND/OR/NOT. Filtro antes de scoring, ou em camadas. Power users, enterprise search.

## Semântico (Dense)

### Embeddings

Mapeia texto (token, frase, doc) em vetor de dimensão fixa (128-4096). Próximo no espaço ≈ semanticamente relacionado.

- **Word embeddings** (Word2Vec, GloVe — 2013+): palavra por palavra.
- **Contextual** (BERT, 2018+): mesma palavra, contextos diferentes, embeddings diferentes.
- **Sentence/doc embeddings**: SBERT (Reimers & Gurevych, 2019); Sentence-Transformers lib. Trained via contrastive learning.
- **Cross-encoder** (para re-ranking): processa query + doc juntos, mais acurado, mais caro.
- **Multimodal**: CLIP (image + text), embeddings alinhados.

### Modelos populares (2025+)

- **OpenAI text-embedding-3-large**, **Cohere embed v3**.
- **BGE, GTE, E5, mxbai, Nomic, Voyage**: open/comercial competitivos.
- **jina-embeddings-v3, Snowflake Arctic**.
- **Matryoshka representation learning**: vetor pode ser truncado (2048 → 512) com perda graceful.

### Vector search

Achar vizinhos mais próximos em espaço vetorial.

- **Exato**: O(N) scan. Funciona até ~10k-100k vectors.
- **ANN (Approximate Nearest Neighbor)**: sacrifica % de recall para speed.

### ANN algoritmos

- **LSH** (Locality-Sensitive Hashing): hash functions que colidem para vizinhos. Simples, não estado-da-arte.
- **IVF** (Inverted File): cluster vetores (k-means); query checa só clusters próximos.
- **PQ (Product Quantization)**: comprime vetores em códigos pequenos.
- **IVF-PQ**: combinado. FAISS classic.
- **HNSW (Hierarchical Navigable Small World)**: grafo em camadas; busca top-down. State-of-the-art em qualidade × velocidade.
- **DiskANN**: HNSW-like com memória/disk otimizado.
- **SCANN** (Google): vantagem residual via anisotropic quantization.

### Vector DBs

- **Standalone**: Pinecone, Weaviate, Milvus, Qdrant, Chroma, LanceDB, Vespa.
- **Lucene/ES/OS**: HNSW built-in em versões recentes.
- **Postgres**: pgvector (IVF, HNSW); pgvectorscale (Timescale) extension otimizado.
- **Cloud**: Vertex AI Matching Engine, AWS OpenSearch kNN, Azure AI Search.
- **FAISS, Annoy, ScaNN**: libs em-process, não servidores.

Escolha prática: poucas centenas de milhares → pgvector é sufficient; milhões → especialistas (Milvus/Vespa/Qdrant/Pinecone); bilhões → Vespa, Milvus distribuído, ou sharding pesado.

### Embedding quality

- **Domain fit**: modelo genérico pode ser ruim em jurídico, médico, código. Fine-tuning ajuda.
- **Chunk size**: 200-500 tokens é baseline; muito grande → dilui; muito pequeno → perde contexto.
- **Overlap entre chunks**: ~20% típico.
- **Update cadence**: embeddings ficam stale quando conteúdo muda.
- **Versionamento**: modelo v2 não é intercambiável com v1 — re-embed all.

## Híbrido

Combinar lexical + semântico domina em 2024-2026:

### Reciprocal Rank Fusion (RRF)

Cada sistema dá seu ranking; combinar posições:

```
score(d) = Σ_systems 1 / (k + rank_system(d))
```

Simples, sem calibração entre scores. `k=60` costuma funcionar.

### Weighted combination

`score_final = α · BM25 + (1-α) · semantic_score`. Exige calibração de scores (diferentes distribuições).

### Learned fusion

Modelo (geralmente gradient-boosted tree) combina features. Maior performance, menos interpretável.

### Por que híbrido

- BM25 captura **termos raros** específicos (nome próprio, código de produto).
- Semântico captura **sinônimos, paráfrase**.
- Overlap pequeno entre seus top-k — juntos cobrem melhor.

## Learning to Rank (LTR)

Treinar modelo que ordena candidatos.

### Paradigmas

- **Pointwise**: prever score absoluto por doc. Regressão.
- **Pairwise**: prever qual de dois docs é mais relevante. RankNet, LambdaRank, LambdaMART (XGBoost).
- **Listwise**: otimizar métrica diretamente (nDCG). ListNet, ListMLE.

LambdaMART em Lucene/Solr LTR plugin, Elastic Learning to Rank, Vespa — clássicos.

### Features

- BM25 score.
- Freshness (idade do doc).
- Popularidade (views, clicks históricos).
- Autoridade (PageRank-like).
- Document quality signals.
- Query-document overlap (term match, BM25 por campo).
- Semantic similarity.
- Click-through rate histórico.

Qualidade do training data (labels) importa mais que modelo.

## Neural Rankers e Cross-Encoders

- **Bi-encoder (dual encoder)**: query e doc encodados separadamente; dot product. Escala mas menos preciso.
- **Cross-encoder**: query e doc juntos como input; pode modelar interação profunda. **Muito mais preciso mas O(N) por query** — use em re-ranking de top 50-100.

Popular: **BGE Reranker**, **Cohere Rerank**, **mxbai-rerank**, **Jina Reranker**. LLMs também usados como rerankers (LLM-as-ranker).

### ColBERT

Late interaction: tokens encodados independentemente; matching token-a-token em query time. Balanceia bi-encoder (fast) e cross-encoder (precise).

## Avaliação

### Métricas offline

**Precision@k, Recall@k, F1**: baseline, pois não capturam ordem no top-k.

**MRR (Mean Reciprocal Rank)**: `1 / rank do primeiro relevante`. Boa para casos "um certo documento".

**MAP (Mean Average Precision)**: média de precision at each relevant rank.

**nDCG (normalized Discounted Cumulative Gain)**: rico em graded relevance (não só 0/1). Ponderação logarítmica por posição. **Padrão de facto** em ranking.

```
DCG = Σ (2^rel_i - 1) / log2(i+1)
nDCG = DCG / IDCG  (ideal ordering)
```

**MRR@10**, **nDCG@10**, **Recall@100** são típicos.

### Benchmarks

- **MS MARCO** (Microsoft): passage e doc ranking.
- **TREC** tracks (DL, Fair, MIRACL).
- **BEIR**: 18 datasets de IR diversificados. Benchmark standard.
- **MTEB** (Massive Text Embedding Benchmark): embeddings cross-task.
- **MIRACL**: multilíngue.

### Online metrics

O que realmente importa em produção:

- **CTR (Click-Through Rate)**: clique em resultado top.
- **Dwell time**: tempo na página após clique.
- **Conversion**: compra, signup, subscribed.
- **Abandonment / reformulation**: usuário reformula query → sinal de insucesso.
- **Position click bias**: top-1 recebe muitos cliques só por estar no topo.

### A/B Testing

Divida usuários; compara métricas. Múltiplas semanas (sazonalidade, novelty effect). Cuidado com interference, multiple testing, leakage.

**Interleaving** (Radlinski): alterna resultados de dois rankings na mesma SERP; reduz variance; usado em escala (Netflix, Bing).

### Judged evaluation

Humano rotula. Caro, mas golden. Combinar com click log.

## Query Understanding

### Reformulação

- **Spell correction**: "netflics" → "netflix". Edit distance, language model.
- **Query expansion**: adicionar sinônimos (WordNet, learned).
- **Relevance feedback**: pseudo-RF (assume top-k relevant, expand query).
- **Semantic expansion**: embed query → buscar termos próximos.

### Intent classification

- **Navigational** ("facebook.com") → direto.
- **Informational** ("how to ...") → content.
- **Transactional** ("buy iphone 15") → produtos.
- **Ambiguous** → SERP diversificada.

### Entity extraction

"voos Rio Paris em abril" → origem/destino/data estruturados. Power search vertical (flights, hotels).

### Multi-turn

Conversational search — manter contexto entre queries. Foco recente com LLMs.

## Indexing e Scale

### Batch vs near-real-time

- **Batch**: reindex periódico. Simples.
- **NRT**: Lucene refresh (sec), Elasticsearch/OpenSearch.
- **Streaming**: Kafka → indexer.

### Sharding

Particionar índice:

- **Term-based**: cada shard guarda subset de termos. Pouco usado.
- **Document-based (padrão)**: docs distribuídos em shards; query vai a todos em paralelo. Scatter-gather.

### Replicação

Para availability e throughput de leitura.

### Merging

Inverted index é append-only; segments merge em background. Trade-off entre performance de write e read.

### Infra custosa

- Memory: índice grande em RAM é performance.
- Disk: SSD essencial.
- CPU em query (analysis, scoring).
- GPU opcional para neural.

## Caching

Ver `CACHING.md`.

- **Query cache**: resultado de query idêntica.
- **Filter cache**: resultado de filter booleano reutilizável.
- **Shard-level cache**.
- **CDN**: SERP static para queries super populares ("clima hoje" com location).

## Personalização

- **Por usuário**: histórico de clicks, preferências implícitas.
- **Contextual**: location, device, hora.
- **Por cohort**: segmentação.
- Cuidado: **filter bubble, privacy** (ver `PRIVACY_ENGINEERING.md`).

## Diversidade e Fairness

- **MMR (Maximal Marginal Relevance)**: score = relevance - λ · redundancy vs já selecionados. Reduz duplicação no top.
- **Submodular optimization** para diversidade.
- **Fairness**: exposure justa entre vendedores, produtores, demographics (Beutel, Singh).

## RAG (Retrieval-Augmented Generation)

Tornou-se dominante em aplicações LLM modernas:

1. Embed query.
2. Retrieve top-k chunks relevantes de knowledge base.
3. Injetar em contexto do LLM.
4. LLM responde com citação.

Beneficia de toda teoria de ranking. Problemas RAG-específicos:

- **Chunking strategy**.
- **Chunk-level vs document-level retrieval**.
- **Hybrid retrieval** (BM25 + semantic) vence naive semantic sozinho.
- **Re-ranking** top-50 → top-5 com cross-encoder.
- **Query rewrite / multi-query**: expande intenção via LLM antes do retrieve.
- **HyDE (Hypothetical Document Embeddings)**: gerar resposta hipotética, embed, buscar docs parecidos.
- **Contextual embeddings** (Anthropic 2024): prepend contexto do documento inteiro em cada chunk.
- **Evaluation**: MS MARCO generalistas; datasets específicos de RAG (RAG Evaluation, MIRAGE, HotpotQA).

Ferramentas: LangChain, LlamaIndex, Haystack, Vectara, Pinecone, Weaviate com LLM integrations.

## Search System Architecture

Setup típico:

```
Query → Query Parser → Understanding (intent, entity, expand)
  ├── Index retrieval (BM25, vector, hybrid)
  ├── Candidate generation (top-1000)
  ├── Feature extraction (per candidate)
  ├── LTR model → top-100
  ├── Re-ranker (cross-encoder or LLM) → top-10
  ├── Diversification / business rules / personalization
  └── SERP rendering
```

Cada camada contribui ms de latência — orçamento típico ~100-300ms p99. Budget tight.

## Especializações

### E-commerce

- **Facets** (filtros). Preço, brand, tamanho. Cardinality conta.
- **Relevance** ≠ popularity; balancear.
- **Stock**, promoção, personalization.
- **Typo tolerance** crítica.

### Code search

- Sourcegraph, GitHub code search, grep.app.
- AST-aware tokenization.
- Definition vs reference.

### Enterprise

- **Permissions** — mostrar só o que user tem acesso.
- **Data sources múltiplas** (Slack, Drive, Confluence, GitHub, email).
- **Freshness vs permanence**: docs recentes geralmente mais relevantes.

### Legal, medical, scientific

- Stemming específico (legal jargon).
- Citations network.
- Temporal (leis revogadas, artigos retraídos).

### Recommendation vs search

Diferente mas relacionado. Search = query explícita; recommendation = query implícita. Tooling convergindo (Vespa, Vertex AI, Databricks, etc.).

## Anti-Padrões

1. **Default-only (BM25 ou apenas semantic)**. Híbrido quase sempre melhor.
2. **Evaluate só offline**. Métricas offline ≠ user satisfaction.
3. **Todos os chunks iguais**. Chunking estratégico importa.
4. **Vector search sem filtros**. Filtros lexicais/categóricos complementam.
5. **Embedding stale**. Modelo v2 sem re-embed = qualidade cai silenciosa.
6. **Não observar queries**. Logs de query revelam UX problems.
7. **Top-1 optimization cega**. Diversidade/fairness importam em muitos contextos.
8. **LLM-as-ranker sem filtros**. Caro, vulnerável a injection (`AI_SECURITY.md`).

## Recursos

- **"Introduction to Information Retrieval"** — Manning, Raghavan, Schütze (gratuito, standard).
- **"Search Engines: Information Retrieval in Practice"** — Croft, Metzler, Strohman.
- **"Deep Learning for Search"** — Teofili.
- **"Relevant Search"** — Turnbull, Berryman.
- **"AI-Powered Search"** — Grainger, Trevor, Turnbull.
- **Trey Grainger talks, Daniel Tunkelang blog**.
- **BEIR, MTEB benchmarks papers**.
- **Elastic, Vespa, Milvus, Weaviate docs** — práticos.
- **SIGIR, CIKM, RecSys, ECIR** conferências.

## Princípios

1. **BM25 é baseline forte**. Qualquer coisa nova deve bater. Muitas vezes não bate.
2. **Híbrido lexical + semântico ganha**. Um sozinho é incompleto.
3. **Chunking importa muito em RAG**. Iterate.
4. **Evaluate em dados próprios**. MS MARCO ≠ seu domínio.
5. **Latência é feature**. P99 afeta engagement; budget sacro.
6. **Re-ranking small pays off**. Cross-encoder em top-50 é ROI alto.
7. **Online metrics dominam**. Offline guia, online decide.
8. **Instrumente queries + clicks**. Log rico é como você melhora.
9. **Personalize com parcimônia**. Filter bubbles são UX e ethical issue.
10. **Freshness, diversity, fairness**: explicitamente considere, não emergem automaticamente.
