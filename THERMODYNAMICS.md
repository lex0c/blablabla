# Termodinâmica — Energia, Calor e Conversão

Energia no sentido físico: **que formas ela assume, como se transforma, quanto se perde e por quê**. `ENERGY.md` trata da cadeia elétrica (grid, geração em escala, baterias); `MECHANICS.md` tem o resumo de leis e ciclos dentro do escopo mecânico; aqui está o conceito completo — incluindo a parte que decide tudo na prática: **qualidade** da energia, não só quantidade.

## Conceitos de base

- **Sistema, fronteira, vizinhança:** o recorte que você escolhe analisar. **Fechado** troca energia mas não massa (pistão selado); **aberto / volume de controle** troca ambos (turbina, bocal, radiador); **isolado** não troca nada.
- **Propriedade intensiva × extensiva:** intensiva não depende do tamanho (temperatura, pressão, densidade); extensiva depende (massa, volume, energia). Dividir extensiva por massa dá a **específica** (`u`, `h`, `s` em J/kg).
- **Estado e processo:** estado é o conjunto de propriedades num instante; processo é o caminho entre estados. **Ciclo** volta ao estado inicial.
- **Equilíbrio:** sem gradientes internos de temperatura, pressão ou composição. Toda a termodinâmica clássica fala de estados de equilíbrio — o caminho entre eles é aproximação.
- **Reversível × irreversível:** reversível é a idealização em que nada se dissipa e o processo pode voltar sem deixar rastro. **Tudo que é real é irreversível**: atrito, mistura, diferença finita de temperatura, expansão livre. Irreversibilidade é exatamente o que gera entropia.
- **Calor (Q) × trabalho (W):** ambos são **energia em trânsito**, não coisas que um corpo "tem". Um corpo tem **energia interna (U)**. Calor é transferência por diferença de temperatura; trabalho é transferência organizada (força × deslocamento, torque × ângulo, tensão × carga).
- **Temperatura × calor:** temperatura é intensiva (medida da agitação média); calor é energia transferida. Uma faísca de vela a 800 °C tem energia irrisória; uma banheira a 40 °C tem muitíssima. **Confundir os dois é o erro conceitual mais comum da área.**
- **Gás ideal:** `P·V = n·R·T`. Boa aproximação longe de condensação e de pressões extremas.

## Formas de energia e transformação

Toda a engenharia energética é uma cadeia de conversões, e **cada elo cobra sua taxa**.

| Forma | Onde aparece | Expressão típica |
|---|---|---|
| **Cinética** | volante, veículo, vento, turbina | `½mv²`, `½Iω²` |
| **Potencial gravitacional** | reservatório, hidrelétrica reversível | `mgh` |
| **Elástica** | mola, gás comprimido, arco | `½kx²` |
| **Térmica (interna)** | qualquer corpo acima do zero absoluto | `m·c·ΔT` |
| **Química** | combustível, alimento, bateria | entalpia de reação |
| **Elétrica** | corrente, campo, capacitor | `V·I·t`, `½CV²`, `½LI²` |
| **Radiante** | luz solar, IR, micro-ondas | `E = hν` por fóton |
| **Nuclear** | fissão, fusão | `E = mc²` |

### Cadeias reais e seus rendimentos

| Conversão | Dispositivo | Rendimento típico |
|---|---|---|
| Elétrica → mecânica | motor elétrico industrial | **85–97%** |
| Elétrica → elétrica | transformador de potência | 98–99,5% |
| Elétrica → térmica | resistência (chuveiro, forno) | ~100% (mas veja *exergia*) |
| Elétrica → luz | LED | ~30–40% |
| Elétrica → luz | incandescente | ~5% |
| Química → térmica | queima em caldeira | 80–95% |
| Térmica → mecânica | motor Otto automotivo | 20–30% |
| Térmica → mecânica | diesel grande (marítimo) | ~50% |
| Térmica → mecânica | turbina a gás simples | 35–42% |
| Térmica → elétrica | termelétrica a carvão | 33–40% |
| Térmica → elétrica | **ciclo combinado a gás** | **58–64%** |
| Térmica → elétrica | nuclear (Rankine) | 33–37% |
| Química → elétrica | célula a combustível | 50–60% |
| Química → elétrica | bateria (round-trip) | 80–95% |
| Radiante → elétrica | fotovoltaico comercial | 20–23% |
| Radiante → química | fotossíntese | 1–2% |
| Cinética → elétrica | turbina eólica | 35–45% (limite de Betz: 59,3%) |
| Potencial → elétrica | hidrelétrica | 85–92% |

**Leitura:** os rendimentos ruins estão todos no mesmo lugar — **quando o calor está no meio do caminho**. Conversões que não passam por calor (elétrica↔mecânica, potencial↔elétrica) são quase perfeitas. Isso não é falha de engenharia; é a **segunda lei**.

### Multiplicando a cadeia

Rendimentos se multiplicam. Comparação clássica, do poço à roda (*well-to-wheel*):

- **Carro a combustão:** extração/refino (~85%) × motor (~25%) × transmissão (~90%) ≈ **~19%**.
- **Carro elétrico com rede a gás:** usina ciclo combinado (60%) × transmissão (93%) × carregador (90%) × bateria (95%) × inversor+motor (90%) ≈ **~43%**.
- **Carro elétrico com rede hidrelétrica** (caso brasileiro): a primeira etapa vira ~90% → **~65%**.

O elo que domina é sempre o pior. Otimizar um elo de 95% é irrelevante enquanto houver um de 25% na cadeia.

## Unidades e ordens de grandeza

| Unidade | Em joules | Contexto |
|---|---|---|
| 1 J | 1 | SI |
| 1 cal | 4,184 | química |
| 1 kcal ("Caloria" de alimento) | 4.184 | nutrição |
| 1 BTU | 1.055 | HVAC americano |
| 1 Wh | 3.600 | elétrica |
| **1 kWh** | **3,6 MJ** | conta de luz |
| 1 TEP (tonelada equiv. petróleo) | 41,87 GJ | balanço energético nacional |
| 1 t de TNT | 4,184 GJ | explosivos |
| 1 eV | 1,602×10⁻¹⁹ | física atômica |

### Escala, para calibrar intuição

| Evento | Energia |
|---|---|
| Fóton de luz visível | ~3×10⁻¹⁹ J (~2 eV) |
| Pilha AA alcalina | ~14 kJ (~4 Wh) |
| Bateria de celular | ~50 kJ (~14 Wh) |
| Barra de chocolate (100 g) | ~2,2 MJ (~530 kcal) |
| 1 L de gasolina | ~32 MJ (~8,9 kWh) |
| Bateria de EV (75 kWh) | 270 MJ |
| Alimentação humana, 1 dia | ~8,4 MJ (~2.000 kcal ≈ **100 W médios**) |
| Casa brasileira, 1 mês (200 kWh) | 720 MJ |
| Raio | ~10⁹ J (ordem de grandeza) |
| Bomba de Hiroshima | ~63 TJ (15 kt) |
| Terremoto M7.0 | ~2×10¹⁵ J |
| Sol, por segundo | 3,8×10²⁶ J |

Duas regras práticas que resolvem muita conta de guardanapo:

- **1 kW por 1 h = 1 kWh.** Chuveiro de 5.500 W por 10 min = 0,92 kWh.
- **Um humano em atividade sustentada rende ~100 W.** Uma bicicleta gerando eletricidade não carrega nem um secador de cabelo.

## As leis

### Lei zero

Se A está em equilíbrio térmico com C e B também, então A e B estão entre si. É o que **define temperatura** e permite que termômetro exista. Parece trivial; foi enunciada depois das outras, daí o "zero".

### Primeira lei — conservação

`ΔU = Q − W`

Energia não é criada nem destruída, só muda de forma. Convenção usual: **Q positivo entra** no sistema, **W positivo sai** (o sistema realiza trabalho).

Para **volume de controle** (escoamento), usa-se a **entalpia**: `H = U + P·V`, porque o fluido que entra carrega, além da energia interna, o trabalho de empurrar-se para dentro. Turbinas, compressores, bocais e trocadores são todos analisados em `h`, não em `u`.

"Energia se perde" é linguagem informal e **está errada**: ela se **degrada**. A quantidade se conserva; o que some é a utilidade.

### Segunda lei — degradação

Vários enunciados equivalentes:

- **Clausius:** calor não flui espontaneamente do frio para o quente.
- **Kelvin-Planck:** nenhuma máquina cíclica converte **todo** o calor de uma fonte em trabalho. Sempre há rejeito para um sorvedouro frio.
- **Entropia:** em sistema isolado, `ΔS ≥ 0`. Igualdade só no caso reversível (inatingível).

Visão estatística (Boltzmann): `S = k·ln W`, onde W é o número de microestados compatíveis com o macroestado. **Entropia é contagem**: estados desordenados são esmagadoramente mais numerosos, então o sistema "cai" neles. Não há força empurrando — é estatística de números gigantescos.

Consequência prática direta: **não existe moto-perpétuo**. De 1ª espécie viola a conservação; de 2ª espécie respeita a conservação mas tentaria converter calor ambiente 100% em trabalho — e esse é o que ainda aparece em pitch de investidor.

### Terceira lei

A entropia de um cristal perfeito tende a zero quando T → 0 K. Corolário operacional: **o zero absoluto é inatingível** em número finito de etapas. Cada etapa de resfriamento fica mais cara que a anterior — por isso criogenia a milikelvin é engenharia pesada.

## Qualidade da energia: exergia

O conceito que falta na conversa comum sobre energia, e o que realmente decide projeto.

- **Exergia:** a parcela da energia que pode virar **trabalho útil**, dado um ambiente de referência (T₀ do meio).
- **Anergia:** o resto — energia que só sabe ser calor ambiente.

`Energia = exergia + anergia`. A **1ª lei conserva a soma; a 2ª lei destrói a exergia**.

Por isso:

- **1 kWh de eletricidade ≈ 1 kWh de exergia** (converte quase todo em qualquer coisa).
- **1 kWh de calor a 40 °C ≈ 0,05 kWh de exergia** — praticamente inútil para produzir trabalho.
- **1 kWh de calor a 800 °C ≈ 0,7 kWh de exergia.**

Isso explica desconfortos aparentes:

- Um **chuveiro elétrico é 100% eficiente pela 1ª lei** e um desastre pela 2ª: destrói eletricidade (exergia pura) para fazer água morna (exergia quase zero). É como usar um bisturi para cortar mato.
- Uma **bomba de calor** com COP 4 entrega o mesmo aquecimento gastando **1/4** da eletricidade — porque **move** calor em vez de **criar**. Não viola nada: o resto veio do ambiente.
- **Cogeração (CHP)** só faz sentido por exergia: extrair primeiro o trabalho (alta temperatura) e usar o rejeito para calor de processo (baixa temperatura). Rendimento total pode passar de 85%.

**Eficiência de 2ª lei** = o que você conseguiu ÷ o máximo termodinamicamente possível. É a métrica honesta. Uma caldeira de 90% de eficiência (1ª lei) costuma ter eficiência de 2ª lei em torno de **10%**.

## Máquinas térmicas, refrigeradores e bombas de calor

Todas são o mesmo dispositivo, mudando o que se quer:

| Máquina | Objetivo | Métrica | Fórmula |
|---|---|---|---|
| Motor térmico | trabalho | rendimento η | `W/Q_quente` |
| Refrigerador | tirar calor do frio | COP_r | `Q_frio/W` |
| Bomba de calor | entregar calor ao quente | COP_bc | `Q_quente/W = COP_r + 1` |

### Limites de Carnot (temperaturas em **kelvin**)

- `η_Carnot = 1 − T_frio/T_quente`
- `COP_r,Carnot = T_frio/(T_quente − T_frio)`
- `COP_bc,Carnot = T_quente/(T_quente − T_frio)`

**Toda a estratégia de projeto sai daí: aumentar T_quente e/ou diminuir T_frio.** É por isso que turbinas a gás perseguem temperaturas de entrada de 1600 °C (e viram um problema de metalurgia e resfriamento de pá, não de termodinâmica), e por que usinas se instalam à beira de rio ou mar.

Exemplo: ciclo com fonte a 550 °C (823 K) rejeitando a 30 °C (303 K) tem teto de `1 − 303/823 = 63%`. Uma usina real ali entregando 40% está a ~63% do máximo possível — não é "40% de ineficiência de engenharia".

### COP na prática

- Ar-condicionado split inverter: **COP 3–5** (frequentemente anunciado como IEER/SEER).
- Bomba de calor para aquecimento: COP 3–4 a 7 °C externos, caindo para 2–2,5 abaixo de zero (o ΔT cresce e ainda entra o degelo).
- Geladeira doméstica: COP 1,5–3.

Por que o COP cai no frio é direto pela fórmula: `T_quente − T_frio` no denominador aumenta.

## Ciclos

### Potência a gás (fluido sempre gasoso)

- **Otto** (faísca): compressão isentrópica → **calor a volume constante** → expansão → rejeição. `η = 1 − 1/r^(γ−1)`. **Só a razão de compressão manda** — e ela é limitada pela detonação, ou seja, pela octanagem do combustível (ver `FUELS.md`).
- **Diesel** (compressão): calor a **pressão constante**. Razão de compressão maior (16–22:1) → mais rendimento. Sem limite de detonação, mas com limite de NOx e pressão máxima.
- **Atkinson / Miller:** expansão maior que a compressão (via comando de válvulas). Mais rendimento, menos densidade de potência — daí serem os ciclos dos híbridos.
- **Brayton:** compressor → câmara → turbina. Turbinas a gás e aeronáuticas. Fluxo contínuo, alta densidade de potência, ruim em carga parcial.

### Potência a vapor e combinado

- **Rankine:** bomba → caldeira → turbina → condensador. Base de **carvão, nuclear, biomassa, solar térmica**. Truques que empurram o rendimento: **superaquecimento**, **reaquecimento** (reheat) e **regeneração** (pré-aquecer a água com vapor sangrado).
- **Ciclo combinado (CCGT):** Brayton no topo (gases a ~600 °C na saída) alimenta um Rankine embaixo. **É a máquina térmica mais eficiente já construída — 60–64%.** O princípio é puramente exergético: aproveitar o rejeito ainda quente em vez de jogar fora.
- **Cogeração/CHP e trigeração:** eletricidade + calor de processo (+ frio via chiller de absorção).

### Refrigeração

- **Compressão de vapor:** compressor → condensador → **dispositivo de expansão** → evaporador. O truque é a mudança de fase: o fluido absorve **calor latente** ao evaporar (muito mais energia por kg do que calor sensível).
- **Absorção:** troca o compressor por uma bomba de calor química (brometo de lítio/água, amônia/água). **Movida a calor**, não a eletricidade → serve onde há calor residual barato ou gás.
- **Ciclo reverso de Brayton:** ar como fluido — aviação e criogenia.
- **Peltier (termoelétrico):** estado sólido, sem fluido nem partes móveis, **COP < 1**. Só vale onde silêncio, tamanho ou precisão importam mais que eficiência.

### Motores de calor externo

- **Stirling:** teoricamente atinge o rendimento de Carnot, aceita qualquer fonte de calor, é silencioso. Perde em densidade de potência e custo — nicho eterno (ver `MOTORS.md`).

## Propriedades térmicas

- **Calor sensível:** `Q = m·c·ΔT`. Muda temperatura.
- **Calor latente:** `Q = m·L`. Muda fase **sem mudar temperatura**.

| Substância | c (kJ/kg·K) | Latente (kJ/kg) |
|---|---|---|
| Água (líquida) | 4,18 | fusão 334 / **vaporização 2.257** |
| Ar (cₚ) | 1,005 | — |
| Alumínio | 0,90 | — |
| Aço | 0,49 | — |
| Cobre | 0,39 | — |
| Óleo mineral | ~1,9 | — |

O número que domina a engenharia térmica está em negrito: **vaporizar 1 kg de água custa o mesmo que aquecer 1 kg de água em 540 °C** (se ela não fervesse). Daí virem do calor latente: refrigeração, torre de resfriamento, panela de pressão, suor, e o poder de queimadura do vapor.

- **Diagramas T-s e P-v:** em T-s, **área sob a curva = calor**; a área fechada do ciclo = trabalho líquido. É a leitura visual de qualquer ciclo.
- **Psicrometria:** ar úmido (bulbo seco/úmido, umidade relativa, ponto de orvalho, entalpia). É a base de HVAC, de por que ar-condicionado desumidifica, e de por que climatizador evaporativo funciona no seco e falha no litoral.

## Transferência de calor

Três mecanismos, quase sempre atuando juntos.

### Condução

`q = −k·A·(dT/dx)` (Fourier). Analogia elétrica: `ΔT = q·R_t`, com `R_t = L/(k·A)`. **Resistências em série somam** — exatamente como circuito.

| Material | k (W/m·K) |
|---|---|
| Cobre | ~400 |
| Alumínio | ~205 |
| Aço | ~50 |
| Vidro | ~1,0 |
| Água | 0,6 |
| Madeira | ~0,15 |
| EPS / lã de vidro | ~0,035 |
| Ar parado | 0,026 |
| Aerogel | ~0,015 |

Nenhum isolante comum vence o **ar parado** — todos eles são, na prática, formas de **impedir o ar de circular**. É por isso que isolante molhado ou comprimido perde quase toda a função.

**R-value** (usado em construção) é `L/k`, resistência por unidade de área. Somar camadas = somar R.

### Convecção

`q = h·A·(T_sup − T_∞)`. O `h` não é propriedade do material — **depende do escoamento**, e varia por ordens de grandeza:

| Situação | h (W/m²·K) |
|---|---|
| Ar, convecção natural | 5–25 |
| Ar forçado (ventilador) | 10–200 |
| Água forçada | 300–15.000 |
| Ebulição / condensação | 2.500–100.000 |

É a tabela que explica: um ventilador transforma um dissipador; refrigeração líquida vence ar por uma ordem de grandeza; e mudança de fase (heat pipe, câmara de vapor) vence tudo.

### Radiação

`q = ε·σ·A·(T⁴ − T_viz⁴)`, com σ = 5,67×10⁻⁸ W/m²·K⁴.

- Depende de **T⁴** → irrelevante perto da temperatura ambiente, dominante em altas temperaturas (fornos, chamas, reentrada, espaço).
- **Emissividade ε:** metal polido ~0,05 (péssimo emissor, ótimo refletor — princípio da garrafa térmica e da manta térmica); tinta fosca, tijolo, pele ~0,9.
- **No vácuo é o único mecanismo.** Satélite e spacecraft rejeitam calor só por radiação — o problema térmico do espaço é quase sempre **como se livrar do calor**, não como se aquecer.

### Trocadores de calor

- Configurações: **paralelo**, **contracorrente** (mais eficaz — permite maior ΔT médio), **cruzado**.
- Métodos: **LMTD** (média log das diferenças) quando se conhecem as temperaturas; **ε-NTU** quando não.
- **Incrustação (fouling)** é o inimigo operacional: uma fina camada de depósito derruba U e é a causa mais comum de queda de desempenho em campo.

### Aplicações onde essas três aparecem juntas

- **Refrigeração de CPU/GPU:** condução (die → IHS → base do cooler, com **TIM** para eliminar o ar das microrrugosidades) → condução/mudança de fase (heat pipe) → convecção forçada (aletas + ventilador). O gargalo quase sempre é o TIM e o `h` do ar.
- **Isolamento residencial:** conduzir menos (R alto), cortar convecção (vedação de frestas), refletir radiação (manta sob telha). Telhado é o maior ganho no Brasil.
- **Data center:** ver PUE em `ENERGY.md`; a física é a mesma — contenção de corredor quente/frio existe para evitar mistura, que destrói exergia térmica.

## Combustão e termoquímica

- **PCS × PCI:** o **superior (PCS/HHV)** conta o calor latente da água formada; o **inferior (PCI/LHV)** não. Motores e turbinas usam PCI (a água sai como vapor). Caldeira de **condensação** recupera parte desse latente — por isso aparece com "eficiência de 108%": está medindo sobre o PCI. Não há mágica, só base de cálculo.
- **Estequiometria (AFR):** massa de ar por massa de combustível para queima completa.

| Combustível | AFR estequiométrico |
|---|---|
| Gasolina | 14,7:1 |
| Etanol (E100) | ~9:1 |
| Diesel | ~14,5:1 |
| Metano (GNV) | ~17,2:1 |
| Hidrogênio | ~34:1 |

O AFR do etanol explica por que motor flex injeta **muito mais** combustível no álcool — e por que os bicos e a bomba precisam ter vazão maior.

- **λ (lambda)** = AFR real / AFR estequiométrico. λ=1 é estequiométrico (o que a sonda busca, porque o catalisador de três vias só funciona ali); λ<1 rico (mais potência, mais consumo, mais CO); λ>1 pobre (economia, mais NOx e risco térmico).
- **Temperatura adiabática de chama:** metano no ar ~1.950 °C; no oxigênio puro ~2.800 °C. É o teto teórico ignorando perdas — e o que define material de câmara e formação de NOx (que dispara acima de ~1.600 °C).
- **Entalpia de formação e lei de Hess:** o calor de reação é diferença de entalpias de formação, independente do caminho. É como se calcula PCI de tabela.

## Aplicações e comparações práticas

### Aquecer água em casa

| Método | Eficiência 1ª lei | Custo relativo de energia |
|---|---|---|
| Chuveiro elétrico | ~100% | **alto** (destrói exergia) |
| Boiler a gás | 80–85% | médio |
| Boiler a gás de condensação | 90–98% | médio-baixo |
| Solar térmico + apoio | — | muito baixo (mas CAPEX) |
| **Bomba de calor (heat pump boiler)** | COP 3–4 | **o mais baixo em eletricidade** |

### Onde o calor manda a conta

- **Motor a combustão:** ~30% vira trabalho, ~30% sai no escape, ~30% no líquido de arrefecimento, ~10% em atrito/radiação. Turbo e turbocompound existem para reciclar parte do escape; ciclo Rankine de rejeito (ORC) aparece em caminhão e navio.
- **Eletrônica:** 100% da potência elétrica de um chip vira calor (ele não realiza trabalho mecânico nem armazena energia). Por isso `TDP` é, literalmente, o problema térmico inteiro.
- **Iluminação:** trocar incandescente (5%) por LED (35%) reduz a conta **e** a carga térmica do ar-condicionado — que, com COP 3, economiza de novo. Ganhos térmicos se compõem.

## Mitos e erros comuns

- **"A energia se perdeu."** Não. Ela se **degradou** em calor ambiente. Quantidade conservada, exergia destruída.
- **"Eficiência acima de 100% é impossível."** Para conversão, sim. Para **bomba de calor**, COP 4 significa mover 4x — não criar. E caldeira de condensação "acima de 100%" é escolha de base (PCI).
- **"Deixar a geladeira aberta esfria a cozinha."** Aquece. O sistema despeja no ambiente o calor retirado **mais** o trabalho do compressor.
- **"Ventilador esfria o ambiente."** Não muda a temperatura do ar (aliás, o motor aquece um pouco). Esfria **você**, aumentando `h` na pele e evaporando suor.
- **"Água quente congela mais rápido"** (efeito Mpemba). Reprodutibilidade questionável e mecanismo disputado — não construa nada em cima.
- **"Água a 100 °C ferve na hora."** Depende da **pressão**: em La Paz (~3.600 m) ferve a ~88 °C — e por isso feijão não cozinha em altitude sem pressão.
- **"Metal está mais frio que madeira."** Estão à mesma temperatura. O metal apenas **conduz** calor da sua mão muito mais rápido — você mede fluxo, não temperatura.
- **Moto-perpétuo / "energia livre / ponto zero"** com investidor por perto: viola 1ª ou 2ª lei. Sem exceção conhecida.
- **Confundir potência (kW) com energia (kWh)** em conta e em especificação. Um painel "de 500 W" não produz 500 W — produz isso em condição de teste, e o que interessa é o kWh do dia.

## Regras práticas

- **Nunca use eletricidade para gerar calor de baixa temperatura** se houver alternativa (bomba de calor, solar, gás). É a maior destruição de exergia do cotidiano.
- **Case a temperatura da fonte com a da necessidade.** Calor de 1.000 °C para secar roupa é desperdício termodinâmico, ainda que barato.
- **Aumente T_quente, reduza T_frio** — é o único caminho para rendimento em máquina térmica.
- **Isolamento é o kWh mais barato:** o calor que você não deixa passar não precisa ser gerado nem removido.
- **Ache o pior elo da cadeia antes de otimizar qualquer coisa.** Multiplicação de rendimentos é impiedosa com quem otimiza o elo errado.
- **Todo watt elétrico dissipado num ambiente climatizado é pago duas vezes**: uma na tomada, outra no ar-condicionado que o remove (dividido pelo COP).
