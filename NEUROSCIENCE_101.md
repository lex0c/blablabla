# Neurociência 101

O cérebro é o órgão mais complexo conhecido: ~86 bilhões de neurônios, ~100-500 trilhões de sinapses, consumindo ~20% da energia do corpo. Entender seus princípios básicos — como neurônios comunicam, como circuitos emergem, como sono e neurotransmissores moldam cognição — não é apenas curiosidade. É fundamento para raciocinar sobre motivação, aprendizado, decisão, saúde mental e mesmo para calibrar expectativas sobre o que IA faz ou não reproduz.

Complementa `AI.md` (redes neurais artificiais), `SOCIAL_PSYCHOLOGY.md`, `MEDITATION.md`, `CONTINUOUS_LEARNING.md`, `CHAOS_CREATIVITY.md`.

## Níveis de Análise

Neurociência opera em escalas:

1. **Molecular**: canais iônicos, neurotransmissores, receptores.
2. **Celular**: neurônios, glias, sinapses.
3. **Circuitos**: conjuntos de neurônios conectados com função.
4. **Sistemas**: regiões cerebrais.
5. **Comportamento / cognição**: emergente do resto.

Conectar níveis é o grande desafio; "um neurônio explica X" raramente é correto.

## Neurônio

### Anatomia

- **Soma** (corpo): núcleo, maquinaria metabólica.
- **Dendritos**: ramificações que recebem input.
- **Axônio**: prolongamento longo que envia output. Pode ter metros (nervo ciático).
- **Terminais axonais**: sinapses com outros neurônios.
- **Bainha de mielina**: isolamento (oligodendrócitos no SNC, células de Schwann no SNP). Acelera condução 10-100x.

### Potencial de Membrana

Diferença de carga entre interior e exterior da célula.

- **Em repouso**: ~-70 mV. Mantido por bomba Na⁺/K⁺ ATPase (exporta 3 Na⁺, importa 2 K⁺; custa ATP).
- **Despolarização**: menos negativo (Na⁺ entra).
- **Hiperpolarização**: mais negativo (K⁺ sai, Cl⁻ entra).

### Potencial de Ação (AP)

Quando despolarização cruza limiar (~-55 mV), canais de Na⁺ voltage-gated abrem rapidamente → entrada massiva → +40 mV → K⁺ voltage-gated abrem → saída → volta a -70 mV. Dura ~1 ms.

Propaga pelo axônio a velocidades de 1-120 m/s dependendo de diâmetro e mielinização.

**All-or-none**: o AP tem amplitude fixa; informação está na **frequência** (rate coding) e **timing** (temporal coding).

### Sinapse

Ponto de comunicação entre neurônios.

- **Química** (dominante): AP chega no terminal → Ca²⁺ entra → vesículas com neurotransmissor fundem à membrana → neurotransmissor atravessa fenda sináptica → liga-se a receptores na membrana pós-sináptica → abre canais ou ativa cascatas.
- **Elétrica** (gap junctions): passagem direta; rápida, raras no cérebro adulto.

Sinapses podem ser **excitatórias** (despolarizam — aumentam chance de AP) ou **inibitórias** (hiperpolarizam — diminuem). Integração em dendritos decide se AP dispara.

## Neurotransmissores e Neuromoduladores

### Excitatórios/Inibitórios principais

- **Glutamato**: excitatório dominante no SNC. Receptores AMPA (rápido), NMDA (plasticidade), metabotrópicos (lento).
- **GABA**: inibitório dominante. Receptores GABAₐ (Cl⁻, rápido) e GABA_B (metabotrópico). Álcool, benzos potencializam GABAₐ.
- **Glicina**: inibitório no tronco e medula.

### Moduladores (lentos, difusos, amplo efeito)

- **Dopamina**: predição de recompensa, motivação, movimento. Vias: mesolímbica (recompensa), mesocortical (função executiva), nigroestriatal (movimento; Parkinson é degeneração aqui), tuberoinfundibular (prolactina).
- **Serotonina (5-HT)**: humor, ansiedade, sono, apetite, sociabilidade. SSRIs bloqueiam reuptake.
- **Noradrenalina**: atenção, vigília, stress. Locus coeruleus.
- **Acetilcolina**: atenção, memória, movimento (junção neuromuscular). Alzheimer envolve depleção.
- **Histamina**: vigília (antihistamínicos sedativos atravessam BBB).
- **Endocanabinoides** (anandamida, 2-AG): retroalimentação, modulação.
- **Opioides endógenos** (endorfinas, encefalinas): analgesia, recompensa.

### Neuropeptídeos

Ocitocina (bonding, confiança), vasopressina, substância P, CRH, neuropeptídeo Y, orexinas (vigília).

## Glia — não é coadjuvante

Antes subestimada; hoje central:

- **Astrócitos**: homeostase iônica, recaptação de glutamato, metabolismo, regulação vascular (neurovascular coupling), faz parte da barreira hematoencefálica.
- **Oligodendrócitos** (SNC) / **Schwann** (SNP): mielina.
- **Microglia**: imunidade residente, poda sináptica, resposta a lesão.
- **Ependimais**: revestem ventrículos, LCR.

Número de glias ≈ neurônios em humanos (correção de estimativas antigas).

## Anatomia Funcional

### Divisão grosseira

- **Telencéfalo**: córtex cerebral + núcleos da base + sistema límbico (hipocampo, amígdala).
- **Diencéfalo**: tálamo, hipotálamo.
- **Mesencéfalo**: substância negra, teto, colículos.
- **Metencéfalo**: ponte, cerebelo.
- **Mielencéfalo**: bulbo.

### Córtex cerebral

~2-4mm espessura, 6 camadas (com exceções). ~16 bilhões de neurônios.

**Lobos**:
- **Frontal**: função executiva, planejamento, motor primário, linguagem expressiva (Broca em dominante).
- **Parietal**: somatossensorial, atenção, mapeamento espacial.
- **Temporal**: audição, linguagem receptiva (Wernicke), memória (hipocampo), reconhecimento visual (córtex inferotemporal).
- **Occipital**: visão.
- **Insular, cingulado**: emoção, interocepção, controle.

### Estruturas subcorticais chave

- **Hipocampo**: consolidação de memória episódica, navegação espacial.
- **Amígdala**: medo, emoção, valoração.
- **Tálamo**: relay de quase toda informação sensorial (exceto olfato) para córtex.
- **Hipotálamo**: controle autônomo, hormônios, fome/sede/sono/temperatura/sexo.
- **Núcleos da base** (striatum, pallidum, etc.): seleção de ação, hábito, movimento.
- **Cerebelo**: coordenação motora, timing, crescentemente creditado com cognição.
- **Tronco encefálico**: funções vitais (respiração, cardiovascular), sono-vigília (reticular).

### Sistema nervoso periférico

- **Somático**: controle voluntário de músculos esqueléticos.
- **Autônomo**:
  - **Simpático**: luta/fuga. Norepinefrina, epinefrina.
  - **Parassimpático**: descanso/digestão. Acetilcolina (vago).
- **Entérico**: "segundo cérebro" no trato digestivo; ~500M neurônios; regula digestão largamente autônomo.

## Neurotransmissor Distúrbios

- **Parkinson**: perda de dopaminérgicos na substância negra.
- **Depressão**: desregulação de serotonina/NE/dopamina (modelo monoaminérgico — simplista mas útil).
- **Esquizofrenia**: hiperatividade dopaminérgica mesolímbica; déficit glutamatérgico (NMDA hipofunção).
- **Alzheimer**: placas Aβ, tangles tau, colinérgico degenerado.
- **Vício**: mesolímbica hijacked; dopamina-recompensa condicionada.
- **Ansiedade**: GABAérgico / serotonérgico.

Neurobiologia informa farmacologia mas cada distúrbio é sistema — não química única.

## Plasticidade

### Hebbian ("neurons that fire together wire together")

Coincidência temporal fortalece sinapse. Base de aprendizado associativo.

### LTP (Long-Term Potentiation)

Estimulação de alta frequência em sinapse glutamatérgica → fortalecimento durável via NMDA + AMPA trafficking, CaMKII, mudanças estruturais. Mecanismo molecular central de memória.

### LTD (Long-Term Depression)

Enfraquecimento análogo em outras condições. Ambos necessários — memória requer tanto fortalecer relevante quanto enfraquecer ruído.

### STDP (Spike-Timing Dependent Plasticity)

Timing preciso (pre antes de post → LTP; depois → LTD) em janelas de milissegundos.

### Plasticidade estrutural

Sinapses nascem e morrem. Dendritos ramificam/podam. **Podas sinápticas** massivas na infância e adolescência.

### Neurogênese

Adult neurogenesis em hipocampo (giro denteado) e bulbo olfatório — evidência em humanos disputada; experimentos animais claros. Rapamycin, exercício aumentam; estresse diminui.

### Mielinização

Contínua até ~25 anos no córtex pré-frontal. Acelera circuitos com uso.

## Aprendizado e Memória

### Tipologia

- **Declarativa (explícita)**: episódica (eventos pessoais) + semântica (fatos). Hipocampo-central.
- **Não-declarativa (implícita)**: procedural (habilidades), priming, condicionamento, hábitos. Núcleos da base, cerebelo.
- **De trabalho**: segurar info por segundos (Baddeley model: phonological loop, visuospatial sketchpad, episodic buffer, central executive). Córtex pré-frontal central.

### Consolidação

Memória nova é frágil; consolida em horas a semanas via **replay** hipocampal, frequentemente durante sono. Eventualmente torna-se independente do hipocampo (teoria standard; debate vs teoria multiple trace).

### Esquecimento

Funcional — pick fibers que merecem ser fortalecidas. Interferência, supressão ativa, reconsolidation errors.

### Reconsolidação

Lembrar → memória fica temporariamente lábil → pode ser alterada. Mecanismo explorado em terapia para PTSD (imaginal reconsolidation, MDMA-assisted).

## Atenção

### Redes

- **Alerting** (locus coeruleus, noradrenalina): vigilância.
- **Orienting** (parietal, frontal eye fields): shift.
- **Executive** (ACC, DLPFC): resolução de conflito, controle top-down.

### Bottom-up vs top-down

- **Bottom-up**: saliência estimular (movimento, cor, surpresa) captura atenção.
- **Top-down**: objetivo dirige (buscar carro azul no estacionamento).

### Limites

Capacidade atencional é estreita. Multi-tasking real é raro — é task-switching, com custo cognitivo. "Atenção" não é recurso único; vários sistemas dissociáveis.

## Sono

Ciclo 4-5 vezes por noite, ~90 min cada:

- **NREM 1-3**: sono progressivamente profundo. N3 (slow wave) é onde consolidação declarativa acontece e crescimento/reparação ocorre.
- **REM**: rápido movimento ocular, atonia muscular, sonhos vívidos, consolidação procedural/emocional.

### Glinfático

Durante sono, espaço intersticial aumenta ~60%; LCR flushing remove resíduos (incluindo β-amiloide). Privação de sono → acúmulo. Possível elo com neurodegeneração.

### Ritmo circadiano

Gerado por núcleo supraquiasmático no hipotálamo, sincronizado com luz via células ganglionares retinianas (melanopsina). Melatonina (pineal) sinaliza "escuro". Disruption (trabalho noturno, jet lag, exposição azul à noite) tem custos.

### Efeito de privação

1-2 noites perdidas: atenção, humor, decisão, imune degradam.
Crônica: risco cardiovascular, metabólico, cognitivo, psiquiátrico.

Consumer wearables (Oura, Whoop, Apple) dão proxy útil mas imperfeito.

## Emoção

### Teorias

- **James-Lange**: percepção → mudança corporal → emoção.
- **Cannon-Bard**: percepção dispara corpo e emoção em paralelo.
- **Schachter-Singer (two-factor)**: arousal + interpretação cognitiva.
- **Appraisal** (Lazarus): avaliação cognitiva modula.
- **Constructed Emotion** (Barrett): emoção é categoria construída, não módulo universal. Controverso; ganhando tração.

### Amígdala

Detecção de ameaça, condicionamento de medo. Pavlov + amígdala = base de fobias e PTSD.

### Regulação

Córtex pré-frontal, cingulado anterior modulam amígdala. Reappraisal (reinterpretar) é estratégia mais saudável que suppression.

## Recompensa e Motivação

### Sistema mesolímbico

VTA (área tegmental ventral) → nucleus accumbens (NAc) → córtex pré-frontal. Dopamina é sinal de **erro de predição de recompensa** (Schultz) — não de prazer per se. Predição cumprida, dopamina fica quiet; predição violada (positivamente), spike.

### Wanting vs liking (Berridge)

- **Wanting** (motivação): dopaminérgico.
- **Liking** (prazer hedônico): opioid/endocannabinoid em hotspots específicos do NAc/pálido ventral.

Dissociáveis — vício aumenta wanting sem aumentar liking. Explica craving residual após prazer cessar.

### Intrinsic motivation

Autonomia, competência, relacionamento (Deci & Ryan SDT). Recompensa externa pode **minar** motivação intrínseca (overjustification effect).

## Desenvolvimento

- **Neurogênese fetal** massiva; bilhões de neurônios por dia em certos picos.
- **Migração neuronal**: glia radial serve de guia.
- **Poda sináptica**: reduzir de ~50% excesso até adolescência/adulthood.
- **Períodos críticos**: janelas em que experiência molda circuitos irreversivelmente (visão binocular, linguagem materna, attachment).
- **Mielinização**: até ~25 anos em pré-frontal — imaturidade de controle em adolescência tem base biológica.
- **Envelhecimento**: atrofia seletiva; hipocampo e pré-frontal vulneráveis.

## Gut-Brain Axis

- Nervo vago.
- Microbioma intestinal produz/modula neurotransmissores (serotonina em parte é gut).
- Sistema imune interface.
- Probióticos, dieta afetam humor/cognição — evidência crescente mas ainda ruidosa.

## Inflamação e Cérebro

- Citocinas inflamatórias atravessam BBB ou sinalizam via vago.
- Depressão associada a marcadores inflamatórios (cytokine hypothesis).
- COVID-19 e "brain fog": neuroinflamação possível.

## Como Medimos

### Técnicas

- **EEG**: eletroencefalograma; não-invasivo; alta resolução temporal (ms), baixa espacial (cm). Sleep staging, epilepsia, pesquisa.
- **MEG**: magnetoencefalografia; melhor espacial que EEG.
- **fMRI**: BOLD signal (hemodinâmica, proxy de atividade). Alta espacial (mm), baixa temporal (segundos). Estrutura + função.
- **PET**: metabolismo, receptores específicos com ligantes.
- **Single-cell recording**: eletrodo intracraniano (animais, humanos em casos clínicos).
- **Two-photon imaging, calcium imaging** (animais): ver neurônios individuais.
- **Optogenetics**: controlar neurônios geneticamente modificados com luz. Revolucionou causalidade em pesquisa.
- **DBS (Deep Brain Stimulation)**: clínico em Parkinson, OCD, depressão refratária.
- **TMS, tDCS**: modulação não-invasiva; efeito real, modesto.

### Limites

- fMRI ≠ atividade neuronal direta; BOLD é indireto.
- Resolução temporal e espacial são trade-off.
- Estudos pequenos + muitos voxels → false positives (scandal "dead salmon" de Bennett et al.).

## Mitos Comuns

- **"Usamos 10% do cérebro"**: falso. Usamos todo, em horários diferentes.
- **"Hemisfério esquerdo/direito"**: lateralização é real mas nuançada; "creative right brain" é simplificação popular.
- **"Triune brain" (reptilian/mammalian/cortical)**: MacLean; simplificação inválida — evolução não funciona em camadas sobrepostas estanque.
- **"Brain games treinam QI"**: transferência é limitada.
- **"Brain fog se resolve com suplemento X"**: maioria sem evidência.

## Aplicações

### Aprendizado

- **Spaced repetition**: intervalos crescentes. Ebbinghaus + software (Anki, SuperMemo). Ver `CONTINUOUS_LEARNING.md`.
- **Interleaving**: misturar tópicos > blocar.
- **Retrieval practice**: testar > reler.
- **Sleep em consolidação**: dormir antes de exame/apresentação.
- **Desafio Goldilocks**: zona de dificuldade ótima.

### Produtividade

- **Ritmos ultradianos** (~90 min): alternar foco e pausa real.
- **Evitar multitasking**.
- **Bright light de manhã, dim light à noite**: circadian.
- **Exercício**: BDNF↑, neurogênese↑, cognição.

### Saúde mental

- **Exercise, sleep, social, nutrition** têm mais evidência em depressão leve-moderada que muita terapia alternativa.
- **CBT, ACT, EMDR, DBT** com evidência em RCTs.
- **Farmacologia** (SSRIs, SNRIs, etc.) variável; resposta individual; efeitos colaterais reais.
- **Meditação** (atenção plena): efeitos modestos mas consistentes (Brewer). Ver `MEDITATION.md`.
- **Psicodélicos** (psilocibina, MDMA assistido): evidência crescente em depressão refratária, PTSD. Campo em expansão regulatória.

### Dependência

Entender wanting/liking + predição de recompensa + plasticidade informa tratamento. Comportamental + farmacologia (naltrexona, buprenorfina, metadona) + social support.

## Neurociência vs IA

Redes neurais artificiais inspiradas, mas **não são o que o cérebro faz**:

- Cérebro: spiking, estocástico, massivamente paralelo, 20W, plástico, online, embodied.
- Deep learning: síncrono, gradient descent, megawatts, treino offline, disembodied.
- Convergências existem (representações parecidas emergem), divergências são grandes.

Neuromorphic computing (Loihi, SpiNNaker) tenta aproximar. Interfaces cérebro-máquina (Neuralink, Synchron, precedente Utah array em pacientes paralisados) avançam.

## Recursos

- **"The Brain: The Story of You"** — David Eagleman (pop).
- **"Why We Sleep"** — Matthew Walker (com ressalvas críticas).
- **"The Body Keeps the Score"** — Bessel van der Kolk (trauma).
- **"Principles of Neural Science"** — Kandel et al. (referência graduada).
- **"Neuroscience: Exploring the Brain"** — Bear, Connors, Paradiso (undergrad).
- **"Behave"** — Robert Sapolsky.
- **"Seven and a Half Lessons About the Brain"** — Lisa Feldman Barrett.
- **Andrew Huberman podcast** — popular mas use com filtro; alguns episódios oversell.
- **Stanford's openly available lectures (Sapolsky Human Behavioral Biology)**.

## Princípios

1. **Níveis importam**. Não confunda fenômeno molecular com cognitivo.
2. **Correlação ≠ causalidade** em imagem. "Área X ativa durante Y" não diz que X causa Y.
3. **Plasticidade ao longo da vida**. Cérebro muda; idade diminui mas não zera.
4. **Sono, exercício, social, nutrição**: as quatro intervenções consistentes em evidência para saúde cerebral.
5. **Farmacologia é arte incompleta**. Mesmo em estado da arte, resposta é idiossincrática.
6. **Mitos pop endurecem**. Vigiar próprias crenças sobre cérebro.
7. **Emoção e cognição não são separáveis**. Decisão "racional" sempre envolve emoção.
8. **Individualidade real**. Populações têm padrões; indivíduos desviam muito. Clinical trial p < 0.05 não é garantia para você.
