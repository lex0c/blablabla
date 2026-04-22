# Circuitos Elétricos e Eletrônica

Circuitos elétricos são a base material de quase toda tecnologia moderna — da rede elétrica ao smartphone. Um **circuito** é um caminho fechado através do qual **cargas elétricas** fluem sob ação de uma **tensão** aplicada, passando por componentes que limitam, armazenam, redirecionam ou processam esse fluxo. Eletrônica é o subcampo que trata de **circuitos de baixa potência** controlando sinais e informação — análogo ao que mecânica está para máquinas.

## Grandezas Fundamentais

| Grandeza | Símbolo | Unidade | Intuição |
|---|---|---|---|
| Carga | Q | coulomb (C) | "quantidade" de eletricidade |
| Corrente | I | ampère (A = C/s) | fluxo de carga por tempo |
| Tensão (ddp) | V | volt (V = J/C) | energia por unidade de carga |
| Resistência | R | ohm (Ω = V/A) | oposição à corrente |
| Capacitância | C | farad (F = C/V) | carga armazenada por volt |
| Indutância | L | henry (H = V·s/A) | fluxo magnético por ampère |
| Potência | P | watt (W = V·A) | taxa de energia |
| Energia | E | joule (J) | capacidade de trabalho |
| Frequência | f | hertz (Hz = 1/s) | ciclos por segundo |

**Analogia hidráulica** (imperfeita mas útil):

- Tensão ≈ pressão da água.
- Corrente ≈ vazão (litros/segundo).
- Resistência ≈ estreitamento do cano.
- Capacitor ≈ reservatório flexível.
- Indutor ≈ turbina com inércia.

## Leis Fundamentais

### Lei de Ohm

`V = I · R`. Vale para resistores ideais (lineares, ôhmicos). Dispositivos não-ôhmicos (diodos, transistores) têm relações não-lineares.

### Leis de Kirchhoff

1. **Lei das Correntes (KCL)**: a soma das correntes entrando em um nó é igual à soma das que saem. `Σ I_in = Σ I_out`. Consequência de conservação de carga.
2. **Lei das Tensões (KVL)**: a soma das quedas de tensão em um laço fechado é zero. `Σ V = 0`. Consequência de conservação de energia.

Juntas com Ohm, resolvem todo circuito linear.

### Lei de Joule

`P = V · I = I² · R = V² / R`. Potência dissipada. Limita resistor, fio, trilha — cada um tem um *rating* máximo.

### Teorema de Thévenin

Qualquer rede linear com dois terminais é equivalente a uma **fonte de tensão ideal** (V_th) em série com uma **resistência** (R_th). Simplifica análise enorme.

### Teorema de Norton

Dual do Thévenin: **fonte de corrente ideal** (I_n) em paralelo com resistência (R_n = R_th). `I_n = V_th / R_th`.

### Superposição

Em circuito linear, resposta a múltiplas fontes = soma das respostas a cada fonte isoladamente (demais zeradas).

### Máxima Transferência de Potência

Potência entregue a uma carga é máxima quando `R_load = R_source`. Em AC, `Z_load = Z_source*` (conjugado complexo). Conceito-chave em RF, antenas, audio.

## Componentes Passivos

### Resistor

Converte energia elétrica em calor. Parâmetros:

- **Valor** (Ω).
- **Tolerância** (1%, 5%, 10%).
- **Potência** (1/8W, 1/4W, 1W — dissipação máxima).
- **Coeficiente de temperatura** (ppm/°C).
- **Tipos**: filme de carbono (barato), filme metálico (precisão), wirewound (alta potência), SMD (superfície).

**Código de cores** (eixo de competência estudantil):
- Preto 0, Marrom 1, Vermelho 2, Laranja 3, Amarelo 4, Verde 5, Azul 6, Violeta 7, Cinza 8, Branco 9.

### Capacitor

Armazena energia em campo elétrico. `Q = C · V`, `I = C · dV/dt` — ou seja, corrente só flui quando tensão muda.

**Propriedades**:
- **Valor** (pF, nF, µF, mF).
- **Tensão máxima** (superada = estourar).
- **ESR** (Equivalent Series Resistance): resistência parasitária; baixa = melhor para high-freq.
- **Tipos**: cerâmica (pequenos, rápidos, não-polarizados), eletrolítico (grandes, polarizados, baixa freq), tântalo (estáveis), filme (audio, alta qualidade), super (Faradays, armazenamento).

**Aplicações**: decoupling/bypass (filtrar ruído em alimentação — dogma: cap próximo a cada chip IC), acoplamento AC, timing (RC), filtros, armazenamento.

### Indutor

Armazena energia em campo magnético. `V = L · dI/dt` — "resiste à mudança de corrente".

- **Valor** (µH, mH, H).
- **Corrente de saturação**: acima, núcleo satura e L cai.
- **Q factor**: qualidade — baixo = perdas.

**Aplicações**: filtros LC, SMPS (buck/boost), transformadores, chokes EMI, antenas, sensores.

### Diodo

Permite corrente **em apenas uma direção**. Junção P-N.

- **Queda direta** (Vf): ~0.6-0.7V em silício, ~0.3V Schottky, ~3V LEDs azuis.
- **Tensão reversa máxima** (PIV): excedendo = avalanche.
- **Corrente máxima**.

**Tipos especializados**:
- **Zener**: conduz reverso em Vz definido → referência de tensão.
- **Schottky**: queda baixa, rápida; SMPS.
- **LED**: emite luz; Vf varia com cor.
- **Fotodiodo**: corrente proporcional à luz incidente.
- **TVS (Transient Voltage Suppression)**: protege contra picos.

**Aplicações**: retificação (AC → DC), proteção polaridade, detectores, referências.

## Componentes Ativos

### Transistor Bipolar (BJT)

Três terminais: base (B), coletor (C), emissor (E). NPN (mais comum) ou PNP.

- **Modo ativo** (amplificação): `I_C = β · I_B`, onde β (ganho) ~100-300.
- **Modo saturação**: V_CE ≈ 0.2V, "fechado" (chave ON).
- **Modo corte**: I ≈ 0 ("aberto", chave OFF).

Usado em amplificadores de áudio, chaves de corrente alta, circuitos analógicos.

### MOSFET

Field-Effect Transistor. Controlado por **tensão** de gate (alta impedância → quase sem corrente no gate). N-channel ou P-channel.

- **Threshold voltage (Vth)**: tensão de gate para conduzir.
- **R_DS(on)**: resistência quando ligado. Quanto menor, menos perda.
- **Capacitâncias de gate**: limitam velocidade de comutação.

**Vantagem sobre BJT**: controle por tensão → driver simples; baixíssimo gate current em regime; linear em potência por área. MOSFET é a base de toda eletrônica digital moderna (CMOS).

### CMOS

Complementary MOS: par NMOS + PMOS. Consumo estático ~zero (apenas durante comutação). Toda CPU/GPU/SoC moderna é CMOS. Trade-off fundamental: frequência × potência dinâmica (`P ∝ C · V² · f`).

### Amplificador Operacional (Op-Amp)

IC com: entrada diferencial (V+, V-), saída, alimentação (±V ou single supply).

**Regras de ouro** em circuito com feedback negativo:

1. Tensão: `V+ = V-` (virtual short).
2. Corrente nas entradas: ~0 (alta impedância).

Configurações básicas:
- **Seguidor (buffer)**: ganho 1, alta impedância de entrada. Isolamento.
- **Inversor**: `Vout = -(R_f / R_in) · Vin`.
- **Não-inversor**: `Vout = (1 + R_f / R_g) · Vin`.
- **Somador, subtrator, integrador, derivador**.
- **Comparador** (sem feedback): saída binária.

Parâmetros: GBW (gain-bandwidth product), slew rate, offset, CMRR, noise. LM358, TL072, LM324, OPA2134 são clássicos.

## DC vs AC

### DC (Direct Current)

Tensão/corrente constante (ou quase). Baterias, fontes reguladas, circuitos lógicos.

### AC (Alternating Current)

Varia senoidalmente. Rede elétrica (50/60 Hz), áudio, rádio.

- **Amplitude** (pico, pico-a-pico, RMS): `V_RMS = V_pk / √2` em senoide.
- **Frequência** (Hz), **período** (T = 1/f), **fase**.
- **Impedância** (Z): generalização complexa de resistência. `Z_R = R`, `Z_C = 1/(jωC)`, `Z_L = jωL`, onde `ω = 2πf`.

### Reatância

- **Capacitiva** `X_C = 1/(ωC)`: diminui com frequência. Cap é curto em alta freq.
- **Indutiva** `X_L = ωL`: aumenta com frequência. Indutor bloqueia alta freq.

Diferencia capacitor (passa AC, bloqueia DC) de indutor (passa DC, bloqueia AC).

### Fasores

Sinais senoidais representados como números complexos — facilita álgebra de circuitos AC. `v(t) = V·cos(ωt + φ)` ↔ `V·e^(jφ)`.

## Análise no Domínio da Frequência

### Resposta em Frequência

`H(jω) = V_out / V_in` como função complexa de ω. Gráfico típico: **diagrama de Bode** — magnitude em dB vs log(ω) + fase em graus.

**Pólos e zeros**: raízes do denominador/numerador de H(s). Determinam estabilidade e formato da resposta.

### Filtros

- **Passa-baixas (LPF)**: atenua acima de f_c. Simples: RC (f_c = 1/(2πRC)).
- **Passa-altas (HPF)**: atenua abaixo de f_c.
- **Passa-faixa (BPF)**: janela entre f_low e f_high.
- **Rejeita-faixa (notch)**: remove faixa estreita.

**Passivos** (RC, RL, LC) têm queda de 20 dB/década/pólo. **Ativos** (com op-amp): ordens maiores, ganho. Topologias: Sallen-Key, multiple-feedback, state-variable.

**Resposta**: Butterworth (plano na banda), Chebyshev (ripple mas rolloff agressivo), Bessel (fase linear), elíptico (rolloff máximo mas ripple em ambas bandas).

**Digital**: FIR/IIR implementados em DSP/FPGA/MCU.

## Transformador

Dois (ou mais) enrolamentos acoplados magneticamente. `V1/V2 = N1/N2`, `I1/I2 = N2/N1` (ideal).

**Aplicações**: step-up/down AC, isolamento galvânico, casamento de impedância em áudio/RF.

## Semicondutores

Material entre condutor e isolante (silício, germânio, GaAs, GaN, SiC). Dopagem:

- **Tipo N**: excesso de elétrons (fósforo em Si).
- **Tipo P**: déficit (buracos; boro em Si).

**Junção P-N**: zona de depleção cria barreira → diodo.

- **Bandgap**: energia para elétron pular da banda de valência à de condução. Si 1.12 eV; GaN 3.4 eV; SiC 3.2 eV.
- **SiC/GaN**: wide bandgap → mais eficiência em alta potência/frequência. Revolução em SMPS, carros elétricos, RF.

## Eletrônica Digital

Sinais discretos (0/1), representados por níveis de tensão (ex.: TTL: 0 = 0-0.8V, 1 = 2-5V; LVCMOS 3.3V; LVCMOS 1.8V).

### Lógica Booleana

- Operadores: AND, OR, NOT, XOR, NAND, NOR.
- NAND e NOR são **universais** — podem implementar qualquer função.
- **Leis de De Morgan**: `¬(A·B) = ¬A + ¬B`, `¬(A+B) = ¬A · ¬B`.
- **Mapas de Karnaugh**: simplificação visual até 4-5 variáveis.
- **Quine-McCluskey**: algorítmico, escala melhor.

### Portas Lógicas

Implementadas em CMOS (par de NMOS/PMOS). Família 74xx (HC, HCT, AHC, LVC, etc.) e 40xx CMOS. Parâmetros: tempo de propagação (ns), consumo, fan-out.

### Combinacional

Saída depende apenas das entradas no momento. Multiplexadores, decoders, encoders, comparadores, somadores (half/full adder, ripple-carry, carry-lookahead).

### Sequencial

Saída depende de estados anteriores. Elementos de memória:

- **Latch (SR, D)**: sensível a nível.
- **Flip-flop (D, JK, T)**: sensível a borda de clock.
- **Registradores**: cadeia de flip-flops.
- **Contadores, máquinas de estado (FSM)**: Moore/Mealy.

### CPU Simplificada (HDL mental)

Caminho de dados: ALU + registradores + memória + PC + controle. Pipeline quebra em etapas (fetch-decode-execute-memory-writeback). Ver `COMPUTER_THEORY.md` para teoria.

### HDL

- **Verilog, SystemVerilog, VHDL**: descrevem hardware (paralelo, por natureza).
- Simulação (ModelSim, Icarus, Verilator) antes de síntese.
- Síntese → netlist → fitter → bitstream (FPGA) ou máscara (ASIC).

### FPGA vs ASIC

- **FPGA** (Xilinx/AMD, Intel/Altera, Lattice, Efinix, open-source com yosys/nextpnr): programável, desenvolvimento rápido, caro em volume.
- **ASIC**: otimizado por design, barato em volume, altíssimo NRE (centenas de k$ a dezenas de M$).

## Amplificadores

### Classes

- **Classe A**: transistor conduz sempre; baixa distorção, baixa eficiência (~25%).
- **Classe B**: dois transistores em push-pull, cada um metade do ciclo. Eficiência ~78%, distorção de cruzamento.
- **Classe AB**: bias pequeno elimina cruzamento. Padrão em áudio analógico.
- **Classe D**: chaveamento PWM. Eficiência > 90%. Áudio moderno.
- **Classe E, F, G, H**: RF e especializadas.

### Feedback

Feedback negativo reduz distorção e aumenta linearidade ao custo de ganho. Base do op-amp. Feedback positivo → oscilação (útil em osciladores, destruidor em amps).

### Estabilidade

Sistema com feedback pode oscilar se ganho > 1 com fase de 360° em alguma freq (critério de Barkhausen). Margem de fase/ganho mede quão próximo está da instabilidade. Bode plots analisam.

## Eletrônica de Potência

### Retificação

AC → DC.

- **Meia-onda**: 1 diodo; perda de metade do ciclo.
- **Onda completa**: ponte de 4 diodos. Padrão.
- **Filtro capacitivo** após: reduz ripple.

### Reguladores Lineares

- **LM317, 7805**, LDOs (low drop-out): simples, baixo ruído, mas dissipam diferença como calor. `P = (Vin - Vout) · I`.
- Bom para analógico sensível.

### SMPS (Switched-Mode Power Supply)

Alta eficiência (>90%) via chaveamento. Topologias:

- **Buck** (step-down): Vout < Vin.
- **Boost** (step-up): Vout > Vin.
- **Buck-boost, SEPIC, Ćuk**: inversão, Vout acima ou abaixo.
- **Flyback, forward, push-pull, half-bridge, full-bridge**: isolados (transformador).
- **Resonant (LLC)**: baixa perda, alta densidade.

**Componentes**: chave (MOSFET/IGBT), indutor, diodo (ou retificador síncrono), cap. **Controle** por PWM de frequência fixa ou ripple-based.

### Conversores DC-DC

Módulos integrados (TI, Analog, ST, MPS, Vicor) simplificam — apenas alguns passivos externos.

### Inversores

DC → AC. Base de UPS, sistemas solares, VFD para motores.

## Instrumentação

### Multímetro

- **Tensão DC/AC, corrente DC/AC, resistência, continuidade, diodo, capacitância, frequência, temperatura**.
- **True RMS** em AC — essencial para sinais não-senoidais.
- Classe: 3½ dígito (~2000 counts) a 6½ dígito (laboratório).

### Osciloscópio

Visualiza sinal ao longo do tempo.

- **Bandwidth** (50 MHz-8 GHz): frequência máxima útil.
- **Sample rate** (GS/s).
- **Canais** (2-8).
- **Sondas** (10x, diferenciais, corrente).
- **Protocolos decode** (I2C, SPI, UART, CAN — embutidos em modelos modernos).
- **MSO (Mixed Signal)**: canais digitais adicionais.

Marcas clássicas: Keysight, Tektronix, Rohde&Schwarz; accessible: Rigol, Siglent, Hantek, PicoScope.

### Analisador Lógico

Captura sinais digitais em canais (8-32+). Útil para decode de protocolos. Saleae é referência moderna. Barato: DSLogic, clones USBee.

### Gerador de Sinal

- **Função**: seno, quadrada, triangular, DC.
- **Arbitrário (AWG)**: forma de onda customizada.
- **RF sig gen** para modulações.

### Analisador de Espectro / VNA

- **SA**: domínio da frequência. Essencial em RF.
- **VNA (Vector Network Analyzer)**: parâmetros S (reflexão, transmissão). NanoVNA democratizou acesso.

### Outros

- **Power supply (bench)**: tensão ajustável, current limit. Rigol DP800, Keysight E36xxA.
- **Carga eletrônica**: para testar fontes.
- **Câmara térmica**: hotspots em PCB. FLIR, Seek.
- **LCR meter**: precisão em L, C, R.

## PCB — Printed Circuit Board

### Processo

1. **Esquemático** (KiCad, Altium, Eagle, OrCAD).
2. **Assignment / footprints**.
3. **Layout** (placement + routing).
4. **DRC (Design Rule Check)**.
5. **Gerber files** → fabricação (JLCPCB, PCBWay, OshPark).
6. **Montagem** (SMT pickup, hand-solder, reflow).

### Camadas

Típico: 2 camadas (hobby), 4 camadas (sinal+GND+power+sinal), 6+ (BGA, DDR, alta densidade).

### Regras críticas

1. **Plano de terra sólido**: reduz loops, crosstalk, EMI. Evite cortes.
2. **Return path**: toda corrente volta; deve ter caminho curto (abaixo da trilha).
3. **Decoupling**: 100nF cerâmico próximo a cada VDD de IC; bulk 10µF na entrada.
4. **Trilhas curtas** em alta freq; vias aumentam indutância.
5. **Impedância controlada** em > ~100 MHz: largura da trilha + dielétrico + plano → Z0 (50Ω / 75Ω / 100Ω diff).
6. **Separação analógica ↔ digital**: planos separados com junção em um ponto.
7. **Thermal relief** em pads ligados a plano — facilita solda.
8. **Teardrops** em vias reduzem stress.
9. **Silkscreen claro**: referência dos componentes.
10. **DFM (Design for Manufacturing)**: tamanhos mínimos de via, clearance, etc.

## Sinal e Integridade

### Crosstalk

Sinal em trilha vizinha induz em outra. Reduzir: espaçamento, planos de referência, routing perpendicular entre camadas.

### Reflexão e Casamento

Em linhas de transmissão (alta freq), descasamento de impedância causa reflexão. Terminação:

- **Série**: resistor próximo ao driver.
- **Paralelo**: na carga.
- **Thevenin**: dois resistores.
- **AC termination** (R + C).
- Differential pairs (USB, HDMI, PCIe, DDR): 100Ω diff.

### Jitter e Skew

- **Jitter**: variação temporal de borda.
- **Skew**: diferença de chegada entre bits de um bus paralelo.

High-speed design minimiza ambos — bus serializado (PCIe, SATA, DisplayPort) substitui paralelo.

### EMI / EMC

- **Emissão**: circuito gera interferência.
- **Suscetibilidade**: circuito vítima.
- **Mitigação**: ferrites, filtros, blindagem, plano de terra, stackup cuidadoso, cabos torcidos/blindados.
- **Compliance**: FCC Part 15 (EUA), CE EN 55032/55035 (UE), CISPR.

## Protocolos de Comunicação Embarcados

Mais em `IOT_FIRMWARE_SECURITY.md` (ângulo de segurança). Aqui, resumo técnico.

### UART

Assíncrono, ponto-a-ponto, 2 fios (TX, RX). Taxas comuns: 9600, 115200, 1M. Sem clock compartilhado → ambos acordam em baud rate.

### SPI

Síncrono, full-duplex, master-slave (múltiplos via CS). 4 fios (MISO, MOSI, SCLK, CS). Rápido (MHz-GHz), simples. Padrão para flash, displays, ADCs.

### I2C

Síncrono, half-duplex, multi-drop. 2 fios (SDA, SCL) com pull-up. Endereçamento (7/10 bit). Velocidades: 100k / 400k / 1M / 3.4M. Simples fiação; lento para high-speed.

### CAN

Diferencial, multi-master, com arbitração e CRC. Automotivo, industrial. 1 Mbps (CAN) / 5 Mbps (CAN-FD) / 10+ (CAN-XL).

### RS-485

Diferencial, half/full-duplex, até 1200m em baixa velocidade. Industrial. Base de Modbus RTU.

### USB

Host/device, stacks ricos, enumeração, classes (HID, CDC, mass-storage). De USB 2.0 (480 Mbps) a USB4 (40 Gbps).

### Ethernet

10/100/1G/2.5G/10G. PHY discreto ou integrado. PoE adiciona alimentação via par.

### Wireless

- **Bluetooth Classic / BLE**.
- **Wi-Fi** (802.11).
- **Zigbee, Thread, Z-Wave**: baixa potência.
- **LoRa, NB-IoT, LTE-M**: longa distância.

## Microcontroladores e SoCs

### MCU populares

- **ARM Cortex-M (STM32, nRF, MSP432, i.MX RT, ESP32 Xtensa/RISC-V)**: dominantes.
- **AVR (Atmega328 do Arduino UNO)**: 8-bit, simples, legado.
- **PIC**: industrial.
- **RP2040 (Raspberry Pi Pico)**: dual-core M0+, PIO, ótimo para hobby.
- **ESP32**: Wi-Fi/BT integrado. Ubíquo em IoT.
- **RISC-V**: crescente — CH32V, GD32V, BL602.

### SoCs

Linux embarcado em Cortex-A (i.MX, Allwinner, Broadcom/RPi, Rockchip, Amlogic, Qualcomm). Bootloader → kernel → userspace.

### RTOS

FreeRTOS, Zephyr, ThreadX/Azure RTOS, ChibiOS, NuttX. Ver `ROBOTICS.md`.

## Fabricação e Prototipagem

### Solda

- **Through-hole**: fácil manualmente.
- **SMD**: 0805 (fácil), 0603 (ok), 0402 (desafiador), 0201 (microscópio).
- **QFN, BGA**: hot-air ou reflow; BGA exige X-ray para inspecionar.
- **Leaded vs lead-free (RoHS)**: temperatura diferente. SnPb derrete ~183°C; SAC305 ~217°C.

### Ferramentas

- **Ferro de solda regulado** (Hakko, Weller, Pinecil).
- **Ar quente** para SMD.
- **Estação de microscópio** para precisão.
- **Soldering flux**, **solder wick**, **dessoldering pump**.
- **ESD safe workstation** (esteira aterrada, pulseira).

### Protoboard, perfboard, milled PCB

Prototipagem rápida antes de layout final.

## Segurança Elétrica

### Choque

Corrente > ~10 mA AC através do corpo é perigosa; ~100 mA pode fibrilar. Resistência da pele seca ~10kΩ; molhada ~1kΩ.

- **Classes**: classe I (terra de proteção), II (isolação dupla), III (SELV baixa tensão).
- **Rede elétrica**: 127/220V AC. Sempre assumir viva até provar descarregado.
- **Capacitores carregados** em fontes podem matar após desligada. Descarregar com resistor antes de tocar.

### ESD (ElectroStatic Discharge)

Corpo humano acumula kV facilmente. Descarga em IC sensível mata. Mitigar: pulseira ESD, esteira, sacos antiestáticos, ambiente controlado.

### Isolação

Isolação galvânica (transformador, optoacoplador, isolador digital) separa referencias. Exigida em medicina, industrial, equipamentos de testes em rede.

### Fusíveis e Proteção

- **Fusível**: abre sob corrente excessiva.
- **PTC (polyfuse)**: auto-reset após resfriar.
- **TVS, MOV**: supressão de transients.
- **Disjuntor DR (GFCI)**: diferença entre fase e neutro → desliga.

## Temas Modernos

### Energia Renovável

Painéis fotovoltaicos + MPPT + inversor. Microinversores substituem central. Baterias Li (LiFePO4 ganhando terreno por segurança).

### Veículos Elétricos

Motores PMSM / inversores trifásicos / baterias (thousands of cells). Gerenciamento térmico e BMS críticos. SiC/GaN em inversores de alta eficiência.

### Internet das Coisas

Convergência MCU + wireless + sensores + cloud. Ver `ROBOTICS.md`, `IOT_FIRMWARE_SECURITY.md`.

### Computação Quântica

Circuitos criogênicos, qubits supercondutores (Josephson junctions), controle em microondas. Fronteira.

### Mixed-Signal

Conversão A/D e D/A. ADCs (SAR, pipeline, sigma-delta), DACs. SDR (Software Defined Radio) é mixed-signal levado ao extremo — RF digital.

### MEMS

Micro-ElectroMechanical Systems. Acelerômetros, giroscópios, microfones em smartphones. Sensores industriais.

### Biomédica

ECG, EEG, pulse oximetry. Front-ends de ultra-baixo ruído, acoplamento AC, proteção ESD reforçada, isolação galvânica.

## Ferramentas de Design Open

- **KiCad** (esquemático + layout; open-source sério).
- **LTspice** (simulação; gratuita mas closed).
- **Qucs-S, ngspice, Xyce** (simulação open).
- **FreeCAD** para mecânica.
- **Ngspice, Verilog-A** para modelos.

## Recursos

- **"The Art of Electronics"** — Horowitz & Hill. **A** referência.
- **"Practical Electronics for Inventors"** — Scherz.
- **"High-Speed Digital Design"** — Howard Johnson.
- **"Planar Microwave Engineering"** — Thomas H. Lee (RF).
- **Phil's Lab, EEVblog, GreatScott!, bigclivedotcom** (YouTube).
- **AllAboutCircuits, SparkFun, Adafruit** tutoriais.
- **Ridley Engineering** papers sobre SMPS.
- **EMC Europe, DesignCon** conferências.

## Princípios

1. **Plano de terra é sagrado**. Maioria dos bugs vem de retorno mal roteado.
2. **Decouple everywhere**: cap cerâmico próximo de todo VDD.
3. **Meça antes de assumir**. Multímetro + osc antes de achar que sabe.
4. **Start simple, layer complexity**: começa com LED pisca; adiciona peça por vez.
5. **Folha de dados é bíblia**. Ler tudo do IC antes de usar.
6. **Projete para falhar bem**: fusíveis, TVS, clamp diodes. Hardware não tem `try/except`.
7. **Simule antes de prototipar; prototipe antes de produzir**. LTspice barato; PCB revisão não.
8. **Respeite eletricidade**. Tensão alta não perdoa. Trabalhe desligado, descarregue, isole.
