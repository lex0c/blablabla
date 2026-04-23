# Data Science — preparação de dados para ML

"Garbage in, garbage out" é clichê por um motivo. 60-80% do tempo real de um projeto de ML é dados — coletar, entender, limpar, rotular, versionar, debugar. Arquitetura e hiperparâmetros são a parte visível; qualidade dos dados é o que decide se o modelo funciona.

Este arquivo foca em **preparação**. Fundamentos de treino estão em `ML_FOUNDATIONS.md`; ferramentas em `ML_FRAMEWORKS.md`.

## Princípio norteador

Modelos modernos são famintos e crédulos. Aprendem exatamente o que está nos dados — **inclusive os erros, vieses, atalhos e vazamentos**. Se existe correlação espúria explorável, a rede vai explorar. Toda decisão de dados é decisão de modelo.

Três perguntas antes de qualquer linha de código:

1. **Qual é o processo gerador?** Como esses dados foram produzidos no mundo real, por quem, com que viés de seleção?
2. **Qual é o rótulo exatamente?** Se dois anotadores humanos divergem, qual é a "verdade"?
3. **Qual é a distribuição de produção?** Treino e inferência vêm da mesma distribuição? Se não, o que muda?

## EDA (Exploratory Data Analysis)

Antes de treinar qualquer coisa: entender os dados. EDA não é opcional — é detecção precoce de problemas que, ignorados, custam semanas.

### Checklist mínimo

- **Shape e tipos**: `df.info()`, `df.describe(include='all')`, contagem de nulls por coluna.
- **Distribuições marginais**: histograma de cada numérica; value_counts de cada categórica. Skew, outliers, modalidades inesperadas.
- **Cardinalidade de categóricas**: coluna com 50k valores únicos em 100k linhas é quase-id — inútil ou tóxica.
- **Correlação com target**: Pearson/Spearman para numéricas, mutual information para não-lineares, chi² / Cramér's V para categóricas.
- **Correlação entre features**: multicolinearidade não mata árvore mas afeta interpretação e modelos lineares.
- **Padrões de missingness**: missing correlacionado com target é sinal — não só ruído.
- **Duplicatas**: exatas e aproximadas.
- **Target leakage**: feature que só existe *depois* do evento alvo (timestamp de churn em cliente já churned).

### Ferramentas

- **pandas / polars** — manipulação. Polars é ~10× mais rápido em dados médios; API mais limpa.
- **ydata-profiling** (ex `pandas-profiling`) — relatório HTML automático. Útil para primeira passada.
- **sweetviz, dtale, AutoViz** — variantes com UIs diferentes.
- **matplotlib / seaborn / plotly** — plots ad-hoc. Plotly interativo é útil para inspecionar outliers.
- **DuckDB** — SQL direto em Parquet/CSV sem carregar em RAM. `duckdb.sql("SELECT ... FROM 'file.parquet'")`.
- **great_expectations, pandera** — codificar assumptions como testes.

### Alvos específicos a olhar

- **Distribuição do target**: desbalanceado? Long-tail? Multi-modal?
- **Taxa de missingness por coluna**: >50% missing geralmente = drop ou feature binária "present".
- **Distribuições por split temporal**: feature muda entre período inicial e final? Possível drift já no treino.
- **Distribuições por grupo sensível**: demografias, regiões, clientes — revela viés e risco de fairness.

## Limpeza

### Missing values

Tipologia (Rubin):

- **MCAR** (Missing Completely At Random): missingness independente de tudo. Imputar é seguro.
- **MAR** (Missing At Random): missingness depende de outras features observadas. Imputação condicional funciona.
- **MNAR** (Missing Not At Random): missingness depende do próprio valor faltante (renda alta que não reporta). Imputação é enviesada; às vezes melhor feature indicadora.

Estratégias:

- **Drop linha** — só se poucas linhas e MCAR.
- **Drop coluna** — se >50-70% missing e sem valor preditivo.
- **Imputação simples**: média/mediana (numérica), moda ou "Missing" (categórica). Mediana > média com outliers.
- **Imputação por grupo**: mediana por categoria (ex: preço mediano por bairro).
- **KNN imputation** (`sklearn.impute.KNNImputer`): usa vizinhos. Caro em datasets grandes.
- **Iterative / MICE** (`IterativeImputer`): prediz cada coluna condicionada nas outras, itera. Boa qualidade.
- **Feature indicadora**: `col_was_missing = col.isna().astype(int)` + imputação. Preserva sinal de MNAR.
- **Árvores tratam NaN nativamente** (LightGBM, XGBoost, CatBoost) — não imputar pode ser melhor.

**Regra de ouro**: imputação estatística deve ser fittada **no train** e aplicada em val/test. Nunca fit no dataset inteiro pré-split.

### Outliers

Detecção:

- **Z-score** (|z| > 3): assume normalidade.
- **IQR** (fora de `[Q1 - 1.5·IQR, Q3 + 1.5·IQR]`): robusto.
- **Isolation Forest, LOF**: multivariado.
- **Domain rules**: idade > 150, preço < 0.

Decisão:

- **Erro de medida** → drop ou correct.
- **Valor raro mas legítimo** → manter. Outlier ≠ erro.
- **Modelo sensível** (linear, KNN) → winsorize, log-transform, ou RobustScaler.
- **Modelo robusto** (árvores, boosting) → geralmente deixa.

### Duplicatas

- **Exatas**: `df.drop_duplicates()`.
- **Near-duplicates**: mesmo conteúdo com diferença trivial (whitespace, case, timestamp). Crítico em texto e imagens — ver seção Deduplicação.
- **Duplicatas com target diferente**: indicam ruído de rotulagem ou feature insuficiente. Agregue ou investigue.

### Consistência

- Tipos coerentes (datas parseadas, numéricas como números e não strings).
- Units: misturar kg e lb em `weight` acontece mais do que se admite.
- Encoding: UTF-8 vs Latin-1 vs mojibake. `ftfy` resolve bastante.
- Trailing/leading whitespace em strings. `.str.strip().str.lower()` é quase sempre necessário.
- Timezones: armazenar em UTC, converter para display.

## Feature Engineering (tabular)

Ainda relevante mesmo na era do deep learning — tabular é o domínio onde gradient boosting + boas features ainda bate rede neural na maioria dos casos.

### Numéricas

- **Scaling**:
  - **StandardScaler** (média 0, std 1): padrão para linear, SVM, redes.
  - **MinMaxScaler** ([0,1]): quando limites são conhecidos.
  - **RobustScaler** (mediana, IQR): resistente a outliers.
  - **QuantileTransformer** (uniforme ou normal): mapeia para distribuição alvo; forte em features enviesadas.
  - **PowerTransformer** (Yeo-Johnson, Box-Cox): normaliza skew.
- **Log / log1p**: receita, renda, contagens com cauda longa.
- **Binning**: contínuo → ordinal (quantile bins, tree-based bins). Perde informação mas ganha robustez e não-linearidade em modelos lineares.
- **Interactions**: `x1 * x2`, `x1 / x2`, polynomial features. Árvores descobrem sozinhas; lineares precisam explícitas.

Árvores são **invariantes a transformações monotônicas** — não scale para XGBoost/LightGBM/CatBoost. É puro desperdício.

### Categóricas

- **One-hot**: baixa cardinalidade (<10-50). Explode memória em alta card.
- **Ordinal / label encoding**: quando existe ordem natural (educação: fundamental < médio < superior). Usar em árvores mesmo sem ordem — toleram.
- **Target encoding** (mean encoding): substitui categoria pela média do target condicionada a ela. Poderoso, mas **leakage óbvio** — fit com cross-validation out-of-fold ou smoothing. CatBoost faz nativamente com ordered target encoding.
- **Frequency encoding**: substitui por contagem. Barato, útil em alta cardinalidade.
- **Hashing trick**: hash da categoria mod K. Controla dimensionalidade; colisões toleráveis em modelos lineares.
- **Embeddings**: aprendidos (entity embeddings de Guo & Berkhahn, ou via rede). Custo-benefício em alta cardinalidade + deep learning.

### Datetime

Extrair:

- Hora, dia da semana, dia do mês, semana do ano, mês, trimestre, ano.
- Is_weekend, is_holiday, is_business_hour.
- Cíclico: `sin(2π·hour/24)`, `cos(2π·hour/24)` — continuidade entre 23h e 0h.
- Delta para eventos relevantes (dias desde último login, tempo até próximo feriado).

### Texto (clássico, não-LLM)

- **Bag-of-words, TF-IDF**: baseline forte para classificação de texto. `TfidfVectorizer` do sklearn.
- **Char n-grams**: robusto a ruído e idiomas.
- **Embeddings pré-treinados**: word2vec, GloVe, fastText para tokens; sentence-transformers para sentenças.
- **Estatísticas**: comprimento, contagem de palavras, ratio upper/lower, número de URLs/emails/hashtags.

### Geoespacial

- Haversine distance entre coordenadas.
- Geohash / H3 (Uber) para bucketizar regiões.
- Distância a pontos de interesse.

### Agregações (feature store logic)

Em dados relacionais/transacionais: feature é quase sempre **agregação temporal**. Para cada entidade (user, device), nas últimas N janelas:

- Count, sum, mean, std, min, max, p50, p95.
- Contagem de eventos distintos.
- Time-since-last, time-between.
- Rolling windows com decay exponencial.

Framework: **featuretools** (automatizado), **tsfresh** (séries temporais). Em produção: **Feast, Tecton, Hopsworks** (feature stores reais).

## Classes desbalanceadas

Dataset onde classe minoritária é 1-5% (ou menor) do total. Acurácia vira métrica inútil.

### Estratégias

- **Métrica correta**: precision, recall, F1, PR-AUC, cost-sensitive loss. Acurácia e ROC-AUC enganam em desbalanceamento severo.
- **Class weights**: `class_weight='balanced'` em sklearn, `scale_pos_weight` em XGBoost. Penaliza erros na minoria. Barato, muitas vezes suficiente.
- **Oversampling**:
  - **Random oversampling**: replica minoria. Risco de overfit.
  - **SMOTE** (Synthetic Minority Oversampling): interpola entre vizinhos da minoria. Variantes: Borderline-SMOTE, ADASYN, SMOTENC (mistura com categóricas).
- **Undersampling**:
  - **Random undersampling**: descarta maioria. Perde informação.
  - **Tomek links, ENN, NearMiss**: remove majoritários próximos de minoritários (cleaning boundaries).
- **Ensembles balanceados**: BalancedRandomForest, EasyEnsemble (`imbalanced-learn`).
- **Focal loss** (Lin et al.): reduz peso de exemplos fáceis (majoritários bem classificados). Padrão em detecção de objetos.
- **Threshold tuning**: treinar com distribuição natural, ajustar threshold de decisão no PR curve. Muitas vezes o mais simples que funciona.

**Anti-padrão**: oversample antes do split → leakage (cópias da minoria em train e val). Sempre oversample **dentro do fold de treino**.

### Detecção de anomalia vs classificação desbalanceada

Se minoria é <0.1% ou mal definida, muitas vezes melhor tratar como anomaly detection (Isolation Forest, One-Class SVM, autoencoder reconstrução) em vez de supervisionado.

## Splits e validação

(Complementa a seção em `ML_FOUNDATIONS.md`.)

### Tipos

- **Random hold-out**: simples; IID suficiente.
- **Stratified**: preserva proporção de classes. Obrigatório em desbalanceado.
- **GroupKFold**: dados com agrupamento natural (paciente com múltiplas visitas, usuário com múltiplas sessões). **Nunca** divida linhas do mesmo grupo entre train e val — inflaciona métricas drasticamente.
- **TimeSeriesSplit**: split temporal; train sempre antes de val. Pode ser expanding window (train cresce) ou rolling window (tamanho fixo).
- **Purged/Embargoed CV** (de Prado, finance): time series com gap entre train e val para evitar leakage de autocorrelação.
- **Nested CV**: CV externo para avaliar, CV interno para hyperparam tuning. Única forma honesta quando dataset é pequeno.

### Leakage — catálogo de erros comuns

1. **Target em feature**: coluna derivada diretamente do alvo (churn_date para prever churn).
2. **Normalização pré-split**: fit do StandardScaler no dataset inteiro, split depois. Vaza mean/std.
3. **Imputação pré-split**: idem.
4. **Seleção de feature pré-split**: rodar correlação com target no dataset todo e escolher top-K → escolha usou informação de val/test.
5. **Oversampling pré-split**: ver seção anterior.
6. **Group leakage**: mesmo grupo em train e val.
7. **Temporal leakage**: futuro no train, passado no val.
8. **Near-duplicate leakage**: augmentação ou download repetido coloca quase-cópia em splits diferentes.
9. **Contaminação de benchmark**: modelo pré-treinado já viu o test set. Crítico em avaliação de LLM.

### Boas práticas

- **Pipeline sklearn**: `Pipeline([scaler, imputer, model])` + CV garante que transformações são fittadas só em train dentro de cada fold.
- **Dataset card**: documente procedência, filtros, data de extração, tamanho de cada split, balanceamento.
- **Congelar test**: mover para pasta separada, nunca rodar até modelo final.

## Tokenização (texto)

Fundamental para todo o stack NLP moderno. Escolha impacta vocabulário, OOV, eficiência, e qualidade.

### Algoritmos

- **BPE** (Byte-Pair Encoding, Sennrich 2016): merge iterativo dos pares mais frequentes. GPT, RoBERTa.
- **WordPiece**: variante BPE otimizando likelihood. BERT.
- **SentencePiece** (Kudo): opera em bytes brutos sem pre-tokenizer, trata whitespace como caractere normal. T5, LLaMA, Mistral.
- **Unigram LM**: probabilístico; escolhe subset de subwords maximizando likelihood. T5 usa variante.
- **BBPE** (Byte-level BPE): BPE sobre bytes em vez de unicode. Cobre qualquer string, sem UNK. GPT-2+, LLaMA 3.
- **Tiktoken** (OpenAI): BBPE otimizado; open-source, rápido.

### Decisões

- **Vocab size**: 32k-128k é típico; maior vocab = menos tokens por doc mas embedding maior. LLaMA 3 foi para 128k.
- **Idiomas**: tokenizer treinado só em inglês quebra línguas com muitos caracteres raros (japonês, árabe) — explode em tokens, performance cai. Multilingual tokenizer compensa.
- **Código**: tokenizer de prosa é péssimo para código (quebra identifiers, perde indentação). Code-trained tokenizers (StarCoder, CodeLlama) são ~30% mais eficientes em fontes.
- **Números**: alguns tokenizers quebram "1234" em "1", "23", "4"; outros têm dígitos separados. Afeta raciocínio aritmético.
- **Special tokens**: `<|im_start|>`, `<|im_end|>`, `<|system|>`, `<|tool_call|>`, etc. Crítico em chat e tool use; erro silencioso se trocado.

### Prática

```python
from tokenizers import Tokenizer, models, trainers, pre_tokenizers

tok = Tokenizer(models.BPE())
tok.pre_tokenizer = pre_tokenizers.ByteLevel(add_prefix_space=False)
trainer = trainers.BpeTrainer(vocab_size=32000, special_tokens=["<|endoftext|>"])
tok.train(files=["corpus.txt"], trainer=trainer)
```

Para usar modelo existente: sempre `AutoTokenizer.from_pretrained(MODEL)` — vocab + special tokens + chat template.

## Deduplicação (especialmente para LLM pretrain e SFT)

Duplicatas inflam loss em amostras repetidas, reduzem diversidade efetiva, e em LLMs **causam memorização direta**. Paper do GPT-3 e subsequentes mostraram deduplication melhora perplexity e reduz regurgitation.

### Exata

- Hash (SHA-256 ou xxhash) do conteúdo normalizado → dict.
- Normalizar antes: lowercase, remove whitespace redundante, trim.

### Near-duplicate

- **MinHash + LSH** (`datasketch`): assinaturas de k-shingles; Jaccard aproximado em tempo sublinear.
- **SimHash**: hash preservando similaridade; bit distance ≈ cosine.
- **Embeddings + FAISS**: embeddar com modelo pequeno (sentence-transformers), buscar vizinhos, threshold cosine.
- **Suffix arrays** (Lee et al. 2022): encontra substrings duplicadas exatas >N chars — padrão em pretrain de LLM.
- **SemDeDup** (Abbas et al.): cluster por embedding, mantém representante de cada cluster.

### Parâmetros

- **Shingle size (k)**: 5-9 palavras para texto. Menor = mais sensível, mais falsos positivos.
- **Jaccard threshold**: 0.7-0.9 típico. Depende do domínio.
- **Escala**: datasets de pretraining têm bilhões de documentos. Ferramentas: **text-dedup** (HuggingFace/BigScience), **Datasketch**, **GTE-dedup**. Spark para distribuir.

### Imagens

- **Perceptual hashing** (pHash, dHash, wHash).
- **Embedding-based** (CLIP, DINO) + FAISS.
- **FastDup** — ferramenta focada em dedup/quality de datasets visuais.

## Contamination (benchmark leakage)

O LLM pode ter visto o test set durante pretraining. Invalida avaliação.

### Tipos

- **Exact contamination**: benchmark copiado verbatim em common crawl.
- **Paraphrased contamination**: reformulação do teste nos dados.
- **Label contamination**: enunciado + resposta correta presente (pior caso).

### Mitigação

- **Decontamination**: remover do train docs que contêm substrings >N chars de test. MMLU, BIG-bench documentam isso.
- **Held-out benchmarks**: criados depois do cutoff do modelo (LiveBench, SWE-bench, GPQA).
- **Canary strings**: sequências únicas em benchmarks; se modelo completa, contaminou.
- **Dynamic eval**: gerar variações do teste on-the-fly.

Contamination é suspeita em todo ranking. "Beats GPT-4 on MMLU" com modelo treinado depois do MMLU existir deve soar alarme.

## Curadoria de dados para fine-tuning LLM

Crítica em SFT (Supervised Fine-Tuning) e RLHF/DPO. Qualidade >> quantidade.

### Formatos

- **Alpaca**: `{"instruction": ..., "input": ..., "output": ...}`. Legado mas comum.
- **ShareGPT / OpenAI chat**: `[{"role": "user", "content": ...}, {"role": "assistant", "content": ...}]`. Padrão atual.
- **JSONL** é o formato de storage default. Um exemplo por linha.
- **Preference pairs** (DPO/ORPO/KTO): `{"prompt": ..., "chosen": ..., "rejected": ...}`.

### Quantidade

LIMA paper (Meta, 2023): 1000 exemplos bem curados bate 50k ruidosos. Replicado.

- **SFT de estilo/formato**: 500-5000 exemplos de alta qualidade.
- **SFT de capacidade nova** (reasoning, tool use): 10k-100k+.
- **Instruction tuning geral**: 50k-500k (Dolly, ShareGPT, OpenAssistant, UltraChat).

### Qualidade — filtros

- **Length filtering**: remover muito curto (<50 tokens) ou muito longo (>max_seq).
- **Language detection** (fasttext lid.176): manter só línguas alvo.
- **Perplexity filtering**: modelo base prediz probabilidade do texto; extremos (alta = ruído, baixa = repetitivo) são descartados.
- **Heuristic filters** (CCNet, Gopher): ratio de stopwords, pontuação, caracteres especiais, repetição.
- **Quality classifiers**: treinar classificador leve (fastText, small LM) com exemplos bons/ruins.
- **LLM-as-judge**: usar GPT-4/Claude para scorear cada exemplo (1-5); filtrar <3.
- **Duplicate removal**: ver seção de dedup.
- **Contaminação**: remover se solapa com benchmarks que pretende avaliar.

### Diversidade

Diversity > repetition. Medidas:

- **Embedding-based diversity**: cluster, maximize cobertura.
- **Task diversity**: taxonomia manual (código, matemática, escrita, análise, role-play, ...).
- **Difficulty spread**: não só fácil nem só hard.

### Sistema e roles

- **System prompt**: define persona/regras. Muitos datasets abertos têm system vazio ou trivial — prejudica instruction following condicionado.
- **Multi-turn**: treino só em single-turn colapsa capacidade de conversa. Incluir diálogos de 3-10 turnos.
- **Refusals**: ensinar a recusar quando apropriado (sem over-refusal). Balanço delicado.

### Para preference (DPO/KTO)

- **Chosen** claramente melhor que **rejected** para o prompt.
- Margem pequena (chosen e rejected parecidos) > margem grande em aprendizado — gradiente útil.
- **UltraFeedback, HH-RLHF, OpenAssistant** — datasets base.
- **Self-play**: modelo gera duas respostas, LLM juiz escolhe.

### Receitas modernas

- **Evol-Instruct** (WizardLM): LLM reescreve instruções para torná-las mais complexas iterativamente.
- **Self-Instruct** (Wang et al.): LLM gera novas instruções a partir de seed.
- **Magpie** (Xu et al.): extrai instruction do próprio LLM alinhado via prompt vazio — zero human seed.
- **Persona-Hub** (Tencent): 1B personas sintéticas como condicionantes para diversidade.

## Anotação (rotulagem humana)

Labels não caem do céu. Qualidade do rótulo = teto do modelo.

### Processo

1. **Guideline escrito**: o que conta como cada label, casos-limite, exemplos. Iterado em discussão.
2. **Annotators treinados**: testados em gold set antes de anotar.
3. **Overlap**: múltiplos anotadores no mesmo item (2-3× é padrão) para medir agreement.
4. **Adjudication**: casos de divergência revisados por árbitro.
5. **Gold set**: subset rotulado por experts, usado para treinar/validar anotadores e medir drift.
6. **Revisão contínua**: guideline evolui; re-rotular amostras antigas quando muda.

### Métricas de agreement

- **Cohen's κ** (dois anotadores, categorias): ajusta por concordância aleatória. >0.8 excelente, 0.6-0.8 bom, <0.4 ruim.
- **Fleiss' κ** (múltiplos anotadores).
- **Krippendorff's α**: qualquer número de anotadores, qualquer tipo de dado (nominal, ordinal, interval, ratio).
- **F1 pairwise** em labels de spans/NER.

Baixo agreement geralmente = guideline ambíguo, não anotadores ruins. Fix upstream.

### Ferramentas

- **Label Studio** — open-source, generalista. Texto, imagem, áudio, vídeo.
- **Prodigy** — de Explosion (spaCy); rápido, focado em active learning e NLP.
- **CVAT** — visão computacional (bbox, segmentação, vídeo).
- **doccano** — NLP simples, open-source.
- **Argilla** — LLM feedback, preferências, conversas.
- **Scale AI, Labelbox, SuperAnnotate, Snorkel** — enterprise.

### Active learning

Em vez de rotular aleatório: priorizar exemplos onde modelo está **incerto** (baixa confiança, alta entropia, disagreement de comitê). Rotula 10-20% dos dados para chegar perto do máximo.

- **Uncertainty sampling**: margin, entropy, least confidence.
- **Query-by-committee**: ensemble, vote disagreement.
- **BALD** (Bayesian): maximiza mutual information entre predição e pesos.

### Weak supervision

Programmatic labeling quando anotação manual não escala.

- **Snorkel**: labeling functions (regras, heurísticas, modelos fracos) + generative model para resolver conflitos → labels probabilísticos.
- **Data programming**: expressa conhecimento de domínio como funções, não como labels individuais.

## Synthetic data

Dados gerados por modelo. Crescimento explosivo 2023-2026 — deixou de ser truque e virou pipeline dominante em SFT.

### Usos

- **Pré-treino** (Phi series, Microsoft): textbooks sintéticos de alta qualidade superam web crawl a igual volume.
- **SFT**: instruções geradas via self-instruct, evol-instruct, magpie.
- **Reasoning traces**: gerar chain-of-thought via modelo maior, fine-tune modelo menor (distilação de raciocínio — DeepSeek-R1 distills, OpenAI o1-style).
- **Augmentation**: paraphrase, back-translation.
- **Privacy**: tabular sintético (SDV, Gretel) para compartilhar dados sem PII.

### Riscos

- **Model collapse** (Shumailov et al.): treinar recursivamente em outputs próprios degrada distribuição. Caudas desaparecem.
- **Viés amplificado**: synthetic herda vieses do gerador, acentua em loop.
- **Hallucination embedded**: geração incorreta vira label de treino → modelo aprende o erro.
- **Copyright / PII**: modelo gerador pode regurgitar dados proprietários.

### Mitigações

- **Hybrid**: misturar real + sintético. Real ancora distribuição.
- **Filtering agressivo**: LLM juiz, rejection sampling, verificadores (executar código, checar fatos).
- **Ground truth validação**: synthetic testes executáveis; matemática com verificador simbólico.
- **Diversidade forçada**: temperature alta na geração, personas variadas, seeds diversas.

### Geradores tabulares

- **SDV** (Synthetic Data Vault): modelos clássicos + copulas + GANs para tabular.
- **CTGAN, TVAE**: redes específicas para tabular.
- **Gretel, MOSTLY AI, Tonic**: comerciais.

## Data pipelines e formatos

### Formatos de arquivo

| Formato | Uso |
|---|---|
| **CSV** | Interoperável, human-readable. Lento, sem tipos, sem schema. Evite em pipelines sérios. |
| **JSON / JSONL** | Flexível, aninhado. JSONL é o default para LLM datasets. Leitura streaming. |
| **Parquet** | Colunar, comprimido, schema, predicate pushdown. Padrão moderno para tabular analítico. |
| **Arrow / Feather** | In-memory columnar zero-copy. Base de polars, DuckDB, pandas 2+. |
| **HDF5** | Hierárquico, grande, científico. Legado em deep learning. |
| **TFRecord** | Binário do TF. Protobufs sequenciais. Ainda ok em ecossistema TF. |
| **WebDataset** | Tar shards de arquivos. Ideal para streaming em treino de rede (torchdata, HF). |
| **MDS / Mosaic StreamingDataset** | Shards otimizados para LLM pretraining distribuído. |
| **LanceDB / Delta / Iceberg** | Table formats sobre Parquet com transactions, time travel. |

### Escolha

- **Tabular analítico / ML clássico**: Parquet.
- **Texto para LLM SFT**: JSONL.
- **Pretrain LLM**: MDS, WebDataset, ou Parquet + tokenização online.
- **Imagens para treino**: WebDataset shards (tar com imagem+metadata) >> milhões de arquivos soltos (destrói filesystem).

### Streaming vs download

Datasets grandes (>100GB) não cabem em disco local. Opções:

- **HF `datasets` streaming=True**: lazy, não materializa.
- **WebDataset** em S3/GCS: shards baixados sob demanda.
- **Mosaic StreamingDataset**: otimizado para multi-node, random sampling via index.

### num_workers e prefetch

DataLoader Python:

- **num_workers = 2-8 × GPU count** típico. Depende de CPU, I/O, transform cost.
- **pin_memory=True**: acelera transfer CPU→GPU.
- **persistent_workers=True**: evita re-spawn a cada epoch.
- **prefetch_factor**: quantas batches cada worker pré-carrega.

Se GPU fica ociosa entre steps, dataloader é gargalo. Profile com `nvidia-smi dmon` ou PyTorch profiler.

### ETL / orquestração

- **Airflow, Prefect, Dagster** — orquestração geral.
- **dbt** — transformação SQL em warehouse.
- **Spark / Dask / Ray** — processamento distribuído.
- **DuckDB** — SQL single-node super rápido em Parquet.
- **Polars** — dataframe single-node > pandas em performance.

Regra: **só distribuir quando realmente precisa**. DuckDB/Polars em máquina forte processa 100GB com tranquilidade.

## Qualidade e profiling

### Data contracts / expectations

Codificar assumptions sobre dados como testes executáveis:

- **Great Expectations**: expectations declarativas (`expect_column_values_to_be_between`, `expect_column_values_to_not_be_null`). Gera data docs.
- **Pandera**: schemas pandas/polars com validação tipada.
- **pydantic** para rows individuais.
- **Deequ / PyDeequ**: Spark-native.

Rode em CI/CD do pipeline. Quebra se dado upstream mudou → alerta antes de treinar em dados podres.

### Profiling

- **ydata-profiling**: relatório automático.
- **Deepchecks**: verificações ML-específicas (drift, leakage, class imbalance, weak segments).
- **Evidently**: drift, performance, produção (dashboards).
- **WhyLabs / whylogs**: profiling escalável, lineage.

### Monitoramento em produção

- **Feature drift**: KL divergence, PSI (Population Stability Index), KS test por feature.
- **Prediction drift**: distribuição de outputs muda mesmo com input estável.
- **Label delay**: em muitos sistemas label só chega semanas depois (churn, fraud). Monitorar proxies.
- **Alarme != causa**: drift detectado exige investigação, não retreino automático cego.

## Versionamento

Dado é código. Sem versão, experimento não é reproduzível.

- **DVC** — git-like para datasets/modelos. Pointers em git, dados em S3/GCS/Azure/local. `dvc add data/`, `dvc push`.
- **LakeFS** — git-like sobre object store, mais pesado/enterprise.
- **Pachyderm** — versionamento + pipelines.
- **HF Hub datasets** — versionamento via commits, LFS.
- **Delta Lake / Iceberg** — time travel em warehouse.

Mínimo viável: dataset tem **hash** + **data de snapshot** + **script de geração**. Commit esses no repo junto do código do modelo.

## Privacy, PII, compliance

### Identificação

- **PII direta**: nome, CPF, email, telefone, endereço.
- **Quasi-identifiers**: CEP + idade + gênero → reidentificação em 87% dos americanos (Sweeney).
- **Sensitive**: saúde, orientação, religião, biometria.

### Técnicas

- **Remoção/mascaramento**: regex para emails, CPFs. Cuidado com falsos negativos em texto livre — melhor NER + denylist.
- **Pseudonimização**: substituir por token consistente. Reversível (com chave).
- **Hashing**: irreversível mas determinístico (permite join).
- **k-anonymity**: cada combinação de quasi-ids com ≥k indivíduos.
- **ℓ-diversity, t-closeness**: proteção adicional contra ataques homogêneos.
- **Differential privacy**: ruído calibrado em queries/aggregates; garantia matemática (ε-DP). Trade-off com utilidade.
- **Federated learning**: treino distribuído sem centralizar dados.
- **Secure enclaves (SGX, TEE)**: computação em hardware isolado.

### Regulação

- **GDPR** (EU), **LGPD** (BR), **HIPAA** (saúde US), **CCPA** (California). Definições de PII, consentimento, direito ao apagamento (right to erasure) impactam diretamente ML — "desfazer treino" é problema aberto (machine unlearning).

## Viés e fairness

Dados refletem o mundo, viés incluso. Modelo amplifica se não tratado.

### Fontes de viés

- **Seleção**: amostra não representa população. Quem está nos dados, quem está de fora.
- **Histórico**: processo passado era enviesado (crédito, contratação).
- **Medição**: proxies imperfeitos (arrest ≠ crime, diagnosis ≠ doença).
- **Anotação**: preconceitos dos rotuladores.
- **Agregação**: padrão populacional não se aplica a subgrupos.

### Métricas

- **Demographic parity**: P(ŷ=1|A=0) = P(ŷ=1|A=1). Critério forte e muitas vezes inapropriado.
- **Equalized odds**: TPR e FPR iguais entre grupos.
- **Equal opportunity**: só TPR igual.
- **Calibration**: P(y=1|ŷ=p, A=a) = p independente de A.

**Impossibility theorem** (Kleinberg/Chouldechova): em base rates diferentes, não se pode satisfazer simultaneamente demographic parity + equalized odds + calibration. Escolha normativa.

### Mitigação

- **Pré-processamento**: reweighting, resampling por grupo.
- **In-processing**: loss regularizado por fairness (adversarial debiasing).
- **Pós-processamento**: thresholds por grupo para equalizar métricas.

Nenhuma "solução técnica" — problema é sociotécnico. Envolver stakeholders afetados.

## Princípios

1. **Dados dominam o resultado**. 20% do ganho vem de mudar modelo; 80% vem de mudar dados.
2. **Data cleaning nunca termina**. Cada iteração revela problemas novos.
3. **Entenda o processo gerador** antes de pre-processar. "Missing" pode ser sinal.
4. **Leakage é o erro mais fatal** — verifique de 3 ângulos diferentes antes de publicar número.
5. **Qualidade > quantidade** em SFT/instruction tuning. Em pretraining a equação inverte, mas com limiar de qualidade.
6. **Duplicate é tóxico**. Inflaciona métrica e causa memorização.
7. **Contamination invalida benchmark**. Suspeite de rankings.
8. **Anotação é produto, não tarefa**. Guideline, agreement, iteração.
9. **Synthetic data é alavanca, não substituto**. Mistura com real; filtra agressivamente.
10. **Versionamento é pré-requisito de reprodutibilidade**. Hash o dataset, commit o pipeline.
11. **Privacy não é opcional**. Planeje antes da coleta, não depois.
12. **Fairness exige escolha normativa**. Métricas técnicas não substituem discussão ética.

## Recursos

- **"Feature Engineering for Machine Learning"** — Alice Zheng, Amanda Casari.
- **"Designing Machine Learning Systems"** — Chip Huyen.
- **"The Hundred-Page Machine Learning Book"** — Burkov (overview).
- **"Applied Predictive Modeling"** — Kuhn, Johnson.
- **"Fairness and Machine Learning"** — Barocas, Hardt, Narayanan (livre, fairmlbook.org).
- **"Data Management for Machine Learning"** — Polyzotis et al., Google (paper).
- **CS329S Stanford** — Machine Learning Systems Design (Chip Huyen).
- **HuggingFace docs** — `datasets`, tokenizers, data processing recipes.
- **Awesome Data Labelling, Awesome Synthetic Data** — listas curadas.
