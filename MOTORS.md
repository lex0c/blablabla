# Motores

**Motor** é qualquer máquina que converte uma forma de energia em **movimento mecânico** — rotação ou translação. Elétrica (mais de 45% da eletricidade mundial passa por eles), térmica (frota de ~1.5 bilhões de veículos ICE), reação (aviação e espaço), hidráulica (industrial), pneumática (robótica). Cada tipo tem física, controle e trade-offs próprios. Este arquivo mapeia a família.

## Taxonomia

Classificação por **fonte de energia** → **princípio** → **geometria**:

| Fonte | Família | Exemplos |
|---|---|---|
| Eletricidade | eletromagnético | DC, BLDC, PMSM, indução, stepper, relutância, universal, linear |
| Combustível + ar | combustão interna | Otto, Diesel, Wankel, turbina a gás |
| Combustível + oxidante | combustão externa | Stirling, vapor (Rankine externo) |
| Combustível + oxidante (expansão em bocal) | reação | turbojet, turbofan, turboprop, ramjet, scramjet, foguete |
| Fluido pressurizado | hidráulico / pneumático | pistão, palheta, gerotor, aerofólio |
| Biológico | músculo (nem sempre chamado motor) | skeletal muscle |

## Grandezas Essenciais

- **Torque** (τ): força rotacional, N·m.
- **Potência** (P = τ · ω): N·m · rad/s = W.
- **Rotação**: rpm ou rad/s. `ω[rad/s] = rpm · 2π/60`.
- **Eficiência** (η): energia mecânica útil / energia de entrada.
- **Densidade de potência**: kW/kg ou kW/L.
- **Densidade de torque**: N·m/kg.
- **Regime de operação**: curva torque × rotação.

## Motores Elétricos

Domínio industrial total; consumidor massivo de energia global.

### Princípio

Interação entre **campo magnético** e **corrente elétrica** produz força (lei de Lorentz) ou torque via mudança de fluxo magnético (Faraday). Todos os motores elétricos derivam disso.

### Motor DC Escovado

Rotor com enrolamento + comutador mecânico + escovas. Campo por imã ou bobina de campo.

- **Prós**: controle simples (tensão ↔ velocidade, corrente ↔ torque), partida com alto torque, barato em pequeno porte.
- **Contras**: escovas desgastam, faíscas (risco em atmosfera explosiva), ruído RF.
- **Variantes**: série (locomotivas antigas, partida automóvel), shunt, compound.

Ainda presente em ferramentas elétricas baratas, brinquedos, partida (starter motors 12V).

### Motor DC Sem Escovas (BLDC)

Rotor com imãs permanentes (PM) + estator com bobinas + **comutação eletrônica** via ESC (Electronic Speed Controller). Sensores Hall ou feedback sensorless detectam posição.

- **Prós**: sem manutenção de escovas, alta eficiência (85-95%), alta densidade de torque, longa vida.
- **Contras**: exige eletrônica de controle; mais caro que DC escovado em pequeno porte.
- **Dominante em**: drones, e-bikes, ventiladores PC, HVAC modernos, veículos de pequeno porte.

### PMSM — Permanent Magnet Synchronous Motor

Primo-irmão do BLDC com distinção sutil:

- **BLDC**: força contra-eletromotriz trapezoidal, controle por blocos (6-step).
- **PMSM**: FEM senoidal, controle vetorial (FOC — Field Oriented Control).

**FOC** decompõe corrente em componentes direta (fluxo) e quadratura (torque), permite operação ótima em qualquer ponto. Requer microcontrolador rápido (DSP, ARM Cortex-M4+) e, idealmente, encoder de alta resolução.

- **Dominante em EV modernos**, robótica de precisão, CNCs, servo.
- **Imãs de terras raras** (NdFeB, SmCo) dão alta densidade — geopolítica relevante (ver `ENERGY.md`).

### Indução (IM — Induction Motor)

Tesla, 1887. Estator com campo girante; rotor em gaiola de esquilo (alumínio/cobre barrado em lâminas ferrosas). Campo girante induz corrente no rotor → torque.

- **Escorregamento (slip)**: rotor gira ligeiramente mais devagar que campo — necessário para induzir.
- **Prós**: simples, baratíssimo, extremamente robusto (sem imã, sem escova), padrão industrial.
- **Contras**: eficiência um pouco menor que PMSM (historicamente); controle mais complexo para precisão.
- **Ubíquo em**: bombas, ventiladores, compressores, esteiras, elevadores, trens.

**WRIM (Wound Rotor IM)**: rotor com enrolamento em vez de gaiola; permite resistência externa variável → controle de partida.

### Relutância Síncrona (SynRM)

Rotor de aço laminado com "barreiras de fluxo" — sem imã. Torque vem da tendência do rotor de alinhar seu eixo de menor relutância com campo.

- **Prós**: sem imã (custo/supply), eficiência alta, robustez.
- **Contras**: densidade de torque menor que PMSM.
- **Emergente em**: inversores industriais eficientes, EV low-cost (ABB, Tesla).

**PMa-SynRM** (Permanent Magnet Assisted): adiciona imãs pequenos — meio-caminho.

**SRM (Switched Reluctance Motor)**: relutância simples chaveada, rotor saliente. Alta eficiência, ruído/vibração notáveis historicamente; melhora com controle moderno.

### Motor de Passo (Stepper)

Move em passos angulares discretos (1.8°, 0.9°, etc.) ao energizar bobinas em sequência.

- **Prós**: posicionamento open-loop (sem encoder), torque parado alto, simples.
- **Contras**: perde passos em sobrecarga (sem feedback), ineficiente vs PMSM com FOC, ressonância.
- **Uso**: impressoras 3D, CNC hobby, câmeras, pequenos robôs.

Variantes: bipolar vs unipolar; híbrido (permanent magnet + variable reluctance).

### Motor Universal

Roda em AC ou DC. Escovado em série. Leve, alta rotação, alta densidade de potência por curto tempo.

- **Uso**: furadeira elétrica, liquidificador, aspirador, serra circular portátil. Mundo das ferramentas de mão.
- **Vida curta** sob uso contínuo — projetado para duty cycle intermitente.

### Servo Motor

Não é um tipo de motor *per se*, é um **pacote**: motor (DC, BLDC ou PMSM) + redução + controlador + feedback em malha fechada (encoder ou potenciômetro).

- **Hobby servo**: motor DC + engrenagens + potenciômetro + PCB controlador. PWM 20 ms define ângulo. Limitado (geralmente 0-180°) e com torque modesto.
- **Servo industrial**: PMSM + encoder incremental/absoluto de alta resolução + drive dedicado (controle de torque, velocidade, posição em cascata).

Onipresente em CNC, impressoras 3D (algumas), manipuladores industriais, antenas.

### Motor Linear

Topologia "desenrolada" de rotativo — produz **força** em linha reta.

- **Linear DC**, **linear indução** (trens maglev, catapultas), **linear síncrono** (mesas CNC de precisão, maglev Shanghai, loopers aceleradores de partículas).
- Sem transmissão (gear, belt, ball-screw) → precisão e velocidade extremas, custo alto, sensibilidade a alinhamento e carga.

### Motor Piezoelétrico

Cerâmica piezo vibra → fricção cuidadosamente controlada move elemento. Ultrapreciso (nm), pequeno, silencioso, caro. **Autofoco em câmera, posicionamento em microscopia, semiconductor stage**.

### Motor de Corrente Universal DC Direto

Em aeromodelismo moderno: outrunners BLDC (rotor externo) para hélice direta, inrunners para redução.

### Comparação Rápida (motores elétricos)

| Tipo | Eficiência típica | Controle | Densidade torque | Custo | Uso típico |
|---|---|---|---|---|---|
| DC escovado | 75-85% | simples | média | baixo | ferramentas, starter |
| BLDC | 85-95% | ESC | alta | médio | drones, e-bikes |
| PMSM | 90-97% | FOC | **altíssima** | médio-alto | EV, robótica |
| Indução | 85-95% | VFD | média | baixo | industrial |
| SynRM | 88-96% | VFD+FOC | média-alta | médio | industrial eficiente |
| Stepper | 60-80% | pulso | alta parado | baixo-médio | posicionamento |
| Universal | 50-70% | triac/PWM | alta | baixo | ferramentas manuais |

## Controle de Motores Elétricos

### VFD / VSD (Variable Frequency Drive)

Converte AC da rede → DC → AC sintético de frequência/tensão variáveis. Controla velocidade e torque de motor AC (indução ou PMSM).

- **V/f scalar**: simples — mantém razão V/f constante. Boa para bombas/ventiladores.
- **Vetorial (FOC)**: controle preciso de torque instantâneo. Padrão em servo e EV.
- **DTC (Direct Torque Control)**: ABB, alternativa a FOC.

VFDs economizam 30-60% de energia em cargas com curva cúbica (ventilador, bomba centrífuga) ao reduzir velocidade em vez de estrangular.

### ESC em drones / e-bike

Motor BLDC pequeno. ESC lê throttle + sensor Hall (ou sensorless back-EMF) → 6-step commutation ou FOC.

### Servo drive industrial

Loop fechado de corrente → velocidade → posição, cascateados. Latência microsegundos; comunicação em EtherCAT, CANopen, Profinet. Ciclo típico 100-250 µs.

### Frenagem

- **Dinâmica (resistiva)**: motor dissipa energia em banco resistor.
- **Regenerativa**: energia retorna à rede ou bateria (crucial em EV, bondes, elevadores).
- **Plug braking**: inverter polaridade brevemente.
- **DC injection**: em motor AC, aplicar DC → freio.

### Partida

- **Partida direta (DOL)**: conectar na rede. Corrente de partida 6-8× nominal.
- **Estrela-triângulo**: reduz corrente inicial em fator 3.
- **Soft-starter**: tiristores controlam rampa.
- **VFD**: rampa suave a partir de 0 Hz.

## Combustão Interna (ICE)

Queima combustível + ar dentro do motor → expansão do gás → movimento. Dominou transporte no século XX; em declínio relativo com eletrificação mas ainda dominante em 2026.

### Ciclo Otto (Gasolina)

4 tempos:

1. **Admissão**: pistão desce, válvula admissão abre, mistura ar+combustível entra.
2. **Compressão**: pistão sobe, ambas válvulas fechadas.
3. **Combustão/expansão**: vela inflama, pressão empurra pistão.
4. **Escape**: pistão sobe, válvula escape abre.

- **Razão de compressão**: ~9-12 em motores atmosféricos de gasolina (limite anti-detonação); ~14+ em Miller/Atkinson.
- **Eficiência térmica**: ~25-35% em automóvel típico. Melhor em atualidade (40%+ em motores híbridos otimizados, ex.: Toyota TNGA).

### Ciclo Diesel

Sem vela; **auto-ignição** por compressão alta.

- **Razão de compressão**: 16-22.
- **Eficiência**: 35-45% (maior que Otto por razão de compressão + ar em excesso).
- **Torque** alto em baixas rotações → tração pesada (caminhão, trator, ferroviário, marítimo).
- **Emissões**: NOx e material particulado — exigem DPF (filtro de partículas), SCR (redução catalítica seletiva com ureia/AdBlue), EGR (recirculação gases).

### Ciclos Miller / Atkinson

Válvula de admissão fecha tarde → compressão efetiva menor que expansão. Mais eficiente (45%+), menos potente — bom para híbridos onde o motor elétrico assume transientes.

### 2-Tempos vs 4-Tempos

- **2-tempos**: admissão+compressão em um movimento, combustão+escape em outro. Simples, leve, alta densidade de potência. Emissões ruins (queima óleo de lubrificação). Nichos: motosserras, outboards pequenos, scooters antigos.
- **4-tempos**: padrão em automóveis, motos modernas.

### Wankel (Rotativo)

Rotor triangular em câmara epitrocoidal. Menos peças móveis, alta rotação, densidade de potência. Emissões e consumo piores; selagem de ápices é dor. Mazda RX-7/RX-8; Mazda MX-30 R-EV usa como range extender (2023+).

### Turbo e Supercharger

- **Turbocompressor**: turbina acionada pelos gases de escape comprime ar de admissão. Sem parasitismo direto, "lag" em transientes.
- **Compressor mecânico (supercharger)**: acionado por correia. Resposta imediata, parasita potência.
- **Twin-scroll, VGT (variable geometry)**: reduz lag.
- **E-booster (compressor elétrico)**: assiste em baixa RPM.

Aumentam densidade de potência 30-100% sobre atmosférico de mesmo deslocamento → downsizing (motores menores com mesma potência, melhor consumo).

### Controle (ECU)

Motor moderno é **sistema de controle em tempo real**:

- Sensores: crank/cam position, MAP, MAF, temperatura, O₂ (lambda wideband), knock (acústico), pressão common-rail.
- Atuadores: injetores (direto ou port), bobinas de ignição, VVT/VCT (variable valve timing), wastegate, EGR valve, throttle by wire.
- ECU: 32-bit MCU (Bosch, Continental, Denso) com maps calibrados (VE, fuel, spark, boost) em memória flash.
- OBD-II / UDS para diagnóstico (ver `CAR_HACKING.md`).

### Eficiência real

- **Carros ICE típico**: 20-30% tanque-a-roda.
- **Caminhão diesel**: ~40%.
- **Navio grande** (2-tempos cross-head tipo Wärtsilä-Sulzer): **~50%** — campeão de eficiência térmica em combustão interna móvel.

### Combustíveis alternativos

- **Etanol** (E85, E100 no Brasil): octanagem alta → compressão maior; menos energia por litro.
- **GNV**: combustível "mais limpo", mas densidade de energia baixa.
- **HVO / biodiesel**: substitutos de diesel renováveis.
- **E-fuels**: sintetizados com CO₂ + H₂ — neutralidade em ciclo, caros.
- **Hidrogênio** em ICE: Toyota Corolla H₂ em testes; dominante espera-se em célula combustível, não em combustão.

## Motores Térmicos Externos

Calor gerado fora da câmara de trabalho.

### Stirling

Gás selado é aquecido e resfriado alternadamente em dois reservatórios, move pistão. Qualquer fonte de calor serve. Alta eficiência teórica (limite Carnot), baixa densidade de potência, prática em nichos: backup militar, submarinos AIP (Air-Independent Propulsion — suecos Gotland, alemães 212A), bombas solares, aplicações criogênicas (invertido = cooler).

### Vapor (Rankine)

Caldeira aquece água → vapor move pistão/turbina. Base de usinas termelétricas (ver `ENERGY.md`). Em transporte, obsoleto exceto heritage; navio a vapor dominou até meados do século XX.

## Motores de Reação

Propulsão por momento de massa ejetada — Newton 3ª lei.

### Turbojato (Turbojet)

Compressor axial/centrífugo → combustor → turbina → bocal. Turbina só basta para mover compressor; todo o empuxo vem da exaustão acelerada.

- Eficiente em alta velocidade (Mach 1+); ruidoso, consumo alto em baixa.
- Avião militar Mach 2-3 (MiG-21, Concorde).

### Turbofan

Adiciona um **fan grande** à frente, movido pela turbina via eixo concêntrico. Maior fração do ar não passa pelo combustor — é apenas acelerada pelo fan (**bypass**).

- **Alto-bypass** (10:1+): moderno comercial (CFM LEAP, Pratt GTF, GE9X). Eficiente em subsônico; grande diâmetro.
- **Baixo-bypass** (<1): caça militar.
- Dominante em aviação comercial.

### Turbopropulsor (Turboprop)

Turbina movendo hélice via redutor. Muito eficiente em baixa/média velocidade (<600 km/h).

- ATR 72, Embraer E1, Bombardier Q400, King Air.

### Turboshaft

Turbina fornece potência mecânica para eixo (sem propulsor próprio) — helicópteros (Bell, Airbus H135), tanques M1 Abrams, grupos geradores terrestres, navios.

### Ramjet

Sem compressor mecânico — ar é comprimido pela velocidade de voo. Opera ~Mach 2-5. Precisa de boost inicial. Mísseis (Meteor, BrahMos).

### Scramjet

Combustão em escoamento supersônico. Mach 5-15. Hipersônicos (X-43, X-51).

### Foguete (Rocket)

Carrega **oxidante + combustível** ambos — não depende de ar.

- **Sólido**: propelente já misturado em grão. Simples, não desliga, high thrust. ICBM, boosters, artilharia.
- **Líquido**: tanques separados + câmara + bocal de Laval. Throttle, shutdown/restart. Kerosene/LOX (RP-1, Falcon), hidrogênio/LOX (Saturn V S-II, Ariane), metano/LOX (Raptor, BE-4, modern).
- **Híbrido**: combustível sólido + oxidante líquido. Virgin Galactic.
- **Nuclear térmico**: reator aquece H₂, exaustão. DRACO em desenvolvimento (NASA+DARPA).
- **Iônico / Hall effect**: acelera íons eletricamente. Empuxo baixíssimo (µN-mN), ISP altíssimo (~3000s). Satélites, sondas interplanetárias (Dawn, Psyche, BepiColombo).
- **Plasma (VASIMR, MPD, FRC)**: pesquisa.
- **Fotônico / vela solar**: sem combustível; aceleração minúscula, campo de pesquisa (Breakthrough Starshot).

**ISP (Impulso Específico)**: métrica chave — empuxo por unidade de massa de propelente consumido por segundo. Kerolox ~300s, hidrolox ~450s, iônico ~3000s, fusão teórica 100k+s.

## Motores Hidráulicos e Pneumáticos

### Hidráulico

Fluido sob pressão (tipicamente 150-350 bar em mobile, 700+ em industrial) move motor rotativo ou cilindro linear.

- **Pistão axial**: eficiência alta, pressão alta. Escavadeira, bomba-motor.
- **Gerotor, palhetas, engrenagens**: baratos, baixas pressões.
- **Cilindros** (linear): prensas, elevadores, guindastes.

Densidade de **força** extrema — bom para cargas pesadas. Perda em mangueiras, ruído, vazamentos, fluido (óleo ou bio-óleo).

### Pneumático

Ar comprimido. Baixa precisão, rápido, barato, limpo. Indústria alimentícia, medical (evita contaminação por óleo), ferramentas de linha de montagem. Soft robotics usa atuação pneumática.

## Músculos Artificiais

Emergente — mimetizam músculo biológico:

- **SMA (Shape-Memory Alloy)**: Nitinol contrai ao aquecer.
- **HASEL (Hydraulically Amplified Self-healing ELectrostatic)**: eletrostático + hidráulico.
- **DEA (Dielectric Elastomer Actuators)**.
- **Fibras poliméricas enroladas** (Haines et al.).
- **Tecidos musculares de engenharia**: tentativas iniciais.

Densidade e vida útil ainda inferiores a motores tradicionais. Interesse em próteses, soft robotics, vestível.

## Aplicações por Domínio

### Automotivo

- **Partida (12V/48V)**: DC escovado.
- **Tração EV**: PMSM (Tesla, maioria), indução (Tesla Model S traseiro antigo), SynRM (BMW i4/iX).
- **Híbridos**: ICE + PMSM.
- **Acessórios**: BLDC, steppers, pequenos DC.

### Drones / UAVs

- **Multirotores**: BLDC outrunner + ESC + FOC.
- **Asa fixa pequena**: BLDC inrunner + redutor.
- **Grande (cargo, militar)**: turbina, híbrido.

### Robótica

- **Industrial**: servo PMSM + redutor harmonic/ciclóidico.
- **Colaborativos (cobots)**: servo com sensor de torque, controle de impedância. Ver `ROBOTICS.md`.
- **Humanoide/quadrúpede**: PMSM high-torque direct drive (e.g. MIT Cheetah QDD), frame-less motors.
- **Hidráulico em Boston Dynamics (Atlas antigo)**; BD moderno migrou para elétrico.

### Industrial

- **Bombas, compressores, esteiras**: indução + VFD.
- **Servo em CNC, máquinas ferramenta**: PMSM.
- **Elevadores**: PM synchronous gearless + VFD + regenerativo.
- **Ventiladores HVAC**: BLDC ou indução VFD.

### Aeroespacial

- Comercial: turbofan + APU turboshaft.
- Militar: turbofan baixo-bypass, afterburner, turboshaft helicóptero.
- Espaço: foguetes variedade ampla + iônico em manobra.

### Doméstico

- Máquina de lavar: BLDC inverter (Samsung, LG) ou indução.
- Geladeira: compressor scroll ou pistão, motor indução ou BLDC.
- Liquidificador, ventilador, aspirador: universal ou BLDC.
- Ar-condicionado inverter: BLDC + VFD.

### Grande escala

- Navio cargueiro (2-tempos cross-head diesel, 15-80 MW por motor).
- Locomotiva: diesel-elétrica (motor diesel → gerador → motor trativo DC ou AC indução), ou elétrica pura (catenária → trafo → motor).
- Mineração (shovel, haul truck): diesel-elétrico ou tether elétrico.

## Transmissão Mecânica

Motor + carga raramente casam diretamente. Entra:

- **Redução de engrenagens**: fixa, planetária, harmonic drive, ciclóidica. Aumenta torque, reduz velocidade.
- **Correias, correntes**.
- **CVT (continuously variable)**: razão contínua, common em scooters e alguns carros.
- **Embreagens**: desacoplam, permitem arranque, troca.

Backlash, folga, eficiência da redução (~70-98%) são considerações de projeto. Ver `MECHANICS.md` (elementos de máquina).

## Eficiência, Emissões, Regulamentação

### Regulamentação de eficiência

- **IE (IEC 60034-30)**: classes IE1 (standard) a IE5 (ultra premium) para motores industriais. Europa exige IE3/IE4 mínimo em vários casos.
- **NEMA Premium / Super Premium** nos EUA.
- **MEPS (Minimum Energy Performance Standards)** em vários países.

Transição para IE3+ motoriza economia global significativa.

### Emissões ICE

- **EURO 6/7**, **Tier 3/4 non-road**, **EPA Tier** (EUA), **Proconve** (Brasil): limitam NOx, PM, HC, CO.
- **Euro 7** (2025-2027) inclui também partículas de freios e pneus.
- **ZEV mandates**: Europa 2035 novo venda ICE proibido (em revisão 2026); Califórnia e outros estados alinhados.
- **Marítimo**: IMO 2020 (enxofre <0.5%), IMO 2050 metas de descarbonização — LNG, metanol, amônia, vento-assistido, híbrido.

### Ruído e Vibração

- **Automotivo, aéreo, industrial**: normas ISO + ABNT + setoriais. NVH (Noise, Vibration, Harshness) é subdisciplina própria.
- **Balanceamento dinâmico** em rotores — ver `MECHANICS.md`, `CONDITION_MONITORING.md`.

## Manutenção e Diagnóstico

Ver `CONDITION_MONITORING.md`. Específico a motores:

### Elétrico

- **Análise de vibração** em mancais + carcaça.
- **MCSA** (Motor Current Signature Analysis): detecta barras quebradas, excentricidade, desalinhamento via espectro da corrente, sem sensor adicional.
- **Termografia IR**: hotspots em enrolamentos, conexões.
- **Resistência de isolação (megger)**, **tangente delta**, **DC hipot, PD (partial discharge)**.
- **Análise de óleo** em redutores.

### ICE

- **Compressão e leakdown**: diagnóstico de cilindros.
- **Análise de óleo**: ferrografia, metal por espectrometria, viscosidade, fuligem.
- **Vibração e NVH**.
- **OBD-II codes** em automotivo.
- **Emissão** via tubo de escape.
- **Blow-by** (vazamento para cárter): indicador de desgaste de anéis.

### Turbina

- **EGT (Exhaust Gas Temperature)** trends — creep em palhetas.
- **Oil consumption, chip detector**.
- **Boroscopia**: câmera flexível inspeciona internamente sem desmonte.
- **Engine Health Monitoring (EHM)**: fleet data analytics (Rolls-Royce TotalCare foi pioneiro).

## Ferramentas de Projeto

### Elétrico

- **ANSYS Maxwell / Motor-CAD**: FEA eletromagnético.
- **JMAG**.
- **MATLAB Simulink / Simscape Electrical** para controle.
- **MotorSolve, FEMM** (free).
- **Opera (Cobham)**, **Flux**.

### ICE

- **AVL BOOST, Ricardo WAVE, GT-Power**: modelagem 1D de ciclo.
- **CONVERGE, STAR-CCM+, ANSYS Forte**: CFD 3D com combustão.
- **Dynos**: banco de provas com dinamômetros de freio (absorve potência).

### Turbina / Foguete

- **NPSS (NASA), Gasturb, GSP**: modelagem de ciclo de turbina.
- **CEA (Chemical Equilibrium with Applications, NASA)**: combustão de foguete.
- **Rocket Propulsion Analysis (RPA)**.
- **CFD especializado**: compressível, combustão, multifase.

## Tendências

1. **Eletrificação**: ICE de passeio em declínio; motor elétrico dominando.
2. **SiC / GaN em inversores** (ver `CIRCUIT.md`): menor peso, melhor eficiência em EV e industrial.
3. **Motor sem imã**: SynRM, SRM em ressurgência por custo/supply de terras raras.
4. **In-wheel motor** em EV: emergente (Protean, Elaphe) — maior custo unsprung mass mas elimina transmissão.
5. **Humanoide direct-drive**: motores frameless de alta density (e.g. em Optimus, Figure).
6. **Hidrogênio ICE** como nicho industrial (Komatsu, Toyota trucks test).
7. **Eletrificação da aviação**: regional (Heart Aerospace, Eviation) — limitada por bateria; híbrido de turbo-elétrico em nichos.
8. **Foguetes reutilizáveis + motano-LOX**: Raptor, BE-4 marcam era. Starship 2024-2025 test.
9. **Iônico e plasma** crescendo em satélites (Hall thrusters muitas vezes preferidos sobre químicos).

## Padrões e Normas

- **NBR 5432** (ABNT): motores de indução trifásicos.
- **IEC 60034**: motores rotativos.
- **NEMA MG-1**: EUA.
- **ISO 1940**: balanceamento.
- **ISO 10816, 20816**: vibração (ver `CONDITION_MONITORING.md`).
- **SAE J1739**: FMEA em automotivo.
- **FAR Part 33** (EUA aviação), **CS-E** (EASA): certificação de motores aeronáuticos.

## Recursos

- **"Electric Motors and Drives"** — Hughes & Drury. Referência prática.
- **"Electric Machinery"** — Fitzgerald, Kingsley, Umans. Clássico acadêmico.
- **"Internal Combustion Engine Fundamentals"** — John Heywood.
- **"Mechanics and Thermodynamics of Propulsion"** — Hill & Peterson (foguete + turbina).
- **"Rocket Propulsion Elements"** — Sutton & Biblarz.
- **"Gas Turbine Theory"** — Cohen, Rogers, Saravanamuttoo.
- **AC Propulsion, Protean, Yasa, Phi-Power**: whitepapers sobre motores EV modernos.
- **EEVBlog**, **Learn Engineering**, **Jason Cammisa** (automotivo), **Everyday Astronaut** (foguetes) — YouTube aprofundado.

## Princípios

1. **Motor certo para a carga**: perfil torque × rotação + duty cycle + ambiente. Over-engineering e sob-especificação pagam preços diferentes, igualmente caros.
2. **Eficiência nominal ≠ eficiência em uso**: motor industrial frequentemente opera parcialmente carregado onde η cai. Dimensionar corretamente.
3. **Controle define desempenho tanto quanto motor**: FOC vs scalar separa servo de bomba. ECU bem calibrada vale mais que motor exótico mal controlado.
4. **Torque × rotação = potência**. Dobrar rotação com mesma potência = metade do torque. Redução troca um pelo outro.
5. **Eletrônica de potência é 50% do motor moderno**. Inversor/ESC/driver não é acessório; é parte do sistema.
6. **Combustão interna ainda tem décadas** em cargas pesadas (marítimo, mineração, espaço). Não subestimar.
7. **Manutenção preditiva** (vibração, MCSA, óleo) paga por si rapidamente em flotas.
8. **Energia elétrica de entrada é "mais limpa" que o motor elétrico faz parecer**: matriz no local determina pegada real.
9. **Segurança**: motores armazenam energia em imãs, girando, em combustível, em pressão. LOTO, EPI, treinamento. Ver `ENERGY.md`.
10. **Todo motor é compromisso**: densidade × eficiência × custo × vida × ruído × emissão × manutenção. Não existe "melhor"; existe "mais adequado".
