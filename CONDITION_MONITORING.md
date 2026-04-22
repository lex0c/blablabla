# Condition Monitoring

**Condition monitoring (CM)** é a disciplina de **inferir o estado de saúde** de equipamentos mecânicos/elétricos a partir de medições — vibração, óleo, temperatura, ruído, corrente, pressão — para detectar degradação **antes** que vire falha catastrófica. É o pilar técnico da **manutenção preditiva**: substitui calendário fixo ("troca rolamento a cada 6 meses") por condição observada ("rolamento mostra pico em 120 Hz com harmonics, troca em ~200 h").

Análogo mecânico de `OBSERVABILITY.md` — mesma filosofia (medir, correlacionar, alertar em desvio), aplicada a metal em rotação.

Complementa `MECHANICS.md`, `ROBOTICS.md`, `OBSERVABILITY.md`, `MLOPS.md`, `CIRCUIT.md`, `AI.md`.

## Filosofias de Manutenção

Evolução histórica, todas coexistindo hoje:

1. **Reativa / corretiva**: consertar quando quebra. Barato na frente, caro atrás (paradas não planejadas, dano colateral, segurança). Apropriada para ativos baratos, não críticos, redundantes.
2. **Preventiva (calendário)**: trocar/inspecionar em intervalos fixos. Reduz falhas mas **sobre-manutenção** (trocar peça boa) e **sub-manutenção** (falha antes do intervalo).
3. **Preditiva (condition-based)**: medir condição; intervir quando sinais aparecem. Maximiza vida útil real sem surpresa.
4. **Prescritiva**: preditiva + recomendação automatizada de ação (quais peças pedir, quando parar, por quanto tempo). ML + CMMS integrado.
5. **RCM (Reliability-Centered Maintenance)**: análise sistemática escolhe a estratégia por ativo/função baseada em consequência de falha.

Empresas maduras têm mix — não uma estratégia única. Criticidade define qual ativo merece CM caro.

## Curva P-F

Conceito central em preditiva:

```
Desempenho
    │
    │── — — — — — — — — ╲
    │                    ╲  P (falha potencial)
    │                     ╲
    │                      ╲
    │                       ╲
    │                        F (falha funcional)
    └───────────────────────→ Tempo

```

- **P**: ponto onde degradação vira detectável (ex.: mudança em vibração).
- **F**: ponto de falha funcional (parada, produto ruim).
- **Intervalo P-F**: janela para agir. Curta demais (dias) → pouco valor. Longa (meses) → excelente preditiva viável.

Cada **técnica** tem intervalo P-F típico. Ultrassom detecta mais cedo; termografia depois; humano ouvindo/cheirando, último.

## Técnicas

### Vibração

Técnica central em máquinas rotativas. Todo rotor desbalanceado, desalinhado, com rolamento degradado, ou folga vibra de forma característica.

#### Medições

- **Aceleração (g)**: para alta frequência (rolamentos, engrenagens).
- **Velocidade (mm/s RMS)**: banda média (desbalanço, desalinhamento). Padrão ISO 10816/20816.
- **Deslocamento (µm p-p)**: baixa frequência, máquinas grandes com proximity probes (turbomáquinas).

#### Sensores

- **Acelerômetro piezo (ICP/IEPE)**: padrão industrial. Montagem colada, parafusada, ou magnética. Resposta 0.5 Hz - 20+ kHz.
- **MEMS**: barato, IoT, wireless. Menos sensível em alta freq.
- **Proximity probes (Bently Nevada, Eddy current)**: sem contato, mede deslocamento do eixo relativo ao mancal. Turbomáquinas grandes.
- **Velocímetros (transdutores de velocidade)**: legados; hoje integra-se acelerômetro.
- **Laser Doppler Vibrometer**: laboratório; sem contato; preciso.

#### Domínios de análise

**Tempo**: forma de onda. Bom para impactos, transientes, modulações visíveis.

**Frequência (FFT, espectro)**: energia por frequência. Cada defeito tem assinatura.

**Órbita (X vs Y)**: em máquinas com dois proximity probes ortogonais. Revela precessão, rub, whirl.

**Cascade / waterfall**: espectro vs RPM ou tempo. Rampas de partida/parada expõem ressonâncias.

**Envelope / demodulação**: destaca impactos de baixa amplitude (rolamentos incipientes) modulando alta frequência.

**Cepstrum**: FFT do log do espectro. Detecta harmonics e sidebands regulares (engrenagens).

**Order tracking**: amostragem sincronizada com rotação — transforma eixo em "ordens" de RPM. Essencial em máquinas com velocidade variável.

#### Frequências características de rolamentos

Dado RPM do eixo e geometria, cada defeito gera frequência específica:

| Defeito | Frequência típica |
|---|---|
| **BPFO** (outer race) | ~40% × n × RPM |
| **BPFI** (inner race) | ~60% × n × RPM |
| **BSF** (ball spin) | depende de geometria |
| **FTF** (cage, fundamental train) | ~40% × RPM |

(`n` = número de esferas; valores aproximados.) Catálogos de fabricantes dão fórmulas exatas. Software (SKF, CSI, Emerson) calcula automaticamente.

#### Assinaturas típicas

- **Desbalanço**: 1× RPM, radial.
- **Desalinhamento**: 1× + 2× RPM; axial + radial; ângulo pior em 2×.
- **Folga mecânica**: 1×, 2×, 3× + harmônicos; sub-harmônicos (0.5×) em folga grave.
- **Rolamento**: BPFO/BPFI + harmônicos + sidebands; envelope mostra cedo.
- **Engrenagem**: GMF (gear mesh frequency = Z × RPM) + sidebands em frequência de rotação; cepstrum detecta.
- **Cavitação em bomba**: banda larga aleatória em alta frequência.
- **Whirl/whip em mancal hidrodinâmico**: ~0.42-0.48× RPM.
- **Electrical**: 2× freq. linha (120 Hz) em motor com problema.

#### Padrões

- **ISO 10816** / **ISO 20816**: limites de vibração aceitável por classe de máquina (I-IV) e estado (A bom, D inaceitável). Medida em velocidade RMS.
- **API 670**: proteção de máquinas (petroquímico); proximity probes + trip logic.
- **ISO 7919**: vibração em eixo (vs carcaça).

### Análise de Óleo (Tribologia)

Óleo de máquina é **biópsia**. Carrega partículas de desgaste, produtos de degradação, contaminação.

#### Testes

- **Viscosidade (40°C e 100°C)**: desvio do spec indica degradação/contaminação.
- **Índice de viscosidade (VI)**: resposta à temperatura.
- **TAN (Total Acid Number)**: oxidação.
- **TBN (Total Base Number)**: reserva alcalina (motores).
- **Teor de água (Karl Fischer)**: umidade → corrosão, mancal hidráulico comprometido.
- **ISO 4406** (contagem de partículas): codifica limpeza do óleo. Aplicações hidráulicas sensíveis (ex: servovalvulas 18/16/13).
- **Espectrometria (ICP/AES)**: elementos de desgaste (Fe=aço, Cu=bronze, Al=pistão, Cr=anéis, Pb=mancal, Si=poeira). Em ppm.
- **Ferrografia analítica**: microscópio ótico sobre lâmina magnética — morfologia de partículas diz modo de desgaste (fadiga, adesivo, abrasivo, corrosivo).
- **FTIR (espectroscopia infravermelha)**: grupos funcionais — oxidação, nitração, sulfação, água, glicol, fuligem.

Uma amostra por trimestre + bancos históricos + correlação com horas → detecção precoce muito antes de falha.

#### Aplicações

Transmissões, compressores, turbinas, motores de combustão, hidráulicos, caixas de engrenagem eólica.

### Termografia Infravermelha

Câmera IR mede temperatura superficial. Detecta:

- **Rolamentos sobreaquecidos**: falha iminente.
- **Acoplamentos/polias**: atrito anormal.
- **Motores**: enrolamentos desequilibrados, conexões soltas.
- **Conexões elétricas**: alta resistência elétrica em contatos → hotspot.
- **Vasos de pressão**: isolamento degradado, depósitos.
- **Rolamentos de forno/cimenteira**: refratário interno comprometido.

Boa para varredura rápida; ruim para defeitos internos sem expressão térmica.

**Classes NETA/ITC**: severidade por ΔT vs ambiente/similar.

### Ultrassom

- **Airborne ultrasound (20-100 kHz)**: vazamentos de ar/gás, arco/corona elétrica, descargas parciais.
- **Structure-borne**: contato via sonda em mancal — detecta defeitos incipientes em rolamentos **antes** de vibração.
- **Lubrificação guiada por ultrassom**: adicionar graxa enquanto ouve o ruído; parar quando atinge baseline. Evita sub/sobre-lubrificação.

Alcance de intervalo P-F costuma ser **maior** (mais tempo para agir) do que vibração.

### MCSA — Motor Current Signature Analysis

Análise espectral da corrente trifásica do motor. Detecta, sem sensor adicional:

- **Barras quebradas do rotor** (motores de indução): sidebands em (1 ± 2s)·f_linha, onde s é escorregamento.
- **Excentricidade de entreferro**: sidebands em f_linha ± k·f_rot.
- **Desbalanço entre fases**.
- **Defeitos em rolamentos** (menos sensível que vibração direta, mas possível).
- **Falhas em acoplamento mecânico** propagam pela carga.

Vantagens: não-intrusivo, remoto (CT no CCM), barato. Crescente adoção.

### Emissão Acústica (AE)

Ondas elásticas de alta frequência (100 kHz - 1 MHz) emitidas quando há **nucleação ou propagação de trinca**, deformação plástica localizada, atrito metal-metal.

Aplicações: monitoramento de vasos de pressão (NR-13 no Brasil), oleodutos, compósitos, rolamentos, válvulas (detecção de vazamento interno).

### Performance Monitoring

Variáveis de processo:

- **Rotação, torque, potência ativa**.
- **Vazão, pressão (sucção/descarga), Δp**.
- **Temperatura** (bobina, mancal, óleo, ambiente).
- **Corrente, tensão, fator de potência**.

Desvio de baseline (condições iguais) sinaliza degradação mesmo quando vibração/óleo estão limpos. Centrífugas, compressores, ventiladores exibem **queda de eficiência** mensurável antes de falhar.

### Outros

- **Eddy current testing** em tubos de trocador de calor: detecta corrosão, pitting.
- **Radiografia (RX/gamma)**: defeitos internos em soldas, fundidos.
- **Partículas magnéticas, líquido penetrante**: trincas superficiais.
- **Laser alignment** (Optalign, Rotalign, Easy-Laser): alinhar acoplamentos a <0.02 mm.
- **Balanceamento de campo**: acelerômetro + phase + massa teste.
- **Shock pulse (SPM)** — método proprietário, envelope-like.
- **Holografia de vibração, speckle interferometry**: imagens full-field de modos de vibração.

## Sensoreamento e Aquisição

### Condicionamento de sinal

Ver `CIRCUIT.md`. Acelerômetros IEPE precisam de alimentação constante de corrente (2-20 mA, 24 VDC). Proximity probes exigem drivers específicos. Termopares linearizam. Strain gauges usam pontes Wheatstone + amplificador.

### DAQ

- **Taxa de amostragem ≥ 2.56 × f_max de interesse** (ISO recomenda; Nyquist + margem anti-aliasing).
- **Janelas** (Hanning é padrão) reduzem leakage em FFT.
- **Averaging** (linear, exponencial, peak-hold): estabiliza espectros.

### Online vs portátil

- **Portátil** (Emerson CSI 2140, SKF Microlog, Fluke, Erbessd): técnico em rota mensal/trimestral. Barato.
- **Online fixo** (Bently Nevada 3500/ADAPT, CSI 6500, SKF IMx): ativos críticos 24/7, proteção + análise.
- **Wireless / IoT** (Emerson AMS Wireless, SKF Enlight, ABB Ability): meio-termo; expansão grande 2020+.

## Frameworks

### RCM — Reliability-Centered Maintenance

Metodologia (SAE JA1011/JA1012). Perguntas por ativo:

1. Qual a função?
2. Que falhas funcionais?
3. Modo de falha?
4. Efeito (segurança, operacional, econômico)?
5. Consequências?
6. Tarefa de manutenção preditiva, preventiva, detectiva ou funcionar até falhar?
7. Se nenhuma é efetiva, redesign.

RCM produz uma estratégia **específica por modo de falha**, não uma política global.

### FMEA / FMECA

**Failure Modes and Effects Analysis** (+ Criticality). Tabela estruturada:

- Função → Modo de falha → Causa → Efeito local → Efeito global → Severidade × Ocorrência × Detecção = **RPN**.
- Priorizar RPN alto para ação.
- FMECA adiciona análise de criticidade quantitativa.

Padrão em automotivo (AIAG-VDA), aeroespacial (ARP5580), nuclear.

### TPM — Total Productive Maintenance

Cultura originada na Toyota/Nissan. 8 pilares; ênfase em **operador envolvido em manutenção autônoma** (limpeza, inspeção diária) + kaizen contínuo.

Métrica chave: **OEE**.

### OEE — Overall Equipment Effectiveness

`OEE = Disponibilidade × Performance × Qualidade`

- **Disponibilidade**: tempo operando / tempo planejado.
- **Performance**: taxa real / taxa ideal.
- **Qualidade**: peças boas / peças produzidas.

Benchmark mundial ~85%; bom 75-85%; fabricas médias 50-60%. Cada componente tem perdas identificáveis ("seis grandes perdas").

### Bathtub Curve

Padrão clássico de taxa de falha vs tempo:

1. **Mortalidade infantil**: defeitos de fabricação, early failures.
2. **Vida útil**: taxa aproximadamente constante.
3. **Desgaste (wear-out)**: aumenta no fim.

Burn-in resolve (1); preditiva detecta (3). A forma real varia por componente — eletrônica é mais plana; mecânica clássica tem wear-out claro.

## CMMS / EAM

**Computerized Maintenance Management System** / **Enterprise Asset Management**:

- Hierarquia de ativos (plant → area → system → equipment → component).
- Ordens de serviço (OS).
- Histórico de manutenção, MTBF, MTTR.
- Catálogo de peças sobressalentes.
- Planos preventivos com frequência/uso.
- Integração com SCM, finanças.

**Produtos**: IBM Maximo, SAP PM, Infor EAM, Oracle eAM, IFS, UpKeep, Fiix, eMaint, Maintenance Connection.

Integração CM → CMMS automatiza: alerta de vibração → OS gerada com peça sugerida e prazo.

## IoT Industrial e ML

### Pipeline moderno

```
Sensores → edge gateway → cloud/historian → ML/analytics → alertas + dashboards → CMMS
```

### Tecnologias

- **Historian**: OSIsoft PI, Aveva, Canary, InfluxDB. Time-series optimized para dado de planta.
- **Protocolos**: OPC UA, Modbus TCP, Profinet, MQTT, EtherNet/IP (ver `IOT_FIRMWARE_SECURITY.md` para segurança).
- **Edge**: Siemens S7+, Rockwell, GE, AVEVA Edge, Ignition Edge.
- **Plataformas**: AWS Lookout for Equipment, Azure IoT + Stream Analytics, GE Predix, Siemens MindSphere, PTC ThingWorx, IBM Maximo APM.

### ML Aplicado

- **Anomaly detection**: autoencoders, isolation forest, one-class SVM. Treina com estado normal; flaga desvio. Valioso quando rotulagem de falha é escassa.
- **Classification** (uma vez que eventos rotulados existem): supervisionado — RF, XGBoost, CNN em espectros.
- **RUL (Remaining Useful Life)**: regressão temporal; LSTM, Transformer, survival models. C-MAPSS turbina NASA é dataset de referência.
- **Degradation trending**: fit paramétrico (exponencial, Weibull) em métrica de condição.
- **Digital twin**: modelo físico + dados em operação; reconciliação contínua.

### Armadilhas comuns

1. **Dados desbalanceados**: falha é evento raro; dataset é 99% normal.
2. **Rótulos ruidosos**: técnico anota "trocado rolamento" em vez de causa raiz confirmada.
3. **Concept drift**: máquina muda após reforma, retrofits, condições ambientais.
4. **Modelo por máquina vs flota**: modelos universais raramente funcionam; ativos têm personalidade.
5. **Dado sem contexto operacional**: mesmo motor sob carga diferente tem vibração diferente — tratar operating condition como feature.
6. **Explicabilidade**: engenheiro de manutenção quer saber **por que** o modelo alertou, não só que alertou.

## Modos de Falha por Classe

Conhecer catálogo dirige monitoramento.

### Motores elétricos de indução

- Rotor: barras quebradas, anel de curto-circuito.
- Estator: curto entre espiras, aterramento, sobretemperatura.
- Rolamentos: 41% das falhas (IEEE), maior peso.
- Eixo: desalinhamento, flexão.
- Entreferro: excentricidade estática/dinâmica.

### Rolamentos

- Desgaste superficial (spalling, pitting).
- Contaminação (partícula, água).
- Falha de lubrificação.
- Instalação errada (carga axial indevida).
- Passagem de corrente elétrica (EDM pits) em motores com VFD sem aterramento de eixo.

### Bombas

- Cavitação (erosão, queda de head).
- Desgaste em rotor e carcaça.
- Vazamento em selo mecânico.
- Problemas de aspiração (NPSH insuficiente).

### Compressores

- Válvulas (alternativos): trinca, acumulação.
- Rotor (centrífugos): fouling, depósitos.
- Pistão / anéis: desgaste, gás leak-by.

### Turbinas

- Palhetas: erosão, creep, trinca, depósito.
- Selagem (labirinto): desgaste.
- Instabilidade (flutter, surge).

### Transmissões / Redutores

- Engrenagem: pitting, micropitting, fratura de dente, scuffing.
- Óleo: degradação, contaminação.
- Rolamentos + selagem.

### Estruturas

- Fadiga de juntas soldadas, parafusos.
- Corrosão (uniforme, pitting, SCC, galvânica).
- Creep em alta temperatura.

## Certificações e Profissionalização

- **ISO 18436**: competência de pessoal em CM. Categorias I-IV por técnica (vibração, óleo, termografia, ultrassom).
- **Vibration Institute, Mobius Institute**: treinamento amplamente aceito.
- **ISO 18436-2** (vibração): Cat I (observação básica), Cat II (diagnóstico), Cat III (análise avançada + balanceamento), Cat IV (especialista/consultor).
- **ICML MLA/MLT** (lubrificação).
- **ASNT NDT Level I-III** para ensaios não-destrutivos.
- **CMRP (Certified Maintenance and Reliability Professional)** — SMRP: conhecimento amplo, gerencial.
- **CMRT (Technician)** — SMRP.

## Standards

- **ISO 17359**: condition monitoring geral.
- **ISO 13372-381** (série): vocabulário, data processing, prognóstico, comunicação.
- **ISO 13379**: data interpretation e diagnosis.
- **ISO 14224**: collect reliability data (oil & gas).
- **ISO 10816 / 20816**: vibração mecânica — evaluation by measurements on non-rotating parts.
- **ISO 7919**: vibration em rotating shafts.
- **ISO 4406**: cleanliness de óleo hidráulico.
- **API 670**: machinery protection systems.
- **SAE JA1011/JA1012**: RCM standard.
- **MIL-STD-1629**: FMECA.

## Cases e Referências

- **NASA PCoE datasets**: turbofan (C-MAPSS), bearing, lithium-ion — standard em papers.
- **IMS bearing dataset** (Univ. Cincinnati, NASA): run-to-failure de rolamentos.
- **Case Western Reserve University bearing dataset**: clássico em diagnóstico.
- **Emerson / SKF / Bently Nevada case studies**: defeitos reais, assinaturas, intervenções.

## Recursos

- **"An Introduction to Predictive Maintenance"** — R. Keith Mobley.
- **"Practical Machinery Vibration Analysis and Predictive Maintenance"** — Cornelius Scheffer, Paresh Girdhar.
- **"Machinery Failure Analysis Handbook"** — Luis Sanchez.
- **"Vibration Fundamentals and Practice"** — Clarence W. de Silva.
- **"Reliability-Centered Maintenance"** — John Moubray.
- **"Maintenance Engineering Handbook"** — Keith Mobley.
- **Emerson Reliability blog, SKF Insight, Mobius Learning resources, Uptime Magazine**.
- **Vibration Institute, Mobius Institute, ICML, SMRP** — treinamento e certificação.
- **NASA Prognostics Center of Excellence** — datasets e research.
- **IEEE Reliability Society, ASME PVP**.

## Princípios

1. **Criticidade guia investimento**. Ativo parado custa pouco? Reativa ok. Custa milhões/h? Online + ML.
2. **Múltiplas técnicas ganham** — vibração + óleo + termografia se complementam; defeito que escapa uma é pego pela outra.
3. **Baseline antes de alarme**. Sem referência do "normal" da *sua* máquina, alerta é chute.
4. **Operating condition é feature**. Comparar espectro em 50% de carga com 100% é errado.
5. **FMEA dirige monitoramento**. Monitore o que falha e importa, não o que é fácil de medir.
6. **Rota + historial**. Ronda disciplinada + dados em histórico valem mais que sensor caro sem registro.
7. **Envolver operador**. TPM vs silo de manutenção. Olhos na máquina todo turno > câmera sem interpretação.
8. **Intervir no momento certo**. Cedo demais = desperdício; tarde = catástrofe. Curva P-F é a bússola.
9. **Dado + domínio**. ML sem engenheiro de confiabilidade produz modelos caros que ninguém aciona.
10. **Close the loop**: CM → OS → execução → feedback. Sem isso, monitoramento é museu.
