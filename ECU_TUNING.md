# ECU Tuning — Remap, Calibração e Riscos

Reprogramação de central de motor: o que realmente se altera, como se acessa, o que se ganha, o que quebra e o que a lei diz. `CAR_HACKING.md` cobre barramento e segurança (CAN, UDS, ataques); `MOTORS.md` cobre os ciclos; `FUELS.md` cobre octanagem e detonação — que é o teto de quase todo remap. Aqui é a **calibração**.

## O que remap é (e o que não é)

Remap é **reescrever as tabelas de calibração** gravadas na memória flash da ECU. Não se troca o programa — troca-se o conjunto de mapas que o programa consulta.

Evolução do acesso:

| Era | Método | Realidade |
|---|---|---|
| Anos 80–90 | **Chip**: dessoldar a EPROM e soldar outra | Daí o nome "chiptuning", hoje obsoleto |
| ~1996–2010 | Flash via **OBD** | Leitura e gravação diretas, checksum simples |
| ~2010–2018 | TriCore com **checksum forte**; muitas travadas em OBD | Aparece a **bancada** (BDM/BSL/boot mode) |
| 2017+ | **MD1/MG1, SIMOS 18+**: firmware assinado, secure boot, HSM | Exige desbloqueio de bootloader; muitos casos só via serviço |

O que **não** é remap: piggyback ("módulo de potência" em série com sensores, enganando a ECU) e chicote de pressão. Funciona, é reversível, e é inferior — o módulo mente para a ECU, que segue calculando errado e corrigindo contra você. Piggyback tem lugar em ECU impossível de abrir; fora disso, é remendo.

## Arquitetura por torque — o conceito central

**Este é o ponto que separa calibração de quem só mexe em slider.**

ECU moderna (Bosch ME7 em diante, ~1998) não é "pedal → borboleta". É **estruturada por torque** (*Drehmomentstruktur*):

```
pedal → torque desejado pelo motorista
                ↓
      arbitragem entre demandas
   (motorista, câmbio, ESP, ar-cond., cruise, proteções)
                ↓
        torque-alvo do motor
                ↓
   conversão em enchimento (carga) necessário
                ↓
   borboleta + pressão de turbo + ignição + injeção
                ↓
      torque modelado ⟷ torque real (monitoração)
```

Consequências práticas:

- Existe um **torque modelado** que a ECU acredita estar produzindo, e uma camada de **monitoração** que compara pedido × modelado × plausível. Divergiu além do tolerado → corte, limp mode, DTC.
- **Subir pressão de turbo sem subir os limitadores de torque e o modelo é inútil**: a ECU corta, fecha borboleta ou puxa avanço. Metade dos remaps ruins do mercado é exatamente isso.
- O **câmbio** também pede torque. Um DSG/ZF com limite de 350 N·m vai **cortar a festa** por conta própria se a TCU não for calibrada junto.
- Proteções (EGT, knock, temperatura de líquido, temperatura de ar de admissão) entram como **limitadores de torque**, não como avisos. Um carro remapeado que "não anda no calor" normalmente está batendo em limitador térmico, não perdendo potência por física.

Nomes clássicos de tabela na Bosch ME7 — úteis porque a nomenclatura sobreviveu na literatura da comunidade:

| Tabela | Função |
|---|---|
| `KFMIRL` | carga necessária a partir do torque pedido |
| `KFMIOP` | torque ótimo a partir da carga (o modelo inverso) |
| `LDRXN` | limite de enchimento — **o limitador real de boost** |
| `KFLDRL` / `KFLDIMX` | duty da wastegate (N75) e limite |
| `KFZW` / `KFZW2` | avanço de ignição (mapa principal e alternativo) |
| `LAMFA` | enriquecimento por carga (proteção térmica) |
| `MLHFM` | linearização do MAF |
| `DIMX` | duty máximo do bico |

## Anatomia dos mapas

### Gasolina turbo

| Mapa | O que faz | Risco ao mexer |
|---|---|---|
| **Alvo de pressão de turbo** | define o enchimento | Sobre-rotação do turbo, pressão fora do mapa do compressor |
| **Duty da wastegate / PID** | como o alvo é atingido | Overshoot, oscilação, cortes |
| **Avanço de ignição** | onde está a maior parte do ganho | **Detonação** — a forma mais rápida de matar pistão |
| **Alvo de lambda** | mistura por carga/rotação | Pobre em carga alta = EGT e detonação |
| **Modelo/limitadores de torque** | permite que o resto tenha efeito | Sem isso, nada funciona |
| **Mapeamento de pedal** | resposta (não é potência) | Zero ganho real — é o truque do "ficou mais rápido" |
| **VVT / fases de comando** | enchimento e resposta | Requer entendimento de overlap |
| **Corte de giro** | rev limiter | Válvula, corrente, biela |
| **Rail de alta pressão (GDI)** | vazão de combustível | Se o alvo cai, você achou o limite da bomba |

### Diesel turbo

| Mapa | O que faz |
|---|---|
| **Quantidade de injeção (IQ)** | o análogo direto da potência |
| **Pressão de rail** | atomização; comum ir a 1800–2200 bar |
| **Timing de injeção** | pré/piloto/principal/pós — dirigibilidade, ruído, NOx |
| **Mapa de fumaça (smoke limiter)** | teto de combustível **em função da massa de ar** |
| **Alvo de turbo (VGT)** | geometria variável |
| **Limitadores por marcha** | proteção de transmissão |
| **EGT** | proteção de turbina e pós-tratamento |

No diesel, o limite prático não é detonação — é **fuligem, EGT e transmissão**. Passar do mapa de fumaça é o que produz a "fumaça preta de carro remapeado": combustível não queimado, ou seja, **potência que você pagou e não recebeu**, além de entupir DPF e diluir óleo.

## Acesso e ferramentas

### Vias de acesso

- **OBD:** pela porta de diagnóstico, motor no carro. Rápido, e o modo padrão onde a ECU permite.
- **Bancada (bench):** ECU fora do carro, alimentada em bancada, acessada por **BDM**, **JTAG** ou **BSL/boot mode** (bootstrap loader do TriCore). Necessário quando o OBD está travado.
- **Boot mode com pinagem:** abrir a ECU e puxar um pino para forçar o bootloader. Invasivo, mas às vezes é o único caminho.
- **Serviços de desbloqueio:** para MD1/MG1 e similares, o mercado gira em torno de servidores que destravam o bootloader antes da gravação.

### Cuidados operacionais que evitam brick

- **Fonte estabilizada / battery stabilizer** durante a gravação. Queda de tensão no meio de um flash é a causa número um de ECU morta.
- **Ler e guardar o arquivo original** antes de qualquer coisa. Sem backup você não tem plano B — e é o primeiro sinal de tuner ruim.
- **Correção de checksum**: a ECU valida o flash (CRC, e nas modernas assinatura). Ferramenta boa corrige automaticamente; arquivo com checksum errado não roda ou entra em recovery.
- **Leitura virtual**: algumas ferramentas "reconstroem" o arquivo a partir de banco de dados em vez de ler o carro. Serve para orçamento, **não** para calibrar — o carro pode ter revisão de software diferente.
- **Imobilizador / component protection**: trocar ou zerar ECU (virgin/clone) esbarra em Komponentenschutz (VAG), ISN/CAS (BMW) e equivalentes. É um problema à parte, não resolvido pelo remap.

### Famílias de ECU

| Fabricante | Famílias | Nota |
|---|---|---|
| Bosch | ME7.x, MED9, **MED17/EDC17** (TriCore), **MD1/MG1** | MD1 (diesel/GDI) e MG1 (gasolina) são a geração assinada |
| Siemens/Continental | SIMOS 3/8/18+, PCR 2.1, EMS | SIMOS 18+ com proteção forte |
| Delphi | DCM 3.x/6.x, MT | Comum em diesel comercial e GM |
| Magneti Marelli | IAW 4/5/7/8, 8GMF | Forte em Fiat/GM/Stellantis e no Brasil |
| Denso | SH705x e sucessores | Toyota, Honda, Mazda, Subaru |
| GM (Delco) | E38/E39/E80/E92 | Ecossistema HP Tuners |

### Software

- **WinOLS** — padrão de fato para achar e editar mapas, com **DAMOS**/A2L quando existe (o arquivo de definição da montadora), ou por *checksum/pattern matching* quando não.
- **ECM Titanium, Swiftec** — camadas com driver/definições prontas, mais acessíveis e menos profundas.
- **HP Tuners, EcuTek, COBB Accessport** — ecossistemas fechados com edição + datalog integrados (fortes em GM/Ford, Subaru/Nissan, e plataformas específicas).
- **Hardware:** Alientech (KESS/KTAG), Dimsport (New Genius/Trasdata), Autotuner, MagicMotorsport Flex, CMD.
- **Datalog:** VCDS / OBDeleven (VAG), ME7Logger, ferramentas nativas dos ecossistemas acima. **Dongle OBD genérico não serve** — taxa de amostragem baixa demais para ver detonação.

## Stages — o que significam de verdade

| Stage | Hardware | Ganho típico (turbo) |
|---|---|---|
| **Stage 1** | tudo original | +20–30% torque |
| **Stage 2** | downpipe, intake, intercooler maior | +30–45% |
| **Stage 3 / híbrido** | turbo maior, bicos, bomba, embreagem, às vezes internos | +60%+ |

E o número que ninguém gosta de ouvir: **motor aspirado rende 2–5%**. Aspirado só ganha de verdade com hardware (comando, admissão, escape, taxa de compressão) — remap sozinho ali só ajusta dirigibilidade e tira limitador de giro. Quem promete "15% no aspirado" está mentindo ou tirando margem de segurança do motor.

**OTS (off-the-shelf) × custom:** OTS é arquivo genérico da mesma família — funciona porque as montadoras deixam margem, mas ignora o seu combustível, seu clima, seu estado de manutenção. Custom é calibrado com datalog e, idealmente, dinamômetro. A diferença de preço reflete isso.

Onde a margem de fábrica vem: a montadora calibra para o pior caso global — pior combustível disponível, 45 °C, altitude, óleo esticado, 200.000 km, garantia longa e metas de emissão. **O remap consome essa margem.** Isso não é gratuito: é uma troca de robustez por desempenho, e a honestidade profissional está em dizer quanto se consumiu.

## Combustível — o teto do remap

Avanço de ignição é onde mora a maior parte do ganho em ciclo Otto, e o limite dele é **detonação** (ver `FUELS.md`).

- Remap de gasolina normalmente **passa a exigir** premium. Rodar com comum um arquivo calibrado para 95+ significa a ECU puxando avanço o tempo todo — você perde o ganho e ainda vive na borda.
- **Etanol é a vantagem brasileira.** E100 é RON ~109 com alto calor de vaporização: permite mais avanço e mais pressão de turbo do que qualquer gasolina de posto. Um mesmo motor pode fazer números "de combustível de competição" com álcool de posto.
- O custo: etanol precisa de **~30–40% mais vazão de combustível** (AFR estequiométrico ~9:1 contra 14,7:1). O limite passa a ser **bico e bomba de alta pressão** — e aparece no datalog como **queda da pressão de rail em relação ao alvo**.
- **Mapa flex** precisa saber o teor de etanol: ou por **sensor de flex fuel**, ou por interpolação a partir da correção de lambda (o "flex por software" dos nacionais). Sem isso, um arquivo agressivo de álcool com tanque de gasolina é receita de furo de pistão.

## Datalog e validação

Sem datalog, não é calibração — é chute com nota fiscal. O que registrar em pulls de carga plena:

| Parâmetro | O que observar |
|---|---|
| **Knock retard / contagem por cilindro** | Qualquer retardo sustentado = passou do ponto |
| **Boost alvo × real** | Overshoot, queda no fim, oscilação |
| **Duty da wastegate** | Batendo em 95–100% = turbo no limite |
| **Lambda (wideband)** | Sonda de banda estreita não serve em carga plena |
| **Rail pressure alvo × real** | Desvio = limite de bomba/bico |
| **Duty do bico** | >85% = sem margem |
| **IAT / temperatura pós-intercooler** | *Heat soak*: o 3º pull dizendo a verdade sobre o 1º |
| **EGT** | Turbina de gasolina >950 °C, diesel pré-turbina >750 °C sustentado = perigo |
| **Trims (STFT/LTFT)** | Vazamento, MAF, injetor sujo |
| **Falhas de combustão (misfire)** | Vela e bobina no limite |

Sobre **dinamômetro**: número de dyno **não é comparável entre dinamômetros** (inercial × frenado, fator de correção SAE/DIN/EEC, temperatura, pneu, marcha). Serve para comparar **antes e depois no mesmo equipamento, no mesmo dia**. Print de dyno de terceiro na propaganda não significa nada sobre o seu carro.

## Riscos mecânicos

| Componente | Como falha | Sinal |
|---|---|---|
| **Embreagem / DSG** | patina acima do torque de projeto | Rotação sobe sem acelerar; DQ200 seco é o mais frágil |
| **Turbo** | sobre-rotação, eixo, selo de óleo | Fumaça azul, apito diferente, consumo de óleo |
| **Pistão / biela** | detonação, LSPI, carga alta em baixo giro | Batida, perda de compressão |
| **Bronzina** | carga alta em rotação baixa ("lugging") | Barulho grave; conhecido em N54/S55 e EA888 |
| **Corrente/tensionador** | mais torque na cadeia | Ruído em partida a frio |
| **Intercooler/radiador** | não dá conta do calor extra | Ganho some do 2º pull em diante |
| **DPF (diesel)** | mais fuligem, mais regeneração | Regen frequente, **diluição do óleo por combustível** |
| **Catalisador** | EGT e mistura pobre | Perda de fluxo, luz de emissão |
| **Freio e pneu** | não foram remapeados | Óbvio, e o mais ignorado |

Regra dura: **manutenção antes de remap.** Vela no fim da vida, bobina fraca, mangueira ressecada, intercooler sujo, óleo vencido — o remap não causa a falha, ele **antecipa** todas elas de uma vez.

## Emissões e "deletes"

O mercado oferece supressão de **EGR, DPF, catalisador, sonda e SCR/AdBlue**. Registro do que é, sem instruções:

- É **ilegal para uso em via pública** — no Brasil, adulterar sistema de controle de emissões viola o PROCONVE e configura infração ambiental; nos EUA, o Clean Air Act já gerou multas milionárias contra oficinas e vendedores de arquivo. Na Europa, reprova em inspeção.
- Tecnicamente também é ruim: DPF removido joga material particulado fino direto no ar (o poluente diesel com efeito à saúde mais documentado); EGR removido **aumenta NOx**; catalisador removido normalmente rende quase nada em motor turbo de rua — o ganho vem do *downpipe* menos restritivo, não da ausência de catalisador (um esportivo de alto fluxo entrega quase o mesmo).
- Problemas práticos: DTC permanente, luz de avaria, impossibilidade de vender o carro sem reverter, e recusa de garantia/seguro.

Não vou detalhar procedimento. Descrevo o que é, e por que a maioria dos casos que se apresentam como "solução para DPF entupido" é, na verdade, um problema de uso (curtas distâncias, regen sempre interrompida) que se resolve com uso correto ou limpeza.

## Outras alterações comuns

- **Pops & bangs / crackle map:** retarda ignição e injeta no desaceleração para estourar no escape. Queima válvula de escape, destrói catalisador, e é infração de ruído. Puro custo, zero desempenho.
- **Launch control / no-lift shift:** útil em aplicação de pista; castiga embreagem e transmissão.
- **Corte hard × soft:** hard cut é agressivo com o trem de força; soft é mais civilizado.
- **Start-stop off:** conforto, sem efeito de potência.
- **Remoção de limitador de velocidade:** verifique o **índice de velocidade do pneu** antes. Pneu não é software.
- **TCU tune (DSG/ZF):** pressão de embreagem, limites de torque, estratégia de troca. Em muitos casos é **obrigatório** para o remap de motor funcionar.
- **Rev limiter maior:** só com hardware que aguente — comando, mola de válvula, biela.

## Legal, garantia e seguro (Brasil)

- **Garantia:** a montadora nega powertrain ao detectar. Detecção é rotina hoje: **contador de gravações**, hash/checksum do flash, log das ferramentas de fábrica (ODIS, ISTA), e a flag histórica **TD1** no grupo VAG. Não conte com "eu volto o original antes da revisão" — o contador não volta.
- **Homologação:** alteração de característica do veículo exige inspeção em ITL e emissão de **CSV**, com anotação no CRV. A resolução do CONTRAN sobre modificações muda de tempos em tempos — **confirme a vigente** antes de afirmar qualquer coisa.
- **Seguro:** modificação não declarada agrava o risco e dá base para negativa de sinistro. Declare, ou aceite conscientemente o risco.
- **Uso em pista:** contexto diferente e é onde a maior parte disso faz sentido de verdade.

## EV e híbridos

O "remap" muda de natureza: não há mistura nem avanço. O que limita é **corrente de bateria, corrente e temperatura do inversor, e derating térmico**. Alterações significam liberar limites de software — às vezes vendido pela própria montadora (Tesla *Acceleration Boost* é literalmente um remap oficial pago). Riscos migram para pack, inversor e semieixo, e o pacote de segurança é bem mais fechado (firmware assinado, OTA que reverte, telemetria que registra). Ver `BATTERIES.md` e `ENERGY.md`.

## Como escolher — red flags

Fuja se o profissional:

- **Não lê e não entrega o arquivo original.**
- Faz tudo em 15 minutos sem nenhum datalog nem test drive.
- **Não pergunta qual combustível** você usa, nem o estado de manutenção.
- Promete ganho grande em **aspirado**.
- Não menciona TCU num carro de câmbio automatizado com limite de torque.
- Trabalha com **leitura virtual** para calibrar.
- Grava com o carro na bateria fraca, sem fonte.
- Não fala em nenhum risco — só em número.

Bons sinais: pergunta antes de vender, pede revisão em dia, mostra log de antes e depois, explica o que vai mudar em consumo e em exigência de combustível, e oferece o retorno ao original.

## Erros comuns

- Subir turbo sem tocar no modelo/limitadores de torque → cortes e limp mode.
- Aplicar arquivo agressivo de etanol com tanque de gasolina.
- Ignorar `heat soak`: calibrar no primeiro pull frio e entregar um carro que puxa avanço em toda subida de serra.
- Remapear com manutenção atrasada e culpar o remap.
- Confiar em sonda de banda estreita para carga plena.
- Comparar dyno de oficinas diferentes.
- Esquecer a transmissão, os freios e o pneu.
- Achar que mapa de pedal agressivo é ganho de potência (é resposta — o carro não ficou mais rápido).
- Deletar emissão "porque todo mundo faz" e descobrir na hora de vender o carro.

## Regras práticas

- **Torque manda em dirigibilidade; potência manda em número de propaganda.** Um Stage 1 bem-feito ganha onde você usa: 2.000–4.000 rpm.
- **O remap consome margem de fábrica.** Isso é uma escolha legítima, desde que consciente.
- **O ganho é limitado pelo combustível** em Otto, e por **fumaça/EGT/transmissão** em diesel.
- **Se não foi logado, não foi calibrado.**
- **Manutenção primeiro, potência depois.**
- **O elo mais fraco define o resultado** — quase sempre a embreagem, o intercooler ou a bomba de combustível, nunca o "arquivo".
