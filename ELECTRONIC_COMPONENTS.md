# Componentes Eletrônicos

Catálogo técnico dos componentes que aparecem em projeto real: o que escolher, que parâmetro de datasheet ignorar é barato e qual é caro, armadilhas. Complementa `CIRCUIT.md` (leis e topologia) e `ENERGY.md` (potência em escala de sistema).

Não é introdução — para fundamentos (V=IR, leis de Kirchhoff, conceitos básicos), ver `CIRCUIT.md`.

## Princípio norteador

Componente real ≠ modelo ideal. Todo componente tem:

- **Parasíticos** (capacitância, indutância, resistência não-ideais).
- **Derating** (operar em fração da spec máxima).
- **Vida** (função de stress: tensão, temperatura, corrente, ciclagem).
- **Tolerância** (variação lote-a-lote, com temperatura, com tempo).
- **Package** (tamanho físico define capacidade térmica e mecânica).

Datasheet tem 30 páginas por um motivo. Lê ou paga depois.

## Resistores

### Parâmetros críticos

- **Valor nominal** (Ω) + **tolerância**: 0.1%, 0.5%, 1%, 5%, 10%. 1% virou default SMD.
- **Potência** (W): 1/16W (0402), 1/10W (0603), 1/8W (0805), 1/4W (1206), 1W+ (wirewound/planar).
- **Coeficiente de temperatura (TC)**: ppm/°C. Carbono ~1000, film metálico 100, Vishay foil 0.2.
- **Tensão máxima**: em pequenos SMD pode ser só 50-200V. Divisor de alta tensão exige séries.
- **Ruído** (µV/V): carbono é ruidoso, wirewound/foil silencioso.
- **Indutância parasita**: wirewound tem indutância, ruim em RF. Film metálico/non-inductive para alta freq.

### Tipos

- **Carbon composition**: legado. Alta tolerância (5-10%), alto ruído. Ainda em amp de guitarra por "sonoridade".
- **Carbon film**: barato, 5% típico. Consumer geral.
- **Metal film**: 1%, baixo TC, baixo ruído. Padrão em SMD analógico.
- **Thick film** (SMD comum): misto de metal + vidro. 1-5%, TC modesto. Absoluto default.
- **Thin film**: precisão 0.1%, TC <25 ppm/°C. Medição, instrumentação.
- **Metal foil (Vishay Bulk Metal Foil)**: 0.005%, TC 0.2 ppm. Referência científica.
- **Wirewound**: alta potência, precisão ok, indutivo.
- **Power (cement, wirewound alum-clad, planar)**: 5-500W. Heatsink-mounted.
- **Shunt de corrente**: 4-wire (Kelvin) conexão; miliohm. PCW SMD (CRL2512 etc), busbar em alta corrente.
- **Array/network**: 4-8 resistores em um package; matched para divisor/terminação.

### Resistores especiais

- **Thermistor NTC**: R cai com temperatura. Medição de temperatura (`-β(1/T - 1/T₀)`), inrush limiting em fonte.
- **Thermistor PTC**: R sobe abruptamente acima de curie. Auto-reset fuse, aquecimento self-limiting.
- **Varistor (MOV, metal oxide)**: R cai em sobretensão → absorve energia. Proteção contra surge (raio). Desgasta: troque após eventos.
- **LDR / fotorresistor**: R cai com luz. Legado (CdS).
- **Strain gauge**: R muda com deformação (≈0.3% full-scale). Ponte de Wheatstone + instrumentation amp. Sensores de força, torque, pressão.
- **Rheostat / potenciômetro**: variável. Cermet (precisão), condutive plastic (vida longa), wirewound (potência).
- **RTD (PT100, PT1000)**: platina; linearidade excelente em ampla faixa. Padrão industrial.

### Armadilhas

- **Pull-up em linha rápida**: resistor + capacitância parasita = tempo de subida. 10k + 10pF = τ=100ns → bordas lentas.
- **Shunt em corrente alta**: efeito termoelétrico em soldas dissimilares gera mV de erro. Kelvin + low-EMF solder.
- **Divisor de alta tensão**: derating de tensão → use séries (3× 100k em série → 300k com triple da tensão).
- **Potência de pico vs contínua**: pulsos curtos permitem 5-10× nominal (curva em datasheet).

## Capacitores

### Dielétricos cerâmicos

**Classe 1** (estável, não-ferroelétrico):
- **C0G/NP0**: tolerância 1-5%, TC <30 ppm/°C, **zero drift com tensão**, zero aging. Até 1 µF em 0603. Padrão em filtro e timing.

**Classe 2** (ferroelétrico, altos valores mas drift):
- **X7R**: ±15% em -55 a 125°C. Padrão "general purpose".
- **X5R**: ±15% em -55 a 85°C. Menos temperatura, mais C/volume.
- **Y5V / Z5U**: ±22/-82% nas pontas. **Evite** em qualquer coisa séria.

### MLCC derating (a armadilha mais comum)

MLCC classe 2 perde capacitância com tensão aplicada (DC bias effect). Típico:

- X7R 10µF/25V em 12V DC: **perde 50%** → 5µF real.
- X7R 22µF/6.3V em 5V DC: **perde 70-80%** → 4-6µF real.

**Regra prática**: nunca use MLCC com Vdc > 50% da spec. Dobre tensão de rating ou coloque 2× em paralelo. Faça rolltoff de frequência do filtro com C *real*.

Curvas no datasheet (Murata, TDK, Samsung). **Murata SimSurfing** permite simular.

### Outros dielétricos

- **Eletrolítico alumínio**: grande, polarizado, barato. ESR alto, vida limitada por eletrólito seco (10⁻³× a 20°C → t↓ 2×/10°C). Decoupling bulk, SMPS output.
- **Eletrolítico polymer (OS-CON, PolyCap)**: mesmo volume, ESR muito menor, vida 5-10× mais longa, sem secar. Preferível em novo design quando couber no BOM.
- **Tântalo**: estável, baixo ESR, pequeno. **Falha em curto explosivo** sob sobretensão/polaridade invertida. Sempre usar com 2× margin de tensão. Polymer tantalum (KEMET T520, AVX TPS) é mais seguro.
- **Filme (polyester, polypropylene, polystyrene, PEN, PET, PPS)**: audio, power factor, snubbers, X/Y safety. Polypropylene é king em audio/crossover (baixa distorção, baixa perda).
- **Mica**: alta precisão, baixa perda, alta estabilidade, ampla freq. Cara; RF e instrumentação.
- **Supercapacitor (EDLC)**: Farads, baixa tensão (2.5-3V). Backup de RTC/SRAM, energy harvesting, regenerative short-term.
- **Hybrid (alumínio + polymer)**: meio-termo; automotivo.

### Parâmetros além de C e V

- **ESR** (Equivalent Series Resistance): perdas, ripple, aquecimento. Alto ESR em SMPS output = thermal runaway.
- **ESL** (Equivalent Series Inductance): limita resposta em alta freq. MLCC 0402 tem ~0.5 nH; eletrolítico radial ~10 nH.
- **SRF** (Self-Resonant Frequency): acima, capacitor vira *indutor*. 10µF MLCC pode ter SRF ~5 MHz. Por isso decoupling usa múltiplos capacitores de valores diferentes (100n + 10n + 1n) em paralelo.
- **Dissipation factor (tan δ)**: perdas relativas; crítico em audio, RF.
- **Ripple current**: em SMPS output, corrente AC aquece por ESR. Derating por temperatura.
- **Vida (h)**: eletrolítico tem spec em 105°C; cada 10°C abaixo dobra vida.

### Aplicações e escolha

- **Bypass/decoupling digital**: 100nF X7R 0402 perto de cada pino de alimentação de IC. Plus bulk eletrolítico/polymer na entrada do regulador.
- **SMPS input/output**: MLCC + polymer + electrolytic em paralelo pra cobrir banda e ripple.
- **Timing (oscilador, ADC sample-hold)**: C0G.
- **Acoplamento de áudio**: filme ou eletrolítico dedicado audio (Nichicon KZ, MUSE).
- **Y cap (line-to-ground)**: safety rating (Y1/Y2); não pode falhar em curto. Filme específico.
- **X cap (line-to-line)**: safety (X1/X2); falha aceita mas deve queimar aberto.
- **Snubber**: RC em paralelo com chave ou diodo pra absorver ringing. Filme específico de alta dV/dt.

### Armadilhas

- **MLCC piezoelétrico**: classe 2 vibra ao ouvir áudio próximo; vira microfone. Ruído acústico em fontes. Classe 1 não.
- **Crack por flexing de PCB**: MLCC grande (1210+) racha em panelização. Use menor package ou flexible termination.
- **Polaridade invertida**: eletrolítico e tântalo explodem. MLCC não tem polaridade.

## Indutores

### Parâmetros

- **Valor nominal** (nH, µH, mH, H).
- **Tolerância** (±10-20% comum; 1% em RF).
- **Saturation current (Isat)**: acima, núcleo satura, L cai abruptamente. Curvas no datasheet.
- **Corrente térmica (Irms)**: aquecimento por I²R na bobina. Menor que Isat geralmente.
- **DCR** (DC resistance): perdas estáticas.
- **SRF**: como capacitor, acima vira capacitor (capacitância interbobinar domina).
- **Q factor**: ωL/R em freq de operação; alto = baixa perda.

### Núcleos

- **Ar**: Q alto, L baixo. Solenóide em RF.
- **Ferrite** (MnZn, NiZn): alta permeabilidade, saturação a baixa B. SMPS, EMI filtering. NiZn melhor em >1 MHz.
- **Iron powder**: saturação soft (gradual). SMPS output choke, high-current.
- **Molypermalloy powder (MPP)**: low-loss, estável. SMPS premium.
- **Sendust, Kool Mu**: baratos, saturação gradual.
- **Amorphous, nanocrystalline (Finemet, Vitroperm)**: baixa perda em alta freq. EV, solar.
- **Laminado de aço silício**: transformador 50/60 Hz.

### Saturação soft vs hard

Ferrite satura "duro" (L cai de 100% a 50% em pequeno ΔI). Powder/iron satura "soft" (queda gradual ao longo de faixa ampla). **Ferrite quebra projeto** se Isat for excedido em pico de corrente; powder tolera.

### Tipos em projeto

- **Axial / radial através-furo** (choke): legado, potência média.
- **SMD shielded (molded)**: 4-20 µH típico, 1-10 A. SMPS buck/boost moderno.
- **Toroidal**: baixo EMI radiado. Custoso de enrolar.
- **Bead ferrite** (EMI bead): "resistor em AC, zero em DC". Filtro de linha de alimentação.
- **Common-mode choke**: duas bobinas em mesmo núcleo, sentidos opostos; cancela corrente diferencial, impede modo comum. Filtragem EMI em data line (USB, Ethernet) e AC line.
- **Transformer**: dois enrolamentos acoplados. Isolação, step-up/down, flyback. Planar PCB-integrated para SMPS de alta densidade.

### Armadilhas

- **Saturação em transiente**: startup de SMPS com Cout grande dá pico de corrente alto. Dimensione Isat com margem.
- **Ruído acústico**: núcleo ferrite vibra em audio se corrente varia nessa faixa. Emcapsulamento resina mitiga.
- **Acoplamento indesejado**: indutor próximo a outro induz tensão. Espaçar ou orientar 90°.

## Cristais, osciladores e ressonadores

### Quartz crystal

Piezoelétrico, frequência determinada por corte + dimensão. Q altíssimo (10⁴-10⁶). Tolerância 10-50 ppm; drift com temperatura em forma cúbica (corte AT) ou parabólica.

- **Load capacitance (CL)**: especificado; PCB deve prover match (CL1+CL2 balanced). Erro de match = frequência off.
- **ESR**: perdas; afeta startup.
- **Drive level**: potência máxima. Sobre-drive = aging acelerado.
- **Temperature-compensated (TCXO)**: ±0.1-2 ppm; circuito interno compensa. GPS, comunicação.
- **Oven-controlled (OCXO)**: ±0.01 ppm; forno mantém quartz em ponto estável. Telecom, rádio HF.

### MEMS oscillator

Silício ressonante + PLL + sensor de temperatura + compensação digital. Alternativa moderna a XO.

- Vantagens: shock resistance, small, drop-in replacement for quartz, wide frequency via PLL.
- Desvantagens: phase noise maior que quartz premium, consumo levemente maior.
- Vendors: SiTime, Microchip DSC, Abracon.

### SAW (Surface Acoustic Wave)

Filtro/ressonador em GHz. Celular (front-end), RF. Não é "oscilador", é filtro de banda.

### Ceramic resonator

Baixo custo, baixa precisão (±0.5%). µC onde serial UART tolera. Não usar em wireless, RTC.

### RTC crystal 32.768 kHz

Tuning-fork quartz; baixo consumo; deriva ~20 ppm → 1 min/mês. RTC dedicado.

## Componentes EMC / supressão

- **Ferrite bead**: perfil de impedância Z(f). 600 Ω em 100 MHz, ~0 Ω em DC. Limpa ruído de alimentação. Escolha Z no pico de ruído do sistema.
- **Common-mode choke**: 90% dos filtros EMI em AC line. USB, Ethernet, HDMI, automotive CAN também (CMCs pequenos).
- **Y cap / X cap**: safety-rated, safety-agency-tested. Nunca substitua por MLCC comum em linha AC.
- **TVS** (Transient Voltage Suppression): ver seção diodos.
- **Gas discharge tube (GDT)**: centelhador; surge protection principal em antena/linha telefônica.

## Diodos — família extendida

### Retificador standard (1N4001-1N4007)

Junção PN silício, Vf ~0.7 V, PIV 50-1000 V, 1 A típico. Recuperação reversa lenta (~2 µs) → não usar acima de 100 kHz.

### Fast / Ultrafast recovery

Recuperação em <100 ns (fast), <50 ns (ultrafast). SMPS offline.

### Schottky

Junção metal-semicondutor. Vf 0.3-0.4 V, sem minority carrier → recuperação quase instantânea. Alta corrente, baixa Vr (geralmente <100 V). Padrão em SMPS de baixa tensão.

- **SiC Schottky**: Schottky de carbeto de silício; 600-1700 V. Mata MOSFET body diode em PFC, inversor.

### Zener

Breakdown reverso controlado. Vz de 2.4 V a 200 V.

- **True Zener** (<5 V): efeito Zener, TC negativo.
- **Avalanche** (>6 V): efeito avalanche, TC positivo.
- ~5.6 V: TC zero — referência intermediária estável.
- Aplicações: ref grosseira, proteção.

### TVS (Transient Voltage Suppression)

Zener rápido, alta energia, low clamping voltage. Protege contra ESD, surge, load dump automotivo.

- **Unidirecional** (cátodo a GND) para rails DC.
- **Bidirecional** para linhas AC / data.
- Ratings: Vrwm (working), Vbr (breakdown), Vclamp @ Ipp.
- Séries: SMAJ, SMBJ (Littelfuse), ESD7xxx (ST) para USB/HDMI.

### LED

Vf depende da cor:
- Vermelho ~1.8-2 V
- Amarelo ~2.2 V
- Verde ~2.2-3.3 V (InGaN)
- Azul ~3.2 V
- Branco ~3-3.6 V (blue + phosphor)
- UV ~3.5-4 V
- IR ~1.3-1.7 V

Driver: resistor (simples, não-eficiente), LED driver CC (buck regulator).

- **Power LEDs**: 1-10W, exigem heatsink (junção <150°C).
- **RGB**, **adressable (WS2812/SK6812 NeoPixel)**: driver integrado no encapsulamento.
- **LED matrix / dots display**: alto pin count → multiplexing.

### Laser diode

LED com cavidade ressonante. Threshold current abrupto. Requer current regulation com proteção (ESD mata instantâneo). Fiber optic, barcode, laser módulo.

### Photodiode, phototransistor, solar cell

Junção exposta à luz. Current proporcional à iluminância.

- **PIN**: rápido, alta sensibilidade, transimpedance amp.
- **APD** (avalanche): ganho interno, single-photon detection com bias alto.
- **Solar cell**: PIN grande otimizado para fóton → elétron com máx potência.

### Varactor

Cap reverso-polarizado usa capacitância de depleção como função de V. VCO, PLL, tuning.

### Diodos especiais

- **Tunnel diode**: resistência diferencial negativa. Uso histórico, RF exótico.
- **IMPATT, Gunn**: fonte RF em mm-wave (radar).
- **Step-recovery diode (SRD)**: geração de harmônicas (frequency multiplier).

## BJT

### Regiões de operação (curva IV)

- **Corte**: Ib = 0, Ic = 0.
- **Ativo**: VBE ~0.7 V, Ic = β·Ib (ou mais precisamente Ic = IS·exp(VBE/VT)).
- **Saturação**: VCE pequeno (~0.2 V), Ic limitado por circuito externo.
- **Breakdown**: VCE excede BVCEO → avalanche.
- **Inverso**: roles de E e C trocados; β baixíssimo.

### Parâmetros críticos

- **β (hFE)**: ganho; varia 2-3× lote e 2-3× com temperatura. Design defensivo: suponha mínimo do datasheet.
- **fT**: transit frequency. Ganho cai a 1 acima de fT/β.
- **Early voltage (VA)**: VCE estende Ic no ativo; modela resistência de saída.
- **SOA (Safe Operating Area)**: DC + pulsed limits em V·I. Second breakdown em BJT é thermal runaway localizado.
- **VCEsat**: mínimo VCE em saturação forçada.

### Tipos

- **Small signal** (2N3904, BC547): genérico.
- **High-β (Darlington)**: β ~1000+; VBEon dobrado (~1.4 V); lento.
- **High-speed switching (2N2222)**.
- **Power (2N3055, MJL3281A)**: Icmax ~15-30 A, Vceo 200-350 V.
- **RF (2N5179, BFR93)**: fT alto, pequeno.
- **Darlington Power (TIP142)**: par Darlington integrado.

### Quando BJT ainda vence

- **Baixa tensão precisa**: VBE matched pair em par diferencial (LM394, MAT02) — ainda referência em amp de instrumentação baixo ruído.
- **Audio linear**: topologia classic, amplificadores high-end.
- **High-current switching barato**: driver de relé, solenóide.

Em 95% dos outros casos, MOSFET ganha.

## MOSFET — detalhes de potência

Transistor dominante. Controlado por tensão; gate tem capacitância que precisa ser carregada/descarregada rapidamente.

### Parâmetros

- **Vgs(th)**: limiar de condução. 1-4 V típico em logic-level; 2-4 V standard.
- **RDS(on)**: resistência no estado ligado, como função de Vgs. Sempre specifiqued em Vgs específico (4.5 V ou 10 V).
- **ID(max)**: current contínuo com heatsink ideal; realistic usage é menor.
- **VDS(max)**: ruptura dreno-source.
- **Qg (gate charge total)**: quanta carga para levar gate de 0 a Vgs. Tempo de chaveamento = Qg / Ig(drive).
- **Qgd (Miller charge)**: dominante em switching loss; tempo em Miller plateau.
- **Capacitâncias Ciss, Crss, Coss**: ressonâncias, EMI.
- **Body diode**: Vf, trr (reverse recovery). Crítico em synchronous rectification e meia-ponte.
- **SOA**: para MOSFET de alta tensão, existe thermal instability em Vgs alto + V
DS médio (derating específico).
- **Avalanche rating (EAS)**: energia que suporta em clamp de Vds; em aplicações flyback, driving indutivo.

### Topologias

- **Lateral MOSFET**: baixa potência, IC. Sensível a ESD.
- **Vertical DMOS / VDMOS / TrenchFET**: power MOSFET clássico. Dreno embaixo, source em cima.
- **SuperJunction (CoolMOS, Prime, GenC)**: colunas P-N verticais reduzem RDS(on) por tensão drasticamente. 600 V offline SMPS.
- **Shielded-gate / split-gate trench**: menor Qgd, melhor figure of merit.
- **LDMOS**: alta freq RF.

### Logic-level vs standard gate

- **Standard**: Vgs(th) 2-4 V, turn-on completo em Vgs=10 V.
- **Logic-level**: Vgs(th) 1-2 V, ok em 3.3-5 V drive. Microcontroller direto.

Confundir quebra projeto. Logic-level em Vgs=10 V geralmente ok; standard em Vgs=3.3 V fica em região linear com RDS(on) enorme.

### Wide-bandgap — GaN e SiC

**SiC (Carbeto de Silício)**:
- Bandgap 3× maior que Si.
- **MOSFET SiC** 650-1700 V, RDS(on) menor que Si equivalente, switching mais rápido, tolerância a alta temperatura (200°C junction).
- Uso: EV inverter traction, solar string inverter, PFC, charger fast.
- Vendors: Wolfspeed (Cree), ROHM, Infineon, ST, onsemi.

**GaN (Nitreto de Gálio)**:
- Ultra-rápido switching (ns). Menor Qg, menor Coss.
- **HEMT lateral** 100-650 V. Normally-on (cascode com MOSFET baixo-V torna normally-off).
- Uso: USB-C fast chargers (100W+), data center, RF de potência.
- Vendors: GaN Systems (Infineon), Navitas, EPC, Transphorm, Power Integrations, TI.

Trade-off: WBG são caros, exigem drive layout impecável (bounce, ringing amplificado pela velocidade), parasíticos devastam.

### Armadilhas

- **Drive lento**: gate não chega a Vgs(full) → MOSFET "passa" por região linear → aquece → morre.
- **Miller turn-on espúrio**: dV/dt no dreno acopla ao gate via Cgd → Vgs cresce → MOSFET liga parcialmente quando não devia. Gate driver com Miller clamp ou Vgs negativo em off.
- **Body diode em meia-ponte**: trr do body diode → shoot-through momentâneo. Use MOSFET com diodo rápido ou Schottky em paralelo; dead time.
- **ESD em gate**: gate é capacitor com óxido fino. Zener 15V entre gate-source em driver não-integrado.

## IGBT

Híbrido BJT + MOSFET: input MOS (impedância alta) + output BJT (VCEsat baixa).

- **Vantagem sobre MOSFET**: em tensões altas (>600 V) e correntes altas, VCEsat ~1.5-2 V enquanto RDS(on)·I pode ser maior. Menos condução.
- **Desvantagem**: tail current durante turn-off — elétrons minoritários recombinam lentamente. Limita freq a 20-50 kHz (vs MHz em MOSFET).
- **Uso**: inversores industriais, tração ferroviária, solda elétrica, indução, HVDC. SiC MOSFET está comendo mercado do IGBT 600-1200 V.

## Tiristores

Semicondutor de quatro camadas (PNPN). Latching: dispara e fica ligado até corrente zero (em AC, meio-ciclo).

- **SCR (Silicon Controlled Rectifier)**: unidirecional. Ângulo de fase em AC (dimmer, crowbar, partida soft).
- **TRIAC**: bidirecional. Dimmer AC clássico (lâmpada incandescente, não-dimmable LED).
- **DIAC**: trigger bidirectional pra TRIAC.
- **GTO (Gate Turn-Off)**: pode desligar via gate. Motor drive industrial legado.
- **IGCT (Integrated Gate-Commutated Thyristor)**: GTO melhorado. HVDC, motor drive heavy.

Tiristores estão em declínio frente a IGBT/SiC em novo projeto, mas instalados em massa industrial.

## JFET

Controlado por tensão inversa de gate (normally-on, pinched off em Vgs negativo).

- **Alta impedância de entrada** (GΩ).
- **Baixo ruído** em audio e medição.
- **Saturação quadrática**: `Id = Idss·(1 − Vgs/Vp)²`.
- Uso: front-end de fono preamp, eletrômetro, detector de partículas, chaves de áudio analógico (J201, 2N5457).

## Amplificador Operacional — topologias internas

### Arquiteturas

- **2-stage classical** (LM741, TL072): diferencial input → Miller cap → output stage. Simples, histórico.
- **Rail-to-rail input/output (RRIO)**: swings até os rails ± offset. Low voltage (single supply 3.3 V). Topologia complementar dupla pra input.
- **Zero-drift / auto-zero / chopper**: auto-calibração interna elimina offset. Microvolt-grade. MAX9615, LTC2050, AD8628. Médico, load cell, termopar.
- **Current-feedback (CFA)**: entrada invertente de baixa impedância, bandwidth independente de ganho. Video, alta velocidade (AD8009).
- **Fully-differential (FDA)**: saída diferencial; ADC driver. THS4551, LMH5401.
- **Transimpedance (TIA)**: convert I→V; fotodetector. Estabilidade com Cf calibrado pro Cin da photodiode.
- **JFET-input (TL072, LF356, OPA2134)**: alta Zin, low noise audio.
- **Bipolar-input (NE5532, OP27)**: low voltage noise, high current noise. Audio low-Z source.

### Parâmetros críticos

- **Vos** (input offset voltage): µV (zero-drift) a mV (general purpose). Amplifica em ganho DC.
- **Ib / Ios**: bias / offset current. JFET-input <pA; bipolar nA.
- **CMRR**: common-mode rejection. 80-120 dB típico.
- **PSRR**: power supply rejection.
- **GBW**: gain-bandwidth product (MHz). Bandwidth útil = GBW / ganho.
- **Slew rate** (V/µs): taxa máxima de subida da saída. Limita bandwidth de sinal grande.
- **Noise voltage (nV/√Hz)**: referred to input.
- **Noise current (pA/√Hz)**: multiplica por Zsource.
- **Settling time**: tempo para ±0.01% após step.

### Instrumentation amplifier

Três op-amps (ou chip dedicado AD620, INA128): alta Zin, ganho ajustável por Rg, **CMRR altíssimo**. Bridge de termopar, strain gauge, ECG.

### Comparadores dedicados

Op-amp sem feedback funciona mas tem problemas:
- Saturation recovery lenta.
- Slew rate modesto.
- Algumas falham em Vin fora de common-mode range.

Comparadores (LM339, MAX9203, TLV3501) têm output open-collector ou push-pull, histerese opcional, recovery rápido. Use específico.

## Referências de tensão

### Topologias

- **Zener simples**: 5-50 V, ±2-5%. Barato.
- **Buried Zener**: Zener enterrado sob superfície, low noise, low drift (LM399, LT1021, LT1027). 7 V típico.
- **Bandgap**: soma VBE (TC negativo) + ΔVBE amplificado (TC positivo) ≈ 1.25 V estável. Série LM385, LM4040, REF02, MAX6023.
- **XFET / FGA**: JFET charge-coupled (ADR series Analog Devices); baixo ruído.

### Parâmetros

- **Initial accuracy**: ±0.05% a ±1%.
- **Temperature drift** (ppm/°C): 2 (premium) a 100 (barato).
- **Long-term drift** (ppm/1000h): 5-50.
- **Noise** (µVpp em 0.1-10 Hz): menor melhor.
- **Line/load regulation**.

### Shunt vs series

- **Shunt**: LM4040, LM431. Tipo Zener integrada. Simples, requer resistor de polarização; corrente flui sempre.
- **Series**: 3 terminais (IN, OUT, GND), drop-out. MAX6023, REF5025. Melhor PSRR, dropout baixo.

Aplicações: ADC/DAC reference, current source, calibração. ADC 16-bit precisa reference com ruído <1 LSB em banda de sample.

## Reguladores lineares (LDO)

"LDO" (Low Dropout) é categoria moderna.

### Parâmetros

- **Dropout voltage**: Vin - Vout mínimo. 100 mV a 1.5 V (velho).
- **PSRR**: rejeição de ripple de Vin. Varia dramaticamente com frequência; datasheet mostra curva.
- **Load regulation**: ΔVout/ΔIload.
- **Line regulation**: ΔVout/ΔVin.
- **Quiescent current (Iq)**: autoconsumo. Crítico em sleep.
- **Transient response**: degraus de carga; definido por Cout.
- **Noise**: µVrms; ultra-low-noise LDOs (LT3042, ADP7118) <1 µV/√Hz para clock e ADC.

### Topologias

- **PNP pass transistor (tradicional, 78xx)**: dropout alto (~2 V), estável com Cout pequeno.
- **NPN**: dropout alto.
- **PMOS pass**: low dropout (100 mV), exige Cout com ESR específico pra compensação.
- **NMOS pass (charge pump gate drive)**: dropout baixo, Iq moderado.

### Escolha entre LDO e switcher

- **Dropout pequeno**: LDO (menos ruído, sem indutor). Ex: 5 V → 3.3 V.
- **Dropout grande + corrente alta**: switcher (eficiência). LDO de 12 V → 3.3 V a 1 A dissipa 8.7 W.
- **Noise-sensitive (ADC, oscilador)**: LDO depois de switcher pra pós-filtrar.
- **Espaço limitado + low current**: LDO.

## Switching regulators

Overview em `CIRCUIT.md` na seção "Eletrônica de Potência". Componente chave aqui: **gate driver**.

### Gate driver

Circuito que entrega corrente alta ao gate de MOSFET/IGBT. Especifica:

- **Peak current (A)**: carrega Qg rápido.
- **Propagation delay**.
- **Matched delay** entre canais high/low side.
- **Miller clamp**: mantém gate em GND durante dV/dt alto.
- **Desaturation detection**: mede Vds durante on; dispara shutdown em short.
- **Isolated** (optocoupler, magnético, capacitivo): barreira entre controle e potência.

Vendors: TI UCC, Infineon 1ED, Silicon Labs Si82xx, Analog Devices ADuM.

## Conversores ADC / DAC

### ADC topologias

- **SAR (Successive Approximation)**: iterativo bit-a-bit. 8-18 bit, até ~10 MS/s. General purpose. AD7606, MCP3421.
- **Sigma-Delta (ΣΔ)**: oversampling + noise shaping. 16-24 bit @ kHz. Audio, medição precisa, temperature. ADS1256, LTC2400.
- **Flash**: comparadores paralelos (2ⁿ-1). Ultra-rápido (GS/s), 6-10 bit, consumo alto. Video, scope.
- **Pipeline**: combinação flash + SAR em estágios. 12-16 bit @ MS/s. ADC de instrumentação, SDR.
- **Dual-slope integrating**: precisão linhe DVM. Lento (50-100 ms/sample), ~6 digits.
- **Time-interleaved**: N ADCs intercalados pra aumentar sample rate.

### DAC topologias

- **R-2R ladder**: resistores em rede. Linear, barato. 8-16 bit.
- **String / segmented**: string de resistores iguais + MUX. Monotonic garantido.
- **Current-steering**: fontes de corrente chaveadas; altíssima velocidade (GS/s). RF, video.
- **ΣΔ-DAC**: análogo dual do ADC ΣΔ; 24-bit audio (ESS SABRE, AKM AK4499).

### Parâmetros

- **Resolution** (bits): quantização mínima.
- **ENOB** (effective number of bits): real, contando ruído. Sempre < bits nominal.
- **INL / DNL**: integral / differential linearity.
- **SNR / SINAD / SFDR / THD+N**: figures of merit dinâmicas.
- **Sample rate**.
- **Power consumption**.

## Sensores

### Temperatura

- **NTC thermistor**: não-linear; linearização Steinhart-Hart.
- **RTD (PT100, PT1000)**: linear; 4-wire para remover lead resistance em alta precisão.
- **Thermocouple** (J, K, T, R, S, B): µV/°C; precisa cold-junction compensation (IC dedicado MAX31855, AD595).
- **Semiconductor IC** (LM35, TMP36, TMP117, DS18B20): µC-friendly. TMP117 é state-of-art com ±0.1°C.
- **Infrared** (MLX90614, MLX90640 array): sem contato.
- **Pt100 film / wire**: 100 Ω em 0°C; padrão industrial.

### Pressão

- **MEMS piezoresistive** (BMP280, BMP388, DPS310): absoluto, barômetro. Altímetro ~10 cm resolução.
- **Capacitive** (MS5611): baixa potência.
- **Gauge / absolute / differential**: zero reference.
- **Strain gauge + diaphragm**: industrial high-pressure.

### Inerciais (IMU)

- **Acelerômetro**: MEMS capacitivo. ADXL, LSM, BMI series. 3-axis, g-range variável.
- **Giroscópio**: MEMS ressonante. Drift é limitação.
- **Magnetômetro**: Hall ou AMR/GMR. Compass.
- **IMU combo (6-axis, 9-axis)**: BMI270, LSM6DS/LSM9DS, ICM-20948, BNO055 (c/ sensor fusion on-chip).

### Corrente

- **Shunt + amp**: resistor baixo + instrumentation amp. Isolation opcional (isoamp).
- **Hall-effect IC** (ACS712, ACS758, LEM): isolated, linear. Banda modesta.
- **CT (current transformer)**: AC only, ampla faixa. Métron domiciliar.
- **Fluxgate / Rogowski coil**: precisão alta, AC+DC (fluxgate), AC rápido (Rogowski).

### Luz

- **Photodiode**: linear.
- **LDR**: legado, lento.
- **Lux sensor IC** (BH1750, VEML7700, TSL2591): calibrado.
- **Color sensor** (TCS34725, AS7341).
- **ToF distance** (VL53L0X/L1X/L4CD): laser + SPAD array.

### Outros

- **Humidade** (SHT31, DHT22, BME280): resistive ou capacitive polymer.
- **Gás** (MQ series baratos, BME680, SGP30, MiCS): MOS heated.
- **PM2.5 / particulado** (SPS30, PMS7003).
- **pH, condutividade, O₂ dissolvido**: eletroquímicos.
- **Hall** (A1324, SS49): campo magnético + corrente.

## Optoacopladores / SSR

### Optocoupler

LED + phototransistor/PhotoMOS em package isolado. **CTR (Current Transfer Ratio)**: Io/Iin. Degrada com idade (20-50% em 10 anos). Usar com margin.

- Digital (4N35, PC817, TLP785): feedback de SMPS, nível de sinal.
- High-speed logic (6N136, HCPL): MHz.
- Gate driver isolado (HCPL-3120, FOD3120): drives MOSFET isolado.

### Solid State Relay (SSR)

Optocoupler + TRIAC/MOSFET integrado. Zero-cross detection em SSR de AC. Vida essencialmente infinita comparada a relé EM. Custoso, ESR em on-state.

- AC SSR (Crydom, Panasonic): zero-crossing, 10-100 A.
- DC SSR: MOSFET back-to-back.
- Omron G3NA/G3PE: dominante industrial.

### Isolation amplifier

Amp com barreira galvânica (capacitiva, magnética ou óptica). Shunt current sensing em motor drive. AMC1200, ACPL-C79A.

## Relés eletromecânicos

Obsoletos em baixa potência, insubstituíveis em alta.

- **Bobina**: 5/12/24 VDC ou AC; corrente de 20-200 mA.
- **Contatos**: SPST, SPDT, DPDT. Rating AC e DC (muito diferentes; AC zero-cross ajuda apagar arco).
- **Tipos**: signal (baixa corrente, áudio), power (1-40 A), automotive (Mini/Micro ISO), DIN rail, reed (glass sealed, rápido, baixa corrente), latching (mantém estado sem alimentar bobina), solid-state (ver SSR).
- **Vida**: mecânica 10⁶-10⁸, elétrica função de carga.

Drive: MOSFET/BJT + diodo flyback. Relé indutivo mata transistor sem clamp.

## Fusíveis

### Tipos

- **Fast-blow (F)**: abre rápido. Eletrônica sensível.
- **Slow-blow (T, time-delay)**: tolera inrush. Motor, transformador.
- **Very fast (FF)**: semiconductor protection.
- **Cartucho de vidro** (5×20, 6.3×32): legado, visível.
- **Blade automotivo** (ATO, ATC, mini): veículos.
- **SMD** (0603, 1206, 2410): pick&place friendly; open = troca de board.
- **Resettable PTC (Polyswitch)**: PTC thermistor; re-arma ao resfriar. USB overcurrent, battery protection.
- **eFuse IC** (TPS2595, SLG59M16xV, LM73100): MOSFET + monitor + clamp + soft-start. Programável, reset via I²C/GPIO. Substitui fuse físico em sistemas modernos.

### Especificação

- **Rated current (In)**: contínuo sem abrir. Derating com temperatura.
- **Breaking capacity (Ir)**: max current que pode interromper sem arco sustentado (AIC). Crítico em AC.
- **I²t**: energia que queima. Para proteger semicondutor: I²t fusível < I²t device.
- **Voltage rating**.

### eFuse vs thermal fuse

Para proteção térmica em fim de carcaça (radiador elétrico, bateria): **thermal cutout** (Klixon) ou thermal fuse (dispara por temperatura). Não é I²t.

## Conectores

Subestimados; primeiro falha em projeto real.

### Parâmetros

- **Contact resistance (mΩ)**: perdas, queda.
- **Current rating**: térmico.
- **Voltage rating**: separação.
- **Mating cycles (1000-100000+)**: vida.
- **Contact force**: preenche microgaps.
- **Plating**: ouro (low-cycle, low-resistance, audio, signal), estanho (commodity), prata (audio, high-current).
- **Retention**: latch, locking, screw.

### Famílias

- **Header pins 2.54 mm (pitch 0.1")**: Dupont. Protótipo.
- **JST (PH 2 mm, XH 2.5 mm, SH 1 mm, GH 1.25 mm)**: battery, sensor.
- **Molex Mini-Fit Jr / Micro-Fit**: power.
- **Phoenix MSTB / COMBICON**: terminal block industrial.
- **D-Sub** (DB9, DB25): legado mas onipresente industrial.
- **USB (A/B/C, mini, micro)**: USB-C é o futuro (PD 240W, SS 40 Gbps).
- **RJ45**: Ethernet; CAT5e-8 classes.
- **M8/M12 circular**: industrial IP67.
- **SFP/QSFP**: data center.
- **PCIe / M.2 / SATA**: internal computer.
- **SMA/SMB/BNC/N-type**: RF.
- **Anderson PowerPole**: high-current field.
- **XT30/60/90, EC3/5, Deans**: RC/drone.

### Armadilhas

- **Current derating por pino**: conector 10A não significa cada pino 10A — datasheet tem tabela.
- **Tin-lead corrosion (fretting)**: oxidação + microvibração = contact resistance cresce. Gold plating mitiga.
- **USB-C sem VID/DAC**: cabos ruins não passam 3A. CC chip obrigatório.

## Chaves e botões

### Tipos

- **Toggle (SPST, SPDT, DPDT)**: mecânica robusta.
- **Rocker, slide, rotary**: variações.
- **Tactile pushbutton**: 6×6 mm PCB, vida 10⁵-10⁷ ciclos.
- **Membrane**: painel flat, vida alta, sensação pobre.
- **Capacitive touch**: sem mecânica; IC dedicado (CAP1xxx, AT42QT).
- **Rotary encoder**: ω com quadratura. Click opcional.
- **Hall-effect**: magnético, sem contato. Vida infinita.

### Debounce

Contato mecânico oscila 5-20 ms quando fecha. Sem debounce: MCU registra múltiplas pressões.

- **Hardware**: RC + Schmitt trigger.
- **Software**: amostragem com filtro (ler a cada 10 ms, comparar n consecutivas).
- **Dedicated IC**: MAX6816, MC14490.

### DIP switches

Configuração persistente em produção. Substituíveis por EEPROM na era moderna, mas ainda usados em legado.

## Displays

- **LED 7-segment**: common anode/cathode. Multiplex dinâmico via driver (74HC595, MAX7219).
- **LED matrix** (8×8, WS2812 grids).
- **LCD character** (HD44780, 16×2, 20×4): legado robusto.
- **LCD graphic mono** (ST7565, SSD1306 OLED): 128×64 barato; I²C/SPI.
- **TFT colorido** (ILI9341, ST7789): 240×320-480×320 via SPI.
- **Capacitive touch screen**: overlay + controller (FT6x36, GT911).
- **E-paper / e-ink**: bistável, ultra-low-power, slow refresh. Label, tag, Kindle, sinalização.
- **VFD (vacuum fluorescent)**: legado audio equipamento.
- **Seven-segment OLED, micro-OLED**: wearable.
- **Flexible PCB + microLED** (emergente): high-end consumer.

## Transducers

- **Speaker dinâmico**: bobina + ímã. Impedância (4-32 Ω), potência, sensibilidade (dB/W/m).
- **Piezo buzzer**: barato, limitado a tons.
- **Piezo transducer + driver (2-30 V)**: beeper passivo.
- **Microfone MEMS** (I²S digital, analog): SPH0645, ICS-43434.
- **Solenoide**: atuador linear; duty cycle importante.
- **Motor CC pequeno / servo**: ver `MOTORS.md`.
- **Vibration motor (ERM, LRA)**: feedback háptico.
- **Peltier (TEC)**: resfriamento/aquecimento sólido, baixa eficiência.

## Armadilhas gerais e princípios

1. **Datasheet é fonte única de verdade**. Application notes complementam; qualquer outra fonte (incluindo este arquivo) é sugestão.
2. **Typical ≠ guaranteed**. Specs "typical" não saem de QA; min/max sim.
3. **Derate**. 50% de tensão, 80% de corrente, 10°C abaixo de max junction temp é base de produto confiável.
4. **Parasíticos dominam em alta freq**. Em RF e SMPS, layout é metade do projeto.
5. **Lote-a-lote varia**. Prototypes lindos passam; produção expõe hFE min, Vth min, offset max.
6. **Thermal é o killer invisível**. Arrhenius: vida cai 2×/10°C em eletrólito, solder, semicondutor.
7. **Second-source**. Depender de fabricante único custa no longo prazo. Especifique com alternativas.
8. **EOL (End of Life)** é real. Componente em stock hoje pode ser obsoleto em 2 anos.
9. **Package termal** sem heatsink/copper pour = derating brutal.
10. **Simular antes de prototipar**. SPICE economiza iterações.

## Ferramentas

- **Octopart, Findchips, Digi-Key Scheme-it**: busca de parte, preço, stock.
- **SnapEDA, Ultra Librarian, Component Search Engine**: símbolos/footprints.
- **Murata SimSurfing, TDK SEAT, Samsung mSEED**: simulação de MLCC DC bias / impedance.
- **Kemet K-SIM, Panasonic DesignSite**: idem para eletrolítico/polymer.
- **Vishay VIPer Design Suite, ST eDesign**: LED, power.
- **TI Power Stage Designer, WEBENCH**: SMPS topology.
- **LTspice, QSPICE (Marsh), ngspice, KiCad IC Spice**: simulação SPICE.
- **FreeCAD / KiCad / Altium**: layout.

## Recursos

- **"The Art of Electronics"** — Horowitz & Hill. Bíblia. 3ª edição (2015).
- **"Practical Electronics for Inventors"** — Scherz & Monk.
- **"Electronic Principles"** — Albert Malvino. Base universitária.
- **"Switching Power Supply Design"** — Pressman, Billings, Morey.
- **"High-Speed Digital Design"** — Johnson & Graham. Signal integrity.
- **TI Precision Labs, TI e²e, Analog Devices EngineerZone, ST Community**: fóruns com engenheiros reais.
- **Application Notes**: TI (SLVA, SLUA), ADI (AN), Linear Tech (LT Journal), Maxim, Infineon. Leia pelo menos umas dezenas.
- **EEVblog, Phil's Lab, Altium Academy** (YouTube): aprendizado prático.
- **Components101, Circuit Basics, All About Circuits**: web refs.
- **datasheet.live, alldatasheet**: índice rápido.
