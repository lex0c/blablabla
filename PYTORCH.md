# PyTorch

Framework dominante em pesquisa desde ~2019 e em produção nova desde ~2022. Ganhou por **ergonomia** (eager por default, dynamic graph, Python-first) — não por superioridade técnica sobre TF/JAX. Hoje tem paridade ou liderança em quase todos os eixos: compile, distributed, quantização, deployment.

Este arquivo é notas de uso sério — não tutorial inicial. Fundamentos de ML em `ML_FOUNDATIONS.md`, panorama de frameworks em `ML_FRAMEWORKS.md`.

## Mental model

- **`Tensor`** é array N-dim com `.device`, `.dtype`, `.requires_grad`, e histórico de operações (se `requires_grad=True`).
- **Autograd** constrói um **DAG dinâmico** no forward. `.backward()` percorre reverso, aplicando regra da cadeia. O grafo é **reconstruído a cada forward** — daí "dynamic graph" e por que loops Python funcionam.
- **`nn.Module`** é container de parâmetros + forward. Convenção, não mágica.
- **`optim`** atualiza pesos com base em `.grad`.
- **`DataLoader`** paraleliza I/O e batching.

Tudo o resto é composição em cima disso.

## Tensor — o que importa

### Criação

```python
torch.zeros(3, 4)                       # CPU, float32
torch.randn(3, 4, device='cuda')        # GPU
torch.arange(10, dtype=torch.int64)
torch.empty(3, 4)                       # uninitialized — cuidado
torch.from_numpy(np_arr)                # zero-copy se dtype bate
tensor.clone()                          # cópia com grad ligado
tensor.detach()                         # mesma memória, sem grad
```

### dtypes e quando cada um

- `float32` (fp32): default, treino clássico.
- `float16` (fp16): metade da memória; range menor, overflow em gradient — usar com `GradScaler`.
- `bfloat16` (bf16): mesma range de fp32, mantissa menor. **Preferido em Ampere/Hopper/H100** — não precisa GradScaler.
- `float8` (fp8, e4m3/e5m2): Hopper+. Treino experimental, quantização de inferência em LLM.
- `int8`, `int4`: quantização de inferência.
- `bool`: máscaras.

**Regra**: treino moderno em bf16 com master weights em fp32 (mixed precision). Inferência em fp16/bf16/int8.

### Device management

```python
t = t.to('cuda')                    # allocate + copy, blocking
t = t.to('cuda', non_blocking=True) # com pin_memory, overlapping com compute
t.pin_memory()                      # CPU mem page-locked, DMA direto
torch.cuda.synchronize()            # barrier — use antes de medir tempo
```

Kernels CUDA são **assíncronos por default**. `print(t)` no meio força sync implícito e mascara perf real.

### Shape/stride/contiguity

Tensor = buffer + shape + stride + offset. View não copia. Após transpose/permute, tensor fica **não-contíguo**:

```python
x.transpose(0,1).view(...)          # ERRO: view exige contíguo
x.transpose(0,1).contiguous().view(...)
x.transpose(0,1).reshape(...)       # reshape faz contiguous se necessário
```

Bugs sutis vêm daqui. `x.is_contiguous()` pra debug.

### Broadcasting

Regras iguais a NumPy. Silencioso e perigoso:

```python
a = torch.zeros(5)       # shape (5,)
b = torch.zeros(5, 1)    # shape (5, 1)
a + b                    # resultado (5, 5) — provavelmente bug
```

`assert a.shape == b.shape` em pontos críticos vale o custo.

## Autograd

### Como funciona

Cada op em tensor com `requires_grad=True` adiciona um nó no **grafo computacional** (`.grad_fn`). `loss.backward()` faz **reverse-mode autodiff**: percorre o grafo do loss até leaves, acumulando gradientes em `.grad`.

```python
x = torch.randn(3, requires_grad=True)
y = (x ** 2).sum()
y.backward()
x.grad  # [2*x[0], 2*x[1], 2*x[2]]
```

### Regras críticas

- **Leaves**: tensores criados pelo user (não resultado de op em tensor com grad). Só leaves recebem `.grad` por default.
- **`.backward()` acumula**. Por isso o loop padrão tem `optimizer.zero_grad()`.
- **Grafo é descartado após backward** por default. `backward(retain_graph=True)` mantém (raro, caro).
- **`.detach()`** quebra o grafo: o tensor continua, mas ops sobre ele não propagam grad.
- **`with torch.no_grad():`** desliga tracking no bloco — use em eval/inferência.
- **`@torch.inference_mode()`** é mais agressivo que `no_grad`: também desliga version counter. Preferir em inferência pura.

### Higher-order gradients

```python
grad_x = torch.autograd.grad(y, x, create_graph=True)[0]
hessian = torch.autograd.grad(grad_x.sum(), x)[0]
```

`create_graph=True` faz o grafo do grad ser diferenciável. Uso: meta-learning, physics-informed NN, HVP.

### Custom autograd

```python
class MyFn(torch.autograd.Function):
    @staticmethod
    def forward(ctx, x):
        ctx.save_for_backward(x)
        return x.sin()
    @staticmethod
    def backward(ctx, grad_out):
        x, = ctx.saved_tensors
        return grad_out * x.cos()

y = MyFn.apply(x)
```

Use quando: op não diferenciável (quantização), precisa truque numérico (straight-through estimator), ou gradiente analítico é muito mais barato que autodiff.

## `nn.Module`

### Idioma

```python
class MLP(nn.Module):
    def __init__(self, dim):
        super().__init__()
        self.fc1 = nn.Linear(dim, dim * 4)
        self.fc2 = nn.Linear(dim * 4, dim)

    def forward(self, x):
        return self.fc2(F.gelu(self.fc1(x)))
```

`nn.Linear`, `nn.Conv2d` etc são Modules também. `self.x = Module()` registra automaticamente em `self._modules` e seus parâmetros aparecem em `.parameters()`.

### `.parameters()` vs `.buffers()`

- **Parameters**: aprendíveis (`requires_grad=True`), incluídos em `optimizer` e `state_dict`.
- **Buffers**: estado persistente não-aprendível (running mean de BN, RoPE cache, positional encoding fixo). `self.register_buffer('name', tensor)`. Vão para `state_dict`, movem com `.to(device)`, mas não são treinados.

### `train()` vs `eval()`

Muda comportamento de Dropout (ativo/passivo) e BatchNorm (usa batch stats / running stats). Sempre explicitar antes de validar/inferir:

```python
model.eval()
with torch.inference_mode():
    preds = model(x)
```

## Training loop idiomático

```python
model = MyModel().to(device)
optimizer = torch.optim.AdamW(model.parameters(), lr=3e-4, weight_decay=0.01)
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=steps)
scaler = torch.amp.GradScaler('cuda')  # se fp16
criterion = nn.CrossEntropyLoss()

for epoch in range(epochs):
    model.train()
    for x, y in train_loader:
        x = x.to(device, non_blocking=True)
        y = y.to(device, non_blocking=True)

        optimizer.zero_grad(set_to_none=True)
        with torch.amp.autocast('cuda', dtype=torch.bfloat16):
            logits = model(x)
            loss = criterion(logits, y)

        # bf16: sem scaler
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        optimizer.step()
        scheduler.step()
```

Notas:

- **`zero_grad(set_to_none=True)`** é o default moderno (pouco mais rápido, semântica igual).
- **bf16**: `autocast` + loss em bf16, sem `GradScaler`. Padrão atual em Ampere+.
- **fp16**: precisa `GradScaler` — escala loss pra cima antes de backward pra evitar underflow; unscale antes de optimizer.step.
- **Gradient clipping** (`clip_grad_norm_`): quase sempre útil em transformers/RNN. Max norm 1.0 é default razoável.

### Gradient accumulation

Simular batch grande com VRAM pequena:

```python
for i, (x, y) in enumerate(loader):
    loss = model(x) / accum_steps
    loss.backward()
    if (i + 1) % accum_steps == 0:
        optimizer.step()
        optimizer.zero_grad(set_to_none=True)
```

Equivalente matematicamente a batch `accum_steps × micro_batch`.

### EMA de pesos

Diffusion, SSL, alguns LLMs:

```python
for p, p_ema in zip(model.parameters(), ema_model.parameters()):
    p_ema.data.mul_(decay).add_(p.data, alpha=1 - decay)
```

Decay típico 0.999-0.9999. Modelo EMA é usado em eval.

## Mixed precision — detalhes

```python
from torch.amp import autocast, GradScaler

# bf16 (Ampere+) — preferido
with autocast('cuda', dtype=torch.bfloat16):
    out = model(x)
    loss = loss_fn(out, y)
loss.backward()
optimizer.step()

# fp16 — precisa scaler
scaler = GradScaler('cuda')
with autocast('cuda', dtype=torch.float16):
    out = model(x)
    loss = loss_fn(out, y)
scaler.scale(loss).backward()
scaler.unscale_(optimizer)              # antes de clip
torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
scaler.step(optimizer)
scaler.update()
```

- **bf16** tem mesma range de fp32, mantissa menor (7 bits). Overflow não acontece na prática; precisão pior em alguns casos (softmax, normalização). Ok pra maioria de DL moderno.
- **fp16** tem range estreita (±65504); loss < ~6e-5 vira zero. GradScaler compensa escalando loss.
- **Master weights em fp32**: `optimizer` mantém cópia fp32 dos pesos, atualiza com grad em fp32 (após unscale), cast para fp16/bf16 no forward. FSDP/DeepSpeed gerenciam isso.

## Memory — otimização

### Gradient checkpointing

Troca compute por memória: forward re-executa em backward em vez de guardar ativações. Redução típica ~sqrt(N) em memória de ativação.

```python
from torch.utils.checkpoint import checkpoint

def forward(self, x):
    x = checkpoint(self.block1, x, use_reentrant=False)
    x = checkpoint(self.block2, x, use_reentrant=False)
    return x
```

`use_reentrant=False` é a API nova (mais rápida, mais composable). Padrão em treinos grandes: checkpoint bloco transformer inteiro.

### Memory snapshot

Debug de OOM:

```python
torch.cuda.memory._record_memory_history(max_entries=100000)
# ... código ...
torch.cuda.memory._dump_snapshot("mem.pickle")
# abrir em pytorch.org/memory_viz
```

Mostra timeline de alocações — identifica leaks e picos.

### Outras técnicas

- **`torch.cuda.empty_cache()`**: não libera pra OS, só pro allocator reusar. Raramente necessário.
- **`del tensor` + loop com `.step()`**: Python GC não libera CUDA imediatamente; referências presas são fonte comum de leak.
- **Activation offload**: mover ativações pra CPU entre forward/backward. FSDP suporta. Custoso em banda PCIe.
- **`torch.cuda.max_memory_allocated()`** pra medir pico.
- **`PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`**: reduz fragmentação em cargas dinâmicas. Default em versões recentes.

## `torch.compile`

Lançado em 2.0 (2023). Compila o modelo pra kernel fundido via **TorchDynamo → TorchInductor → Triton / C++**. Speedups típicos 1.3-2.5× em treino, mais em inferência.

```python
model = torch.compile(model)           # default mode
model = torch.compile(model, mode='reduce-overhead')  # CUDA graphs
model = torch.compile(model, mode='max-autotune')     # autotune kernels
model = torch.compile(model, fullgraph=True)          # erro se Python fallback
```

### Como funciona

1. **TorchDynamo**: Python frame evaluation, intercepta bytecode, captura grafo FX.
2. **AOT Autograd**: extrai grafo forward + backward.
3. **TorchInductor**: lowering pra IR intermediário, fuse ops, gera Triton (GPU) ou C++/OpenMP (CPU).
4. **Graph breaks**: quando Dynamo não consegue capturar (ops dinâmicas, print, Python externo), quebra grafo e cai pra eager nessa parte.

### Modos

- `default` — balanceado.
- `reduce-overhead` — usa CUDA graphs pra eliminar overhead de launch (modelos pequenos, batches pequenos). Mais memória.
- `max-autotune` — autotune de kernels Triton. Compile lento, execução rápida. LLM inference.

### Pegadinhas

- **Primeira execução é lenta** (compile). Subsequentes são rápidas. Aquecer antes de medir.
- **Dynamic shapes**: compile por default assume shapes fixos. `torch.compile(model, dynamic=True)` ou `mark_dynamic` pra dim que varia — senão recompila toda vez.
- **Graph breaks custam**. `fullgraph=True` força erro em vez de quebra; use pra achar e consertar.
- **`torch._dynamo.explain(fn)(args)`** mostra onde quebra.
- **Efeitos colaterais** (print, logging, list.append) causam breaks.

## Distributed

### DDP (DistributedDataParallel)

Padrão pra treino multi-GPU onde modelo cabe em 1 GPU.

```python
# torchrun --nproc_per_node=8 script.py
import torch.distributed as dist
dist.init_process_group('nccl')
local_rank = int(os.environ['LOCAL_RANK'])
torch.cuda.set_device(local_rank)

model = MyModel().to(local_rank)
model = DDP(model, device_ids=[local_rank])

sampler = DistributedSampler(dataset)
loader = DataLoader(dataset, sampler=sampler, batch_size=bs)

for epoch in range(epochs):
    sampler.set_epoch(epoch)  # reshuffle correto
    for x, y in loader:
        ...
```

- Replica modelo em cada GPU. Cada uma vê shard diferente de dados.
- No backward, **allreduce** de gradientes entre GPUs (NCCL). Mesma weight update em todas.
- Escala bem até 8-16 GPUs. Comunicação overlap com compute via buckets.

### FSDP (Fully Sharded Data Parallel)

Modelo grande demais pra 1 GPU. ZeRO-3 style: shard parâmetros, grad e optimizer state entre GPUs. Gather sob demanda no forward/backward.

```python
from torch.distributed.fsdp import FullyShardedDataParallel as FSDP
from torch.distributed.fsdp.wrap import size_based_auto_wrap_policy

model = FSDP(
    model,
    auto_wrap_policy=size_based_auto_wrap_policy,
    mixed_precision=MixedPrecision(param_dtype=torch.bfloat16, ...),
    sharding_strategy=ShardingStrategy.FULL_SHARD,  # ZeRO-3
    device_id=local_rank,
)
```

- **FULL_SHARD (ZeRO-3)**: shard tudo. Max memória economizada, comunicação mais alta.
- **SHARD_GRAD_OP (ZeRO-2)**: shard grad + opt state, params replicados. Intermediário.
- **NO_SHARD (ZeRO-0)**: equivalente a DDP.
- **HYBRID_SHARD**: shard dentro de nó, replica entre nós. Típico em multi-node.

**FSDP2** (em 2.4+) é reescrita com API `fully_shard` + device mesh. Mais limpa, mais performante. Adote se começar novo.

### Tensor / Pipeline parallel

Quando camada individual não cabe. `torch.distributed.tensor.parallel`:

```python
from torch.distributed.tensor.parallel import parallelize_module, ColwiseParallel, RowwiseParallel

parallelize_module(model, device_mesh, {
    "fc1": ColwiseParallel(),
    "fc2": RowwiseParallel(),
})
```

Pipeline parallel via `torch.distributed.pipelining`. Megatron/DeepSpeed ainda dominam produção LLM em escala.

### Device mesh

Abstração N-dim pra compor paralelismos:

```python
mesh = init_device_mesh('cuda', (dp_size, tp_size), mesh_dim_names=('dp', 'tp'))
```

DP ao longo de um eixo, TP ao longo do outro. 3D mesh para DP+TP+PP.

## Checkpointing

### State dict básico

```python
torch.save({
    'model': model.state_dict(),
    'optimizer': optimizer.state_dict(),
    'scheduler': scheduler.state_dict(),
    'step': step,
}, 'ckpt.pt')

# load
state = torch.load('ckpt.pt', map_location='cpu', weights_only=True)
model.load_state_dict(state['model'])
```

**`weights_only=True`** é padrão em 2.6+. Restringe deserialização (bloqueia arbitrary code execution em pickle). Sempre use a menos que saiba o que faz.

### Load parcial

```python
model.load_state_dict(state, strict=False)   # ignora missing/unexpected keys
```

Usado em transfer learning quando head muda.

### DCP (Distributed Checkpoint)

Pra FSDP/multi-node. `torch.distributed.checkpoint` salva shards paralelos, carrega independente do world size:

```python
import torch.distributed.checkpoint as dcp

dcp.save(state_dict=model.state_dict(), checkpoint_id='ckpt/')
dcp.load(state_dict=model.state_dict(), checkpoint_id='ckpt/')
```

Checkpoint treinado em 8 GPUs pode ser carregado em 16. Essencial em cluster.

## Profiling

### `torch.profiler`

```python
from torch.profiler import profile, ProfilerActivity, schedule

with profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    schedule=schedule(wait=1, warmup=1, active=3, repeat=1),
    on_trace_ready=tensorboard_trace_handler('./log'),
    record_shapes=True,
    profile_memory=True,
    with_stack=True,
) as prof:
    for step, batch in enumerate(loader):
        train_step(batch)
        prof.step()
```

Abre em TensorBoard (`tensorboard --logdir log`) ou `chrome://tracing` (`.json`). Mostra timeline de kernels, bottlenecks CPU/GPU, memória.

### Nsight Systems

Mais poderoso pra GPU deep-dive. `nsys profile -o report python train.py`. Abre no Nsight Systems GUI. Timeline de CUDA streams, NCCL, overlap.

### Regras de ouro

- **GPU util em `nvidia-smi dmon` < 80%** = bottleneck em dataloader, sync, ou algum kernel mal fundido. Profile.
- **Kernel launch overhead** domina em modelos pequenos — `torch.compile(mode='reduce-overhead')` ou CUDA graphs manuais.
- **`torch.cuda.synchronize()` antes de cronômetro**. Sem isso tudo é mentira.

## Debugging

### Anomaly detection

NaN/Inf aparecendo no backward?

```python
torch.autograd.set_detect_anomaly(True)
```

Ativa checagens; reporta qual op produziu NaN. **Muito lento** — use só pra diagnosticar, desligar em prod.

### Hooks

Inspecionar ativações/gradientes:

```python
def hook(module, input, output):
    print(module.__class__.__name__, output.shape, output.abs().mean())

h = model.fc1.register_forward_hook(hook)
# ...
h.remove()
```

Backward hook: `register_full_backward_hook`.

### Shape checking

`from torchtyping import TensorType` ou `jaxtyping` — anotações de tipo com shape. Pega dimensão errada em type-check.

### Reproducibilidade

```python
torch.manual_seed(42)
torch.cuda.manual_seed_all(42)
torch.use_deterministic_algorithms(True)   # pode quebrar alguns kernels
torch.backends.cudnn.deterministic = True
torch.backends.cudnn.benchmark = False     # benchmark é não-determinístico
```

Mesmo assim, floating-point non-associativity + NCCL non-determinism tornam bit-perfect reprodução impossível em GPU. Busque "estatisticamente igual", não "idêntico".

## Custom ops e extensions

### C++/CUDA extension

```python
# setup.py
from torch.utils.cpp_extension import CUDAExtension
setup(ext_modules=[CUDAExtension('myop', ['myop.cu', 'myop.cpp'])], ...)
```

`myop.cpp` implementa a op com tensores; registra via `TORCH_LIBRARY`. Chamada de Python como `torch.ops.myop.forward(x)`.

### `torch.library` (moderno)

API limpa pra custom ops com autograd + dispatch correto:

```python
@torch.library.custom_op("mylib::sin_squared", mutates_args=())
def sin_sq(x: torch.Tensor) -> torch.Tensor:
    return x.sin() ** 2

@sin_sq.register_kernel("cuda")
def _(x): ...
```

Integra com `torch.compile`, autograd, FSDP.

### Triton

Escrever kernel GPU em Python-like DSL. Virou padrão pra kernels customizados (FlashAttention, Mamba kernels):

```python
import triton
import triton.language as tl

@triton.jit
def add_kernel(x_ptr, y_ptr, out_ptr, N, BLOCK: tl.constexpr):
    pid = tl.program_id(0)
    offs = pid * BLOCK + tl.arange(0, BLOCK)
    mask = offs < N
    x = tl.load(x_ptr + offs, mask=mask)
    y = tl.load(y_ptr + offs, mask=mask)
    tl.store(out_ptr + offs, x + y, mask=mask)
```

`torch.compile` gera Triton internamente — escrevê-lo manualmente é pra quando você quer squeeze final ou op que Inductor não fusiona.

## Export e deploy

### TorchScript (legado)

`torch.jit.script(model)` ou `torch.jit.trace(model, example)`. Salva `.pt` auto-contido, carrega em C++ (LibTorch). **Legado** desde PyTorch 2.0 — `torch.export` é o sucessor. Não começar novos projetos em TorchScript.

### `torch.export`

API moderna (2.1+). Produz grafo ATen estável serializável:

```python
ep = torch.export.export(model, (example_input,))
torch.export.save(ep, 'model.pt2')
```

Alvo: AOTInductor (AOT compile pra binário), ExecuTorch (mobile/edge), backends de inferência.

### ONNX

Interop com outros runtimes (ONNX Runtime, TensorRT):

```python
torch.onnx.export(model, example, 'model.onnx', dynamo=True, opset_version=17)
```

`dynamo=True` usa TorchDynamo (novo path, 2.1+); legado usava trace. ONNX é padrão em produção multi-framework e edge.

### ExecuTorch

Runtime mobile/edge da Meta (sucessor de PyTorch Mobile). Pipeline: `torch.export` → backend-specific (XNNPACK, CoreML, Qualcomm, MediaTek) → `.pte`. Rodando em Llama mobile, Wearables.

### Serving

- **TorchServe** — oficial PyTorch. Nunca ganhou tração grande; maioria migrou.
- **vLLM, Triton Inference Server** (NVIDIA), **TGI** (HF) — dominam LLM serving.
- **LibTorch** (C++) — se precisa embedar em binário nativo sem Python.
- **ONNX Runtime** — deploy cross-platform.

Para inferência LLM séria: vLLM sobre PyTorch. Para edge: ExecuTorch ou ONNX Runtime.

## Pitfalls (catálogo)

### Autograd

- **In-place ops + autograd**: `x += 1` em tensor com grad pode corromper grafo. `x.add_(1)` idem. Use versões não-in-place se estiver no meio de computação diferenciável.
- **`.data` vs `.detach()`**: `.data` bypassa autograd **sem versionamento** — pode produzir silently wrong grad. **Nunca use `.data`** em código novo; sempre `.detach()`.
- **Esquecer `optimizer.zero_grad()`**: grads acumulam entre steps. Loss cresce misteriosamente.
- **`loss.item()` dentro de loop crítico**: força sync CPU-GPU. Acumule tensor, `.item()` no final.
- **Chamar `.backward()` duas vezes sem `retain_graph=True`**: erro "Trying to backward through the graph a second time".

### Memória

- **Lista acumulando tensors com grad**: `losses.append(loss)` → grafo inteiro preso em memória. Use `losses.append(loss.item())` ou `.detach().cpu()`.
- **`torch.save(model)`** (sem `.state_dict()`): pickle do objeto Python inteiro — frágil a refactor de classe. Sempre `state_dict()`.
- **`map_location` esquecido no load**: tentando carregar checkpoint CUDA em máquina CPU-only. Usar `map_location='cpu'`.

### DataLoader

- **`num_workers > 0` + fork + CUDA**: worker herda estado CUDA não-fork-safe. Use `multiprocessing_context='spawn'` ou não inicialize CUDA antes do worker.
- **Random state em workers**: cada worker tem mesmo seed se não for setado. `torch.utils.data.get_worker_info()` + seed por worker.
- **Dataset reads arquivos no `__init__`**: RAM explode com `num_workers`. Lazy load em `__getitem__`.
- **`pin_memory=True` sem `non_blocking=True`**: metade do ganho só.

### Distributed

- **Diferentes shapes entre ranks**: DDP/FSDP exigem mesma estrutura. Padding ou drop_last.
- **`BatchNorm` + DDP**: cada rank vê só sua shard, stats incorretas. Use `SyncBatchNorm` ou substitua por LayerNorm/GroupNorm.
- **Salvar checkpoint em todos os ranks**: só rank 0 escreve, outros aguardam `dist.barrier()`.
- **NCCL timeout**: rank lento (dataloader preso, GPU diferente) trava allreduce. `TORCH_NCCL_ASYNC_ERROR_HANDLING=1` + timeout customizado.

### Determinismo

- **`use_deterministic_algorithms(True)`** lança erro em op sem versão determinística. Ative antes de benchmark sério.
- **cuBLAS non-determinism** em matmul: `CUBLAS_WORKSPACE_CONFIG=:4096:8` mitiga.

## Ecosistema prático

- **Lightning / Fabric** — tira boilerplate de loop; Fabric é a versão minimal (só device management + distributed).
- **HuggingFace Accelerate** — abstração sobre DDP/FSDP/DeepSpeed pra não reescrever loops.
- **HuggingFace Transformers** — modelos pré-treinados + loop (Trainer) + peft + trl.
- **timm** — biblioteca de visão.
- **torchmetrics** — métricas distribuídas corretas (calcula em ranks + allreduce).
- **torchtune** — fine-tuning nativo de LLM em PyTorch, sem HuggingFace.
- **FSDP2, DeepSpeed, Megatron-LM** — treino em escala.
- **Hydra + wandb** — config + tracking.

## Versões e futuro

- **2.0 (2023)**: `torch.compile` chega.
- **2.1**: `torch.export` começa.
- **2.3+**: FSDP2, compile maturity, fp8 training.
- **2.4+**: DTensor maduro, device mesh, compile em larga escala.
- **2.5-2.6**: AOTInductor, torch.compile em modelos enormes, `weights_only=True` default.

Direção clara: unificar compile + export + distributed sob `torch.compile` + `torch.export` + DTensor, com eager como path de pesquisa/debug e compile como path de produção.

## Princípios

1. **Eager pra desenvolver, compile pra produção**. Não brigue com `torch.compile` no começo do projeto.
2. **bf16 é o novo default**. fp32 só em lugares que sabidamente precisam precisão; fp16 só em hardware pré-Ampere.
3. **Pequenos ganhos de perf são ruído; profile antes de otimizar**. `torch.profiler` existe pra isso.
4. **`state_dict`, não `torch.save(model)`**. Pickle de objeto é bomba-relógio.
5. **Em distributed, trate rank 0 como especial** (print, save, log). Evita race conditions.
6. **Autograd é dinâmico — mas custa memória**. Use `no_grad`/`inference_mode` em tudo que não treina.
7. **Grafo é invisível; bugs são sutis**. Quando algo está estranho, imprima shape, dtype, device, `requires_grad` antes de tudo.
8. **PyTorch é ergonomia + ecossistema**. Aproveita — não reinvente dataloader, profiler, distributed.

## Recursos

- **Docs oficiais** — `pytorch.org/docs`. Tutoriais atualizados.
- **PyTorch Developer Podcast** — Edward Yang explica internals.
- **"Deep Learning with PyTorch"** — Stevens, Antiga, Viehmann (livro; introdutório mas sólido).
- **`torch.compile` deep dive** — posts no blog oficial (Horace He, Edward Yang).
- **`pytorch-labs/` repos** — experimentos oficiais (gpt-fast, segment-anything-fast, float8-experimental).
- **ezyang's blog** — `blog.ezyang.com` — internals e decisões de design.
- **Discussão técnica** — `dev-discuss.pytorch.org`.
