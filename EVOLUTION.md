# Evolução

**Evolução** é o processo que gera complexidade sem design central. Sua lógica — variação, seleção, herança — não é apenas teoria biológica: é algoritmo aplicável em genética, cultura, economia, software, IA, linguagem. Entender evolução é entender uma das poucas explicações robustas de **como algo interessante emerge de nada planejando**.

Complementa `SYSTEMS_THINKING.md`, `COMPLEXITY_SCIENCE.md` (se existir), `AI.md`, `GAME_THEORY.md`, `CHAOS_CREATIVITY.md`.

## O Algoritmo

Evolução darwiniana exige três ingredientes — nada mais:

1. **Variação**: indivíduos/entidades diferem entre si.
2. **Seleção**: diferenças afetam sobrevivência/reprodução (ou analogia).
3. **Herança**: filhos se parecem com pais (parcialmente).

Satisfeitos os três, você **terá** evolução. Em qualquer substrato — genes, memes, software, mercado, ideias.

Darwin (1859, *Origin of Species*) + Wallace + século de refinamento = **síntese moderna** (Mayr, Dobzhansky, Huxley 1930-40s).

## Mecanismos

### Mutação

Origem de variação — mudanças aleatórias. Em DNA: substituição, inserção, deleção, duplicação, inversão, translocação, gene doubling, polyploidy. Maioria silenciosa ou deletéria; raras são benéficas.

### Recombinação

Sexo embaralha alelos a cada geração. Acelera exploração de espaço de fenótipos. Explica parcialmente por que sexo existe apesar do "custo do sexo duplo".

### Seleção natural

Ambiente filtra. Subtipos:

- **Directional**: fenótipo extremo favorecido (ex.: pescoço de girafa).
- **Stabilizing**: fenótipo médio (peso de recém-nascido — muito pequeno morre, muito grande complica parto).
- **Disruptive**: dois extremos favorecidos; estágio em especiação.
- **Frequency-dependent**: vantagem depende de raridade. Base de polimorfismo persistente.

### Seleção sexual

Darwin (*Descent of Man*). Traços explicados não por sobrevivência mas por escolha do parceiro: cauda de pavão, canto de aves. Runaway selection (Fisher), handicap principle (Zahavi — ver `GAME_THEORY.md`).

### Deriva genética

Mudanças aleatórias em frequências alélicas. Dominante em populações pequenas. Pode fixar alelos neutros ou mesmo levemente deletérios.

### Fluxo gênico

Migração entre populações homogeneiza.

### Herança não-mendeliana

- Epigenética (metilação, histonas — herdável parcialmente).
- Horizontal gene transfer (bactérias, arqueas, vírus).
- Simbiose (mitocôndrias, cloroplastos — endosimbiose).

## Unidade de Seleção

Debate clássico: seleção age em quê?

- **Gene** (Dawkins, *The Selfish Gene*): genes são replicadores; indivíduos são veículos.
- **Indivíduo**: intuitivo; sobrevive ou morre.
- **Grupo / kin**: seleção de parentesco (Hamilton's rule): `rB > C` — altruísmo evolui se beneficia parentes o bastante. Explica abelhas, naked mole rats.
- **Multinível**: seleção simultânea em múltiplos níveis (célula, indivíduo, grupo).
- **Species**: Gould defendeu seleção inter-espécies; controverso.

Na prática, níveis coexistem — seleção gênica é matematicamente universal, mas níveis superiores têm poder explicativo.

## Speciação

Como uma linhagem vira duas:

- **Alopátrica**: isolamento geográfico → acúmulo de diferenças → incompatibilidade.
- **Peripátrica**: pequena subpopulação periférica.
- **Parapátrica**: distribuição contígua com gradiente.
- **Simpátrica**: mesmo espaço, divergência de nicho (cichlids em lagos africanos).

Isolamento reprodutivo é o critério (biological species concept); existem outras definições (morphological, phylogenetic, ecological).

## Coevolução

Duas linhagens evoluem em resposta mútua.

- **Predador-presa**: Red Queen — "correr para ficar no mesmo lugar".
- **Parasita-hospedeiro**: arms race imunológica.
- **Planta-polinizador**: mutualismo, frequentemente altamente específico.
- **Host-microbioma**: ecossistema em coevolução permanente.

Gene-culture coevolution em humanos: tolerância à lactose evoluiu com pastorileira.

## Evolução Neutra e Quase-Neutra

Kimura (1968, 1983): majoritária das mudanças moleculares são **neutras** — não selecionadas, fixadas por drift. Constraint-driven mais que adaptation-driven em muitas partes do genoma.

Implicação para evolução molecular: relógio molecular, filogenia; maior parte do genoma não é "adaptação" — é resíduo, repetição, reconstrução.

## Punctuated Equilibrium

Eldredge & Gould (1972): evolução frequentemente em bursts após longa estase, não gradual. Fossil record suporta em muitos táxons. Debate sobre prevalência; hoje: ambos ocorrem.

## Mass Extinctions

Eventos que reorganizam biodiversidade:

- **Ordoviciano-Siluriano** (~444 Ma).
- **Devoniano-Fameniano** (~372 Ma).
- **Permiano-Triássico** (~252 Ma, "the Great Dying", ~96% espécies).
- **Triássico-Jurássico** (~201 Ma).
- **Cretáceo-Paleogeno** (~66 Ma, K-Pg — impacto de Chicxulub, dinossauros não-aves).
- **Antropoceno / 6ª extinção** (em curso).

Extinções liberam nichos, permitem radiações adaptativas.

## Árvore da Vida

- **Three domains** (Woese): Bacteria, Archaea, Eukarya.
- **LUCA**: Last Universal Common Ancestor, ~3.5-4 Ga.
- **Endosymbiosis**: mitocôndria e cloroplasto = bactérias engolidas.
- **Horizontal transfer** borrou árvore em procariontes; "web of life" mais que tree.

Humanos:

- Primates → Hominoidea → Hominidae → Homo → Homo sapiens.
- Divergência de chimpanzés ~6-7 Ma.
- Interbreeding com Neanderthal e Denisovan → genomas não-africanos atuais têm 1-2% Neanderthal.

## Molecular Evolution

- **DNA → RNA → protein** (central dogma, com exceções).
- **Código genético** quase universal, pequena variação mitocondrial.
- **Pseudogenes, retrotransposons**: fósseis genômicos.
- **Conserved regions**: sinal de constraint funcional.
- **Rates of evolution**: diferentes genes evoluem em diferentes taxas; histonas entre os mais conservados.
- **Filogenia molecular**: árvores baseadas em seqüências. MrBayes, RAxML, IQ-TREE, BEAST.

## Evo-Devo

**Evolutionary Developmental Biology**: como mudanças em desenvolvimento produzem evolução morfológica.

- **Hox genes**: arquitetura corporal conservada.
- **Toolkit genes**: pequeno conjunto reusado.
- **Regulação** (timing, local, amount) > sequência de proteína em muitos casos.
- **Heterochrony, heterotopy**: mudanças temporais/espaciais em expressão.

## Evolução Cultural

Memes (Dawkins como meme): unidade de informação cultural replicando por imitação. Variação (erro, criatividade) + seleção (fit a mente, ambiente) + herança (aprendizado social).

- **Dual inheritance**: genes + cultura.
- **Major transitions**: fogo, linguagem, escrita, imprensa, internet.
- **Gene-culture coevolution**.

Paralelos e dis-análogos:

- Cultura tem transmissão horizontal massiva; genes é vertical (majoritariamente).
- Lamarckiana em cultura (herança de aprendizado) — em genes, não.
- Cultura evolui muito mais rápido; cumulativo.

## Linguagem

Evolução linguística estudada há século. Mudança fonológica regular (leis de Grimm), morfológica, sintática, semântica. Árvores linguísticas análogas a filogenias.

Línguas emergem, ramificam-se, morrem. ~7000 vivas em 2025; metade em risco.

## Algoritmos Evolutivos

Aplicam o algoritmo darwiniano a problemas de engenharia:

### Componentes

1. **Encoding**: representar solução (string binária, lista de parâmetros, árvore).
2. **Population**: conjunto de candidatos.
3. **Fitness function**: avaliar qualidade.
4. **Selection**: escolher quem reproduz (roulette, tournament).
5. **Crossover**: combinar pais.
6. **Mutation**: variar.
7. **Iterate**.

### Variantes

- **Genetic Algorithms (GA)** — Holland, 1960s.
- **Evolution Strategies (ES)** — Rechenberg, Schwefel. Focadas em parâmetros reais, mutação gaussiana.
- **Genetic Programming (GP)** — Koza. Evolui programas (árvores).
- **Neuroevolution**: evolui redes neurais (NEAT, HyperNEAT).
- **CMA-ES**: state-of-the-art em optimization black-box.
- **Differential Evolution**.
- **Novelty search, quality-diversity**: recompensar exploração vs performance pura.

### Aplicações

- Design de antenas (NASA ST5).
- Discovery de algoritmos (AlphaEvolve em 2024: descobriu novos algoritmos de multiplicação de matrizes).
- Hyperparameter tuning.
- Robot controller evolution.
- Architecture search (NAS).

Evolução como algoritmo é **massivamente paralela** e **gradient-free** — aplicável onde gradientes não existem.

## Evolução em Software

- **Biblioteca velha acumula dependências, hacks, compat layers** — análogo a genoma acumulando lixo.
- **Forks** = especiação.
- **Refactoring** = reestruturação.
- **Sunsetting** = extinção.
- **Padrões virais** (React, Kubernetes): memes em ecossistema tech.
- **Linux vs BSD**: linhagens com LUCA comum, evolução paralela.

## Evolução de Vírus

- **High mutation rate** (RNA vírus).
- **Recombinação** em coinfecções.
- **Seleção**: immune evasion, transmission, replication.
- **SARS-CoV-2**: Omicron surge rápida com muitas mutações; cada wave = evolutionary rate acelerada.
- **HIV**: evolui dentro de um hospedeiro.

Modelagem evolutiva informa vaccine design, antiviral choice.

## Antibiotic Resistance

Seleção acelerada por uso indiscriminado. Superbugs (MRSA, XDR-TB). Evolução em tempo humano, visível em clínica.

Estratégias de contenção: stewardship, rotação, combinações, novas classes.

## Cancer

Evolução somática: células em tumor acumulam mutações; seleção favorece proliferação, invasão, resistência a terapia. Cada tumor é ecossistema evolutivo.

Terapias adaptativas (Gatenby): manter carga tumoral controlada para preservar clones sensíveis que suprimem resistentes — em vez de maximizar kill.

## Questões Filosóficas

### Teleologia

Evolução **não tem propósito**. "Serves-to" é narrativo, não causal. "Designed for" só em sentido metafórico.

### Progresso?

Evolução não "progride" — diversifica e explora. Complexidade não necessariamente cresce; tênias perderam sistema digestivo.

### Adaptationism

Gould & Lewontin (1979): crítica ao "tudo é adaptação". Muitos traços são **spandrels** — subprodutos sem função direta. Ou restrições desenvolvimentais, ou drift.

### Evolution vs creation

Cientificamente não há controvérsia. Socialmente, depende do país.

## Aplicações em Pensamento Geral

Usar lente evolutiva:

- **Por que essa instituição persiste?** Não porque é ótima — porque sobreviveu seleção específica.
- **Esse bug histórico faz sentido?** Frequentemente vestigial de estado anterior.
- **Por que esse hábito é comum?** Transmissão social + alguma utilidade/apelo.
- **Seleção está acontecendo aqui?** Procure variation/selection/inheritance em qualquer sistema com mudança temporal.

## Recursos

- **"The Selfish Gene"** — Richard Dawkins.
- **"The Extended Phenotype"** — Dawkins.
- **"The Greatest Show on Earth"** — Dawkins (evidências).
- **"Why Evolution Is True"** — Jerry Coyne.
- **"The Ancestor's Tale"** — Dawkins (tour por clades).
- **"Your Inner Fish"** — Neil Shubin (evo-devo).
- **"The Vital Question"** — Nick Lane (origem de vida complexa).
- **"Darwin's Dangerous Idea"** — Daniel Dennett (filosofia).
- **Talk.origins, University of Berkeley Evolution 101**.
- **"Genetic Algorithms in Search, Optimization, and Machine Learning"** — Goldberg.
- **"Evolutionary Computation"** — De Jong.

## Princípios

1. **Algoritmo, não doutrina**. Variação + seleção + herança suficientes.
2. **Sem telos**. Não há fim nem "destino". "Adaptado a" é descritivo, não prescritivo.
3. **Vestígios importam**. Sistemas persistem carregando história; função atual ≠ origem.
4. **Seleção é implacável em média, ruidosa em particular**. Drift e acaso influenciam.
5. **Co-evolução é a regra**. Nada evolui isolado.
6. **Coexistência de níveis**: gene, indivíduo, grupo, cultura.
7. **Analogias cuidadosas**: cultura/software/mercado evoluem sim, mas detalhes diferem. Importar mecanismos literalmente engana.
8. **Gradualismo + saltos**: dinâmica mista; nem contínuo liso nem pontual puro.
