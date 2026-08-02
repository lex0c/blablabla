# Combustíveis — Motores e Geradores

Nota prática sobre **o que se coloca no tanque**: propriedades, escolha, armazenamento, contaminação e consumo. Os motores em si estão em `MOTORS.md`; geração/rede em `ENERGY.md`.

## Fundamentos

### O que define um combustível

- **Poder calorífico inferior (PCI / LHV):** energia liberada na queima, descontando o calor que vai embora como vapor d'água. É o número que importa em motor. Medido em **MJ/kg** (massa) ou **MJ/L** (volume). **Motor consome volume, então MJ/L é o que manda no consumo.**
- **Densidade (kg/L):** converte um no outro. Diesel é denso (0,84), etanol leve (0,81) mas pobre em energia por kg.
- **Octanagem (RON/MON/IAD):** resistência à **detonação** (auto-ignição espontânea). Vale para **ciclo Otto** (gasolina, etanol, GNV). Octanagem alta não é "mais força" — é **permissão para mais compressão/avanço de ponto**.
- **Número de cetano (NC):** o oposto — **facilidade de auto-ignição**. Vale para **ciclo Diesel**. Cetano alto = ignição pronta, partida fácil, menos batida de diesel.
- **Ponto de fulgor (flash point):** menor temperatura em que os vapores pegam fogo com uma faísca. Define o risco de manuseio.
- **Volatilidade / pressão de vapor (RVP):** facilidade de evaporar. Alta ajuda partida a frio, atrapalha em calor (**vapor lock**) e evapora no tanque.
- **Lubricidade:** o combustível lubrifica a bomba e os bicos (crítico em diesel, que não tem óleo separado no sistema de injeção).
- **Estabilidade oxidativa:** quanto tempo aguenta guardado antes de virar goma/borra.

### Tabela de energia (valores típicos)

| Combustível | Densidade (kg/L) | PCI (MJ/kg) | PCI (MJ/L) | kWh/L | Octanagem / Cetano |
|---|---|---|---|---|---|
| Gasolina pura | 0,74 | ~43,5 | ~32,2 | ~8,9 | RON 95+ |
| Gasolina comum BR (E27) | 0,75 | ~40 | ~30 | ~8,3 | RON ~92, IAD 87 |
| Gasolina premium BR | 0,75 | ~40 | ~30 | ~8,3 | RON 95–98 |
| Etanol hidratado (E100) | 0,81 | ~26,4 | ~21,3 | ~5,9 | RON ~109 |
| Diesel S10 (com biodiesel) | 0,84 | ~42,5 | ~35,5 | ~9,9 | Cetano ≥ 48 |
| Biodiesel (B100) | 0,88 | ~37,5 | ~33 | ~9,2 | Cetano 50–60 |
| GLP (propano/butano) | 0,54 | ~46 | ~25 | ~6,9 | RON ~105–110 |
| GNV (metano) | 0,0007 @1 bar / ~0,16 @220 bar | ~50 | ~36 MJ/m³ @1 bar; ~8 MJ/L @220 bar | ~2,2 (@220 bar) | RON ~120+ |
| Querosene / Jet A-1 | 0,80 | ~43 | ~34,5 | ~9,6 | — |
| Hidrogênio (700 bar) | 0,04 | ~120 | ~4,8 | ~1,3 | — |

Leitura da tabela: **etanol tem ~2/3 da energia da gasolina por litro** → consumo ~30–40% maior, e é por isso que existe a regra do preço. **Diesel é o mais denso em energia por litro** entre os líquidos comuns → aliado a motor mais eficiente, explica a autonomia de caminhão e gerador diesel.

### Regras de conversão úteis

- 1 L de gasolina ≈ **8,9 kWh térmicos**. Um motor de 25–30% de eficiência entrega ~**2,5 kWh elétricos** por litro.
- 1 L de diesel ≈ **9,9 kWh térmicos**. Gerador diesel (30–40%) entrega ~**3,3–3,8 kWh/L**.
- **1 kg ≈ 1,2 L de gasolina**, ≈ 1,19 L de diesel (para converter especificação de fabricante em consumo real).

## Gasolina

### Composição e o etanol anidro

No Brasil a gasolina do posto **não é gasolina pura**: leva **etanol anidro** misturado por lei (historicamente 18–27,5%, hoje na faixa de **E27–E30** — a ANP muda o percentual periodicamente, então **confirme o teor vigente**). Consequências:

- Menor energia por litro que a gasolina europeia/americana → comparações de consumo entre países não batem.
- **Higroscopicidade:** o etanol puxa água. Água no tanque + etanol → risco de **separação de fase** (a camada de baixo vira água+etanol, e é ela que o motor vai chupar).
- Ataca borrachas, alumínio e fibra de vidro antigos — problema em carros clássicos, barcos e equipamentos anteriores aos anos 90.

**Teste caseiro do teor de etanol:** proveta com 50 mL de gasolina + 50 mL de água, agita, deixa decantar. O volume de água aumenta porque o etanol migra para ela. Se a água foi de 50 para 77 mL, havia ~27% de etanol.

### Octanagem — quando premium vale

Octanagem é **resistência à detonação (knock)**, não energia. Sob detonação, a mistura auto-ignita antes da hora e a onda de pressão martela pistão e biela.

- Motor **aspirado de baixa compressão** que pede comum: gasolina premium **não dá ganho mensurável**. É dinheiro fora.
- Motor **turbo, alta compressão ou com sensor de detonação ativo**: a ECU trabalha com **avanço de ponto** limitado pelo knock. Com combustível melhor, ela adianta o ponto e **realmente entrega mais torque** e às vezes menos consumo.
- **Aditivada ≠ premium.** "Aditivada" é a comum com pacote detergente. "Premium/podium" tem octanagem maior. São eixos independentes.
- **Octane booster** de frasco: efeito muito pequeno pelo preço. Para ganhar octanagem de verdade em motor flex, o caminho é etanol.

### Detonação vs pré-ignição vs LSPI

- **Detonação (knock):** auto-ignição **depois** da vela, colidindo com a frente de chama. Ruído de "batida de pino". Cronicamente destrói o pistão.
- **Pré-ignição:** algo incandescente (carvão, vela errada) acende a mistura **antes** da vela. Pior que detonação — funde pistão rápido.
- **LSPI (Low-Speed Pre-Ignition):** praga de motores **turbo de injeção direta** em baixa rotação e alta carga. Gotas de óleo/depósito iniciam a queima. Mitigação é **óleo com especificação correta (API SP / ILSAC GF-6)**, não combustível.

## Etanol

- **Hidratado (E100)** é o do posto: ~95% etanol + ~5% água (azeótropo). **Anidro** (~99,6%) é o que se mistura na gasolina.
- **Vantagens:** octanagem altíssima (RON ~109) e **calor latente de vaporização ~3x o da gasolina** — evapora esfriando a mistura, o que aumenta a densidade do ar admitido. Por isso motor preparado/turbo adora etanol.
- **Desvantagens:** ~30% menos energia por litro; higroscópico; corrosivo para ligas e elastômeros antigos; partida a frio ruim (queima mal abaixo de ~15 °C — daí o reservatório de partida a frio com gasolina nos flex antigos, hoje substituído por bicos/velas aquecidos).

### A regra dos 70%

Vale a pena abastecer com etanol quando o **preço do litro de etanol < ~70% do preço da gasolina**.

O 70% é média. O número correto é a **razão real de consumo do seu carro**: se ele faz 10 km/L na gasolina e 7 km/L no etanol, seu ponto de equilíbrio é **70%**. Carros turbo modernos e motores bem calibrados chegam a 72–75% (etanol compensa mais cedo); carros antigos ou muito pesados podem ficar em 65%.

Fora do bolso: etanol roda **mais frio** e mantém válvulas/câmara mais limpas; gasolina é melhor para carro parado por muito tempo (etanol e água guardados dão problema).

## Diesel

### S10 vs S500

O número é **ppm de enxofre**. **S10 = 10 ppm** (obrigatório para veículos rodoviários modernos com pós-tratamento), **S500 = 500 ppm** (uso restrito, frota antiga, algumas aplicações rurais/marítimas).

- Enxofre **envenena catalisador, DPF e SCR**. Colocar S500 em caminhão Euro 6 / Proconve P8 destrói o pós-tratamento.
- Dessulfurizar **reduz a lubricidade natural** do diesel → o S10 leva **aditivo de lubricidade** de fábrica. Diesel "melhorado" caseiro (óleo 2T no tanque) é folclore que às vezes ajuda, às vezes entope bico.

### Biodiesel na mistura (BX)

O diesel brasileiro carrega biodiesel obrigatório — a mistura subiu ao longo dos anos (B10 → B12 → B14 → **B15**, com trajetória política para B20). **Confirme o percentual vigente na ANP.** O que muda na prática:

- **Solvente:** biodiesel dissolve a borra acumulada por anos em tanques e linhas → **entope filtro** logo após a troca de tipo de combustível. Em frota velha, trocar filtro extra na transição.
- **Higroscópico e biodegradável:** segura mais água e é **comida para micróbio**.
- **Estabilidade oxidativa menor:** ~6 meses guardado, contra 12+ do diesel mineral. Ruim para **gerador standby**, que fica meses com o tanque parado.
- Ligeiramente menos energia por litro e maior ponto de névoa (pior no frio).

### Cetano, frio e partida

- **Número de cetano** mínimo no Brasil é 48. Cetano baixo = partida difícil, fumaça branca, ruído de diesel alto.
- **Cloud point / ponto de névoa:** temperatura em que a parafina começa a cristalizar (fica turvo).
- **CFPP (Cold Filter Plugging Point):** temperatura em que essa parafina **entope o filtro** — é o número operacionalmente relevante. Em Sul do Brasil no inverno, ou altitude, usa-se **diesel de inverno** ou **aditivo antigel**, e vale o **filtro aquecido**.
- Aditivo antigel só funciona **antes** de gelificar. Depois de entupido, a solução é calor.

### Água e o "diesel bug"

Água é o inimigo número um do diesel moderno:

- **Common rail** trabalha a 1800–2500 bar com folgas de micrômetros. Água mata a lubricidade local, cavita e **corrói bico e bomba** — conserto caríssimo.
- Na **interface água/diesel** cresce contaminação microbiana (fungo *Hormoconis resinae*, bactérias). Vira uma lama escura que **entope filtro e corrói o tanque por baixo**.
- Defesa: **dreno do filtro separador de água** na rotina (Racor e similares têm copo transparente), tanque cheio para reduzir condensação, **biocida** quando já contaminado (e depois limpeza — biocida mata, mas a biomassa morta ainda entope), e **fuel polishing** (recirculação por filtragem) em tanques de gerador.

## Combustíveis gasosos

### GNV (metano comprimido)

- Armazenado a **200–220 bar** em cilindro pesado. Autonomia por m³ baixa, mas **preço por km costuma ser o menor**.
- **Octanagem altíssima**, queima limpa, quase não contamina o óleo do motor.
- Perde **10–15% de potência** (ocupa volume de admissão e queima mais lento) e exige **sedes de válvula endurecidas** — GNV não lubrifica a válvula como o combustível líquido; motores não preparados afundam válvula.
- No Brasil: **cilindro tem prazo de reteste (a cada 5 anos)** e a conversão precisa de inspeção/CSV. Cilindro vencido é item de segurança, não burocracia — falha de vaso de pressão a 200 bar é catastrófica.
- Ótimo para **gerador estacionário** com rede de gás: sem tanque, sem degradação, sem borra, autonomia infinita enquanto houver rede.

### GLP (propano/butano, "autogas")

- Líquido sob pressão modesta (~8–15 bar) → cilindro leve comparado ao GNV.
- **Uso automotivo é proibido no Brasil** (permitido e comum na Europa como *autogas*). Uso legal: **empilhadeira** (queima limpa o bastante para operar dentro de galpão), geradores, aquecimento.
- Para **gerador**, é o melhor combustível de estoque: **não oxida, não gera borra, não puxa água**. Um botijão guardado por 10 anos ainda funciona — o que não se pode dizer de gasolina.
- Menos energia por litro que gasolina → consumo volumétrico maior.

### Hidrogênio

- Em **ICE** existe (Toyota e outros testaram), mas o destino natural é a **célula a combustível**. Densidade volumétrica ruim mesmo a 700 bar, fragilização de metais, infraestrutura escassa. Ver `ENERGY.md`.

## Combustíveis especiais

- **Motores 2 tempos:** o óleo vai **misturado no combustível** (proporção 25:1 a 50:1 conforme o fabricante — obedecer, não "chutar mais óleo por segurança": excesso carboniza e entope escape). Especificações: **JASO FD** (refrigerado a ar: motosserra, roçadeira) e **TC-W3** (refrigerado a água: motor de popa). Não são intercambiáveis.
- **Gasolina de aviação (AvGas 100LL):** octanagem alta, **com chumbo tetraetila**, sem etanol. Usada como referência e, informalmente, em equipamentos guardados — mas o chumbo destrói catalisador e sonda lambda.
- **Querosene / Jet A-1:** turbinas e turboshaft. Basicamente um destilado próximo do diesel, com controle rígido de ponto de congelamento.
- **HVO (diesel renovável):** hidrotratado, **drop-in real** — quimicamente parecido com diesel mineral, cetano alto (70+), sem os defeitos do biodiesel éster (não é higroscópico, guarda bem). É o substituto tecnicamente superior; limitado por oferta e preço.
- **E-fuels:** sintetizados de CO₂ + H₂ renovável. Neutros no ciclo, caríssimos, nicho (motorsport, aviação, coleção).

## Aditivos — o que faz efeito e o que é marketing

| Aditivo | Faz efeito? | Comentário |
|---|---|---|
| Detergente / limpa-bicos | Sim, moderado | Útil em injeção direta (a válvula não é lavada pelo combustível). Efeito acumulativo, não instantâneo. |
| Estabilizante (fuel stabilizer) | **Sim, muito** | Essencial para equipamento sazonal e gerador standby. |
| Biocida (diesel) | Sim | Só quando há contaminação real. Depois exige limpar a biomassa morta. |
| Antigel / melhorador de fluxo | Sim | Só preventivo, aplicado antes de gelificar. |
| Melhorador de cetano (2-EHN) | Sim, pequeno | Ganha 2–5 pontos de cetano. |
| Aditivo de lubricidade | Sim | Relevante em S10 e em bomba de equipamento antigo. |
| "Octane booster" | Marginal | Ganho de fração de ponto. Etanol é mais barato e eficaz. |
| "Economizador de combustível" | **Não** | Pastilhas, ímãs, catalisadores de tanque: pseudociência. |

## Armazenamento e degradação

### Prazos realistas

| Combustível | Sem tratamento | Com estabilizante | Nota |
|---|---|---|---|
| Gasolina com etanol | 1–3 meses | 6–12 meses | Etanol evapora/absorve água primeiro |
| Gasolina sem etanol | 6 meses | 1–2 anos | Ideal para equipamento sazonal |
| Diesel mineral | 6–12 meses | até 2 anos | Se seco e frio |
| Diesel com biodiesel (BX) | ~6 meses | ~1 ano | Oxida e alimenta micróbio |
| GLP / GNV | **anos** | — | Não degrada |

### Como o combustível "estraga"

- **Perda de voláteis:** as frações leves evaporam → gasolina velha custa a pegar e perde octanagem.
- **Oxidação → goma/verniz:** entope giclê de carburador e bico. É a causa clássica de "gerador que ficou 8 meses parado e não pega".
- **Separação de fase:** água + etanol descem e formam camada. Não dá para "misturar de volta" — tem que drenar.
- **Contaminação microbiana:** só em diesel (precisa de água e carbono).

### Boas práticas

- **Tanque cheio reduz condensação** (menos ar úmido lá dentro) — vale para tanque estacionário e carro parado. Exceção: equipamento pequeno guardado por meses, onde o melhor é **esvaziar tanque e carburador** ou rodar até apagar.
- Recipiente **certificado e opaco**, com respiro, longe de sol e de fonte de ignição. Vermelho = gasolina, amarelo = diesel, azul = querosene (convenção norte-americana, mas amplamente usada).
- **Rotacionar estoque (FIFO)** e datar os galões. Diesel de gerador standby exige **plano de rotação ou polimento**, senão a emergência chega e a bomba entope.
- **Filtro separador de água** com dreno acessível em qualquer instalação estacionária.
- **Aterrar** o recipiente ao transferir combustível: fluxo em bocal gera eletricidade estática, e essa é uma causa real de incêndio em abastecimento. Recipiente plástico **no chão**, nunca na caçamba forrada.

## Segurança

| Combustível | Ponto de fulgor | Risco dominante |
|---|---|---|
| Gasolina | ~ −43 °C | Vapor inflamável **sempre** presente; explosivo em ambiente fechado |
| Etanol | ~ 13 °C | Inflamável; **chama quase invisível à luz do dia** |
| Diesel | 52–96 °C | Não pega fogo fácil como líquido, mas **névoa/spray de alta pressão pega** |
| GLP | gás | Mais **pesado que o ar** — acumula em porão, poço, fosso |
| GNV | gás | Mais **leve que o ar** — dissipa, mas alta pressão no cilindro |

Pontos que matam gente de verdade:

- **Vapor de gasolina é mais pesado que o ar** e escorre pelo chão até achar uma faísca a metros de distância. Garagem fechada com galão vazando é uma bomba.
- **Nunca reabastecer motor quente ou funcionando.** Coletor de escape passa de 300 °C; gasolina derramada nele acende sem precisar de faísca.
- **Monóxido de carbono:** gerador a combustão **só ao ar livre**, longe de janelas. CO é inodoro e mata dormindo — não basta "garagem com a porta aberta". Esta é a causa mais comum de morte associada a gerador portátil.
- **Injeção de alta pressão (diesel common rail):** um vazamento em agulha injeta combustível **sob a pele** — lesão grave que parece um furinho. Nunca procurar vazamento com a mão.
- **Chama de etanol é praticamente invisível** — incêndio de álcool no sol se detecta pelo calor e pela distorção do ar.

## Combustível em geradores

### Escolha por perfil de uso

| Perfil | Combustível recomendado | Por quê |
|---|---|---|
| Portátil, uso ocasional, casa | Gasolina ou **inverter** a gasolina | Barato, leve, disponível |
| Standby residencial automático | **GLP ou gás encanado** | Não degrada, não exige rotação de estoque |
| Standby comercial / datacenter | **Diesel** com tanque e polimento | Densidade energética e autonomia longa |
| Uso contínuo pesado / rural | Diesel | Durabilidade e consumo por kWh |
| Interior de galpão (empilhadeira) | GLP | Emissões toleráveis com ventilação |
| Sem certeza de suprimento | **Dual/tri-fuel** | Gasolina + GLP (+ gás natural) na mesma máquina |

Um gerador a gasolina perde ~5–10% de potência com GLP (menor energia na mistura), mas ganha em partida após meses parado e em vida do carburador.

### Consumo — cálculo prático

Regra de bolso, a **75% de carga** (ponto mais eficiente típico):

- **Diesel:** ≈ **0,25–0,30 L/h por kW gerado**
- **Gasolina:** ≈ **0,35–0,45 L/h por kW gerado**
- **GLP:** ≈ **0,45–0,55 L/h por kW gerado** (líquido)

Exemplo: gerador diesel de 20 kW a 75% de carga (15 kW) → 15 × 0,28 ≈ **4,2 L/h**. Para 8 h de autonomia: **~34 L de tanque útil** (dimensione 20% acima).

Caminho inverso, para saber o que cabe: um tambor de 200 L de diesel sustenta esses 15 kW por **~48 horas**.

### Wet stacking — o erro clássico

Gerador **diesel** operando cronicamente com **carga baixa (<30–40%)** não atinge temperatura de queima completa. O resultado é **diesel não queimado escorrendo pelo escape** ("wet stack"), carbonização de bicos, vitrificação de camisas e perda de compressão permanente.

Defesas: dimensionar o gerador para a carga **real** (não para o pico imaginário), ou fazer **teste periódico com banco de cargas (load bank)** levando a máquina a 70–80% por 1–2 h. Geradores a gasolina e a gás sofrem muito menos com isso.

### Dimensionamento — onde o combustível entra

- **kVA vs kW:** placa em kVA assume fator de potência 0,8 → **20 kVA ≈ 16 kW úteis**. Consumo se calcula sobre kW.
- **Corrente de partida:** motor elétrico puxa 3–7× a corrente nominal ao partir. Um compressor de 3 kW pode exigir 15 kW instantâneos. Superdimensionar por causa disso, porém, joga o gerador na zona de wet stacking — a saída é **partida suave (soft-starter)** ou **inversor**.
- **Qualidade da energia:** eletrônica sensível pede **inverter generator** (THD < 5%) ou pelo menos AVR. Gerador de campo fixo sem regulação entrega onda suja o suficiente para queimar fonte chaveada.
- **Altitude e temperatura:** motor a combustão perde ~**1% de potência a cada 100 m** de altitude e ~2% a cada 5,5 °C acima de 25 °C. Gerador de 10 kW a 1500 m entrega ~8,5 kW.

### Rotina de manutenção ligada a combustível

1. **Dreno do separador de água** — mensal em standby, ou conforme visor.
2. **Troca de filtro de combustível** — no prazo, mais uma troca extra após mudar de fornecedor ou de teor de biodiesel.
3. **Teste com carga (exercise run)** mensal, 30 min sob carga real — mantém bicos limpos, bateria carregada e revela problema antes da emergência.
4. **Análise/rotação do diesel estocado** — a cada 6–12 meses: água, aparência, contaminação microbiana. Polir ou consumir e repor.
5. **Estabilizante** em qualquer tanque que vá ficar mais de 3 meses parado.
6. **Equipamento sazonal (motosserra, roçadeira, gerador pequeno):** guardar com o **carburador seco** — fechar a torneira e deixar apagar sozinho.

## Erros comuns

- Guardar gerador com gasolina comum no carburador por uma temporada → goma, e a próxima emergência começa com uma oficina.
- Superdimensionar gerador diesel "por segurança" → wet stacking e motor arruinado em poucos anos.
- Colocar S500 em veículo com DPF/SCR → pós-tratamento entupido, milhares de reais.
- Achar que premium "limpa o motor" — quem limpa é o pacote detergente (aditivada), não a octanagem.
- Ignorar dreno de água em tanque de diesel até a bomba de alta pressão falhar.
- Misturar óleo 2T "a olho" ou usar TC-W3 em motor refrigerado a ar.
- Reabastecer gerador quente sem esperar esfriar — a causa mais comum de incêndio de equipamento portátil.
- Rodar gerador em varanda/garagem "ventilada" — CO não perdoa.
