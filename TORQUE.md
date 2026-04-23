# Torque

Torque é o análogo rotacional da força — o que faz algo girar, acelerar angularmente, ou manter-se rígido sob carga. Aparece em toda máquina rotativa, junta aparafusada, motor, robô, broca, roda. Este arquivo foca em **fundamentos, medição, controle e transmissão**, integrando o que está espalhado em `MOTORS.md`, `MECHANICS.md`, `GEARS.md`, `ROBOTICS.md`.

## Fundamentos

### Definição

```
τ = r × F
```

Produto vetorial entre braço de alavanca (`r`, do ponto de pivô ao ponto de aplicação) e força. Magnitude: `|τ| = |r|·|F|·sin(θ)`. Unidade SI: **N·m**.

Conceitos próximos mas distintos:

- **Torque** (τ ou T): quando a intenção é *girar*. Chave inglesa num parafuso.
- **Momento (M)**: tensão flexora em viga. Mesma dimensão, contexto estrutural.
- **Binário (couple)**: duas forças iguais e opostas em linhas paralelas, net force zero, só torque. Volante de direção puxado em ambas as mãos.

Rigor pedante aparte — em engenharia prática os três se sobrepõem.

### Unidades comuns

| Unidade | Equivalente | Contexto |
|---|---|---|
| N·m | SI | Universal |
| kgf·m | 9.807 N·m | Legado brasileiro, alguns catálogos |
| lb·ft | 1.356 N·m | Automotivo EUA |
| lb·in | 0.113 N·m | Parafusos pequenos EUA |
| oz·in | 0.00706 N·m | Motores pequenos, servo hobby |
| dyn·cm | 10⁻⁷ N·m | Físico (CGS) |

**Regra de ouro**: 100 lb·ft ≈ 135 N·m ≈ 13.8 kgf·m. Para conferir rapidamente.

### Dinâmica rotacional

Segunda lei de Newton angular:

```
τ_net = I · α
```

I = momento de inércia (kg·m²), α = aceleração angular. Análogo exato de `F = ma`.

Para corpo com torque constante:
- `ω(t) = ω₀ + α·t`
- `θ(t) = θ₀ + ω₀·t + ½α·t²`

Conservação de momento angular (L = Iω) em torque externo nulo — patinadora, giroscópio, satélite.

### Torque e potência

```
P = τ · ω           (SI: W = N·m · rad/s)
HP = T · RPM / 5252 (imperial: lb·ft · rpm)
```

De onde vem 5252: `HP = 33000 lb·ft/min`; `ω_rpm` em rad/min = `2π·RPM`; dividindo `33000 / (2π) ≈ 5252`. Torque e rotação em rpm se cruzam em HP no dashboard a 5252 rpm — curiosidade que gera gráficos característicos.

**Dobrar rotação com mesma potência = metade do torque**. Redução mecânica troca um pelo outro (menos perdas). Isso é a razão de existir de qualquer caixa de redução.

## Curvas torque × velocidade

Cada "motor" (elétrico, térmico, humano) tem curva própria. Conhecer essa curva é projeto.

### DC com escova — série

Torque explode em baixa rotação, cai monotonicamente. Arranque de veículo elétrico simples, motores de ferramentas domésticas.

### DC shunt / PMSM (ímã permanente)

Torque quase constante até a rotação base; acima, **field weakening** reduz fluxo pra ir mais rápido com menos torque (potência ~constante). Padrão em EV e servo moderno.

### AC indução

- Pico (breakdown) em alto slip (baixa rotação mas não parado).
- Torque de partida (locked rotor) alto mas não máximo.
- Região de trabalho linear perto de síncrona.
- Curva distintiva de Kloss.

### Stepper

Torque cai com rotação (perde com ressonância); zero-velocidade alto (holding torque).

### ICE (motor de combustão interna)

Curva em sino: pico de torque em meia rotação (~2000-4000 rpm carro), pico de potência em rotação mais alta. Exige caixa de marchas pra operar em zona boa.

### Turbina / turbojato

Alta rotação ótima; torque modesto em rotação de operação. Por isso **geared turbofan** existe (Pratt & Whitney GTF).

### Motor humano (ciclista profissional)

Pico ~1000-1200 W em sprint, sustentável 300-400 W. Torque no pedal ~100-200 N·m. Cadência ótima ~90 rpm.

## Torque em parafusos — o caso mais onipresente

A maioria dos engenheiros vão mais aparafusar que qualquer outra coisa mecânica. Torque de aperto é engenharia séria.

### Equação básica

```
T = K · F · d
```

- T = torque de aperto
- K = coeficiente de atrito global (~0.2 seco, 0.15 lubrificado, 0.1 anti-seize)
- F = força de pré-carga (preload) desejada
- d = diâmetro nominal

Problema: K varia ±30% por condição de superfície, lubrificação, rosca, acabamento. Torque→preload não é preciso.

### Decomposição de K (fórmula ISO 16047 / VDI 2230)

Torque total ≈ torque pra vencer atrito na **face de apoio** (cabeça ou porca) + atrito na **rosca** + componente que realmente tensiona:

```
T = F · [ 0.16·P + 0.58·μ_th·d2 + 0.5·μ_b·D_b ]
```

Apenas ~10-15% do torque vira pré-carga. 85% é perdido em atrito. **Lubrificação reduz atrito e aumenta preload** — sempre especifique condição.

### Métodos de controle

1. **Torque control**: chave com limite. Simples, impreciso (±25% na pré-carga).
2. **Torque + ângulo (Yield-controlled)**: aperta até snug torque baixo, depois N graus. Muito mais preciso (±5-10%). Padrão automotivo em juntas críticas (cabeçote, volante).
3. **Torque-to-yield (TTY)**: estira até limite elástico; parafuso deforma plasticamente. **Uso único** (substituir a cada montagem). Bielas, cabeçotes modernos.
4. **Elongation (stretch measurement)**: mede comprimento com ultrassom ou micrômetro antes/depois. Preciso (±3%), caro. Flanges de turbina, reatores.
5. **Hidráulico**: porca hidráulica tensiona o parafuso sem torquear. Zero atrito → preciso. Torre eólica, reatores.
6. **Ultrasonic load measurement**: calibra tempo de ida-volta antes, mede após aperto. High-end manufacturing.

### Torque wrenches

- **Beam**: barra flexiona, ponteiro indica. Calibração inerente.
- **Clicker**: mola calibrada; estala no torque-alvo. Mais popular. Calibrar anualmente; armazenar com mola relaxada.
- **Dial**: mostra valor contínuo, bom pra verificar.
- **Digital / electronic**: strain gauge, display, memória. Pode fazer torque+angle. Controle de qualidade industrial.
- **Hydraulic torque wrench**: alta capacidade (>10000 N·m), reações de torque. Flanges industriais.
- **Pneumatic / shut-off (nutrunner)**: linha de produção.
- **Transducerized**: sensor calibrado, PLC registra curva. Controle estatístico em montagem.

### Torque multipliers

Caixa planetária que multiplica torque manual por 3-125×. Útil em campo quando espaço/peso limitam. Reação exige ponto fixo (anti-reação).

## Rigidez torcional e eixo

Torque transmitido por eixo causa torção e tensão de cisalhamento.

### Tensão de cisalhamento

```
τ_max = T·r / J
```

- r = raio externo
- J = momento polar de inércia (eixo circular cheio: `J = πd⁴/32`; oco: `π(D⁴-d⁴)/32`)

Dimensionamento: `τ_max < τ_allow` (fadiga considerar ~60% do estático).

### Ângulo de torção

```
θ = T·L / (G·J)
```

G = módulo de cisalhamento (~80 GPa aço). Rigidez torcional `k_t = GJ/L` (N·m/rad).

### Velocidade crítica torcional

Eixo longo + cargas inerciais nas pontas forma sistema torcional vibratório. Frequência natural `f_n = (1/2π)·√(k_t/I)`. Operar próximo a `f_n` excita ressonância — quebra de eixo.

### Axial vs torcional — não confundir

Um parafuso tensionado axialmente também sofre torção durante aperto. Relaxamento torcional após aperto (rosca dá "um pulo pra trás") reduz preload inicial ~5%. Tempo de espera pós-aperto + reaperto ajuda.

## Medição de torque

### Reaction torque

Célula de carga mede força na **fixação** do estator/carcaça. Corpo → carcaça só transmite se estiver girando o rotor. Baixo custo, não precisa slip ring. Usado em dinamômetros de bancada.

### Rotary (in-line)

Strain gauges colados no eixo; sinal transmitido via:
- **Slip ring**: contato deslizante (desgasta, ruído).
- **Telemetria**: wireless RF, mais comum moderno.
- **Magnetoelástico**: medição sem contato, magnetização residual muda com tensão.
- **Optical/SAW (surface acoustic wave)**: frequência ressonante muda com torção.

Fabricantes: HBM, Kistler, Honeywell, Magtrol, Interface, NCTE (magnetoelástico).

### Estimação por corrente de motor

Para PMSM: `τ ≈ Kt · iq` (constante de torque × componente de quadratura). Preciso se Kt bem calibrado e atrito modelado. Cobots e servos usam em vez de sensor dedicado.

Limitações: variação térmica do Kt, cogging torque, ripple de comutação, atrito dependente de velocidade. Em robótica colaborativa séria, sensor dedicado na junta é preferível.

### Torque cells em robôs

- **Full wrist sensor** (6-DoF F/T): mede 3 forças + 3 torques no end-effector. ATI, Schunk, Robotiq. Essencial em impedance control, assembly, polishing.
- **Joint torque sensing**: sensor interno em cada junta. Base de robots "torque-controlled" (Franka Emika, Kuka iiwa). Permite controle macio, interação segura, detecção de colisão.

### Calibração

Torque sensors exigem certificação periódica rastreável (ISO 17025). Bancos de torque padrão nacionais (NIST EUA, PTB Alemanha, Inmetro BR).

## Controle de torque em motores

### Controle escalar V/f (AC indução)

Mantém razão V/f constante. Torque aproximadamente constante em rotação ampla, resposta lenta. Bom pra ventiladores, bombas. Mau pra posicionamento.

### Field-Oriented Control (FOC)

Transformações Clarke e Park decompõem correntes trifásicas em duas componentes:
- **id** (direct): controla fluxo.
- **iq** (quadrature): controla torque (`τ = Kt · iq` em PMSM).

Com encoder ou observer de posição, controle independente de fluxo e torque. Resposta de torque em ms, bandwidth kHz. **Padrão em servo moderno, EV, cobot, drone**.

Implementação: DSP ou ARM Cortex-M4+ com periféricos dedicados (timers PWM, ADC sincronizados). Pacotes: InstaSPIN (TI), STM32 Motor Control SDK, ODrive, VESC.

### Direct Torque Control (DTC)

Alternativa ABB a FOC. Seleciona vetor de tensão do inversor direto, sem PWM intermediário. Resposta torque ainda mais rápida, ruído maior, controle mais complexo. Usado em drives industriais ABB.

### Cascade: position → velocity → torque

Servos modernos têm **três loops aninhados**:

```
comando posição → PID posição → ref velocidade → PID velocidade → ref torque → loop torque/corrente
```

Bandwidth crescente de fora pra dentro: posição ~100 Hz, velocidade ~1 kHz, torque ~10 kHz. Separar permite sintonia independente.

### Torque ripple

Fontes:
- **Cogging** (PMSM): interação ímã-ranhura. Minimizado com skew, números de polos/slots coprimos.
- **Commutation** (DC com escova): transição entre escovas cria picos.
- **6th harmonic** (PMSM): imperfeições de FOC + harmônicas de EMF.
- **ICE**: pulsos de combustão. Volante de inércia filtra.

Robótica de precisão exige ripple < 1%. ICE + volante + embreagem + transmissão = filtro mecânico de ripple firing.

## Transmissão de torque — além de engrenagens

### Eixos e árvores

Elemento primário. Dimensiona por fadiga (Soderberg/Goodman), rigidez torcional, velocidade crítica, concentração de tensão em keyways/filetes.

### Acoplamentos

- **Rígidos** (flange, sleeve, taper lock): transmitem sem compensação.
- **Flexíveis**:
  - **Elastoméricos** (jaw, tire): absorvem choque, tolerância média.
  - **Disco / membrane**: alta torção, baixa deslocamento.
  - **Grid (grade)**: molas serpentinas, absorve shock.
  - **Universal / Cardan**: ângulos grandes (até 30°); gera variação de velocidade secundária (homocinético evita — CV joints).
  - **Gear coupling**: alto torque, alguma angulação.
  - **Magnetic**: sem contato; limite de torque fixo (slip em sobrecarga).
- **Torque limiters**: slip clutch (atrito), shear pin, ball detent, magnetic. Proteção contra jam.

### Embreagens

Acopla/desacopla sob carga. Fricção seca (carro manual), banho úmido (moto, ATV), eletromagnética (industrial, impressora), viscosa (AWD), magnética (climatizador carro).

### Conversor de torque

Automotivo AT. Rotor bombeador (bomba) + turbina + estator, óleo como fluido de transmissão. **Multiplica torque** em baixa rotação (até 2-3×) com escorregamento. Lock-up clutch elimina perdas em cruzeiro.

### CVT (continuously variable transmission)

Razão contínua entre limites. Variantes:
- **Polias + correia/faixa metálica** (Van Doorne). Carros japoneses, scooters.
- **Toroidal** (Extroid, NSK). Industrial.
- **e-CVT** (Toyota): planetária + dois motores elétricos, sem correia. Brilhante.

Torque limitado pela capacidade de atrito da correia/faixa.

### DCT (dual clutch)

Duas embreagens, duas árvores paralelas (marchas pares e ímpares). Troca sem cortar torque. VAG DSG, Getrag PowerShift. Rápido e eficiente; custa complexidade e mecatrônica.

## Diferenciais — distribuição de torque

Entre duas rodas/eixos mesmo com diferentes velocidades.

### Aberto

Permite velocidades distintas; torque dividido igualmente. Problema: roda com menor tração domina — spin perde torque no outro lado.

### Limited-slip (LSD)

- **Clutch-pack (viscous, mechanical)**: atrito interno trava parcialmente.
- **Gerodisc, Torsen** (gear-based, torque-biasing): usa pressão de hipoide pra travar proporcional ao torque.
- **Eletronicamente controlado** (Haldex AWD, ATTESA): embreagens servo-acionadas.

### Lockers

Trava rígida: 100% equalização. Off-road, ATV.

### Torque vectoring

Diferencial ativo distribui torque **desigualmente** entre rodas — carro "gira mais" porque roda externa recebe mais torque. Mitsubishi Evo, M3 Competition, Porsche PTV+. Em EV, dois motores (um por roda) fazem nativamente.

## Torque em regeneração

Motor como gerador. Em EV, frenagem regenerativa converte energia cinética em elétrica. Torque de frenagem:
- Pequeno em baixa velocidade (eficiência baixa).
- Máximo perto de rotação nominal.
- Limitado por: batería SoC (cheia = rejeita), aderência, limite elétrico do inverter.

Blending entre regen e freio mecânico é software delicado.

## Segurança e confiabilidade

### Safety factor

Industrial: 3-5 em torque estático, 5-10 em fadiga. Aeroespacial: tipicamente menor (peso), validado estatisticamente (fatores estatísticos A-basis, B-basis).

### Fadiga torcional

Tensão cíclica causa propagação de trinca em chaveta, filete, raiz de rosca. Nucleação superficial domina; **shot peening** aumenta vida ~20-50%.

### Slip vs fail

Projete pra slippar (fusível) em vez de fraturar. Shear pin, slip clutch, magnetic coupling protegem downstream de torque súbito.

### Brake vs park

**Self-locking** (sem-fim) pode segurar torque estático mas não é "freio" confiável — vibração, choque, temperatura degradam. Sistema com carga gravitacional deve ter freio positivo (disc brake, pawl).

## Aplicações práticas — calibragem de intuição

| Contexto | Ordem de grandeza |
|---|---|
| Parafuso M6 (preload normal) | 10 N·m |
| Parafuso M12 (cabeçote) | 100 N·m |
| Roda de carro | 90-140 N·m |
| Motor elétrico ferramenta (parafusadeira) | 30-80 N·m |
| Motor EV (Tesla Model 3 traseiro) | ~400 N·m |
| Motor carro 2L NA | ~200 N·m |
| Motor diesel V8 pickup | ~600-900 N·m |
| Motor navio (RTA-96C) | ~7,600,000 N·m |
| Junta de cobot (Franka) | ~10-90 N·m |
| Junta de robô industrial grande (KR 500) | ~10,000 N·m |
| Torque de partida locomotiva diesel-elétrica | ~400,000 N·m |
| Turbina eólica (low-speed shaft) | ~5,000,000 N·m |

Saber essas ordens economiza erro grosseiro em spec.

## Torque em robótica colaborativa

Cobots medem torque em cada junta para:

- **Impedance control**: junta responde como mola-amortecedor virtual. Empurrar robô "dá pra empurrar" em vez de resistir rígido.
- **Admittance control**: inverso de impedância. Force-input → velocity-output.
- **Collision detection**: torque externo estimado (real - esperado do modelo dinâmico). Above limiar → para.
- **Force-controlled assembly**: push-until-contact, peg-in-hole, polimento. Exige low impedance.
- **Zero-gravity / hand-guided teaching**: operador move robô; controle cancela gravidade.

Hardware: sensor na junta (Franka, iiwa) ou wrist F/T (UR + sensor externo). Robô de braço industrial puro (Fanuc M-*, KUKA KR) geralmente não sente torque — por isso perímetro de segurança físico.

## Princípios

1. **Torque e potência são coisas diferentes**. `P = τω`. Confusão custa projeto.
2. **Redução troca torque por velocidade**, conservando potência menos perdas. Default.
3. **Torque de aperto é estimativa, não medida de preload**. Use angle control em juntas críticas.
4. **Atrito come 80-90% do torque aplicado**. Lubrificação muda tudo.
5. **FOC é o padrão moderno** de controle de torque elétrico. Domine.
6. **Stiffness mata resposta em baixa velocidade**; compliance intencional permite controle fino.
7. **Projete pra slip, não fratura**. Torque limiter é barato; eixo quebrado, não.
8. **Meça quando possível; estime quando precisa**. Sensor > observer > modelo + corrente > pura especificação.
9. **Torque ripple é onipresente** em motores elétricos. Filtro mecânico (volante, elastômero) e elétrico (FOC avançado) atacam.
10. **Unidades consistentes**. SI em cálculo, converte no final. Erros em lb·ft vs N·m destroem hardware.

## Recursos

- **"Shigley's Mechanical Engineering Design"** — Budynas & Nisbett. Cálculo de eixo, parafusos, fadiga.
- **"VDI 2230"** — cálculo de junta parafusada, alemão, referência.
- **"Bolted Joint Engineering"** — Bickford. Bíblia de preload.
- **"Electric Motors and Drives"** — Austin Hughes. FOC explicado sem mágica.
- **"Industrial Motor Control"** — Herman.
- **"Robot Dynamics and Control"** — Spong, Hutchinson, Vidyasagar.
- **"Impedance and Admittance Control"** — Hogan (paper seminal 1985).
- **Kistler, HBM, ATI** — application notes de torque e F/T sensing.
- **TI InstaSPIN, ST MC SDK** — FOC real em código.
- **ISO 16047, ISO 898-1** — normas de parafuso e materiais.
- **"Engineering Mechanics"** — Hibbeler / Meriam. Fundamentos.
