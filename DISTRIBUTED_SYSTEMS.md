# Sistemas Distribuídos

Um sistema distribuído é um conjunto de computadores independentes que aparentam, ao usuário, ser um único sistema coerente. A complexidade não está nas máquinas individualmente, mas nas interações entre elas: redes falham, relógios divergem, mensagens se perdem, nós morrem em momentos inconvenientes. A engenharia distribuída é, em boa parte, o estudo de como tolerar essas falhas sem mentir para o usuário.

## As Oito Falácias da Computação Distribuída

Lista clássica atribuída a Peter Deutsch (Sun Microsystems), premissas falsas que iniciantes tendem a assumir:

1. A rede é confiável.
2. A latência é zero.
3. A banda é infinita.
4. A rede é segura.
5. A topologia não muda.
6. Há um administrador.
7. O custo de transporte é zero.
8. A rede é homogênea.

Toda vez que um código distribuído quebra em produção, geralmente uma dessas premissas foi assumida implicitamente.

## Teorema CAP

Formulado por Eric Brewer. Em um sistema distribuído sujeito a **partições de rede (P)**, é impossível garantir simultaneamente:

- **C — Consistência**: toda leitura retorna o dado mais recente escrito ou um erro.
- **A — Disponibilidade**: toda requisição recebe uma resposta (não necessariamente o dado mais recente).

Quando ocorre uma partição, o sistema deve escolher:

- **CP**: rejeita requisições no lado da partição que não tem quórum (ex.: MongoDB em modo majoritário, Zookeeper, etcd).
- **AP**: aceita requisições de ambos os lados e reconcilia depois (ex.: Cassandra, DynamoDB, Riak).

**Cuidado com o mito**: CAP é sobre o comportamento durante a partição, não sobre a "natureza" do banco. Um sistema CP em rede saudável também é disponível. Nem todo sistema classificado como AP é eventualmente consistente da mesma forma.

## PACELC

Extensão de Daniel Abadi que cobre o caso sem partição:

> Se houver **P**artição, troca-se entre **A**vailability e **C**onsistency; **E**lse (rede saudável), troca-se entre **L**atency e **C**onsistency.

Exemplos:
- DynamoDB: **PA/EL** (prioriza disponibilidade e latência).
- Spanner: **PC/EC** (consistência forte, aceita latência maior e reduz disponibilidade em partição).

PACELC é mais realista: mesmo sem partição, consistência forte custa latência (ida e volta a um líder, quórum).

## Modelos de Consistência

Do mais forte ao mais fraco:

1. **Linearizabilidade (strong consistency)**: operações parecem ter ocorrido instantaneamente em alguma ordem global consistente com a ordem real. Cara — exige coordenação.
2. **Sequencial**: todas as réplicas veem as operações na mesma ordem, mas não necessariamente em tempo real.
3. **Causal**: operações causalmente relacionadas são vistas na mesma ordem por todos; operações independentes podem divergir.
4. **Read-your-writes**: um cliente sempre vê suas próprias escritas, mesmo que outros não vejam ainda.
5. **Monotônica**: nunca se vê o relógio voltar (uma leitura mais nova nunca retorna dado mais antigo que uma leitura anterior).
6. **Eventual**: dadas ausência de novas escritas, todas as réplicas convergem para o mesmo valor "eventualmente".

A escolha do modelo define o contrato com a aplicação. Muitos bugs de sistemas distribuídos são aplicações que assumem um modelo mais forte do que o banco oferece.

## Consenso

Problema: fazer N nós concordarem sobre um valor, mesmo com falhas. FLP (Fischer-Lynch-Paterson, 1985) prova que **consenso determinístico é impossível em um sistema assíncrono com mesmo uma única falha**. Na prática, usamos timeouts (o que torna o sistema parcialmente síncrono) para contornar.

### Paxos

Algoritmo clássico de Lamport (1989). Correto, mas notoriamente difícil de entender e implementar corretamente. Variantes: Multi-Paxos, Cheap Paxos, Fast Paxos.

### Raft

Desenhado por Ongaro e Ousterhout (2014) para ser **compreensível**. Separa explicitamente:

- **Eleição de líder**.
- **Replicação de log**.
- **Segurança**.

Usado por etcd, Consul, CockroachDB, TiKV. Na prática, é hoje o algoritmo de consenso mais implementado.

### Quórum

Em vez de consenso total, muitos sistemas usam quóruns (`W + R > N`, onde N é o número de réplicas, W escritas, R leituras). Garante que toda leitura intersecta pelo menos uma escrita recente.

## Relógios

### Problema

Relógios de parede divergem entre máquinas. NTP reduz mas não elimina o drift. Comparar timestamps de máquinas diferentes é enganoso.

### Relógios Lógicos

- **Lamport timestamps**: contador monotônico que é incrementado em cada evento e propagado em mensagens. Estabelece ordem parcial (`a → b` implica `L(a) < L(b)`), mas não identifica eventos concorrentes.
- **Vector clocks**: um contador por nó. Detecta concorrência (se nem `V(a) ≤ V(b)` nem `V(b) ≤ V(a)`, são concorrentes). Caro em sistemas com muitos nós.
- **HLC (Hybrid Logical Clocks)**: combina timestamp físico com contador lógico. Usado em CockroachDB.

### TrueTime (Google Spanner)

Hardware (GPS + relógios atômicos) garante que o erro do relógio é limitado (`ε`). Spanner espera `2ε` antes de confirmar uma transação, garantindo linearizabilidade externa sem coordenação global. Caro e específico.

## Falhas e Padrões

### Split-Brain

Partição de rede separa o cluster em dois lados, cada um achando que é majoritário. Resultado: escritas conflitantes. Mitigação: quórum majoritário (um lado não consegue formar quórum) ou *fencing token* (um número monotônico que impede um líder antigo de causar dano depois de substituído).

### Failover e Fencing

Quando um líder é substituído, o antigo pode estar vivo mas "lento" (GC pause, swap). Se ele voltar achando que ainda é líder, corrompe o sistema. Fencing tokens (Chubby, ZooKeeper) resolvem: toda operação externa precisa apresentar o token mais recente.

### Replicação

- **Síncrona**: primário espera confirmação das réplicas antes de commit. Durável, mas aumenta latência e acopla disponibilidade.
- **Assíncrona**: primário commita e replica depois. Rápido, mas pode perder dados em failover.
- **Semi-síncrona**: espera pelo menos uma réplica. Meio-termo comum (MySQL, PostgreSQL).

### Particionamento (Sharding)

Divisão horizontal de dados por chave. Estratégias: hash, range, lookup table. Ver também `SHARDING.md`.

## Observações Práticas

1. **Evite sistemas distribuídos quando possível**: uma máquina grande é mais simples que um cluster pequeno. Distribuição é um custo, não um benefício em si.
2. **Idempotência não é opcional**: redes reentregam mensagens. Ver `IDEMPOTENCY.md`.
3. **Timeouts são inevitáveis e enganosos**: um timeout não te diz se a operação aconteceu ou não. Sempre assuma que pode ter acontecido.
4. **Teste com falhas injetadas**: Jepsen mostrou que praticamente todo banco distribuído tem bugs de consistência sob partição. Caos engineering não é luxo — ver `CHAOS_ENGINEERING.md`.
5. **Leia Jepsen e o Martin Kleppmann**: "Designing Data-Intensive Applications" é, hoje, a referência prática mais importante.
