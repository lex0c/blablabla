# ML Frameworks e Ferramentas

> **Snapshot do ecossistema: 2026-04.** Conteúdo deste arquivo envelhece rápido — bibliotecas, modelos nomeados, APIs proprietárias, "estado da arte". Espere reler e atualizar a cada 12-18 meses.
>
> Para **conceitos atemporais** (optimizers, schedules, regularização, paralelismo, precisão numérica, taxonomia de arquiteturas) ver `ML_FOUNDATIONS.md`. Para fundamentos de ML/DL (neurônios, backprop, transformers) ver `AI.md`. Para arquitetura de sistemas de IA (RAG, agentes LLM) ver `AI_SYSTEMS_DESIGN.md`. Para RL ver `AGENTS.md`.

Este arquivo cobre **o que está em uso agora**: decisão de stack, bibliotecas core, serving, bibliotecas de training distribuído, ferramentas periféricas, modelos nomeados.

## Como escolher — decisão em uma página

A escolha de stack depende de **(1) o que você está treinando**, **(2) quem vai operar**, **(3) onde vai rodar**.

| Problema | Default sensato |
|---|---|
| Tabular estruturado (< 10M linhas) | **scikit-learn** + **XGBoost/LightGBM** |
| Tabular grande / time-series | **LightGBM** ou **CatBoost** |
| NLP com modelo pré-treinado | **HuggingFace Transformers** sobre **PyTorch** |
| Visão pré-treinada | **timm** sobre **PyTorch** |
| Pesquisa deep learning / paper novo | **PyTorch** (de longe) |
| Pesquisa em escala Google/DeepMind | **JAX + Flax** |
| Produção legacy em empresa grande | **TensorFlow / Keras** |
| Mobile / edge | **TFLite**, **CoreML**, **ONNX Runtime Mobile** |
| Treinar LLM do zero em cluster | **PyTorch + FSDP/DeepSpeed** ou **Megatron-LM** |
| Fine-tune LLM com LoRA | **HF transformers + peft + trl** |
| Serving LLM em produção | **vLLM**, **TGI**, **SGLang** |
| Inferência local no laptop | **llama.cpp / Ollama** |
| Deploy CPU genérico em microserviço | **ONNX Runtime** |
| Robótica / RL contínuo | **PyTorch + SB3 / CleanRL** |

Regra: **pesquisa = PyTorch**, **produção antiga = TF**, **produção nova = PyTorch ou JAX**, **tabular = boosting tree**, **LLM = HuggingFace/vLLM**. Keras 3 muda a equação porque virou multi-backend (TF/JAX/PyTorch).

## Stack em camadas

```
┌─────────────────────────────────────────────────────────────┐
│  APPLICATION           LangGraph, LlamaIndex, Gradio, Streamlit │
├─────────────────────────────────────────────────────────────┤
│  HIGH-LEVEL            HuggingFace, PyTorch Lightning, fast.ai, │
│                        Keras 3, Flax, Equinox                   │
├─────────────────────────────────────────────────────────────┤
│  FRAMEWORK             PyTorch, TensorFlow, JAX                 │
├─────────────────────────────────────────────────────────────┤
│  TENSOR / ARRAY        torch.Tensor, tf.Tensor, jax.numpy,      │
│                        NumPy, CuPy                              │
├─────────────────────────────────────────────────────────────┤
│  COMPILER / KERNEL     cuDNN, cuBLAS, MIOpen, Metal, XLA,       │
│                        Triton, TensorRT, MLIR, torch.compile    │
├─────────────────────────────────────────────────────────────┤
│  HARDWARE              CUDA GPU, ROCm GPU, TPU, Neural Engine,  │
│                        Gaudi, CPU AVX/AMX                       │
└─────────────────────────────────────────────────────────────┘
```

A maioria das conversas sobre "qual framework" acontece nas duas camadas do meio (framework + tensor). As camadas de baixo são partilhadas — todo mundo usa cuDNN por baixo — e as de cima são intercambiáveis.

## PyTorch

Dominante em pesquisa desde ~2018 e hoje também em produção. API imperativa (eager), autograd dinâmico, extensível.

### Core

- **Tensor** — N-dim array com autograd opcional. `torch.tensor([1,2,3], device='cuda', requires_grad=True)`.
- **nn.Module** — classe base de todos os modelos. `forward()` define computação; parâmetros são rastreados automaticamente.
- **autograd** — grafo dinâmico, construído a cada forward. `.backward()` propaga gradientes. `with torch.no_grad():` desliga para inference.
- **Optimizer** — `SGD`, `Adam`, `AdamW`, `Adafactor`, `Lion`. `optimizer.zero_grad(); loss.backward(); optimizer.step()`.
- **DataLoader** — wrapping sobre `Dataset`, cuida de batching, shuffling, multi-worker, pinned memory.

### torch.compile (2.0+)

Compilador nativo via TorchDynamo + TorchInductor. `model = torch.compile(model)` converte eager → graph → Triton kernels. Speedup típico 1.3-2× em treino, mais em inference. Gotchas: recompila em shape change (usar `dynamic=True`), alguns ops ainda caem fora do graph ("graph break").

### Distributed

- **DataParallel** — deprecated, só histórico.
- **DistributedDataParallel (DDP)** — padrão para multi-GPU. Um processo por GPU, all-reduce nos gradientes. Escala bem até ~64 GPUs.
- **FSDP** (Fully Sharded Data Parallel) — shard de parâmetros, gradientes, optimizer states entre GPUs. Substituto oficial do ZeRO-3 do DeepSpeed. Necessário para modelos que não cabem em uma GPU.
- **Tensor Parallel / Pipeline Parallel** — via `torch.distributed.pipelining` ou bibliotecas externas (Megatron, DeepSpeed).

### Quantização e deployment

- **Dynamic quantization** — pesos INT8, ativações FP32, decidido em runtime. Para LSTMs/Linear.
- **Static quantization** — calibra com dataset, converte tudo para INT8.
- **QAT** (Quantization Aware Training) — simula quantização durante treino. Melhor qualidade, mais complexo.
- **torch.export** + **ExecuTorch** — stack novo para edge/mobile, substitui TorchScript em novos projetos.
- **TorchScript** (`torch.jit.script` / `torch.jit.trace`) — legado mas ainda funcional, serialização compilada.

### Mudanças 1.x → 2.x (o que importa)

- `torch.compile` como ponto focal.
- `functorch` foi absorvido (`torch.func`): `vmap`, `grad`, `jvp` estilo JAX.
- `torch.export` substituindo TorchScript.
- FSDP2 (reescrita) mais limpo que FSDP1.
- `nn.attention.flex_attention` — API oficial para variantes de atenção com compilação eficiente.

### Ecossistema PyTorch

- **Lightning** — wrap que remove boilerplate (training loop, logging, checkpointing, multi-GPU). Opinionated mas produtivo.
- **timm** — biblioteca de imagem (ResNets, ViT, ConvNeXt, EfficientNet, …). `timm.create_model('resnet50', pretrained=True)`.
- **torchvision / torchaudio / torchtext** — oficiais, mantêm datasets/transforms/modelos clássicos.
- **PyTorch Geometric (PyG)** e **DGL** — GNN / graphs.
- **Kornia** — computer vision diferenciável (homografias, augmentation).

## TensorFlow e Keras

Dominante em pesquisa até ~2018, em produção em muitas empresas. Perdeu mindshare mas sobrevive em base instalada enorme, Android/mobile, e TPU-first workflows.

### TF2 vs TF1

- **TF1** (pré-2019) — grafo estático declarativo (`tf.Session`, placeholders, feed_dict). Horrível de debugar.
- **TF2** — eager execution por default, `tf.function` para compilar hot paths, Keras como API oficial. API bem mais parecida com PyTorch.

### Keras

Nascido como abstração alto-nível sobre Theano/TF/MXNet (~2015), virou parte do TF em 2.0, e em **Keras 3** (2023) voltou a ser multi-backend: pode rodar sobre **TensorFlow, JAX ou PyTorch**.

- **`keras.Sequential`** — stack linear de layers, bom para protótipos.
- **Functional API** — `inputs → layers → outputs`, grafos arbitrários.
- **Subclassing `keras.Model`** — para lógicas custom, estilo PyTorch.
- **`model.fit()`** — training loop de alto nível. Callbacks para early stopping, tensorboard, checkpointing.

Keras 3 é relevante porque **você escreve o modelo uma vez, roda em qualquer backend**. Útil para bibliotecas que querem portabilidade.

### tf.data

Pipeline de dados do TF. Baseado em dataflow graph (`.map`, `.batch`, `.prefetch`, `.interleave`). Muito eficiente em grandes volumes, especialmente com TFRecord. Padrão de ouro para TPU training.

### Serialização

- **SavedModel** — formato canônico (graph + weights + signatures). Usado em TF Serving e TFLite.
- **HDF5 `.h5`** — formato legado do Keras, só pesos + arquitetura JSON.
- **Keras `.keras`** (v3) — novo formato zip, multi-backend.

### Deployment

- **TF Serving** — servidor gRPC/REST para SavedModels. Canary deploys, versionamento.
- **TFLite** — runtime para mobile/embedded. Quantização INT8, float16, GPU delegate, NNAPI (Android).
- **TensorFlow.js** — treino e inference em browser via WebGL/WebGPU.
- **Coral / Edge TPU** — hardware dedicado para TFLite quantizado.

### Por que perdeu mindshare em pesquisa

API mudou demais (TF1 → TF2), ficou atrás em ergonomia, Google internamente migrou para JAX, papers deixaram de soltar código em TF. Keras 3 é a aposta de relevância — se você quer rodar em TPU sem escrever JAX, ainda é a melhor opção.

## JAX

Biblioteca do Google Research (2018) para **transformações de funções numéricas**. Escolha default em DeepMind, Google Research, Anthropic (historicamente).

### Mentalidade

JAX é **funcional**: funções puras, sem estado escondido. Em vez de `model.train()` que muta, você passa params explicitamente. Estado vive em **pytrees** (estruturas aninhadas de arrays).

### Transformações core

- **`jit`** — compila a função com XLA. Primeira chamada é lenta (trace), depois quase-nativo.
- **`grad`** — diferenciação automática funcional. `grad(loss_fn)(params, x, y)` retorna gradientes no mesmo shape de `params`.
- **`vmap`** — vetoriza automaticamente. `vmap(f)` aplica `f` em batch sem loop. Mais natural que batching manual.
- **`pmap`** — paraleliza entre devices (GPUs/TPUs). SPMD.
- **`scan`** — loops compilados (equivalente a `for` mas diferenciável).
- **`shard_map` / `jax.Array` sharding** — modelo mais novo de paralelismo, substitui `pmap` em novos projetos.

### Pytrees

Estruturas arbitrariamente aninhadas (dict, list, namedtuple) de arrays. JAX sabe navegar automaticamente. Permite escrever código genérico.

### Libs sobre JAX

- **Flax** — framework neural estilo PyTorch mas funcional. `nn.Module` com `init` + `apply` separados. De facto padrão.
- **Equinox** — alternativa "PyTorch-like" em JAX. Models são pytrees com parâmetros + tudo.
- **Haiku** — antigo framework da DeepMind, ainda usado em código legado.
- **Optax** — optimizers composáveis (chain de transformações).
- **chex** — asserts e utilities para testar código JAX.
- **rlax, dm-acme** — RL.
- **Brax, MJX** — física diferenciável em GPU/TPU.

### Prós e contras

Prós: performance excelente em TPU, transformações elegantes, perfeito para research de otimizador/arquitetura, escala via sharding nativo.

Contras: curva de aprendizado mais íngreme, debug pior (erros dentro de `jit` são opacos), menos modelos pré-treinados, immutability exige mudar estilo de programação. Integrar `jax` com bibliotecas Python normais (que esperam mutação) dá trabalho.

## Scikit-learn

A biblioteca de ML clássico por excelência. Não faz deep learning (intencional), mas domina tudo que não é.

### O que oferece

- **Classificação/regressão**: `LogisticRegression`, `RandomForest`, `SVM`, `GradientBoosting`, `KNN`, `NaiveBayes`.
- **Clustering**: `KMeans`, `DBSCAN`, `HDBSCAN`, `GaussianMixture`, `AgglomerativeClustering`.
- **Dimensionality reduction**: `PCA`, `t-SNE`, `UMAP` (via `umap-learn`), `TruncatedSVD`.
- **Model selection**: `train_test_split`, `cross_val_score`, `GridSearchCV`, `RandomizedSearchCV`, `HalvingGridSearchCV`.
- **Preprocessing**: `StandardScaler`, `OneHotEncoder`, `OrdinalEncoder`, `PolynomialFeatures`, `PowerTransformer`.
- **Métricas**: tudo que existe — classification, regression, ranking, cluster.

### API uniforme

Todo estimador tem `.fit(X, y)` e `.predict(X)` (ou `.transform(X)`). Isso é o que permite **pipelines**:

```python
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.ensemble import RandomForestClassifier

pre = ColumnTransformer([
    ("num", StandardScaler(), num_cols),
    ("cat", OneHotEncoder(handle_unknown="ignore"), cat_cols),
])
pipe = Pipeline([("pre", pre), ("clf", RandomForestClassifier(n_estimators=300))])
pipe.fit(X_train, y_train)
pipe.score(X_test, y_test)
```

Pipeline vira artefato único, serializável com `joblib.dump`. Não há razão de usar sklearn sem Pipeline em código sério — evita leakage de preprocessing.

### Complementa deep learning

Mesmo em projeto que treina rede neural, sklearn aparece para: split, cross-validation, métricas, baseline (RandomForest antes de treinar Transformer), feature selection, calibration (`CalibratedClassifierCV`).

## Gradient Boosting (a família que bate deep learning em tabular)

Para dados tabulares estruturados, gradient boosting trees **ainda bate redes neurais** em qualidade e treina ordens de magnitude mais rápido. Três implementações dominam:

### XGBoost

Mais antigo (2014), mais estável, ecossistema enorme. Suporta GPU. Histograma por default desde 2.0. Feature importance built-in.

### LightGBM (Microsoft)

Mais rápido que XGBoost na maioria dos benchmarks. **Leaf-wise growth** (vs level-wise), GOSS sampling, exclusive feature bundling. Melhor em datasets com muitas categorias cardinais.

### CatBoost (Yandex)

Melhor tratamento **automático de categóricas** via ordered target encoding (sem vazar futuro). **Symmetric trees** (oblivious) aceleram inference. Menos tuning necessário — boa escolha se você não quer ficar ajustando.

### Quando preferir deep learning em tabular

Quando há **alta cardinalidade categorial com estrutura semântica** (texto, sequência), **dados multimodais** (tabular + imagem + texto), ou **datasets enormes com representation learning útil**. Em 95% dos casos restantes: boosting tree.

### Tuning prático

Principais hiper:
- `learning_rate` (0.01-0.1) e `n_estimators` — jogue `lr=0.05`, `n_estimators=1000`, early stopping.
- `max_depth` (3-10) / `num_leaves` (31-255).
- `subsample`, `colsample_bytree` (0.7-1.0) para regularização.
- `min_child_weight` / `min_data_in_leaf`.
- `reg_alpha` (L1), `reg_lambda` (L2).

Use **Optuna** com `TPESampler` para tuning — 50-100 trials resolvem.

## HuggingFace — o stack NLP/multimodal

Coleção de bibliotecas e hub de modelos. Em NLP, é praticamente mandatório. Expandiu para visão, áudio, multimodal.

### `transformers`

Biblioteca central. Modelo = arquitetura + pesos + tokenizer + config.

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

tok = AutoTokenizer.from_pretrained("meta-llama/Llama-3.1-8B")
model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3.1-8B",
                                              torch_dtype="bfloat16",
                                              device_map="auto")
ids = tok("Once upon a time", return_tensors="pt").to(model.device)
out = model.generate(**ids, max_new_tokens=50)
print(tok.decode(out[0]))
```

APIs principais:
- `AutoModelFor*` — task-specific heads (CausalLM, SequenceClassification, TokenClassification, QuestionAnswering, ImageClassification, AudioClassification, ...).
- `pipeline("sentiment-analysis")` — conveniência para inference rápida.
- `Trainer` — training loop de alto nível com logging, checkpointing, FSDP/DeepSpeed integrados.
- **PreTrainedTokenizer** — tokenização rápida em Rust (`tokenizers`).

### `datasets`

Carrega/processa datasets com lazy loading via Apache Arrow. `load_dataset("imdb")`. Map/filter/shuffle com cache em disco. Streaming para datasets que não cabem em RAM.

### `tokenizers`

Implementações em Rust: BPE, WordPiece, Unigram, SentencePiece. Treino de tokenizer custom em minutos para milhões de docs.

### `accelerate`

Abstrai multi-GPU/TPU/mixed-precision. Você escreve PyTorch "normal", `accelerator.prepare(model, optimizer, dataloader)` distribui.

### `peft` (Parameter-Efficient Fine-Tuning)

LoRA, QLoRA, DoRA, prefix-tuning, IA³. Treina 0.1-1% dos parâmetros, produz adapter de MBs em vez de GBs. Essencial para fine-tuning local de LLM.

### `trl` (Transformer Reinforcement Learning)

SFT, DPO, GRPO, PPO, ORPO para alinhamento de LLMs. Base de muitos pipelines RLHF.

### `diffusers`

Stable Diffusion, Flux, modelos de vídeo/áudio de difusão. Composa de schedulers, UNets, VAEs, text encoders.

### Hub

`huggingface.co` hospeda 1M+ modelos, datasets, spaces (apps Gradio). API `huggingface_hub` para download/upload programático. Model cards padronizados.

## Serving e inference

### LLMs

- **vLLM** — padrão atual para serving LLM. **PagedAttention** (memória KV em páginas estilo SO), continuous batching, prefix caching, speculative decoding. OpenAI-compatible API. Se você está servindo LLM em produção, provavelmente é isso.
- **SGLang** — alternativa a vLLM com ênfase em programas estruturados (constrained decoding, workflows). RadixAttention cache. Ganhando tração.
- **TGI** (Text Generation Inference, HuggingFace) — servidor em Rust, bom ecossistema. Perdeu terreno para vLLM em performance pura.
- **TensorRT-LLM** (NVIDIA) — mais rápido em hardware NVIDIA quando bem compilado, mais dor de cabeça. Engine compilado por (modelo, GPU, batch, seq length).
- **llama.cpp** — inference em CPU/GPU via GGUF quantizado. Padrão para rodar local.
- **Ollama** — wrapper amigável sobre llama.cpp. `ollama run llama3.2`. Ideal para dev local.
- **MLX** (Apple) — serving em Apple Silicon, aproveita Neural Engine + unified memory.

### Modelos gerais

- **ONNX Runtime** — runtime portátil multi-plataforma. Converte de PyTorch/TF via `torch.onnx.export`. Quantização INT8, providers (CUDA, TensorRT, DirectML, CoreML, OpenVINO).
- **NVIDIA Triton Inference Server** — servidor unificado para ONNX/TensorRT/PyTorch/TF. Dynamic batching, model ensemble, A/B testing. Padrão em grande escala.
- **TorchServe** — serving oficial do PyTorch, menos popular que Triton.
- **BentoML** — packaging + serving em Python, mais dev-friendly.
- **Ray Serve** — serving distribuído dentro do Ray, bom para pipelines multi-modelo.

### Mobile / edge

- **TFLite** — Android first-class, também iOS/embedded.
- **CoreML** — Apple devices. `coremltools` converte de PyTorch.
- **ExecuTorch** — stack PyTorch novo para edge.
- **ONNX Runtime Mobile** — portátil.

## Training em escala — bibliotecas

Para conceitos (DP/TP/PP, ZeRO, mixed precision, gradient checkpointing, flash attention, MoE), ver `ML_FOUNDATIONS.md`. Aqui só as **implementações** atuais.

- **FSDP** (PyTorch nativo) — padrão oficial para ZeRO-3-style sharding. **FSDP2** é a reescrita atual, API mais limpa.
- **DeepSpeed** (Microsoft) — ZeRO 1/2/3, offloading CPU/NVMe, fused optimizer, pipeline parallel. Integra com HF Trainer e `accelerate`.
- **Megatron-LM** / **Megatron-Core** (NVIDIA) — TP+PP estado da arte para LLMs grandes. Base de muitos treinos de fronteira.
- **NVIDIA NeMo** — framework end-to-end (treino + inference) sobre Megatron-Core + PyTorch Lightning.
- **torch.distributed.pipelining** — pipeline parallel nativo no PyTorch.
- **Ray Train** — scheduling distribuído em clusters via Ray.
- **MosaicML Composer / LLM Foundry** — abstração sobre PyTorch com receitas otimizadas.
- **Levanter, t5x, MaxText** — stacks JAX/TPU para treino grande.
- **Transformer Engine** (NVIDIA) — FP8 training em H100+.
- **Accelerate** (HuggingFace) — abstração fina sobre DDP/FSDP/DeepSpeed para não reescrever loops.

Na prática, 2026: **FSDP2 + Accelerate** para treinos até ~100B; **Megatron + DeepSpeed** para além disso; **JAX + MaxText** em TPU pod.

## Receitas prontas (pontos de partida atuais)

Valores concretos variam com modelo/dataset, mas esses defaults economizam tempo. Conceitos dos optimizers/schedules em `ML_FOUNDATIONS.md`.

**Fine-tune transformer (BERT-like)**:
- Optimizer: AdamW, `betas=(0.9, 0.999)`, `eps=1e-8`, `weight_decay=0.01` (sem decay em bias/LN).
- LR: `2e-5` (base), `5e-5` (small), `1e-5` (large).
- Schedule: warmup 10% + linear decay. BF16. Grad clip 1.0.
- Epochs: 2-4.

**Train LLM from scratch (GPT-like)**:
- AdamW, `betas=(0.9, 0.95)`, `weight_decay=0.1`.
- LR: `6e-4` (125M) até `3e-4` (6B+); scaling inverso com tamanho.
- Schedule: warmup 2000 steps + cosine até 10% do max.
- BF16, grad clip 1.0. Batch tokens 0.5M-4M por step via accumulation.

**Visão from scratch (ResNet ImageNet)**:
- SGD+Nesterov, momentum 0.9, `weight_decay=1e-4`.
- LR: 0.1 (batch 256), linear scale.
- Schedule: step ×0.1 a cada 30 epochs, 90-120 epochs total. Ou cosine.
- Augmentation forte: RandAugment, CutMix, MixUp.

**ViT fine-tune**:
- AdamW, `lr=3e-4`, `weight_decay=0.05`.
- Warmup + cosine, 10-100 epochs.
- DropPath 0.1-0.2. Label smoothing 0.1.

**LoRA fine-tuning (LLM)**:
- AdamW, `lr=2e-4` a `3e-4`.
- Rank `r=8-64`, alpha tipicamente `2·r`.
- Target modules: `q_proj, k_proj, v_proj, o_proj` no mínimo; adicione `gate/up/down_proj` para mais capacidade.
- 1-3 epochs.

## Experiment tracking

- **Weights & Biases (wandb)** — padrão de facto hoje. Logging de métricas, hyperparams, artifacts, imagens, plots, system metrics. Free tier generoso para uso pessoal/OSS.
- **MLflow** — open-source, auto-hospedável. Tracking + model registry + deployment. Escolha em ambientes enterprise.
- **TensorBoard** — oficial TF, ainda funciona com PyTorch (`torch.utils.tensorboard`). Bom para curvas e gráfico de grafo; inferior em comparação/colab.
- **Aim** — alternativa open-source, rápido, boa para muitos experimentos.
- **Neptune**, **Comet** — comerciais, features similares a W&B.
- **DVC Studio** — se você já usa DVC, integração natural.
- **ClearML** — open-source com features enterprise.

Dicas:
- Logar **sempre**: commit hash, hyperparams completos, versão de código, tempo de wall clock, uso de GPU/memória.
- Usar **runs group / tag** para comparar sweeps.
- Salvar **checkpoint + métricas** em mesmo run_id facilita reprodução.

## Dados e pipelines

### DataFrames

- **pandas** — default histórico. API imensa, funciona bem até ~1-10GB. Single-threaded, memory-hungry.
- **Polars** — DataFrame em Rust, lazy evaluation, multi-threaded. Muito mais rápido que pandas em praticamente tudo. API limpa. Migração vale a pena em pipelines novos.
- **DuckDB** — SQL embarcado sobre Parquet/Arrow. Excelente para agregação analítica em arquivos grandes sem subir servidor.
- **Dask** — pandas distribuído, quando você **realmente** precisa de cluster.
- **Ray Data** — pipeline data paralelo em Ray, integra com Ray Train.
- **Modin** — pandas drop-in distribuído (experimente antes de migrar código).

### Formatos

- **Parquet** — colunar comprimido, padrão em analytics. Leitura seletiva de colunas.
- **Arrow** — in-memory zero-copy entre processos/linguagens. Base de Polars, DuckDB, HF datasets.
- **Feather** — Arrow em disco, rápido de ler/escrever.
- **TFRecord** — padrão TF, protobuf.
- **WebDataset** — tar shards de arquivos, streaming ideal para treino.
- **LMDB** — key-value memory-mapped, comum em visão.
- **HDF5** — científico, hierarchical, muitos hooks.
- **Zarr** — arrays chunked em cloud storage.

### Versionamento de dados

- **DVC** — git-like para datasets e modelos. Pointers em git, dados em S3/GCS/local.
- **Git LFS** — large files em git, mais simples mas limitado.
- **LakeFS** — versionamento de data lake estilo git.
- **Quilt** — packages de dados versionados.

### Augmentation

- **Albumentations** — imagem, rápido, pipeline flexível. Padrão em visão.
- **torchvision.transforms.v2** — oficial, cada vez melhor.
- **Kornia** — augmentation diferenciável em GPU.
- **audiomentations** / **torchaudio.transforms** — áudio.
- **nlpaug** — NLP (sinônimos, back-translation).

## Especializações por domínio

### Visão

- **timm** — ResNet, ViT, ConvNeXt, Swin, DINO, MAE, tudo pré-treinado. Um `.create_model()` único.
- **torchvision** — oficial, cobre classics (ResNet, Faster-RCNN, Mask-RCNN).
- **detectron2** (FAIR) — detecção/segmentação state-of-the-art.
- **MMDetection / MMSegmentation / MMAction** (OpenMMLab) — suites enormes, chinesas, muito usadas.
- **Ultralytics YOLO** — YOLOv8/v11, API ultra simples, boa para deploy.
- **Segment Anything (SAM / SAM2)** — segmentação promptável.
- **CLIP / OpenCLIP** — embedding vision-language.
- **Grounding DINO, Florence-2, OWL-ViT** — detecção open-vocabulary.

### NLP

- **HuggingFace Transformers** (coberto acima) — dominante.
- **spaCy** — NLP industrial (NER, parsing, tokenização) rápido em CPU. Boa para pipelines sem LLM.
- **NLTK** — legado, útil em educação/pesquisa linguística.
- **sentence-transformers** — embeddings de sentença, retrieval.
- **FastText** — embeddings clássicos, CPU-friendly.
- **Gensim** — Word2Vec, LDA, TF-IDF.

### Áudio / fala

- **torchaudio / tensorflow-io-audio** — I/O e transforms.
- **librosa** — análise tradicional (MFCC, STFT, beat tracking).
- **OpenAI Whisper** / **faster-whisper** — ASR estado da arte.
- **pyannote.audio** — diarização.
- **NeMo** (NVIDIA) — ASR/TTS/NLP toolkit industrial.
- **Speechbrain** — toolkit pesquisa áudio.
- **demucs** — separação de fontes musicais.

### Series temporais

- **statsmodels** — ARIMA, SARIMAX, state-space.
- **Prophet** (Facebook) — forecasting rápido, baseline sensato.
- **sktime** — scikit-learn API para séries.
- **Darts** — abstração alta para forecasting, ensembles fáceis.
- **NeuralForecast / NeuralProphet** — modelos neurais de séries.
- **tsfresh** — feature extraction automática.

### Grafos

- **PyTorch Geometric (PyG)** — GNN em PyTorch, padrão pesquisa.
- **DGL** — alternativa, suporta PyTorch/TF/MXNet.
- **NetworkX** — grafo clássico em Python puro (não GNN).
- **igraph** — C core, muito mais rápido para grafos grandes.

### RL

Ver `AGENTS.md` para detalhes. Stack: **Gymnasium** (envs), **Stable-Baselines3** (algoritmos prontos), **CleanRL** (single-file didático), **RLlib** (escala), **TorchRL** (moderno).

### Difusão / generativo

- **diffusers** (HF) — Stable Diffusion, Flux, Cogvideo, etc.
- **ComfyUI** — grafo visual para pipelines generativos (muito usado em creative).
- **InvokeAI, Automatic1111** — UIs para SD.

### Otimização / AutoML

- **Optuna** — hyperparameter tuning, TPE, pruning. Padrão moderno.
- **Ray Tune** — escala distribuída.
- **Hyperopt** — antigo, Optuna substituiu.
- **AutoGluon** (Amazon) — AutoML tabular/texto/imagem batendo Kaggle sem tunar.
- **H2O AutoML, FLAML** — alternativas AutoML.
- **auto-sklearn** — AutoML sobre sklearn.

### Simulação / física

- **MuJoCo** (DeepMind open-source desde 2022) — contato e articulação.
- **PyBullet** — física geral, robótica.
- **Isaac Sim / Isaac Lab** (NVIDIA) — robótica em GPU.
- **Brax, MJX** — física diferenciável em JAX.

## Multimodal — VLMs, modelos any-to-any e convergência

"Multimodal" = modelo que consome/produz mais de uma modalidade (texto, imagem, áudio, vídeo, 3D, ação, tabular, sinais de sensor). Em 2025-2026, a fronteira de IA é dominantemente multimodal — GPT-4o, Claude, Gemini, Llama 3.2 Vision, Qwen2.5-VL, Pixtral, Molmo, Phi-3.5-Vision.

### Arquiteturas principais

**Dual encoder contrastivo** (CLIP-like):
- Dois encoders (texto + imagem/áudio) projetam em espaço compartilhado.
- Treinado com InfoNCE: imagem e legenda casadas são "positivas", tudo mais é "negativo".
- **Sem gerar texto** — só compara. Usado para retrieval, zero-shot classification, filtering de datasets.
- Exemplos: **CLIP** (OpenAI, 2021), **OpenCLIP** (LAION), **SigLIP/SigLIP 2** (Google, sigmoid loss, mais estável), **EVA-CLIP**, **ALIGN** (Google), **LiT** (image frozen + text tuned).
- Áudio: **CLAP** (audio-text), **AudioCLIP**.
- Vídeo: **VideoCLIP**, **InternVideo**.

**VLM / MLLM (Multimodal LLM)** — arquitetura dominante hoje:
```
  image → vision encoder → projector/connector → LLM ← text tokens
                                                   ↓
                                                text output
```
Três partes:
1. **Vision encoder** — geralmente CLIP ViT-L/14, SigLIP, DINOv2, ou custom. Extrai features visuais.
2. **Connector** — alinha espaço visual → espaço do LLM. Tipos:
   - **MLP projector** (LLaVA) — 2 camadas lineares. Simples, funciona surpreendentemente bem.
   - **Q-Former** (BLIP-2) — transformer com queries aprendíveis que "extraem" N tokens visuais.
   - **Perceiver Resampler** (Flamingo) — cross-attention com latents fixos.
   - **Cross-attention layers** injetados no LLM (Flamingo, Chameleon).
3. **LLM** — decoder (Llama, Mistral, Qwen, Phi). Recebe visual tokens + text tokens, gera texto.

Famílias:
- **LLaVA / LLaVA-Next / LLaVA-OneVision** — MLP projector, padrão open-source. Receita replicável em hardware modesto.
- **BLIP-2 / InstructBLIP** — Q-Former, treino em 2 estágios.
- **Flamingo** (DeepMind) — cross-attention gated, interleaved image-text.
- **MiniGPT-4 / MiniGPT-v2** — simplificações de LLaVA.
- **CogVLM / CogVLM2** — "visual expert" (dupla de pesos atenção/FFN para tokens visuais).
- **InternVL / InternVL 2/2.5** — série grande, competitiva com modelos fechados.
- **Qwen-VL / Qwen2-VL / Qwen2.5-VL** — resolução dinâmica, vídeo, embarcamento multi-imagem. Referência open em 2025.
- **Pixtral** (Mistral) — vision + Mistral Large.
- **Molmo** (AI2) — dataset inteiramente aberto, pointing/localization.
- **Phi-3.5-Vision / Phi-4-Multimodal** (Microsoft) — pequeno e bom, edge-friendly.
- **Llama 3.2 Vision** (11B/90B) — cross-attention, Meta.

**Any-to-any (nativamente multimodal)**:
- **Chameleon** (Meta) — tokens discretos para imagem e texto no mesmo vocabulário, treino unificado.
- **GPT-4o / Gemini 1.5/2.0** — nativo em texto/imagem/áudio/vídeo, entrada e saída (proprietário).
- **Fuyu** (Adept) — sem vision encoder separado, patches de imagem direto no decoder.
- **Unified-IO 2, 4M** — qualquer modalidade in/out via token unificado.

**Difusão multimodal** (geração):
- **Text-to-image**: Stable Diffusion 3, Flux, DALL-E 3, Imagen 3, Midjourney.
- **Text-to-video**: Sora, Veo 2, Runway Gen-3, Kling, Mochi, CogVideoX.
- **Text-to-audio/music**: Stable Audio, MusicGen, AudioLDM, Suno.
- **Text-to-3D**: DreamFusion, Zero123, TRELLIS, Meshy.

**Speech-language**:
- **Whisper / faster-whisper** — ASR.
- **Qwen2-Audio, Qwen2.5-Omni** — áudio in + texto in/out.
- **SeamlessM4T** (Meta) — speech/text translation.
- **GPT-4o Realtime / Voice Mode** — voice-in voice-out sem passar por texto.
- **Moshi** (Kyutai) — full-duplex voice, open.

**Vision-Language-Action (VLA) — robótica**:
- **RT-1 / RT-2** (Google) — VLM que emite ações de robô como tokens.
- **OpenVLA** — aberto, 7B, treinado em Open X-Embodiment.
- **π0** (Physical Intelligence) — flow matching para ações, generalização cross-embodiment.
- **Octo, RDT-1B** — alternativas.

### Treino de um VLM — receita padrão

1. **Stage 1 — Alignment / Pretraining**: só treina o **projector** (ou Q-Former), vision encoder e LLM congelados. Dataset de ~500k-5M pares imagem-caption (LAION, CC3M, CC12M filtrado, DataComp). Curto (~1 epoch), LR alto. Ensina o projector a mapear features visuais para o espaço do LLM.
2. **Stage 2 — Visual Instruction Tuning**: treina **projector + LLM** (LoRA ou full), vision encoder ainda congelado (ou descongelado em late stage). Dataset de **instruções visuais** (LLaVA-Instruct, ShareGPT4V, Cambrian, Cauldron). Aprende a conversar sobre imagens, seguir instruções, raciocinar.
3. **Stage 3 — Task / Domain Fine-tune** (opcional): especialização em OCR, doc understanding, grounding, médico, etc.

Frequentemente com **mixed resolution / dynamic patching** (AnyRes do LLaVA-Next, NaViT) para não perder detalhe em imagens grandes.

### Conectores — o que realmente importa

**MLP projector** venceu a guerra na prática: 2 camadas `Linear → GELU → Linear`. Barato, bom, é o que LLaVA e derivados usam. O segredo é **quantidade e qualidade dos dados**, não sofisticação do connector.

Q-Former e Perceiver comprimem para N tokens fixos (ex: 32, 64), útil em contextos longos ou vídeo. Trade-off: perde informação espacial fina; pior em OCR/grounding.

Visual tokens por imagem:
- LLaVA 1.5: 576 (24×24 patches de ViT-L/14 @ 336).
- LLaVA-Next (AnyRes): até ~2880 (grid 2×2 de 576).
- InternVL 2: dinâmico 256-3328 dependendo da resolução.
- Qwen2-VL: dinâmico sem limite rígido, patches temporais em vídeo.

Mais tokens = mais qualidade em detalhes, mais memória e latência. Quase tudo hoje usa resolução dinâmica.

### Datasets multimodais

**Pretraining (pares imagem-texto)**:
- **LAION-5B / LAION-400M** — escala. Qualidade variada, contém NSFW.
- **DataComp / DataComp-1B** — foco em curadoria/filtering.
- **COYO-700M**.
- **CC3M / CC12M** (Conceptual Captions) — curados do Google.
- **ShareGPT4V / ShareCaptioner** — captions densas geradas por GPT-4V.

**Instruction tuning**:
- **LLaVA-Instruct-150k** — o seminal. GPT-4 turbine gerando conversas sobre imagens COCO.
- **ShareGPT4V** — captions longas.
- **Cauldron / The Cauldron** (Idefics) — 50 datasets misturados.
- **Cambrian-7M**, **LLaVA-OneVision Data** — agregados grandes.
- **M3IT, MIMIC-IT, VFlan** — multi-task.
- **Vídeo**: **VideoChat**, **Video-ChatGPT**, **ShareGPT4Video**.
- **OCR/Doc**: **DocVQA**, **ChartQA**, **InfographicVQA**, **UReader**.

**Ação (VLA)**:
- **Open X-Embodiment** — 1M+ demonstrações de 22 robôs.
- **RT-X**, **BridgeData V2**, **DROID**.

### Benchmarks

Gerais (knowledge/reasoning visual):
- **MMMU** — multidisciplinar nível universitário. Métrica padrão de MLLM "frontier".
- **MMBench / MMBench-V** — amplo, multiple choice.
- **MMStar** — filtrado para remover vazamentos.
- **MME** — perception + cognition.

Específicos:
- **VQAv2, GQA, OKVQA, A-OKVQA** — VQA clássico.
- **TextVQA, OCRBench, DocVQA, ChartQA, InfoVQA** — OCR/documento.
- **MathVista, MathVerse** — raciocínio matemático visual.
- **ScienceQA** — ciências multimodal.
- **RefCOCO/RefCOCO+/RefCOCOg** — grounding/referring expression.
- **POPE, HallusionBench** — alucinação.
- **VideoMME, MVBench, LongVideoBench, EgoSchema** — vídeo.

### Fine-tuning de MLLM

- **Full fine-tune** — caro, melhor qualidade. Treina LLM + projector; vision encoder opcional.
- **LoRA no LLM + projector full** — padrão prático. 1-2 GPUs 40-80GB para modelos 7-13B.
- **Só projector** — rápido, limitado. Bom para adaptar a domínio visual muito diferente.
- **Unfreeze do vision encoder** — ajuda em OCR/domínios especializados; caro.

Bibliotecas:
- **transformers** suporta nativamente LLaVA, Idefics, CogVLM, Qwen-VL, Pixtral, Llama-Vision, Phi-Vision, etc. Use `AutoModelForVision2Seq` ou específicos.
- **LLaVA repo oficial** — receita canônica para fine-tune de LLaVA.
- **SWIFT** (ModelScope) — ferramenta abrangente para treinar MLLMs open-source.
- **Xtuner** — InternLM's MLLM training.
- **ms-swift, LLaMA-Factory** — suportam multi-modal fine-tuning com LoRA.

### Serving multimodal

- **vLLM** — suporta LLaVA, Phi-Vision, Pixtral, Qwen2-VL, InternVL, MiniCPM-V, Llama-Vision etc. OpenAI-compatible `/v1/chat/completions` aceitando `image_url` base64 ou URL.
- **SGLang** — suporte multimodal crescente, bom para pipelines com cache.
- **TGI** — suporta MLLMs selecionados.
- **Ollama / llama.cpp** — multimodal via GGUF (LLaVA, MiniCPM-V, Llama-Vision, Qwen2-VL em suporte variável).
- **TensorRT-LLM** — suporte em modelos específicos com overhead de compilação.

Request típico (vLLM/OpenAI-compatible):
```python
import base64, requests

img = base64.b64encode(open("x.jpg","rb").read()).decode()
resp = requests.post("http://localhost:8000/v1/chat/completions", json={
    "model": "llava-hf/llava-1.5-7b-hf",
    "messages": [{
        "role": "user",
        "content": [
            {"type": "text", "text": "What's in this image?"},
            {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{img}"}},
        ],
    }],
})
```

### APIs proprietárias multimodais

- **Claude** (Anthropic) — imagem in, texto out. Bom em documentos, gráficos, UI.
- **GPT-4o / GPT-4o-mini** — imagem/áudio in, texto/áudio out.
- **Gemini 1.5/2.0** — nativo image/video/audio/text. Janela de contexto enorme (1M-2M tokens) = vídeos longos.
- **Mistral Pixtral Large** — open-weights via API.

Padrão da indústria é content blocks: `[{type: text}, {type: image_url}, {type: audio}, ...]`. Maioria aceita base64 ou URL.

### Embeddings multimodais (retrieval / search)

- **CLIP / OpenCLIP** — imagem-texto, padrão histórico.
- **SigLIP / SigLIP 2** — melhor que CLIP em quase tudo.
- **Jina CLIP v2**, **Nomic Embed Vision** — alternativas open com bom trade-off.
- **Voyage multimodal-3** — proprietário, forte.
- **ImageBind** (Meta) — seis modalidades num espaço (image, text, audio, depth, thermal, IMU).
- **LanguageBind** — versão estendida.

Uso típico: indexar catálogo visual, busca cross-modal ("show me images matching this description"), dedup, content moderation, dataset filtering.

### Multimodal RAG

Indexa documentos visuais (PDFs, apresentações, screenshots) por **embedding multimodal** ou por **conversão para texto** via OCR/VLM. Abordagens:

- **Text-first**: extrair texto (PyMuPDF, unstructured, Docling, LlamaParse) + embedar com modelo de texto. Perde figuras/tabelas.
- **Vision-first**: embedar páginas inteiras como imagens com **ColPali / ColQwen2** (embedding page-as-image, retrieval sem OCR). Novo, muito forte em PDFs visuais.
- **Híbrido**: texto + captions geradas por VLM + embeddings multimodais. Melhor qualidade, pipeline mais complexo.

Ver `AI_SYSTEMS_DESIGN.md` para padrões de RAG.

### Gotchas práticos

- **Pré-processamento de imagem é importante** — cada modelo tem resolução alvo, normalização, split/tile logic. `AutoProcessor.from_pretrained(...)` do HF cuida.
- **Visual tokens consomem contexto** — imagem 1024×1024 pode virar 2k-4k tokens. Em conversas longas, sobra pouco para texto.
- **Hallucination multimodal** é pior que em texto puro — modelo "vê" coisas que não estão. Benchmarks como POPE e HallusionBench medem. Mitigação: DPO/RLHF específico, prompting de verificação.
- **OCR fino ainda é fraco** em muitos MLLMs abertos. Qwen2-VL e InternVL-2 são referência.
- **Vídeo tem dois eixos** — temporal sampling (quantos frames) + spatial. Trade-off brutal com contexto.
- **Áudio em LLM** via discretização (neural codec tokens — EnCodec, SoundStream) é o truque. Permite tratar áudio como "texto" no mesmo decoder.
- **Formatos de imagem exóticos** (TIFF multipage, HEIC, 16-bit) frequentemente quebram pipelines que assumem JPEG/PNG.

### Onde o campo vai

- **Unificação** — um único transformer processando tokens de qualquer modalidade (Chameleon, GPT-4o, 4M). Eliminando encoders especializados.
- **Janelas gigantes** — Gemini 1M+ tokens permite vídeos de 1h+ ou documentos enormes em uma call.
- **Geração multimodal nativa** — mesmo modelo que entende também gera imagem/áudio sem pipeline à parte.
- **Agentic multimodal** — MLLM vendo tela/browser e agindo (Claude Computer Use, Anthropic; UI-TARS; OmniParser).
- **Físico / embodied** — VLA models convergindo com foundation models gerais.

## Debugging e profiling

### Python geral

- **py-spy** — sampling profiler, nenhuma instrumentação, attach em processo rodando. Primeiro comando quando algo está lento.
- **line_profiler** — profiling linha-a-linha (`@profile`).
- **memory_profiler**, **tracemalloc**, **guppy/heapy** — memória Python.
- **pdb / ipdb / pudb** — debuggers.
- **rich.traceback** — traceback colorido e legível.

### PyTorch

- **torch.profiler** — GPU kernels, CPU, memória, operador. Exporta trace para Chrome/Perfetto/TensorBoard.
- **`torch.cuda.memory_summary()`** — diagnóstico de VRAM.
- **`torch.autograd.detect_anomaly()`** — rastreia NaN/Inf até op que causou.
- **torchinfo** / **torchsummary** — sumário de parâmetros e shapes.
- **`torch.compile` gotchas** — graph breaks silenciam; use `TORCH_LOGS=graph_breaks` e `torch._dynamo.explain(model)(x)`.

### GPU / kernel

- **nvidia-smi** — básico, GPU util / memória.
- **nvtop / nvitop** — interactive top for GPUs.
- **Nsight Systems (nsys)** — timeline detalhada CPU+GPU, o gold standard para profiling GPU.
- **Nsight Compute (ncu)** — análise de kernel individual (occupancy, memória, warp efficiency).
- **NCCL_DEBUG=INFO** — diagnosticar comunicação multi-GPU.
- **CUDA_LAUNCH_BLOCKING=1** — sincroniza kernels, força erros aparecerem no lugar correto.
- **torch.backends.cudnn.benchmark = True** — ligar quando shapes fixos, acelera; desligar quando shapes variam.

### Erros numéricos

- **NaN/Inf em loss** — quase sempre: LR alto, gradiente explodindo, exp/log sem clamp, softmax sem numerical stability, mixed precision em operação sensível.
- **Gradient clipping** (`torch.nn.utils.clip_grad_norm_`) é barato, ligue por default.
- **Layer norm vs batch norm** em loss que explode — layer norm é mais robusto em transformers.

### Determinismo (reproducibilidade)

```python
import torch, random, numpy as np
torch.manual_seed(0); random.seed(0); np.random.seed(0)
torch.backends.cudnn.deterministic = True
torch.backends.cudnn.benchmark = False
torch.use_deterministic_algorithms(True, warn_only=True)
```

Custo: ~10-30% mais lento. Worth it em debug, desligar em treino final.

## Convenções e boas práticas que economizam tempo

- **Fixe versões** em `requirements.txt` / `pyproject.toml`. PyTorch + CUDA specifically (`torch==2.5.1+cu124`).
- **Ambiente isolado** — `uv` (rápido, novo) ou conda/mamba. Dependências ML têm binários, evite pip puro em CUDA.
- **Seeds e determinismo** nos experimentos críticos, aceitar não-determinismo em dev.
- **Checkpoint a cada N steps, não a cada N epochs** — treinos longos crasham, epoch pode ser gigante.
- **Eval em subset pequeno durante treino**, eval completo apenas no final ou a cada M steps.
- **Log hyperparams completos** (config inteiro) junto das métricas; meio config perdido = experiment irreproduzível.
- **Smoke test com toy data** antes de rodar no dataset grande — pega 80% dos bugs em 1% do tempo.
- **Overfit em batch único** para validar que modelo consegue aprender. Se não overfitta 1 batch, há bug.
- **Verifique shapes** no forward (pode usar `einops.rearrange` + asserts). `einops` é obrigatório em código com tensor manipulation não-trivial.
- **Profile antes de otimizar** — intuição sobre onde está o gargalo em pipeline ML é quase sempre errada.
- **Use bibliotecas em vez de reimplementar** — loss funcs, schedulers, attention variants. Há 10 anos de otimização em código maduro.

## Mapa: o que combinar na prática

**Projeto protótipo ML tabular em 1 dia**: `polars` + `scikit-learn` + `lightgbm` + `optuna` + `wandb`. Pipeline sklearn, log de cada run.

**Fine-tune LLM 7B em 1-2 GPUs**: `transformers` + `peft` (QLoRA) + `trl` (SFTTrainer/DPOTrainer) + `accelerate` + `wandb`. GGUF export via `llama.cpp` para deploy.

**Treinar modelo de visão custom**: `pytorch` + `timm` (backbone) + `albumentations` + `lightning` ou loop direto + `wandb`. Deploy via `onnxruntime` ou `torchscript`.

**LLM serving em produção**: `vllm` atrás de API OpenAI-compatible, `pydantic` para schema, `prometheus` para métricas, autoscale em k8s.

**Pesquisa em escala Google**: `jax` + `flax` + `optax` em TPU pod, checkpoint com `orbax`, logging com `wandb` ou interno.

**App local de IA**: `ollama` + `langchain`/`llamaindex` + `gradio`/`streamlit` para UI. Embeddings via `sentence-transformers`, vector store em `chroma`/`lancedb`/`faiss`.

**Pipeline produção classic ML**: `airflow`/`prefect`/`dagster` para orquestração, `dvc` para versionar dados, `mlflow` para tracking + registry, `bentoml`/`triton` para serving, `evidently`/`whylogs` para monitoring/drift.
