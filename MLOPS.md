# MLOps

**MLOps** é a disciplina que leva modelos de ML do notebook à produção e os mantém saudáveis em escala. Combina práticas de DevOps (versionamento, CI/CD, observability) com peculiaridades de ML: **dados como artefato**, **reprodutibilidade probabilística**, **deriva**, **re-treino**, **multi-artefato** (modelo + features + pipeline + métricas). Um modelo treinado é inútil sem pipeline que o entregue confiável e que saiba quando ele deteriora.

Complementa `AI.md`, `AI_SECURITY.md`, `OBSERVABILITY.md`, `APPSEC_SDLC.md`, `DATA_ENGINEERING.md`.

## Por Que ML É Diferente

Software tradicional:

- Código versionado, testável, determinístico.
- Uma vez deployado, comportamento constante até próximo release.

ML:

- **Três artefatos** — código + dados + modelo. Todos evoluem independentemente.
- **Não-determinístico** em treino (seeds, paralelismo) e às vezes em inference.
- **Comportamento degrada com tempo** (distribution shift) mesmo sem mudança de código.
- **Avaliação é probabilística** — acurácia, F1, não "passou/falhou".
- **Pipeline de produção ≠ de pesquisa**. Feature em notebook ≠ feature em produção (training-serving skew).
- **Custo de compute** dominante em treino.

## Ciclo de Vida

### Fases

1. **Discovery / Business case**: problema vale ML? Qual métrica de negócio?
2. **Data collection + labeling**: preparação; label quality é fundação.
3. **Exploration / EDA**: entender distribuições.
4. **Feature engineering**.
5. **Training + experimentation**: muitos experimentos; rastreamento obrigatório.
6. **Evaluation**: offline + online.
7. **Deployment**: serving + integração com aplicação.
8. **Monitoring**: métricas ML + sistêmicas.
9. **Feedback loop**: novos dados, re-treino, iteração.

Diferente de waterfall — iteração continua indefinidamente.

### Níveis de maturidade (Google)

- **MLOps 0**: manual, pesquisa fica com pesquisador, deploy é zip no servidor.
- **MLOps 1**: pipeline automatizado de re-treino.
- **MLOps 2**: CI/CD completo com test/validação de pipeline, auto-deploy, monitoramento.

Maioria das empresas está entre 0 e 1. MLOps 2 é objetivo, não ponto de partida.

## Versionamento

### Código

Git + branches. Nada novo em relação a `COMMIT.md`.

### Dados

- **DVC**: versiona dados como código. Metadata em git, conteúdo em S3/GCS/etc.
- **LakeFS**: Git-like para data lakes.
- **Delta Lake, Apache Iceberg, Apache Hudi**: time travel em tables (lakehouse).
- **Pachyderm**: data lineage automático.
- **Snapshot em warehouse**: snapshot tables em BigQuery/Snowflake.

**Princípio**: experimento reprodutível exige hash/versão do dataset usado, não só código.

### Modelos

- **MLflow Model Registry**: artefato + metadata + staging/production.
- **Weights & Biases (W&B) Artifacts**.
- **Vertex AI Model Registry, SageMaker Model Registry, Azure ML**.
- **Hugging Face Hub**: models + datasets + spaces.
- **Bento / BentoML**: modelos empacotados.

### Experimentos

- **MLflow Tracking**: parâmetros, métricas, artefatos.
- **Weights & Biases**: forte em dashboards, comparação.
- **Neptune.ai, Comet, ClearML, Aim**.
- **TensorBoard**: estado antigo mas sólido.

## Feature Stores

Problema central: feature engineering em produção repete lógica de treino — facilmente divergente (*training-serving skew*).

Feature store centraliza:

- **Online serving**: baixa latência (Redis, DynamoDB).
- **Offline training**: batch (Parquet em lake).
- **Garantia de consistência**: mesma transformação no treino e no serve.
- **Point-in-time correctness**: feature "value as of timestamp T" para evitar leak.
- **Reuso**: features compartilhadas entre modelos/equipes.
- **Linhagem**: quem consome o quê.

**Ferramentas**:

- **Feast** (open-source), **Tecton** (commercial), **Vertex AI Feature Store**, **SageMaker Feature Store**, **Databricks Feature Store**, **Hopsworks**.

Não toda org precisa — viável quando múltiplos modelos reutilizam features ou features são caras/complexas.

## Training Pipelines

### Orchestration

- **Airflow**: clássico em batch.
- **Prefect, Dagster**: modernos, data-first.
- **Kubeflow Pipelines**: K8s native para ML.
- **Argo Workflows**.
- **Metaflow** (Netflix): ML-centric Python.
- **SageMaker Pipelines, Vertex AI Pipelines, Azure ML Pipelines**.
- **Flyte**: Kubernetes-native, multi-tenant.
- **Ray + Ray AIR**: distributed compute + ML tasks.

### Compute

- **Spot / preemptible**: treino tolera (checkpointing).
- **Spot com auto-resume**.
- **Distributed training**: data parallel, model parallel (ver `GPU.md`).
- **Frameworks**: PyTorch DDP/FSDP, DeepSpeed, Megatron-LM, Colossal-AI.
- **Ray Train**: abstração multi-framework.

### Hyperparameter Tuning

- **Grid / random search**: baseline.
- **Bayesian optimization**: Optuna, Hyperopt, Ax.
- **Population-based training (PBT)**: tuning + treino simultâneo.
- **SigOpt, Weights & Biases Sweeps, Ray Tune**.

### Checkpointing

Essencial em treino longo. Periódico; resume from checkpoint; evicação antiga.

## Serving

### Padrões de deploy

- **Batch inference**: job que processa cronograma. Bom para latência não-crítica.
- **Online inference / real-time**: REST/gRPC endpoint. Latência p99 importa.
- **Streaming inference**: consome Kafka/Kinesis, produz predictions.
- **Edge inference**: on-device (mobile, IoT).
- **Embedded**: modelo compilado dentro da aplicação.

### Serving frameworks

- **Triton Inference Server** (Nvidia): multi-framework, alta performance.
- **TorchServe, TensorFlow Serving**.
- **vLLM, TGI (Text Generation Inference), SGLang, TensorRT-LLM**: LLM-specific; continuous batching, PagedAttention.
- **BentoML, Cortex, Seldon Core, KServe**.
- **Ray Serve**: flexível, Python-first.

### Performance

- **Batching dinâmico**: agrupa requests → throughput. Latência adicional aceitável?
- **Continuous batching** (LLMs): essencial — libera slots em vez de esperar batch inteiro.
- **Quantização**: INT8/FP8/INT4 para LLMs, tornou-se padrão. Ver `GPU.md`.
- **Pruning, distillation**: modelos menores para inference.
- **Compilação**: TensorRT, ONNX Runtime, TorchCompile, OpenVINO.
- **Speculative decoding**: modelo pequeno propõe tokens; grande valida.

### Deployment strategies

- **Blue/green**: swap atômico.
- **Canary**: %, subindo se saudável.
- **Shadow**: roda em paralelo sem servir; comparar predictions.
- **A/B test**: comparar métricas de negócio real.
- **Multi-armed bandit**: alocação adaptativa entre modelos.

## Monitoring

ML monitoring tem três camadas:

### 1. Sistêmica (mesmo que qualquer serviço)

Latência, throughput, errors, recursos. Ver `OBSERVABILITY.md`.

### 2. ML métricas (accuracy, F1, etc.)

- **Em tempo real quando há ground truth imediato**: fraud detection (ok tive fraude), ad click.
- **Delayed**: churn (meses); re-cálculo sobre feedback.
- **Por cohort / slice**: qualidade não uniforme entre segmentos.

### 3. Data drift

Sem ground truth rápido, proxies:

- **Feature drift**: distribuição de feature de entrada mudou? (KS test, PSI, Wasserstein).
- **Label drift**: distribuição da predição mudou?
- **Concept drift**: relação feature → label mudou (harder to detect).
- **Embedding drift**: centroides/distâncias em espaço de embedding.

**Ferramentas**: Evidently AI, WhyLabs, Arize, Fiddler, Aporia, Neptune, Modelmetry. Prometheus + Grafana para básico.

Alertas específicos ML:

- Drift > threshold.
- Qualidade cai em cohort crítica.
- Volume de predictions anormal.
- Distribuição de output (se softmax muda shape).

### Debugging e explicabilidade

- **SHAP, LIME**: explicação local.
- **Feature importance global**.
- **Counterfactuals** (DiCE).
- **Model cards, dataset datasheets**: documentação estruturada.
- **Integrated Gradients, Captum**: deep learning specific.

## Re-treino

### Gatilhos

- **Cronograma**: diário, semanal, mensal.
- **Drift detected**: re-treino reativo.
- **Performance drop**: em métricas online.
- **New data available**: streaming de novos exemplos.
- **Code change**: mudou pipeline.

### Padrões

- **Stateless**: re-treino do zero sempre.
- **Continual learning**: update incremental (catastrophic forgetting é risco).
- **Online learning**: atualização por exemplo (rara em prod séria).

### Validação antes de promover

Mesmo automatizado:

- Métricas em hold-out atingem limiar?
- Não regrediu em slices-chave?
- Shadow run contra versão atual OK?
- Aprovação humana opcional em modelos de alto impacto.

## Reprodutibilidade

Dura. Componentes:

- **Seeds** (torch.manual_seed, np.random.seed, CUDA determinism).
- **Lock files**: requirements.txt, Pipfile.lock, poetry.lock, conda-lock.
- **Container images** (Docker) pinadas por digest.
- **Data version**.
- **Hardware info**: GPU tipo/driver/CUDA versão podem mudar results.
- **Random operations em CUDA** podem não ser determinísticas sem flags explícitos.
- **Seeds em DataLoader, workers**.

100% reprodutibilidade em training GPU em geral é impossível; <0.1% de variação é alcançável.

## CI/CD para ML

### Testes

- **Unitários** em código de data processing, features, serving.
- **Contract tests** em pipelines.
- **Data validation tests**: Great Expectations, Deequ, TFX DataValidation, Soda — schemas, ranges, freshness.
- **Model validation**: métricas mínimas, performance em slices específicos, fairness.
- **Behavioral tests** (Ribeiro): modelo responde como esperado a invariâncias, directional expectations, minimum function.
- **Integration tests** ponta-a-ponta.
- **Stress/latency tests** em serving.

### Pipeline CI

1. Lint + type-check código.
2. Unit tests.
3. Build image.
4. Data validation on sample.
5. Train em conjunto reduzido (smoke).
6. Deploy staging.
7. Evaluate em test set fresh.
8. Se passa, promove a registry como candidate.
9. Canary em produção.
10. Full rollout.

### Infrastructure as Code

Terraform / CDK / Pulumi para GPU pools, model endpoints, feature store infra, etc.

## Governança

### Model documentation

- **Model card** (Mitchell et al., 2019): intended use, metrics, caveats, ethical considerations.
- **Dataset datasheets** (Gebru et al.): composição, motivação, collection, limitations.
- **Responsible AI** frameworks: Google PAIR, Microsoft RAI, NIST AI RMF.

### Compliance

- **EU AI Act (2024-2026 rollout)**: categorias de risco; high-risk obriga documentação, registro, supervisão humana.
- **NIST AI RMF**: recomendado EUA (não obrigatório federal ainda).
- **ISO/IEC 42001**: gestão de IA.
- **LGPD/GDPR**: decisões automatizadas, right to explanation.
- **Setoriais**: FDA em medical AI; FCRA em credit scoring; SR 11-7 em bancos.

### Auditoria

- **Trilhas** de decisão por prediction: qual modelo, versão, features.
- **Access logs**: quem retreinou, quem aprovou deploy.
- **Bias audits**: métricas de fairness em subgrupos.

## Casos Especiais

### LLMs em produção

Diferente de modelos clássicos:

- **Treino é rarely in-house**; fine-tuning + RAG + prompt engineering.
- **Inference é caro**; batching, quantização, caching críticos.
- **Prompt injection** (ver `AI_SECURITY.md`).
- **Continuous evals**: métricas domínio-específicas, LLM-as-judge, human eval.
- **Prompt versioning**: prompt é código.
- **Observability** específica: token cost, latency, tools called.
- **Guardrails**: input/output filters.

Ferramentas: **LangSmith, LangFuse, Braintrust, Arize Phoenix, Helicone, Weights & Biases Weave**.

### Recommender systems

- **Offline vs online metrics divergem** (offline gain ≠ online lift).
- **Two-tower, sequential, transformer-based** dominantes.
- **Cold start** (novo user, novo item).
- **Diversity, novelty, fairness** vs relevance.
- **Explore/exploit** (bandit) em otimização contínua.

### Computer vision

- **Edge deployment** em mobile/embedded (ONNX, TFLite, Core ML).
- **Versionamento de labeling**.

### NLP clássico

- **Embeddings** como artefato.
- **Drift em language** (novas gírias, tópicos).

## Anti-Padrões

1. **Notebook em produção**: colabora pesquisa, nunca deploy.
2. **"Data is the new code, code is the new config"** levado a sério demais — código ainda é código; review e test.
3. **Training/serving skew ignorado**: feature diferente em prod → performance cai misteriosamente.
4. **Sem monitoring pós-deploy**: "modelo funciona" é ponto no tempo.
5. **Re-treino automatizado sem validação**: pode promover modelo pior.
6. **Um único modelo servindo mundo**: sem canary, sem rollback, sem shadow.
7. **Offline metrics como proxy suficiente**: real-world lift é o juiz.
8. **Build your own MLOps platform**: cada org reinventando a roda. Use ferramentas maduras.
9. **Data quality ignorada**: garbage in, garbage out.

## Ferramentas — Quick Reference

| Função | Ferramentas |
|---|---|
| Tracking | MLflow, W&B, Neptune, Comet |
| Orchestration | Airflow, Prefect, Dagster, Kubeflow, Metaflow, Flyte |
| Feature store | Feast, Tecton, Hopsworks |
| Serving | Triton, TorchServe, vLLM, TGI, BentoML, KServe, Ray Serve |
| Monitoring | Evidently, WhyLabs, Arize, Fiddler, Aporia |
| Data versioning | DVC, LakeFS, Pachyderm |
| Data validation | Great Expectations, Soda, TFX DV, Deequ |
| Labeling | Labelbox, Scale AI, CVAT, Snorkel, Argilla |
| Explainability | SHAP, LIME, Captum, InterpretML |
| LLM ops | LangSmith, Langfuse, Braintrust, Phoenix, Helicone |
| End-to-end | Vertex AI, SageMaker, Azure ML, Databricks |

## Recursos

- **"Designing Machine Learning Systems"** — Chip Huyen.
- **"Machine Learning Engineering"** — Andriy Burkov.
- **"Building Machine Learning Powered Applications"** — Emmanuel Ameisen.
- **"Reliable Machine Learning"** — Chen, Mitchell, Mouly.
- **"ML Ops: Operationalizing Machine Learning"** — Mark Treveil.
- **Google ML Engineering Best Practices (Martin Zinkevich)**.
- **"Machine Learning: The High Interest Credit Card of Technical Debt"** — Sculley et al. (NeurIPS 2015). Clássico.
- **Chip Huyen blog, mlops.community**.

## Princípios

1. **Versione tudo**: código, dados, features, modelo, config, ambiente.
2. **Teste tudo testável**, inclusive data e behavior.
3. **Reproduzibilidade é investimento** — paga em debugging futuro.
4. **Monitoramento prós começa no dia 1**, não quando começa a quebrar.
5. **Prevenção de training-serving skew** via feature store ou pipeline compartilhado.
6. **Deploy gradual** sempre: shadow → canary → full.
7. **Drift é inevitável**. Pipeline de re-treino parte do design.
8. **ML é processo, não projeto**. Modelo não "termina"; vive.
9. **Simplicidade primeiro**. Model simples + pipeline robusto > modelo complexo + pipeline frágil.
10. **Compliance cresce**. EU AI Act, setoriais; documentar desde cedo.
