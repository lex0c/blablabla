# Energia — Eletricidade, Geração e Baterias

Eletricidade é o **vetor de energia mais versátil** já usado por humanos: gerada em escala industrial, transmitida por milhares de km, convertida eficientemente em luz, calor, movimento, computação. Cada passo dessa cadeia — geração, transmissão, distribuição, armazenamento, uso — tem sua engenharia própria e desafios atuais. Este arquivo é o mapa macro: enquanto `CIRCUIT.md` trata de eletrônica de baixa potência, aqui está o mundo de megawatts, alta tensão, grid, baterias de escala.

## Fundamentos

### Grandezas principais

| Grandeza | Símbolo | Unidade | Intuição |
|---|---|---|---|
| Carga | Q | coulomb (C) | "quantidade" de eletricidade |
| Corrente | I | ampère (A = C/s) | fluxo por tempo |
| Tensão | V | volt (V = J/C) | energia por unidade de carga |
| Potência | P | watt (W = V·A) | taxa instantânea de energia |
| Energia | E | joule ou kWh | acumulada; 1 kWh = 3.6 MJ |
| Frequência | f | hertz (Hz) | 60 Hz Américas, 50 Hz resto do mundo |

### AC vs DC

Debate histórico ("War of Currents", 1880-90s) — Edison (DC) vs Tesla/Westinghouse (AC). AC venceu para transmissão:

- **Transformadores** só operam em AC — permitem elevar tensão para transmissão e baixar para consumo.
- Perdas `P = I²R` — transmitir em alta tensão reduz corrente logarítmicamente perda.
- Motores AC eram simples e robustos.

DC voltou com força em **HVDC** (transmissão em corrente contínua em altíssima tensão, centenas de kV a >1 MV), porque:

- Menores perdas em distâncias muito longas (>500-1000 km).
- Conexão assíncrona entre redes de frequências diferentes.
- Submarino e subterrâneo: capacitância de cabo AC mata eficiência.

Projetos: Xiangjiaba-Shanghai (China, 6.4 GW, 2071 km), Changji-Guquan (±1100 kV, 3.28 GW, 3324 km), Ultranet na Alemanha, HVDC entre Reino Unido-França.

### Sistema Trifásico

AC comercial é **trifásico**: três fases defasadas 120°. Vantagens:

- Potência instantânea constante (não pulsante como monofásico).
- Menos cobre por watt transmitido.
- Motores trifásicos são auto-partida sem capacitor.

Tensões típicas no Brasil: 127/220 V (residencial), 380/220 V (comercial), 13.8 kV (distribuição urbana), 69-138-230-500-750 kV (subtransmissão/transmissão), ±600/±800 kV DC (linhas de Itaipu, Madeira).

## Geração de Energia

### Térmica a Combustão

Queima → calor → vapor → turbina → gerador.

- **Carvão**: barato historicamente, alta emissão de CO₂/SOx/NOx/mercúrio/cinza. Declinando na UE/EUA, persistente China/Índia.
- **Gás natural**: ~metade da emissão do carvão por MWh; ciclo combinado (CCGT) atinge 60%+ eficiência.
- **Óleo combustível**: caro, em declínio; nicho insular.
- **Biomassa**: carbono neutro em ciclo longo; competição com alimento e uso da terra.

### Nuclear

Fissão de U-235 ou Pu-239 em reator → calor → vapor → turbina.

- **PWR (pressurized water)**: dominante global.
- **BWR (boiling water)**.
- **CANDU**: água pesada, urânio natural.
- **VVER**: russo, derivado de PWR.
- **SMRs (Small Modular Reactors)**: 50-300 MW, fabricação modular, deploying 2025-2030.
- **Geração IV**: sal fundido, alta temperatura, rápido — pesquisa.
- **Fusão**: ITER (2025 em comissionamento), stellarators, tokamaks comerciais (Commonwealth Fusion, TAE). Sem comercialização ainda; milestone "plasma fazendo mais energia do que consome" atingido em ignição inercial (NIF 2022).

Considerações: custo alto de capex + regulatório; operação extremamente barata; zero emissão de CO₂ operacional; resíduo radioativo por séculos; risco baixo mas catastrófico (Chernobyl 1986, Fukushima 2011).

### Hidrelétrica

Água descendo → turbina Pelton/Francis/Kaplan → gerador. Maior fonte renovável histórica (~15% geração global).

- **Grande porte**: Três Gargantas (22.5 GW), Itaipu (14 GW), Belo Monte (11.2 GW).
- **PCH** (pequenas centrais): menor impacto, menor capacidade.
- **Reversível / pumped storage**: bomba água para reservatório superior em excesso, gera em pico. Maior forma de armazenamento em escala mundial hoje.
- **Run-of-the-river**: sem grande reservatório; capacidade menor, dependente de vazão.

Impactos: deslocamento humano/ambiental (barragens grandes), emissão de metano em reservatórios tropicais (decomposição biomassa submersa), sedimentação.

### Eólica

- **Onshore**: torres 80-150 m, pás 60-100 m. Turbina ~2-6 MW típica, 8-15 MW em top-end.
- **Offshore**: fixas (monopile, jacket) em águas rasas até ~60 m; flutuante em profundas (Hywind, etc.). Turbinas 12-20 MW.
- **Capacity factor**: onshore ~30-40%, offshore 45-60%. Fator de capacidade = média / pico.

Desafios: variabilidade, impacto em aves/morcegos, aceitação social (visual, ruído, infrassom), fim-de-vida das pás (reciclagem difícil de compósitos).

### Solar Fotovoltaica (PV)

Efeito fotovoltaico em junção P-N. Silício cristalino (mono ou poli) domina ~95% mercado.

- **Custos em queda espetacular**: 2010 ~$2/Wp → 2024 ~$0.15/Wp (módulo). **Learning curve** histórica ~20% por dobra de capacidade instalada.
- **Eficiência**: módulos comerciais 20-23%; células laboratório >26% (mono); tandem perovskita-silício >33% em lab.
- **Thin-film**: CdTe (First Solar), CIGS — nicho.
- **Bifacial**: captura reflexo do solo, +5-15% geração.
- **Tracking**: seguidor solar +20-30% vs fixo.

Desafios: intermitência diurna + sazonal, ocupação de terra, reciclagem (vidro + alumínio + silício + pequena fração tóxica).

### Solar Térmica Concentrada (CSP)

Espelhos concentram luz → fluido quente → turbina. Calha parabólica, torre central, Fresnel linear, prato Stirling.

- **Com armazenamento de sal fundido**: gera à noite. Crescent Dunes, Noor Ouarzazate, Gemasolar.
- Menos competitiva que PV + bateria em 2024+ para a maioria dos usos.

### Geotérmica

Calor do subsolo. Tradicional em zonas vulcânicas (Islândia, Filipinas, Nova Zelândia, Quênia). **EGS (Enhanced Geothermal)**: fraturar rocha profunda e circular fluido — viabiliza qualquer geografia; early deployment 2024+.

Baseload renovável (constante).

### Maremotriz e Ondas

Nicho técnico; poucos projetos comerciais (La Rance França 1966, Sihwa Coreia).

### Hidrogênio

Não é fonte primária — é **vetor**. Produzido por:

- **Eletrólise** da água (hidrogênio verde, se eletricidade renovável).
- **Reforma a vapor** de metano (cinza/azul, se com CCS).
- Depois: queima em turbina, célula de combustível, uso industrial (aço, fertilizante, refinaria).

Problemas: eficiência round-trip baixa (~30-40% eletricidade → H₂ → eletricidade); armazenamento e transporte custosos (líquido criogênico -253°C ou alta pressão); infraestrutura zero.

Nichos onde faz sentido: siderurgia (substituir carvão em alto-forno), fertilizantes, aviação (SAF derivado), shipping.

## Transmissão e Distribuição

### Cadeia

```
Geração (kV) → Elevadora → Transmissão (EHV, 138-800 kV)
                              ↓
                       Subestação AT/MT
                              ↓
                   Distribuição MT (13.8 kV)
                              ↓
                     Trafo pole-top
                              ↓
                    BT (127/220 V) → consumidor
```

### Transformadores

Princípio de Faraday. `V1/V2 = N1/N2`. Núcleo de aço laminado reduz perdas por correntes de Foucault. Óleo isola + refrigera em trafos grandes.

- **Eficiência** >99% em trafos de potência.
- **Perdas a vazio (núcleo)** + **perdas em carga (cobre)**.
- **Saturação magnética** limita operação.

### Linhas de Transmissão

- **Cabos de alumínio** com alma de aço (ACSR) mais econômicos que cobre em distância.
- **Isoladores cerâmicos ou polímero** suportam tensão para a torre.
- **Para-raios** (surge arresters) absorvem descargas atmosféricas.
- **Efeito corona** em alta tensão: ionização do ar → perda + rádio interferência.
- **Efeito pelicular (skin)**: em AC, corrente concentra-se na superfície; piora em alta freq.
- **Compensação reativa**: capacitores + reatores ajustam fator de potência.

### FACTS e HVDC

- **FACTS (Flexible AC Transmission Systems)**: eletrônica de potência (SVC, STATCOM, UPFC) controla fluxo AC dinamicamente.
- **HVDC** (já citado): conversores rectificador ↔ inversor em pontas.
- **VSC-HVDC (Voltage Source Converter)**: moderno; black start, controle de potência ativa/reativa independente.

### Smart Grid

Sensoriamento + comunicação + controle automatizado no grid:

- **PMU (Phasor Measurement Units)**: medem fase sincronizadas via GPS; visibilidade em tempo real.
- **AMI (Advanced Metering Infrastructure)**: medidores inteligentes bidirecionais.
- **SCADA + DMS (Distribution Management System)**: centraliza operação.
- **Autohealing**: religadores + automação detectam falta e isolam setor, realimentam sãos.

Cybersegurança é preocupação central — ver `IOT_FIRMWARE_SECURITY.md` (cross-ref para ataques em ICS).

### Grid Moderno e Renováveis

Integrar solar + eólica (intermitentes) em grid desenhado para geradores síncronos (térmica + hidro) é desafio técnico:

- **Inércia sintética**: inversores simulam inércia que geradores síncronos têm naturalmente.
- **Grid-forming inverters**: além de seguir grid, podem estabilizá-lo.
- **Reserva girante** substituída por bateria + controle rápido.
- **Curtailment**: cortar geração renovável em excesso (sinal de mercado a faltar armazenamento/transmissão).

## Armazenamento de Energia

Chave para viabilizar 100% renovável.

### Bombeada (PHS)

95% da capacidade global. Maduro, barato por MWh armazenado, dependente de geografia.

### Baterias Eletroquímicas

Conceito: célula com ânodo (oxidação), cátodo (redução), eletrólito (íon transport), separador. Íons migram no eletrólito, elétrons pelo circuito externo.

#### Chumbo-ácido

- 1859 (Planté). **Maduro, barato, reciclável, pesado**.
- Aplicações: automotivo (partida SLI), UPS, off-grid barato.
- Vida: ~500-1000 ciclos; deep cycle até 2000.
- Eficiência round-trip ~75-85%.
- Desvantagens: densidade energética baixa (~30-50 Wh/kg), ácido/chumbo (tóxicos se não reciclados).

#### Níquel-Cádmio (NiCd)

Histórico em aplicação profissional. Densidade ~40-60 Wh/kg. Efeito memória. Cádmio tóxico → banido em EU consumer.

#### Níquel-Hidreto (NiMH)

Sucessor do NiCd. ~60-80 Wh/kg. Dominou em híbridos (Toyota Prius até 2010s) e eletrônicos pré-Li.

#### Lítio-íon (Li-ion)

Dominante moderno. Densidade ~150-270 Wh/kg (célula). Químicas:

- **LCO (LiCoO₂)**: celulares; alta densidade, menor segurança.
- **NMC (Li-Ni-Mn-Co)**: EVs premium; balance densidade-custo-vida. Variantes 622, 811 (mais níquel, menos cobalto).
- **NCA (Li-Ni-Co-Al)**: Tesla histórico. Alta densidade.
- **LFP (LiFePO₄)**: mais seguro, ciclos mais longos (>3000), menor densidade (~150 Wh/kg), sem cobalto. Dominando EV entry-level e grid storage. BYD, CATL, Tesla migraram parcialmente.
- **LTO (Lithium Titanate)**: ciclos extremos (>10000), baixa densidade, rápida carga.

**BMS (Battery Management System)** crítico: balanceamento entre células, proteção térmica/tensão/corrente, SoC/SoH estimation.

**Thermal runaway**: célula Li-ion ao exceder temperatura crítica (~150-200°C) inicia reação autossustentada, chama + gás tóxico. Pack design (espaçamento, barreiras, refrigeração) mitiga; LFP é muito mais resistente que NMC/NCA.

#### Sódio-íon

Emergente 2023+. CATL, BYD, Hina. Densidade menor (~100-160 Wh/kg) mas sódio abundante e barato, sem lítio/cobalto, boa performance em baixa temperatura. Aplicações: grid storage, EV entry, substrato industrial.

#### Estado Sólido

Eletrólito sólido em vez de líquido → maior densidade, menor risco de vazamento/fogo, potencial vida mais longa. QuantumScape, Solid Power, SES AI, Toyota anunciam. Ainda não em produção consumer em larga escala (2025-2030 projetado).

#### Fluxo (Flow)

Eletrólito em tanques externos circula através de célula. **Vanádio redox** é mais maduro. Escala: tamanho da bateria é **tamanho do tanque** (desacoplado de potência). Vida extrema (>20 anos, >20k ciclos). Densidade baixa (~15-50 Wh/kg) → grid storage, não mobile.

Zinco-bromo, ferro-cromo são outras químicas emergentes.

#### Metal-ar

Lítio-ar (teórica, densidade extrema), zinco-ar (aparelhos auditivos, aplicações grid). Desafios: ciclagem.

### Capacitores vs Baterias

| | Capacitor | Supercap | Bateria |
|---|---|---|---|
| Densidade energia | baixa | média | alta |
| Densidade potência | altíssima | alta | média |
| Ciclos | milhões | ~100k-1M | 500-10k |
| Tempo de carga | µs | s | h |
| Aplicação | filtragem, pulsos | regeneração em EV, UPS bridge | bulk energia |

Híbridos (bateria + supercap) otimizam ambos.

### Térmico

- **Sais fundidos** em CSP: já citado.
- **Gelo** (ice storage) em climatização comercial: congelar à noite (tarifa barata), resfriar de dia.
- **Cerâmica refratária**: projetos tipo Rondo, Antora — aquecer bloco a >1500°C, extrair como calor industrial.

### Mecânico

- **Volantes de inércia (flywheel)**: massa rotativa; segundos a minutos; UPS + power quality; Beacon Power.
- **Ar comprimido (CAES)**: em cavernas de sal. Huntorf 1978, McIntosh 2019.
- **Gravidade**: elevar massa (Energy Vault torres de concreto, Highview líquido de ar). Geopolítica — alguns são vaporware.

### Hidrogênio como armazenamento

Já citado: eletrólise → tanque → célula combustível. Round-trip baixo, infra cara.

## Veículos Elétricos

Convergência elétrica + automotiva. Ver também `ROBOTICS.md` (automotivo é ramo), `CAR_HACKING.md` (segurança).

### Arquitetura típica

- **Pack de bateria**: 50-100+ kWh em sedan; 150-200 kWh em pickup/SUV premium.
- **Inversor** (SiC/GaN em modernos — eficiência e tamanho).
- **Motor síncrono de imã permanente (PMSM)** — dominante. Algum Tesla usa indução em eixo secundário.
- **Redução** fixa (raramente multi-speed).
- **DC-DC** para 12V acessórios.
- **Carregador AC** onboard (7-22 kW típico).

### Carregamento

- **AC onboard** (L1 120V 1.4 kW, L2 240V 7-11 kW, 3ph 11-22 kW).
- **DC rápido**: conecta direto ao pack, 50 kW (antigo CHAdeMO) a 350+ kW (CCS2 moderno, Tesla V4 Supercharger).
- **Padrões**: CCS1 (EUA pré-2024), CCS2 (UE), CHAdeMO (Japão, declinante), GB/T (China), **NACS** (Tesla, adotado como padrão EUA 2024+).
- **Tempo de carga**: 10-80% em 15-20 min em DC rápido; noite inteira em L2.

### V2G e V2H

Veículo → grid ou casa. Usar bateria do carro como backup/flex. Protocolos (ISO 15118). Frota comercial > consumer.

### Sustentabilidade

- CO₂ de fabricação é maior (bateria custa); quebra-even em ~20-40k km contra gasolina dependendo do grid.
- Descarte/reciclagem de bateria: Li-Cycle, Redwood Materials, Ecobat. Cadeia em desenvolvimento.
- Reuse antes de reciclagem: baterias de EV retiradas ainda têm 70-80% capacidade → grid storage de segunda vida.

## Eletrônica de Potência

Dispositivos modernos controlam fluxo de centenas de kW até GW:

### Dispositivos

- **Diodos**: retificação. Em alta potência: diodos de recuperação rápida.
- **Tiristores (SCR)**: ligam via gate, desligam via ausência de corrente. GTO, IGCT são variantes. HVDC clássico usa.
- **IGBT (Insulated Gate Bipolar Transistor)**: dominante 600V-6.5 kV.
- **MOSFET**: até ~1 kV; baixíssima perda.
- **SiC (Carboneto de Silício)**: wide-bandgap; maior tensão + frequência + temperatura que Si. Revolucionando EV, inversores fotovoltaicos, HVDC.
- **GaN (Nitreto de Gálio)**: similar a SiC em benefícios; domina < 1 kV high-frequency (carregadores laptop, data center PSUs).

### Topologias

- **Retificador**: AC → DC. Pontes Graetz, retificadores controlados.
- **Inversor**: DC → AC. Dominante em solar, EV, UPS. VSI (voltage source) vs CSI (current source).
- **Conversor DC-DC**: buck, boost, buck-boost. Ver `CIRCUIT.md`.
- **Active rectifiers / PFC**: corrigem fator de potência na entrada.

### Modulação

- **PWM (Pulse Width Modulation)**: gera senoide "equivalente" por switching rápido.
- **SPWM, SVPWM**: variantes.
- **Hysteresis control**.
- **Modulação multinível** em alta tensão (MMC — Modular Multilevel Converter em HVDC).

## Motores Elétricos

Consomem ~45% eletricidade global. Tipos:

- **Síncrono de imã permanente (PMSM)**: EV, robótica, alto-desempenho.
- **Indução (IM)**: industrial dominante. Robusto, barato.
- **Brushless DC (BLDC)**: drones, pequenos.
- **Universal**: ferramentas manuais; roda em AC ou DC.
- **Stepper**: posicionamento. Ver `ROBOTICS.md`.
- **Relutância síncrona (SynRM)**: emergente, sem imã.

### VFDs (Variable Frequency Drives)

Controlam velocidade via inversor + PWM. Economia de energia brutal em bombas, ventiladores, compressores (cargas com curva cúbica de potência por velocidade — reduzir 20% de velocidade = economizar ~50% energia).

## Proteção Elétrica

### Disjuntores e Fusíveis

- **Fusível**: elemento que derrete em sobrecorrente. Uso único. Barato.
- **Disjuntor termomagnético (MCB)**: bimetal (sobrecarga térmica lenta) + bobina (curto-circuito magnético rápido). Rearmável.
- **DR (diferencial residual / GFCI)**: detecta fuga à terra (diferença entre fase e neutro) — protege contra choque. Obrigatório em banheiros, cozinhas, áreas molhadas.
- **DPS (Surge Protective Device / para-raios)**: absorve surtos de tensão.

Coordenação (seletividade): disjuntor de ramal abre antes do geral; fusível fornecedor abre antes do transformador.

### Aterramento

Sistema de referência comum para tensões e descarga segura de correntes de falta.

- **TN-S, TN-C, TN-C-S, TT, IT**: sistemas definidos pela IEC 60364. Brasil usa majoritariamente TN-S ou TN-C-S.
- **Malha de aterramento** em subestações: reduz gradiente de potencial em falta.
- **SPDA (Sistema de Proteção contra Descargas Atmosféricas)**: captor (Franklin, Faraday, radiativo), descida, aterramento. NBR 5419 no Brasil.

### Isolação e Classes

Classes de proteção de equipamento:
- **Classe I**: carcaça metálica + terra de proteção.
- **Classe II**: isolação dupla; sem terra (símbolo de quadrado dentro de quadrado).
- **Classe III**: SELV (Safety Extra-Low Voltage, ≤50V AC).
- **IP rating** (Ingress Protection): IP67, IP68... primeiro dígito = sólidos, segundo = líquidos.

## Segurança Humana

Eletricidade mata em condições triviais.

- **Corrente ~10 mA AC** através do corpo causa perda de controle muscular.
- **~50-100 mA** pode fibrilar.
- **~1 A** quase sempre fatal (variação por caminho, duração, frequência).
- DC é ligeiramente menos perigoso em freqüência zero, mas tensões iguais ainda matam.
- Mão-para-mão cruza o tórax — pior.
- Resistência da pele seca: ~10-100 kΩ; molhada: ~1-5 kΩ; perfurada: muito menor.

### Normas no Brasil

- **NR-10**: segurança em instalações e serviços em eletricidade. Treinamento obrigatório, medidas de controle.
- **NR-35**: trabalho em altura (frequentemente combinado).
- **NBR 5410**: instalações de baixa tensão.
- **NBR 14039**: média tensão.
- **NBR 5419**: proteção contra descargas atmosféricas.
- **NBR 15749**, **NBR 13534** (áreas hospitalares).

EUA: **NEC (NFPA 70)**, **NFPA 70E** (safety workplace).

### LOTO (Lockout/Tagout)

Isolar energia antes de manutenção. Cadeado + etiqueta identificando responsável. Impede reenergização acidental. Em ambiente industrial, processo disciplinado salva vidas regularmente.

### EPI Elétrico

- **Luvas isolantes** classe 0 (500V) a 4 (36 kV).
- **Botas dielétricas, capacetes, protetores faciais, mantas isolantes**.
- **Detector de tensão** (fasímetro) antes de tocar.
- **Arc flash suit** em trabalhos próximos a falta potencial (ATPV em cal/cm²).

### Primeiros Socorros

- **Desligar circuito primeiro**; se impossível, afastar vítima com material não-condutor (madeira seca, borracha).
- **Parada cardiorrespiratória** exige RCP imediato. DEA onde disponível.
- Queimaduras elétricas frequentemente são piores internamente que aparentam superficialmente.
- **Vítima pode estar "colada"**: contração muscular tetânica impede soltar — falso contato do observador pode ser fatal.
- **Fibrilação tardia**: paciente consciente pós-choque pode entrar em FV horas depois. Hospital obrigatório mesmo se "parece bem".

### Arc flash

Curto em equipamento energizado libera energia em arco elétrico: plasma a 19.000 °C (quatro vezes superfície do sol), onda de choque, estilhaços, luz UV intensa. Causa a maior parte dos óbitos e queimaduras graves em trabalho elétrico industrial — não choque direto.

- **Incident energy**: cal/cm² entregues a distância do arco. Calculado por **IEEE 1584** ou NFPA 70E.
- **Boundary distances**:
  - **Flash protection boundary**: onde energia atinge 1.2 cal/cm² (queimadura de 2º grau).
  - **Restricted approach** e **prohibited approach**: zonas mais próximas; só qualificado com EPI adequado.
- **PPE Category (NFPA 70E)**: 1 (4 cal/cm²) até 4 (40 cal/cm²). ATPV do traje deve cobrir.
- **Arc rated clothing**: tecido não derrete nem propaga chama (Nomex, CarbonX, aramidas). Algodão comum pega fogo e gruda na pele.
- **Mitigação**: trabalhar **desenergizado** (sempre preferível), redução de corrente de curto via impedância, manutenção de relés (tempo de atuação), barriers, arc-resistant switchgear (IEC 62271-200).

### Capacitores e energia armazenada

Equipamento "desligado" pode estar letal. Verifique **sempre** antes de tocar.

- **Inversores de frequência, fontes chaveadas, drives EV**: barramento DC com capacitores de alta tensão. Sangram em minutos via resistor de descarga, mas **assuma cheio até medir**.
- **Fornos de microondas**: capacitor de 2 kV, energia suficiente pra fatalidade. Aterrar com hastes antes de abrir.
- **TVs CRT antigas**: tubo carregado ~25 kV via flyback. Descarregar pra chassi via resistor + fio com alicate isolado.
- **Flash de câmera**: capacitor de ~300 V; queima mas raramente mata.
- **UPS, no-breaks**: bancos de bateria + capacitores. LOTO + descarga.
- **Placas PV**: ver seção abaixo — não têm capacitor, mas são fonte.

**Procedimento**: desenergizar → aguardar tempo especificado → testar tensão → aterrar temporariamente → testar de novo → trabalhar.

### DC vs AC — perigos distintos

AC foi "vitorioso" em transmissão; DC tem propriedades únicas que mudam a análise de risco.

- **AC 50/60 Hz**: ressonância com sistema de condução cardíaca → **fibrilação ventricular** em faixa estreita (limiar ~50-100 mA).
- **DC**: limiar de fibrilação é 3-5× maior (~300 mA), mas causa contração muscular sustentada. Menos fatal por fibrilação, mais por queimadura interna e impossibilidade de soltar.
- **AC tem zero-crossing** 100/120× por segundo → arco AC se auto-extingue facilmente em disjuntor.
- **DC não tem zero-crossing** → arco sustentado, se auto-alimenta. **Disjuntores DC** são dramaticamente mais robustos; misturar é receita de falha catastrófica.
- **Alta tensão DC** (fotovoltaico, EV, HVDC): arco pode sustentar em centímetros de distância. Manutenção exige procedimentos próprios.
- **Capacitância dos circuitos DC**: mesmo desligado, tensão persiste (ver capacitores).

### Sistemas fotovoltaicos (FV)

Riscos específicos que não aparecem em instalação AC convencional.

- **Sempre energizado sob luz**: string de 20 módulos × 40 V = 800 V DC open-circuit. **Não há "desligar módulo"** — cobri-los ou trabalhar à noite.
- **Rapid shutdown**: exigência em NEC 690.12 (EUA) e incorporada a muitas normas: botão que leva strings a <30 V em <30s. Crucial pra bombeiro em incêndio de telhado.
- **Arco DC FV**: sustentado, difícil extinguir. Conectores MC4 mal crimpados são causa comum de incêndio.
- **Falha de terra**: corrente de fuga em módulo defeituoso não é detectada como em AC. GFDI (Ground-Fault Detection/Interruption) específico.
- **Potential-induced degradation (PID)**: em módulos com V alto vs terra, íons migram no vidro — perda de potência. Mitigação: aterramento do negativo, módulos PID-free.
- **Quedas de telhado**: a maior causa de morte em instaladores FV não é choque; é NR-35 (altura). Harness + ancoragem + procedimento.
- **Animais e instalação**: cabos DC em telhado atraem roedores; isolação comprometida → arco → incêndio.

### Baterias de lítio — thermal runaway

Densidade de energia alta + eletrólito inflamável + química que se auto-sustenta quando aquecida.

- **Thermal runaway**: aumento de temperatura libera O₂ do cátodo → reage com eletrólito → mais calor → cascata. Pode atingir 800+ °C, incêndio ou explosão.
- **Gatilhos**: sobrecarga, sobrecorrente, curto interno (dendritos), dano mecânico (furo), superaquecimento externo.
- **Química e risco**:
  - **NMC, NCA**: alta densidade, maior risco térmico.
  - **LFP**: muito mais estável termicamente (por isso predomina em grid storage e EV entry).
  - **Lítio-metal, sólida**: P&D; promessas de segurança ainda não validadas.
- **Venting**: célula com BMS bom expele gás inflamável antes de pegar fogo. Ambiente precisa ventilação.
- **Extinção**: água em grande volume resfria, mas célula reignita. **F500, Lith-Ex, Avertex** — agentes especializados. CO₂ e pó químico são inúteis; secam, não resfriam.
- **BMS (Battery Management System)**: vigiar temperatura por célula, balanceamento, corte em falha. BMS ruim = bomba-relógio.
- **Pack EV em acidente**: assumir energizado + potencialmente danificado. Bombeiro moderno tem protocolo (Tesla, Rivian, BYD publicam emergency response guides — EV-FRG).
- **Hobby/DIY**: célula 18650/21700 exposta, em bolso com chaves, pode shortar e explodir. Guarde em caixa não-condutora.

### Eletricidade estática (ESD)

Descarga de corpo humano carregado: tensão 2-15 kV, corrente microampère, energia mínima para humano — mas **destroi eletrônicos** e **ignita vapores**.

- **Hardware**: CMOS modernos vulneráveis desde 500 V (sensível); usar pulseira ESD aterrada, piso/mesa ESD, sapatos condutores, embalagem antiestática (saco preto/prateado).
- **Ambientes ATEX**: descarga em atmosfera com vapor/poeira inflamável é fonte de ignição. Calçado antiestático + piso condutor + controle de umidade obrigatórios.
- **Postos de combustível**: "não entre/saia do carro enquanto abastece" → separação de carga em tecido + bico metálico + combustível = estática suficiente pra arco.

### RF / alta frequência

Não fibrila (frequência muito alta pra excitar tecido cardíaco) mas queima profundamente.

- **RF burns**: aquecimento dielétrico em tecido — cozinha por dentro antes de superfície responder. Sem sensação de choque até dano consumado.
- **Fontes**: transmissores, antenas, aquecimento indutivo industrial, plasma, radar, diatermia médica.
- **Campos próximos**: acopla com estruturas metálicas (joias, piercings, implantes). Marcapassos podem ser interferidos.
- **Exposição**: ICNIRP e FCC definem limites SAR (Specific Absorption Rate, W/kg). Celular OK; dipolo de 1kW ligado pode queimar mão em minutos.
- **Lockout específico**: RF não sente no fasímetro comum. Usar detector de RF próprio.

### Step e touch voltage (subestação, linha caída)

Falta a terra em subestação ou cabo aéreo caído energiza a terra ao redor. Tensão decresce em forma de gradiente radial.

- **Step voltage (tensão de passo)**: diferença entre os dois pés. Pessoa caminhando normalmente leva choque entre pernas; corrente atravessa pelve. Morte por fibrilação ou tetania.
- **Touch voltage (tensão de toque)**: entre mão e pés. Tocar cerca metálica energizada por linha caída.
- **Mitigação**: **malha de aterramento** em subestações equaliza potencial. Em campo com cabo caído: **saltar em pés juntos** (não caminhar), manter mínimo 10 m de distância, não tocar no cabo nem em veículo afetado.
- **Animais de pazinho longo** (bovinos, cavalos) morrem em campo eletrificado por step voltage mesmo sem tocar nada.

### Atmosferas explosivas (ATEX / IECEx / Classe-Divisão)

Gases inflamáveis, vapores, poeira combustível em ambiente industrial (química, farma, moagem, combustíveis, minas).

- **Classificação por zona (IEC/ATEX)**:
  - **Zona 0**: atmosfera explosiva presente continuamente. Gasômetros, vasos fechados.
  - **Zona 1**: provável em operação normal.
  - **Zona 2**: improvável, só em falha.
  - **Zonas 20/21/22**: análogos pra poeira (grãos, carvão, metais).
- **EUA: Class/Division** (NEC 500) — taxonomia paralela, menos usada hoje.
- **Equipamento "Ex"**: à prova de explosão (`Ex d`), segurança intrínseca (`Ex i` — baixa energia, não ignita nem em falha), aumentado (`Ex e`), pressurizado (`Ex p`), não-incendiado (`Ex n`).
- **Ex i** é o único método seguro com circuitos vivos em atmosfera explosiva. Sensores, calibradores portáteis.
- **Erro comum**: usar multímetro comum em zona Ex → fonte de ignição. Use intrinsicamente seguro (Fluke 28 II Ex, Metrix OX 9000 Ex).

### Hazards residenciais e hobby

Onde acidente doméstico acontece:

- **Trabalho "com a chave ligada, rapidinho"**: quase toda fatalidade residencial. Desligue o disjuntor. Não presuma.
- **Testar sem fasímetro**: "se não dói, não tem tensão". Pele seca + dedo rápido pode ocultar 220 V. Use detector de tensão sem contato antes.
- **Aliança, relógio, piercing metálico**: curto em corrente alta = queimadura anelar profunda. Remova antes.
- **Escada de alumínio perto de fios aéreos**: contato direto ou arco por proximidade. Fibra de vidro ou madeira.
- **Ferramenta elétrica em umidade**: sem DR, choque é provável. Use DR portátil entre tomada e ferramenta.
- **Gambiarra com fita isolante**: emenda sem luva termo ou conector adequado = curto a médio prazo, incêndio a longo.
- **Tomadas sobrecarregadas**: filtros/adaptadores triplos em aquecedor ou ar condicionado. Incêndio comum.
- **Fios roídos por pet**: inspecione periferia.
- **Piscina/banheira/aquário**: circuito GFCI obrigatório; pequeno vazamento em fonte de aquário pode matar em banho.
- **Religação após queda de energia**: bombas, aquecedores, máquinas ligam sozinhas. Desligue na chave antes de inspecionar.

### Geradores portáteis e backfeed

Gerador ligado à casa sem chave de transferência: alimenta **também a rede pública**, matando lineman trabalhando a km de distância.

- **Sempre** use chave de transferência (manual ou automática) ou interlock no quadro. Isolamento completo entre gerador e rede.
- **Ventilação**: geradores a combustão emitem CO. Morte por CO em garagem fechada é rotineira pós-furacão. Use fora, longe de portas/janelas.
- **Aterramento**: conforme fabricante. Muitos portáteis modernos são "separately derived system" e não precisam aterramento externo.
- **Backup residencial sério** = gerador fixo + ATS + interlock certificado.

### Trabalho em altura + elétrica

Quase todo acidente grave em subestação elevada, poste, telhado FV é combinação.

- **NR-35** (altura) e **NR-10** (elétrica) simultaneamente.
- **Ancoragem dupla** (100% tie-off) — um mosquetão sempre preso.
- **Queda após choque**: músculo contrai, pessoa cai mesmo se suspensa. Trauma de suspensão (suspension trauma) exige resgate em <15 min.
- **Tempo máximo em harness pendurado**: 10-15 minutos antes de danos por isquemia. Equipe de resgate pré-posicionada.

## Medição Elétrica

- **Multímetro**: V, I, R, continuidade. True RMS importa em AC não-senoidal.
- **Alicate amperímetro (clamp)**: corrente sem romper circuito.
- **Wattímetro, medidor de energia**: consumo.
- **Analisador de qualidade de energia**: harmônicas, flicker, afundamentos, fator de potência.
- **Terrômetro**: resistência de aterramento (método queda de potencial, Wenner).
- **Megômetro**: resistência de isolação em alta tensão.
- **Osciloscópio de alta tensão** + pontas diferenciais em pesquisa de conversores.

## Eficiência Energética

### Em edificações

- **Envelope**: isolamento térmico, janelas de baixo fator solar.
- **Iluminação LED**: substitui incandescente (5-10× mais eficiente) e fluorescente (+20-40%).
- **HVAC eficiente**: bombas de calor (COP 3-5 contra 1 de resistência); variable speed; recuperação de ar.
- **Selos**: Procel (BR), Energy Star (EUA), Energy Label (UE), Minergie (CH).

### Industrial

- **VFDs em motores** (já citado).
- **Compressed air**: vazamentos consomem ~20-30% do compressor em fábricas típicas.
- **Cogeração (CHP)**: calor residual de gerador produz vapor/aquecimento → 80%+ eficiência sobre combustível.
- **Trigeração (CCHP)**: + refrigeração via absorção.

### Data Centers

- **PUE (Power Usage Effectiveness)**: (total facility) / (IT). 1.0 perfeito; hyperscalers em 1.1-1.2; médias empresariais 1.5-2.0.
- **Water cooling direto ao chip, liquid cooling** em racks GPU modernos. Ver `GPU.md`.
- **Renovável matching** + deslocamento temporal de carga → "carbon-free energy 24/7".

## Geopolítica e Transição

### Transição Energética

Descarbonização do sistema elétrico é meta de política global (Paris 2015, atualizações COP). Caminho:

1. **Eletrificar** o que queima combustível diretamente (transporte, aquecimento, parte da indústria).
2. **Descarbonizar** a geração elétrica (renováveis + nuclear + capture).
3. **Armazenamento + transmissão** para integrar intermitência.
4. **Eficiência** reduz demanda total.

### Atores e cadeias

- **Solar PV**: China domina ~80% da cadeia (polisilício, wafer, célula, módulo).
- **Baterias**: China domina química, refino de minerais, produção de célula.
- **Eólica**: Vestas, Siemens Gamesa, GE, Goldwind, Envision. Mais distribuída.
- **Minerais críticos**: lítio (Austrália/Chile/China), cobalto (RDC), níquel (Indonésia), cobre (Chile/Peru/RDC), terras raras (China).
- **Nuclear**: Rosatom (Rússia), Westinghouse/GE (EUA), EDF/Areva (França), KHNP (Coreia), CNNC (China).

### Riscos estratégicos

- **Concentração em cadeias**: Taiwan (semicondutor para eletrônica de potência), China (solar/bateria), Rússia (urânio enriquecido) — dependência assimétrica.
- **Minerais críticos**: geopolítica da mineração e refino.
- **Grid resilience**: interconexões são alvo (físico e cyber); ataque na Ucrânia 2015-ongoing é caso real (ver `NETWORK_ATTACKS.md` para vetor ICS/SCADA).

## Off-grid e Microgrids

Sistema isolado ou semi-isolado:

- **Rural, ilhas, militar, industrial crítico**.
- **Híbrido solar + bateria + gerador diesel** é arquitetura comum.
- **Microgrid** pode conectar/desconectar do grid principal ("ilhamento").
- **Navios, aviões, espaçonaves**: microgrids especiais.

## Padrões e Instituições

- **IEC (International Electrotechnical Commission)**: padronização internacional.
- **IEEE**: papers, padrões (802 para LAN, 1547 para DER, etc.).
- **ABNT NBR**: Brasil.
- **ANSI / NEC** (EUA).
- **CENELEC** (Europa).
- **Operadores**: ENTSO-E (EU), NERC (EUA), ONS (Brasil).
- **Reguladores**: ANEEL (BR), FERC (EUA), Ofgem (UK).

## Ferramentas

### Análise de sistemas de potência

- **PSS/E, PSS/SINCAL, DIgSILENT PowerFactory, ETAP, ATPDraw**: fluxo de carga, curto-circuito, estabilidade, proteção.
- **PSCAD, EMTP**: transitórios eletromagnéticos.
- **MATLAB Simulink + Simscape Electrical**: simulação geral.
- **HOMER Pro**: otimização de microgrids.
- **SAM (System Advisor Model)** NREL: avaliação renovável.

### Baterias e EV

- **Gamry, Bio-Logic**: caracterização eletroquímica.
- **BatPaC (Argonne), LFP-NMC tools**: modelagem custo + performance.
- **Ansys Battery, Siemens Simcenter Battery Design**: multifísica.

## Tendências 2025-2030

1. **Bateria de grid em boom**: LFP dominando, preços caindo para $60-80/kWh sistema.
2. **Eletrônica de potência em SiC/GaN** se padronizando em inversores e chargers.
3. **Nuclear modular (SMR)** entrando em mercado.
4. **Hidrogênio verde** em siderurgia e amônia (fertilizante) antes de mobilidade.
5. **Eletrificação industrial**: bombas de calor em processos < 200°C.
6. **V2G escala piloto → comercial**.
7. **AI + grid**: previsão de geração renovável, otimização de despacho, detecção de falha.
8. **Data center AI** como driver de demanda nova: gigawatts adicionais 2024-2030 pressionam grids em localização.

## Recursos

- **"Electric Power Systems"** — Weedy, Cory et al. Texto clássico.
- **"Power System Analysis and Design"** — Glover, Overbye, Sarma.
- **"Battery Design and Manufacturing"** — Gregory Plett online (UCCS, gratuito).
- **"Electric Power Distribution Handbook"** — T. A. Short.
- **"Sustainable Energy — Without the Hot Air"** — David MacKay (gratuito online). Clássico numérico e honesto.
- **"Energy and Civilization"** — Vaclav Smil. Histórico e lúcido.
- **Volts, Catalyst, BloombergNEF, IEA, IRENA** — newsletters e relatórios.
- **Practical Engineering (Grady Hillhouse)**, **Engineering with Rosie**, **Matt Ferrell (Undecided)** — YouTube sério.
- **EEVBlog** para engenharia eletrônica prática.

## Princípios

1. **Eletricidade é irrevogável**. Erros em alta tensão matam em segundos; plano antes de tocar.
2. **Aterramento é vida**. Proteções só funcionam com sistema de terra correto.
3. **Potência × energia**: watts vs watt-horas. Dimensionar requer ambos (potência de pico, energia acumulada).
4. **Transmitir em alta tensão, consumir em baixa**. Perdas cai com `I²` — razão de toda a arquitetura do grid.
5. **Intermitência exige armazenamento ou transmissão**. Solar/eólica puro sem um dos dois falha noite/vento-parado.
6. **Baterias têm química; química tem limites**. Nenhuma é perfeita; escolha por workload (densidade, vida, custo, segurança).
7. **Eficiência > geração**. O kWh mais barato é o que não se consome.
8. **Grid é sistema distribuído em tempo real** — balanço geração ↔ demanda a cada milissegundo. Falhas cascadeiam. Resiliência por design.
9. **Transição energética é física + política**. Tecnologia resolve parte; incentivos, regulação, cadeia de suprimentos resolvem o resto.
10. **Respeite eletricidade como elemento**. Corrente não perdoa; projeto e operação assumem pior caso.
