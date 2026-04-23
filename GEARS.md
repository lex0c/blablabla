# Engrenagens

Transmissão mecânica positiva (sem escorregamento) entre eixos via dentes engrenados. Transforma torque e velocidade mantendo potência (menos perdas). É um dos elementos de máquina mais antigos e ainda dos mais densos em engenharia — geometria, materiais, lubrificação, ruído, fabricação e falha são disciplinas próprias.

Fundamentos mecânicos em `MECHANICS.md`. Motores (clientes típicos) em `MOTORS.md`. Controle em `ROBOTICS.md`. Torque em `TORQUE.md`.

## Por que dente involuto — conjugate action

Uma engrenagem não é "círculo com saliências". Para transmitir movimento com **razão constante** (ω₁/ω₂ fixo em todo o engrenamento), a lei fundamental exige que a normal comum às superfícies em contato passe sempre pelo **pitch point** (ponto fixo na linha de centros). Isso é *conjugate action*.

A curva que satisfaz isso universalmente, com dentes intercambiáveis, é a **involuta de círculo**. Desenrola-se um fio tenso do círculo base; a ponta traça a involuta. Propriedades:

- **Razão constante** independente da distância entre centros (tolerante a erro de montagem).
- **Linha de ação** é reta tangente aos círculos base das duas engrenagens.
- **Ângulo de pressão** constante ao longo da ação.
- **Fácil de cortar** com ferramenta reta (rack-cutter, hob) — chave pra fabricação em massa.

Alternativas (cicloidal, circular-arc) existem mas involuta é o padrão absoluto desde ~1900.

## Nomenclatura

Parâmetros geométricos que aparecem em todo catálogo e cálculo:

- **Módulo (m)**: diâmetro primitivo dividido por número de dentes, em mm. `m = d/Z`. Padrão métrico. Dois dentes engrenam se têm mesmo módulo e ângulo de pressão.
- **Diametral Pitch (DP)**: dentes por polegada de diâmetro primitivo. Inverso do módulo (1/DP ≈ 25.4/m). Padrão imperial.
- **Pitch circle** (primitivo): círculo imaginário onde dentes "rolariam sem escorregar" se fossem rodas lisas. Diâmetro `d = m·Z`.
- **Base circle**: círculo de onde involuta é gerada. `db = d·cos(α)`.
- **Ângulo de pressão (α)**: ângulo entre linha de ação e tangente ao pitch circle. **20°** é o padrão mundial; 14.5° (legado), 25° (alta carga), 17.5°.
- **Addendum (a)**: altura do dente acima do pitch. `a = m`.
- **Dedendum (b)**: profundidade abaixo do pitch. `b = 1.25m` típico.
- **Whole depth**: `a + b`.
- **Clearance**: folga no fundo. `c = 0.25m`.
- **Circular pitch (p)**: distância entre dentes medida ao longo do pitch circle. `p = π·m`.
- **Chordal thickness**: espessura do dente na corda do pitch.
- **Line of action**: trajetória do ponto de contato, tangente aos base circles.
- **Contact ratio (mc)**: dentes em contato simultâneo em média. >1.2 mínimo pra engrenamento contínuo; 1.4-2.0 típico em spur; >2.0 em helical bem projetadas.
- **Backlash**: folga angular entre dentes quando flancos opostos estão em contato. Necessário pra lubrificação, dilatação térmica, deformação sob carga. Zero backlash exige preload ou projeto especial.
- **Profile shift / addendum modification (x)**: deslocamento da ferramenta no corte. Permite casar engrenagens com `Z` pequeno sem undercut, ajustar distância entre centros, equilibrar desgaste.

## Tipos

### Spur (cilíndrica de dentes retos)

Eixos paralelos, dentes paralelos ao eixo. Mais simples, mais barata, mais ruidosa. Sem empuxo axial. Contact ratio modesto (1.4-1.7). Usada em redutores industriais, caixas leves, protótipos. Default quando ruído/velocidade não são críticos.

### Helical (helicoidal)

Dentes em hélice (ângulo β típico 15-30°). Engrenamento gradual → muito mais silenciosa, maior contact ratio (até 4+), transmite mais potência por tamanho. Gera **empuxo axial** → exige rolamento de encosto.

**Herringbone (double helical / dupla helicoidal)**: duas hélices espelhadas, cancelam empuxo. Cara de fabricar (Sykes-type cutter). Usada em transmissões pesadas (navio, turbina).

### Cônicas (bevel)

Eixos concorrentes (geralmente 90°). Dentes em cone.

- **Straight bevel**: análogo spur; ruidosa.
- **Spiral bevel**: análogo helical; silenciosa, maior capacidade.
- **Zerol**: spiral bevel com ângulo espiral zero — meio-termo.
- **Hypoid**: eixos não se cruzam (offset), típico diferencial traseiro automotivo. Permite pinhão abaixo do eixo (piso mais baixo). Alto escorregamento exige **óleo hipoide (GL-5 com EP extreme pressure)**.

### Worm (parafuso sem-fim)

Parafuso + coroa. Razão de redução enorme em um único estágio (até 300:1). **Auto-lock** se ângulo de avanço < ângulo de atrito: coroa não consegue mover o parafuso (seguro pra carga gravitacional; use com cautela — não é brake verdadeiro). Eficiência: 40-95% dependendo de razão, material, lubrificação. Desgaste alto (escorregamento puro na linha de contato); coroa em **bronze fosforoso** + parafuso em aço endurecido + óleo EP.

### Rack and pinion (cremalheira)

Rack = engrenagem de raio infinito (linear). Converte rotação ↔ translação. Usada em: direção automotiva, máquina-ferramenta, portões, elevadores de obra, atuadores lineares.

### Engrenagens internas (anel / ring gear)

Dentes na face interna. Casam com pinhão externo. Fundamentais em trens planetários e motores de arranque.

### Crown / face gear

Dentes na face plana, não na circunferência. Nichos (instrumentos, antigos tanks).

### Spiroid, helicon

Variantes híbridas worm-bevel, usos especializados.

## Trens de engrenagens — cálculo de razão

### Trem simples

`i = Z2/Z1 = ω1/ω2 = τ2/τ1`

Velocidade divide, torque multiplica. Eficiência ~98% por estágio (spur/helical).

### Trem composto

Múltiplos pares em série no mesmo eixo. Razão total = produto das razões individuais.

### Trem revertido

Eixos de entrada e saída coaxiais. Comum em caixas de câmbio manual.

### Idler gear

Engrenagem intermediária que **inverte o sentido** sem afetar razão. Qualquer engrenagem intermediária desaparece da equação — só entra a inicial e final.

## Engrenagens planetárias (epicicloidais)

O elemento mais importante em transmissão moderna. Três membros:

- **Sun** (sol, central)
- **Planets** (3-5 satélites)
- **Ring** (coroa interna)
- **Carrier** (suporte dos planetas)

**Fórmula fundamental de Willis**:

```
(Zs + Zr) · ωc = Zs · ωs + Zr · ωr
```

Mantenha um elemento fixo e duas funções resultam:

| Fixo | Entrada | Saída | Razão |
|---|---|---|---|
| Ring | Sun | Carrier | `i = 1 + Zr/Zs` (redução) |
| Carrier | Sun | Ring | `i = -Zr/Zs` (inversão) |
| Sun | Carrier | Ring | `i = 1 + Zs/Zr` (redução suave) |

### Propriedades

- **Compacta**: carga distribuída entre 3-5 planetas; densidade de torque altíssima.
- **Coaxial**: entrada e saída no mesmo eixo.
- **Múltiplas razões** em um mesmo conjunto ao travar/liberar diferentes membros (automatic transmission).
- **Power split**: um membro recebe parte da potência, outro recupera. Base de **e-CVT** do Toyota Prius.
- **Torque equilibrado nos planetas** — sem carga radial sobre os rolamentos do carrier (se Z bem distribuído).

### Composições

- **Ravigneaux**: dois sóis, um par de planetas (um longo + um curto), um ring. 4+ razões com três clutches. ZF 6-speed, Ford/GM antigos.
- **Simpson**: dois planetários compartilhando carrier+ring. TurboHydramatic GM clássica.
- **Lepelletier**: Ravigneaux + planetary frontal. Base dos ZF 6HP/8HP modernos (BMW, Audi, Jaguar).
- **Wolfrom**: redução extrema em um passo (~100:1). Usa em cobot joints.

### Aplicações

- Transmissões automáticas (AT): conversor de torque + clutches + planetárias.
- Hub gears em bicicletas (Rohloff 14-speed).
- Redutor de turbina (geared turbofan).
- EV single-speed reducers.
- Guinchos, escavadeiras, tratores.
- Bocais de ferramentas (parafusadeiras elétricas).

## Harmonic drive (strain wave)

Inventado por Musser (1957). Três componentes:

1. **Wave generator**: cam elíptico + rolamento flexível.
2. **Flex spline**: copo de aço flexível com dentes externos (Z fixo).
3. **Circular spline**: anel rígido com dentes internos (Z + 2 tipicamente).

Wave generator deforma flex spline em elipse; apenas poucos dentes engrenam nos "bicos" da elipse. Diferença de 2 dentes por volta do wave generator → redução enorme em um estágio (50:1 até 320:1).

**Vantagens**:
- Zero backlash real (pré-carga intrínseca na deformação).
- Densidade de torque altíssima.
- Precisão (repetibilidade < 1 arcmin).
- Coaxial.

**Desvantagens**:
- Eficiência 70-85% (deformação elástica consome energia).
- Fadiga do flex spline limita vida.
- Custo alto (HarmonicDrive AG patentes; Leaderdrive chinês).
- Windup (flex em carga) — rigidez torcional < planetária.

**Uso**: robôs industriais (juntas), cobots (UR, Franka), satélite, óptica de precisão, semi.

## Cicloidal drive / RV

Alternativa a harmonic drive, mais robusta a choque.

- Disco cicloidal excêntrico + ring de pinos + output pins/rolos.
- Redução em um estágio 10-200:1.
- **RV reducer** (Nabtesco, dominante em robótica japonesa): combinação planetária + cicloidal. Backlash baixo, rigidez alta, **absorve choque** melhor que harmonic.

**Uso**: juntas de robôs industriais pesados (Fanuc, ABB, Yaskawa), escavadoras, sistemas que tomam impacto.

## Modos de falha (AGMA)

Engrenagem não falha aleatoriamente — segue padrões. Diagnóstico é inspeção + análise de óleo.

### Desgaste (wear)

- **Adhesion** / pickup: filme de óleo rompe, solda microscópica. Superfícies levemente deterioradas.
- **Abrasion**: partícula dura risca dente. Filtro de óleo importa.
- **Corrosive wear**: ácido na lubrificação (aditivos degradados, condensação).

### Fadiga superficial

- **Pitting** (micro, macro): cratera causada por ciclos hertzianos sob pico de contato. Início em ~10⁶ ciclos típicos. **Indicador precoce** de sobrecarga.
- **Spalling**: lascas maiores — progressão avançada de pitting.
- **Micropitting (frosting)**: superfície acinzentada microscópica. Óleo de viscosidade baixa em alta carga.

### Fadiga de flexão (bending fatigue)

Tensão cíclica na raiz do dente. Propaga trinca até **fratura do dente** (catastrófica). Raio de filete na raiz é determinante. **Lewis equation**:

```
σ = W·Pd / (F·Y)
```

W = carga tangencial, Pd = diametral pitch, F = largura, Y = Lewis form factor (tabelado). Moderna AGMA 2001 refina com fatores de concentração, sobrecarga, distribuição.

### Scuffing / scoring

Ruptura do filme EHL em alta carga + velocidade + temperatura. Solda-e-arranca das superfícies. Risca profunda radial. Óleo EP (extreme pressure) mitiga.

### Fratura de dente

Trinca atingindo profundidade crítica. Fim de vida; perigo secundário (fragmentos no óleo destroem resto do trem).

### Plastic deformation

Rolling, rippling, peening — escoamento superficial sob sobrecarga em material insuficientemente duro. Surface hardness < core hardness.

## Materiais e tratamento

### Aços comuns (pinhões e coroas)

- **SAE 8620, 4320, 9310**: cementação (carburizing). Superfície 60 HRC, núcleo 30-35 HRC tenaz. Mais comum em engrenagens de alta carga.
- **SAE 4140, 4340**: thru-hardening, quench + temper. Uniforme, mais simples, menor capacidade que cementado.
- **SAE 4150, 4150H**: indução hardening — aquece dente, quencha, núcleo mole.
- **Inox**: 17-4PH, 440C — corrosão importa. Capacidade menor.

### Não-ferrosos

- **Bronze fosforoso** — coroa worm (baixo atrito, escorregamento alto).
- **Latão** — instrumentos, baixo torque.
- **Alumínio** — equipamentos não-carga (relógios de oficina).

### Polímeros

- **POM (acetal)**, **nylon 6/6 com lubrificante**, **PEEK**. Baixo custo, silenciosos, leves, auto-lubrificantes. Carga modesta. Crescente em eletrodomésticos, automotivos leves (regulador de janela, wiper), robótica hobby.

### Tratamentos superficiais

- **Carburizing (cementação)**: difusão de C, quench. HRC 58-63 em 0.5-2.5 mm de profundidade. Padrão em alta carga.
- **Nitriding (nitretação)**: difusão de N em baixa temperatura. Distorção mínima. HV 700-1100. Aços Nitralloy, 31CrMoV9.
- **Carbonitriding**: carbono + nitrogênio. Compromisso.
- **Induction hardening**: só superfície, distorção baixa, rápido.
- **Shot peening**: pré-tensão compressiva na superfície. Aumenta vida em fadiga ~20-50%.
- **Superfinishing / ISF (isotropic superfinishing)**: reduz rugosidade a < 0.1 μm Ra. Menor atrito, maior vida.

## Lubrificação

Engrenagem engrenada opera em regime **EHL (elastohydrodynamic lubrication)**: filme de óleo de 0.1-1 μm sob pressão de GPa deforma elasticamente as superfícies. Filme é tudo — sem ele, contato metal-metal → scuffing imediato.

### Viscosidade

- ISO VG (Viscosity Grade) padroniza em cSt a 40°C. VG 220, 320, 460 típicos em redutores industriais.
- SAE J306 pra automóvel (75W-90, 80W-140, etc).

### Classificação API GL

- **GL-1**: simples, manual sem EP.
- **GL-4**: EP moderado. Manuais comuns.
- **GL-5**: EP alto (aditivos S-P). Diferenciais hipoides. **Não usar GL-5 em sincronizadores de latão** — aditivo corrói.
- **GL-6**: obsoleto.
- **MT-1**: heavy-duty manual.

### Aditivos EP (extreme pressure)

Reagem com superfície em alta temperatura, formando filme sacrificial sulfurado/fosfatado. Compromisso: lubrificação em extremo vs corrosão em amarelos (bronze, cobre).

### Métodos

- **Splash**: banho parcial; carcaça do redutor com óleo. Barato; suficiente até certa velocidade tangencial.
- **Spray / jet**: bomba força óleo nos dentes. Alta velocidade.
- **Mist**: óleo atomizado em ar. Alta velocidade, baixa contaminação.
- **Graxa**: em sistemas abertos ou low-duty. NLGI #2 típico.

### Análise de óleo

- Ferrografia, espectrometria de elementos, TAN/TBN. Tendência de Fe, Cu, Si é indicador precoce de falha. Base de manutenção preditiva.

## Ruído e NVH

Engrenagens "cantam". Fonte: **transmission error (TE)** — desvio angular entre teórico e real por variação de rigidez ao longo do engrenamento + erros de perfil + pitch + runout.

### Mitigação

- **Helical > spur** (engrenamento gradual).
- **Profile modifications**: tip relief, root relief, crowning. Compensam deflexão sob carga, evitam impacto na entrada/saída do engrenamento.
- **Precision grade** (AGMA Q): Q10-Q14 típico industrial; Q14+ automotivo premium; Q15+ aeroespacial.
- **Balanceamento dinâmico**.
- **Damping** na carcaça.

### Ordens

Ruído tem ordem = número de dentes × rotação. FFT revela "gear mesh frequency" e harmônicos. Sidebands em ±shaft frequency indicam desalinhamento/excentricidade.

## Manufatura

### Corte

- **Hobbing**: hob (sem-fim com dentes cortantes) em movimento síncrono com blank. Altíssima produtividade em externos. Padrão industrial.
- **Shaping (Fellows)**: cutter piniforme reciprocante. Único método pra engrenagem interna (além de broach/skiving).
- **Broaching**: rápido, carro de dentes cortando em uma passada. Alta produção (automotivo).
- **Skiving (power skiving)**: alternativa a shaping pra internas. Rápido. Emergente em máquinas CNC 5-eixos.
- **Milling (form cutter)**: cutter com forma do espaço entre dentes. Flexível; baixa produtividade e precisão. Oficinas pequenas.

### Acabamento

- **Shaving**: "razor" remove 0.02-0.1 mm com cutter helicoidal. Pré-tratamento.
- **Grinding**: pós-tratamento após hardening. Máquinas Reishauer, Gleason. Cara, precisa.
- **Honing**: super-acabamento com ferramenta abrasiva.
- **Lapping**: abrasivo em suspensão; acaba pares cônicas hipoides.
- **Superfinishing / ISF**: vibratório com mídia, reduz rugosidade isotropicamente.

### Aditiva

Metal 3D printing (DMLS, LPBF) + HIP + heat treat + grinding. Ainda caro, viável pra geometria impossível subtractive (gear interno de planetário integrado a carcaça).

## Eficiência típica

| Tipo | Eficiência por estágio |
|---|---|
| Spur | 98-99% |
| Helical | 97-98% |
| Bevel | 95-98% |
| Hipoid | 85-95% |
| Worm (razão baixa) | 80-95% |
| Worm (razão alta) | 40-70% |
| Planetária | 95-98% |
| Harmonic | 70-85% |
| Cicloidal / RV | 85-95% |

Multiestágio multiplica perdas. Redutor de 4 estágios spur: 0.98⁴ = 92%.

## Normas

- **AGMA** (American Gear Manufacturers Association): 2001 (rating), 915 (inspection), 923 (materials).
- **ISO 6336**: cálculo de capacidade (bending, pitting).
- **DIN 3990, 867** (legado alemão, referência ampla).
- **JIS B 1701/1702** (Japão).
- **ANSI/AGMA 6006** (wind turbine gearbox) — baseline prático pra alta confiabilidade.
- **VDI 2230** (parafusos — tangentelmente relevante em montagem).

## Processo de projeto (sizing típico)

1. **Definir carga**: torque entrada/saída, razão, duty cycle, shock factor.
2. **Escolher tipo**: ruído, eixos, redução, ambiente.
3. **Material + tratamento**: baseline 8620 cementado.
4. **Módulo e largura** pra satisfazer bending (Lewis/AGMA) + pitting (Hertz/AGMA).
5. **Razão**: distribuir entre estágios (proporcional ao output típico pra minimizar tamanho).
6. **Profile shift** se Z pequeno ou distância não-padrão.
7. **Lubrificação + rolamentos + carcaça**.
8. **Verificar NVH e vida L10 dos rolamentos**.
9. **Prototipar + ensaiar** (bancada de carga, run-in, análise de óleo).
10. **Iterar**.

Software: KISSsoft, Romax, AGMA Mesh Generator, Masta. Em projetos sérios, cálculo manual é sanity-check; software é obrigatório.

## Aplicações — onde cada tipo brilha

- **Carro automatic (AT)**: planetárias compostas (Simpson, Ravigneaux, Lepelletier).
- **Carro manual (MT)**: helical com sincronizadores.
- **Diferencial traseiro RWD**: bevel hipoide.
- **DCT (dual-clutch)**: helicais em eixos paralelos + duas embreagens.
- **CVT automotivo**: polia variável + correia aço (não engrenagem tradicional).
- **EV single-speed**: helical (reduce 8-11:1 típico).
- **Robô industrial 6-axis**: RV nas juntas 1-3, harmonic nas 4-6.
- **Cobot leve**: harmonic em todas as juntas.
- **Eletrodoméstico**: polímero (POM) + sem-fim.
- **Guincho industrial**: worm self-locking ou planetária com freio.
- **Turbina eólica**: planetária + helical paralelo, 3 estágios, ~100:1.
- **Aeroespacial (turbofan geared)**: planetária de alta precisão (Pratt & Whitney GTF).
- **Relógio mecânico**: cicloidais modificadas (Mickey Mouse), razões irracionais pra não repetir desgaste.
- **Parafusadeira elétrica**: planetária 2-3 estágios com clutch ajustável.

## Princípios

1. **Involuta + 20° + módulo** é a base universal. Desvie com motivo.
2. **Um estágio resolve 3-10:1**; além disso, estagie.
3. **Planetária é trunfo de densidade de torque**. Domine antes de qualquer projeto compacto.
4. **Helical silencia, empurra axialmente**. Rolamento de encosto é parte do projeto.
5. **Harmonic zero-backlash, cicloidal robusta a choque**. Escolha pelo ambiente dinâmico.
6. **Falha começa na superfície**. Pitting é teu canário. Monitor óleo.
7. **EHL é tudo**. Viscosidade errada vira scuffing em horas.
8. **Bending fadiga mata dente; pitting mata vida útil**. Duas análises separadas, ambas obrigatórias.
9. **Precision costs**. Q15 é 10× mais caro que Q10; escolha pela aplicação.
10. **Lubrificação certa, não cara**: GL-5 em sincronizadores de latão destrói. Leia o manual.

## Recursos

- **"Dudley's Handbook of Practical Gear Design and Manufacture"** — Radzevich. Bíblia.
- **"Gear Geometry and Applied Theory"** — Litvin. Matemático, profundo.
- **"Design of Machine Elements"** — Shigley / Budynas (capítulos 13-15). Base universitária.
- **AGMA standards** — ref obrigatória.
- **KHK Gear Technical Reference** — fabricante japonês, PDF excelente.
- **KISSsoft Theory Manual** — expansão moderna das normas.
- **"Handbook of Gears"** — Khiralla, disponível para clássicos.
- **Gear Technology magazine** — atualidade industrial.
- **Bosch Automotive Handbook** — seção de transmissão.
