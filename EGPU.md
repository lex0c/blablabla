# eGPU — External GPU

**External GPU** (eGPU) é uma placa de vídeo desktop colocada em um gabinete externo e conectada ao host (laptop, mini-PC, handheld) por um link PCIe tunelado sobre Thunderbolt, USB4 ou OCuLink. A promessa: ter GPU upgradeable em máquina portátil. A realidade: **depende inteiramente da interface** — o link é invariavelmente mais estreito que o slot PCIe nativo de desktop, e isso muda drasticamente o que funciona bem.

Complementa `GPU.md`, `COMPUTER_ARCHITECTURE.md`, `CIRCUIT.md`.

## Por que existe

1. **Laptops e mini-PCs** têm GPU integrada ou discreta modesta, não upgradeable. eGPU dá acesso a RTX 4090/5090 ou Radeon 7900/9070.
2. **Upgrade path**: troca a GPU, mantém o host.
3. **Setup híbrido**: laptop para mobilidade, eGPU na mesa para sessão de jogo, ML, rendering, video edit.
4. **Handhelds (Steam Deck, ROG Ally, Legion Go)**: eGPU via OCuLink vira desktop gaming-class quando plugado.
5. **Servidores homelab / dev**: eGPU em mini-PC para ML inference local sem build de torre.

## Interfaces

A interface é **a** variável crítica. Todas tunelam PCIe, mas com larguras diferentes:

| Interface | PCIe tunelado | Bandwidth real (GPU) | Overhead vs nativo |
|---|---|---|---|
| Thunderbolt 3 | PCIe 3.0 x4 | ~22 Gbps ≈ 2.8 GB/s efetivo | ~85% de PCIe 3.0 x4 |
| Thunderbolt 4 | idem TB3 (mesma banda) | ~2.8 GB/s | idem |
| USB4 (v1) | PCIe 3.0 x4 | ~2.8 GB/s | idem |
| **USB4 v2 / TB5** | PCIe 4.0 x4 | ~6 GB/s | melhor |
| **OCuLink (PCIe x4)** | PCIe 4.0 x4 direto, sem tunel | ~7-7.5 GB/s | ~full PCIe 4 x4 |
| **PCIe cable (M.2 x4)** | PCIe 4.0 x4 | ~7.5 GB/s | mesma coisa |
| Desktop PCIe 4.0 x16 (referência) | — | ~32 GB/s | 100% |

**Conclusão aritmética**: até a melhor conexão eGPU (TB5, OCuLink) entrega ~**1/4** da banda de um slot PCIe x16 nativo. Em muitos workloads não importa; em alguns, importa muito.

### Thunderbolt / USB4

- **Versátil**: mesmo cabo carrega PCIe + DisplayPort + USB + power.
- **Hot-plug** quase plug-and-play.
- **Overhead de tunneling** do Thunderbolt: ~15%.
- **Compatibilidade**: host precisa suportar (Intel Thunderbolt controller, AMD USB4, Apple Silicon).
- **Latência** ligeiramente maior que OCuLink.

### OCuLink (PCIe over cable)

- Padrão SFF-8611. Cabo externo carregando PCIe x4 direto.
- **Sem overhead** de tunneling. Latência mínima.
- **Não hot-plug** em geral — requer configuração BIOS/driver; plug com sistema ligado pode travar.
- **Sem DP/USB/power no cabo**; só PCIe. Precisa cabos separados.
- **Adotado por handhelds** (GPD WIN Max/Win 4, OneXPlayer, Minisforum V3) e mini-PCs. Tornou-se padrão de facto do mercado "performance eGPU".

### M.2 NVMe adapter

Solução **barata**. Remove SSD M.2 (ou ocupa slot extra), insere cabo adapter para gabinete externo. Efetivamente PCIe x4 direto, similar a OCuLink.

- **Prós**: barato (~$30 de adapter); mesma banda do OCuLink.
- **Contras**: você perde um slot NVMe; requer abrir o host; fisicamente frágil.

### Desktop PCIe externo?

Alguns SFF/AIO usam slot PCIe externo não-padronizado. Nicho.

## Bandwidth, Latência, e Impacto Real

GPU não é homogênea em sensibilidade à banda host:

### O que **é** sensível

- **Gaming competitivo 1080p** em GPU top: CPU/GPU trocam muitos frames; eGPU perde 10-25% de FPS vs desktop.
- **Loading de texturas grandes**: streaming via DirectStorage, asset loading — PCIe x4 é gargalo.
- **Simulações que streaming dados**: Frostbite/UE engine em cenário grande.
- **Workload com muitos small host→device copies** (típico de code não otimizado).

### O que **não é** muito sensível

- **Gaming 4K** em GPU top: GPU é gargalo, não PCIe. Overhead pequeno (~5%).
- **Rendering offline** (Blender, V-Ray): dados vão uma vez, GPU trabalha sem comunicação com host.
- **ML inference local**: modelo carrega uma vez; inference é on-device. PCIe quase não importa.
- **Video encoding** com NVENC/NVDEC.

### Latência

Thunderbolt adiciona ~1-2 ms a frame render. Em 60 Hz (16.7 ms/frame) é perceptível competitive; em 120+ Hz, mais notável. OCuLink tem overhead menor (~centenas de µs).

### Uso do display

Duas opções:

1. **Display externo plugado no eGPU** (melhor): frame nunca volta ao host.
2. **Laptop display** via loopback: frame renderiza na eGPU, vai de volta via Thunderbolt para iGPU/laptop display, ocupando banda. **~30% de overhead adicional**.

Gamer sério sempre usa display externo conectado diretamente à eGPU.

## Gabinetes eGPU (Enclosures)

### Thunderbolt

Para TB3/TB4:

- **Razer Core X / X Chroma** — clássico.
- **Sonnet Breakaway Box**.
- **Mantiz Saturn Pro II**, **AORUS Gaming Box** (vendem com GPU embutida).
- **ASUS ROG XG Mobile** — proprietário, GPU + conexão customizada.

Limitações comuns: PSU 500-700W, tamanho GPU suporta (full-length / 2.5-slot), cooling passivo/ativo.

### OCuLink

Mais recente, menor ecossistema:

- **Minisforum DEG1**, **ADT-Link DIY**.
- **GPD G1** (mais integrado, pequeno).
- **OneXPlayer ONEXGPU**.

Preço geralmente mais baixo que TB de qualidade comparável.

### DIY

Cabo M.2-to-PCIe + ATX PSU de sucata + riser PCIe = eGPU artesanal de $50-100. Fio adapter, power e ventilação são responsabilidade do montador.

## Setup e Compatibilidade

### Windows

- **Thunderbolt**: driver do controller (Intel TBT, etc.) + driver GPU do vendor. Reboot frequente após primeiro plug.
- **OCuLink**: depende de BIOS/UEFI expor PCIe root port. Alguns hosts exigem habilitar manualmente.
- **Error 12 / Code 12**: PCI resource exhaustion, comum em laptops que não reservam espaço para GPU externa. Às vezes resolvido com "Large memory BAR" / "Above 4G decoding".
- **Optimus / MUX**: laptop com iGPU + dGPU interna, agora + eGPU = 3 GPUs. Sistema de seleção "per-app GPU" em Windows 10+.

### Linux

- **Thunderbolt**: `bolt` / `thunderbolt` kernel module; autorização explícita de devices (security rating, mitigação anti-DMA).
- **OCuLink**: PCIe hot-plug não é sempre estável; muitas distros assumem cold-plug.
- **Nvidia**: driver proprietário, `nvidia-prime` ou `optimus-manager` para switching.
- **AMD**: driver amdgpu, suporte variado.

### macOS

- Apple Silicon: **não há suporte oficial a eGPU Nvidia** (Apple removeu Nvidia em drivers desde Mojave). AMD eGPU via TB funciona em Intel Macs; problemático em Apple Silicon (sem driver).
- Antes da transição ARM, macOS tinha eGPU AMD como feature de primeira classe.

### Plug sequencing

Ordem geralmente:

1. Gabinete ligado e GPU inserida.
2. Host desligado → conecta cabo → liga host (mais seguro, menos bugs).
3. Alternativamente, hot-plug em sistema ligado (TB só, não OCuLink).
4. Espera driver carregar.
5. Configurar display output e preferência por aplicação.

## Casos de Uso por Workload

### Gaming

- **4K + GPU top** (RTX 5080/5090, RX 9070): eGPU entrega ~85-92% do desempenho desktop. Excelente trade.
- **1080p high FPS competitive**: perde mais (15-25%); desktop ganha.
- **VR** depende de codec e headset: Quest Link via USB adiciona overhead; display direto no PC é ok.

### ML / IA

- **Inference local (Stable Diffusion, LLM 7B-70B quantizado)**: bandwith pouco importa; modelo carrega uma vez. eGPU via TB3 = desktop em throughput prático.
- **Training pequeno**: idem.
- **Training grande com dataset streaming**: pode haver gargalo — mas em escala grande ninguém usa eGPU.
- **Modelos quantizados** (INT8/FP8) toleram ainda menos bandwidth.

### Creative / Productivity

- **Blender, V-Ray, OctaneRender**: GPU-bound, pouco dependente de PCIe. eGPU excelente.
- **DaVinci Resolve, Premiere**: timeline playback e renders — eGPU funciona bem, melhor ainda com display externo.
- **Unreal Engine, Unity**: compile/shader ok; playback de cena complexa depende de streaming.
- **CAD** (Solidworks, Fusion 360): GPU pouco usada na maioria; ganho marginal.

### Dev / Homelab

- **ML experiments em casa** sem gastar R$30k em workstation.
- **Compile farm** não se beneficia de GPU.
- **Servidor local de inferência** (Ollama, llama.cpp com CUDA/ROCm) — excelente.

### Handhelds

- **Steam Deck / ROG Ally / Legion Go / GPD WIN 4** + OCuLink eGPU = modo "dock gaming class". Transição entre portátil e "desktop".
- Limite: CPU do handheld (Z1 Extreme, Steam Deck APU) pode virar gargalo em títulos CPU-intensivos com GPU top.

## Limitações Estruturais

1. **Bandwidth**. Sempre PCIe x4-tunneled, nunca x16. Para 90% dos cargos ok; para 10% de casos específicos é o gargalo.
2. **Latência**. ~1-2 ms TB; ~0.5-1 ms OCuLink. Perceptível em gaming competitivo de alta refresh.
3. **Custo total**. Gabinete TB + GPU de ponta supera o custo de um bom desktop. Só vale se portabilidade importa.
4. **Tamanho**. Gabinetes não são pequenos; levar junto não é realístico.
5. **Driver/compat quirks**. Primeiro setup é sempre adventure. Macs ARM especialmente problemáticos.
6. **Ruído**. Gabinete com GPU 300-500W em ambiente doméstico é barulhento.
7. **Power draw**. GPUs topo sacam 400-600W + sistema. PSU do gabinete precisa ter margem.

## Tendências Recentes

- **TB5 / USB4 v2**: dobra banda efetiva. Mudando curva de custo-benefício em 2024-2026.
- **OCuLink popularização**: mini-PCs e handhelds tornaram-no padrão do tier "performance sem compromisso".
- **NVIDIA GeForce NOW / xCloud / PS Remote**: streaming como alternativa a eGPU local.
- **LLMs locais**: reavivou interesse por eGPU para rodar Llama/Qwen/Mistral locally sem servidor DC.
- **Apple Silicon reliance em UMA**: Apple aposta em GPU integrada com memória unificada, não em eGPU.

## Ferramentas e Diagnóstico

- **GPU-Z, HWiNFO** (Windows): confirmar que PCIe link ativo é o esperado (Gen/Width).
- **`lspci -vvv`** (Linux): `LnkSta:` mostra speed (2.5/5/8/16/32 GT/s) e width (x1/x4/x16).
- **`nvidia-smi`**: verificar reconhecimento e utilização.
- **Benchmark** com e sem display externo para isolar overhead de loopback.

## Princípios

1. **Interface é tudo**. OCuLink > USB4 v2 / TB5 > TB3/TB4 > USB4 v1. Escolha informa resultado.
2. **Display externo** sempre que gaming. Loopback custa ~30%.
3. **4K é amigo**. GPU bound; eGPU vira quase igual a desktop.
4. **Workload define ROI**. Rendering offline / ML inference = quase sem perda. Gaming competitivo = consciência das perdas.
5. **Custo total**. Gabinete + GPU + PSU pode superar desktop equivalente. Portabilidade tem preço.
6. **Driver ritual**. Primeiro setup é chato; depois estável. Backup do OS antes de experimentar.
7. **Linux OCuLink** ainda amadurecendo — avançado para usuários técnicos.
