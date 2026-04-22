# Sistemas Operacionais

O **sistema operacional (OS)** é o software que media hardware e aplicações: abstrai CPU, memória, dispositivos e rede em primitivas uniformes, isola processos uns dos outros, escalona recursos, e provê APIs (syscalls) ao userspace. Entender OS é entender **onde o tempo vai** quando um programa não comporta como esperado — context switch, faulting, bloqueio em I/O, contenção de locks kernel.

Este arquivo foca em **Unix-like** (Linux/BSD/macOS) por ubiquidade; Windows com menções. Complementa `COMPUTER_ARCHITECTURE.md`, `STORAGE.md`, `CONCURRENCY.md`, `CIRCUIT.md`, `HIDEPID.md`, `APP_SANDBOX.md`, `MEMORY_OPTIMIZATION.md`.

## Kernel Modes e Userspace

CPU moderna oferece múltiplos **rings** / **privilege levels**:

- **Ring 0 (kernel mode)**: acesso total. Kernel reside aqui.
- **Ring 3 (user mode)**: instruções privilegiadas bloqueadas. Apps rodam aqui.
- Rings intermediários: historicamente sub-utilizados.

Transição user → kernel via:

- **Syscall**: instrução (`syscall` x86-64, `svc` ARM) que comuta modo.
- **Interrupt**: hardware notifica (timer, I/O, exception).
- **Trap / fault**: page fault, divide-by-zero, invalid instruction.

## Arquiteturas de Kernel

### Monolítico

Kernel inteiro em espaço privilegiado único. Drivers, filesystem, rede — tudo em kernel.

- **Pros**: performance (tudo sem IPC), simplicidade histórica.
- **Contras**: bug em driver pode derrubar sistema; superfície grande.
- **Exemplos**: Linux, BSDs.

### Microkernel

Kernel mínimo (scheduling, IPC, memória). Drivers e serviços em userspace.

- **Pros**: robustez (crash de serviço não derruba sistema), modularidade.
- **Contras**: mais IPC = overhead.
- **Exemplos**: L4, Minix, seL4 (verified), QNX.

### Híbrido

Compromisso. NT (Windows), XNU (macOS/iOS: Mach microkernel + BSD server).

### Unikernels / Library OS

Aplicação + OS compilados como imagem única, rodam em hypervisor. Mirage, Unikraft. Nicho (edge, serverless de alta performance).

## Linux

Kernel monolítico modular — módulos carregados dinamicamente (drivers). Dominante em server, cloud, mobile (Android), embarcado.

### Componentes

- **Scheduler**: CFS (Completely Fair Scheduler, padrão) → **EEVDF** desde 6.6 (2023). Real-time: SCHED_FIFO, SCHED_RR.
- **Memory management**: virtual memory, paging, swap, OOM killer.
- **VFS**: Virtual File System — abstração sobre ext4, XFS, Btrfs, NFS, etc.
- **Networking stack**: TCP/IP, netfilter (firewall), eBPF.
- **Block layer**: I/O scheduler, request merging.
- **Drivers**: hardware.

### Init e userspace

- **systemd**: init moderno, service manager, logging (journald), networkd, resolved, etc. Domina distros mainstream.
- **OpenRC, runit, s6**: alternativas. Alpine, Void, Artix.
- **Launchd** (macOS), **SMF** (Solaris), **SCM** (Windows Services).

## Processo e Thread

### Processo

Unidade de alocação de recursos:

- **PID**.
- **Memory space** (virtual address space próprio).
- **File descriptors** (tabela).
- **Credenciais** (UID, GID, capabilities).
- **Signals**.
- **Environment**, argumentos, working directory.

### Thread

Unidade de execução dentro de processo. Múltiplas threads compartilham memória e file descriptors mas têm:

- **Stack** própria.
- **Registradores** próprios.
- **TLS (Thread-Local Storage)**.

### Modelos de thread

- **1:1 (kernel threads)**: cada pthread é task do kernel. Linux, modern Windows.
- **N:1 (user threads)**: biblioteca agenda. Lightweight mas perde paralelismo real.
- **M:N**: híbrido. Goroutines em Go são M:N sobre kernel threads.

### fork, exec, clone

Unix:

- `fork()`: cria processo filho com copy-on-write das páginas.
- `exec()`: substitui imagem do processo por outra (mantém PID).
- `wait()`: pai espera filho.
- `clone()` (Linux): generalização — compartilha o quê (namespaces, FS, VM) via flags. Base de threads e containers.

**Windows**: `CreateProcess()` único passo (sem fork). Posix emulation via WSL ou Cygwin.

## Scheduling

### Objetivos (conflitantes)

- **Throughput** máximo.
- **Latência** / **responsiveness** baixa.
- **Fairness**.
- **Priority respect**.
- **Power efficiency** (mobile).

### Políticas Linux

- **SCHED_OTHER** (CFS/EEVDF): compartilha tempo proporcionalmente a *nice* (-20 a 19).
- **SCHED_BATCH**: throughput, CPU-heavy.
- **SCHED_IDLE**: só quando nada mais.
- **SCHED_FIFO**: real-time, preemptive por prioridade.
- **SCHED_RR**: real-time round-robin dentro de prioridade.
- **SCHED_DEADLINE**: EDF — garante deadline se admitido.

### EEVDF (Earliest Eligible Virtual Deadline First)

Substitui CFS desde kernel 6.6 (2023). Mesma ideia fair mas com "lag" e "deadline" explícitos — melhor latência em mixed workloads.

### Multi-core e NUMA

- **Affinity** (`taskset`, `pthread_setaffinity_np`): prender thread a cores.
- **NUMA balancing**: kernel migra páginas para nó local.
- **CPU sets (cgroups cpuset)**: isolar subsets.
- **Isolcpus, nohz_full** em real-time / HPC: dedicar cores, desligar ticks.

### big.LITTLE / heterogeneous

- **Energy-aware scheduling (EAS)** em Linux mobile/Apple.
- **Intel Thread Director** (Alder Lake+) dá hints.
- **Apple QoS** APIs permitem processo dizer prioridade.

## Memory Management

### Virtual Memory

Cada processo tem espaço virtual próprio. Traduções via **page tables**:

- **4KB páginas** padrão; **2MB (huge)**, **1GB (gigantic)** reduzem TLB pressure.
- **4-level** (x86-64 48-bit), **5-level** (57-bit) page tables.
- **CR3** (x86) / **TTBR** (ARM) aponta para root da tabela.

### Paging e Swap

- **Demand paging**: página carrega só quando tocada.
- **Page fault**: acesso a página ausente. Minor (não no disco) ou major (disco).
- **Swap**: páginas não usadas vão a disco. `swappiness` (0-100) controla agressividade.
- **zram, zswap**: swap compactado em RAM. Útil em memória apertada.

### Kernel memory

- **kmalloc** (contíguo), **vmalloc** (virtualmente contíguo), **slab/slub allocator** (objetos pequenos caches).
- **Buddy allocator**: páginas contíguas, potência de 2.

### OOM killer

Quando kernel esgota memória, mata processo (score baseado em uso, nice, OOM adjustment). Configurar (`oom_score_adj`) ou prevenir (cgroups memory.max).

### Page cache

Mais em `STORAGE.md`. RAM livre é usada como cache de disco.

### mmap e shared memory

- **mmap**: mapear arquivo/memória em espaço virtual. Base de bibliotecas dinâmicas, POSIX shared memory.
- **shm_open**, **memfd_create**: anônimas.
- **Copy-on-write**: após `fork`, páginas só duplicam quando escritas.

### Segmentação

Historicamente (x86 16/32-bit). Em x86-64 é vestigial; flat segments com paging faz o trabalho.

## Syscalls

Interface user ↔ kernel.

- **POSIX**: `read`, `write`, `open`, `close`, `fork`, `exec`, `wait`, `pipe`, `socket`, `mmap`, `futex`, `clock_gettime`, `nanosleep`, etc.
- **Linux-specific**: `clone`, `epoll`, `inotify`, `io_uring`, `bpf`, `perf_event_open`, `memfd_create`.
- **strace**: rastrear syscalls de processo.
- **ltrace**: biblioteca-level.

### Fast path

- **vDSO**: biblioteca injetada no user space com implementações rápidas de syscalls frequentes (`gettimeofday`, `clock_gettime`) sem entrar no kernel.
- **io_uring**: submission + completion queues compartilhadas; amortiza ou elimina syscall por operação.

## I/O

### Modelos

1. **Blocking**: thread bloqueia até I/O completar.
2. **Non-blocking**: retorna imediatamente; pollar.
3. **Multiplexed**: `select`, `poll`, `epoll` (Linux), `kqueue` (BSD/macOS), `IOCP` (Windows).
4. **Async (AIO)**: submit + callback/completion. `io_uring` no Linux; histórico `aio` limitado.
5. **Signal-driven**: SIGIO em mudança de FD. Raro.

### epoll

Pilar de servidores de rede em Linux. Registra FDs de interesse; kernel notifica mudanças. Escalável a milhões de FDs.

### io_uring

Introduzido em 2019. Rings compartilhados user/kernel:

- **SQ**: user submete operações (read, write, send, recv, open, stat, etc.).
- **CQ**: kernel completa.
- Multi-shot, linked ops, kernel polling sem syscall.
- Uso crescente em BD, proxies, storage engines.

## Interrupts

- **Hardware interrupts**: timer, NIC, disk.
- **Software interrupts (softirq)**: kernel difere trabalho.
- **Threaded interrupts**: kernel move handler para thread (RT).
- **APIC, MSI, MSI-X**: entrega em x86.

Interrupts afetam latência — tuning (IRQ affinity, balancer) importa em servidores.

## File System

Ver `STORAGE.md` para camada storage. Aqui, visão OS:

- **VFS** unifica FS heterogêneos.
- **Inodes**: metadata (permissions, owner, timestamps, links).
- **Dentry cache**: path → inode.
- **Page cache**: conteúdo de arquivos.
- **Extended attributes (xattr)**, **ACL POSIX**, **capabilities**.

### Montagem

- Mount table, namespaces (mount namespace isola views).
- `/proc`, `/sys`, `/dev`, `/run`: virtuals.
- **fusermount / FUSE**: FS em userspace.

## Rede no Kernel

- **Socket API**: BSD sockets — família (AF_INET, AF_INET6, AF_UNIX, AF_NETLINK, AF_PACKET).
- **Stack TCP/IP**: completa in-kernel.
- **Netfilter / iptables / nftables**: firewall framework.
- **eBPF + XDP**: filtrar/programar no ingress cedíssimo, bypass stack.
- **RSS, RPS**: scaling em múltiplas CPUs.
- **AF_XDP, DPDK**: kernel bypass para ultra low latency.

## IPC

- **Pipes, FIFOs, Unix sockets**: tradicional.
- **Shared memory (POSIX shm, System V shm)**.
- **Signals**: assíncrono, limitado.
- **Message queues (POSIX mq, SysV)**.
- **Eventfd, signalfd, timerfd** (Linux): integráveis a epoll.
- **Sockets AF_UNIX / abstract namespace**: default em D-Bus, muitos serviços.
- **D-Bus**: sistema de mensagens desktop/system Linux.
- **Binder** (Android): IPC principal.
- **Mach ports** (macOS): IPC core.
- **Futex** (Linux): fast userspace mutex — base de pthread mutexes.

## Namespaces e Cgroups (Linux)

Base de containers (ver `CONTAINER_SECURITY.md`):

### Namespaces

- **mount**: visão de FS própria.
- **pid**: PIDs próprios; PID 1 no namespace.
- **net**: interfaces, rotas, iptables próprios.
- **uts**: hostname.
- **ipc**: SysV IPC.
- **user**: UID mapping.
- **cgroup**: hierarquia cgroup própria.
- **time**: CLOCK_BOOTTIME, CLOCK_MONOTONIC deslocados (kernel 5.6+).

### Cgroups (v2 moderno)

Controla recursos por grupo de processos:

- **cpu**: cotas (`cpu.max`).
- **memory**: limite, swap.
- **io**: throttle.
- **pids**: número máximo.
- **network (via net_cls + tc)**: rate shaping.

Systemd usa cgroups extensivamente.

## Segurança

- **DAC (Discretionary Access Control)**: permissões clássicas (user/group/other × rwx), ACL.
- **MAC (Mandatory)**:
  - **SELinux**: policy baseada em tipos, transitions. Default em Red Hat.
  - **AppArmor**: paths-based, simpler. Default em Ubuntu.
  - **SMACK**: simpler alt.
- **Capabilities** (Linux): decompõe root em ~40 capabilities (CAP_NET_ADMIN, CAP_SYS_ADMIN, etc.).
- **Seccomp**: filtro de syscalls via BPF.
- **Landlock** (Linux 5.13+): confinement userspace-initiated.
- **Pledge / unveil** (OpenBSD): abordagem elegante.
- **Namespaces + cgroups + seccomp + MAC**: ingredientes de sandbox e containers.
- **Secure boot, kernel signing, kernel lockdown, module signing**.

Ver `APP_SANDBOX.md`, `CONTAINER_SECURITY.md`.

## Boot Process

1. **Firmware** (UEFI/BIOS): POST, loads bootloader.
2. **Bootloader** (GRUB, systemd-boot, rEFInd, U-Boot): escolhe kernel, carrega initramfs.
3. **Kernel**: descompacta, inicializa drivers, monta rootfs.
4. **Init / systemd**: PID 1 — services, targets, login.
5. **Login shell / display manager**.

Secure Boot verifica cadeia; TPM mede; atestação comprova integridade.

## Debugging e Observabilidade

- **strace, ltrace, perf, ftrace, SystemTap, eBPF tools (bcc, bpftrace)**.
- **/proc/[pid]/**: `status`, `maps`, `fd/`, `stack`, `smaps`, `status`.
- **/sys**: sysfs — devices, kernel parameters.
- **dmesg**: kernel log.
- **journalctl**: systemd log.
- **top, htop, btop, atop**: processos.
- **iostat, vmstat, mpstat, sar**: recursos.
- **iotop, iftop, nethogs**: I/O, rede por processo.
- **lsof**: arquivos/sockets abertos.
- **ss**: sockets.
- **pmap**: mapeamento de memória.
- **gdb, lldb**: debugger userspace; **kgdb**, crash (kernel).

eBPF é o framework moderno — observabilidade sem overhead alto, sem patches.

## macOS / XNU

- **Kernel híbrido**: Mach microkernel (IPC via ports, tasks/threads) + BSD layer (VFS, sockets, POSIX APIs) + I/O Kit (drivers C++).
- **launchd**: init/service manager.
- **SIP (System Integrity Protection)**: proteção de /System, /usr.
- **Gatekeeper, XProtect**: anti-malware.
- **Sandbox** (seatbelt): per-app profiles.
- **TCC (Transparency, Consent, Control)**: acesso a APIs sensíveis (camera, mic, contacts) por app.
- **APFS** como filesystem (ver `STORAGE.md`).

## Windows / NT

- **Hybrid kernel**: `ntoskrnl.exe` com HAL + executive + kernel.
- **Objects** model: uniform naming/handles.
- **Scheduler**: priority-based preemptive, 32 levels.
- **Subsystems**: Win32 dominante; WSL (Linux), POSIX histórico.
- **Registry**: config hierárquico.
- **DLLs** dinâmicas; **services** gerenciados via SCM.
- **WinAPI, NT API, ETW (Event Tracing for Windows)**.
- **HAL**, **minifilters**, **drivers**.
- **Virtualization-Based Security (VBS)**, **HVCI**, **Credential Guard**: usa Hyper-V para isolar.

## Android

- Linux kernel + Android runtime (ART) + frameworks.
- **Binder** é IPC principal.
- **SELinux** estrito.
- **Zygote**: processo ancestral; apps são forks (prewarm otimização).
- **HALs**: hardware abstraction; padronizado via HIDL/AIDL.

## Real-Time OS

Sistemas com garantias temporais duras:

- **FreeRTOS, Zephyr, ThreadX (Azure RTOS), VxWorks, RTEMS, QNX, VxWorks**.
- **PREEMPT_RT** (Linux): reduz latência, atingiu mainline.
- Tradeoff: throughput por latência determinística.

## Tópicos avançados

### Live patching

Aplicar patches kernel sem reboot. Kpatch (Red Hat), Livepatch (Canonical), Oracle Ksplice.

### Hypervisor e virtualização

- **KVM** (Linux kernel): VT-x/AMD-V para VMs. QEMU para emulação.
- **Xen**: tipo 1 (bare metal).
- **Hyper-V, VMware ESXi, bhyve, jails**.
- **Paravirtualização** vs **full virtualization**.
- **Nested virtualization**.

### Containers vs VMs

Container: namespaces + cgroups — compartilha kernel. VM: hypervisor — kernel próprio. Trade-off entre isolamento e overhead.

## Recursos

- **"Operating Systems: Three Easy Pieces"** — Arpaci-Dusseau. Gratuito, excelente.
- **"Modern Operating Systems"** — Tanenbaum.
- **"Linux Kernel Development"** — Robert Love.
- **"Understanding the Linux Kernel"** — Bovet & Cesati (antigo mas clássico).
- **"The Linux Programming Interface"** — Michael Kerrisk.
- **"Windows Internals"** — Russinovich, Solomon.
- **LWN.net**: news e deep dives sobre Linux kernel.
- **kernel.org docs**, **man7.org** (Kerrisk's pages).

## Princípios

1. **Abstrações vazam**. OS esconde hardware mas com limites; em perf crítica, sempre desce um nível.
2. **Syscall custa µs**. Fast paths (vDSO, io_uring) evitam quando possível.
3. **Context switch custa também**. Thread count ≠ melhor throughput.
4. **Page cache é grátis em RAM livre**. Benchmark de I/O frio ≠ quente.
5. **Cgroups + namespaces são primitivos**. Containers são construção em cima.
6. **Observe antes de adivinhar**. strace, perf, bpftrace revelam o que realmente acontece.
7. **Cada OS tem filosofia**. Linux prioriza flexibilidade; macOS, consistência; Windows, compatibilidade; BSD, limpeza. Respeitar.
