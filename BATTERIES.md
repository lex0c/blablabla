# Baterias — Carga, Corrente e Dimensionamento

Nota prática de **uso**: como medir, carregar, dimensionar, ligar em série/paralelo, proteger e não estragar. A química em nível de grid/EV está em `ENERGY.md`; teoria de circuito em `CIRCUIT.md`.

## Grandezas

| Grandeza | Unidade | O que é |
|---|---|---|
| **Tensão (V)** | volt | "Pressão" elétrica. Definida pela química da célula × número de células em série. |
| **Corrente (A)** | ampère | Fluxo de carga. Definida pela **carga conectada**, não pela bateria — a bateria só limita o máximo. |
| **Capacidade (Ah)** | ampère-hora | Quanta corrente por quanto tempo. **Só comparável na mesma tensão.** |
| **Energia (Wh)** | watt-hora | `Wh = Ah × V`. **A única métrica que compara baterias diferentes.** |
| **C-rate** | 1/h | Corrente em múltiplos da capacidade. 100 Ah a 0,5C = 50 A. |
| **SoC** | % | State of Charge — quanto ainda tem. |
| **DoD** | % | Depth of Discharge — quanto já foi tirado. `DoD = 100 − SoC`. |
| **SoH** | % | State of Health — capacidade atual / capacidade de fábrica. |
| **Resistência interna (mΩ)** | ohm | Determina quanta corrente entrega e quanto calor gera. **Sobe com o envelhecimento — é o melhor indicador de morte iminente.** |
| **CCA** | ampère | Cold Cranking Amps — corrente de partida a −18 °C por 30 s. Só relevante em bateria de partida. |
| **RC** | minutos | Reserve Capacity — quanto tempo entrega 25 A. Métrica de bateria automotiva. |

### Armadilhas de leitura

- **"100 Ah" não diz nada sem a tensão.** 100 Ah @ 12 V = 1,2 kWh; 100 Ah @ 48 V = 4,8 kWh.
- **Ah depende da taxa de descarga** (efeito **Peukert**, forte em chumbo, quase nulo em lítio). Chumbo é especificado em **C20** (descarga em 20 h). A mesma bateria de 100 Ah C20 entrega talvez **60 Ah** se puxada em 2 h. Lítio entrega os 100 Ah praticamente em qualquer taxa.
- **Capacidade útil ≠ nominal.** Chumbo aceita ~50% de DoD sem morrer cedo; LiFePO₄ aceita 80–90%. Uma LFP de 100 Ah entrega **mais energia útil** que uma chumbo de 200 Ah, pesando 1/3.
- **Power bank em "mAh"** é medido na célula (3,7 V), não na saída USB (5 V). 10.000 mAh ≈ 37 Wh ≈ **~7.000 mAh reais a 5 V**, menos as perdas.

## Química — escolha prática

| Tipo | V/célula | Wh/kg | Ciclos (80% DoD) | DoD útil | Custo | Onde usar |
|---|---|---|---|---|---|---|
| Chumbo inundada (flooded) | 2,0 | 30–40 | 300–800 | 50% | $ | Partida, off-grid barato |
| Chumbo AGM | 2,0 | 30–45 | 500–1200 | 50% | $$ | UPS, náutica, start-stop |
| Chumbo GEL | 2,0 | 30–40 | 800–1500 | 50% | $$ | Ciclo profundo lento, solar |
| **LiFePO₄ (LFP)** | 3,2 | 90–160 | **3000–6000** | 80–90% | $$$ | Off-grid, náutica, motorhome |
| Li-ion NMC/NCA | 3,6–3,7 | 150–270 | 800–2000 | 80% | $$$ | EV, ferramenta, notebook |
| LiPo (RC/drone) | 3,7 | 150–200 | 200–500 | 80% | $$ | Alta descarga, curta vida |
| LTO | 2,3 | 60–80 | >10.000 | 100% | $$$$ | Carga ultrarrápida, frio extremo |
| NiMH | 1,2 | 60–100 | 500–1000 | — | $$ | AA/AAA, híbrido antigo |
| Alcalina (primária) | 1,5 | ~100 | — | — | $ | Baixo dreno, não recarregável |

### Regras de escolha

- **Ciclagem diária (solar, motorhome, náutica):** LiFePO₄. O custo por ciclo é menor que chumbo mesmo custando 3x na compra.
- **Partida de motor:** chumbo ainda ganha — corrente absurda por 3 s, barato, tolera frio, e o alternador foi projetado para o perfil dele.
- **Standby que fica flutuando anos (UPS, alarme):** AGM ou LFP. Chumbo em float constante ainda é o padrão de mercado por custo.
- **Frio (abaixo de 0 °C):** lítio **não pode ser carregado** abaixo de 0 °C sem aquecimento (ver *lithium plating*). Chumbo carrega no frio, mas com capacidade reduzida.
- **Peso/volume críticos:** lítio, sem discussão.

## Carga

Carregar é impor **corrente** até chegar a uma **tensão-alvo**, e então controlar. O perfil correto depende da química — usar o carregador errado é a principal causa de morte precoce.

### Chumbo — perfil de 3 estágios

1. **Bulk (corrente constante):** carregador entrega a corrente máxima (tipicamente 0,1–0,3C) e a tensão sobe sozinha. Vai até ~80% do SoC.
2. **Absorção (tensão constante):** segura a tensão de absorção e a corrente cai naturalmente. **Este estágio é o que enche os últimos 20% e não pode ser encurtado** — chumbo carregado só até o bulk sulfata.
3. **Flutuação (float):** tensão baixa que só compensa a autodescarga. Pode ficar assim indefinidamente.

Tensões típicas para banco **12 V a 25 °C**:

| Tipo | Absorção | Flutuação | Equalização |
|---|---|---|---|
| Flooded | 14,4–14,8 V | 13,5–13,8 V | 15,0–15,5 V (periódica) |
| AGM | 14,4–14,7 V | 13,5–13,8 V | geralmente **não** |
| GEL | 14,1–14,4 V | 13,5–13,8 V | **nunca** |

- **Compensação de temperatura:** −3 a −5 mV/°C **por célula** (−18 a −30 mV/°C num banco 12 V). Quente exige tensão menor; sem compensação, bateria em 40 °C entra em *overcharge* e ferve o eletrólito.
- **Equalização:** sobrecarga controlada e proposital para dissolver sulfato e corrigir estratificação. **Só em flooded**, com ventilação, sem cargas sensíveis conectadas. Em AGM/GEL, isso resseca e mata.

### Lítio — CC/CV, e nada de float

1. **CC:** corrente constante (0,5C típico, até 1C em LFP) até atingir a tensão de célula.
2. **CV:** segura a tensão e espera a corrente cair até **C/20–C/10** → carga completa.

| Química | V/célula máx | Pack 12 V | Corte de descarga |
|---|---|---|---|
| LiFePO₄ | 3,65 | **14,4–14,6 V** | 2,5 V/célula (10,0 V) |
| Li-ion NMC | 4,20 | (13,2 V nominal em 4S) | 3,0 V/célula |
| LiPo | 4,20 | — | 3,3 V/célula (na prática) |

Diferenças que importam:

- **Lítio não usa float.** Depois de cheia, o correto é **parar**. Manter 100% permanentemente acelera o envelhecimento calendárico. Em sistemas solares, configura-se float baixo (13,6 V para LFP) ou "rebulk" por SoC.
- **Não carregar abaixo de 0 °C.** Nessa faixa o lítio não se intercala no ânodo e **deposita metal (lithium plating)** — perda permanente de capacidade e formação de dendritos que podem furar o separador. Descarregar no frio é aceitável (com menos potência). Packs bons têm **aquecedor + corte por temperatura no BMS**.
- **Curva de tensão é plana.** Uma LFP fica em ~13,2–13,3 V entre 20% e 80% de SoC. **É impossível estimar SoC por voltímetro** — precisa de **monitor com shunt** (coulomb counting).
- **Não existe efeito memória.** Ciclos parciais são preferíveis a ciclos profundos.

### BMS — o que ele é e o que ele não é

O **BMS (Battery Management System)** faz: corte por sobretensão/subtensão de célula, corte por sobrecorrente e curto, corte por temperatura, e **balanceamento**.

O que ele **não** faz: substituir o carregador. O BMS é uma **rede de proteção** — se ele está desarmando na rotina, o sistema está mal configurado. Carregador e inversor devem ter limites *dentro* dos do BMS, de modo que o BMS nunca precise atuar.

- **Balanceamento passivo:** queima o excesso da célula mais alta em resistor. Corrente pequena (30–200 mA), lento, barato — o padrão.
- **Balanceamento ativo:** transfere carga da célula alta para a baixa. Caro, útil em packs grandes ou desequilibrados.
- Balanceamento só acontece **no topo da carga**. Um banco que nunca chega a 100% **nunca balanceia** — motivo comum de packs LFP com deriva de células.

### Fontes de carga

- **Carregador de bancada inteligente:** detecta química e faz o perfil correto. Sempre a melhor opção.
- **Alternador automotivo:** entrega 13,8–14,4 V fixos, sem estágio de absorção adequado. **Não enche chumbo além de ~80%** em viagens curtas, e é ruim/perigoso para LFP (que aceitaria corrente altíssima e pode sobrecarregar o alternador até queimá-lo). Solução correta em motorhome/náutica: **conversor DC-DC (B2B charger)**, que isola e impõe o perfil certo.
- **Solar PWM:** basicamente uma chave — força o painel a operar na tensão da bateria. Só faz sentido com painéis de tensão casada (36 células para 12 V). Barato e ineficiente.
- **Solar MPPT:** rastreia o ponto de máxima potência do painel e converte para a tensão da bateria. **10–30% mais colheita**, e permite usar painéis de 60/72 células (muito mais baratos por watt) em bancos de 12/24/48 V. Vale quase sempre.
- **USB-PD / PPS:** negocia tensão (5/9/12/15/20 V, PPS em passos de 20 mV) com o dispositivo. O carregamento rápido de celular é o **dispositivo** decidindo o perfil, não o carregador. Ver `ENERGY.md` para carga de EV.

### Carga rápida — o custo

Carga rápida gera calor e estresse mecânico nos eletrodos. Regras práticas de longevidade em lítio:

- Manter entre **20% e 80%** para uso diário; 100% só quando for realmente usar.
- **Calor é o inimigo principal**: envelhecimento calendárico dobra a cada ~10 °C. Bateria a 100% de SoC e 40 °C degrada rapidíssimo.
- Guardar por meses: **40–60% de SoC**, lugar fresco. Guardar cheio ou vazio destrói.

## Série, paralelo e mistura

- **Série:** tensões somam, Ah permanece. 4× LFP 3,2 V/100 Ah = 12,8 V/100 Ah. **A célula mais fraca limita todo o conjunto** e é a que sofre sobretensão/subtensão — daí a necessidade de balanceamento.
- **Paralelo:** Ah somam, tensão permanece. Correntes de circulação aparecem se as baterias tiverem SoC ou resistência interna diferentes.

Regras de mistura (valem para chumbo, e são igualmente severas para lítio):

- **Nunca misturar químicas diferentes** no mesmo banco.
- **Nunca misturar idades ou capacidades diferentes.** A velha puxa a nova para baixo — o banco todo passa a valer o pior elemento.
- Ao montar em paralelo, **equalizar o SoC antes** (carregar cada uma individualmente) e usar **cabos de mesmo comprimento e bitola**, com o positivo saindo de uma ponta e o negativo da outra (*diagonal wiring*), para que todas vejam a mesma resistência.
- Preferir **uma bateria maior a várias pequenas em paralelo**, e **subir a tensão** em vez de somar Ah: 48 V para a mesma potência puxa **1/4 da corrente** de 12 V → cabo 4x mais fino, perdas 16x menores.

## Corrente, cabos e proteção

Aqui é onde bateria vira incêndio. Uma bateria automotiva de chumbo entrega **milhares de ampères em curto** — chave de fenda encostando nos dois polos vira solda incandescente em um segundo. Anel de dedo entre polo e chassi causa amputação.

### Dimensionamento de cabo

Duas restrições, e vale a **mais exigente**:

1. **Capacidade de corrente (ampacidade)** — para não superaquecer o cabo.
2. **Queda de tensão** — normalmente a restrição dominante em DC de baixa tensão. Alvo: **≤3%** para circuitos de carga/inversor.

`V_queda = 2 × comprimento(m) × corrente(A) × ρ / área(mm²)` — o **2×** é a ida e a volta, esquecido com frequência. Para cobre, ρ ≈ 0,0175 Ω·mm²/m.

Exemplo: inversor de 1000 W em 12 V puxa ~90 A. Com 3 m de cabo (ida) e alvo de 3% (0,36 V):
`A = 2 × 3 × 90 × 0,0175 / 0,36 ≈ 26 mm²`. Cabo de 25–35 mm². Em **24 V** a mesma potência puxa 45 A → ~13 mm². Metade da corrente, um quarto da perda.

### Fusíveis e disjuntores

- **Fusível o mais perto possível do polo positivo da bateria** — o trecho antes do fusível não tem proteção alguma. Idealmente ≤ 18 cm.
- **Dimensionar o fusível para proteger o cabo**, não o equipamento: o fusível queima antes de o cabo virar resistência de aquecimento.
- **Capacidade de interrupção (AIC)** importa: um fusível comum não interrompe os >10 kA de curto de um banco LFP. Use **fusível classe T ou ANL** em bancos de lítio.
- **Nunca usar disjuntor AC em circuito DC.** Em AC a corrente cruza o zero 120x por segundo e o arco se apaga sozinho; em DC não cruza, e o arco **sustenta** dentro do disjuntor até derreter tudo. Use disjuntor com **especificação DC explícita** e respeite a polaridade marcada.
- Fusão do **negativo** também, em sistemas isolados de chassi.

## Medição e diagnóstico

### Tensão em repouso — chumbo 12 V (25 °C, ≥4 h sem carga/descarga)

| Tensão | SoC |
|---|---|
| 12,70 V+ | 100% |
| 12,50 V | 75% |
| 12,25 V | 50% |
| 12,05 V | 25% |
| ≤11,90 V | 0% (descarregada) |

**A leitura só vale em repouso.** Com carregador ligado ou carga puxando, a tensão mente. Em **LFP a tabela não existe na prática** — a curva é plana demais.

### Ferramentas

- **Multímetro:** tensão em repouso e sob carga. Só isso já pega 70% dos casos.
- **Densímetro (hidrômetro):** só flooded. 1,265 = cheia, 1,190 = 50%, 1,120 = vazia. **Diferença >0,050 entre células = célula morta.** É o teste mais honesto para chumbo aberto.
- **Testador de condutância/IR:** mede resistência interna e estima CCA/SoH em segundos. É o que a loja usa. IR subindo é sentença de morte.
- **Teste de capacidade real:** descarregar com carga conhecida até o corte, cronometrando. É o único teste que dá a capacidade verdadeira — demora horas e é o padrão-ouro.
- **Monitor com shunt** (BMV-712, Smart Shunt e similares): integra corrente ao longo do tempo (**coulomb counting**). Único jeito decente de saber SoC em lítio. Precisa de **sincronização periódica em 100%**, senão a leitura deriva.
- **Alicate amperímetro DC (efeito Hall)** — o comum de AC não mede DC. Essencial para achar dreno parasita.

### Sintomas → causa

| Sintoma | Causa provável |
|---|---|
| Tensão cheia em repouso, despenca sob carga | Resistência interna alta / placas sulfatadas → bateria acabada |
| Chumbo não passa de ~12,4 V mesmo carregando | Sulfatação ou célula em curto |
| Uma célula com densidade muito menor | Célula em curto — banco condenado |
| Aquece muito ao carregar | Sobrecarga, célula em curto ou tensão sem compensação térmica |
| Carcaça estufada | Sobrecarga/superaquecimento (chumbo) ou geração de gás (lítio) — **descartar, não usar** |
| LFP desarma o BMS perto do fim da carga | Células desbalanceadas; carregar lento até 100% e deixar balancear |
| Bateria de carro morre em dias | Dreno parasita (>50 mA parado) ou alternador não recarregando |
| Autonomia caiu pela metade em 2 anos | Ciclagem profunda demais ou calor |

## Dimensionamento de banco

Roteiro:

1. **Levantar consumo diário em Wh.** Some `potência × horas` de cada carga. Não confie em placa — meça.
2. **Somar perdas.** Inversor 85–92%, fiação 2–3%, carga/descarga da bateria (chumbo ~80% round-trip, lítio ~95%). Fator prático: **÷0,80** para chumbo, **÷0,90** para lítio.
3. **Definir dias de autonomia** (sem recarga): 1–2 para uso com gerador, 2–3 para solar em local ensolarado, 3–5 onde chove muito.
4. **Aplicar o DoD máximo:** 50% em chumbo, 80% em LFP.
5. **Corrigir por temperatura:** chumbo perde ~20% de capacidade a 0 °C, ~40% a −18 °C.

**Exemplo:** 2.000 Wh/dia, 2 dias de autonomia.
- Chumbo: 2000 ÷ 0,80 × 2 ÷ 0,50 = **10.000 Wh** → ~**833 Ah @ 12 V** (~250 kg).
- LFP: 2000 ÷ 0,90 × 2 ÷ 0,80 = **5.560 Wh** → ~**116 Ah @ 48 V** (~50 kg).

A diferença de 5x em massa e volume é a razão de LFP ter tomado conta de náutica, motorhome e off-grid, apesar do preço de etiqueta.

### Corrente de carga adequada

- **Chumbo:** 0,1–0,2C (idealmente C/10). Acima disso aquece e gaseifica. Banco de 200 Ah → carregador de 20–40 A.
- **LFP:** 0,2–0,5C confortável, aceita 1C. Banco de 100 Ah → 20–50 A tranquilamente.
- Carregador **subdimensionado** em chumbo é pior do que parece: nunca completa a absorção → sulfatação crônica.

## Modos de falha e envelhecimento

- **Sulfatação (chumbo):** sulfato de chumbo cristaliza nas placas quando fica descarregada. **Deixar chumbo descarregado por semanas é sentença de morte.** "Dessulfatadores" por pulso recuperam casos leves e recentes; não ressuscitam bateria dura.
- **Estratificação (flooded):** ácido denso desce, água fica em cima → a parte de baixo corrói, a de cima não trabalha. Corrigido por **equalização**.
- **Corrosão de grade:** envelhecimento natural do chumbo, acelerado por calor e sobrecarga.
- **Perda d'água:** flooded precisa de **água destilada** (nunca de torneira — minerais envenenam as placas). Só completar **depois** de carregar, e só até cobrir as placas.
- **Crescimento de SEI (lítio):** camada que consome lítio ativo. É o envelhecimento calendárico — acontece **mesmo parada**, acelerado por SoC alto e calor.
- **Lithium plating:** carga no frio ou rápida demais deposita lítio metálico. Irreversível e perigoso.
- **Thermal runaway:** acima de ~150 °C a célula Li-ion entra em reação autossustentada, liberando oxigênio e gases inflamáveis/tóxicos. **LFP é muito mais resistente que NMC/NCA.** Ver `ENERGY.md`.
- **Efeito memória:** real em **NiCd**, praticamente inexistente em NiMH moderno e **inexistente em lítio**. Não "cicle" um lítio de propósito.
- **Autodescarga por mês:** chumbo 3–5% (mais em calor), NiMH comum 15–20%, NiMH LSD tipo Eneloop ~1,5%, lítio 2–3%.
- **Vazamento de alcalina:** primária descarregada vaza KOH e destrói o compartimento. **Tirar as pilhas de equipamento guardado.**

## Segurança

- **Chumbo em carga libera hidrogênio.** Faixa explosiva a partir de 4% no ar. Carregar em local ventilado; nunca desconectar garra com o carregador ligado (a faísca acontece exatamente onde está o gás).
- **Ácido sulfúrico:** óculos e luvas. Neutralizar respingo com **bicarbonato**; na pele/olhos, água corrente em volume por 15 min.
- **Ordem de conexão em jump start:** positivo da descarregada → positivo da boa → negativo da boa → **negativo no chassi/bloco da descarregada, longe da bateria** (o último ponto é onde a faísca ocorre; mantenha-a longe do hidrogênio). Desconectar na ordem inversa.
- **Curto de banco:** tirar relógio, anel e pulseira antes de trabalhar. Usar chave com cabo isolado. Cobrir o polo oposto ao que está sendo manipulado.
- **Lítio danificado:** célula perfurada, amassada ou estufada é lixo — não teste, não carregue. Guardar/transportar LiPo em **saco resistente a fogo**.
- **Incêndio de Li-ion:** não é fogo de metal alcalino; o combate padrão é **muita água para resfriar** (a reação se autoalimenta enquanto houver calor). Extintor comum apaga a chama mas não impede a reignição. Bateria grande em chamas é caso de evacuar e chamar o corpo de bombeiros.
- **Transporte aéreo:** ≤100 Wh livre na bagagem de mão, 100–160 Wh com autorização da companhia, >160 Wh proibido. **Sempre na bagagem de mão**, com terminais fitados. Baterias de lítio não podem ir despachadas.
- **Descarte:** chumbo é o produto mais reciclado do mundo (>95%) e tem logística reversa obrigatória — devolva ao vendedor. Lítio vai a ponto de coleta; **nunca no lixo comum** (compactador de caminhão fura a célula e incendeia).

## Erros comuns

- Deixar chumbo descarregado "para resolver depois" — sulfata e não volta.
- Usar carregador de chumbo em LiFePO₄ (a equalização a 15,5 V destrói o pack) ou perfil de lítio em chumbo (nunca completa a absorção).
- Ligar bateria nova em paralelo com velha para "aumentar a capacidade".
- Confiar em voltímetro para SoC de LFP.
- Carregar lítio abaixo de 0 °C sem aquecedor.
- Dimensionar cabo pela ampacidade e ignorar a queda de tensão em 12 V.
- Usar disjuntor AC em circuito DC.
- Manter notebook/celular a 100% e quente o tempo todo, e depois culpar "a bateria de hoje em dia".
- Somar Ah em 12 V em vez de subir para 24/48 V — cabo grosso, perda alta, dinheiro jogado em cobre.
- Confiar no BMS como controle operacional em vez de proteção de última linha.
