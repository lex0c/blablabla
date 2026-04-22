# Teoria dos Jogos

**Teoria dos jogos** estuda decisões em que o resultado de cada agente depende das escolhas dos outros. É base formal de economia, ciência política, biologia evolutiva, design de protocolos, cripto-economia e, crescentemente, de segurança de IA. Saber o vocabulário — equilíbrio, dominância, estratégia mista, mechanism design — afia pensamento sobre incentivos em qualquer sistema com múltiplos agentes.

Complementa `PROSPECT_THEORY.md`, `GROUPTHINK_BIAS.md`, `PARADOX_ANALYSIS.md`, `CRITICAL_THINKING.md`.

## Elementos de um Jogo

1. **Jogadores**: N agentes (pessoas, firmas, estados, algoritmos).
2. **Estratégias**: conjunto de ações disponíveis a cada jogador. Pode ser pura (uma ação) ou mista (distribuição sobre ações).
3. **Payoffs** (recompensas): número real por combinação de estratégias.
4. **Informação**: o que cada jogador sabe — perfeita, imperfeita, assimétrica.
5. **Timing**: simultâneo, sequencial, repetido.

Representações:

- **Forma normal (matriz)**: para jogos simultâneos.
- **Forma extensiva (árvore)**: para jogos sequenciais.

## Tipos de Jogo

- **Cooperativos** vs **não-cooperativos**: se acordos vinculantes são possíveis.
- **Soma zero** vs **soma não-zero**: se ganho de um = perda do outro.
- **Informação completa** vs **incompleta**: se payoffs e regras são conhecidos.
- **Finito** vs **infinito**.
- **Estático** (one-shot) vs **dinâmico** (repetido, sequencial).
- **Determinístico** vs **com aleatoriedade**.

## Equilíbrio de Nash

Conceito central (John Nash, 1950). **Perfil de estratégias onde nenhum jogador melhora mudando unilateralmente.**

Nash provou: todo jogo finito tem ao menos um equilíbrio em estratégias mistas.

**Propriedades importantes**:
- Equilíbrio não é necessariamente único.
- Nem sempre é Pareto-ótimo (ex.: Dilema do Prisioneiro).
- Nem sempre é estável dinamicamente.

## Dominância

- **Estratégia A domina estritamente B**: para toda estratégia dos outros, A > B.
- **Estratégia dominada**: existe outra que sempre é melhor ou igual.
- **Iterated elimination**: remover dominadas sucessivamente simplifica jogos; se sobra única ação por jogador → *dominance solvable*.

Princípio: jogador racional nunca escolhe estratégia estritamente dominada.

## Jogos Clássicos

### Dilema do Prisioneiro

| | Coopera | Trai |
|---|---|---|
| **Coopera** | (-1, -1) | (-10, 0) |
| **Trai** | (0, -10) | (-5, -5) |

- **Nash**: (Trai, Trai) — dominance solvable.
- **Pareto-ótimo**: (Coopera, Coopera).
- Racionalidade individual leva a resultado coletivamente inferior — **paradigma da cooperação**.

Generalizações:
- **Tragédia dos comuns** (Hardin): N jogadores explorando recurso compartilhado.
- **Free-rider** em bens públicos.
- **Race to the bottom**: regulação, salários, tributação internacional.

### Jogo do Galinha (Chicken)

Dois veículos em rota de colisão — quem desvia perde face; ambos seguirem = morte.

| | Desvia | Segue |
|---|---|---|
| **Desvia** | (0, 0) | (-1, +1) |
| **Segue** | (+1, -1) | (-10, -10) |

Dois Nash em estratégias puras (Desvia/Segue e Segue/Desvia). **Brinkmanship**: comprometer-se publicamente a não desviar modifica expectativa. Modela crises nucleares, disputas sindicais, impasses políticos.

### Coordenação / Stag Hunt (Caçada ao Cervo)

Rousseau, adaptado por Skyrms. Dois caçadores: juntos pegam o cervo (grande); individualmente só lebre (pequena).

| | Cervo | Lebre |
|---|---|---|
| **Cervo** | (4, 4) | (0, 3) |
| **Lebre** | (3, 0) | (3, 3) |

Dois Nash: (Cervo, Cervo) risk-dominant não é; (Lebre, Lebre) é. **Paradigma da confiança e risco**.

### Batalha dos Sexos

Coordenação com preferências conflitantes. Dois Nash puros + um misto.

### Rock-Paper-Scissors

Zero-sum, sem Nash puro. **Nash misto**: cada ação com prob 1/3.

### Matching Pennies

Jogo simples zero-sum com Nash misto único.

### Ultimatum Game

Jogador 1 propõe divisão de R$ 100; Jogador 2 aceita ou rejeita (nada). Solução teórica (backward induction): J1 oferece mínimo, J2 aceita. **Na prática**: ofertas baixas rejeitadas — equidade pesa. Experimentos cross-cultural são clássicos (Henrich et al.).

### Dictator Game

Variante: J2 não pode rejeitar. Testa altruísmo puro.

### Trust Game

J1 envia X de R$ 100; experimentador triplica; J2 decide quanto devolver. Mede confiança + reciprocidade.

### Cournot vs Bertrand (economia)

- **Cournot**: firmas escolhem quantidades simultâneas → duopólio próximo de monopólio.
- **Bertrand**: firmas escolhem preços → competição quase perfeita em equilíbrio (duas firmas bastam).

Paradoxo Bertrand: pequena diferença em modelo muda conclusão de mercado.

## Estratégia Mista

Distribuição de probabilidade sobre ações. Usada quando não há Nash puro (rock-paper-scissors) ou quando randomizar é ótimo (poker, tênis, penalty kicks em futebol — goleiro vs chutador).

Intuição do equilíbrio misto: cada jogador escolhe mix que torna o outro indiferente — caso contrário, o outro desvia para ação dominante.

## Jogos Sequenciais

### Forma Extensiva

Árvore com nós de decisão. Linhas tracejadas indicam *information sets* (jogador não sabe em qual nó está).

### Backward Induction

Resolver do fim para o começo. Solução: **subgame perfect equilibrium** — Nash em cada subjogo.

Limitação: assume racionalidade comum ao longo da árvore, hipótese forte.

### Centipede Game

Dois jogadores alternam; cada turno, o pot cresce. Equilíbrio backward induction: parar no primeiro turno. Experimental: jogadores continuam muitos turnos.

### Ameaças Críveis e Commitment

Ameaça só dissuade se **crível**. Commitment (burning bridges, reputação, contratos) torna ameaça crível ao custo de flexibilidade. Schelling é a referência.

## Jogos Repetidos

Mesma interação ocorre muitas vezes. Muda dramaticamente equilíbrios.

### Folk Theorem

Em jogos repetidos indefinidamente com paciência suficiente (desconto futuro ≥ δ), quase qualquer payoff individualmente racional pode ser sustentado em equilíbrio — incluindo cooperação em Dilema do Prisioneiro.

**Mecanismos**: estratégias de gatilho (trigger strategies) que punem desvio via retaliação.

### Tit-for-Tat

- Começa cooperando; copia a última jogada do oponente.
- Axelrod tournaments (1984) mostraram alta robustez.
- Propriedades: nice (não inicia traição), provocada (retalia), perdoadora (volta a cooperar), clara.

Variantes: **Tit-for-Two-Tats** (mais permissivo), **Generous Tit-for-Tat** (perdoa com probabilidade).

### Grim Trigger

Coopera até primeiro desvio; traí para sempre. Estratégia severa; sustenta cooperação com ameaça crível.

### Reputação e Perfeição

Modelos de Kreps-Milgrom-Roberts-Wilson: reputação permite cooperação mesmo em jogo finito com probabilidade pequena de tipo "cooperador irracional".

## Jogos de Informação Incompleta

Harsanyi (1967-68) introduziu **tipos**: cada jogador tem um tipo privado; distribuição comum.

**Bayesian Nash Equilibrium**: equilíbrio em estratégias condicionais ao tipo.

### Signaling

Jogador informado **sinaliza** tipo por ação custosa.

- **Spence (1973)**: educação como signaling de produtividade no mercado de trabalho.
- **Qualidade em produto**: garantia estendida sinaliza qualidade.
- **Handicap principle** (Zahavi) em biologia: caudas exuberantes de pavões só se mantêm se sobrevivência for autêntica.

### Screening

Não-informado desenha contratos que induzem informado a revelar tipo.

- **Seguros**: menus com deductibles diferentes separam risk types.
- **Tarifas em duas partes**.

### Cheap Talk

Comunicação sem custo. Crabb-Sobel mostraram: com interesses parcialmente alinhados, talk transmite informação; totalmente opostos, não.

## Mechanism Design

Projetar regras do jogo para alcançar resultado desejado, sabendo que jogadores são racionais e têm informação privada.

### Revelação

**Revelation Principle** (Myerson): qualquer outcome implementável é implementável por mecanismo direto em que verdade é equilíbrio.

### Leilões

- **First-price sealed-bid**: bid sob envelope; maior ganha e paga o que bidou.
- **Second-price (Vickrey)**: maior ganha, paga o **segundo** bid. Truthful — bid óptimo é valor verdadeiro.
- **English (ascending open)**: clássico; equivale em valor a Vickrey.
- **Dutch (descending)**: preço cai; equivale a first-price.

**Revenue Equivalence Theorem**: sob certas premissas, receita esperada é mesma em todos os formatos truthful.

**Leilões de múltiplos itens**: muito mais complexo. FCC leilões de espectro usaram SMR (simultaneous multi-round). Milgrom/Wilson Nobel 2020.

### Matching

- **Gale-Shapley** (deferred acceptance): casa N jogadores estavelmente. Usado em residency medical, escolhas escolares (NYC), trocas de rins (Alvin Roth).
- **Stable matching**: nenhuma parelha prefere desfazer.

### VCG Mechanism

**Vickrey-Clarke-Groves**: truthful + eficiente em ambientes quasi-linear. Pagamentos = externalidade imposta a outros. Computacionalmente caro; nem sempre budget-balanced.

## Biologia Evolutiva

- **Evolutionarily Stable Strategy (ESS)** (Maynard Smith): estratégia que, se adotada por maioria da população, não pode ser invadida por mutante raro.
- **Hawks and Doves**: modelo de conflito animal.
- **Replicator dynamics**: modelo evolutivo, não racional — populações adaptam por frequência.

Conecta com `EVOLUTION.md`.

## Aplicações Contemporâneas

### Economia e negócios

- **Precificação competitiva**: Cournot/Bertrand.
- **Conluio (cartel)**: jogo repetido com detection + retaliação.
- **Entrada e dissuasão**: incumbente investe em capacidade excessiva (threat de guerra de preços).

### Segurança

- **Cyber**: atacantes vs defensores como jogo de Stackelberg (defensor move primeiro em alocação; atacante observa).
- **Detection evasion**: atacante escolhe ação observando política de detecção.
- **Patches**: timing ótimo como jogo contra atacantes.

### Redes e protocolos

- **Roteamento BGP**: cada AS é jogador.
- **Congestion control (TCP)**: folk theorem-like.
- **Spam filtering**: arms race.

### Cripto-economia

- **Bitcoin / PoW**: mining é jogo; segurança depende de incentive alignment.
- **PoS**: slashing como punição.
- **MEV (Maximal Extractable Value)**: jogo entre validators e usuários.
- **Token design**: staking, inflation, governance.

### IA / alignment

- **Principal-agent** entre humanos e agentes IA.
- **Cooperative games** em multi-agent systems.
- **Mechanism design para AI**: VCG-like para allocation de computação.
- **Alignment faking**: agentes estratégicos em treino (ver `AI_SECURITY.md`).

### Política e relações internacionais

- **Deterrence nuclear**: Schelling.
- **Escalation dynamics**.
- **Coalitions**: Shapley value divide ganhos.

### Voting theory

- **Arrow's Impossibility**: nenhum sistema de votação satisfaz simultaneamente un unrestricted domain, Pareto, IIA, não-ditadura.
- **Gibbard-Satterthwaite**: todo sistema determinístico é manipulável ou ditatorial.
- **Approval, Borda, ranked-choice, Condorcet**: trade-offs distintos.

### Negociação

- **Nash bargaining solution**: axiomático, maximiza produto de utility gains.
- **Rubinstein**: alternating offers, desconto → solução única.
- **Splitting the pie**: timing de urgência muda leverage.

## Conceitos Relacionados

### Coalition / Shapley Value

Shapley (1953): maneira canônica de dividir valor em jogos cooperativos, média sobre ordens de entrada. Axiomaticamente único satisfazendo eficiência, simetria, null player, aditividade.

Usos: fair division, feature attribution em ML (SHAP), accounting de contribuição.

### Correlated Equilibrium

Aumann (1974): generalização de Nash permitindo sinal correlacionado (semáforo em cruzamento coordena passagens). Mais permissivo, frequentemente mais eficiente.

### Minimax

Em jogo zero-sum de dois jogadores, minimax = maximin = valor do jogo. Von Neumann (1928). Base do algoritmo homônimo em xadrez/go pré-ML.

## Limitações

1. **Racionalidade é idealização**. Humanos e instituições desviam sistematicamente (ver `PROSPECT_THEORY.md`, `COGNITIVE_BIASES`).
2. **Common knowledge de racionalidade** frequentemente falha.
3. **Equilíbrios múltiplos**: teoria não prediz qual ocorre.
4. **Calibração de payoff**: quantificar utility é difícil.
5. **Dinâmicas de aprendizado** (como jogadores chegam ao equilíbrio) são subestudadas.

Behavioral game theory (Camerer) quer corrigir.

## Recursos

- **"Theory of Games and Economic Behavior"** — von Neumann & Morgenstern (1944). Fundador.
- **"Game Theory"** — Drew Fudenberg & Jean Tirole. Referência graduada.
- **"The Strategy of Conflict"** — Thomas Schelling.
- **"The Evolution of Cooperation"** — Robert Axelrod.
- **"Games and Decisions"** — Luce & Raiffa.
- **"An Introduction to Game Theory"** — Martin Osborne.
- **Yale Game Theory course** (Polak, Open Yale) — gratuito, excelente.

## Princípios

1. **Modele incentivos, não pessoas**. Sistema que depende de boa-fé é frágil; sistema alinhado a incentivos é robusto.
2. **Olhe mais longe que o equilíbrio**. Múltiplos equilíbrios, dinâmica de saída, informação assimétrica.
3. **Repetição muda jogo**. Interação contínua viabiliza cooperação onde one-shot falha.
4. **Signaling é caro, por isso funciona**. Sinal barato não transmite informação.
5. **Racionalidade comum é premissa, não fato**. Calibrar expectativa pelo jogador real.
6. **Mechanism design é engenharia de incentivos**. Desenhar regras > repetir "todos devem ser honestos".
7. **Cuidado com modelos**. Simplificação sempre; conclusões requerem humildade epistêmica.
