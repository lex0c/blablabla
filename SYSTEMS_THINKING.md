# Pensamento Sistêmico

**Systems thinking** é a prática de entender como **partes interagem para formar um todo** — e como o comportamento do todo é frequentemente contra-intuitivo, emergente e resistente a intervenções lineares. Origem: cibernética (Wiener), dinâmica de sistemas (Forrester no MIT nos anos 60), e a síntese de Donella Meadows. Difere de análise cartesiana (quebrar em partes, estudar isoladamente) ao exigir **pensar em relacionamentos, feedback, estoques, fluxos e limiares**.

Complementa `HOLISTIC_VIEW.md`, `COMPUTER_THEORY.md`, `PARADOX_ANALYSIS.md`, `LINEARITY.md`, `EVOLUTION.md`.

## Elementos Básicos

Donella Meadows (*Thinking in Systems*, 2008) cataloga três componentes:

1. **Elementos**: partes. Podem ser tangíveis (peças) ou intangíveis (reputação).
2. **Interconexões**: relações entre elementos. Frequentemente fluxos de informação.
3. **Função ou propósito**: o que o sistema faz, não o que deveria fazer.

**Regra observacional**: o propósito real de um sistema é mais bem inferido do comportamento observado do que da declaração oficial. Universidade que demite maus professores mas mantém trend de notas altas revela propósito diferente do que diz.

## Estoques e Fluxos (Stocks and Flows)

### Estoque

Acumulação medida em um momento: conta bancária, população, estoque de combustível, dívida técnica, goodwill.

### Fluxo

Taxa de mudança do estoque: ingresso ou egresso por unidade de tempo. Contrato por mês, pessoas nascendo, water drained, bugs introduced.

### Regra da banheira

- Estoque sobe se inflow > outflow.
- Estoque cai se outflow > inflow.
- Drenar rápido requer **parar inflow** ou **aumentar outflow** — frequentemente ambos.

### Implicações contra-intuitivas

- **Atrasos** em ajuste: CO₂ em atmosfera continua subindo mesmo reduzindo emissões.
- **Inércia**: grandes estoques respondem lentamente.
- **Dívida compondo**: pequenos fluxos constantes viram estoques enormes (juros compostos, débito técnico).
- **Percepção linear engana**: temos intuição pobre para exponencial.

Ver `LINEARITY.md` para conceito aparentado.

## Loops de Feedback

### Balancing (Negativo)

Estabiliza em direção a uma meta: termostato, manutenção de nível da piscina, regulação de glicose. Um sinal (erro) dispara ação corretiva.

Propriedades:

- Resistem a perturbação.
- Aparecem como homeostase.
- Atraso no loop pode causar **oscilação** (overshoot + correção atrasada + undershoot).

### Reinforcing (Positivo)

Amplifica mudança: juros compostos, efeito de rede, viralização, degradação acelerada. Ação gera mais da mesma ação.

Propriedades:

- Crescimento ou colapso exponencial.
- Sem contrabalanço, termina em limite físico ou quebra do sistema.
- Invertidos, viram "círculos viciosos" (death spirals).

### Sistemas reais

Combinam múltiplos loops. Comportamento emergente surge da interação. Diagrama de laços causais (CLD, causal loop diagram) é ferramenta padrão.

## Atrasos (Delays)

Toda relação causal tem tempo. Atrasos:

- **Perception delay**: percebo depois que aconteceu.
- **Response delay**: decido agir depois de perceber.
- **Action delay**: ação surte efeito depois de executada.
- **Transmission delay**: efeito leva a consequência.

Atrasos criam **instabilidade** em loops balancing. Dirigir carro com 3-segundos de atraso entre virar o volante e o carro responder produz zigue-zague.

Regra: **atraso + feedback = oscilação ou instabilidade**. Projetar respostas proporcionais ao atraso — ou reduzir atraso.

## Limites (Bounds) e Não-Linearidade

- **Limites físicos**: capacidade de aquário, trabalho, produção.
- **Saturação**: rendimentos decrescentes.
- **Breakpoints**: sistema muda comportamento qualitativamente ao cruzar limiar (fase transition, tipping points).

Ver `LINEARITY.md` — a maioria dos sistemas interessantes é não-linear; intuição linear quebra.

## Arquétipos de Sistema

Padrões recorrentes (Senge, popularizado por *The Fifth Discipline*). Reconhecê-los acelera diagnose.

### 1. Limites ao Crescimento (Limits to Growth)

Loop reinforcing alimenta crescimento + loop balancing emerge eventualmente (capacidade, recurso, mercado).

- **Exemplo**: startup escala por word-of-mouth (reinforcing); depois atinge saturação de mercado (balancing).
- **Intervenção eficaz**: antecipar e relaxar o constraint, não empurrar mais forte no reinforcing.

### 2. Shifting the Burden

Dois loops balancing: sintomático (rápido, alívio) e fundamental (lento, resolve raiz). Shifting acontece quando repetidamente escolhemos o sintomático — até capacidade fundamental atrofia.

- **Exemplo**: beber para relaxar vs resolver causa do stress; pagar juros com cartão vs reduzir gasto; patches em vez de refactor.
- **Sinal**: alívio breve; problema maior depois; dependência cresce.

### 3. Eroding Goals

Meta cede quando realidade não atinge. "Suficientemente bom" recalibra para baixo.

- Qualidade de produto, standards de segurança, performance esperada.
- Antídoto: ancorar metas a referência externa, não auto-histórica.

### 4. Success to the Successful

Dois agentes competem por recurso. Quem vai melhor recebe mais recursos, vai melhor ainda.

- Rich-get-richer. Matthew effect.
- Intervenção: canalizar recurso deliberadamente, ou quebrar ligação entre sucesso atual e recurso futuro.

### 5. Escalation

Cada agente aumenta pressão porque o outro aumentou. Arms race.

- Guerra fria, publicidade, litígio.
- Antídoto: compromisso crível, normas, unilateralmente desescalar.

### 6. Tragedy of the Commons

Recurso compartilhado, cada usuário otimiza local → colapso coletivo (ver `GAME_THEORY.md`).

- Oceanos, atmosfera, aquífero, banda de Wi-Fi de dormitório.
- Governança (Elinor Ostrom: comunidades resolvem sob condições).

### 7. Fixes That Fail (ou Fixes That Backfire)

Solução rápida cria problema maior depois.

- Pesticidas criam resistência; analgésicos mascaram sintoma que era diagnóstico; jogar dívida técnica para frente.

### 8. Growth and Underinvestment

Crescimento cria demanda por capacidade; capacidade não acompanha; qualidade cai; demanda cai. Startup que não investe em infra em escala.

### 9. Accidental Adversaries

Parceiros cooperam, cada um sub-otimiza, relação deteriora, cada um blame o outro → inimigos apesar de inicialmente aliados.

- Pair com supplier sem transparência.

## Leverage Points (Meadows)

Lista clássica do "onde intervir em sistema" em ordem **crescente de poder**:

1. **Constantes, parâmetros, números** (tarifas, subsídios) — mais visível, menos poderoso.
2. **Buffers e estoques** — tamanhos relativos.
3. **Estrutura de estoques e fluxos** — o próprio grafo.
4. **Delays** — atrasos no sistema.
5. **Balancing loops** — força dos loops corretivos.
6. **Reinforcing loops** — força de laços amplificadores.
7. **Fluxo de informação** — quem sabe o quê, quando. Barato e poderoso.
8. **Regras do sistema** — incentivos, punições, normas.
9. **Self-organization** — capacidade de adicionar/mudar estrutura.
10. **Goals** — propósito do sistema.
11. **Paradigma** — mindset fora do qual sistema existe.
12. **Poder de transcender paradigma** — capacidade de trocar lentes.

Paradoxo: **os mais poderosos são os mais difíceis de mudar**. Mas mexer em #1 (ajustar parâmetro) frequentemente inútil se problema é em #10-11.

## Comportamento Emergente

Propriedades que o todo tem mas nenhum componente tem. Exemplos:

- Consciência (neurônios não são conscientes).
- Mercado (indivíduos trocando não "sabem" equilíbrio).
- Inteligência coletiva (formigas individuais são burras).
- Temperatura (moléculas não têm).
- Internet (routers individualmente não "são" internet).

Emergência é **resistente a reducionismo** — entender peças não prediz todo. Por isso systems thinking.

## Dinâmica de Sistemas (System Dynamics)

Disciplina quantitativa fundada por Jay Forrester (MIT). Modela estoques, fluxos, loops como ODEs. Simula trajetórias, analisa sensibilidade.

### Ferramentas

- **Vensim**, **Stella/iThink**, **AnyLogic** (comerciais).
- **SimPy**, **BPTK_Py**, **SDWIN**.
- **Loopy** (ncase.me/loopy): CLDs interativos simples.
- **PySD, Bptk**: ODE dynamics em Python.

### Industry applications

- **Previsão populacional / epidemia** (SIR models).
- **Supply chain** (Bullwhip effect).
- **Climate models** (Meadows' Limits to Growth, World3).
- **Organizational dynamics** (Forrester's industrial dynamics).

## Causa Raiz vs Sintoma

Ver `BRUTE_FORCE_VS_ROOT_CAUSE.md`. Systems thinking amplia:

- **5 Whys** (Toyota): perguntar "por quê" em cadeia revela nível mais profundo.
- **Ishikawa / Fishbone**: decomposição de causas em categorias.
- **Causal loop**: mapeamento visual revela loops que 5 whys perde.
- **Event → pattern → structure → mental model** (iceberg): eventos são ponta; estruturas e modelos dominam.

## Anti-Padrões de Raciocínio Sistêmico

1. **Linear mental model em sistema não-linear**: dobrar input não dobra output.
2. **Negar delay**: ação imediata de ver efeito imediato ignora lag.
3. **Culpar pessoa em problema estrutural**: troca de gerência sem mudar incentivos repete comportamento.
4. **Solução ponto-local para problema sistêmico**: fixa um nó; pressão desloca.
5. **Reducionismo em sistema emergente**: "a team is just people" — não. Interação gera algo novo.
6. **Confundir correlação com causa em sistema complexo**.
7. **Otimizar subsistema em detrimento do todo**: eficiência local, ineficiência global.
8. **Ignorar boundary**: definir recorte arbitrário exclui exatamente os feedbacks relevantes.

## Mapeamento de Sistemas — Ferramentas

### Causal Loop Diagram (CLD)

Variáveis como nós; setas positivas/negativas entre elas; loops identificados. Good for narrativa qualitativa.

### Stock and Flow Diagram

Estoques como retângulos, fluxos como setas grossas com válvulas. Quantifica; permite simulação.

### Behavior Over Time (BOT)

Gráficos estilizados de como variáveis evoluem. Exponential growth, oscillation, S-curve, overshoot and collapse.

### Iceberg

Visualização:

- **Events**: o que aconteceu?
- **Patterns**: o que acontece recorrentemente?
- **Structure**: que loops sustentam patterns?
- **Mental models**: que beliefs sustentam structure?

Intervir em eventos é reativo; em estrutura, duradouro.

## Exemplos Aplicados

### Pós-morte de incidente

Sistema thinking evita blame: "Joe errou" → "por que sistema permitiu?". Cultura blameless vem daí. Ver `INCIDENT_RESPONSE.md`.

### Bullwhip effect (supply chain)

Cliente aumenta 10%; distribuidor aumenta 15%; wholesaler 25%; fábrica 40%. Amplificação via atrasos de informação + precaução. Mitigar: share info direto, reduzir atrasos.

### Debito técnico

Loop reinforcing: pressão → shortcuts → código ruim → mais pressão (bugs, lentidão) → mais shortcuts. Loops balancing fracos (code review, tests). Intervenções:

- **Estrutural**: boundary sobre velocidade vs qualidade.
- **Informacional**: measurable cost of tech debt — dedicate capacity formally.
- **Paradigmática**: leader valoriza qualidade tanto quanto entrega.

### Gorjeta (escalation)

Restaurante A começa a incluir 10%; B segue; C 12%; A responde 15%. Em anos, 20% virou norma. Sem coordination, nenhum quer descer (perde staff).

### Pandemia

- Loops reinforcing: R_0 > 1, casos crescem; overstretched health system piora outcomes → mais mortes.
- Loops balancing: medidas (lockdown, vacinação); sistemas de saúde reagem, behavior muda.
- Atrasos: infecção → sintoma (5d), hospitalização → morte (semanas), política → mudança de comportamento (dias).
- Limites: hospital capacity, vacina supply.
- Paradigmas: indivíduo vs coletivo.

Modeling disso guiou políticas; modelos melhores levaram a decisões melhores.

### Sistema financeiro

- Bolhas: reinforcing (preço sobe → compra mais → preço sobe) até limite físico ou confiança quebra.
- Crises: balancing catastrófico, frequentemente overshoot (bank runs).
- Regulação: muda incentivos, quebra loops, adiciona delays ou buffers.

### Climate

- Reinforcing: albedo feedback (gelo derrete, absorção de calor sobe).
- Balancing: vegetação, oceanos absorvem CO₂ (até limite).
- Delay: séculos entre emissão e efeito total.
- Breakpoints: permafrost thaw, AMOC shutdown.

## Limites do System Thinking

- **Não é bala de prata**. Modelo é simplificação; sempre.
- **Fácil virar diagrama confuso** sem ação.
- **Intervenção em leverage alto é politicamente custosa**.
- **Dados frequentemente ruins** em sistemas sociais.
- **Reducionismo continua útil** onde se aplica.

## Recursos

- **"Thinking in Systems: A Primer"** — Donella Meadows. Porta de entrada.
- **"The Fifth Discipline"** — Peter Senge.
- **"Industrial Dynamics"** — Jay Forrester.
- **"Limits to Growth"** — Meadows et al.
- **"The Systems Bible"** — John Gall (humorístico, perspicaz).
- **"Systems Thinking: Managing Chaos and Complexity"** — Jamshid Gharajedaghi.
- **"An Introduction to General Systems Thinking"** — Gerald Weinberg.
- **MIT System Dynamics curso (Sterman)** — excelente.

## Princípios

1. **Olhe relacionamentos, não objetos**. O que faz o sistema comportar-se assim não é cada parte, é a rede.
2. **Mapeie antes de intervir**. Ação sem modelo é superstição sob outro nome.
3. **Espere resistência**. Sistemas tendem a preservar padrões (policy resistance).
4. **Intervenção alta-leverage é rara e difícil**. Recompense persistência em identificá-las.
5. **Comportamento emerge de estrutura, não de personagens**. Trocar pessoas raramente resolve.
6. **Delays importam**. Paciência informada, não impaciência.
7. **Boundary matters**. Definir sistema inclui definir o que é "fora" — às vezes, é aí que está o problema.
8. **Todo modelo está errado; alguns são úteis** (Box). Use com humildade.
