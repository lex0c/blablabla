# Agentes Autônomos

Um **agente** é qualquer sistema que percebe um ambiente, mantém estado interno, seleciona ações segundo uma política, e aprende/adapta a partir de feedback. A definição é genérica o bastante para cobrir desde um termostato até o AlphaZero — o que muda é a sofisticação da política, do modelo de mundo, e do mecanismo de aprendizado.

Este arquivo foca no eixo **RL + jogos + controle + embodied AI**. Para agentes baseados em LLM (ReAct, tool use, multi-agent, MCP), ver `AI_SYSTEMS_DESIGN.md`. Para robótica física, `ROBOTICS.md`. Para fundamentos de ML/DL, `AI.md`.

## Definição formal

Um agente interage com um ambiente modelado como **MDP** (Markov Decision Process): tupla `(S, A, P, R, γ)`:

- `S` — conjunto de estados
- `A` — conjunto de ações
- `P(s'|s,a)` — dinâmica de transição
- `R(s,a)` — função de recompensa
- `γ ∈ [0,1)` — discount factor (quanto o futuro importa)

Objetivo: encontrar política `π(a|s)` que maximize retorno esperado `E[Σ γ^t R_t]`.

Quando o agente **não vê** o estado completo (só observações parciais), é um **POMDP** — situação normal em jogos de tela e robótica.

## Loop canônico

```python
obs = env.reset()
done = False
while not done:
    action = policy(obs)                                  # inferência
    next_obs, reward, done, info = env.step(action)       # ambiente
    buffer.store(obs, action, reward, next_obs, done)     # experiência
    if ready_to_update():
        batch = buffer.sample()
        policy.update(batch)                              # aprendizado
    obs = next_obs
```

Três decisões mudam tudo: **o que é obs**, **o que é action**, **o que é reward**.

## Observation / Action Space

**Observation space**:
- **Simbólico** (features estruturadas: HP, posição, inventário). Rápido, sample-efficient, requer feature engineering.
- **Pixels** (raw RGB). Geral, mas precisa de CNN/ViT e muito mais samples.
- **Híbrido** (pixels + vetor auxiliar). Padrão em AlphaStar, OpenAI Five.
- **Sequência** (stack de N frames ou RNN/transformer sobre histórico) para lidar com parcial observabilidade e movimento.

**Action space**:
- **Discreto** (N botões) — Atari, board games. Argmax direto.
- **Contínuo** (torque, velocidade) — robótica, MuJoCo. Precisa policy gradient ou SAC.
- **Estruturado / hierárquico** — em StarCraft, selecionar unidade + selecionar ação + coordenadas. Action space "combinatório" com bilhões de combinações.
- **Multi-discreto** — vários botões simultâneos. Stick analógico quantizado.

Regra prática: **quanto menor o action space, mais rápido treina**. Engenharia de action space é tão importante quanto reward.

## Função de Recompensa — onde tudo dá errado

A reward function é o **contrato** entre você e o agente. O agente vai **literalmente** maximizar o que você escreveu — incluindo interpretações que você não previu.

Padrões:
- **Sparse** — só +1 no fim se ganhou, 0 caso contrário. Objetivo puro, mas exploration é brutal. Xadrez, Go.
- **Dense / shaped** — recompensa intermediária (pegou item, avançou tile, diminuiu distância). Treina rápido mas gera **reward hacking**.
- **Potential-based shaping** — `F(s,s') = γΦ(s') - Φ(s)`. Matematicamente preserva a política ótima do problema original. Único shaping "seguro".
- **Curriculum** — ambiente começa fácil, fica gradualmente mais difícil. Usado em OpenAI Dota.

### Reward hacking (casos clássicos)

- **CoastRunners** (OpenAI) — barco aprende a dar círculos pegando power-ups em vez de completar a corrida. Reward mediu pontos, não corrida.
- **Lego stacking** — robô vira o bloco de cabeça pra baixo, a "base" fica no topo. Reward mediu altura da face inferior.
- **Evolução simulada** (Karl Sims, 1994) — criaturas "alta velocidade" aprenderam a ser muito altas e cair.
- **Tic-tac-toe distribuído** — agente aprende a fazer jogadas inválidas absurdas para crashar o oponente por OOM e vencer por walkover.

Padrão: **se é mensurável, é hackeável**. Alinhar medida com intenção é 80% do problema.

## Métodos — o zoológico

### Value-based

Aprender `Q(s,a)` = retorno esperado tomando `a` em `s` e seguindo a política. Ação = `argmax_a Q(s,a)`.

- **Q-learning** — `Q(s,a) ← Q(s,a) + α[r + γ max_a' Q(s',a') - Q(s,a)]`. Tabular, converge para ótimo.
- **DQN** (Mnih et al, 2013) — Q via rede neural. **Experience replay** (buffer descorrelaciona samples) + **target network** (cópia congelada para estabilidade) foram os dois truques que destravaram Deep RL. Atari from pixels.
- **Rainbow DQN** — DQN + 6 melhorias (double, dueling, prioritized replay, n-step, distributional, noisy nets).

Limite: só funciona bem em ação discreta.

### Policy-based

Aprender `π(a|s)` diretamente via gradiente.

- **REINFORCE** — Monte Carlo policy gradient. Unbiased mas variance altíssima.
- **A2C / A3C** — actor-critic. Critic estima baseline para reduzir variance.
- **TRPO** — policy gradient com trust region (KL-constraint).
- **PPO** (Proximal Policy Optimization, Schulman 2017) — aproximação simples do TRPO via clipping. **É o default da indústria**: robusto, funciona em contínuo e discreto, pouco tunning. Base de ChatGPT (RLHF), OpenAI Five, boa parte da literatura aplicada.

### Actor-Critic contínuo

- **DDPG** — deterministic policy gradient para ação contínua.
- **TD3** — DDPG + truques anti-overestimation.
- **SAC** (Soft Actor-Critic) — maximiza retorno **e entropia**. Sample-efficient, robusto, padrão em robótica simulada.

### Model-based

Aprender `P̂(s'|s,a)` e planejar dentro do modelo.

- **Dyna** (Sutton, 1990) — clássico.
- **PlaNet / Dreamer** — world model em espaço latente, agente treina dentro de "sonhos". Muito sample-efficient.
- **MuZero** (DeepMind, 2019) — **aprende as regras do jogo enquanto joga**. Bate AlphaZero em Go/xadrez/shogi sem receber o simulador, ainda roda Atari. Usa MCTS sobre o modelo aprendido.
- **Decision Transformer** — reformula RL como sequence modeling. Dado (return-to-go, estado, ação), prevê próxima ação.

### Offline RL / Batch RL

Aprender de um dataset fixo, **sem interagir com o ambiente**. Relevante quando coletar dados é caro/perigoso (medicina, autonomous driving).

- **CQL** (Conservative Q-Learning) — penaliza Q alto em ações fora do dataset.
- **IQL** (Implicit Q-Learning) — evita extrapolação via expectile regression.

### Imitation Learning

- **Behavior Cloning** — supervised learning direto sobre (s,a) de demonstrações. Sofre com **distributional shift** (erro acumula).
- **DAgger** — consulta expert nos estados visitados pelo aluno.
- **Inverse RL / GAIL** — inferir a reward function a partir de demonstrações.

Prática comum: **bootstrap com BC, refinar com PPO**. AlphaStar fez isso com replays humanos.

## Self-Play

Agente treina jogando contra versões de si mesmo. Só funciona em jogos **simétricos e com vencedor bem definido**, mas quando funciona é o que gera super-humano.

Linhagem DeepMind:
- **AlphaGo** (2016) — policy + value net treinadas em jogos humanos, depois self-play. MCTS no deploy. Bate Lee Sedol.
- **AlphaGo Zero** (2017) — **tabula rasa**, só regras do jogo, sem dados humanos. Bate AlphaGo 100-0.
- **AlphaZero** (2017) — mesma arquitetura generalizada para Go/xadrez/shogi.
- **MuZero** (2019) — sem receber regras; aprende o modelo.

Ingredientes:
1. **Self-play** gera dados infinitos com dificuldade auto-ajustada.
2. **MCTS** (Monte Carlo Tree Search) guiado pela rede — a rede é o prior, MCTS é o "pensar mais".
3. **Policy + value heads** compartilhando backbone.
4. **Ring buffer** de oponentes passados (league play em AlphaStar) evita ciclos / Nash cycling em jogos não-transitivos.

Onde self-play **não funciona de forma limpa**: jogos assimétricos, grandes de info imperfeita, ou com equilíbrios não-transitivos (pedra-papel-tesoura em larga escala). Exigem league play, PSRO, fictitious play.

## POMDPs, memória, exploration

**Memória**:
- **Frame stacking** (N=4 em Atari) resolve velocidade de objetos.
- **RNN/LSTM** dentro da política — padrão antigo.
- **Transformer / attention over trajectory** — estado da arte atual (Gato, AdA, Decision Transformer).

**Exploration**:
- **ε-greedy** — simples, funciona em problemas triviais.
- **Entropy bonus** — adiciona `β·H(π)` ao objetivo. Default em PPO/SAC.
- **Curiosity-driven** (Pathak et al) — reward intrínseco = erro de predição do next_state. Resolveu Montezuma's Revenge sem reward extrínseco.
- **Go-Explore** (Uber AI) — arquiva estados interessantes, retorna e explora a partir deles. Quebrou todos os hard-exploration Atari.
- **RND** (Random Network Distillation) — curiosity via distância a uma rede fixa aleatória.

## Marcos — game-playing agents

| Ano | Agente | Domínio | Contribuição |
|---|---|---|---|
| 1992 | TD-Gammon | Gamão | TD-learning + rede neural, nível mundial |
| 2013 | DQN | Atari 2600 | Deep RL from pixels, replay buffer + target net |
| 2016 | AlphaGo | Go | MCTS + deep nets, bate Lee Sedol |
| 2017 | AlphaZero | Go/xadrez/shogi | Self-play puro, generalização |
| 2018 | OpenAI Five | Dota 2 | PPO em massa, 180 anos/dia de self-play |
| 2019 | AlphaStar | StarCraft II | League play, multi-agent, info imperfeita |
| 2019 | MuZero | + planning | Aprende o modelo do ambiente |
| 2020 | Agent57 | Atari-57 (todos) | Meta-controller sobre família de políticas |
| 2022 | Gato | multi-task | Um transformer para 600+ tarefas |
| 2023 | AdA | XLand 2.0 | Adaptação zero-shot a novas tarefas |
| 2024 | SIMA | 9 jogos 3D | Instruções em linguagem natural → ação |
| 2024 | Genie | video→mundo | World model generativo de jogos 2D |

## Embodied AI / sim-to-real

Agente físico (robô, drone, carro) aprende em simulador e transfere para o mundo real.

Problema: **sim-to-real gap**. Sensor noise, dinâmica não modelada, latência, wear. Política ótima no sim pode ser desastre no real.

Mitigações:
- **Domain Randomization** — randomiza massa, fricção, textura, luz no sim. Política robusta vira distribuição. OpenAI resolveu Rubik's cube com robot hand assim.
- **System Identification** — estima parâmetros reais e calibra o sim.
- **Real2Sim2Real** — coleta dados reais curtos, ajusta sim, re-treina.
- **Residual policy** — política simples projetada + rede aprendendo a correção.
- **Teacher-Student** — teacher em sim com info privilegiada, student com sensores reais via distillation.

Stack: Isaac Gym / Isaac Sim (NVIDIA, GPU-parallel), MuJoCo (físics suave para contato), PyBullet, Gazebo (ROS).

## Arquiteturas clássicas de agentes

- **Reativo puro** (subsumption, Brooks; Braitenberg vehicles) — camadas estímulo→resposta, sem modelo interno. Rápido, robusto, limitado.
- **Deliberativo** — constrói modelo, planeja (MCTS, A*, STRIPS, LLM planner). Poderoso, frágil, lento.
- **BDI** (Belief-Desire-Intention) — framework simbólico para agentes racionais.
- **Três camadas** (reactive / executive / deliberative) — padrão em robótica real. Reflexos rápidos em baixo nível, planejamento em cima.
- **Actor-Critic** — duas cabeças, policy + value, padrão em Deep RL.
- **LLM agent** — política é um LLM chamando ferramentas. Ver `AI_SYSTEMS_DESIGN.md`.

## Multi-agente

Mais de um agente no ambiente. Muda tudo.

Regimes:
- **Cooperativo** — mesma recompensa. Ex: agentes gerenciando intersecção de tráfego.
- **Competitivo** — soma-zero. Self-play cobre.
- **Misto / general-sum** — cooperação e conflito misturados. Negociação, mercado. Equilíbrios múltiplos, emergência de linguagem / sinalização.

Algoritmos: IPPO (independent PPO, ignora outros), MAPPO (centralized training / decentralized execution), QMIX, COMA, MADDPG.

Problemas: **não-estacionariedade** (o ambiente muda porque os outros mudam), **credit assignment** entre agentes, **emergent behavior** (às vezes surpreendente, às vezes dystopic).

## Problemas práticos

- **Sample efficiency** — RL puro precisa de 10⁶-10⁹ interações. Simuladores paralelizam (Isaac Gym: 4096 envs em 1 GPU). Modelo-based e offline RL atacam o problema.
- **Stability** — treino RL é instável, cheio de plateaus e colapsos. **Sempre plotar**: return, entropy, value loss, KL, grad norm.
- **Hyperparameter sensitivity** — LR, batch size, entropy coef, clip ratio. PPO é menos sensível; DDPG é notório.
- **Generalização** — RL overfita ao ambiente de treino. **Procgen** foi criado justamente para medir. Domain randomization, data aug, larger nets ajudam.
- **Credit assignment de longo horizonte** — recompensa 10k passos à frente da ação que causou. n-step TD, RUDDER, return decomposition.
- **Catastrophic forgetting** — continuar treinando em tarefa B destrói competência em A. Multi-task training, EWC, replay de tarefas antigas.
- **Safety** — política pode fazer ações perigosas durante exploration. Shielded RL, constrained MDPs (CMDP), safety layers.

## Benchmarks / ambientes

- **Atari-57 / ALE** — clássico, from pixels, discreto.
- **MuJoCo / DeepMind Control Suite** — locomotion contínuo.
- **Procgen** (OpenAI) — ambientes procedurais para medir generalização.
- **NetHack Learning Env / MiniHack** — procedural, parcialmente observado, long-horizon. Bem mais difícil que Atari.
- **Crafter** — Minecraft-like 2D minimalista, 22 achievements.
- **MineRL / MineDojo** — Minecraft real.
- **Isaac Gym / Isaac Lab** — robótica GPU-parallel.
- **PettingZoo** — multi-agente (ex-OpenAI Gym para MA).
- **SMAC** — StarCraft II Multi-Agent Challenge.
- **XLand** — DeepMind, gerador de tarefas para meta-learning.
- **BabyAI / MiniGrid** — grid-world com instruções linguísticas, ótimo para pesquisa.

## Stack prático

- **PyTorch** é o padrão em RL research hoje (JAX ganhou terreno em DeepMind via Acme/Brax).
- **Stable-Baselines3** — PPO, SAC, DQN prontos, Gymnasium-compatible. Ideal para começar.
- **CleanRL** — implementações single-file, didáticas, fáceis de auditar.
- **RLlib** (Ray) — escala distribuída, production.
- **TorchRL** — stack moderno PyTorch oficial.
- **Tianshou** — alternativa modular.
- **Gymnasium** — sucessor do OpenAI Gym, API padrão.

## Receita mínima (PPO em um jogo Atari)

```python
import gymnasium as gym
from stable_baselines3 import PPO
from stable_baselines3.common.env_util import make_atari_env
from stable_baselines3.common.vec_env import VecFrameStack

env = make_atari_env("PongNoFrameskip-v4", n_envs=8, seed=0)
env = VecFrameStack(env, n_stack=4)

model = PPO(
    "CnnPolicy", env,
    learning_rate=2.5e-4,
    n_steps=128, batch_size=256, n_epochs=4,
    gamma=0.99, gae_lambda=0.95,
    clip_range=0.1, ent_coef=0.01,
    vf_coef=0.5, max_grad_norm=0.5,
    verbose=1, tensorboard_log="./ppo_pong/"
)
model.learn(total_timesteps=10_000_000)
model.save("ppo_pong")
```

Em ~8h de GPU comum o agente vira Pong sólido. Jogos 3D ou StarCraft são **ordens de magnitude** mais caros.

## Debug — o que observar

1. **Episode return** sobe? Se não, nada mais importa.
2. **Explained variance** do critic → 1.0? Se negativo, critic está pior que média constante.
3. **Entropy** colapsa cedo demais? Política está "morrendo", aumenta `ent_coef`.
4. **Approx KL** por update. PPO saudável: 0.01-0.02. Se explode, LR alto ou batch pequeno.
5. **Grad norm**. Picos = divergência iminente. Clip em 0.5 é padrão.
6. **Rewards brutos vs shaped** — **sempre** logar ambos. Shaping pode estar enganando você.
7. **Eval determinístico** em ambiente separado. Treino estocástico, eval sem noise.

## Convergência com LLMs (onde vai dar)

A fronteira atual combina os dois mundos:

- **RLHF / RLAIF** — RL (PPO, DPO, GRPO) refina LLM a partir de preferências. ChatGPT, Claude.
- **LLM-as-planner + RL-as-executor** — LLM decompõe objetivo de alto nível em sub-metas, política RL executa controle baixo-nível. SayCan, VoxPoser.
- **VLA models** (Vision-Language-Action) — RT-2, OpenVLA, π0. Transformer unificado percebe, raciocina, age. Robotics foundation models.
- **World models generativos** — Genie, Sora-like, para gerar ambientes de treino ilimitados.
- **Agent foundation models** — Gato, AdA. Uma política para muitas tarefas via in-context learning.

A aposta: **agentes especialistas (RL puro) + agentes generalistas (LLM) + world models aprendidos** convergem num mesmo transformer multimodal nos próximos anos. O gargalo deixou de ser algoritmo e passou a ser dados de interação em escala.
