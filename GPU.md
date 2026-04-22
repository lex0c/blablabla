# GPU — Arquitetura

GPUs (Graphics Processing Units) evoluíram de aceleradores gráficos fixos (1990s) para **máquinas massivamente paralelas de propósito geral**, hoje núcleo de graphics, HPC, e — em escala sem precedentes — treinamento e inferência de IA. Este arquivo foca em **arquitetura**; `GPGPU.md` cobre programação (CUDA, OpenCL, ROCm, etc.).

Complementa `COMPUTER_ARCHITECTURE.md`, `GPGPU.md`, `AI.md`, `EGPU.md`, `FPGA.md`.

## Filosofia de Design: CPU × GPU

**CPU**: poucos cores complexos otimizados para latência de tarefa serial — branch prediction, OoO, grandes caches, tudo para que **um thread** vá rápido.

**GPU**: milhares de cores simples otimizados para **throughput** — executam **muitas threads** em lockstep, escondendo latência de memória com paralelismo em vez de cache.

Métrica definidora: CPU otimiza CPI (cycles per instruction); GPU otimiza IPC agregado × throughput de memória.

## Hierarquia de Execução

Vocabulário Nvidia (AMD e Intel têm paralelos):

- **Thread**: unidade lógica — uma execução do kernel.
- **Warp** (Nvidia) / **Wavefront** (AMD): 32 (ou 64 AMD antigo) threads executando em lockstep (SIMT).
- **Block** (CUDA) / **Work-group** (OpenCL): conjunto de threads que compartilham memória rápida (shared mem) e podem se sincronizar.
- **Grid** / **NDRange**: coleção de blocks.

**SIMT (Single Instruction, Multiple Threads)**: todas as threads de um warp executam a mesma instrução por ciclo. Branch divergence dentro do warp serializa caminhos — grande penalidade.

## Streaming Multiprocessor (SM) — Nvidia

Unidade fundamental de computação. Em H100:

- **4 partições** por SM, cada com:
  - 16 INT32 units.
  - 32 FP32 units.
  - 16 FP64 units.
  - 1 Tensor Core.
- **Shared memory / L1 cache**: ~256 KB configurável.
- **Registradores**: 64K × 32-bit (por SM).
- **Warp schedulers**: elegem warps prontos para executar.

GPU tem dezenas a centenas de SMs: H100 tem 132 SMs (114 ativos em modelos shipping), B200 ~160+.

### CUDA Cores vs Tensor Cores

- **CUDA Cores**: FP32/INT32 escalar.
- **Tensor Cores**: operações de matriz (MMA). Em H100, 4º gen: FP8/FP16/BF16/TF32/FP64. Dominam throughput de ML — treinar LLM sem tensor cores é inviável.
- **RT Cores**: ray-tracing BVH traversal + intersection. Usados em graphics e, recentemente, ML (oclusão).

## AMD — CUs e RDNA/CDNA

- **Compute Unit (CU)** equivale a SM. Em RDNA3: 2 SIMDs de 32 lanes, 4 escalares, scheduler.
- **RDNA**: arquitetura gaming (Radeon RX).
- **CDNA**: arquitetura data-center (Instinct MI series — MI300X tem 304 CUs).
- **Matrix Cores** (CDNA): equivalentes a Tensor Cores.

## Intel — Xe

- **Xe-LP** (low-power, iGPU moderna), **Xe-HPG** (Arc gaming), **Xe-HPC** (Ponte Vecchio, Rialto Bridge, Falcon Shores).
- **Vector Engines, Matrix Engines** (XMX — equivalentes a Tensor Cores).
- **Tiles / chiplets** em Xe-HPC (Intel tem experiência pioneira em tiled GPU).

## Apple — Unified Memory GPU

- **Apple Silicon GPUs**: integradas em M-series (M1/M2/M3/M4 families).
- **Memória unificada (UMA)**: CPU e GPU compartilham LPDDR5X. Zero cópia, baixo overhead.
- **Metal**: API nativa; também CUDA-like via MLX (ML research framework).
- Scaling: M4 Max tem até 40 GPU cores; M4 Ultra (quando lançado) dobra.

## Memória

### Hierarquia

| Nível | Latência | Bandwidth | Escopo |
|---|---|---|---|
| Registers | ~0 | — | thread |
| Shared memory / L1 | ~25 ns | TB/s | block |
| L2 cache | ~100 ns | | SM ↔ SM |
| HBM / GDDR (global) | ~500 ns | ~1-8 TB/s | chip inteiro |
| Host DRAM via PCIe/NVLink | ~µs | ~64 GB/s (PCIe 5 x16) | fora |

### HBM (High Bandwidth Memory)

- Stacks 3D de DRAM dies conectados via TSV (through-silicon via) a um interposer junto ao GPU die.
- **HBM2e, HBM3, HBM3E**: 3-8 TB/s em stacks múltiplos. H100: 3 TB/s; B200: ~8 TB/s.
- Capacidade por stack: 24-36 GB; GPU tem 80-192 GB típico DC.
- Custo alto; domina custos de fabricação em GPUs DC.

### GDDR (G-DDR)

- Memória consumer — RTX 4090 usa GDDR6X a ~1 TB/s.
- Mais barata que HBM, menor banda por pin.
- GDDR7 chega em 2024-2025.

### Memory Coalescing

Threads de um warp acessando memória global contígua são coalescidos em **poucas transações**. Acessos espalhados geram dezenas de transações. Essencial tunear algoritmos para coalescing.

### Shared Memory / Bank Conflicts

Shared memory é dividida em **banks**. Acessos simultâneos ao mesmo bank por threads do warp serializam (bank conflict). Padrões padded / transpose solvem.

## Interconnect

### NVLink

Link proprietário da Nvidia, GPU ↔ GPU e CPU (Grace).

| Geração | Per-link | Total agregado (H100/B200) |
|---|---|---|
| NVLink 3 | 25 GB/s | 600 GB/s |
| NVLink 4 (H100) | 50 GB/s | 900 GB/s |
| NVLink 5 (B200) | 100 GB/s | 1.8 TB/s |

**NVSwitch**: trocador all-to-all em nó com 8 GPUs (HGX / DGX).

### PCIe

Para host ↔ GPU em absência de NVLink. PCIe 5.0 x16 = 64 GB/s unidir.

### AMD Infinity Fabric / xGMI

Equivalente AMD. MI300X tem 896 GB/s por Infinity Fabric.

### CXL

Crescente; CXL 3.0 permite pool coerente de memória. Fronteira em 2026.

## Escalabilidade de Cluster

- **1 GPU**: ~10-100 TFLOPS FP32 em chips modernos; 1-5 PFLOPS FP8 Tensor.
- **Nó (8 GPUs HGX)**: ~8 PFLOPS FP16 típico.
- **Rack / pod**: dezenas a centenas de GPUs via InfiniBand HDR/NDR, Ethernet 400G, NVLink switch (NVL72 tem 72 GPUs em single domain NVLink).
- **Supercomputer**: milhares a dezenas de milhares. Frontier, El Capitan. Treinamento de modelos 100B+ parâmetros exige esse nível.

**Paralelismos** em ML (ver `AI.md` para detalhes):

- **Data parallelism**: mesmo modelo, batches diferentes.
- **Tensor parallelism**: tensor split em dimensões.
- **Pipeline parallelism**: camadas em GPUs diferentes.
- **Expert parallelism**: MoE experts em GPUs diferentes.
- **Sequence parallelism**: sequência split.
- Combina-se 3D/4D parallelism em LLM training.

## Precisões

GPUs modernas suportam várias:

| Formato | Bits | Uso |
|---|---|---|
| FP64 | 64 | HPC científico (simulação) |
| FP32 | 32 | rendering tradicional, ML de alta precisão |
| TF32 | 19 (exp8, mant10) | mixed-precision ML; Tensor Cores |
| BF16 | 16 (exp8, mant7) | treinamento ML moderno |
| FP16 | 16 (exp5, mant10) | inference, training |
| FP8 (E4M3/E5M2) | 8 | LLM training/inference moderno |
| INT8, INT4 | 8, 4 | quantização de inference |
| FP4 (B200+) | 4 | inference ultra-comprimido |
| Binary/Ternary | 1-2 | edge |

**Trade-off**: menos bits → mais throughput (Tensor Cores FP8 ~2x FP16), menos memória, menor precisão. LLM inference em produção usa FP8/INT8 ou menos.

## Chips Representativos (2024-2026)

### Nvidia

| Chip | Uso | HBM | FP8 (sparse) | NVLink |
|---|---|---|---|---|
| A100 (2020) | DC | 40/80 GB HBM2e | — | 600 GB/s |
| H100 (2022) | DC | 80 GB HBM3 | ~4 PFLOPS | 900 GB/s |
| H200 | DC | 141 GB HBM3e | ~4 PFLOPS | 900 GB/s |
| B100/B200 (2024) | DC | 192 GB HBM3e | ~9/14 PFLOPS | 1.8 TB/s |
| GB200 Grace Blackwell | DC | Grace+2 B200 | — | NVLink-C2C |

**Consumer**: RTX 4090, 5090. FP32 alto, HBM ausente (GDDR).

### AMD

- **MI250X**: chiplet; predecessor.
- **MI300X**: 192 GB HBM3, 5.3 TB/s bandwidth, FP8 2.6 PFLOPS. Competindo com H100.
- **MI325X, MI350**: próximas.

### Intel

- **Gaudi 2, 3** (Habana): ASIC ML-focado.
- **Ponte Vecchio** em Aurora (supercomputer).
- **Falcon Shores**: próxima.

### Especialistas

- **Google TPU v5p, v6**: ASIC treinamento interno Google.
- **AWS Trainium, Inferentia**: idem.
- **Cerebras Wafer Scale**: 1 wafer inteiro = 1 chip. 850k cores.
- **Groq LPU**: inferência deterministic latency.
- **SambaNova, Tenstorrent, Graphcore**: alternativos.

## Pipeline Gráfico (Rápido)

Contexto histórico e ainda relevante em rendering:

1. **Vertex shading**: transform vertices.
2. **Tessellation** (opcional).
3. **Geometry shading** (opcional).
4. **Rasterization**: triângulos → pixels.
5. **Fragment (pixel) shading**: cor por pixel.
6. **Raster operations**: depth test, blending, output.

APIs modernas: **DirectX 12, Vulkan, Metal, WebGPU**. "Next-gen" (D3D12, Vulkan) expõe controle fino — redução de overhead vs OpenGL/D3D11.

### Ray Tracing

BVH (Bounding Volume Hierarchy) acelerado em hardware (RT Cores). Reflexos, sombras, GI mais realistas — custo grande, DLSS/FSR/XeSS upscalam.

### Upscaling (DLSS, FSR, XeSS)

ML upscaling de frame renderizado em resolução menor → resolução final. DLSS (Nvidia, ML proprietário), FSR (AMD, open, algoritmos híbridos), XeSS (Intel, ML em XMX).

## Recursos Gráficos Modernos

- **Mesh shaders**: substituem vertex+geometry; mais flexíveis.
- **Variable Rate Shading (VRS)**: shade menos em áreas menos importantes.
- **DirectStorage**: NVMe → GPU direto, bypassando CPU.
- **Sampler Feedback Streaming**, **SFS**: streaming de textura bajo demanda.

## Virtualização de GPU

- **SR-IOV**: Nvidia vGPU, AMD MxGPU.
- **MIG (Multi-Instance GPU)**: A100/H100 divide em até 7 instâncias isoladas.
- **Container runtime**: Nvidia Container Toolkit, AMD ROCm container.
- **Time-slicing**: múltiplas workloads em uma GPU (sem isolamento hardware).

Usado em DC para compartilhar GPU DC cara.

## Power e Cooling

- **TDP consumer**: 200-600W (RTX 5090 ~575W).
- **DC GPUs**: 350-1400W (B200 ~1000W, B200 em SXM ~1200W).
- **Liquid cooling**: mandatório em DC moderno. Direct-to-chip, immersion.
- **PSU** dimensionamento: margem 30-50% acima do pico.
- **PCIe 5 connector (12VHPWR, 12V-2x6)**: 600W. Polêmicas em consumer por adaptadores derretendo — PCB + qualidade de inserção críticos.

## Benchmarks

- **Gaming**: 3DMark, UL, FPS real em engines.
- **HPC**: HPL (LINPACK), HPCG, MLPerf.
- **ML training**: MLPerf Training.
- **Inference**: MLPerf Inference, TRT-LLM benchmarks.
- **Synthetic**: stress em floating-point peak.

Cuidado: TFLOPS peak (datasheet) raramente se alcança em workload real. Utilization prático de 30-70% é comum em LLM training bem-tunado.

## Software Stack

- **Nvidia**: CUDA, cuDNN, cuBLAS, TensorRT, NCCL, Nsight. Ver `GPGPU.md`.
- **AMD**: ROCm, HIP (CUDA-like), MIOpen, rccl.
- **Intel**: oneAPI, SYCL, oneDNN.
- **Vendor-neutral**: SYCL, OpenCL, Kokkos, RAJA.
- **ML frameworks**: PyTorch, TensorFlow, JAX — abstraem backend.
- **Graphics**: Vulkan, Metal, DirectX, WebGPU.

## Quantização e Compressão em Inference

Para rodar LLM em hardware limitado:

- **FP8 (E4M3/E5M2)**: LLM serving DC.
- **INT8**: clássico; GPTQ, AWQ algoritmos.
- **INT4**: menor qualidade; viável em 4-8B param models.
- **BitNet / 1-bit**: binary/ternary — promessas ainda em pesquisa.
- **KV cache**: dominante em memória em LLM; quantização do KV-cache prolonga contexto.

## Segurança

- **Spectre-like em GPU**: pesquisa demonstrou (LeftoverLocals, CVE-2023-4969) — dados residuais em shared memory entre kernels. Mitigações em drivers.
- **Confidential GPU** (Nvidia H100 confidential computing): GPU memory cifrado, atestação.
- **Firmware updates** obrigatórios — várias CVEs em drivers/firmware anualmente.

## Tendências 2025-2026

1. **LLM-driven demand**: data center GPU é mercado maior que consumer.
2. **Chiplets e packaging avançado**: CoWoS, TSMC estrangula supply.
3. **Custom silicon** (TPU, Trainium, Maia): grandes clouds verticalizando.
4. **Low-precision** (FP4, FP6) mainstream.
5. **Memory limits**: modelos grandes dependem de HBM + scale-out; software mitiga (MoE, quantization, flash attention).
6. **Power and cooling** dominantes em planning de DC; liquid mandatório.
7. **Nvidia Rubin / Blackwell Ultra, AMD MI400, Intel Falcon Shores**: próxima onda.

## Recursos

- **"Professional CUDA C Programming"** — Cheng et al.
- **"Programming Massively Parallel Processors"** — Kirk & Hwu.
- **Nvidia Architecture whitepapers** (A100, H100, B200).
- **AMD CDNA whitepapers**.
- **chipsandcheese.com, AnandTech, Tom's Hardware** deep dives.
- **HotChips, SIGGRAPH, GTC** conferências.
- **Horace He, Tim Dettmers** blogs em GPU/ML.

## Princípios

1. **Throughput > latência**: GPU paga latência de memória com paralelismo.
2. **Coalescing é obrigatório**. Acesso espalhado destrói performance.
3. **Branch divergence custa**. Threads de um warp no mesmo caminho.
4. **Precisão baixa é superpoder**. FP8/INT8 em inference dobra/quadruplica throughput.
5. **Memory é o gargalo dominante**. FLOPS cresce mais que bandwidth; arithmetic intensity importa.
6. **Tensor cores são o esteróide**: escrever kernel que os usa é 10x vs CUDA cores puros.
7. **Scaling é multi-dimensional**: 1 GPU, 1 nó, 1 rack, 1 DC — stack de comunicação importa em cada nível.
