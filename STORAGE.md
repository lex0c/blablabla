# Storage

Storage é a pilha que preserva dados entre energizações — do transistor de flash NAND ao filesystem ao bloco mirrored em datacenter remoto. Entender storage é entender **onde latência vai** em quase qualquer sistema não-trivial: um SSD rápido tem 10⁵x a latência de DRAM; um HDD, 10⁸x. Esconder isso — via cache, paralelismo, assincronia — é boa parte da engenharia de sistemas.

Complementa `COMPUTER_ARCHITECTURE.md`, `DB_RUNBOOK.md`, `POSTGRES_RUNBOOK.md`, `SHARDING.md`, `MEMORY_OPTIMIZATION.md`.

## Mídias

### HDD (Hard Disk Drive)

Platos magnéticos girando com cabeça móvel.

- **RPM**: 5400 (laptop), 7200 (desktop), 10k/15k (enterprise antigo).
- **Seek time**: 3-12 ms — dominante em aleatório.
- **Rotational latency**: ~4 ms a 7200 RPM.
- **Bandwidth sequencial**: 150-260 MB/s.
- **IOPS aleatório**: 70-200 — péssimo.
- **Capacidades**: até 30 TB (HAMR, SMR).
- **Tecnologias**: CMR (conventional), SMR (shingled — overlapping tracks, caro em writes), HAMR (heat-assisted), MAMR.

Uso remanescente: arquivo frio, bulk storage, backup. `$/TB` ainda melhor que SSD em capacidades grandes.

### SSD (Solid-State Drive)

Memória flash NAND. Sem partes móveis. Latência em µs, não ms.

#### Interfaces

| | Conexão | Bandwidth | Latência |
|---|---|---|---|
| SATA III | SATA 6 Gb/s | ~550 MB/s | ~60-100 µs |
| SAS | SAS 12-24 Gb/s | ~1-2 GB/s | ~50 µs |
| NVMe (PCIe) | PCIe 3-5 x4 | 3.5-14 GB/s | ~10-30 µs |

NVMe sobre PCIe é estado da arte; SATA sobrevive por compat.

#### Form Factors

- **2.5" SATA** (legacy).
- **M.2** (NVMe 2230, 2242, 2260, 2280, 22110): placa pequena.
- **U.2 / U.3**: 2.5" NVMe enterprise hot-swap.
- **E1.S / E1.L / E3.S**: EDSFF moderno DC.
- **HHHL** (half-height, half-length PCIe card).

#### Células NAND

Bits por célula:

| | Bits/célula | Endurance | Densidade | Uso |
|---|---|---|---|---|
| **SLC** | 1 | ~100k P/E | baixa | enterprise crítico |
| **MLC** | 2 | ~10k | | legado |
| **TLC** | 3 | ~3k | média | consumer/enterprise |
| **QLC** | 4 | ~1k | alta | read-heavy |
| **PLC** | 5 | ~150 | mais alta | pesquisa |

- **P/E cycles**: programa/apagamentos por bloco.
- **Wear leveling**: controller distribui writes para durar.
- **Over-provisioning**: espaço reservado não exposto (5-28%) acomoda wear + GC.

#### Tipos de NAND

- **2D planar**: antigo, limite físico.
- **3D NAND**: camadas verticais. Hoje 200+ camadas. Samsung V-NAND, Kioxia BiCS, Micron CMOS-under-array, SK hynix.

### Storage Class Memory (SCM)

Persistente + DRAM-like latência. Tentativa de meio-caminho.

- **Intel Optane** (3D XPoint): ~300 ns latência, byte-addressable via PMem. **Descontinuada em 2022** — economics não funcionou.
- **Samsung Z-NAND, Kioxia XL-Flash**: "fast NAND", ~5-10 µs. Niche.

Futuro: MRAM, ReRAM, FeRAM em nichos embarcados.

### Tape

Ainda vivo em arquivo frio. LTO-9 (~18 TB nativo / 45 TB compressed), LTO-10 (~30 TB). Barato por TB, lento, sequencial. AWS Glacier Deep Archive usa.

## Write Amplification, TRIM, GC

NAND tem características peculiares:

1. **Programa em páginas** (4-16 KB), **apaga em blocos** (megabytes).
2. **Não reprograma** sem apagar.

Isso gera:

- **Write amplification (WA)**: uma escrita lógica causa múltiplas escritas físicas devido a GC de blocos parcialmente válidos. WA de 1.5-3 é típico; SSDs enterprise tunam para <1.2.
- **Garbage Collection (GC)**: controller recupera espaço. Consome banda de fundo; afeta latência.
- **TRIM** (SATA) / **Deallocate** (NVMe): OS avisa controller quais blocos não são mais usados → GC evita movê-los. Crítico para performance sustained.
- **Over-provisioning** ajuda — mais margem para GC.
- **SLC cache**: SSD trata parte como SLC (rápido, pouco) mas flushes para TLC/QLC em idle. Write burst rápido; sustained cai após cache saturar.

## Controller SSD

CPU embarcado + firmware + DRAM cache. Responsabilidades:

- **FTL (Flash Translation Layer)**: mapeia LBAs ↔ NAND pages.
- **Wear leveling**.
- **ECC**: LDPC modern — NAND tem bit flip rate significativa.
- **GC**.
- **Encryption** at rest (AES-XTS).
- **Power loss protection (PLP)**: capacitores flush cache em queda — essencial em enterprise.

Firmware bugs causam CVEs (Samsung 840 PRO data loss, WD bricking). Update de firmware é tarefa do datacenter.

## NVMe

Protocolo moderno substituindo AHCI/SATA. Nativo PCIe. Diferencial: **múltiplas queues (até 64k)**, cada com **64k comandos** — paralelismo massivo latente pensado para flash.

- **Submission Queue (SQ) + Completion Queue (CQ)** por core em OS bem configurado.
- **Namespaces**: LUN-like, múltiplas "disks" lógicos em um SSD.
- **ZNS (Zoned Namespaces)**: expõe estrutura NAND; app gerencia. Elimina FTL, reduz WA, aumenta durabilidade. Complexo — usado em DC específico.
- **NVMe Sets, Endurance Groups**: QoS/isolation.
- **Streams, Directives**: hints para controller agrupar dados com vida semelhante.

### NVMe-oF

**NVMe over Fabrics**: NVMe via rede.

- **RDMA** (RoCE, InfiniBand): ~100 µs.
- **TCP**: ~200 µs mas universal.
- **FC-NVMe**: Fibre Channel.

Permite pool de SSDs compartilhado — desagregar storage de compute. Ceph, WEKA, Lightbits, Pure, VAST usam.

## Filesystems

Camada que organiza blocos em arquivos/diretórios, metadata, permissões.

### Linux

- **ext4**: padrão legado, maduro, estável. Journaling de metadata.
- **XFS**: jornaled, escala para arquivos/volumes grandes. Dominante em servidores.
- **Btrfs**: CoW (copy-on-write), snapshots, checksums, compression, RAID embutido. Volátil em RAID 5/6; estável em outros.
- **ZFS**: CoW, pooled storage, snapshots, clones, replicação, compression, dedup. Origem Solaris; OpenZFS no Linux. Excelente — pede RAM.
- **F2FS**: flash-friendly, log-structured. Mobile, alguns Chromebooks.
- **bcachefs**: "Btrfs feito direito", entrou no kernel 2023. Ainda amadurecendo.
- **tmpfs**: RAM-backed, volátil.
- **FUSE**: userland filesystems (s3fs, sshfs, rclone mount).

### Windows / macOS

- **NTFS** (Windows): journaled, ACLs ricas, reparse points, ADS. Padrão.
- **ReFS** (Windows Server): CoW, integridade.
- **APFS** (macOS): CoW, snapshots, encryption, space sharing, clones.

### Características-chave

- **Journaling**: log de metadata pré-commit. Recuperação após crash.
- **Copy-on-Write**: modificações em lugar novo; atomicidade natural; snapshots baratos.
- **Extents vs blocks**: extents (start, length) são mais eficientes que listas de blocos.
- **Compression**: LZO, Zstd, LZ4. Reduz footprint + eventualmente aumenta throughput (read menos bytes).
- **Checksums**: Btrfs, ZFS, bcachefs detectam bit rot. ext4 só em metadata.
- **Dedup**: online (durante write) ou offline. Caro em RAM (ZFS).
- **Encryption**: fscrypt (ext4/F2FS), LUKS (volume), BitLocker, FileVault.

## Volume Management

Camada entre dispositivos físicos e filesystem.

- **LVM (Linux)**: PVs, VGs, LVs. Snapshots, resize online.
- **mdadm**: software RAID Linux.
- **ZFS**: combina VM + FS.
- **Storage Spaces** (Windows): similar a ZFS.
- **dm-crypt / LUKS**: encryption de bloco.
- **dm-cache, bcache, dm-writecache**: caching tiered.

## RAID

**Redundant Array of Independent Disks**.

| Nível | Proteção | Performance | Custo |
|---|---|---|---|
| RAID 0 (stripe) | nenhuma | ótima | 0% |
| RAID 1 (mirror) | 1 disco | leitura boa, write 1x | 50% |
| RAID 5 | 1 disco | write penalty (RMW) | 1/N |
| RAID 6 | 2 discos | write mais caro | 2/N |
| RAID 10 (1+0) | depende | ótima | 50% |

**Observações críticas**:

- **URE (Unrecoverable Read Error)** em rebuild de RAID 5 com drives grandes é quase certeza. RAID 5 é obsoleto em drives > ~2TB.
- **Write hole**: ZFS/Btrfs resolvem via CoW; tradicional precisa UPS ou BBU.
- **Bit rot**: RAID sem checksums detecta falha de disco inteiro, não corrupção silenciosa. ZFS/Btrfs fazem escrub.
- **Hot spare + rebuild rápido** importa mais que nível alto.

## Object Storage

Chave → blob de tamanho arbitrário. **Não é filesystem** — sem diretórios, sem metadata POSIX.

- **S3** (AWS) — padrão de facto.
- **Compatíveis**: MinIO, Ceph RadosGW, Wasabi, Backblaze B2, R2 (Cloudflare), GCS, Azure Blob.
- **Durabilidade**: S3 standard tem "11 noves" (99.999999999%). Replicação cross-region + erasure coding.
- **Classes**: Standard, IA (Infrequent Access), Glacier (cold), Deep Archive.
- **Versioning, lifecycle, Object Lock** (WORM).
- **Throughput**: escala com objetos; um objeto tem limite. Sharding por prefix.
- **API**: S3 REST — presigned URLs para uploads/downloads diretos.

Uso: data lakes, backups, media, static assets. ZFS snapshot para S3 (Restic, Rclone) é padrão de backup moderno.

## Distributed Storage

Clusters que apresentam pool lógico sobre muitos nós.

- **Ceph**: unified (object + block + file). RADOS core, RBD (bloco), CephFS (filesystem), RGW (S3).
- **GlusterFS**: filesystem distribuído; menos moderno.
- **HDFS**: Hadoop; legado em batch analytics, substituído por object storage em muitos casos.
- **MinIO**: S3-compatible.
- **Vast Data, Pure Flashblade, Weka**: enterprise high-end.
- **Lustre, BeeGFS, GPFS**: HPC parallel filesystems.

**Trade-offs**:

- **CAP** (ver `DISTRIBUTED_SYSTEMS.md`).
- **Erasure coding vs replicação**: EC (k+m) usa menos espaço, mais CPU; replicação é simples, rápido, mais espaço.
- **Strong consistency** cara (2PC, consensus); eventual barata. Object stores tendem a eventual; filesystems distribuídos a strong.

## Caching de Storage

- **Page cache (Linux)**: RAM usa memória livre como cache de disco. Read de arquivo já tocado é RAM-speed. `free -h` mostra `buff/cache`.
- **ARC (ZFS)**: cache adaptativo próprio do ZFS.
- **L2ARC, SLOG**: ZFS tier secundário em SSD (leitura, ZIL).
- **bcache, dm-cache**: SSD como cache para HDD.
- **Redis/Memcached** no app (ver `CACHING.md`).

## I/O Stack no Linux

De cima para baixo:

1. **App** → read/write/mmap.
2. **VFS** (Virtual File System): interface comum.
3. **Filesystem driver**: ext4, XFS, etc.
4. **Block layer**: I/O scheduler, queueing, requests merge.
5. **Device driver**: SCSI, NVMe, virtio.
6. **Hardware**.

### I/O Schedulers

- **none / mq-deadline / kyber / bfq**: multi-queue modern. NVMe usa none/mq-deadline tipicamente — pouco ganho reorder em SSD moderno.
- **cfq, deadline** (legado single-queue).

### Async I/O

- **POSIX AIO** (`aio_read`): limitado.
- **Linux AIO (`io_submit`)**: só O_DIRECT, edge cases.
- **io_uring**: moderno (2019+), unified, performance extrema. APIs em Rust/Go/C++/Python.
- **epoll + nonblocking**: paradigma clássico. Não é AIO de disco strict mas usado em rede.

### Direct I/O vs Buffered

- **O_DIRECT**: bypassa page cache. Apps que fazem próprio cache (DBs) querem.
- **Buffered**: default. Page cache faz trabalho.

### fsync

Força descarga física. **Sem `fsync` antes do ack, commit não é durável**. DBs o chamam explicitamente. Custo: ~ms em SSD bom, ~10ms em ruim.

- **fsync vs fdatasync**: fdatasync só dado, não metadata. Mais rápido se metadata inalterado.
- **Battery-backed write cache** em RAID/controller permite ack antes de ir à mídia.
- **Write barriers**: garantem ordem mesmo sem fsync completo. Ext4 `data=ordered` faz.

## Performance — Medindo

### Ferramentas

- **fio**: o canônico. Flexível, reproducível.
- **dd**: quick-and-dirty. Suspeito para benchmark sério (cache effects).
- **iostat**: por device, taxas.
- **iotop**: por processo.
- **blktrace / btt**: per-request tracing.
- **bpftrace / perf**: eBPF para latência fina.

### Métricas

- **IOPS**: operações/segundo. Mais útil em random.
- **Bandwidth**: MB/s ou GB/s. Sequential dominado.
- **Latency**: µs/ms por operação. Distribuição (p50/p99/p99.9) mais útil que média.
- **Queue depth**: pedidos em voo. NVMe se beneficia de QD alto (32+).

### Workload patterns

- **Sequential** vs **random**.
- **Read-heavy** vs **write-heavy**.
- **Block size**: 4KB (DBs), 64KB-1MB (analytics).
- **Sync** vs **async**.

Nunca comparar números sem especificar workload.

## Durabilidade e Backups

### Falhas

- **Drive inteiro**: eventual; RAID mitiga.
- **Bit rot**: checksum + scrub.
- **Firmware bug**: raro, devastador. Fabric diversification.
- **Ransomware** (ver `INCIDENT_RESPONSE.md`): criptografa dados + backups online. **Imutável** (S3 Object Lock, WORM tape) é defesa.
- **Erro humano**: `rm -rf /`, DROP TABLE. Snapshots.
- **Site/desastre**: replicação off-site.

### Regra 3-2-1

3 cópias, 2 mídias diferentes, 1 off-site. Mínimo para dados importantes.

### Backup Tools

- **restic, borg, kopia**: dedup + encryption, modernos.
- **rsync**: sync; bom para mirror, ruim para histórico.
- **Duplicati, rclone, duplicity**: outras opções.
- **DB-native**: `pg_dump`, `mysqldump`, Percona XtraBackup.
- **Snapshots de filesystem** (ZFS, Btrfs): baratos, instantâneos.

### Restore testing

Backup sem restore testado periodicamente **não é backup**. Estatística: ~30% dos backups falham em restore real. Agende restore drill.

## Tiering e Hierarquia

Storage é cara ou lenta, não ambos. Múltiplos tiers com movimento automático:

- **Hot**: NVMe enterprise — latência baixa, caro.
- **Warm**: SSD de consumer, HDD rápido.
- **Cold**: HDD capacidade.
- **Archive**: tape, Glacier.

ILM (Information Lifecycle Management) define regras.

## Cloud Storage

### AWS

- **EBS**: block storage, gp3 balanceado, io2 alta IOPS.
- **S3**: object, tiers (Standard, IA, Glacier, Deep Archive).
- **EFS**: NFS managed.
- **FSx**: NetApp, OpenZFS, Lustre, Windows.

### GCP, Azure: equivalentes.

### Ephemeral vs persistente

- **Instance store / local SSD**: some com instance. Muito rápido.
- **Network-attached block** (EBS): persiste. Bandwidth limitada pelo tier.

### Performance

- **Burst credits, baseline**: atenção a gp2/gp3 IOPS model.
- **Cross-AZ latency**: +1-2 ms vs same-AZ.
- **S3 multi-part upload**: paralelizar > 5 MB.

## Princípios

1. **Latência não cai, banda sim**. SSD seek ~10 µs há anos.
2. **Filesystem escolhido importa**. ZFS/Btrfs trazem CoW + checksums; XFS escala; ext4 é sólido default.
3. **fsync é lei**. Ack sem fsync é mentira. DB só é durável se mídia confirmou.
4. **RAID não é backup**. Mirror não salva de DROP TABLE.
5. **Imutabilidade defende contra ransomware**. Snapshots, Object Lock, WORM tape.
6. **Teste restore**. Backup nunca restaurado é ilusão.
7. **SSD ≠ SSD**. QLC consumer vs SLC enterprise vs NVMe DC — 10x diferença em métricas importantes.
8. **Conheça seu workload**. Random 4KB vs sequential 1MB exigem setup diferente.
