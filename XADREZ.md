# Xadrez

Xadrez é um jogo de estratégia abstrata entre dois jogadores em tabuleiro 8×8. Tem ~1500 anos de refinamento (origem no chaturanga indiano, ~séc. VI), regras estáveis desde ~1475 (quando dama e bispo ganharam movimentação atual), e foi o primeiro grande problema de IA a ser "vencido" por máquina (Deep Blue vs Kasparov, 1997). Hoje, **engines** (Stockfish, Leela) jogam em nível superior ao de qualquer humano, e xadrez virou também laboratório de ML, teoria de jogos e educação.

## O Tabuleiro e as Peças

Tabuleiro 8×8, 64 casas alternadas em claras e escuras. Casa `h1` (canto inferior direito do jogador de brancas) é **sempre clara**.

Cada lado começa com 16 peças:

| Peça | Qtd | Movimento | Valor aproximado |
|---|---|---|---|
| Rei (K) | 1 | 1 casa em qualquer direção | ∞ (perder = derrota) |
| Dama (Q) | 1 | linhas, colunas, diagonais | 9 |
| Torre (R) | 2 | linhas e colunas | 5 |
| Bispo (B) | 2 | diagonais | 3 (≈3.25) |
| Cavalo (N) | 2 | "L": 2+1 | 3 |
| Peão (P) | 8 | 1 casa à frente; 2 do início; captura diagonal | 1 |

Valores de peça são heurísticas — em posição concreta, cavalo em outpost pode valer mais que torre passiva.

### Regras especiais

- **Roque (0-0 curto, 0-0-0 longo)**: rei + torre movem simultâneo. Condições: nenhum dos dois se mexeu antes, casas entre eles vazias, rei não em xeque nem passa por casa atacada.
- **Captura en passant**: peão que avança duas casas pode ser capturado como se tivesse avançado uma, só no lance seguinte.
- **Promoção**: peão que alcança 8ª fileira vira peça (dama tipicamente; raramente torre/bispo/cavalo — *underpromotion*).
- **50 movimentos**: sem captura nem movimento de peão em 50 lances → empate reivindicável.
- **Tripla repetição**: mesma posição 3× → empate reivindicável.
- **Material insuficiente**: K+K, K+B+K, K+N+K, K+BB+K (mesma cor) → empate automático.

### Resultado

- **Xeque-mate**: rei em xeque sem escape legal → vitória.
- **Empate (draw)**: afogamento (stalemate), acordo, triplas, 50 lances, material insuficiente.
- **Vitória por tempo, abandono (resign)**.

## Notação

### Algébrica (padrão FIDE desde 1980s)

Casas em coordenadas: `a1` a `h8`. Lance = peça + destino. Peão é implícito.

```
1. e4 e5
2. Nf3 Nc6
3. Bb5 a6      (Ruy Lopez)
```

Modificadores:
- `x` captura: `Nxe5`
- `+` xeque, `#` mate.
- `=` promoção: `e8=Q`
- `0-0`, `0-0-0` roque.
- Desambiguação: `Ngf3` (cavalo da coluna g), `R1e2` (torre da 1ª linha).

Anotações de qualidade:
- `!` bom lance
- `!!` excelente
- `?` questionável
- `??` erro grave
- `!?` interessante
- `?!` duvidoso

### FEN — Forsyth-Edwards Notation

Posição em string única:

```
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1
```

Campos: posição (fileiras 8→1) | lado a mover | direitos de roque | en passant | meio-lances sem captura/peão | número do lance.

### PGN — Portable Game Notation

Formato de arquivo para partidas completas:

```
[Event "Example"]
[White "Player1"]
[Black "Player2"]
[Result "1-0"]

1. e4 c5 2. Nf3 d6 3. d4 cxd4 4. Nxd4 Nf6 5. Nc3 a6 ...
```

Padrão de bancos de partidas, importação entre plataformas, análise.

## Fases da Partida

### Abertura

Primeiros ~10-15 lances. Objetivos:

- **Desenvolver** peças menores (cavalos e bispos).
- **Controlar o centro** (d4, d5, e4, e5).
- **Segurança do rei** (tipicamente roque curto).
- **Conectar torres**.
- **Não mover mesma peça duas vezes** sem razão.

### Meio-jogo

Cálculo tático + planejamento estratégico. Maior parte da luta.

### Final (endgame)

Poucas peças. Técnica pura frequentemente decide. Rei vira peça ativa (não mais em perigo de mate rápido).

Fronteira entre fases é fluida. "Endgame técnico" pode começar já com peças pesadas + passed pawn.

## Aberturas

Catalogadas em **ECO (Encyclopaedia of Chess Openings)**: códigos A00-E99 dividindo ~500 sistemas.

### Abertura de peão de rei (1.e4)

**Abertas e semi-abertas**:

- **Espanhola / Ruy Lopez** (1.e4 e5 2.Nf3 Nc6 3.Bb5) — profunda, estudada há séculos.
- **Italiana** (3.Bc4) — ressurgência moderna (Giuoco Piano renovado).
- **Gambito do Rei** (2.f4) — romântico, raro em alto nível.
- **Siciliana** (1...c5) — a resposta mais comum e combativa. Najdorf, Dragão, Sveshnikov, Taimanov, Scheveningen, Acelerada.
- **Francesa** (1...e6) — sólida, estrutural.
- **Caro-Kann** (1...c6) — sólida, sem fraqueza estrutural.
- **Pirc, Moderna, Alekhine, Escandinava**: hipermodernas/atípicas.

### Abertura de peão de dama (1.d4)

**Fechadas e semi-fechadas**:

- **Gambito da Dama** (1.d4 d5 2.c4) → aceito, recusado (QGD), Slav, Semi-Slav.
- **Indiana da Dama**, **Rei-Índia**, **Grünfeld**, **Nimzo-Indiana**, **Bogo-Indiana**.
- **Defesa Holandesa** (1...f5) — combativa.

### Outras primeiras

- **Inglesa** (1.c4) — flexível, transpõe muito.
- **Réti** (1.Nf3) — moderna.
- **Bird** (1.f4), **Larsen** (1.b3) — menos teóricas.

### Filosofia

Aberturas não "vencem" — igualam ou dão conforto tipicamente. Vitória emerge em meio-jogo/final. Amadores ganham tempo estudando endgames e táticas, não dezenas de variantes de abertura obscura.

## Táticas

Padrões que ganham material ou mate por curto cálculo. Reconhecê-los ao relance separa o clube forte do fraco.

### Padrões fundamentais

- **Garfo (fork)**: peça ataca duas (ou mais) peças ao mesmo tempo. Cavalo é rei dos garfos.
- **Cravada (pin)**: peça não pode se mover por expor peça mais valiosa. Absoluta (rei atrás) vs relativa.
- **Espeto (skewer)**: inverso da cravada — peça de maior valor na frente é atacada, movendo expõe peça menor.
- **Ataque descoberto**: mover peça revela ataque de peça atrás.
- **Xeque duplo**: descoberta + peça que move dá xeque simultâneo. Rei precisa mover (não dá para capturar ou bloquear os dois).
- **Remoção de defensor**: capturar peça que defende outra.
- **Desvio (deflection)**: forçar defensor a sair.
- **Atração (decoy)**: forçar peça a casa desfavorável.
- **Limpeza (clearance)**: tirar própria peça do caminho para dar mate.
- **Interceptação**: bloquear linha de peça adversária.
- **Zugzwang**: sem bom lance; qualquer movimento piora. Comum em finais.
- **Zwischenzug (in-between move)**: lance intermediário forte que quebra sequência esperada.
- **Raio-X**: peça ataca através de outra.

### Padrões de mate

- **Mate do corredor (back rank)**: torre/dama mata rei encurralado pela própria linha de peões.
- **Mate do pastor**: f7/f2 com bispo+dama (armadilha de iniciante).
- **Mate de Anastácia, Arábico, Epaulette, Boden, Legal, Morphy**: padrões clássicos nomeados.
- **Sufoco (smothered) / Mate de Philidor**: cavalo dá mate em rei cercado pelas próprias peças.

### Sacrifícios

- **Sacrifício real**: material perdido sem retorno material claro; busca ataque, iniciativa, ou restrição.
- **Sacrifício temporário / sham**: material volta em poucos lances.
- **Sacrifício posicional** (Petrosian, Karpov): por estrutura ou compensação de longo prazo.

## Estratégia

Plano de longo prazo além de cálculo tático.

### Elementos

1. **Atividade de peça**: peça controlando muitas casas > peça passiva.
2. **Estrutura de peões**: determina planos. Peões isolados, dobrados, passados, encadeados, avançados.
3. **Segurança do rei**: roque + peões próximos + peças defensivas.
4. **Controle do centro**: peões centrais ou peças pressionando centro.
5. **Espaço**: vantagem em casas controladas, especialmente na metade adversária.
6. **Diagonais abertas para bispos**: bispo bom vs mau (bloqueado pelos próprios peões).
7. **Colunas abertas para torres**: vale a pena disputar.
8. **7ª fileira**: torre na 7ª ataca peões adversários e restringe rei.
9. **Par de bispos**: dois bispos frequentemente valem mais que a soma (cobrem todas diagonais).
10. **Outposts**: casas na metade adversária defendidas por peão onde cavalo fica seguro.
11. **Iniciativa**: quem dita o ritmo.
12. **Profilaxia** (Nimzowitsch): prevenir planos adversários.

### Tipos de posição

- **Aberta**: poucos peões centrais, diagonais longas, peças ativas. Bispos brilham.
- **Fechada**: peões bloqueados, manobras. Cavalos brilham.
- **Semi-aberta**: intermediária.
- **Dinâmica vs estática**: dinâmica depende de timing; estática é estrutural.

### Princípios clássicos

- Steinitz: acumule pequenas vantagens.
- Nimzowitsch: profilaxia, superproteção, bloqueio.
- Capablanca: simplicidade; endgames por cálculo claro.
- Botvinnik: preparação científica.
- Tal: tático e especulativo (estudos posteriores mostraram cálculo profundo por trás).
- Karpov: restrição e tortura posicional.
- Kasparov: dinâmica agressiva + preparação teórica.

## Finais

Fase técnica. Alguns finais têm teoria completa; outros são heurística.

### Finais elementares

- **K+Q vs K**: mate em ≤10 lances. Técnica fundamental.
- **K+R vs K**: mate em ≤16 lances. "Técnica da rede".
- **K+BB vs K**: possível mas mais longo.
- **K+B+N vs K**: ~33 lances, técnica complexa.
- **K+N+N vs K**: **não** tem mate forçado (insuficiente).

### Finais de peões

- **Oposição**: conceito central. Rei defensor em oposição frontal segura.
- **Regra do quadrado**: rei adversário no quadrado do peão passado → alcança.
- **Lucena** e **Philidor**: posições canônicas em torre + peão vs torre.
- **Zugzwang** é onipresente.

### Finais de torre

Mais comuns em prática. "Torres atrás de peões passados" é regra de Tarrasch. Ativação da torre > material às vezes.

### Tablebases

Endgames com até **7 peças estão resolvidos** (Syzygy, Nalimov, Lomonosov). Posições têm resultado conhecido com **Distance to Mate** ou Distance to Zero exato. 8 peças é fronteira computacional atual (storage na ordem de petabytes).

Engines consultam tablebases em tempo real quando material ≤ 7.

## Relógios e Controles de Tempo

- **Clássico**: 2h+ por jogador. Campeonato Mundial 120'+60'.
- **Rápido**: 15-30 min.
- **Blitz**: 3-5 min.
- **Bullet**: 1-2 min.
- **Hyperbullet, ultrabullet**: <1 min.
- **Incremento (Bronstein, Fischer)**: +N segundos por lance, reduz apuros.

Xadrez rápido popularizou-se enormemente via plataformas online.

## Ratings

### Elo

Sistema de Arpad Elo (1960s). Novo rating:

```
R_new = R_old + K · (S - E)
```

- `S`: 1 vitória / 0.5 empate / 0 derrota.
- `E`: esperado pela diferença de rating: `E = 1 / (1 + 10^((Ropp - R)/400))`.
- `K`: fator (40 para novatos → 10 para top).

Vantagens: matemático, prediz resultados. Desvantagens: autoalimenta-se em populações isoladas (rating inflation/deflation).

### Variantes

- **FIDE** rating clássico (Elo): profissionais.
- **FIDE Rapid/Blitz**.
- **USCF, ECF, CBX** (nacionais).
- **Chess.com**: Glicko-2 customizado. Ratings inflados vs FIDE.
- **Lichess**: Glicko-2, relativamente calibrado.
- **Glicko, Glicko-2, TrueSkill**: melhorias sobre Elo com incerteza explícita (RD — rating deviation).

### Títulos FIDE

Por norma + rating mínimo:

- **CM (Candidate Master)** ~2200.
- **FM (FIDE Master)** ~2300.
- **IM (International Master)** ~2400 + 3 normas.
- **GM (Grandmaster)** ~2500 + 3 normas. ~2000 GMs no mundo.
- **WCM, WFM, WIM, WGM** (femininos) — critérios mais baixos, controverso mas mantido.
- **Super-GM**: uso informal para 2700+.
- **Top do mundo**: Magnus Carlsen pico ~2882 (2014). Ativo top 2810-2830.

## Campeonatos e Campeões Mundiais

Lista incompleta — marcos:

- **Wilhelm Steinitz** (1886-94): primeiro reconhecido, pai da escola posicional.
- **Emanuel Lasker** (1894-1921): 27 anos, psicologia + prática.
- **Capablanca** (1921-27): simplicidade, finais, "máquina de xadrez" pré-computador.
- **Alekhine** (1927-35, 1937-46): combinacional.
- **Botvinnik** (1948-57, 1958-60, 1961-63): escola soviética.
- **Tal** (1960-61): "Mago de Riga", sacrifício e intuição.
- **Petrosian** (1963-69): estratégia defensiva profilática.
- **Spassky** (1969-72).
- **Fischer** (1972-75): americano icônico, match da guerra fria 1972 em Reykjavik vs Spassky. Título perdido por inatividade.
- **Karpov** (1975-85): posicional, técnico.
- **Kasparov** (1985-2000): dinâmica + preparação com computador. Mais alto rating pré-Carlsen. 15 anos.
- **Kramnik** (2000-07): sistema Berlim contra Kasparov virou lendário.
- **Anand** (2007-13): indiano, escola universal.
- **Carlsen** (2013-23): técnica universal, dominou uma década, renunciou a defender título (2023).
- **Ding Liren** (2023-24): primeiro chinês.
- **Gukesh Dommaraju** (2024-): indiano, aos 18 anos mais jovem campeão mundial.

Cisma 1993-2006: PCA vs FIDE criou duas trilhas; reunificação em 2006 (Kramnik-Topalov).

## Motores (Engines)

### História

- **Turk mecânico** (Kempelen, 1770): fraude engenhosa com humano dentro.
- **Torres y Quevedo** (1912): autômato real mate K+R vs K. Primeiro "engine" mecânico genuíno.
- **Shannon** (1950): paper propondo busca + avaliação.
- **Turing**: primeiro programa (sem computador para rodar!).
- **Anos 70-80**: Belle, Deep Thought, Fritz.
- **Deep Blue** (IBM 1997): derrota Kasparov em match; marca histórica.
- **Stockfish** (open source, desde 2008): engine clássica (alfa-beta + avaliação humana) mais forte.
- **AlphaZero** (DeepMind 2017): self-play com Monte Carlo Tree Search + rede neural; aprendeu xadrez do zero em 9h, derrota Stockfish 8. Paradigma inovador.
- **Leela Chess Zero (Lc0)**: open source, inspirado em AlphaZero, competitivo.
- **Stockfish NNUE** (2020+): Stockfish incorpora rede neural (NNUE — Efficiently Updatable NN) para avaliação; salto de força. Stockfish 16+ dominante hoje.

### Arquitetura clássica

- **Busca**: alfa-beta minimax com poda (null-move, late move reductions, futility, razoring, aspiration windows).
- **Avaliação**: função quantifica posição. Histórica: material + PST + king safety + pawn structure + mobility. NNUE substituiu função manual por rede pequena.
- **Hash tables (transposition tables)**: cache de posições avaliadas.
- **Quiescence search**: estende capturas e xeques para evitar horizon effect.
- **Extensions** em linhas críticas (xeque, recaptura, passed pawns).
- **Opening book** + **endgame tablebase**.

### Neural / AlphaZero-like

- **Monte Carlo Tree Search (MCTS)**: seleciona, expande, simula (ou avalia rede), retropropaga.
- **Rede neural dupla**: value (quem ganha) + policy (quais lances explorar).
- **Self-play** (reinforcement learning) produz dados de treino.
- Contraste: Stockfish = cálculo massivo (100M+ nós/s) + avaliação eficiente; Lc0 = pesquisa mais rasa (~100k nós/s) + avaliação mais profunda.
- **Hybrid** (Stockfish NNUE) combina vantagens.

### Nível atual

- Stockfish 16 com NNUE: ~3600 Elo (ranking CCRL / CEGT). Humanos top: ~2800. Diferença 800 Elo → humano ganha 1 em 50.
- "Cilindro" — engines são super-humanos há anos.
- Open source (Stockfish, Lc0, Komodo) disponível gratuitamente → democratizou análise.

## IA e Xadrez

Xadrez foi o "Drosophila da IA" (Kotok, 1961):

- Pequeno mas rico.
- Regras formais, objetivo claro.
- Complexidade grande (10^44 posições legais, 10^123 árvore de jogo) mas tratável com truques.

Marcos:
- **1997 Deep Blue vs Kasparov**: fim da supremacia humana em clássico.
- **2017 AlphaZero**: mostra que RL + autoaprendizado supera décadas de engenharia.
- **2020+ Maia (Microsoft)**: rede treinada para **imitar humanos** em Elo específico — aprendizado humano-like.
- **LLMs jogam razoavelmente**: GPT-4 nível ~1600 puro; Claude, Gemini também. Integrados com tools (ex.: chamando Stockfish) ficam arbitrariamente fortes.

## Plataformas Online

### Chess.com

- Maior base de usuários (~150M+).
- Puzzles, lessons, Twitch/YouTube reach.
- Match em rapid/blitz/bullet.
- Análise integrada com Stockfish + coaches.
- Jogos anônimos + rated.

### Lichess

- Open-source, gratuito, sem ads.
- Análise, puzzles, streaming.
- Rapid, blitz, bullet + classical + correspondence.
- Variantes nativas.

### Ambas suportam **Fair Play** (anti-cheat por análise estatística); cheating online é problema sério — **cheaters vencem jogando lances Stockfish**.

### Boom do xadrez online

- **2020 pandemia** + **The Queen's Gambit (Netflix, 2020)** explodiram interesse.
- **Twitch/YouTube**: Hikaru Nakamura, GothamChess (Levy Rozman), BotezLive, chessbrah. Commentary + speedrun + reaction a clipes.
- **Pogchamps**: streamers não-xadrezistas aprendendo + torneios cômicos.

## Variantes

### Fischer Random (Chess960)

Posição inicial da 1ª fileira aleatória (entre 960 possibilidades válidas). Elimina teoria de abertura — obriga criatividade. Carlsen, Nakamura campeões mundiais de 960. Fischer propôs em 1996.

### Crazyhouse / Bughouse

Peças capturadas podem ser "dropadas" como próprias. Crazy = 1v1; bug = 2v2 com tabuleiros paralelos, capturas vão ao aliado. Cultura Lichess.

### 3-Check

Vence quem dar xeque 3 vezes primeiro.

### King of the Hill

Vence quem alcança casa central com rei.

### Atomic, Antichess (Giveaway), Horde, Racing Kings

Outras variantes de nicho. Lichess hosteia todas.

### Shogi, Xiangqi, Makruk

Primos culturais (Japão, China, Tailândia). Regras significativamente diferentes; base comum chaturanga.

## Estudo

### Básico (iniciante a ~1500)

1. **Táticas, táticas, táticas**: puzzles diariamente. Chesstempo, Lichess Tactics, Chess.com Puzzles.
2. **Regras de abertura**, não teoria decorada: controlar centro, desenvolver, roque.
3. **Finais elementares**: K+Q, K+R vs K; peão passado; oposição.
4. **Anotar e revisar** partidas próprias.
5. **Jogar longo** (rapid 15+ min) > bullet.

### Intermediário (1500-2000)

- **Finais teóricos**: Silman's Endgame Course, Dvoretsky.
- **Padrões táticos**: The Woodpecker Method.
- **Repertório mínimo**: 1 abertura de brancas, 1 contra 1.e4, 1 contra 1.d4.
- **Análise de partidas de GMs clássicos** anotadas (Capablanca, Alekhine, Karpov).

### Avançado (2000+)

- Meio-jogo estratégico: "My System" (Nimzowitsch), "Strategic Chess Exercises" (Emms).
- Prepara vs oponentes.
- Engine **depois** de análise própria, não antes.
- Coaching.

### Ferramentas

- **Chessable**: flashcards spaced-repetition para aberturas.
- **Chess.com / Lichess Studies**: notas + linhas.
- **SCID / ChessBase**: banco de partidas + análise.
- **Cerebellum, Stockfish, Lc0**: análise.

## Literatura clássica

- **"My System"** — Aron Nimzowitsch (1925). Base posicional moderna.
- **"My 60 Memorable Games"** — Bobby Fischer.
- **"The Life and Games of Mikhail Tal"** — autobiografia tática.
- **"Zurich 1953"** — David Bronstein.
- **"Silman's Complete Endgame Course"** — Silman.
- **"Dvoretsky's Endgame Manual"** — Dvoretsky (referência avançada).
- **"How to Reassess Your Chess"** — Silman (estratégia).
- **"The Art of Attack"** — Vukovic.
- **"Think Like a Grandmaster"** — Kotov.
- **"Logical Chess: Move by Move"** — Chernev (iniciantes).

## Cultura e Influência

- **"The Luzhin Defence"** (Nabokov), **"The Defense"** (livro / filme).
- **"Searching for Bobby Fischer"** (1993).
- **"The Queen's Gambit"** (Netflix 2020): fenômeno cultural, xadrez boom.
- **"Pawn Sacrifice"** (2014, sobre Fischer-Spassky 1972).
- **Alfil em literatura**: Borges, Zweig "The Royal Game".

## Resolução Formal

Xadrez é **finito e com informação perfeita** → teoricamente tem valor (brancas vence, pretas vence, ou empate com jogo perfeito). Não sabemos qual é.

- **Damas (checkers)** resolvido por Schaeffer (2007) — empate.
- **Gomoku, Connect 4** resolvidos.
- Xadrez: árvore gigantesca ($10^{123}$ aprox.) → improvável resolução total em breve. Consenso empírico: **empate com jogo perfeito**, mas sem prova.

7-piece tablebases são "solução parcial"; escala para mais peças é limite de storage.

## Xadrez e Saúde Cognitiva

- Estudos mistos. Correlação com melhor função cognitiva em idosos; causalidade debatível.
- Utilidade em educação: Armênia tornou obrigatório em escolas (2011); pressuposto de melhorar raciocínio. Impacto acadêmico medido com resultados modestos.
- **Gênero**: GMs são ~5% mulheres. Causas debatidas (participação vs performance); Polgar irmãs demonstraram topo é alcançável.

## Economia do Xadrez

- **Prêmios**: World Championship ~€2M total para match. Torneios tops (Norway Chess, Tata Steel, Sinquefield) €100-500k primeiro lugar.
- **Salários super-GMs**: ~$50k-500k/ano via prêmios + sponsors + streaming.
- **Streaming**: Hikaru Nakamura estimado 6-dígitos/mês de Twitch.
- **Plataformas**: Chess.com vale bilhões (estimativas 2024). Lichess é nonprofit.
- **Pirataria e cheating**: economia paralela de cheaters em online (ratings fakes, coaching fake).

## Escândalos e Polêmicas

- **Carlsen vs Niemann (2022)**: Carlsen se recusa a jogar após suspeita de cheating. Chess.com divulga relatório alegando cheating em 100+ partidas online de Niemann. Niemann processa, acordo extrajudicial.
- **Toiletgate (Topalov-Kramnik 2006)**: acusação de cheating via transmissor em banheiro — nunca confirmado.
- **Feller e Marzolo (2011)**: cheating em torneio de equipes francês via cúmplices.

Cheating online é problema estrutural; offline, detecção melhorou com scan eletromagnético em torneios tops.

## Princípios

1. **Toda peça quer mover-se**. Passividade perde.
2. **Rei é peça forte no final**. Ative-o cedo quando material cai.
3. **Controle o centro com peças, não só peões**.
4. **Lance mau: atrasar desenvolvimento para ganhar peão envenenado**.
5. **Dê tempo, não peça**. Tempo é recurso escasso.
6. **Quando não sabe o que jogar, melhore a peça pior**.
7. **Avalie: rei, material, atividade, estrutura, espaço — nessa ordem de urgência**.
8. **Calcule lances de candidatos, não o primeiro que vem**.
9. **Em final, cada tempo conta; em meio-jogo, cada peça; em abertura, cada lance desenvolve**.
10. **Aprender > vencer** em regime de estudo. Analise derrotas sem engine primeiro.