# Concorrência

Concorrência é a habilidade de um programa **progredir em múltiplas tarefas** ao mesmo tempo. Paralelismo é quando essas tarefas efetivamente **executam simultaneamente** em múltiplos núcleos. Os conceitos são distintos: um sistema pode ser concorrente sem ser paralelo (um único núcleo alternando tarefas) e, em teoria, paralelo sem ser concorrente (mas isso é raro fora de SIMD).

> "Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once." — Rob Pike

## Condições de Corrida

Uma *race condition* ocorre quando o resultado de uma computação depende da ordem de execução de operações concorrentes, e essa ordem não é garantida.

Exemplo clássico: dois threads incrementando um contador:

```
thread1: x = x + 1
thread2: x = x + 1
```

Se `x` começa em 0, pode terminar em 2 (correto) ou em 1 (incremento perdido). Porque `x = x + 1` é, na verdade, três operações: ler, somar, escrever. Entre a leitura de um thread e sua escrita, o outro pode ter passado.

## Primitivos

### Mutex (Mutual Exclusion)

Permite que apenas uma thread esteja em uma seção crítica por vez. Outras bloqueiam até o mutex ser liberado.

```
mutex.lock()
// seção crítica
mutex.unlock()
```

**Cuidados**:
- Sempre liberar, mesmo em erro (`defer`, `try/finally`, RAII).
- **Granularidade**: mutex muito amplo (lock no método inteiro) serializa tudo; muito estreito introduz races nas bordas. Meça.
- **Ordem de aquisição**: se múltiplos mutexes, lock sempre na mesma ordem para evitar deadlock.

### RWMutex (Read/Write Lock)

Muitos leitores simultâneos OU um escritor exclusivo. Útil quando leituras dominam. Cuidado: pode causar *writer starvation* se leituras nunca param.

### Semáforo

Controla acesso a um recurso com capacidade N. Generaliza o mutex (mutex = semáforo com N=1). Útil para limitar concorrência (ex.: no máximo 10 conexões ao banco).

### Condition Variable

Permite que uma thread **espere** por uma condição e seja **notificada** quando outra a satisfaz. Usado em padrões produtor-consumidor.

```
mutex.lock()
while (!condition) cond.wait(mutex)  // libera o mutex e dorme; reacquire ao acordar
// condição satisfeita
mutex.unlock()
```

O `while` (não `if`) é essencial: *spurious wakeups* existem, e a condição pode ter sido invalidada entre o signal e o reaquire.

### Atômicos

Operações de hardware que executam indivisivelmente: `compare-and-swap` (CAS), `fetch-and-add`, etc. Base de algoritmos lock-free. Rápidos, mas fáceis de usar errado — memory ordering (ver abaixo) é sutil.

## Memory Models

Em CPUs modernas, o compilador e o hardware reordenam operações de memória para performance. O **memory model** define quais reordenações são permitidas e como a sincronização as contém.

### Happens-Before

Relação fundamental: se operação A *happens-before* B, B vê os efeitos de A. Pode ser estabelecida por:

- Ordem do programa numa mesma thread.
- Unlock de um mutex happens-before o próximo lock do mesmo mutex.
- Escrita em variável atômica com `release` happens-before leitura com `acquire`.
- Criação de thread happens-before a primeira instrução da thread filha.
- Término de uma thread happens-before o `join` correspondente.

Sem uma relação happens-before, uma thread **pode não ver** escritas de outra — mesmo que "no tempo real" tenham ocorrido antes. Esse é o grande armadilha: o bug não é que leu o valor antigo, é que pode ler **qualquer valor**.

### Ordens de Memória (C++/Rust)

Do mais relaxado ao mais estrito:

- `relaxed`: sem garantias de ordem além da atomicidade em si.
- `acquire` (em loads): nada posterior é reordenado para antes.
- `release` (em stores): nada anterior é reordenado para depois.
- `acq_rel`: combina ambos.
- `seq_cst`: ordem total sequencialmente consistente entre todas as ops `seq_cst`.

**Regra prática**: se não tem certeza, use `seq_cst` (é o default em várias linguagens). `relaxed` é para contadores puros e padrões muito específicos.

## Modelos de Concorrência

### Shared Memory + Locks

Threads compartilham memória; coordenação via mutex/atomic. Dominante em C++/Java/C#. Poderoso, mas bugs (deadlock, race, livelock) são fáceis e difíceis de reproduzir.

### Message Passing (CSP, Actors)

Unidades de execução isolam estado e se comunicam por mensagens.

- **CSP (Communicating Sequential Processes)**: gorotinas + channels em Go. "Don't communicate by sharing memory; share memory by communicating."
- **Actor model**: Erlang, Elixir, Akka. Cada actor tem mailbox; mensagens são assíncronas. Isolamento forte — crash de um actor não afeta outros (supervisão).

Geralmente mais fácil de raciocinar que shared memory, mas não é bala de prata.

### Async / Await (Cooperative)

Uma tarefa cede voluntariamente o controle em pontos de `await`. Um runtime escalona. Sem preempção → sem race em variáveis da mesma task, mas ainda há race *entre* tasks que compartilham dados.

Principal ganho: **alta concorrência de I/O** sem o custo de threads OS. Ruby Fibers, Python asyncio, JS Promises/async, Rust Tokio, C# Tasks, Java Virtual Threads (Loom).

**Armadilha clássica**: chamar código bloqueante dentro de um handler async → bloqueia o event loop → todo o sistema engasga.

### STM (Software Transactional Memory)

Transações em memória: operações executam, e ao final, confirmam atomicamente (ou fazem rollback e reexecutam). Elimina locks explícitos. Usado em Haskell (`STM`) e Clojure. Performance varia; em domínios certos é elegante.

## Deadlock

Quatro condições de Coffman (todas necessárias):

1. **Mutual exclusion**: recurso não compartilhável.
2. **Hold and wait**: thread segura recurso e espera outro.
3. **No preemption**: recurso só sai voluntariamente.
4. **Circular wait**: T1 → T2 → ... → T1.

Quebre qualquer uma:

- Ordem global de aquisição (quebra a circular).
- Timeout no lock (quebra hold-and-wait).
- `tryLock` + backoff.
- Reduzir granularidade (menos locks).

## Livelock e Starvation

- **Livelock**: threads estão ativas mas sem progredir (ex.: dois se esquivando no corredor).
- **Starvation**: uma thread nunca consegue recurso porque outras sempre chegam primeiro (ex.: escritor em RWMutex com leitores constantes).

Solução: fairness no scheduler / locks, aging de prioridades.

## Padrões Úteis

1. **Immutable data**: se nada muda, não há race (ver `IMMUTABLE.md`).
2. **Thread confinement**: dado visitado por uma única thread — sem sync necessária.
3. **Copy on write**: em vez de lock para modificar, crie nova versão e troca o ponteiro atomicamente.
4. **Read-copy-update (RCU)**: leitores sem lock; escritor prepara cópia e troca.
5. **Work stealing**: threads ociosas "roubam" trabalho de outras ocupadas. Base dos schedulers modernos (Tokio, ForkJoinPool).
6. **Single-threaded core + async I/O**: Node.js, Redis. Simplifica o modelo mental ao custo de escalar horizontalmente para usar mais CPU.

## Princípios Práticos

1. **Menos concorrência é melhor**: paralelize o suficiente para hit o objetivo de performance, não mais.
2. **Testes concorrentes são insuficientes**: race conditions não aparecem em CI determinístico. Use *race detectors* (`go test -race`, TSAN em C++).
3. **Bibliotecas são aliadas**: `java.util.concurrent`, `golang.org/x/sync`, Rust `tokio::sync`. Não reimplemente mutex ou queue concorrente por diversão.
4. **Benchmarking distingue**: um design "mais escalável" no papel pode ser mais lento com 4 cores que um mutex simples. Meça.
5. **Log de eventos com timestamp é ouro**: bugs concorrentes debugam-se com narrativa temporal, não com debugger.
6. **Reduza estado compartilhado**: quanto mais imutabilidade e confinamento, menos superfície de bug.
