# FPGA — Field-Programmable Gate Arrays

FPGAs são **circuitos integrados reconfiguráveis**: em vez de lógica fixa como CPUs/ASICs, carregam uma "arquitetura" definida por software (bitstream) que, na hora do boot, programa blocos lógicos e interconexões para implementar qualquer circuito digital até o limite de recursos do chip. Permitem prototipar hardware sem fabricar; implantar custom logic com paralelismo e latência que CPU/GPU não alcançam; e, em casos específicos, rodar aplicações onde nada mais serve.

Complementa `CIRCUIT.md`, `COMPUTER_ARCHITECTURE.md`, `GPU.md`, `IOT_FIRMWARE_SECURITY.md`, `HARDWARE_ATTACKS.md`.

## Quando FPGA É a Resposta

Comparativo mental:

| | Latência | Throughput | Flexibilidade | NRE | Custo unidade |
|---|---|---|---|---|---|
| **CPU** | ms | baixo | altíssima | baixo | $ |
| **GPU** | ms | altíssimo (paralelo) | alta | baixo | $$$ |
| **FPGA** | ns-µs determinístico | médio-alto | média | médio | $$-$$$ |
| **ASIC** | ns | altíssimo | nenhuma | $1-100M | $ em volume |

FPGA brilha quando você precisa de **latência determinística** (sub-µs), **paralelismo custom** que GPU não mapeia bem, **interfaces com hardware específico** (alta taxa de I/O), ou **ASIC não compensa** por volume baixo.

Casos típicos:

- **Networking**: SmartNIC, packet processing, 400G line rate.
- **HFT (high-frequency trading)**: parse FIX/ITCH + ordem em nanossegundos.
- **DSP**: radar, sonar, telecomunicações, SDR.
- **Vídeo**: encoders/decoders custom, latência baixa em broadcasting.
- **Prototipagem ASIC**: emulação antes de tapeout.
- **Aeroespacial, militar, médico**: radhard, custom, long lifetime.
- **Aceleração ML**: Microsoft Azure Brainwave, Intel/Altera FPGAs em data center.
- **Hobby/educação**: aprender arquitetura digital de verdade.

Quando **não** usar: workload já bem servido por CPU/GPU, time-to-market crítico, equipe sem experiência HDL, protótipo que só roda uma vez.

## Estrutura Interna

### Blocos fundamentais

- **LUT (Look-Up Table)**: pequena RAM de 4-6 entradas que implementa qualquer função booleana de N entradas. Átomo do FPGA moderno.
- **Flip-flop (FF)**: registrador de 1 bit junto a cada LUT.
- **Slice / CLB / ALM**: agrupa LUTs, FFs, multiplexadores, carry chain.
- **Interconnect**: malha programável de roteamento — define conexões entre slices.
- **Clock network**: árvores dedicadas com baixo skew.
- **I/O banks**: pinos programáveis — padrões LVCMOS, LVDS, HSTL, etc.

### Blocos dedicados (hard IP)

- **Block RAM (BRAM)**: 18-36 Kbit por bloco, dual-port.
- **Ultra RAM, HBM2** (high-end): mais densidade/banda.
- **DSP slices**: multiplicador + acumulador (MACs). Essencial para DSP, ML.
- **SerDes / MGT**: transceivers multi-gigabit (até 112 Gbps hoje). PCIe, Ethernet, SATA, JESD204.
- **PCIe hard IP**: controller em hardware.
- **Memory controllers**: DDR3/4/5, HBM.
- **Processor hard IP**: ARM (Zynq), RISC-V (PolarFire), em SoC FPGA.

Essa mistura é o que torna FPGA prático — reconstruir CORE serial na fabric seria impraticável.

## Famílias / Fabricantes

### AMD (Xilinx)

- **Spartan-7, Artix-7/UltraScale+**: baixo custo, entrada.
- **Kintex-7/UltraScale+**: médio.
- **Virtex-7/UltraScale+, Versal AI Core/Premium/HBM**: high-end datacenter.
- **Zynq-7000 / UltraScale+ MPSoC**: FPGA + ARM Cortex-A9/A53/R5.
- **Versal ACAP**: FPGA fabric + AI engines (VLIW) + ARM + NoC.

### Intel (Altera, rebrand recente)

- **Cyclone** (low-cost), **Arria** (mid), **Stratix** (high-end).
- **Agilex**: topo atual — 7/9/5 gerações.
- **Intel FPGA + Xeon** integrações (Stratix 10 MX com HBM2).

### Lattice

Pequenos, baixo consumo, baixo custo. iCE40, ECP5, CrossLink, Certus.

**iCE40 e ECP5**: muito populares em open-source (toolchain yosys + nextpnr).

### Microchip (Microsemi)

PolarFire, PolarFire SoC (RISC-V), IGLOO. Ênfase em baixo consumo, radhard, secure boot.

### Achronix, Efinix, Gowin, Anlogic

Nichos. Efinix Trion é interessante open-tooling-friendly. Gowin popular em produtos chineses baratos.

### Open silicon

- **OpenTitan** (RoT), **CARAVEL / Efabless** — projetos open sourced ASIC/FPGA.
- **SkyWater PDK** — open process node.

## Bitstream e Boot

O **bitstream** é o arquivo binário que configura fabric. Carregado de:

- **Flash externa** (SPI, QSPI) ao boot.
- **Host** via JTAG ou PCIe.
- **Internal config memory** (algumas famílias têm).

**Partial reconfiguration**: parte do FPGA atualizada sem reset — permite hot-swap de módulos. Caro de configurar mas útil em DC.

**Encryption + authentication** do bitstream protegem IP e impedem que bitstream adulterado rode. CVEs históricos (Starbleed 2020) em Xilinx 7-Series mostraram que crypto em bitstream não é trivial.

## Fluxo de Design

1. **Especificação**: o que o circuito faz.
2. **RTL (HDL)**: Verilog/SystemVerilog/VHDL — descreve comportamento no nível de transferência entre registradores.
3. **Simulação** (testbench): Icarus, Verilator, ModelSim, Questa, Vivado Simulator.
4. **Síntese**: RTL → gate-level netlist para LUTs/FFs específicos do target. Vivado Synthesis, Quartus, yosys.
5. **Place & Route (fitter)**: alocar slices e rotear interconnect. Vivado Implementation, Quartus Fitter, nextpnr (open).
6. **Timing analysis**: static timing analyzer verifica setup/hold em todos os caminhos contra constraint de clock. Falhou → refatorar RTL, pipelining, floorplan.
7. **Bitstream generation**.
8. **Programa FPGA** e **debug** (ILA, SignalTap — logic analyzer interno).

Ciclo pode ser demorado: compile de design médio leva **minutos a horas**. Simulação é o amigo.

## HDL — Verilog, SystemVerilog, VHDL

### Verilog / SystemVerilog

Sintaxe C-like. SystemVerilog adiciona interfaces, structs, classes, verification features (UVM).

```verilog
module counter(
  input  logic clk, rst_n,
  output logic [7:0] q
);
  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n) q <= 8'd0;
    else       q <= q + 1;
endmodule
```

### VHDL

Sintaxe Ada-like, verboso, fortemente tipado. Dominante em Europa, defense, aerospace.

```vhdl
process(clk, rst_n) is
begin
  if rst_n = '0' then
    q <= (others => '0');
  elsif rising_edge(clk) then
    q <= q + 1;
  end if;
end process;
```

### Comparação

- **Verilog**: mais comum, shorter, maior ecossistema open.
- **VHDL**: verboso, seguro, defense-favored.
- Ambos sintetizáveis no mesmo fluxo. Projeto mistos existem.

## Abstrações Modernas

### SystemVerilog + UVM

UVM (Universal Verification Methodology) é framework para testbenches grandes. Classes, sequences, drivers, monitors, scoreboards.

### HLS (High-Level Synthesis)

Compila C/C++ para RTL. Vivado HLS / Vitis HLS (AMD), Intel HLS Compiler, Catapult.

- **Pros**: iteração rápida, acessível a desenvolvedor de software.
- **Contras**: pragmas críticos para performance; entendimento de hardware ainda necessário; não elimina skill HDL.

### Chisel

eDSL em Scala. Compila para Verilog. Usado em RISC-V cores acadêmicos (Rocket, BOOM). Type-safe, parameterizable.

### SpinalHDL, Amaranth (antigo nMigen), MyHDL

Outras alternativas Python/Scala/etc.

### OpenCL for FPGA

Intel FPGA SDK for OpenCL, Xilinx SDAccel (obsoleto → Vitis). Modelo acelerador host+device.

### oneAPI / SYCL

Intel oneAPI suporta FPGA target. Abstrai CPU/GPU/FPGA.

## SoC FPGAs (ARM + Fabric)

Combinação popular:

- **Zynq-7000**: ARM Cortex-A9 dual + Artix-7 fabric. Linux + fabric custom.
- **Zynq UltraScale+ MPSoC**: Cortex-A53 quad + R5 + fabric + PCIe Gen3.
- **Versal ACAP**: AI engines + ARM + fabric + NoC (rede coerente on-chip).
- **Intel Agilex SoC**: ARM A76 + fabric.
- **PolarFire SoC**: RISC-V U54 + fabric.

Permite "CPU software" para control plane + "FPGA hardware" para data plane. Standard em produtos indústria 4.0, 5G base station, storage accelerators.

## AI Engine e Novas Arquiteturas

- **AMD Versal AI Engine (AIE)**: 100+ VLIW vector cores arranjados em 2D mesh, conectados ao fabric. Para NN inference e sinal DSP intenso.
- **Achronix Speedster 7t**: ML-optimized.
- **Intel Agilex AI Tensor Blocks**: novos blocos otimizados.

Fronteira entre "FPGA" e "accelerator reconfigurável" blur.

## Timing

Central em design FPGA:

- **Clock domain crossing (CDC)**: sinais cruzando domínios de clock diferentes exigem sincronizadores (flip-flops em cascata), handshake, FIFOs assíncronos. Bugs em CDC são intermitentes e insidiosos — ferramentas de análise formal específicas.
- **Setup / hold violation**: STA detecta; falha = ferramenta avisa; designer corrige (pipelining, floorplanning).
- **Slack**: margem positiva em cada caminho; design com slack positivo roda na frequência alvo.
- **Pipelining**: dividir caminho longo em estágios reduz delay por ciclo, aumenta latência em ciclos.

## Debug

- **Simulação**: cobertura a maior parte dos bugs em HDL bem escrito.
- **ILA (Integrated Logic Analyzer)** / **SignalTap**: logic analyzer embutido na fabric; captura sinais em RAM interna, lê via JTAG.
- **Waveform viewer**: GTKWave (open).
- **Virtual I/O (VIO)**: injeta/controla valores durante runtime.
- **Formal verification**: prova propriedades (SVA assertions). Jasper, Cadence JasperGold, Symbiyosys (open).

## Open Source Toolchain

Revolução recente:

- **yosys** (Clifford Wolf): synthesis.
- **nextpnr**: P&R moderno, sucessor do arachne-pnr. Suporta iCE40, ECP5, Gowin, Xilinx 7-Series (parcial).
- **Project IceStorm, Project Trellis, Project X-Ray, Project Mistral**: bitstream reverse engineering para habilitar toolchain aberta.
- **cocotb**: testbench em Python.
- **Verilator**: simulador cycle-accurate open, fastest.

Esse ecossistema viabiliza hobby, pesquisa, educação sem dependência de licenças multi-milhares de dólares.

## Boards Populares

### Hobby/educação

- **iCEBreaker, iCE40-HX8K, TinyFPGA**: iCE40 com toolchain open.
- **ULX3S, ECPIX-5**: ECP5.
- **Arty A7**: Artix-7 para entrada Xilinx.
- **Tang Nano/Primer**: Gowin barato.
- **Cyclone V GX Starter Kit**: Intel.

### Prosumer / dev

- **Zybo, ZedBoard, PYNQ-Z1/Z2**: Zynq para aprendizado + Python integration (PYNQ).
- **Ultra96, MPSoC kits**.

### Datacenter / aceleração

- **Alveo U200/U250/U280 (Xilinx), Stratix 10 PAC (Intel)**: placas PCIe full-height com HBM, SFP+/QSFP+.
- **AWS F1 instances**: FPGA na cloud.
- **Intel PAC, Microsoft Catapult**: deployments DC.

## Custos e NRE

Licenças de ferramentas proprietárias dominam o custo:

- **Free tiers**: Vivado ML Standard (cobre devices pequeno-médio), Intel Quartus Prime Lite.
- **Enterprise**: milhares a dezenas de milhares $/seat/ano para ferramentas que suportam chips top.
- **IP cores** (PCIe, Ethernet, DDR, crypto): compra-se ou usa-se gratuitamente da vendor; terceiros vendem premium.

Chip em si: de ~$10 (iCE40) a $50k+ (Versal/Stratix 10 high-end).

## Segurança

Ver `HARDWARE_ATTACKS.md`:

- **Bitstream encryption** (AES, SHA authentication). CVEs em implementations antigas.
- **Anti-tamper sensors, PUF (Physically Unclonable Function)** para key generation.
- **Secure boot** em SoC FPGAs: cadeia assinada.
- **Side channels** em crypto no fabric exigem contramedidas (masking).

## Casos de Uso em Escala

- **Microsoft Catapult/Brainwave**: aceleração de Bing, Azure ML.
- **AWS F1**: FPGA-as-a-service.
- **Nasdaq/NYSE low-latency**: matching engines em FPGA.
- **5G RAN**: ORAN implementations.
- **Smart NIC**: Mellanox BlueField (ARM + fabric/ASIC), Intel IPU.
- **Medical imaging**: CT/MRI image reconstruction.

## Princípios

1. **HDL é descrição de hardware**, não programa sequencial. Pensar em paralelismo + tempo.
2. **Timing closure primeiro**. Funcional em 100 MHz ≠ funcional em 400 MHz. Pipeline cedo.
3. **Simulação antes de bitstream**. Ciclo de síntese é longo; simular é minutos ou segundos.
4. **CDC é a origem de bugs sutis**. Sincronize; use FIFOs; use ferramentas.
5. **Reuse IP**: PCIe, DDR, Ethernet — nunca do zero se vendor provê hard IP.
6. **Open tool ≠ para todos os chips**. yosys/nextpnr brilham em iCE40/ECP5; para Virtex UltraScale+, Vivado.
7. **FPGA ≠ CPU + paralelismo**. Se CPU/GPU dão conta, não agregue complexidade.
8. **Custo total**: licenças + board + tempo de engenheiro >> chip. Orce de modo realista.
