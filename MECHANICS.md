# Mecânica

Mecânica é o ramo da física e da engenharia que estuda **corpos, forças e movimento**. É a disciplina-mãe da engenharia — pontes, motores, aviões, robôs, prédios, implantes ortopédicos, todos se apoiam nela. Enquanto `CIRCUIT.md` cobre o mundo elétrico, aqui está o mundo mecânico: estática (corpos em equilíbrio), dinâmica (corpos em movimento), resistência (como materiais falham), fluidos, vibrações, e os elementos de máquina que a engenharia usa para transformar teoria em objeto.

Complementa `CIRCUIT.md`, `ROBOTICS.md`, `COMPUTER_THEORY.md`, `LINEARITY.md`.

## Grandezas Fundamentais

| Grandeza | Símbolo | Unidade SI | Intuição |
|---|---|---|---|
| Massa | m | kg | inércia |
| Força | F | newton (N = kg·m/s²) | push/pull |
| Torque | τ | N·m | força aplicada em braço |
| Momento linear | p = mv | kg·m/s | "quantidade de movimento" |
| Momento angular | L = Iω | kg·m²/s | rotação |
| Energia | E | joule (J = N·m) | capacidade de trabalho |
| Potência | P | watt (W = J/s) | taxa de energia |
| Pressão | p | pascal (Pa = N/m²) | força por área |
| Tensão (stress) | σ, τ | Pa (MPa comum) | força interna por área |
| Deformação (strain) | ε | adimensional | Δl/l |
| Velocidade angular | ω | rad/s | rotação |
| Momento de inércia | I | kg·m² | inércia rotacional |

## Leis de Newton

Base da mecânica clássica:

1. **Primeira lei (inércia)**: corpo em repouso permanece em repouso; em movimento retilíneo uniforme permanece — salvo força externa.
2. **Segunda lei**: `F = m · a` (ou mais geralmente, `F = dp/dt`).
3. **Terceira lei**: para toda ação, reação de igual módulo e sentido oposto.

Aparentemente simples; dominam aplicações desde bicicleta até foguete.

**Limites**: válidas em referencial inercial, velocidades << c, escalas não-quânticas. Para o resto da engenharia civil/mecânica, funcionam.

## Estática

Corpos em equilíbrio. Requisitos:

1. **Equilíbrio de forças**: `Σ F = 0` em cada eixo.
2. **Equilíbrio de momentos**: `Σ τ = 0` em cada ponto.

### Diagrama de corpo livre (DCL)

Ferramenta central. Isolar o corpo; desenhar **todas** as forças atuando (peso, normais, atrito, tensão, reações de apoio, cargas aplicadas). Aplicar equações.

### Tipos de apoios

- **Rolete / apoio simples**: reação normal.
- **Pino (articulação)**: reações em 2 eixos, sem momento.
- **Engaste**: 2 reações + momento.
- **Cabo**: tensão ao longo do cabo.

### Treliças (trusses)

Estruturas de barras conectadas em nós, cargas aplicadas só nos nós. Métodos:

- **Método dos nós**: equilíbrio em cada nó; barras são tração ou compressão puras.
- **Método das seções**: cortar estrutura e aplicar equilíbrio em pedaço.

Treliças são eficientes em peso/resistência — telhados, pontes, torres.

### Atrito

- **Estático**: `F_s ≤ μ_s · N`. Resiste até limite.
- **Cinético**: `F_k = μ_k · N`. Menor que estático.
- Independente de área de contato (modelo de Coulomb, aproximação).
- **Ângulo de atrito**: `tan(φ) = μ`. Planos inclinados, deslizamento.

## Dinâmica

Corpos em movimento — com aceleração.

### Cinemática

Descreve movimento sem as causas:

- **MRU**: `x = x₀ + v·t`.
- **MRUV**: `v = v₀ + a·t`, `x = x₀ + v₀t + ½at²`, `v² = v₀² + 2a·Δx`.
- **Projétil**: horizontal MRU + vertical MRUV sob gravidade.
- **MCU**: `a_c = v²/r = ω²r`, direção centrípeta.
- **Rotação uniforme**: `θ = θ₀ + ωt`.

### Dinâmica propriamente

`F = ma` ou, para rotação, `τ = Iα`.

### Energia e Trabalho

- **Trabalho**: `W = F · d · cos(θ) = ∫ F · ds`.
- **Energia cinética**: `K = ½mv²` (translação) + `½Iω²` (rotação).
- **Energia potencial gravitacional**: `U_g = mgh`.
- **Energia potencial elástica**: `U_e = ½kx²`.
- **Conservação**: sistema isolado, `K + U = const`. Fricção/dissipação perde para calor.

### Momento Linear

- `p = mv`.
- **Conservação**: ausência de força externa resultante → momento constante. Base de colisões, propulsão.
- **Impulso**: `J = F·Δt = Δp`.

### Colisões

- **Elástica**: preserva K e p.
- **Inelástica**: preserva p, perde K (calor, deformação).
- **Perfeitamente inelástica**: corpos colam.
- **Coeficiente de restituição** `e`: razão de velocidades de separação/aproximação. `e=1` elástica; `e=0` inelástica total.

### Rotação

- **Momento de inércia**: `I = ∫ r² dm`. Exemplos: disco `½mr²`, esfera sólida `⅖mr²`, barra sobre centro `¹⁄₁₂ mL²`.
- **Teorema dos eixos paralelos**: `I = I_cm + md²`.
- **Energia de rotação**: `½Iω²`.
- **Momento angular**: `L = Iω`. Conservação em torque nulo (giroscópios, patinadora fechando braços).

### Referenciais não-inerciais

Aparecem **forças fictícias**:

- **Centrífuga**: `-mω²r` em sistema girando.
- **Coriolis**: `-2m(ω × v)`. Afeta sistemas globais (ventos, correntes marítimas).

## Formulações Analíticas

Equivalentes a Newton, mais poderosas em sistemas complexos.

### Lagrangiana

`L = K - U`. Equações de Euler-Lagrange:

```
d/dt (∂L/∂q̇ᵢ) - ∂L/∂qᵢ = 0
```

Para coordenadas generalizadas `qᵢ`. Ignora forças de vínculo; automático em sistemas restringidos.

### Hamiltoniana

`H = Σ pᵢq̇ᵢ - L`. Equações canônicas:

```
q̇ᵢ = ∂H/∂pᵢ
ṗᵢ = -∂H/∂qᵢ
```

Base de mecânica quântica e estatística. Geometria simplética.

### Quando usar

- **Newton**: problemas simples, ensino, força claramente identificada.
- **Lagrange**: vínculos, múltiplos graus de liberdade (robótica — ver `ROBOTICS.md`).
- **Hamilton**: teórico, conservação, formulações modernas de física.

## Resistência dos Materiais

Como materiais respondem a carregamento — e quando falham.

### Tensão (Stress)

Força interna por área:

- **Normal (σ)**: perpendicular à superfície. Tração (+) ou compressão (-).
- **Cisalhamento (τ)**: paralela à superfície.

Unidades: Pa, MPa, GPa. Aço estrutural típico: σ_escoamento ~250-500 MPa.

### Deformação (Strain)

Relativa:

- **Normal**: `ε = Δl/l₀`.
- **Cisalhamento**: ângulo de distorção.

Adimensional; frequentemente em µm/m ou %.

### Lei de Hooke

Regime elástico linear:

```
σ = E · ε        (tração)
τ = G · γ        (cisalhamento)
```

- **E** = módulo de elasticidade (Young). Aço ~200 GPa; alumínio ~70; concreto ~30; madeira ~10; polímeros ~1-3.
- **G** = módulo de cisalhamento.
- **ν** (coeficiente de Poisson): `-ε_transversal/ε_axial`. Típico 0.25-0.35 em metais; borracha ~0.5.
- Relação: `G = E / (2(1+ν))`.

### Curva tensão-deformação

Teste de tração revela:

1. **Regime elástico**: linear (Hooke).
2. **Limite de proporcionalidade**.
3. **Limite elástico** (escoamento, σ_y).
4. **Regime plástico**: deformação permanente.
5. **Tensão de resistência última** (σ_u).
6. **Ruptura**.

**Dúctil** (aço, alumínio): grande deformação plástica antes de falhar.
**Frágil** (vidro, cerâmica, concreto em tração): pouca deformação; falha súbita.

### Tipos de carregamento

- **Tração** (tração axial).
- **Compressão** — em barras longas: risco de **flambagem (buckling)**, instabilidade geométrica. Fórmula de Euler: `P_crit = π²EI / (KL)²`.
- **Flexão**: viga submetida a momento. Tensão `σ = M·c/I` (flexão).
- **Cisalhamento**: `τ = V·Q/(I·b)` em viga.
- **Torção**: `τ = T·r/J` em eixo circular.
- **Combinada**: superposição + critérios (Von Mises, Tresca).

### Critérios de falha

- **Tensão máxima principal** (Rankine): frágeis.
- **Cisalhamento máximo (Tresca)**: dúcteis conservador.
- **Energia de distorção (von Mises)**: dúcteis, padrão. `σ_vM ≤ σ_y`.

### Fadiga

Cargas cíclicas → falha em tensões muito abaixo do σ_u estático. Curva **S-N** (tensão vs número de ciclos). Aço tem limite de fadiga (~10⁶ ciclos infinitos); alumínio não.

Concentradores de tensão (furos, entalhes, mudança abrupta de seção) são pontos críticos. Design cuidadoso reduz drasticamente falha.

### Fratura

- **Dúctil**: deformação plástica extensa, copa-e-cone.
- **Frágil**: pouca deformação, superfície planar.
- **Mecânica da fratura**: trinca + K_IC (fator de intensidade de tensão crítico). Controla falha catastrófica em estruturas com defeitos.

### Creep

Deformação lenta sob carga constante em alta temperatura. Turbinas, reatores. Resistência específica (superligas Ni-based).

## Elementos de Máquina

Componentes padronizados que compõem máquinas:

### Parafusos e porcas

- **Roscas métricas** (M6, M10, M20): diâmetro × passo.
- **UN/UNC/UNF** (americanas): TPI.
- **Classe de resistência**: aço classe 8.8 (σ_y ~640 MPa), 10.9, 12.9.
- **Torque de aperto**: `T = K·F·d`. K depende de atrito (~0.2 para aço seco).
- **Preload**: força axial induzida pelo aperto — resiste cargas cíclicas.
- **Loctite / porcas de travamento**: anti-desrosqueamento.

### Rolamentos

- **Esferas**: radial, baixo atrito.
- **Rolos cilíndricos**: cargas radiais altas.
- **Rolos cônicos (tapered)**: combinam radial + axial.
- **Rolos esféricos (autocompensadores)**: desalinhamento.
- **Vida útil L10**: ciclos em que 10% falham. `L10 = (C/P)^p`.
- **Lubrificação** (graxa, óleo), **selagem**, temperatura — afetam vida.

### Engrenagens

- **Cilíndricas de dentes retos (spur)**: paralelos, ruidosas.
- **Helicoidais**: dentes inclinados; silenciosas; empuxo axial.
- **Cônicas (bevel)**: eixos perpendiculares.
- **Parafuso sem-fim (worm)**: redução alta em um estágio, não reverso.
- **Planetária (epicicloidal)**: compacta, alta relação.

Parâmetros: módulo (m = D/Z), número de dentes, ângulo de pressão (20° padrão).

### Molas

- **Helicoidais**: `k = Gd⁴/(8D³n)` (compressão/tração).
- **Torção, disco (Belleville), lâmina**.
- **Gás / pneumáticas**: não-lineares.

### Acoplamentos

- **Rígidos**: transmitem sem absorção.
- **Flexíveis** (elastoméricos, garras, grade): toleram desalinhamento.
- **Universal (cardan)**: ângulos grandes.
- **Constant-velocity (CV)**: transmissão de carro.

### Correias e correntes

- **Correias planas / em V / sincrônicas (GT, HTD)**: baixo ruído, barato, escorregamento.
- **Correntes**: transmitem mais torque, mais ruidosas, exigem lubrificação.

### Selos e gaxetas

- O-rings, lip seals, mechanical seals, labirinto.

## Mecânica dos Fluidos

### Fundamentos

- **Densidade (ρ)**, **viscosidade (μ)**.
- **Pressão hidrostática**: `p = ρgh`.
- **Princípio de Arquimedes**: empuxo = peso de fluido deslocado.

### Escoamento

- **Laminar**: camadas ordenadas, baixo Re.
- **Turbulento**: caótico, alto Re.
- **Número de Reynolds**: `Re = ρvL/μ`. Em tubos: Re < 2300 laminar; >4000 turbulento.

### Equação de Bernoulli

Em escoamento ideal (incompressível, invíscido, permanente):

```
p + ½ρv² + ρgh = const
```

Aproximação útil para asas, venturi, tubos de Pitot — em regime apropriado.

### Navier-Stokes

Equação completa do fluido newtoniano viscoso. **Um dos sete Problemas do Milênio** (existência/suavidade em 3D não provada). Resolvida numericamente (CFD) com Simulink/ANSYS Fluent/OpenFOAM/SimScale.

### Perda de carga em tubos

- **Darcy-Weisbach**: `Δp = f(L/D) · ½ρv²`.
- **Fator de atrito f**: Moody chart; função de Re e rugosidade relativa.

### Arrasto e sustentação

- **Arrasto**: `F_D = ½ρv²C_D A`.
- **Sustentação**: `F_L = ½ρv²C_L A`.
- **C_D, C_L**: coeficientes dependentes de forma e Re/Mach.

Aerodinâmica, hidrodinâmica, veículos, aeronaves.

### Compressibilidade

- **Mach**: `M = v/a`. Subsônico M<0.8, transônico 0.8-1.2, supersônico >1.2, hipersônico >5.
- **Ondas de choque** acima de Mach 1.

## Termodinâmica

### Leis

0. **Lei zero**: equilíbrio térmico é transitivo → define temperatura.
1. **Primeira lei**: conservação de energia. `ΔU = Q - W`.
2. **Segunda lei**: entropia de sistema isolado nunca diminui. `ΔS ≥ 0`. Impossibilidade de máquina perfeita / de perpétua.
3. **Terceira lei**: entropia de cristal perfeito a T=0K é zero.

### Ciclos

- **Carnot**: referencial de máxima eficiência teórica. `η = 1 - T_frio/T_quente`.
- **Otto**: motor a gasolina.
- **Diesel**: motor diesel.
- **Rankine**: turbina a vapor (usina termoelétrica).
- **Brayton**: turbina a gás (avião).
- **Refrigeração (vapor compressão)**: inverso, bomba de calor.

### Transferência de calor

- **Condução**: `q = -kA(dT/dx)`. Fourier.
- **Convecção**: `q = hA(T_sup - T_∞)`. h depende de escoamento.
- **Radiação**: `q = εσAT⁴`. Stefan-Boltzmann.

Análise térmica em componentes, HVAC, dissipadores, spacecraft.

## Vibrações

### Sistema massa-mola-amortecedor

`m·ẍ + c·ẋ + k·x = F(t)`.

- **Frequência natural**: `ω_n = √(k/m)`.
- **Razão de amortecimento**: `ζ = c / (2√(km))`.
- **Subamortecido (ζ<1)**: oscila.
- **Criticamente amortecido (ζ=1)**: retorno mais rápido sem oscilação.
- **Superamortecido (ζ>1)**: lento, sem oscilação.

### Resposta em frequência

Sistemas mecânicos têm **ressonância** em ω_n. Amplificação ~1/2ζ. Projetar para evitar excitação nessa banda — ou amortecer.

### Análise modal

Sistemas com múltiplos graus de liberdade têm **modos** (vetores próprios) e **frequências** (valores próprios). Base da análise de edifícios sob sismo, asas sob flutter, eixos em rotação.

### Balanceamento

Desbalanceamento em rotor produz força centrífuga cíclica → vibração + fadiga. Balanceamento estático e dinâmico em bancadas especializadas.

## Materiais

### Metais

- **Aço**: Fe-C-liga. Versátil, forte, soldável. Classes: carbono (AISI 1020, 1045), inoxidável (304, 316, 17-4PH), ferramentas.
- **Alumínio**: leve (⅓ do aço), corrosão moderada. 6061, 7075 (aeroespacial).
- **Cobre, latão, bronze**: condutividade, resistência à corrosão marinha.
- **Titânio**: resistência/peso alta, biocompatível, caro.
- **Superligas** (Ni, Co): turbinas, reatores.

### Polímeros

- **Termoplásticos**: reciclável por aquecimento. ABS, PC, PP, PE, PET, PEEK, PA (nylon).
- **Termofixos**: curam irreversível. Epóxi, fenólica.
- **Elastômeros**: borracha natural, SBR, nitrila, silicone, EPDM.

### Compósitos

- **Fibra de vidro** (GFRP): barato, pesado-médio.
- **Carbono** (CFRP): muito forte/leve, caro, frágil em impacto.
- **Aramida (Kevlar)**: impacto, balística.
- **Sanduíche**: honeycomb + skin.

### Cerâmica

- Óxidos, carbetos, nitretos. Alta dureza, alta temperatura, frágil.
- **Biocerâmica** (próteses), **estrutural** (turbinas), **eletrônica**.

### Seleção de material

Diagramas de Ashby: comparar em eixos (E vs ρ, σ_y vs custo). Escolha depende de carga, ambiente, processo, custo, ciclo de vida.

## Manufatura

### Processos primários

- **Fundição**: areia, molde permanente, cera perdida, injeção (Al, Zn).
- **Conformação**: laminação, forjamento, extrusão, trefilação, estampagem.
- **Usinagem**: torneamento, fresamento, furação, retificação, EDM (eletroerosão). CNC dominante.
- **Soldagem**: eletrodo revestido (MMA), MIG/MAG, TIG, arco submerso, laser, resistência.
- **Pulverizaçao/additive**: SLA, SLS, FDM, binder jetting, directed energy deposition.
- **Corte**: plasma, oxicorte, laser, water jet.
- **União**: parafusos, rebites, adesivos estruturais (epóxi, acrílico), brazagem.

### Tratamentos térmicos

- **Têmpera + revenimento** em aço: aumenta dureza + tenacidade.
- **Recozimento**: alivia tensões, aumenta ductilidade.
- **Cementação, nitretação**: dureza superficial mantendo núcleo tenaz.
- **Envelhecimento** em ligas precipitação-endurecíveis (Al 7075, Inconel).

### Acabamentos

- **Pintura, anodização (Al), cromagem, galvanização, black oxide**.
- **Passivação** (inox).
- **Shot peening**: tensões compressivas residuais aumentam fadiga.

## Tolerâncias e Ajustes

Sem tolerância, nada encaixa. ISO GPS (Geometric Product Specification):

### Dimensionais

- `20 ±0.1`: tolerância bilateral.
- `H7/h6`, `H7/g6`, `H7/p6`: ajustes padronizados — folga, justo, interferência.
- **IT grades** (IT6, IT7): define magnitude da tolerância conforme tamanho.

### Geométricas (GD&T / ISO 1101)

Características de forma, orientação, posição, runout:

- **Retilinidade, planeza, circularidade, cilindricidade**.
- **Perpendicularidade, paralelismo, angularidade**.
- **Posição, concentricidade, simetria**.
- **Runout (batimento)**.

Datums fornecem referência. Desenhos com GD&T comunicam função, não só dimensão.

### Rugosidade

`Ra`, `Rz`: rugosidade média. µm. Usinado médio ~1.6 µm; retificado 0.2-0.8; polido <0.1.

## CAD / CAE

### CAD

- **Sólidos** paramétricos: SolidWorks, Inventor, Creo, Fusion 360, Onshape, Solid Edge.
- **Direct modeling**: SpaceClaim.
- **Livres / scriptable**: FreeCAD, OpenSCAD (code-first), CadQuery.
- **Superfícies / styling**: Rhino, Alias, CATIA.
- **AEC / civil**: Revit, AutoCAD.

### CAE

- **FEA (Análise por Elementos Finitos)**: tensão, deformação, modal, térmica, não-linear. ANSYS Mechanical, Abaqus, Nastran, LS-DYNA (crash), COMSOL (multi-física), SimScale (web).
- **CFD**: Fluent, CFX, OpenFOAM, Star-CCM+.
- **MBD (multibody dynamics)**: Adams, RecurDyn.
- **Otimização topológica**: genera formas otimizadas em peso/rigidez (nTopology, Altair Inspire, Tosca).

### CAM

- **G-code** e pós-processadores para CNC.
- **Mastercam, Fusion CAM, HSMWorks, NX CAM, FeatureCAM**.

## Princípios de Projeto Mecânico

1. **Fator de segurança (FoS)**: `FoS = σ_permitido / σ_atuante`. Tipicamente 1.5-4 dependendo de criticidade, incerteza, modo de falha.
2. **Minimizar concentradores de tensão**: raios em cantos, não rasgos agudos.
3. **Cargas axiais > cisalhamento > flexão** em eficiência.
4. **Prefira compressão em concreto/cerâmica**; tração em metais dúcteis.
5. **Tolerâncias frouxas onde não importa, apertadas onde encaixa função**. Cada décimo custa.
6. **Design for manufacturing (DFM)**: consulte o processo antes de finalizar geometria.
7. **Design for assembly (DFA)**: menos peças, menos parafusos, menos orientações.
8. **Design for maintenance**: acesso a manutenção preventiva.
9. **Modularize**: interfaces padrão entre subsistemas.
10. **Redundância onde falha é cara**: aviação, médico.

## Segurança

- **EPI**: óculos, luvas apropriadas (corte vs químico), botas, protetor auricular.
- **Lockout/tagout (LOTO)**: isolar energia antes de manutenção.
- **Guards em máquinas rotativas**.
- **Zonas de exclusão** em ensaios de alta energia (pressão, vácuo, molas comprimidas).
- **Limite de elevação manual** (NR-17 no Brasil).
- **Design failsafe**: falha vai a estado seguro.

## Biomecânica (brevemente)

Aplicação de mecânica ao corpo vivo:

- **Próteses** (quadril, joelho): Ti, UHMWPE, CrCo; ajuste, fadiga, biocompatibilidade.
- **Marcha e postura**: forças em articulações.
- **Biomateriais**: compromisso entre rigidez do implante e osso (stress shielding).

Intersecta com `ROBOTICS.md` (exoesqueletos, órteses).

## Tendências

- **Manufatura aditiva industrial**: topologia otimizada, peças antes impossíveis.
- **Simulação acoplada** CFD + FEM + térmico: dimensionamento holístico.
- **Digital twin**: gêmeo virtual do produto físico alimentado por sensores em operação.
- **Automação + cobots** (ver `ROBOTICS.md`).
- **Materiais avançados**: compósitos, metamaterials, estruturas bioinspiradas.
- **Sustentabilidade**: life cycle analysis, design for recycling, economia circular.

## Recursos

- **"Shigley's Mechanical Engineering Design"** — Budynas & Nisbett. Referência canônica em projeto.
- **"Mechanics of Materials"** — Hibbeler / Gere.
- **"Engineering Mechanics: Statics & Dynamics"** — Hibbeler / Meriam-Kraige.
- **"Fundamentals of Fluid Mechanics"** — Munson.
- **"Fundamentals of Thermodynamics"** — Borgnakke & Sonntag.
- **"Materials Science and Engineering"** — Callister.
- **"The Art of Electronics"** para contexto elétrico-mecânico.
- **Manufacturing Processes**: Kalpakjian.
- **Practical Engineering (Grady Hillhouse)**, **Engineering Explained**, **SmarterEveryDay** — YouTube sério.
- **Norma ISO 1101, ASME Y14.5** — GD&T.
- **Machinery's Handbook** — a bíblia do projeto mecânico.

## Princípios

1. **Diagrama de corpo livre primeiro**. 80% dos erros em estática vêm de DCL preguiçoso.
2. **Unidades sempre**. Newton, pascal, metro. Errar fator de mil destrói ponte.
3. **Materiais têm personalidade**. Dúctil ≠ frágil ≠ fadiga ≠ fluência. Escolha consciente.
4. **Concentradores de tensão são silenciosos**. Raios generosos, evitar entalhes súbitos.
5. **Fator de segurança é humildade quantificada**. Calibre pela consequência.
6. **Meça antes de presumir**. Rigidez, tensão, frequência modal — intuições enganam.
7. **Ressonância mata**. Em vibração, evite ω_n do sistema.
8. **Tolerância realista**. Cada décimo de mm custa processo; especifique o necessário.
9. **Itere protótipo**. Simulação ajuda; nada substitui peça na mão.
10. **Respeite a energia armazenada**. Molas, pressão, volante em rotação — têm vontade própria.
