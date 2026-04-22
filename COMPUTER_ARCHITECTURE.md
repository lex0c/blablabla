# Arquitetura de Computadores

Arquitetura de computadores é o estudo de como **hardware executa código** — da instrução assembly até o circuito lógico. Para engenheiros de software, entender arquitetura é o que separa otimização real de superstição: latência de cache, *false sharing*, TLB misses, branch misprediction não aparecem em perfis casuais, mas dominam performance de código moderno.

## Camadas

Computação moderna é pilha de abstrações:

1. **Aplicação** (Python, JS, etc.)
2. **Runtime / bytecode** (JVM, V8, .NET CLR, CPython)
3. **Linguagem compilada** (C, Rust, Go → máquina)
4. **ISA (Instruction Set Architecture)**: contrato hardware-software. x86-64, ARMv8, RISC-V.
5. **Microarquitetura**: como a ISA é realmente executada (Apple M4, Intel Raptor Lake, AMD Zen 5). Duas CPUs da mesma ISA podem ter microarquiteturas radicalmente diferentes.
6. **Circuitos lógicos**: portas, flip-flops, buses.
7. **Nível físico**: transistores, interconect, silício.

Hennessy & Patterson categorizam: ISA + organização + hardware = arquitetura.

## Desempenho — Equação Central

```
Tempo/programa = Instruções × CPI × Tempo_ciclo
                        ↑         ↑        ↑
                    compilador  microarch freq
```

- **Instruções/programa**: algorithm + compilador.
- **CPI (cycles per instruction)**: microarquitetura (pipeline, cache, branch).
- **Tempo de ciclo**: tecnologia (processo em nm, projeto físico).

Melhorias em cada dimensão geram ganho. Pôr foco errado = ROI ruim.

**Lei de Amdahl**: `speedup = 1 / ((1-p) + p/s)`. Parte não paralelizada limita ganho. Ver `PROFILING.md`.

## ISA — Instruction Set Architecture

### Estilos

- **CISC**: muitas instruções, comprimento variável, *microcode*. x86-64.
- **RISC**: instruções simples, fixed-length, load-store. ARM, RISC-V, MIPS, POWER.
- **VLIW**: paralelismo explícito no bundle. Itanium (falhou), alguns DSPs.
- **Stack-based**: JVM bytecode, WASM (tradicionalmente).

Distinção acadêmica — hoje x86-64 decodifica em µops RISC-like internamente.

### x86-64

- **Legado acumulado**: 8086 (1978) → 386 → AMD64 (2003). Modes: real, protected, long.
- **Registers**: 16 GPRs de 64-bit (RAX-R15), 16-32 SIMD (XMM/YMM/ZMM).
- **SIMD extensions**: SSE, SSE2-4, AVX, AVX2, AVX-512, AMX.
- **Modes de endereçamento** complexos (`[base+index*scale+disp]`).
- **Backward compat** histórica — instruções de 40 anos ainda funcionam.

### ARM (ARMv8-A, v9-A)

- **AArch64**: 31 GPRs 64-bit + SP. NEON (SIMD 128-bit), SVE/SVE2 (vector length agnostic), SME (matrix extensions).
- **TrustZone**: secure world.
- **Pointer Authentication (PAC)**, **Branch Target Identification (BTI)**, **MTE (Memory Tagging)**.
- Dominante em mobile, crescente em server (AWS Graviton, Ampere, Nvidia Grace) e laptop (Apple Silicon).

### RISC-V

- Open ISA — sem royalty, extensível.
- Base (RV32I, RV64I) + extensions (M multiply, A atomic, F/D floating, V vector, B bit manip).
- Cresce em embedded (SiFive, WCH), em pesquisa, e agora data-center (Ventana, Tenstorrent).

### POWER / OpenPOWER

- IBM mainframes/servers. POWER10 atual.
- Nicho: HPC, IBM z/AIX, mainframes.

### ISA escolha prática

- **Desktop/server commodity**: x86-64.
- **Mobile/laptop moderno**: ARMv8/v9.
- **Embarcado baixo custo**: RISC-V emergente, ARM Cortex-M dominante.
- **Consoles / Apple**: ARM Apple Silicon.

## Pipeline

Executar estágios de instrução simultaneamente em diferentes instruções.

**Estágios clássicos** (RISC 5-stage):

1. **IF** (Instruction Fetch).
2. **ID** (Decode + Register Read).
3. **EX** (Execute / ALU).
4. **MEM** (Memory access).
5. **WB** (Write Back).

CPUs modernas: 14-20+ estágios.

### Hazards

- **Structural**: dois estágios querem mesmo recurso.
- **Data**: instrução depende de resultado ainda não pronto. Mitigação: **forwarding** / **bypass**.
- **Control**: branch indefinido até decode/execute. Mitigação: **branch prediction**.

### Superscalar

Múltiplas instruções por ciclo. Largura típica: 4-10 µops/ciclo em CPUs modernas (Apple M4 ~10+, Zen 5 ~6-8).

### Out-of-Order (OoO)

Instruções executam fora de ordem conforme operandos ficam prontos. **Reorder Buffer (ROB)** mantém commit em ordem de programa.

Componentes: dispatch, issue queue, physical registers, reservation stations, retirement.

### Speculative Execution

CPU executa especulativamente além de branches não resolvidos; descarta se palpite errado. Alta performance — e origem de **Spectre / Meltdown** (ver `CRYPTO_ADVANCED.md`).

### Register Renaming

ISA tem ~30 registers arquiteturais; CPU tem centenas de *physical registers*. Renaming elimina false dependencies (WAR, WAW). Essencial para OoO.

### SMT / Hyperthreading

Um core executa múltiplos threads simultâneos, compartilhando ALUs ocultas quando um thread esperava cache miss. Tipicamente 2-way (Intel HT); Power com 8-way. Ganho 10-30% em workloads mistos.

## Hierarquia de Memória

Pilar do desempenho moderno — **memória é lenta; cache é a cola**.

### Latências Típicas (2026)

| Nível | Tamanho | Latência | Bandwidth |
|---|---|---|---|
| Register | ~200 bytes | 0 ns | ∞ |
| L1 cache | 32-192 KB | ~1 ns (3-5 ciclos) | TB/s/core |
| L2 cache | 256 KB-2 MB | ~3-5 ns (10-15 ciclos) | |
| L3 cache | 8-128 MB | ~10-30 ns (30-70 ciclos) | |
| DRAM | 16-2048 GB | ~70-100 ns (200-300 ciclos) | ~50-500 GB/s |
| SSD NVMe | TBs | ~10-100 µs | ~3-14 GB/s |
| HDD | TBs | ~5-10 ms | ~200 MB/s |
| Network (LAN) | — | ~100 µs-1 ms | ~1-100 GB/s |

**Ponto**: cache miss em L3 → DRAM = ~300 ciclos. Durante isso, CPU podia ter executado ~1000 µops. Cache miss é **o** assassino de performance oculto.

### Caches

**Organização**:

- **Linhas** (típico 64 bytes). Unidade de transferência.
- **Associatividade**: direct-mapped, N-way set associative, fully associative.
- **Políticas**: LRU-approximated, pseudo-random.
- **Writeback vs write-through**.

**Inclusividade**:
- **Inclusive**: L2 contém todo L1. Fácil coerência.
- **Exclusive**: L1 e L2 disjuntos. Capacidade efetiva maior.
- **Non-inclusive/non-exclusive**: Intel Skylake-SP em diante.

### Prefetchers

Hardware adivinha padrão de acesso e pré-carrega. Funcionam bem em sequencial e strided; ruins em ponteiros aleatórios. Prefetch manual (`__builtin_prefetch`) ajuda em casos específicos, estraga em outros.

### TLB

**Translation Lookaside Buffer**: cache de traduções virtuais → físicas. Páginas de 4KB dominantes; **huge pages** (2MB, 1GB) reduzem pressão em TLB para heaps grandes. `madvise(MADV_HUGEPAGE)`, `transparent_hugepage`.

### Coerência de Cache

Múltiplos cores, mesmas linhas — precisa protocolo:

- **MESI**: Modified, Exclusive, Shared, Invalid.
- **MOESI, MESIF**: variantes com estado "Owned" / "Forward".
- Mensagens entre caches custam — **contenção em linhas compartilhadas** é gargalo clássico.

### False Sharing

Duas variáveis (escritas por threads distintos) caem na mesma linha. Threads invalidam linha um do outro a cada escrita. Solução: alinhar/padear para linhas separadas (`alignas(64)`, `#[repr(align(64))]`).

### Memory Ordering

Arquiteturas diferem em **quão agressivamente** reordenam operações de memória:

- **x86-64**: TSO (Total Store Order). Stores visíveis em ordem; loads podem passar à frente de stores próprios.
- **ARM, POWER**: weakly ordered. Exigem `acquire`/`release`/`fence` explícitos.
- **Linguagens** (C++, Rust): modelos formais (SC, acquire-release, relaxed). Ver `CONCURRENCY.md`.

## DRAM — Detalhes

### Organização

- **Canal** (channel) → **DIMM/rank** → **chips** → **banks** → **rows** → **columns**.
- Comando: **activate row** (~15 ns), **read/write column** (~15 ns), **precharge** (~15 ns).
- **Row buffer** cacheia última linha ativada — acessos consecutivos à mesma row são rápidos.

### Timings

`CL-tRCD-tRP-tRAS` em clocks. "DDR5-6400 CL32" ≈ 10 ns CAS.

### Refresh

DRAM precisa ser refrescada periodicamente (~64 ms janela). Refresh rouba ~5% da banda.

### Gerações

- **DDR4**: 1600-3200 MT/s, 64-bit.
- **DDR5**: 4800-8400+ MT/s, duas 32-bit subchannels por DIMM, ECC on-die.
- **LPDDR5/5X**: mobile, baixo consumo.
- **HBM3/3E**: stacks, muito paralelismo, 1+ TB/s. Usado em GPUs (ver `GPU.md`) e aceleradores.
- **GDDR6/6X/7**: GPU consumer.

### CXL

**Compute Express Link**: pool de memória coerente sobre PCIe 5.0/6.0. Permite desagregar memória — host acessa memória remota com latência ligeiramente maior (~170 ns vs ~100 ns local). Usos: expansão barata, memória compartilhada entre hosts, accelerator coherent memory.

## NUMA — Non-Uniform Memory Access

Sistemas multi-socket / chiplet: memória é local a um socket/die, remota para outros. Latência e banda diferem (2-3x em pior caso).

**Implicações**:

- **Afinidade**: prender thread + memória ao mesmo nó (`numactl`, `taskset`, pthread affinity).
- **First-touch policy** (Linux): página alocada fisicamente no nó onde primeiro foi tocada.
- **Interleaved allocation**: `numactl --interleave=all` espalha — menor latência pior, melhor banda agregada.
- **Chiplets (AMD, Intel Sapphire Rapids tiles)**: NUMA em dentro de socket ("sub-NUMA clustering", SNC).

## Branch Prediction

Moderno é altíssimo acerto (98-99% em código bem-comportado).

- **Static**: "backward branches tomadas" (loops).
- **Dynamic**: história local + global. BTB (Branch Target Buffer), RAS (Return Address Stack).
- **TAGE, perceptron**: predictores modernos usam ML-like.
- **Custo de miss**: pipeline flush — ~15-25 ciclos perdidos.

**Branchless code** (cmov, seleção sem branch) às vezes é mais rápido; às vezes mais lento (depende de predictability).

## SIMD

Single Instruction, Multiple Data. Uma instrução opera em vetor.

- **x86**: SSE (128-bit), AVX/AVX2 (256-bit), AVX-512 (512-bit). AMX (tile-based matrix).
- **ARM**: NEON (128-bit), SVE/SVE2 (length-agnostic, 128-2048 bit), SME.
- **RISC-V V extension**: vector-length agnostic, inspirado em Cray.

**Autovec**: compiladores vetorizam loops simples automaticamente. Casos complexos exigem intrinsics ou libs (highway, xsimd, Eigen).

**Cuidado**: `throttling` de AVX-512 em Intel antigo (clock baixava para evitar termal); reduzido em recentes.

## Power e Clock

### DVFS

**Dynamic Voltage and Frequency Scaling**. OS/firmware muda P-states conforme carga. Governor (Linux): `performance`, `powersave`, `schedutil`.

`P ≈ C · V² · f` — dobrar frequência dobra potência; dobrar tensão quadruplica.

### Turbo Boost / Precision Boost

Clock acima do nominal quando térmico + power permitem. TDP especificado é média; PL1/PL2 (Intel) definem janelas curtas.

### Throttling

- **Térmico**: chip passa Tjunction max → reduz clock.
- **Power**: envelope PL1 violado → cap.
- **Current**: excessos de corrente em VRM.

### E-cores vs P-cores

Heterogêneo: big.LITTLE (ARM), hybrid (Intel Alder Lake+), Apple Silicon. OS scheduler precisa escolher (Thread Director em Intel, quality-of-service em Apple).

## Barramentos e I/O

### PCIe

**Peripheral Component Interconnect Express**. Serial, per-lane. Bandwidth por lane (unidirecional):

| Gen | Per-lane |
|---|---|
| 3.0 | 1 GB/s |
| 4.0 | 2 GB/s |
| 5.0 | 4 GB/s |
| 6.0 | 8 GB/s (PAM4) |
| 7.0 | 16 GB/s |

Slots x1/x4/x8/x16 multiplicam. Lanes vêm da CPU (24-48 típico desktop) + chipset (via DMI/link).

### DMI / IF

- **Intel DMI**: link CPU ↔ chipset.
- **AMD Infinity Fabric**: CPU ↔ CPU, CPU ↔ GPU, entre chiplets. Também suporta coerência.

### USB

2.0 (480 Mbps), 3.0/3.1/3.2 (5/10/20 Gbps), 4/4v2 (40/80 Gbps), USB-C conector.

### Thunderbolt

- TB3/4: 40 Gbps.
- TB5 (2024+): 80/120 Gbps bi/assym.
- Encapsula PCIe + DisplayPort. Base de eGPU (ver `EGPU.md`).

### Ethernet

1G dominante consumer; 2.5G/5G/10G crescente; 25/40/100/400/800G server/DC. SFP+/QSFP+/QSFP-DD pluggables.

### InfiniBand / NVLink

- **InfiniBand**: HPC/DC, low-latency, RDMA.
- **NVLink (Nvidia)**: GPU↔GPU, 600-900 GB/s em H100/GB200.
- **NVLink Fusion**: CPU+GPU (Grace-Hopper).

## Chiplets / Packaging

Chip único gigante tem yield baixo → dividir em **chiplets** menores, conectar por interposer/substrate.

- **AMD Ryzen/EPYC**: CCDs (compute) + IOD (I/O + memoria). IF conecta.
- **Intel Sapphire Rapids/Granite Rapids**: tiles (compute, I/O) via EMIB.
- **Apple Silicon Ultra**: dois Max colados por UltraFusion interconnect, apresentando memória unificada.
- **Nvidia GB200**: Grace CPU + Blackwell GPUs via NVLink-C2C.

### Packaging avançado

- **2.5D** (TSMC CoWoS, Intel EMIB): interposer de silício une dies.
- **3D** (Intel Foveros, AMD 3D V-Cache): empilhamento vertical. X3D chips (gaming Ryzen) têm L3 extra empilhado.

## Heterogeneidade e Aceleração

- **GPU**: paralelismo massivo — ver `GPU.md`, `GPGPU.md`.
- **NPU / TPU / IPU**: ASICs para ML. Google TPU, Apple ANE, AMD XDNA, Qualcomm Hexagon.
- **DPU / SmartNIC**: offload de rede/storage/segurança. AWS Nitro, BlueField, Intel IPU.
- **FPGA**: reconfigurável — ver `FPGA.md`.

Tendência: CPU **orquestra**, aceleradores fazem trabalho pesado. Heterogeneous computing dominante em DC/mobile.

## Data Center Systems

- **Nós**: servidor com 1-8 CPUs + GPUs + NVMe + NIC high-speed.
- **Racks**: 42U, TOR switch.
- **Spine-leaf** fabric.
- **Pod / cluster**: milhares de nós. Supercomputadores: El Capitan, Frontier (exascale).
- **Power**: PDUs, PSUs redundantes, liquid cooling crescente (GPUs de 1kW).
- **Cooling**: air → liquid (direct-to-chip, immersion). PUE (Power Usage Effectiveness) ~1.1 é state-of-the-art.

## Mobile / Laptop

- **SoC**: CPU + GPU + NPU + ISP + modem + memory controller em um chip. Apple M-series, Snapdragon, Dimensity.
- **Memória unificada (UMA)**: GPU e CPU compartilham LPDDR — zero cópia. Muito eficiente em ML inference.
- **E-cores** dominam uso normal; P-cores para bursts.
- **LPDDR5X**, mais eficiente que DDR5 desktop.
- **Apple Silicon**: M4 series combina 10+ wide OoO, LPDDR5X, ANE (neural), media engines.

## Benchmarking

### CPU

- **SPEC CPU**: padrão acadêmico/industrial (int + fp).
- **Geekbench, Cinebench**: consumer.
- **STREAM**: bandwidth de memória.
- **MLPerf**: ML training/inference.
- **LINPACK/HPL**: supercomputadores (Top500 ranking).

### Custo real

Microbenchmarks isolam métrica, mas **workload real** frequentemente é diferente. Sempre: benchmark representativo do uso pretendido.

## Segurança Arquitetural

- **Spectre, Meltdown, L1TF, MDS, Zenbleed, Downfall, Inception** — classe de vulnerabilidades em execução especulativa. Ver `CRYPTO_ADVANCED.md`, `HARDWARE_ATTACKS.md`.
- **Mitigações**: microcode updates, compilador (retpoline, LFENCE), OS (KPTI, IBPB).
- **Confidential Computing**: SEV-SNP, TDX, ARM CCA, SGX (obsoleto em client). VMs cifradas com atestação.

## Evolução Histórica Sucinta

- **1971**: Intel 4004 — 4-bit, 2300 transistores.
- **1978**: 8086 — x86 nasce.
- **1985**: 386 — 32-bit, protected mode.
- **1993**: Pentium — superscalar comercial.
- **2003**: AMD64 — 64-bit x86.
- **2006**: Intel Core 2 — dual-core consumer.
- **2011**: ARM Cortex-A15 / Apple A5 dual-core mobile.
- **2017**: AMD Zen ressuscita x86 competitividade.
- **2020**: Apple M1 — ARM laptop competitive.
- **2022**: DDR5, PCIe 5.0, AVX-512 maduro.
- **2024-2026**: chiplets dominantes, AI dedicated silicon (TPUs, NPUs), CXL, PCIe 6.

## Recursos

- **"Computer Architecture: A Quantitative Approach"** — Hennessy & Patterson. A referência.
- **"Computer Organization and Design"** — Patterson & Hennessy (undergrad).
- **Agner Fog optimization manuals** (agner.org): microarquitetura prática, x86.
- **uops.info, InstLatx64**: latência/throughput de instruções x86.
- **WikiChip, chipsandcheese.com, AnandTech deep dives**.
- **Intel Optimization Reference Manual, AMD SoG, ARM Optimization Guide**.
- **Jim Keller talks**: princípios de design moderno.

## Princípios

1. **Cache é rei**. Perfil sem contar cache miss é perfil incompleto.
2. **Latência de memória não cai**. Banda sim; latência faz décadas estagnada em ns.
3. **Paralelismo é a saída**. Single-thread ganhou pouco por ciclo nos últimos anos; multicore + SIMD + aceleradores dominam.
4. **ISA é contrato, microarch é realidade**. Mesma ISA, perfis muito diferentes.
5. **Modelos de memória existem**. Programar concorrência sem entendê-los é convite a bugs.
6. **Heterogeneidade é norma**. E-/P-cores, CPU+GPU+NPU — schedulers conscientes ganham.
7. **Meça no hardware alvo**. Laptop ≠ server; mobile ≠ desktop; mesmo modelo entre gens muda tudo.
