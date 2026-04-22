# Profiling

*Profiling* é a arte de medir **onde** e **por quê** um programa gasta tempo, memória ou outros recursos. Sem profiling, otimização é especulação — e, como adverte `PREMATURE_OPTIMIZATION.md`, especulação é quase sempre errada.

> "Measure. Don't tune for speed until you've measured, and even then don't unless one part of the code overwhelms the rest." — Rob Pike, Rules of Programming

## Regra Antecedente

Antes de profiling, **defina o objetivo**:

1. Qual métrica importa? (latência p99, throughput, uso de memória, custo em nuvem)
2. Qual é o alvo? ("<200ms" é mensurável; "rápido" não é)
3. Qual é o carregamento representativo? Profiling com tráfego sintético pode iluminar hotspots que não existem em produção (e vice-versa).

Sem isso, profiling vira turismo.

## Tipos de Profiler

### Sampling Profiler

Periodicamente (ex.: 100Hz) tira um "retrato" da pilha de chamadas. Funções que aparecem mais são onde o tempo está. **Baixo overhead** — geralmente seguro em produção.

Ferramentas: `perf` (Linux), `pprof` (Go), `py-spy` (Python), async-profiler (JVM), Instruments (macOS), Xcode, `cargo flamegraph` (Rust).

### Instrumenting Profiler

Insere medições no código (entra/sai de função). Dados precisos sobre cada chamada, mas **overhead alto** — distorce a execução. Não rodar em produção sob carga real.

Ferramentas: `gprof`, cProfile (Python, determinístico), Java profilers em modo instrumentado.

### Tracing Profiler

Registra eventos específicos (syscalls, locks, I/O). Poderoso para latências não-CPU. Ex.: `strace`, `eBPF`, `ftrace`, `bpftrace`.

## O Que Medir

### CPU

Onde ciclos são gastos. Responde: "qual função é hotspot?"

- `perf record -F 99 -p <pid> -g -- sleep 30` → `perf report`
- `go test -cpuprofile cpu.prof` → `go tool pprof`
- Python: `py-spy top --pid <pid>` ou `py-spy record -o out.svg`
- JVM: `async-profiler` com `-e cpu`

### Alocação / Heap

Onde memória é alocada. Resolve vazamentos e reduz pressão no GC.

- Go: `go test -memprofile mem.prof`, `pprof -alloc_space`
- Java: `jmap -dump`, JFR, `async-profiler -e alloc`
- Python: `tracemalloc`, `memray`
- Rust: `dhat`, `heaptrack`, valgrind massif
- Node: Chrome DevTools heap snapshot, `clinic doctor`

Distinção importante:
- **Alocação total**: quanto foi alocado ao longo do tempo. Afeta pressão no GC.
- **Retenção**: o que *ainda está vivo*. Afeta footprint e leaks.

### Latência (Off-CPU)

Tempo gasto **esperando** (I/O, locks, GC, descheduling). CPU profiling não captura. Use:

- `perf sched` — atraso de scheduling.
- `async-profiler -e wall` — wall-clock (inclui espera).
- eBPF: `offcputime` do bcc.

### I/O e Syscalls

- `strace -c <cmd>` — resumo de syscalls.
- `iostat`, `iotop`, `blktrace` — I/O de disco.
- `ss`, `tcpdump`, `iftop` — rede.
- eBPF tools: `biosnoop`, `tcptop`, `execsnoop`.

### Locks e Contenção

- Go: `-blockprofile`, `-mutexprofile`.
- Java: JFR "Thread Park" e "Lock Contention" events.
- eBPF: `offwaketime`.

## Flamegraphs

Popularizados por Brendan Gregg. Visualização onde:

- **Largura** de uma função = proporção do tempo gasto nela (inclusive filhas).
- **Altura** = profundidade de pilha.
- Cor geralmente aleatória (ajuda a distinguir), não semântica.

Leitura: procurar **platôs largos no topo** — são folhas onde tempo é realmente consumido. Empilhamento alto sem topo largo = chamadas profundas mas rápidas, não é o hotspot.

Variantes úteis:
- **Differential flamegraph**: compara duas execuções (antes/depois da otimização).
- **Off-CPU flamegraph**: onde o programa está *esperando*, não computando.
- **Icicle graph**: invertido (raiz no topo). Igualmente válido.

Gerar: [FlameGraph scripts](https://github.com/brendangregg/FlameGraph) de Brendan Gregg, ou direto no `pprof -http`, `async-profiler -o flamegraph`.

## Método USE (para recursos)

Para cada recurso do sistema (CPU, memória, disco, rede):

- **Utilization**: % do tempo ocupado.
- **Saturation**: fila de trabalho além da capacidade.
- **Errors**: contagem de falhas.

USE é checklist inicial para problemas de infra. Complementa RED (para serviços). Ver `OBSERVABILITY.md`.

## Princípios

### 1. Meça, não adivinhe

Intuições sobre "onde o tempo vai" são sistematicamente erradas. O compilador, o JIT, o cache da CPU, o GC introduzem efeitos contra-intuitivos. Profiling é o juiz.

### 2. Amdahl's Law

Se uma parte consome 20% do tempo e você a elimina completamente, o programa fica **20% mais rápido**, não 100%. Maximize impacto: otimize o hotspot, não o que é interessante otimizar.

`speedup = 1 / ((1-p) + p/s)` onde *p* é fração afetada e *s* é o fator de speedup local.

### 3. Otimize o caminho quente

80/20 geralmente manda: poucos caminhos dominam o tempo. Foque neles. Otimizar cold paths só piora legibilidade sem ganho mensurável.

### 4. Cuidado com micro-benchmarks

Benchmark isolado pode mostrar ganho que desaparece (ou inverte) em contexto real, por causa de cache, branch prediction, inlining do JIT. Prefira benchmarks de ponta-a-ponta quando possível.

### 5. Ambiente importa

CPU profiling em laptop != produção. Diferenças em:
- Número de cores, NUMA, cache sizes.
- Frequency scaling (turbo boost).
- Carga concorrente (vizinhos barulhentos).
- Versão de kernel, flags do compilador.

Profile em ambiente semelhante ao de produção sempre que possível. *Continuous profiling* (Pyroscope, Parca, Google Cloud Profiler, Datadog) resolve executando em produção com baixo overhead.

## Armadilhas Comuns

1. **Otimizar antes de medir**: violação direta do princípio. Corrigir: sempre medir primeiro.
2. **Medir o código errado**: profiling local quando o bottleneck é de rede ou de banco. Olhe end-to-end antes de mergulhar.
3. **Profiler escondendo o problema**: instrumenting profilers alteram o programa. Use sampling em produção.
4. **"100% de CPU" não é necessariamente gargalo**: pode ser um thread fazendo spin. Complementar com off-CPU profile.
5. **Alocação vs retenção confundidas**: um programa pode alocar TBs e não vazar (GC trabalha); outro pode alocar pouco e vazar centenas de MB. Ferramentas distinguem — use-as.
6. **Otimização invisível do compilador**: em linguagens compiladas, funções podem ser inlineadas, mortas, ou reordenadas. Olhar o assembly (`go tool compile -S`, `godbolt.org`) esclarece.
7. **JIT warmup**: na JVM/V8, medidas iniciais são pessimistas. Aqueça o benchmark antes de medir.

## Fluxo Prático

1. Define objetivo e métrica de sucesso.
2. Reproduza o cenário problemático de forma confiável.
3. Capture profile com a ferramenta adequada ao recurso suspeito.
4. Gere flamegraph ou equivalente. Identifique hotspot.
5. Hipótese sobre causa. Mudança pequena.
6. Remeça. Compare com baseline. Flamegraph diferencial ajuda.
7. Se ganho confirmado → mantenha. Senão, reverta e tente outro ângulo.

Profiling sem reversão é superstição: "parece melhor" não é medição.
