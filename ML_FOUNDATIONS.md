# ML Foundations

Conceitos e princípios de treinamento de redes neurais que envelhecem pouco: matemática de optimizers, schedules, regularização, paralelismo, precisão numérica, taxonomia de arquiteturas. Para bibliotecas, versões e "estado do ecossistema" ver `ML_FRAMEWORKS.md` (snapshot datado). Para fundamentos de ML/DL mais amplos (neurônio, backprop, transformer), `AI.md`. Para RL, `AGENTS.md`.

Exemplos em pseudocódigo ou PyTorch por legibilidade, mas as ideias são framework-agnóstico.

## Loop canônico de treino supervisionado

```
para cada batch (x, y) do dataset:
    ŷ     = modelo(x)              # forward
    L     = loss(ŷ, y)              # scalar
    ∇θ    = ∂L/∂θ                   # backward (autograd)
    θ     = optimizer.step(θ, ∇θ)   # atualiza parâmetros
    opcional: scheduler.step(), grad clip, logging
```

Tudo o que vem abaixo são decisões sobre **como cada uma dessas linhas funciona**.

## Paradigmas de aprendizado

O loop acima é **supervisionado** — existe target `y` explícito. É um caso particular. Os paradigmas que dominam hoje:

- **Supervised learning** — `(x, y)` pareados. Loss compara predição com label humano/automático.
- **Self-supervised (SSL)** — target é **construído a partir de `x` mesmo**. Sem label humano. Subtipos:
  - **Masked modeling** — esconde parte de `x`, prediz o que foi escondido. BERT (palavras), MAE (patches), wav2vec (áudio).
  - **Next-token / autoregressive** — prediz `x_{t+1}` dado `x_{1..t}`. GPT-family, causal language modeling. É a base de todos os LLMs decoder-only.
  - **Denoising** — adiciona ruído a `x`, treina para remover. Base de diffusion models.
  - **Contrastive** — aproxima representações de "views" do mesmo objeto, afasta de outros. SimCLR, MoCo, CLIP (cross-modal).
  - **Joint embedding / BYOL-style** — aprende sem negativos via predictor + EMA teacher.
- **Semi-supervised** — mistura pequenos dados rotulados + muitos não-rotulados. Pseudo-labeling, consistency regularization.
- **Weakly supervised** — labels ruidosos ou em granularidade diferente (image-level label para detecção, por exemplo).
- **Transfer / fine-tuning** — pré-treina em tarefa abundante, adapta a tarefa alvo. Ver seção própria.
- **Distillation** — student aprende a imitar teacher (soft targets). Comprime conhecimento e/ou transfere entre arquiteturas. Hinton 2015.
- **Reinforcement learning** — sem target fixo; reward define objetivo. Ver `AGENTS.md`.
- **RL from human feedback (RLHF) / preference tuning** — humanos comparam pares de outputs; treina reward model + policy (PPO) ou direto (DPO/IPO/KTO). Base do alinhamento de LLMs.
- **Meta-learning ("learning to learn")** — aprende a adaptar rápido a novas tarefas com poucos samples. MAML, Reptile, in-context learning emergente.

Mental model: **quase toda a IA moderna é self-supervised pre-training massivo + fine-tuning/alinhamento supervisionado/RLHF**. O supervisionado puro virou a última etapa, não o jogo principal.

## Fundamentos probabilísticos

Quase toda loss function usada em deep learning tem uma interpretação probabilística. Saber qual conecta "por que essa fórmula" com "o que está sendo maximizado".

- **MLE (Maximum Likelihood Estimation)** — maximizar `p(dados | θ)` sobre `θ`. Equivalente a minimizar **NLL** (`-log p`).
  - **MSE** em regressão = MLE assumindo ruído gaussiano com variância fixa.
  - **Cross-entropy** em classificação = NLL de distribuição categórica.
  - **BCE** = NLL de Bernoulli.
- **Entropia** `H(p) = -Σ p log p` — incerteza média de uma distribuição.
- **Cross-entropy** `H(p, q) = -Σ p log q` — quantos nats/bits precisa para codificar `p` usando `q`.
- **KL divergence** `D_KL(p || q) = H(p, q) - H(p)` — "distância" assimétrica de `p` a `q`. Sempre ≥ 0, zero iff `p=q`.
- Minimizar cross-entropy entre target one-hot e predição ≡ minimizar KL + constante ≡ MLE. Três formulações, uma operação.
- **MAP (Maximum A Posteriori)** = MLE + prior. Weight decay = MAP com prior gaussiano nos pesos (L2). Sparsity = prior Laplace (L1).
- **Variational lower bound (ELBO)** — `log p(x) ≥ E_q[log p(x,z)] - E_q[log q(z)]`. Base de VAEs, diffusion models, variational inference geral. Intratável → aproxima.
- **InfoNCE** em contrastive learning — lower bound em mutual information entre views. "Aprender representação que preserva info".
- **Score matching** e **diffusion** — em vez de `p(x)`, aprende `∇_x log p(x)` (score). Evita normalização.
- **Calibração** — predição probabilística "correta" vs "overconfident". Temperature scaling, Platt scaling, isotonic regression. Modelos modernos são rotineiramente mal-calibrados.
- **Evidence / Bayesian deep learning** — posterior sobre pesos em vez de ponto. MC Dropout, ensembles, Laplace approximation. Custo alto, útil para uncertainty estimation.

Consequência prática: quando você escreve uma loss custom, pergunte "que distribuição estou assumindo?". Se não souber responder, provavelmente está reinventando algo que já tem nome e variantes estudadas.

## Optimizers

- **SGD + momentum** — `v ← μv + ∇θ; θ ← θ - ηv`. Momentum típico μ=0.9. Nesterov adiciona lookahead (calcula gradiente em `θ - μv`, não em `θ`). Ainda é **o melhor optimizer para visão** (ResNet, ConvNeXt) quando bem tunado. Convergência mais lenta, mas solução final frequentemente generaliza melhor que Adam.
- **Adam** (Kingma & Ba, 2014) — adaptive learning rate por parâmetro via médias móveis de gradiente (`m`) e gradiente-quadrado (`v`). Bias correction nos primeiros steps. Betas default `(0.9, 0.999)`, eps `1e-8`. Convergência rápida, robusto.
- **AdamW** (Loshchilov & Hutter, 2019) — Adam com **weight decay desacoplado**. O Adam original aplicava weight decay via L2 no gradiente, o que interage mal com adaptive LR (parâmetros com gradiente grande eram menos regularizados). AdamW aplica decay **diretamente nos pesos** após o step. Dominante em transformers na prática 2020-2026.
- **Adafactor** — variante de Adam que **fatoriza** a matriz `v` em produto de vetores (linha × coluna). Memória O(m+n) em vez de O(mn). Default em treinos grandes de transformer (T5, PaLM).
- **Lion** (Chen et al, 2023) — "EvoLved Sign Momentum". Usa **apenas o sinal** do momentum em vez da magnitude. Memória metade do AdamW (não guarda `v`), frequentemente bate AdamW com LR ~10× menor. Betas `(0.9, 0.99)`.
- **Sophia** (Liu et al, 2023) — second-order, estima diagonal do Hessian via Hutchinson ou GNB. Promessa teórica em convergência, adoção prática ainda limitada.
- **Shampoo** — full-matrix preconditioning via Kronecker. Eficiente por FLOP, caro por step.
- **LARS / LAMB** — normalizam por camada. Permitem batches enormes (8k+). LARS para SGD/visão, LAMB para Adam/BERT.

**Mental model**: SGD é direção "crua" do gradiente escalada por LR; Adam-family reescala cada parâmetro por quão volátil ele tem sido. Adaptativo converge rápido em paisagens ruidosas; SGD+momentum pode generalizar melhor em paisagens bem-comportadas.

## Weight decay

- Weight decay = L2 regularização = shrinkage de pesos a cada step. Mas **como** aplicar importa:
  - **Coupled** (L2 no gradiente, Adam original): `∇θ += λθ`. Interage com o adaptive rescaling, efeito real de decay varia por parâmetro.
  - **Decoupled** (AdamW): `θ ← θ - η(update + λθ)`. Decay uniforme.
- Valores típicos: **0.01-0.1** para transformers, **1e-4 a 5e-4** para CNN com SGD.
- **Convenção universal**: excluir do decay os parâmetros que não se beneficiam dele — **bias** e pesos de **normalização** (LayerNorm, BatchNorm, RMSNorm). Zero-centrar LayerNorm weights (que começam em 1) via decay quebra a normalização.

```python
no_decay = ["bias", "LayerNorm.weight", "layer_norm.weight", "ln_"]
groups = [
    {"params": [p for n, p in model.named_parameters() if not any(k in n for k in no_decay)],
     "weight_decay": 0.1},
    {"params": [p for n, p in model.named_parameters() if any(k in n for k in no_decay)],
     "weight_decay": 0.0},
]
```

## Learning rate schedules

- **Constant** — só para debug ou fine-tuning muito curto.
- **Step decay** — `lr *= γ` a cada N epochs. Clássico em visão, γ=0.1 a cada 30 epochs.
- **Cosine decay** — `lr(t) = lr_max · 0.5 · (1 + cos(π·t/T))`. Suave, sem cliff. Padrão em transformers modernos.
- **Linear decay** — linear de `lr_max` até `lr_min`.
- **Warmup + decay** — **obrigatório em Adam/transformer**. Primeiros 1-10% dos steps com LR subindo de ~0 até `lr_max`, depois decay (cosine ou linear). Sem warmup, Adam com `v` subestimado nos primeiros steps gera updates gigantes e desestabiliza.
- **OneCycle** (Smith) — warmup + annealing em um ciclo longo. Muito eficaz para treinos curtos em visão.
- **ReduceLROnPlateau** — reativo, reduz LR quando métrica estagna. Útil quando não se sabe o número total de steps.
- **Inverse square root (Noam)** — `lr ∝ 1/√t` após warmup. Usado no Transformer original.

Por que schedule importa: LR alto no início explora bem; LR baixo no fim refina. Manter LR fixo alto = nunca converge fino; fixo baixo = lento e para em mínimos rasos.

```python
def cosine_with_warmup(step, warmup=1000, total=100_000):
    if step < warmup:
        return step / warmup
    p = (step - warmup) / (total - warmup)
    return 0.5 * (1 + math.cos(math.pi * p))
```

## Batch size e LR scaling

Batch size afeta **qualidade, estabilidade, throughput e generalização**. Aumentar batch ≠ sempre melhor.

Regras de scaling ao mudar batch:
- **Linear scaling** (Goyal et al, "Accurate Large Minibatch SGD") — se batch × k, LR × k. Funciona até ~8k batch. Exige warmup longo.
- **Square-root scaling** — `lr ∝ √batch`. Mais conservador. Frequentemente melhor para Adam/transformer.
- **Sem scaling** — às vezes o correto em fine-tuning (LR depende da tarefa, não do batch).

**Gradient accumulation** simula batch grande em hardware pequeno. Matematicamente **idêntico** a batch maior — exceto **BatchNorm**, que computa estatísticas por micro-batch e não acumula.

```python
for i, batch in enumerate(loader):
    loss = model(batch) / accum_steps
    loss.backward()
    if (i + 1) % accum_steps == 0:
        optimizer.step()
        optimizer.zero_grad()
```

**Large batch trade-off** — batches enormes tendem a convergir para mínimos "sharp" que generalizam pior (Keskar et al, 2017). Compensado por LR alto, warmup, gradient noise (augmentation forte).

## Gradient clipping

Limita norma do gradiente antes do step. Barato e quase sempre vale ligar em transformers.

```python
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
```

- `max_norm=1.0` é default razoável.
- Clip por value (elementwise) é menos usado.
- Monitore a norma: se **sempre** clipando, LR está alto; se **nunca** clipando, pode desligar.

## Regularização

- **Weight decay** — principal regularizador em transformers.
- **Dropout** — zera ativações com prob `p` no treino. Em transformers: 0.0-0.1 em atenção e FFN. Dropout alto (>0.2) raramente ajuda em modelos grandes — escala de dados já regulariza.
- **DropPath / Stochastic Depth** — dropout de blocos inteiros via skip connection. Padrão em ViT, ConvNeXt. Rate cresce linearmente com profundidade.
- **Label smoothing** — em vez de one-hot, usa `(1-ε)` no correto e `ε/(K-1)` nos outros. Típico ε=0.1. Melhora calibração e generalização.
- **Mixup** (Zhang et al, 2017) — `x = λx₁ + (1-λ)x₂`, mesmo para `y`. `λ ~ Beta(α, α)`.
- **CutMix** — substitui patch de `x₁` por `x₂`, label proporcional à área. Tipicamente bate Mixup em visão.
- **EMA de pesos** — `θ_ema = β·θ_ema + (1-β)·θ`, usa EMA na inference. β=0.999-0.9999. Quase de graça e melhora 0.5-2% em accuracy. Padrão em contrastive (SimCLR, BYOL), diffusion, ViT.
- **Data augmentation** — regularizador mais poderoso em visão. A augmentation "correta" supera qualquer truque de arquitetura.

Princípio: regularização é **restrição no que o modelo pode memorizar**. Quanto menor o dataset relativo à capacidade, mais regularização precisa.

## Inicialização

- **Xavier / Glorot** (tanh/sigmoid) — var ∝ `2/(fan_in + fan_out)`.
- **He / Kaiming** (ReLU-like) — var ∝ `2/fan_in`. Default em CNN modernas.
- **Truncated normal** — normal cortado em ±2σ, evita outliers. Padrão em ViT, BERT (std=0.02).
- **Orthogonal** — útil em RNN para preservar norma de gradiente.
- **muP** (µ-Parametrization, Yang et al) — inicialização + rescaling que faz **hiperparâmetros transferirem entre tamanhos de modelo**. Tuna em small, aplica em large sem retune.

Regra prática para transformers: `trunc_normal(std=0.02)` em Linear e Embedding, zero em bias. Projeções residuais com std reduzido (`0.02/√(2·n_layer)`) estabiliza treino profundo.

## Normalização

- **BatchNorm** — normaliza na dimensão batch. Efetivo em CNN, problemático em batch pequeno (estatísticas ruidosas), e depende de moving averages no inference. **Não use em transformers ou RNN**.
- **LayerNorm** — normaliza na dimensão de features. Sem dependência de batch. Default em transformers.
- **RMSNorm** — LN simplificado, só divide por RMS (sem subtrair média, sem bias). Mais barato, sem perda prática. Dominante em LLMs modernos.
- **GroupNorm** — normaliza em grupos de canais. Substituto de BN quando batch é pequeno (detecção, segmentação).
- **WeightNorm** — normaliza pesos em vez de ativações. Legado.

**Pre-norm vs Post-norm** em transformers:
- Pre-norm: `x + Attn(LN(x))` — mais estável em treino, permite mais camadas com warmup menos agressivo.
- Post-norm (original do Vaswani): `LN(x + Attn(x))` — às vezes generaliza marginalmente melhor, mas exige warmup forte.
- LLMs modernos são quase todos pre-norm.

## Gradient flow, residuals e treino profundo

Por que redes profundas são difíceis de treinar:

- **Vanishing gradient** — em redes profundas com ativação saturante (sigmoid, tanh) ou peso pequeno, produto de Jacobians encolhe exponencialmente com profundidade. Camadas iniciais não recebem sinal.
- **Exploding gradient** — o inverso: produto cresce exponencialmente. Loss vira NaN. Grad clipping mitiga, mas é sintoma, não causa.
- **Signal propagation** — similar, no forward: variância das ativações colapsa ou explode camada a camada se inicialização e norm não estiverem em sintonia.

As três invenções que resolveram e são pilares de toda arquitetura moderna:

1. **Inicialização scale-aware** — Xavier/He escolhem variância para preservar variância do sinal camada a camada. Sem isso, redes com >5 camadas já não treinam bem.
2. **Normalização de ativações** — BN/LN/RMSNorm forçam estatísticas estáveis. Permite LR maior, reduz sensibilidade a init, acelera convergência. Efeito colateral: regularização leve.
3. **Residual connections** `y = x + F(x)` (He et al, ResNet, 2015) — gradiente flui por caminho "skip" sem ser multiplicado pela camada. Permite arbitrariamente profundo. **Transformer, ViT, U-Net, diffusion, LLMs — tudo é residual**. Uma das ideias arquiteturais mais importantes da década.

Por que residuais funcionam (várias lentes complementares):
- **Gradient path analysis** — gradiente do loss em relação a `x` é `∂L/∂y · (I + ∂F/∂x)`. O `I` garante que gradiente **não encolhe** com profundidade.
- **Ensemble implícito** (Veit et al) — rede residual comporta-se como ensemble de caminhos de comprimentos variados.
- **Identity como default** — camada aprende perturbação em torno da identidade, mais fácil do que aprender mapeamento do zero.

Combinação canônica em transformers: **pre-norm + residual + scale-aware init + AdamW + warmup**. Remove qualquer um e treinos profundos começam a quebrar.

Sintomas e mitigações:
- Loss vira NaN cedo → exploding, reduza LR ou aumente warmup.
- Loss fica plano em valor alto → vanishing ou modelo sem capacidade. Checar grad norm por camada.
- Gradient norm oscilando muito → instabilidade numérica. Usar BF16 em vez de FP16, ou promover softmax/LN para FP32.
- Última camada aprende, primeiras não → classic vanishing; verifique init e se há normalização suficiente.

## Funções de ativação

- **ReLU** — `max(0, x)`. Simples, rápido, risco de dead neurons.
- **LeakyReLU / PReLU** — mitiga dead neurons com slope pequeno no negativo.
- **GELU** — `x·Φ(x)`. Suave. Padrão em BERT/GPT clássicos.
- **SiLU / Swish** — `x·σ(x)`. Parecido com GELU, mais barato.
- **SwiGLU** — `SiLU(Wx) ⊙ Vx`. Gated variant em FFN, estado da arte em LLM. Custa ~1.5× parâmetros no FFN.
- **GEGLU** — variante com GELU em vez de SiLU.
- **Softmax** — em output de classificação e em attention. Numericamente estável via logsumexp (frameworks cuidam).

## Loss functions

**Classificação**:
- **Cross-entropy** — default. Combinar log_softmax + NLL em uma op é numericamente estável.
- **Focal loss** — `(1-pₜ)^γ · CE`. Foca em exemplos difíceis. Padrão em detecção desbalanceada.
- **Label smoothing** — modificação do target (ver Regularização).
- **Class weighting** — pesos inversamente proporcionais à frequência.
- **BCE with logits** — multi-label ou binary, numericamente estável.

**Regressão**:
- **MSE / L2** — sensível a outliers.
- **MAE / L1** — robusto a outliers, gradiente descontínuo.
- **Huber / Smooth L1** — quadrático perto de 0, linear longe.
- **Quantile loss** — predição de percentis.

**Similaridade / embedding**:
- **Contrastive** (pairwise margin) — `max(0, margin - d(pos) + d(neg))`.
- **Triplet** — anchor/pos/neg.
- **InfoNCE / NT-Xent** — softmax sobre similaridades num batch. Base de CLIP, SimCLR.
- **Sigmoid pairwise** — usado em SigLIP, mais estável em larga escala.

**Segmentação**:
- **Dice loss** — `1 - 2|A∩B|/(|A|+|B|)`. Robusto a desbalanceamento.
- **Tversky** — generalização de Dice com peso FP/FN.
- Frequentemente combinado com CE: `0.5·CE + 0.5·Dice`.

**Language modeling**:
- **Cross-entropy per token**, `ignore_index=-100` para padding. Perplexity = `exp(mean_ce)`.

## Precisão numérica

Formatos:
- **FP32** (single, E8M23) — default histórico. Overkill em treino moderno.
- **FP16** (half, E5M10) — range pequeno (±65k), underflow em gradientes. Exige **loss scaling** (multiplica loss antes de backward, divide gradientes depois).
- **BF16** (E8M7) — mesmo expoente de FP32, menos mantissa. Sem overflow prático, sem loss scaling. Padrão em TPU e GPU NVIDIA Ampere+.
- **TF32** — formato interno NVIDIA para matmul (E8M10). FP32 nas I/O.
- **FP8** (E4M3/E5M2) — hardware-dependente. Dobra throughput quando suportado, exige tooling especial.
- **INT8 / INT4** — inference-only na prática, via quantization.

Regra prática (contingente a hardware atual): **BF16 quando disponível** (mais estável que FP16); FP16 com loss scaling em hardware que não suporta BF16; FP32 só em módulos sensíveis (softmax, LN, reduções grandes). Frameworks autocast fazem isso automaticamente.

Mixed precision básico (padrão conceitual, PyTorch API):
```python
with autocast(device_type="cuda", dtype=torch.bfloat16):
    loss = model(batch)
loss.backward()
optimizer.step()
```

Onde precisão quebra:
- **Softmax/LogSoftmax/LayerNorm** em FP16 — promova para FP32.
- **Reduções grandes** (soma de milhões de elementos) acumulam erro em FP16.
- **Gradiente de embeddings** com vocabulário enorme pode underflow.

## Paralelismo em treino (conceitos)

Todas as formas combinam dois eixos: **como os dados são distribuídos** e **como o modelo é distribuído**.

- **Data Parallel (DP)** — cada device tem cópia completa do modelo, processa batch diferente, sincroniza gradientes via all-reduce. Simples, escala até comunicação saturar.
- **Tensor Parallel (TP)** — split de camadas individuais entre devices (split da matriz de peso na dim apropriada). Necessário quando uma layer não cabe em uma GPU.
- **Pipeline Parallel (PP)** — split do modelo em stages sequenciais, um por device. Micro-batching para manter devices ocupados (GPipe, PipeDream, 1F1B).
- **Expert Parallel (EP)** — para MoE, cada expert em device diferente.
- **Sequence / Context Parallel** — split da dimensão de sequência. Necessário para contextos longos.
- **ZeRO / FSDP** — shard de optimizer states (nível 1), gradientes (nível 2), parâmetros (nível 3). Reduz memória sem mudar matemática — all-gather antes de usar, scatter depois.

Treinamento moderno de LLM combina **DP + TP + PP + FSDP** em "3D/4D parallelism".

## Truques de memória

- **Mixed precision** — FP16/BF16 em ativações e gradientes, FP32 para accumulation em optimizer.
- **Gradient checkpointing** — não guarda ativações intermediárias; recomputa no backward. Troca compute por memória. Tipicamente +30% tempo, -50% memória.
- **Gradient accumulation** — acumula gradientes de N micro-batches antes de `step()`. Simula batch maior sem memória extra.
- **Flash Attention** — atenção exata, IO-aware, memória linear no seq length em vez de quadrática. Estado da arte.
- **Paged / KV cache paging** — em inference, aloca KV cache em páginas fixas em vez de contíguo. Reduz fragmentação.
- **Offloading** — optimizer states e parâmetros em CPU/NVMe, GPU só carrega o que precisa no momento. Treina modelo maior que VRAM.
- **Quantização em treino** — QLoRA-style: base em NF4/INT4 congelado, adapters pequenos em FP16/BF16.
- **MoE** — ativa subset de experts por token. FLOPs reduzidos proporcionalmente a `ativos/total`.

## Scaling laws

Uma das descobertas mais importantes da última década: qualidade de modelos neurais segue **leis de potência** previsíveis em função de três eixos — **tamanho do modelo (N)**, **tamanho do dataset (D)**, e **compute (C)**.

**Kaplan et al (OpenAI, 2020)** — "Scaling Laws for Neural Language Models":
- `Loss ≈ (N_c/N)^α_N + (D_c/D)^α_D + irreducible`
- Loss decai como power-law em N, D e C, com expoentes pequenos (~0.05-0.1).
- Perdas são **previsíveis** por ordens de magnitude — treinar modelo pequeno permite extrapolar.
- Overhead arquitetural (depth vs width, heads, etc) é marginal comparado a N, D, C.

**Hoffmann et al (DeepMind, Chinchilla, 2022)** — corrigiu Kaplan:
- Para um orçamento fixo de compute `C`, há alocação **ótima** entre N (parâmetros) e D (tokens).
- Kaplan subestimou D, treinou modelos grandes demais para pouco dado. Chinchilla mostrou que N e D devem escalar **aproximadamente igual** (~20 tokens por parâmetro).
- Reversão: GPT-3 (175B, 300B tokens) é subtreinado. Chinchilla (70B, 1.4T tokens) bate com ~metade dos parâmetros.
- Desde 2022, quase todo LLM segue proporções Chinchilla-style ou **mais** tokens (LLaMA, Qwen, Gemma treinam com 10-50× Chinchilla-ótimo porque inference é o custo dominante — vale a pena modelo pequeno com muito treino).

Consequências práticas:
- **Compute-optimal training** — para pesquisa com orçamento fixo, `N_opt ∝ C^0.5`, `D_opt ∝ C^0.5`.
- **Inference-optimal training** — se o modelo vai ser usado muito, treine mais além do Chinchilla-ótimo. Gasta mais em treino, economiza em deployment.
- **Previsão de performance** — antes de gastar 10M em compute, rode 6-8 runs menores e extrapole.

**Emergência** (Wei et al, 2022; contestado por Schaeffer et al, 2023):
- Algumas capacidades parecem "aparecer do nada" em certa escala (aritmética multi-dígito, raciocínio multi-step). Métricas discretas (accuracy exata) mascaram progresso contínuo em log-likelihood.
- Debate ativo se "emergência" é fenômeno real ou artefato de métrica.

**Para outras modalidades**:
- Visão (Zhai et al, ViT scaling) segue leis similares com expoentes diferentes.
- Contrastive / CLIP escala com N e D mas **a qualidade dos pares** domina.
- RL tem scaling muito mais fraco — dados são escassos e ruidosos.

**Limite irredutível** — existe um floor de loss que não depende de N/D, imposto pela entropia do próprio dado (ruído intrínseco, ambiguidade).

**Custo e data wall**: em 2024-2026 discute-se se estamos aproximando de um "data wall" — texto de qualidade disponível < o que modelos de fronteira absorvem. Síntese de dados, filtering agressivo, e múltiplas épocas viraram práticas comuns.

## Bias-variance, overparametrização e double descent

O dogma clássico de ML (anos 1990-2010): "mais parâmetros = overfit; regularize para reduzir variance". Deep learning quebrou essa intuição.

**Bias-variance trade-off clássico**:
- `Erro esperado = Bias² + Variance + Ruído irredutível`.
- Modelo simples: alto bias, baixa variance.
- Modelo complexo: baixo bias, alta variance.
- Curva em U: há um sweet spot intermediário.

**O que deep learning mostrou**:
- Redes com **muito mais parâmetros que samples** (overparametrized) **generalizam bem**, contra a intuição clássica.
- Zhang et al (2017) — redes conseguem memorizar labels aleatórios perfeitamente, **mas** generalizam em labels reais. A capacidade de memorizar não impede generalização quando os dados têm estrutura.

**Double descent** (Belkin et al, 2019):
- Aumentando capacidade do modelo, erro de teste cai (regime clássico), sobe perto do interpolation threshold (modelo consegue ajustar exatamente os dados de treino), e **cai de novo** além dele.
- Há duas descidas, não uma. O regime moderno (deep learning) opera na **segunda descida**.
- Fenômeno também ocorre em função de N (tamanho do dataset) e de epochs.

**Por que modelos grandes generalizam**:
- **Implicit bias de SGD/Adam** — entre as muitas soluções que ajustam perfeitamente o treino, otimização favorece soluções "simples" (flat minima, low norm, low complexity).
- **Lottery ticket hypothesis** (Frankle & Carbin, 2019) — redes grandes contêm subredes pequenas ("winning tickets") que, quando treinadas sozinhas do init correto, atingem performance comparável. Grande rede = muitas "tentativas" paralelas.
- **Neural tangent kernel (NTK)** — no limite de largura infinita, rede comporta-se como kernel method, com teoria de generalização conhecida.

Consequências práticas:
- **Regularização ≠ modelo menor**. Muitas vezes vale mais regularizar (weight decay, augmentation) um modelo grande do que encolher.
- **Para um mesmo budget**, modelo mais largo frequentemente bate modelo mais profundo até certo ponto.
- **Early stopping** é uma forma de regularização implícita — para no ponto onde gap train-test é aceitável.

## Inductive biases por família de arquitetura

Toda arquitetura embute **suposições sobre estrutura dos dados**. Entender os biases explica quando cada uma vence.

- **MLP / fully-connected** — **nenhum bias espacial**. Cada input conecta a cada hidden. Flexível, mas precisa aprender do zero qualquer estrutura. Bom baseline em dados tabulares sem ordem.
- **CNN** — **localidade** + **translation equivariance** (deslocar input desloca feature map) + **composicionalidade hierárquica**. Feito sob medida para pixels/sinais onde padrões locais repetem-se em várias posições.
- **RNN / LSTM / GRU** — **sequencialidade** + **memória via estado oculto**. Ordem importa, processamento passo-a-passo. Limitação: gradiente difícil em sequências longas, paralelismo ruim.
- **Transformer** — **permutation-invariance** (ordem via positional encoding explícito), **global dependence** via attention, **paralelismo total**. Poucos biases fortes, **aprende estrutura dos dados**. Por isso precisa de mais dados para superar RNN/CNN em regimes pequenos, mas escala melhor.
- **GNN** — **permutation-invariance em vizinhos** + **message passing em grafos**. Bom para dados relacionais (moléculas, redes sociais, código).
- **State Space Models (Mamba, S4)** — **sequencialidade linear** + **memória comprimida de tamanho fixo**. Competitivo com transformer em sequências longas, complexidade linear em tamanho vs quadrática.
- **Diffusion / score-based** — **modelagem de densidade via denoising**. Bias para mapas suaves em espaços contínuos.

**Princípio**: quanto mais bias correto a arquitetura tiver, menos dado precisa. Quanto mais geral, mais dado + compute exige **mas** mais generalizável a novas tarefas.

Por isso a progressão histórica: MLP → CNN (visão) → RNN → Transformer. Cada passo reduz bias e aumenta necessidade de dados, mas destrava mais capacidade.

## Transfer learning e fine-tuning (princípios)

Treinar do zero é quase sempre subótimo quando existe modelo pré-treinado relevante.

- **Pre-training** — treina em dados abundantes em tarefa genérica (language modeling, classification em ImageNet, contrastive). Aprende representações úteis.
- **Fine-tuning** — adapta ao problema alvo com dados limitados.

Variantes por quanto do modelo é treinado:
- **Linear probing** — congela backbone, treina só cabeça linear. Rápido, funciona se representação pré-treinada já separa as classes.
- **Full fine-tuning** — treina tudo. Melhor qualidade, exige mais dados e compute, risco de catastrophic forgetting.
- **Partial fine-tuning** — descongela últimas camadas. Meio-termo.
- **Adapter methods** — insere pequenos módulos treináveis em módulos congelados. Adapters, LoRA (low-rank update das matrizes de peso), IA³, prefix tuning. Muito menos parâmetros treináveis (0.1-1%), qualidade próxima de full FT.
- **Prompt tuning / soft prompts** — aprende tokens de prompt contínuos, modelo totalmente congelado.

**Catastrophic forgetting** — fine-tuning em tarefa B pode destruir performance em A. Mitigações: rehearsal de dados de A, EWC (penaliza mudança em parâmetros importantes), adapters (não mexe no backbone).

**LoRA** (Hu et al, 2021) como princípio: `W + ΔW`, onde `ΔW = BA` com `B∈R^{d×r}`, `A∈R^{r×d}`, `r << d`. Representa ΔW como soma de r outer products. Hipótese: atualização necessária para fine-tuning tem baixa rank intrínseca. Empiricamente verdadeiro na maioria dos casos.

**Princípio geral**: LR de fine-tuning é **muito menor** que LR de pre-training (tipicamente 10-100× menor). Fine-tuning muda muito parâmetro pouco; pre-training muda parâmetro muito.

**In-context learning** (emergente em LLMs grandes) — sem fine-tuning nenhum, modelo "aprende" da próprio prompt. Não é aprendizado no sentido clássico (pesos não mudam), mas capacidade emergente com escala. Zero-shot, few-shot prompting.

## Hyperparameter search (metodologia)

Ordem de impacto típica (do mais ao menos importante):
1. **Learning rate** — log-uniform `[1e-5, 1e-2]`.
2. **Batch size** — rescale LR ao mudar.
3. **Weight decay** — `[0, 0.3]`.
4. **Warmup fraction** — `[0, 10%]` do total.
5. **Arquitetura** (width/depth) — se orçamento permitir.
6. **Dropout** — `[0.0, 0.3]`.
7. **Optimizer betas** — default geralmente ótimo.

Estratégias:
- **Grid search** — só ≤3 params.
- **Random search** — surpreendentemente eficaz (Bergstra & Bengio). Cobre LR e WD bem com 20-50 trials.
- **Bayesian (TPE, GP)** — TPE (Optuna default) converge com menos trials. Combine com pruner para matar runs ruins cedo.
- **Population Based Training (PBT)** — múltiplas configs em paralelo, copiam pesos/params de quem vai bem. Caro mas bom para schedules.

Heurísticas:
- Fixe seed durante tuning (reduz variância).
- Use dataset/steps reduzidos; valide final com full.
- Tune primeiro LR + WD; depois architectural; depois aug/dropout.
- Log de **tudo** — configs que pareciam ruins podem se tornar insights.

## Taxonomia de arquiteturas multimodais

Independente de modelos específicos, há poucos padrões arquiteturais que se repetem:

**Dual encoder contrastivo**:
- Dois encoders (ex: texto + imagem) projetam em espaço compartilhado.
- Treinado com InfoNCE ou sigmoid pairwise loss.
- **Não gera** — só compara embeddings. Usado em retrieval, zero-shot classification, filtering de datasets.
- Exemplos clássicos da família: CLIP, SigLIP, ALIGN.

**Fusion / cross-attention (encoder-decoder multimodal)**:
- Encoders especializados (vision, audio) produzem features.
- Cross-attention injeta essas features em um decoder de texto.
- Permite geração condicional em modalidade não-textual.

**VLM / MLLM (vision encoder + connector + LLM)**:
```
  modalidade → encoder → connector/projector → LLM ← tokens de texto
                                                 ↓
                                            saída (tokens)
```
- **Encoder** extrai features (CLIP, SigLIP, DINOv2, ou custom).
- **Connector** alinha espaço do encoder ao espaço do LLM:
  - **MLP projector** — 2 camadas lineares + ativação. Simples, venceu na prática.
  - **Q-Former** — transformer com queries aprendíveis, comprime para N tokens.
  - **Perceiver Resampler** — cross-attention com latents fixos.
  - **Cross-attention layers** injetados no LLM.
- **LLM** decoder gera texto condicionado em tokens de texto + visual.

Trade-off: mais visual tokens = mais detalhe, mais memória e latência. Connectors que comprimem (Q-Former, resampler) perdem detalhe fino; MLPs preservam. Escolha é função do uso (OCR exige detalhe; QA geral tolera compressão).

**Any-to-any (tokenização unificada)**:
- Imagem/áudio/vídeo discretizados em tokens (via VQ-VAE, neural codec) e tratados no mesmo vocabulário que texto.
- Um único transformer processa e gera qualquer modalidade.
- Custo: qualidade de reconstrução depende do tokenizer.

**Difusão condicional**:
- Processo de denoising reverso, condicionado por embedding de texto (ou outro sinal).
- Arquitetura típica: U-Net ou DiT com cross-attention para o condicionamento.
- Schedulers (DDPM, DDIM, DPM-Solver, Flow Matching) decidem trajetória de denoising.

**Vision-Language-Action (VLA)**:
- MLLM que produz **tokens de ação** em vez (ou além) de tokens de texto.
- Ação discretizada (bins por dimensão) ou contínua (flow matching, diffusion).
- Embodied / robótica.

## Receita universal (treinar qualquer coisa)

Vale para qualquer arquitetura nova até você ter razão de mudar:

1. **Overfit em 1 batch**. Se o modelo não consegue memorizar um único batch, há bug. Este passo sozinho pega 80% dos erros.
2. **Sanity-check dimensões** (via asserts ou einops). Shape mismatch silencioso é o bug mais comum.
3. **Train curto com dataset pequeno**. Métricas sobem? Loss não explode? Ainda no "papel".
4. **Full train com seed fixo** e logging pesado (loss, grad norm, LR, entropy se aplicável).
5. **Ablations só depois** do baseline estar sólido. Mudar uma coisa por vez.

## Princípios de debugging numérico

- **NaN/Inf em loss** — quase sempre: LR alto, gradiente explodindo, exp/log sem clamp, softmax sem estabilidade, mixed precision em operação sensível.
- **Loss não desce** — primeiro checar LR (alto demais ou baixo demais), depois inicialização, depois normalização. Problema quase nunca é a arquitetura.
- **Loss desce e volta a subir** — LR alto demais no início; adicione warmup ou reduza `lr_max`.
- **Gradient norm oscilando muito** — instabilidade. Diminua LR, aumente clip.
- **Treino estável mas val pior que random** — data leakage ou bug no eval, não no modelo.
- **Determinismo** para reproduzir bug: seed de tudo (random/numpy/torch), desligar benchmark, usar algoritmos determinísticos. Custa 10-30% de velocidade; ligue para debug, desligue para treino final.

## Métricas e avaliação

### Higiene de avaliação

- **Loss ≠ métrica de negócio**. Sempre monitore ambos. Loss pode descer enquanto métrica piora (overfit, calibração ruim, proxy vs verdadeiro objetivo).
- **Train/val/test split** — test **só toca no final**. Se você "tunar" no test, ele virou val.
- **Cross-validation** quando dataset é pequeno; hold-out simples quando é grande.
- **Estratificação** em classes desbalanceadas mantém distribuição entre splits.
- **Splits temporais** em séries temporais/forecasting — nunca aleatório. Futuro vaza em passado trivialmente.
- **Leakage** é o erro mais fatal e mais comum: feature derivada do target, split temporal quebrado, normalização pré-split, duplicações entre train/val (near-duplicates contam), group leakage (mesmo paciente/usuário/clipe em múltiplos splits).
- **Calibração** (ECE, reliability diagram) importa tanto quanto accuracy quando a saída vira decisão com threshold. Temperature scaling resolve 90% dos casos.

### Métricas por tipo de problema

- **Classificação balanceada** — accuracy, log-loss.
- **Classificação desbalanceada** — F1, precision/recall, PR-AUC (melhor que ROC-AUC em desbalanceamento forte).
- **Ranking / retrieval** — NDCG, MAP, MRR, Recall@k.
- **Regressão** — RMSE (penaliza grandes erros), MAE (robusta), MAPE (escala-invariante mas instável perto de zero).
- **Detecção** — mAP em IoU thresholds.
- **Segmentação** — IoU, Dice.
- **Generation (texto)** — perplexity, BLEU/ROUGE (limitadas), BERTScore, benchmarks humanos, LLM-as-judge.
- **Generation (imagem)** — FID, CLIP score, preferência humana.

Regra: **sempre reporte múltiplas métricas**. Uma única métrica é gameável; um conjunto ortogonal expõe trade-offs.

### Estatística de comparação

Diferença de 0.3% entre dois modelos geralmente é **ruído** — seed, ordem de batches, augmentation estocástica. Sem quantificar incerteza, conclusões são anedotais.

- **Múltiplas seeds** — treine cada config 3-5 vezes com seeds diferentes. Reporte média ± std.
- **Bootstrap** para intervalos de confiança na **métrica** — resample (com reposição) o test set K=1000 vezes, computa métrica em cada, percentis 2.5/97.5 viram CI 95%.
- **Paired tests** — quando modelos A e B são avaliados nos mesmos samples, use **paired bootstrap** ou **permutation test**. Muito mais poder estatístico que unpaired.
- **Correção para múltiplas comparações** — se testar 20 modelos, alguns parecerão "estatisticamente melhores" por acaso. Bonferroni ou Benjamini-Hochberg.
- **Effect size > p-value** — p<0.05 com diferença de 0.01% é irrelevante. Sempre reporte tamanho do efeito e CI.

Bootstrap de CI (esqueleto):
```python
import numpy as np
def bootstrap_ci(metric_fn, y_true, y_pred, n=1000, alpha=0.05):
    n_samples = len(y_true)
    scores = []
    for _ in range(n):
        idx = np.random.choice(n_samples, n_samples, replace=True)
        scores.append(metric_fn(y_true[idx], y_pred[idx]))
    return np.percentile(scores, [100*alpha/2, 100*(1-alpha/2)])
```

### Erros silenciosos

- **Dataset shift entre train e prod** — treino em dados limpos, prod em dados sujos. Monitore distribuição de features em produção (Evidently, WhyLogs).
- **Label drift** — definição do label mudou (re-anotação, nova política). Versione labels junto dos dados.
- **Evaluation metric drift** — bug silencioso muda métrica. Snapshot resultados + checksum.
- **Cherry-picking** de seed — se você relatou o melhor de 10 seeds como "nosso modelo", está mentindo (para si, se não para o leitor).

## Princípios que sobrevivem

Se toda biblioteca, framework e modelo nomeado neste documento desaparecer, essas ideias continuam valendo:

- **Gradiente estocástico + LR schedule com warmup** bate quase qualquer alternativa.
- **Normalizar ativações** permite treinos profundos; a variante específica importa menos.
- **Regularização = restrição na capacidade de memorização**. Dados, augmentation, decay, dropout são substitutos parciais.
- **Precisão mista bem-escolhida** é quase grátis; precisão total raramente vale o custo.
- **Paralelismos compõem**: data × tensor × pipeline × sharding. Todo treino grande é uma combinação desses eixos.
- **Modelo generalista + condicionamento** (via tokens, prompts, adapters) supera modelo especializado treinado do zero.
- **Overfit em 1 batch** é o melhor teste barato de sanidade que existe.
- **Bug está nos dados** com frequência desproporcional.
