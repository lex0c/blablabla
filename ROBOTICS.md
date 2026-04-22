# Robótica

[Robótica](https://en.wikipedia.org/wiki/Robotics) é o ramo interdisciplinar da engenharia e ciência da computação dedicado ao projeto, construção, operação e uso de robôs. Envolve mecânica, eletrônica, teoria de controle, ciência da computação e, cada vez mais, inteligência artificial. Um robô é, em definição útil, uma **máquina agente**: percebe o ambiente por sensores, toma decisões (com ou sem autonomia), e atua no mundo físico por atuadores.

Aqui estão algumas áreas principais e conceitos relacionados à robótica:

1. **[Sensoriamento](https://en.wikipedia.org/wiki/Sensor):** converte fenômenos físicos (luz, som, força, rotação, distância) em sinais elétricos. Sem bons sensores, qualquer algoritmo sofisticado opera sobre ficção.

2. **[Atuação](https://en.wikipedia.org/wiki/Actuator):** converte sinais elétricos/hidráulicos em movimento. Motores, servos, solenoides, músculos pneumáticos. Define a capacidade física — velocidade, precisão, força, compliance.

3. **[Cinemática](https://en.wikipedia.org/wiki/Kinematics):** descreve o movimento sem referência às forças. Cinemática direta calcula a pose do efetuador a partir das juntas; inversa resolve o problema oposto — geralmente mais difícil.

4. **[Dinâmica](https://en.wikipedia.org/wiki/Dynamics_(mechanics)):** relaciona forças, torques, massas e acelerações. Essencial para controle preciso em manipuladores rápidos e robôs com pernas.

5. **[Controle](https://en.wikipedia.org/wiki/Control_theory):** fecha o laço entre referência desejada e comportamento real. Vai de PID clássico a controle preditivo, adaptativo, de impedância.

6. **[Percepção](https://en.wikipedia.org/wiki/Machine_perception):** interpretação dos dados de sensores — reconhecer objetos, estimar pose, segmentar cenas. Faz ponte com visão computacional e IA (ver `AI.md`).

7. **[Localização e Mapeamento](https://en.wikipedia.org/wiki/Simultaneous_localization_and_mapping):** saber onde o robô está e como é o ambiente. SLAM (Simultaneous Localization and Mapping) é o problema canônico.

8. **[Planejamento](https://en.wikipedia.org/wiki/Motion_planning):** encontrar caminhos e trajetórias viáveis entre estados — evitando obstáculos, respeitando limites dinâmicos, otimizando tempo/energia.

9. **[Autonomia](https://en.wikipedia.org/wiki/Autonomous_robot):** capacidade de operar sem supervisão humana contínua. Há um espectro — de teleoperação pura a autonomia total. A maioria dos robôs em produção hoje é semi-autônoma.

10. **Aplicações**:
    - **Indústria**: braços robóticos em manufatura, soldagem, pintura, montagem.
    - **Logística**: AMRs (Autonomous Mobile Robots) em armazéns, drones de entrega.
    - **Saúde**: robôs cirúrgicos (Da Vinci), próteses, reabilitação.
    - **Agricultura**: pulverização, colheita seletiva, monitoramento.
    - **Espaço e subaquático**: exploração em ambientes hostis.
    - **Serviço e doméstico**: aspiradores, cortadores de grama, assistentes.
    - **Militar e segurança**: vigilância, desminagem, UAVs.
    - **Inspeção**: dutos, pontes, estruturas, detecção de falhas.

11. **Desafios**:
    - **Incerteza sensorial**: todo sensor tem ruído, viés, falhas.
    - **Discrepância sim-to-real**: o que funciona em simulação raramente funciona igual no robô físico (ver `Sim-to-Real Gap` abaixo).
    - **Segurança**: robôs operando perto de pessoas precisam de garantias funcionais, não só estatísticas.
    - **Energia e autonomia elétrica**: baterias pesadas são o gargalo físico de robôs móveis.
    - **Generalização**: um robô treinado para uma tarefa raramente lida bem com variações.

12. **Futuro**:
    - Convergência com aprendizado de máquina (foundation models para robôs, políticas aprendidas end-to-end).
    - Robôs colaborativos (cobots) em espaços antes restritos a robôs enjaulados.
    - Manipulação destra generalista e humanoides de propósito geral.
    - Robôs blandos (soft robotics) e materiais programáveis.

## Tipos de Robôs

A taxonomia é imprecisa, mas organiza o campo.

### Por Morfologia

1. **Manipuladores fixos (braços robóticos)**: base fixa, múltiplas juntas, efetuador na ponta. Articulados (6-7 DOF — graus de liberdade), SCARA (rápido em planos), cartesianos/gantry (precisão em 3 eixos), delta/paralelos (alta velocidade com baixa inércia).

2. **Robôs móveis terrestres**:
   - **Com rodas**: diferencial (duas rodas motrizes + uma livre), omnidirecional (rodas mecanum, esféricas), ackermann (como carros), tracionados (esteiras).
   - **Com pernas**: bípedes (Atlas, humanoides), quadrúpedes (Spot, ANYmal), hexápodes. Trade-off clássico: mais pernas = estabilidade estática fácil; menos pernas = locomoção dinâmica, mais eficiente, mais difícil.

3. **Aéreos (UAVs / drones)**: multirotores (quadcopter, hexacopter — manobráveis, pouco eficientes), asa-fixa (eficientes, precisam de pista/lançamento), VTOL híbridos.

4. **Aquáticos**: AUVs (submarinos autônomos), ROVs (teleoperados com cabo), embarcações de superfície.

5. **Humanoides**: morfologia antropomórfica. Vantagem alegada: operam em ambientes desenhados para humanos, sem retrofit. Complexidade muito alta.

6. **Robôs moles (soft robotics)**: estruturas compliantes, frequentemente com atuação pneumática ou SMA (shape-memory alloys). Segurança intrínseca ao contato, morfologia adaptável.

7. **Microrrobôs e enxames**: unidades pequenas agindo coletivamente. Inspirados em insetos sociais; desafios: comunicação limitada, coordenação descentralizada.

### Por Aplicação

- **Industriais**: precisão, velocidade, repetibilidade. ISO 10218 é norma central de segurança.
- **Colaborativos (cobots)**: operam perto de humanos sem jaula. ISO/TS 15066 define limites de força/velocidade em contato.
- **De serviço**: ambiente não estruturado, interação com pessoas.
- **De campo**: outdoor, com intempéries, longo tempo sem supervisão.

## Graus de Liberdade (DOF)

Número de variáveis independentes que definem a configuração do robô. Um braço humano tem ~7 DOF (3 no ombro, 1 no cotovelo, 3 no punho). Robôs industriais comuns têm 6 DOF — mínimo para atingir qualquer pose 6D (posição + orientação) em um workspace.

**Redundância**: mais DOF que os necessários para a tarefa. 7+ DOF em manipulação permite evitar obstáculos ou singularidades mantendo a pose do efetuador. Custo: maior complexidade de controle; múltiplas soluções inversas.

**Subatuação**: menos atuadores que DOF. Robôs com pernas dinâmicos são subatuados (não há motor no "joelho aéreo"). Exige exploração ativa da dinâmica — domínio de controle não-linear.

## Sensores

Divisão clássica:

### Proprioceptivos (estado interno do robô)

- **Encoder** (incremental ou absoluto): posição angular das juntas. Base do controle de posição.
- **Potenciômetro**: variante analógica; barata, menos precisa.
- **Resolver**: robusto a ambientes agressivos, usado em aplicações militares/aeroespaciais.
- **IMU (Inertial Measurement Unit)**: acelerômetros + giroscópios (e frequentemente magnetômetros). Dá aceleração linear e velocidade angular. Drift acumulado; integrado com outros sensores (fusão).
- **Sensor de força/torque**: 6 eixos na base ou na junta. Essencial para manipulação com controle de impedância.
- **Tensão/corrente** de motores: estimativa indireta de torque (via modelo `τ ≈ K_t · i`; preciso se modelo térmico e de atrito é bom).
- **Temperatura** de motores e drivers: proteção térmica.

**Estimação de torque e estado avançada**:

- **Model-based torque**: combinar corrente + modelo dinâmico (M(q)q̈ + C q̇ + g + τ_atrito) para inferir torque externo sem sensor F/T dedicado. Usado em braços compliantes modernos (Franka Panda, Kinova).
- **Momentum-based observer**: detecta colisão comparando momento esperado vs real; trigger para parada ou recuo.
- **Joint torque sensors** (strain gauges na junta): mais caros mas precisos; permitem controle de torque direto (DLR LWR, Franka).
- **Kinesthetic teaching / lead-through**: humano move o robô fisicamente; ele grava trajetória. Requer back-drivability + torque sensing + gravity compensation — ponte para HRI.
- **Whole-body estimation**: fusão de IMU + encoders + F/T + kinemática para estimar forças externas distribuídas em humanoides/quadrúpedes.

### Exteroceptivos (ambiente)

- **Câmera RGB**: imagem passiva. Baixo custo, alta informação, sensível a iluminação.
- **Câmera estéreo**: duas câmeras, profundidade por disparidade.
- **Câmera RGB-D** (Kinect, RealSense, Structure): profundidade ativa (luz estruturada ou time-of-flight).
- **LIDAR**: laser scanner. 2D (plano) ou 3D (nuvem de pontos densa). Preciso, caro, pouco afetado por luz ambiente. Tempo de voo (ToF) ou FMCW.
- **Radar**: ondas de rádio. Menos preciso angularmente, mas funciona em chuva, névoa, pó. Mede velocidade via Doppler.
- **Ultrassom**: distância curta, baixa resolução. Barato, usado em sensores de ré automotivos.
- **Infravermelho (IR)**: curtas distâncias, sensível a materiais.
- **GPS / GNSS**: posição global. Precisão ~meter; RTK chega a centímetros. Sem cobertura indoor.
- **Bússola magnética**: orientação absoluta referente ao norte; sujeita a distúrbios.
- **Sensores táteis**: arrays de pressão em efetuadores. Cruciais para manipulação fina.
- **Câmeras de evento (event cameras)**: reportam mudanças de brilho por pixel, baixíssima latência e alta faixa dinâmica. Emergente.

### Fusão de Sensores

Nenhum sensor isolado basta. **Fusão sensorial** combina múltiplas fontes para estimar estado com menor variância que qualquer uma isolada.

Métodos clássicos:

- **Filtro de Kalman (KF)**: ótimo para sistemas lineares com ruído Gaussiano.
- **Filtro de Kalman Estendido (EKF)**: linearização local; padrão em SLAM clássico.
- **Unscented Kalman Filter (UKF)**: melhor para não-linearidades sem derivação analítica.
- **Filtro de partículas (Monte Carlo)**: não-paramétrico; lida com multimodalidade. Base de localização como AMCL.
- **Factor graphs / Bayes smoother**: inferência global, usada em SLAM moderno (GTSAM, iSAM2).

## Atuadores

1. **Motor DC escovado**: simples, controle linear, escovas se desgastam.
2. **Motor brushless (BLDC)**: eficiente, alta densidade de torque, exige controlador eletrônico (ESC). Dominante em drones e robôs modernos.
3. **Motor de passo (stepper)**: posicionamento por passos discretos, sem encoder em muitos casos. Perde passos sob torque alto.
4. **Servo motor**: motor + redução + controlador + feedback em um pacote. Padrão em hobby e em muitas juntas.
5. **Atuador hidráulico**: alta densidade de força, bom para cargas pesadas (Boston Dynamics Atlas antigo). Caro, pesado, vaza.
6. **Atuador pneumático**: rápido, barato, pouco preciso. Soft robotics tira proveito.
7. **Série-elástico (SEA — Series Elastic Actuator)**: mola entre motor e carga permite medir força via deformação e absorver impactos. Crítico em cobots e humanoides.
8. **Músculos artificiais / SMA / HASEL**: tecnologias emergentes; biomiméticas.

**Trade-offs fundamentais**:

- **Rigidez vs compliance**: rígido → preciso mas perigoso em colisão; compliant → seguro mas impreciso.
- **Velocidade vs torque**: reduções aumentam torque mas reduzem velocidade.
- **Back-drivability**: o eixo pode ser movido manualmente? Importante para sensoriamento passivo de força. Reduções altas (harmonic drives) matam back-drivability.
- **Densidade de energia**: torque por unidade de massa. Motores elétricos vs músculos humanos ainda é uma comparação desfavorável ao robô.

## Cinemática

### Cinemática Direta (Forward Kinematics)

Dado o vetor de ângulos das juntas **q**, calcular a pose do efetuador. Sempre tem solução única. Formalização clássica: **parâmetros de Denavit-Hartenberg (DH)** — tabela que codifica geometria do robô em 4 parâmetros por elo. Matrizes de transformação homogênea encadeadas produzem a pose final.

### Cinemática Inversa (Inverse Kinematics)

Dada pose desejada do efetuador, encontrar **q**. Pode ter:

- **Sem solução**: pose fora do workspace.
- **Solução única**: robôs simples.
- **Múltiplas soluções**: cotovelo para cima/baixo; robôs redundantes têm infinitas.
- **Solução analítica fechada**: para muitos robôs industriais (Pieper's solution exige pulso esférico).
- **Solução numérica**: iterativa (Newton-Raphson no Jacobiano, gradiente descendente, CCD, FABRIK). Padrão em pacotes como KDL, Pinocchio, TRAC-IK.

### Jacobiano

Matriz que relaciona velocidades de juntas (**q̇**) a velocidades do efetuador (**v**): **v = J(q) q̇**. Central para:

- Controle de velocidade no espaço cartesiano.
- **Singularidades**: pontos onde **J** perde posto — o robô perde graus de liberdade de movimento, ganhos tendem ao infinito se controle for naïve. Soluções: damped least squares (DLS), singularity-robust inverse.
- Manipulação coordenada.

### Workspace

Conjunto de poses alcançáveis. **Reachable workspace**: todas as poses atingíveis em alguma orientação. **Dexterous workspace**: poses atingíveis em *qualquer* orientação. Segundo é subconjunto do primeiro — frequentemente muito menor.

## Dinâmica

Enquanto cinemática ignora massas e forças, dinâmica as modela.

### Formulações

- **Newton-Euler recursivo**: O(n) por elo; eficiente para simulação e controle em tempo real.
- **Lagrange**: parte de energia cinética e potencial; mais elegante matematicamente, menos eficiente computacionalmente.
- **Equação geral**: **M(q) q̈ + C(q, q̇) q̇ + g(q) + τ_atrito = τ**, onde *M* é matriz de inércia, *C* trata Coriolis/centrífugas, *g* gravidade, *τ* torques de junta.

### Desafios

1. **Identificação de parâmetros**: massas, centros de massa, inércias raramente são conhecidos exatamente. Identificação por experimentos.
2. **Atrito**: não-linear, com stiction e efeitos dependentes de velocidade (Stribeck). Modelos: Coulomb, viscoso, LuGre.
3. **Elasticidade e backlash**: engrenagens, correias. Cada vez mais modelado explicitamente em robôs de precisão.
4. **Dinâmica de contato**: impacto, atrito entre superfícies, rigidez do contato. Núcleo duro de simulação física.

## Controle

### Controle Cinemático

Ignora dinâmica. Bom para movimentos lentos ou em robôs com alta redução (dinâmica dominada pelo motor).

### PID

Base de controle clássico. *P* proporcional ao erro, *I* acumula erro (elimina offset), *D* antecipa via derivada. Tuning (Ziegler-Nichols, manual) é arte. Limitações óbvias em sistemas acoplados e não-lineares, mas incrivelmente resiliente na prática.

### Controle Feedforward

Computa o torque necessário a partir do modelo dinâmico (*inverse dynamics*), fechando a malha apenas para corrigir erros residuais. Superior a PID puro quando modelo é confiável.

### Controle de Impedância e Admissão

Define a **relação dinâmica** entre força externa e movimento, não só a posição. Essencial para contato seguro e manipulação. Impedância: posição → força; admissão: força → posição. Compõem virtualmente um sistema massa-mola-amortecedor.

### LQR (Linear Quadratic Regulator)

Controle ótimo para sistemas lineares sob custo quadrático. Fundamento de muitos controladores em robôs com pernas e drones.

### MPC (Model Predictive Control)

Prevê N passos à frente usando modelo, otimiza trajetória, aplica só o primeiro passo, repete. Lida com restrições (limites de torque, obstáculos) naturalmente. Custo computacional alto; hoje viável em tempo real em robôs modernos. Base do controle avançado em humanoides, drones agressivos, carros autônomos.

### Controle Adaptativo

Parâmetros do controlador se ajustam online a mudanças de planta (ex.: carga variável). Alternativa a modelos fixos.

### Controle Aprendido

Políticas (policies) treinadas por RL ou imitação substituem ou complementam controle analítico. Ver seção sobre aprendizado.

## Percepção e Visão Computacional

Sobrepõe-se a `AI.md`, mas com sabor robótico:

1. **Detecção de objetos**: bounding boxes. YOLO, Faster R-CNN, DETR. Robótica tende a precisar em tempo real.
2. **Segmentação semântica / instância**: pixel a pixel. Mask R-CNN, SAM.
3. **Estimação de pose 6D**: posição + orientação de objetos no espaço. Crítico para manipulação. Métodos: PoseCNN, PVN3D, FoundationPose.
4. **Reconstrução 3D**: stereo, structure from motion, NeRFs, Gaussian splatting emergentes.
5. **Tracking visual**: seguir objeto entre frames. Filter-based (KCF) ou deep (SiamRPN, OSTrack).
6. **Visual servoing**: controle que usa imagem diretamente como feedback. Image-based (IBVS) ou position-based (PBVS).

### Percepção Ativa (Active Perception)

Em vez de receber passivamente o que o ambiente oferece, o robô **move sensores** para reduzir incerteza ou revelar informação oculta.

- **Next-Best-View (NBV)**: escolher próxima pose de câmera/LIDAR que maximiza ganho de informação. Base em entropia, information gain, frontier-based exploration.
- **Active SLAM**: planejar trajetória que simultaneamente reduz erro de localização e explora mapa.
- **Gaze control / foveation**: em humanoides, mover cabeça/olho para região relevante (biomimetic).
- **Tactile exploration**: mover dedo sobre superfície para mapear forma/textura onde visão falha (objetos transparentes, refletivos, oclusão).
- **Acoustic / probing**: bater em objeto para identificar material, contato-push para sentir rigidez.
- **Exploration policies** em RL: recompensa por "visitar desconhecido" além de alcançar objetivo.

Custo: movimento extra. Benefício: evita travar em ambiguidade sensorial — frequentemente a diferença entre robô que "quase funciona" e confiável.

### Percepção Semântica

Além de "o quê há onde", entender **o que significa**:

- **Affordance detection**: superfícies pegáveis, empurráveis, sentáveis. CLIP-based affordance em 2023+.
- **Relational reasoning**: "copo **sobre** a mesa **dentro da** cozinha".
- **Scene graphs**: representação grafo de objetos + relações; úteis para planejamento TAMP.
- **Vision-Language Models (VLMs)**: CLIP, SigLIP, PaLI — conectam imagem a texto; base de robôs que seguem instrução natural.
- **Open-vocabulary segmentation**: SAM + CLIP, Grounding DINO — identifica objetos não vistos em treinamento por nome.
- **Functional categorization**: "serve para beber" vs "azul e cilíndrico".

Essa camada habilita robôs que entendem "traga o copo **vazio**" vs "traga **qualquer** copo".

## Localização, Mapeamento e SLAM

Três problemas relacionados:

- **Localização**: mapa conhecido; achar onde o robô está. Métodos: AMCL (Monte Carlo), EKF-localization.
- **Mapeamento**: pose conhecida; construir mapa.
- **SLAM**: nenhum dos dois conhecido; resolver simultaneamente. O "problema da galinha e do ovo" da robótica.

### Variantes de SLAM

- **Filter-based (EKF/UKF SLAM)**: clássico, escala mal (complexidade quadrática).
- **Graph-based SLAM**: representa o problema como grafo de fatores; otimiza globalmente. g2o, iSAM2, GTSAM. Dominante hoje.
- **Visual SLAM**: apenas câmera. ORB-SLAM, LSD-SLAM, DSO. Monocular tem escala ambígua; stereo/RGB-D resolve.
- **LIDAR SLAM**: LOAM, Cartographer, LIO-SAM. Padrão em robôs com LIDAR.
- **Visual-Inertial SLAM**: câmera + IMU. VINS-Mono, OpenVINS.
- **Dense SLAM**: reconstrói superfície/volume, não só poses + landmarks. KinectFusion, ElasticFusion.

### Componentes-Chave

- **Odometria**: estimativa de movimento entre frames (wheel odom, VO, LO).
- **Loop closure**: detectar que se está visitando local antes visitado; reduz drift acumulado.
- **Relocalization**: recuperar pose após "kidnapped robot" (robô pego e colocado em outro lugar).

### Mapeamento Semântico

Mapas geométricos (nuvens de pontos, occupancy grids) dizem **onde** há obstáculo. Mapas semânticos dizem **o que** é: parede, porta, mesa, pessoa, obstáculo dinâmico vs estático.

- **Labels por célula / voxel**: grid semântico (cada célula tem classe + ocupação). SemanticKitti, nuScenes como datasets.
- **Object-centric maps**: armazenar objetos detectados com pose, classe, tamanho — não cada pixel.
- **Scene graphs hierárquicos** (Kimera, Hydra): edifícios → cômodos → objetos. Habilita queries "vá para a cozinha".
- **Dynamic-aware SLAM**: distinguir pontos estáticos (úteis para localização) de dinâmicos (pessoas, carros — descartáveis). DynaSLAM, DS-SLAM.
- **OctoMap semântico**: octree com rótulos.
- **Open-vocabulary mapping**: LangSplat, OpenScene, CLIP-Fields — embed features linguísticas em voxels; queryable por texto em runtime.

Aplicações diretas: navegação socialmente consciente (abaixo), TAMP, natural-language commands, long-term autonomy (mapa estável apesar de mudança em objetos móveis).

## Planejamento

### Espaço de Configurações (C-Space)

Espaço onde cada ponto é uma configuração do robô. Obstáculos no espaço físico viram regiões proibidas em C-Space. Planejamento é então: encontrar caminho contínuo livre de colisão no C-Space.

### Planejamento de Caminho (Path Planning)

Sem dinâmica; só geometria.

- **Grafos em grade + busca**: A*, Dijkstra, D* Lite (para replan). Simples, ótimo, sofre em alta dimensão.
- **PRM (Probabilistic Roadmap)**: amostra C-Space, constrói grafo, busca. Bom para consultas múltiplas.
- **RRT (Rapidly-exploring Random Trees)**: cresce árvore a partir do start em direção a amostras aleatórias. RRT\*, BIT\*, RRT-Connect são variantes. Padrão em braços robóticos.
- **Visibility graphs / cell decomposition**: métodos exatos em 2D.

### Planejamento de Trajetória (Trajectory Planning)

Adiciona tempo e restrições dinâmicas. Splines (cúbicos, quínticos), perfis trapezoidais, S-curve. Otimização: time-optimal (TOPP-RA), energy-optimal, jerk-minimal.

### Planejamento de Tarefa (Task Planning)

Decomposição de alto nível. PDDL, HTN (Hierarchical Task Networks), behavior trees. Integração com planejamento de movimento: Task and Motion Planning (TAMP).

### Evitação de Obstáculos (Local Planning)

Reação em tempo real a obstáculos não previstos.

- **DWA (Dynamic Window Approach)**: amostra velocidades no espaço dinâmico viável, pontua.
- **TEB (Timed Elastic Band)**: otimiza trajetória localmente como "elástico" deformável.
- **Potential fields**: atração ao alvo, repulsão de obstáculos. Clássico, sofre com mínimos locais.
- **MPC local**: previsão curta + restrições, muito usado em carros autônomos.

### Navegação Socialmente Consciente

Em ambientes com pessoas, trajetórias "ótimas" em distância são frequentemente **ruins socialmente** — cortar caminho entre duas pessoas, aproximar-se por trás, passar rápido demais em corredor estreito.

- **Proxemics** (Hall, 1966): zonas sociais — íntima (<0.45m), pessoal (0.45-1.2m), social (1.2-3.6m), pública (>3.6m). Robô respeita.
- **Social Force Model** (Helbing): pedestres como partículas com forças sociais; aplicável a robô.
- **ORCA / RVO (Reciprocal Velocity Obstacles)**: cada agente assume parte da responsabilidade de evitar.
- **Learning from demonstration**: IRL sobre trajetórias humanas para capturar convenções.
- **RL social**: recompensa inclui distância a pessoas, velocidade em proximidade, explicabilidade.
- **Legibilidade** (Dragan): trajetórias que **comunicam intenção**. Robô que vai à direita deve parecer ir à direita desde cedo, não em cima de decisão.
- **Predição de trajetória humana**: modelos de pedestrian forecasting (Social-LSTM, Trajectron++, social transformers) alimentam planner.
- **Contextos culturais**: convenções variam (pedestre na direita vs esquerda, contato visual, ceder passagem).

Benchmarks: SocNavBench, CrowdNav, HuNavSim. Métricas: taxa de colisão, tempo, desconforto humano (subjetivo + proxy como "intrusões em zona pessoal").

## ROS / ROS2

[Robot Operating System](https://www.ros.org) é o framework de fato para software de robótica de pesquisa e boa parte da indústria. Não é um SO; é um middleware + conjunto de convenções + ecossistema de pacotes.

### Conceitos em ROS1

- **Node**: processo que executa uma tarefa.
- **Topic**: canal pub/sub de mensagens (anônimo e assíncrono).
- **Service**: chamada síncrona request/response.
- **Action**: para tarefas longas com feedback (moveto, pick).
- **Message**: tipo de dado estruturado. Linguagens múltiplas (C++, Python, outras).
- **Parameter server**: configuração global.
- **tf / tf2**: árvore de transformações entre frames (world, base, camera, gripper...). Essencial — a maioria dos bugs em robótica são de referenciais.
- **rosbag**: gravar/replay de mensagens. Depuração e desenvolvimento offline.
- **rviz**: visualização 3D.
- **launch**: arquivos XML para inicializar sistemas multi-node.

### ROS2

Reescrita baseada em DDS (Data Distribution Service). Ganhos:

- **Sem master central**: descoberta descentralizada.
- **Qualidade de serviço (QoS)**: profiles para confiabilidade vs velocidade.
- **Tempo real melhor** (com DDS apropriado).
- **Suporte a Windows, embarcado** (micro-ROS).
- **Multi-plataforma mais sério**.

ROS2 é o futuro em implantações industriais sérias; ROS1 ainda domina pesquisa legada.

### Críticas

- Curva de aprendizado íngreme.
- Nem sempre robusto para produção sem endurecimento.
- Fragmentação de pacotes em qualidade.

## Simulação

Indispensável — robôs reais quebram, são lentos, caros, perigosos.

### Simuladores Populares

- **Gazebo / Ignition (gz)**: padrão histórico, integra ROS.
- **Webots**: amigável, boa para ensino.
- **MuJoCo**: física rápida e precisa, popular em RL (originalmente DeepMind).
- **PyBullet**: Python, popular em pesquisa.
- **NVIDIA Isaac Sim / Isaac Lab**: GPU-accelerated, fotorrealismo, ótimo para ML.
- **Drake**: modelagem rigorosa, controle e otimização.
- **CoppeliaSim** (V-REP): completo, comercial com licença acadêmica.

### O Sim-to-Real Gap

O modelo nunca é perfeito. Discrepâncias:

- Atrito e contato (notoriamente difíceis).
- Flexão de estruturas, backlash.
- Ruído sensorial realista.
- Latências de comunicação.
- Iluminação e texturas (para visão).

**Técnicas para reduzir o gap**:

- **System identification**: ajustar parâmetros do simulador para casar com o robô.
- **Domain randomization**: variar parâmetros aleatoriamente durante o treino (massa, atrito, ruído visual). Política aprende a ser robusta à variação.
- **Domain adaptation**: alinhar distribuições real e simulada (GANs, etc.).
- **Real-to-sim**: coletar dados reais e replicar em sim.
- **Residual policies**: aprender só a correção sobre controlador analítico.

## Manipulação

### Grasping

Pegar objetos com um end-effector. Problemas:

- **Detecção de pontos de preensão** (grasp pose detection). Dex-Net, GraspNet, Contact-GraspNet.
- **Controle de força**: garra nem esmaga nem deixa cair. Tactile feedback ajuda.
- **Garras**: paralelas (duas "dedos"), sucção, multi-fingered (mãos humanoides). Trade-off entre simplicidade e destreza.

### Manipulação Destra

Tarefas além de pick-and-place: montagem, manipulação em mão (in-hand), manipulação de objetos articulados (abrir portas, virar torneiras). Ainda fronteira de pesquisa.

### Motion Planning em Manipulação

- **MoveIt!** (ROS): stack padrão. Integra planejadores (OMPL), colisão, cinemática.
- **OMPL (Open Motion Planning Library)**: biblioteca de planejadores amostrais.
- **cuRobo** (NVIDIA): GPU-accelerated, tempo real.

## Aprendizado em Robótica

Convergência com IA (ver `AI.md`).

### Aprendizado por Reforço (RL)

Agente aprende política **π(a|s)** que maximiza recompensa cumulativa. Algoritmos-chave em robótica:

- **PPO (Proximal Policy Optimization)**: on-policy, estável, popular.
- **SAC (Soft Actor-Critic)**: off-policy, eficiente em dados, padrão em controle contínuo.
- **DDPG/TD3**: precursores.
- **Model-based RL**: aprende modelo do ambiente e planeja. Melhor sample efficiency.
- **World models** (Dreamer, etc.): lugar ativo de pesquisa.

**Desafios em RL para robôs reais**:

- **Sample efficiency**: robôs são lentos. Milhões de episódios são inviáveis diretamente.
- **Safe exploration**: exploração aleatória quebra hardware.
- **Reward shaping**: desenhar recompensa boa é difícil e é onde muito engenheiro RL gasta tempo.

### Aprendizado por Imitação (Imitation Learning)

Em vez de recompensa, demonstrações humanas.

- **Behavioral Cloning (BC)**: regressão supervisionada (estado → ação). Sofre de compounding error fora da distribuição.
- **DAgger**: corrige BC coletando correções iterativamente.
- **Inverse RL / GAIL**: inferir função de recompensa a partir de demonstrações.
- **Diffusion policies**: modelos de difusão como políticas — forte em multimodalidade.

### Visuomotor Policies e Foundation Models

Movimento atual: políticas end-to-end que mapeiam pixels (e linguagem) a ações. RT-2, Open X-Embodiment, π0, Gemini Robotics. Analogias com LLMs (ver `AI.md`): dados em escala + transformers.

### Sim-to-Real no Aprendizado

Domain randomization, adaptive simulation, real-world fine-tuning. Meio-caminho entre simulação pura e treino no robô.

## Interação Humano-Robô (HRI)

Quando o robô compartilha espaço com pessoas, o problema extrapola a engenharia pura. Combina robótica, psicologia, design, e ergonomia. Complementa `SOCIAL_PSYCHOLOGY.md`, `NEUROSCIENCE_101.md`.

### Segurança Física

- **Controle de força / impedância**: robô cede a contato em vez de resistir (ver `Controle de Impedância e Admissão`).
- **Collision detection** via momentum observer ou F/T sensors.
- **Stop 0/1/2** conforme ISO 10218 / 15066 (ver `Segurança Funcional`).
- **Limites biomecânicos** por região do corpo (cobot não pode bater no olho com mesma força aceitável em joelho).
- **Zonas de proximidade**: robô reduz velocidade ao detectar humano próximo (safety laser scanners, 3D vision).

### Legibilidade e Previsibilidade

**Legibilidade** (Dragan, 2013): movimento que **comunica intenção ao observador**, às vezes ao custo de trajetória ótima.

- Um braço que vai à esquerda deve iniciar movimento em direção esquerda cedo, mesmo que caminho reto exigisse desvio só no final.
- **Predictability** (complementar): movimento consistente com expectativa dada intenção conhecida.
- Em HRI, humanos inferem intenção continuamente — robôs "opacos" geram ansiedade.

### Interfaces de Controle

- **Voz**: reconhecimento + NLU. Robustez a ambientes ruidosos, sotaques, multi-idioma. LLMs como orquestradores (RT-2, PaLM-E).
- **Gestos**: reconhecimento via câmera (MediaPipe, OpenPose), luvas com IMU, anéis, pulseiras EMG (Myo legado, CTRL-Labs).
- **Teleoperação**: direta (mestre-escravo com feedback de posição) ou supervisionada (humano dá waypoints). Da Vinci em cirurgia, ROVs subaquáticos, Optimus em telepresença.
- **Haptic feedback em teleoperação**: força/textura retornada ao humano. Dispositivos: Phantom, Sigma, Touch, exoesqueletos háptico. Essencial em cirurgia remota e manipulação destra.
- **BCI (Brain-Computer Interfaces)**: EEG não-invasivo (Emotiv, NextMind, OpenBCI) — baixa largura de banda. Invasivo (Utah array, Neuralink, Synchron) — alto bitrate; usado em pacientes tetraplégicos para controle de braço robótico.
- **AR/VR**: Hololens, Quest — interface espacial; robô mostra intenção em overlay.
- **GUI clássica** ainda domina em fábrica.

### Detecção de Intenção Humana

Antes que usuário fale/gesticule, robô pode **antecipar**:

- **Gaze estimation**: direção do olhar sinaliza alvo.
- **Head pose, body orientation**: para onde a pessoa se dirige.
- **Reach prediction**: durante approach, inferir objeto alvo e alinhar entrega.
- **Action anticipation** via modelos sequência-a-sequência: dado vídeo dos últimos segundos, prever próxima ação.
- **Estado afetivo**: expressão facial (AU via OpenFace), tom de voz, postura. Fundamento em `SOCIAL_PSYCHOLOGY.md`.

Cuidado: falsos positivos em detecção de intenção geram comportamento robótico "paranoico" — calibrar limiares.

### Kinesthetic Teaching e Programming by Demonstration

Programa-se o robô **mostrando** a tarefa:

- **Lead-through**: humano move fisicamente o braço compliant; robô grava. Requer back-drivability + gravity compensation + torque sensing.
- **Teleop demonstration**: humano controla remotamente; robô aprende política via imitação (BC, DAgger, diffusion policies — ver `Aprendizado em Robótica`).
- **Vídeo demonstration**: observar humano fazendo; transferir para morfologia robótica. Cross-embodiment learning.
- **Trajectory shaping**: demonstrações múltiplas; DMP (Dynamic Movement Primitives) generalizam.

Habilita operador sem skill de programação — essencial em cobots industriais.

### Haptics

Além de feedback em teleop, haptics é canal bidirecional em co-manipulação:

- **Kinesthetic haptics**: força/torque em interface (joystick háptico).
- **Cutaneous / tactile**: vibração, pressão, textura — luvas, braceletes.
- **Virtual fixtures**: guias virtuais em teleop que restringem movimento a plano/eixo, reduzindo carga cognitiva.

### Modelo Mental e Antropomorfização

Pessoas atribuem intenção, emoção, confiabilidade a máquinas — **incluindo** quando sabem que é máquina (ELIZA effect, Reeves & Nass "The Media Equation").

- **Antropomorfismo calibrado**: design sugere capacidades realistas, evita over-promise (robô com olhos muito humanos gera expectativa de entender emoção).
- **Uncanny valley** (Mori, 1970): humanoides que são quase-humanos geram estranhamento; solução é ou simples-cartoon ou muito realista.
- **Transparência**: robô comunica o que sabe/não sabe ("não tenho certeza se é você, Maria"). Aumenta confiança calibrada.
- **Erro e recuperação**: robô que admite erro de forma competente mantém confiança melhor que robô silencioso.

### Socialmente Consciente

Ver também `Navegação Socialmente Consciente`. Em manipulação e co-trabalho:

- **Handover**: dar/receber objeto — timing, localização, força de grip.
- **Shared workspace**: alternância em bancada; robô percebe onde humano está trabalhando.
- **Turn-taking em diálogo**: saber quando falar, interromper.
- **Proxemics dinâmica**: robô adapta distância conforme tarefa, contexto cultural, estado emocional.

### Aceitação e Ética

- **Desconforto percebido**: robô "invasivo" mesmo sem tocar.
- **Privacy**: robôs de serviço com câmera em casa = sensor distribuído.
- **Substituição vs colaboração**: narrativa importa; robô "tira emprego" vs "alivia trabalho repetitivo".
- **Vulnerabilidade**: idosos, crianças, pessoas com deficiência merecem design cuidadoso.
- **Manipulação**: robô com vieses persuasivos ou UX "addictivos" é problema ético (cross-ref `PERSUASION`, `SOCIAL_ENGINEERING.md`).

### Avaliação

Métricas quantitativas + qualitativas:

- **Tempo de tarefa, erros, intervenções**.
- **NASA-TLX** (carga cognitiva).
- **Godspeed questionnaire** (antropomorfismo, animacidade, simpatia, inteligência percebida, segurança).
- **Trust scales** (Muir, Jian et al.).
- **Fisiologia**: HRV, EDA, eye-tracking durante interação.
- **Observação comportamental**: hesitação, distância mantida, engajamento.

HRI é frequentemente **qualitativa** antes de quantitativa — entender experiência do usuário precede métrica.

## Segurança Funcional

Robôs industriais e médicos são sistemas de segurança crítica.

- **ISO 10218**: robôs industriais em geral (jaula).
- **ISO/TS 15066**: cobots (contato intencional). Define limites biomecânicos de força/velocidade por região do corpo.
- **ISO 13849 / IEC 62061**: níveis de performance (PL) e integridade de segurança (SIL).
- **IEC 61508**: funcional safety geral de sistemas eletrônicos programáveis.
- **Categorias de parada**: Stop 0 (remoção imediata de potência), Stop 1 (parada controlada, depois potência removida), Stop 2 (controlada, potência mantida).
- **Redundância**: duplo canal para funções críticas, diversidade em hardware/software.

Software "inteligente" (ML) geralmente não é certificável hoje — daí arquiteturas híbridas onde ML informa mas não decide sozinho em loops de segurança.

## Robôs Específicos — Detalhes

### UAVs / Drones

Multirotores são sistemas subatuados (4 motores, 6 DOF). Controle em cascata: atitude (rápido) → posição (lento). MPC e backstepping são comuns. Desafios: estimativa de atitude, vento, latência de visão.

### Robôs com Pernas

Base conceitual: **ZMP (Zero-Moment Point)** para estabilidade quase-estática. Em locomoção dinâmica: capture point, divergent component of motion (DCM), LIP (Linear Inverted Pendulum). Boston Dynamics abriu o caminho da locomoção dinâmica; hoje reproduzido em academia (MIT Cheetah, ANYmal). RL para locomoção em terreno difícil é estado-da-arte.

### Soft Robotics

Materiais compliantes (elastômeros, tecidos). Atuação por câmaras infláveis, cabos (tendões), SMAs. Modelagem é difícil — métodos contínuos (Cosserat rods, FEM). Aplicações: manipulação segura, próteses, robôs médicos endoscópicos.

### Enxames (Swarm Robotics)

Muitos agentes simples com regras locais geram comportamento coletivo. Inspiração: cupins, abelhas. Desafios: comunicação local, robustez a perda de unidades, descentralização.

### Robôs Bioinspirados

Copiar princípios biológicos: locomoção de insetos, cauda de geckos, estrutura de ossos, morfologia. Dialoga com biomimética.

## Hardware Computacional em Robôs

1. **Microcontroladores** (STM32, ESP32, RP2040): loops de controle rápidos, baixo consumo.
2. **SBCs / computadores embarcados** (Raspberry Pi, NVIDIA Jetson, Intel NUC): ROS nodes, percepção, planejamento.
3. **FPGAs**: processamento determinístico, baixa latência (filtros de imagem, controle).
4. **GPU embarcada** (Jetson): inferência deep learning em tempo real.
5. **Time-sensitive networking (TSN) / EtherCAT / CAN**: barramentos determinísticos para comunicação motor-controlador.

## Sistemas Multi-Robô

Quando múltiplos robôs operam juntos, emergem problemas que não existem em robô único.

### Taxonomias

- **Centralizado vs descentralizado**: servidor único coordena vs regras locais.
- **Homogêneo vs heterogêneo**: idênticos vs especializados complementares.
- **Cooperativo vs adversarial**: mesmo objetivo vs competindo (CIC, ver `GAME_THEORY.md`).
- **Síncrono vs assíncrono**: relógio comum vs independente.

### Problemas Centrais

- **Task allocation**: quem faz o quê. Auction-based (market), hungarian, consensus. CBBA (Consensus-Based Bundle Algorithm) é clássico.
- **Formation control**: manter geometria relativa. Leader-follower, virtual structure, behavior-based.
- **Coverage**: cobrir área coletivamente (Voronoi coverage, Lloyd's algorithm).
- **Rendezvous**: encontrar-se em ponto.
- **Consensus**: concordar em valor (média, max/min) com comunicação limitada.
- **Distributed SLAM**: cada robô mapeia parcialmente; fundem mapas periodicamente. CCM-SLAM, Kimera-Multi.
- **Cooperative perception**: observações de múltiplas posições → cobertura + robustez. Crítico em veículos autônomos (V2X).

### Comunicação

- **Rede ad-hoc**: Wi-Fi mesh, DSRC, 5G URLLC, ZigBee, LoRa para longos alcances.
- **ROS 2 + DDS** permite discovery sem broker central.
- **Bandwidth limitada** muda design: compressão, summary estatística, prioritização.
- **Latência e packet loss**: algoritmos devem tolerar.
- **Sem comunicação** (comm-aware planning): robôs assumem o pior caso; retornam a rendezvous quando perdem contato.

### Enxames (ver `Swarm Robotics` abaixo)

Regime de muitos (dezenas+) com regras locais simples. Inspiração biológica.

### Aplicações

- **Armazéns** (Amazon Robotics, AutoStore): centenas de AGVs coordenados.
- **Busca e resgate**: cobertura rápida de área.
- **Agricultura de precisão**: tratores + drones cooperando.
- **Militar**: enxames de drones.
- **Exploração espacial / subaquática**: redundância em ambiente hostil.
- **Formação aérea** (shows de drones: Intel, Skymagic) — demonstração pública.

### Frameworks

- **ROS 2 multi-robot**: namespaces, discovery.
- **Multi-Agent PettingZoo / RLlib**: RL multi-agente.
- **Gazebo / Isaac Sim multi-robot**: simulação.
- **Buzz** (linguagem de swarm), **ARGoS** (simulador de enxame).

## Calibração

Sem calibração, nenhuma precisão teórica é realizada. É disciplina própria — muitos fracassos em robótica são "calibração ruim" disfarçados.

### Câmera

- **Intrínseca**: matriz de câmera (foco, centro ótico), distorção radial/tangencial. **Zhang's method** com checkerboard ou ChArUco. Software: OpenCV `calibrateCamera`, Kalibr, Basalt.
- **Estéreo**: baseline + rectification.
- **Rolling shutter** em CMOS: linhas capturadas em tempos diferentes; distorção em movimento exige calibração temporal ou escolha de global shutter.
- **Fisheye, wide-angle**: modelo de distorção específico (MEI, Scaramuzza).

### Extrínseca

- **Hand-eye calibration**: transformação entre câmera e base/flange do robô. Métodos Tsai-Lenz, Park-Martin, otimização não-linear. Setups eye-in-hand vs eye-to-hand.
- **Sensor-to-sensor**: LIDAR ↔ câmera (LiDAR-Camera calibration toolkits), câmera ↔ IMU (Kalibr), radar ↔ câmera.
- **Sensor ↔ robot base**: alvos conhecidos, múltiplas poses.

### IMU

- **Bias (viés)**: estimado em repouso; varia com temperatura.
- **Fator de escala**.
- **Desalinhamento entre eixos**.
- **Allan variance**: caracterizar random walk, bias stability.
- **Temperature compensation** em IMUs industriais.

### Cinemática do robô

Parâmetros DH reais divergem de projetados por tolerância de fabricação, flexão, desgaste. **Calibração cinemática** identifica parâmetros a partir de medições de pose do efetuador.

- **Leica laser tracker, API Radian** como referência externa.
- Ajuste mínimos quadrados sobre dezenas de poses.
- Em braços de precisão (ABB IRC5, KUKA KSS), calibração de fábrica + opcional no cliente.

### Força/torque

- **Viés** (mudança de zero) deriva com temperatura e tempo.
- **Gravity compensation do end-effector**: pesar tool, centro de massa, inércia. Importante para precisar forças externas reais.
- **Tool Center Point (TCP)**: geometria do efetuador em relação ao flange.

### Cronometragem (time sync)

Sensores diferentes em tempos distintos → fusão errada. Soluções: PTP (Precision Time Protocol), hardware trigger sync, interpolação com timestamping de cada amostra.

### Online / self-calibration

Sistemas modernos re-calibram em operação:

- **VINS** (Visual-Inertial) estima bias IMU em runtime.
- **Hand-eye online** quando hardware permite.
- **Drift correction** via loop closure (ver SLAM).

Calibração é chata, ingrata e essencial. "Mede duas vezes, monte uma."

## Desafios Práticos e Armadilhas

1. **Referenciais (frames)**: a maior parte dos bugs em robótica é simples: o dado estava no frame errado. Use `tf2` disciplinadamente.
2. **Timing**: sistemas reais têm latências e jitter. Desenhar assumindo "instantâneo" dá controle oscilatório ou instável.
3. **Energia**: baterias acabam. Ciclos de carga são caros. Regeneração é marginal.
4. **Poeira, umidade, temperatura**: laboratório é fantasia; em campo, IP rating e teste ambiental são obrigatórios.
5. **Cabos**: trançam, fadigam, quebram em juntas rotativas. Slip rings, roteamento, tolerâncias de torção importam.
6. **EMI (interferência eletromagnética)**: motores de alto torque geram ruído que fritam sensores mal blindados.
7. **Reprodutibilidade**: o robô "ontem funcionava". Ingrediente mudou: temperatura, atrito, bateria, tolerância mecânica.
8. **Depuração**: robô é sistema distribuído de tempo real; breakpoint trava o mundo. Logs + visualização + simulação são as ferramentas reais.

## Robustez em Ambiente Não-Estruturado

"Funcionou na demo, quebrou em campo" é o clichê. Robô deploy real enfrenta o que laboratório esconde.

### Iluminação e visão

- **Luz direta do sol**: saturação, sombras duras, baixo contraste. Exposure adaptativa, HDR imaging, câmera de evento.
- **Baixa luz, noite**: IR ativo (illuminator), LIDAR independente de luz, sensor fusion que tolera falta de uma modalidade.
- **Reflexos (piscina, vidro, metal)**: LIDAR sofre, câmera vê fantasmas, objetos transparentes "somem" em depth sensor.
- **Chuva, neve, neblina, poeira**: degrada visão e LIDAR; radar vê através. Fusão beneficia.

### Terreno e contato

- **Superfícies variadas**: grama alta, areia, lama, rocha, neve. Locomoção muda drasticamente.
- **Obstáculos negativos**: buracos, penhascos — difíceis de detectar em LIDAR (ausência de retorno ≠ vazio seguro).
- **Superfícies deformáveis**: pé afunda; odometria baseada em encoder erra.
- **Vegetação**: galho flexível parece obstáculo ao LIDAR; empurrável vs não é questão semântica.

### Ambiente dinâmico

- **Pessoas imprevisíveis**: crianças, animais, multidão.
- **Outros robôs / veículos**.
- **Objetos móveis não-cooperativos**: vento movendo vegetação, folhas voando confundindo detecção.

### Comunicação degradada

- **Outdoor sem Wi-Fi confiável**: mesh, celular, offline operation.
- **Subterrâneo, subaquático, espaço**: rádio restrito ou inexistente; acústico, cabo, atraso.
- **Interferência industrial**: multipath em armazéns metálicos.

### Resiliência de hardware

- **IP rating adequado** (IP67 para imersão; IP54 para indoor sujo). Testado, não declarado.
- **Vibração, choque**: testes conforme MIL-STD-810, IEC 60068.
- **Faixa de temperatura**: -20°C a +60°C é comum para campo; armazenamento frio em carga inicial.
- **Radiação** em nuclear, espacial, médico — rad-hardened electronics.

### Manutenção e operação

- **Manutenibilidade**: quem troca o motor no campo? Campo agrícola, mina subterrânea, alto-mar não têm bancada.
- **Autonomia energética**: recarga sem humano (docking stations), troca automática de bateria.
- **Failover e graceful degradation**: sensor morre → robô funciona em modo reduzido, não trava.
- **OTA updates** (ver `IOT_FIRMWARE_SECURITY.md`) seguras e com rollback.
- **Telemetria e monitoramento** (ver `CONDITION_MONITORING.md`): detectar degradação antes de falha catastrófica no campo.

### Regulamentação

- **Drones**: ANAC (BR), FAA (EUA), EASA (UE) — autorização por peso, altitude, BVLOS.
- **AMRs**: certificação ANSI R15.08 (EUA), ISO 3691-4 (UE), normas locais.
- **Autônomos em rua**: L1-L5 SAE, regulação variável por estado/país.

## Ética e Impacto Social

- **Automação e trabalho**: deslocamento e transformação de empregos.
- **Autonomia letal**: armas autônomas; debate ativo na ONU.
- **Responsabilidade**: quem responde quando um robô fere alguém? Engenheiro, operador, fabricante, algoritmo?
- **Privacidade**: robôs móveis com câmeras em casa/rua.
- **Viés em percepção**: modelos de visão treinados em datasets limitados falham em grupos sub-representados.
- **Dependência**: próteses, cuidadores robóticos — a pessoa torna-se dependente do fabricante que pode falir ou descontinuar.

## Futuro Próximo (meados da década de 2020 em diante)

1. **Foundation models para robótica**: políticas generalistas pré-treinadas, fine-tuned para tarefas específicas.
2. **Humanoides para uso geral**: Tesla Optimus, Figure, 1X, Agility Digit. Apostas caras; retorno ainda não provado.
3. **Cobots em escritório e serviço**: fora da jaula, perto de pessoas.
4. **Autonomia off-road e em campo**: agricultura, mineração, logística subterrânea.
5. **Microrrobótica médica**: intracorporal, drug delivery.
6. **Manipulação destra generalista**: o último "grande problema" da manipulação.
7. **Integração com LLMs**: robôs que seguem instrução em linguagem natural e explicam seus planos.

## Livros, Referências e Ferramentas

- **"Robotics: Modelling, Planning and Control"** — Siciliano et al. Clássico, denso.
- **"Introduction to Autonomous Mobile Robots"** — Siegwart, Nourbakhsh, Scaramuzza.
- **"Modern Robotics"** — Lynch & Park. Curso gratuito excelente no Coursera.
- **"Probabilistic Robotics"** — Thrun, Burgard, Fox. Bíblia de localização/mapeamento.
- **"Planning Algorithms"** — LaValle. Referência em planejamento.
- Pacotes: **ROS/ROS2, MoveIt!, OMPL, GTSAM, Pinocchio, Drake, Isaac Lab**.

## Conclusão

Robótica é a engenharia de sistemas que agem no mundo físico. Combina teoria de controle, percepção, planejamento e, crescentemente, aprendizado. Diferente de software puro, cada bug tem uma contraparte física — e o mundo real não tem pena de modelos idealizados. Por isso, um bom engenheiro de robótica vive entre o rigor matemático e a humildade diante do atrito, do ruído e da gravidade.
